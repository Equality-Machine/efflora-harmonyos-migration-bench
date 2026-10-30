# 模拟器 UI 测试

需要验证界面与交互还原度的任务用 Hypium UI 测试验证：构建在任务容器内完成，UI 测试运行在宿主机的 HarmonyOS 模拟器上。

## 工作方式

`tests/test.sh` 依次执行：

1. 导出 `model.patch`，把 `tests/ohosTest` 还原到 `entry/src/ohosTest`。
2. 在容器内构建应用 HAP 和测试 HAP。任一失败时奖励为 0。
3. 在容器内启动 hdc server，用 `hdc tconn` 直连模拟器的调试端口。
4. 对 `tests/fpoints.json` 里的每个 F-point：卸载 → 把两个 HAP 放进设备同一目录后 `bm install -p` → 按顺序运行各测试阶段，阶段之间 `aa force-stop` 模拟重启。所有阶段通过，该 F-point 才算通过。
5. 写入 `/logs/verifier/reward.json`：

   ```json
   {"reward": 0.75, "build": 1, "task_complete": 0, "F1": 1, "F2": 1, "F3": 1, "F4": 0}
   ```

   `reward` 是通过的 F-point 占比；`task_complete` 为 1 表示全部通过。

容器必须自己起 hdc server，不能连宿主机的 server：hdc 的文件传输由 server 读取本地文件，宿主机的 server 看不到容器里的 HAP。

## 宿主机准备

- Harbor CLI **0.24.0**（`uv tool install harbor==0.24.0`），任务 `task.toml` 使用 `schema_version = "1.4"`。
- 安装 DevEco Studio，创建并启动一台手机模拟器，**统一环境**为：HarmonyOS 6.1.1(24)、API 24、设备型号 Pura 90 Pro、分辨率 1256 × 2760、语言 zh-Hans-CN、时区 Asia/Shanghai。测试在这套环境下验证，换其他设备或分辨率可能失效。
- **新建的模拟器先手动完成一次输入法的首次使用**：点任意输入框调出键盘，同意系统输入法的使用协议。否则协议弹窗会遮住应用界面，所有 F-point 都会失败。
- 确认模拟器的 hdc 连接地址：在宿主机执行 `hdc list targets`，通常是 `127.0.0.1:5555`。
- 运行前设置模拟器在**容器内**的地址：

  ```bash
  export EFFLORA_HDC_DEVICE=host.docker.internal:5555   # Docker Desktop（macOS / Windows）的默认值
  harbor run -p tasks -i 'ios2hm-*' -a <agent> -m <provider/model>
  ```

  Linux 上 Docker 没有 `host.docker.internal`，改用宿主机在 docker 网桥上的地址（如 `172.17.0.1:5555`），并确认模拟器端口对该网段可达。

## 并发与稳定性

- 同一台模拟器上的多个 trial 通过设备上的锁目录 `/data/local/tmp/efflora_bench.lock` 依次执行。超过 1 小时未释放的锁视为残留并清除。需要更高并发时，可以启动多台模拟器，用不同的 `EFFLORA_HDC_DEVICE` 分批运行。
- 遇到已知的模拟器环境错误（如 `Can not connect to AAMS, RET_ERR_CONNECTION_EXIST`）时，该 F-point 自动重试一次。仍失败、或无法连接设备时，`test.sh` 不写奖励文件并以非 0 退出，Harbor 会记为验证异常，不计入 0 分。
- 设备不可用、HAP 安装出现环境错误等情况都不应算作 Agent 失败。
