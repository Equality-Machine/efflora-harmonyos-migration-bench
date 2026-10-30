# Migrate Aegis Authenticator from Android to HarmonyOS

You are given a complete, real-world open-source Android application and must migrate it into a HarmonyOS
application written in ArkTS / ArkUI.

## Input

- Android source (read-only): `/workspace/source`
  - Aegis Authenticator, a 2FA (TOTP / HOTP / Steam / Yandex / mOTP) vault, version 3.4.3.
  - Upstream: https://github.com/beemdevelopment/Aegis @ `d6f4e5925a97e4e91593f1542085eae03432a759` (GPL-3.0)
  - Java, XML views, about 28,000 lines of non-test code.
- HarmonyOS project: `/workspace/app`, an empty Stage-model skeleton (single `entry` module, API 24).

## What to build

Migrate the whole app into `/workspace/app`, not a demo.

1. **Functional parity.** Every user-facing screen, menu action, dialog, setting, gesture, background
   behaviour, notification and data import/export path of the Android app needs a HarmonyOS counterpart
   that actually performs the behaviour. This includes the encrypted vault (scrypt + AES-GCM), lock/unlock,
   importers, exports and backups. UI shells, placeholders, hard-coded demo data, mocks and `TODO` stubs
   count as not migrated. Data must persist across app restarts the way it does on Android.
2. **Structural correspondence.** Keep the Android app's screen and data-layer decomposition recognisable.
   Each Activity or Fragment becomes a registered ArkTS page or navigation destination. Models, vault,
   importers and preferences keep equivalent names and responsibilities.
3. **Platform-native replacements.** Replace Android APIs with HarmonyOS Kit equivalents (ArkUI,
   `@ohos.data.preferences`, `@kit.CryptoArchitectureKit`, `@kit.CoreFileKit` pickers, `@kit.BasicServicesKit`
   pasteboard, `@kit.ScanKit`, `@kit.UserAuthenticationKit`, etc.). Port the required behaviour of third-party
   libraries instead of dropping the feature. Declare every permission the migrated features need in
   `module.json5`. Where HarmonyOS genuinely differs (for example, the phone document picker cannot select
   folders), pick the closest platform-native behaviour.
4. **Resources.** Migrate strings (the default English strings and the Simplified Chinese translation
   `values-zh-rCN`, plus the other locales the app offers in its language setting), colors, drawables and
   icons, and app identity: name, icon, and the version name `3.4.3`.
5. **Builds.** The project must build:

   ```bash
   cd /workspace/app && hvigorw --mode module -p product=default -p buildMode=debug assembleHap --no-daemon
   ```

   Fix all ArkTS compile errors (strict typing: no `any`/`unknown`, no untyped object literals, no structural
   typing, etc.).

Rules:

- Target API 24 (HarmonyOS 6.1.1) only.
- Read and write only inside `/workspace/app`. Do not modify `/workspace/source`.
- No device or emulator is available in this environment, so do not try to install or run the app.
- Network access may be used for HarmonyOS documentation, not for downloading an existing HarmonyOS port of
  this app.
- Keep `@ohos/hypium` in the root `oh-package.json5` `devDependencies`. It is needed to build test HAPs.

## Interface contract (relied on by automated UI tests)

The finished app is exercised on a HarmonyOS phone emulator through its UI. The device language is
**Simplified Chinese (zh-Hans-CN)**, time zone Asia/Shanghai. The tests
locate controls only by visible text, by `.id()`, or by a text field's placeholder (hint). They cannot see
`.accessibilityText()`. The tests rely on the following:

- **Identity.** Bundle name `com.a2h.migration` and main ability `EntryAbility` stay unchanged. A fresh install
  starts with the Android intro (Welcome, then Security, then Setup completed).
- **Text.** Visible labels, titles, menu items, dialog titles, messages and buttons, preference titles,
  summaries, chips, list footers and toasts show exactly what the Android app shows on a zh-CN device: its
  Simplified Chinese strings verbatim (`res/values-zh-rCN/strings.xml` with the `values-zh-rCN-v29`
  overrides). Where Android has no zh-rCN translation for a string (for example the literal items of string arrays such as the
  Language list, or texts Android hardcodes), the default English string applies (`res/values`, `values-v29`).
  Dialog buttons that Android takes from the system (OK / Cancel) read 确定 / 取消. Plurals, placeholders,
  dates and numbers are formatted as Android formats them under zh-CN (e.g. the list footer
  `正在显示 8 个条目`). Entry cards show the issuer and the account name as separate texts; next to the
  issuer the name may be shown as `(name)`, as on Android. OTP codes are shown as text, grouped as on Android
  (e.g. `140 937`).
- **Ids.** Icon-only controls carry `.id()` equal to the Android resource id name:

  | Control | id |
  | --- | --- |
  | Main toolbar: Search, Sort, Lock | `mi_search`, `action_sort`, `action_lock` |
  | Search text field (expanded search) | `search_src_text` |
  | Floating "+" button on the main list; floating confirm button on "Import entries" | `fab` |
  | HOTP refresh button on an entry card | `buttonRefresh` |
  | Selection action bar: Favorite, Copy, Edit, Transfer (QR), Delete | `action_toggle_favorite`, `action_copy`, `action_edit`, `action_share_qr`, `action_delete` |
  | Entry editor and group manager: Save | `action_save` |
  | Import entries: toggle all checkboxes | `toggle_checkboxes` |
  | Group manager rows: rename / remove | `button_edit`, `button_delete` |
  | Intro: next / finish (round arrow / check) button | `btnNext` |
  | Entry editor: show/hide toggle of the Secret field | `text_input_end_icon` |
  | TOTP countdown bars (the shared bar under the chips and per-entry bars) | `progressBar` |

  For controls without an Android id, use this fixed set: `overflow_menu` (the ⋮ overflow button of any
  toolbar or action bar), `navigate_up` (toolbar back / close arrow) and `action_mode_close` (the × that ends
  the selection action bar). Menu items behind `overflow_menu` (e.g. Settings, About, Select all, Assign to
  group, Delete, Restore default icon, Reset usage count, Delete unused groups, Wipe vault contents) are found
  by their text.
- **Text fields.** Text fields use the Android hint, as shown under zh-CN, as their placeholder: for example the
  editor's Name, Issuer, Note, Secret, Period (seconds) / Counter, Digits, PIN, Usage count and Group name
  fields, and the Password field on the unlock screen. In password dialogs the password field comes before the
  confirmation field.
- **Controls.** Dropdowns (e.g. Type, Hash function, Export format) show their current value as text and open
  when tapped. Their options are picked by text. Tapping a switch preference's title toggles it, and the switch
  on the same row exposes its checked state. Checkboxes sit on the same row as their label.
- **Data in and out.** Data is loaded through the app's own *Settings > Import & Export > Import from file*
  flow (and the intro's *Import Aegis vault*), using the HarmonyOS system document picker on files in the
  device's `Download` folder. Exports and backup files are saved with the system save picker into
  `Download`, using the proposed file names. The clipboard is read back by pasting into the app's search
  field through the system paste control.
- **Screen security.** Screen security defaults to **on**, as in the Android release build. It must use the
  window privacy mode only (blank screenshots), which does not block UI-test interaction.
- **Emulator.** QR images cannot be generated on the emulator. An error dialog about it on the Transfer screen
  is acceptable if it can be dismissed with OK, but the rest of the screen and of any export must still work.
