# 任务分类与出题规范

本基准聚焦**跨平台迁移**：给定一个其他移动端技术栈的真实工程，要求 Agent 将其迁移为功能等价的 HarmonyOS（ArkTS / ArkUI）应用。所有任务都必须满足 [通用要求](#通用要求)，并按各自分类补充要求出题。

## 通用要求

1. **真实性**：源工程来自真实开源项目或贴近真实业务的自研项目，具备完整的页面、数据与交互。
2. **可验证**：存在确定性的自动化验证；`tests/test.sh` 必须向 `/logs/verifier/reward.txt` 写入奖励值；需要记录细分结果时可同时写 `reward.json`（Harbor 优先读取 `reward.json`）。
3. **可解**：`solution/solve.sh` 在干净环境中执行后，测试必须 100% 通过（`harbor run -a oracle`）。
4. **不可平凡通过**：未做任何修改的初始环境运行测试，奖励必须为 0。
5. **说明自洽**：`instruction.md` 只描述迁移范围、等价性要求和必要约束，不泄露解法；测试只验证说明中提到或可从源工程合理推断的行为。
6. **可复现**：源工程固定到 commit，依赖版本锁定，构建不依赖不稳定的外部服务（网络接口需 mock 或本地化）。
7. **环境一致**：遵循 [基准环境约定](environment.md)（HarmonyOS 6.1.1(24) / API 24、Pura 90 Pro 1256 × 2760、Asia/Shanghai、zh-Hans-CN、Harbor 0.24.0）。

---

## 1. iOS → HarmonyOS（`ios2hm-`）

**场景**：给定一个 iOS 工程（SwiftUI / UIKit / Objective-C），要求 Agent 迁移为鸿蒙应用。

- 环境：
  - `/workspace/source`：iOS 源工程固定 commit 的快照（只读参考，容器内无法运行 Xcode）；
  - `/workspace/app`：鸿蒙工程脚手架或部分迁移完成的工程。
- 说明：明确迁移范围（整应用 / 指定模块 / 指定页面）、需对齐的行为，以及测试定位所需的约束（组件 id、路由名、方法签名等）。
- 测试：
  - 工程能通过 `hvigorw assembleHap`；
  - 隐藏单元测试覆盖迁移后的数据模型、业务逻辑与持久化；
  - 模拟器 UI 测试覆盖关键页面与交互流程。
- 典型迁移点：SwiftUI 声明式 UI → ArkUI、`@State`/`@Binding`/`ObservableObject` → `@State`/`@Link`/`@Observed`、`NavigationStack` → `Navigation`、`UserDefaults`/Core Data → Preferences / RDB、`URLSession` → `@kit.NetworkKit`、通知 / 权限 / 生命周期差异。

## 2. Android → HarmonyOS（`android2hm-`）

**场景**：给定一个 Android 工程（Jetpack Compose / View 体系，Kotlin / Java），要求 Agent 迁移为鸿蒙应用。

- 环境：
  - `/workspace/source`：Android 源工程固定 commit 的快照（只读参考）；
  - `/workspace/app`：鸿蒙工程脚手架或部分迁移完成的工程。
- 说明与测试要求同上。
- 典型迁移点：Compose / XML 布局 → ArkUI、`ViewModel`/`LiveData`/`Flow` → 状态管理与 Emitter、Activity / Fragment / Intent → UIAbility / 页面路由 / Want、Room / SharedPreferences → RDB / Preferences、WorkManager / Service → 后台任务、运行时权限差异。

> 后续可按同一规范扩展其他源技术栈（如 `flutter2hm-`、`rn2hm-`），新增前缀需同步更新 `scripts/validate_tasks.py`。

---

## 环境内路径约定

| 路径 | 含义 |
| --- | --- |
| `/workspace/source` | 源平台工程（iOS / Android），只读参考 |
| `/workspace/app` | 鸿蒙工程根目录（Agent 的主要工作目录） |
| `/tests` | 验证阶段由 Harbor 挂载的测试文件（Agent 不可见） |
| `/logs/verifier/reward.txt` / `reward.json` | 奖励输出 |
