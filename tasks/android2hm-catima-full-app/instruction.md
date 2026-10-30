# Migrate Catima (Android loyalty-card wallet) to HarmonyOS

You are given a complete, real-world open-source Android application and must migrate it into a native
HarmonyOS application written in ArkTS / ArkUI.

## Input

- Android source (read-only): `/workspace/source`
  - Catima, upstream https://github.com/CatimaLoyalty/Android @ `4be0ce14489d825e16b48d7d04aed3558ac1eabb` (GPL-3.0)
  - Java + Kotlin, XML Views, about 15,900 lines of non-test code. Domain: loyalty-card wallet.
- HarmonyOS project: `/workspace/app` — an empty Stage-model skeleton (single `entry` module, API 24 /
  HarmonyOS 6.1.1). Do the migration in this project.

## Requirements

1. **Functional parity.** Migrate the whole app, not a demo. Every user-facing screen, menu action, dialog,
   setting, gesture, background behaviour and data import/export path of the Android app needs a HarmonyOS
   counterpart that actually performs the behaviour. UI shells, placeholders, hard-coded demo data, mocks and
   `TODO` stubs count as not migrated.
2. **Structural correspondence.** Keep the Android app's screen and data-layer decomposition recognisable:
   each Activity / Fragment becomes a registered ArkTS page or navigation destination; data models, DAOs,
   helpers and services keep equivalent names and responsibilities.
3. **Platform-native replacements.** Replace Android APIs with HarmonyOS Kit equivalents (ArkUI,
   `@ohos.data.relationalStore`, `@ohos.data.preferences`, `@kit.ScanKit`, `@kit.CoreFileKit`,
   `@kit.MediaLibraryKit`, `@kit.ImageKit`, `@kit.CryptoArchitectureKit`, `@kit.FormKit`, …). Port the required
   behaviour of third-party libraries (ZXing, uCrop, zip4j, CSV) rather than dropping the feature. Declare every
   permission the migrated features need in `module.json5`.
4. **Builds.** The project must build with the HarmonyOS SDK, fixing all ArkTS compile errors (strict typing: no
   `any`/`unknown`, no untyped object literals, no structural typing, …):

   ```bash
   cd /workspace/app && hvigorw --mode module -p product=default -p buildMode=debug assembleHap --no-daemon
   ```

5. **Resources.** Migrate strings — the default English locale `res/values/strings.xml` **and** the Simplified
   Chinese locale `res/values-zh-rCN/strings.xml` (the evaluation device runs in Chinese); the other locales are
   part of the app's language setting (e.g. German) — plus colours, drawables / icons and the app identity
   (name, icon) into `resources/`.

Rules: do not modify `/workspace/source`; do not look for or download an existing HarmonyOS port of Catima
(network access may be used for HarmonyOS documentation). There is no device or emulator in this environment;
do not try to install or run the app. Do not introduce third-party `ohpm` dependencies you cannot vendor; keep
the `@ohos/hypium` dev dependency that the skeleton declares.

## Interface contract

The finished app is exercised on a HarmonyOS emulator (Pura 90 Pro, 1256 × 2760, system language
**zh-Hans-CN**, Asia/Shanghai)
through its UI, in the same way a user would use the Android app. Automated UI tests can find controls only by
their visible **text** or by their ArkUI **`.id()`**; the uitest `description` attribute does not expose
`.accessibilityText()`, so accessibility labels alone are not enough. Keep to the following:

### Identity

- Bundle name stays `com.a2h.migration`, the launcher ability stays `EntryAbility` (module `entry`).

### Texts

- Visible labels, screen titles, menu items, dialog titles / messages / buttons, tab names, preference titles
  and summaries, list-option names, toasts and validation errors show **exactly what the Android app shows on a
  device set to Simplified Chinese (zh-CN)**: the string from `app/src/main/res/values-zh-rCN/strings.xml`
  verbatim (plurals included — zh-rCN only has the `other` form, e.g. `选中了 1 张卡`, `2 卡片`,
  `删除 2 张卡片`, `新余额：US$70.00`, `字段不能为空`), and the default English string from
  `res/values/strings.xml` where zh-rCN has no translation (e.g. `Catima`, `Fidme (BETA)`). Text inputs, text
  areas and dropdown / select fields expose their current value as the component's text.
- Dates and amounts are formatted as the Android app formats them under zh-CN (`DateFormat.MEDIUM` / `LONG`,
  `NumberFormat` currency / number instances): `2030年1月15日` in lists and in the editor, `US$100.00`,
  `€12.50`, `150 点`, balance field `100.00`. Seed data (card names, notes, card IDs) is shown as stored.
- Each card in the card list shows its name as a text.

### Component ids

Icon-only (or otherwise text-less) controls carry `.id()` equal to the Android resource id name. Ids used:

| Screen | Control | `.id()` |
| --- | --- | --- |
| any toolbar | navigate up (back arrow) | `navigate_up` |
| main list | search / display options / sort | `action_search`, `action_display_options`, `action_sort` |
| main list | ⋮ overflow menu (normal and selection mode) | `overflow_menu` |
| main list | add-card floating button | `fabAdd` |
| main list, selection mode | edit / share selected | `action_edit`, `action_share` |
| card tile (main list, edit group) | starred / archived badge, selected check mark | `star`, `archivedIcon`, `selected_thumbnail` |
| card view | share / star toggle / ⋮ overflow menu | `action_share`, `action_star_unstar`, `action_overflow` |
| card view | header thumbnail (tap / long press) | `icon_holder` |
| card view | main barcode / image area, previous / next image | `main_image`, `main_left_button`, `main_right_button` |
| card view | bottom bar: info, previous card, next card, update balance | `bottom_app_bar_info_button`, `bottom_app_bar_previous_button`, `bottom_app_bar_next_button`, `bottom_app_bar_update_balance_button` |
| card view | edit floating button | `fabEdit` |
| card editor | save floating button, thumbnail | `fabSave`, `thumbnail` |
| card editor, Card tab | name, card ID, barcode value, barcode type, barcode encoding | `storeNameEdit`, `cardIdView`, `barcodeIdField`, `barcodeTypeField`, `barcodeEncodingField` |
| card editor, Options tab | note, currency, balance, valid from, expiry date | `noteEdit`, `balanceCurrencyField`, `balanceField`, `validFromField`, `expiryField` |
| card editor, Photos tab | front / back image holders | `frontImageHolder`, `backImageHolder` |
| select barcode | card ID field | `cardId` |
| groups | add floating button | `fabAdd` |
| groups, each row | move up / move down / edit / delete | `moveUp`, `moveDown`, `edit`, `delete` |
| edit group | name field, save floating button | `editTextGroupName`, `fabSave` |
| image cropper | confirm, rotate | `menu_crop`, `menu_rotate` |

Badges and check marks are only present while the state applies (a card that is not starred shows no `star`).

### Platform behaviour the tests rely on

- **Data import** goes through the app's own *导入/导出 → 使用文件系统 → (format) → 确定* flow (*Import/export → From filesystem → … → OK* in English), which
  opens the system document picker (`DocumentViewPicker`); the tests pick backups from the device's
  **Download** folder (*浏览 → 我的手机 → Download*). System dialogs and pickers appear in Chinese. **Export** writes through the system save dialog
  (suggested name `catima_<yyyyMMdd>.zip`) and then shows the *导出成功* (*Exported*) dialog with *发送……* (*Send…*).
- **Images** are picked with the system photo picker, followed by the app's own cropper (title
  `设置正面图像` / `设置背面图像`, aspect choices including the card shape labelled with the upper-cased `card` string (`卡片`), confirm `menu_crop`).
- **Dates** (valid from / expiry) are picked with the ArkUI date picker dialog (year / month / day wheels,
  confirmed with *确定*).
- **Sharing** cards or an export uses the system share sheet.
- **Camera**: the tests deny the camera permission (*不允许*). The *扫描条码* screen must still offer
  *更多选项* (add without barcode, enter manually, …) when the camera is unavailable.
- **Barcode images** cannot be generated on the emulator (Scan Kit reports "Emulator is not supported"). The app
  must still treat a value that is valid for a barcode type as valid (e.g. it can be selected in *选择条码*
  and saved) and only reject values that are invalid for the type, with Android's message. No test
  looks at barcode images.
- Theme, pure-black and language changes take effect immediately, as on Android (`recreate()`); the
  language list offers the locales the Android app ships (e.g. *Deutsch*) and *系统* to follow the device
  language again.
- Volume keys switch cards in the card view when the corresponding setting is on.
