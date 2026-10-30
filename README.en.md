<div align="center">

# Efflora HarmonyOS Migration Bench

**An open-source benchmark for evaluating coding agents on migrating iOS / Android apps to HarmonyOS**

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Task Format](https://img.shields.io/badge/task%20format-Harbor-6f42c1.svg)](https://docs.harborframework.com/tasks/overview)
[![Hugging Face](https://img.shields.io/badge/Hugging%20Face-Dataset-FFD21E?logo=huggingface&logoColor=FFD21E)](https://huggingface.co/datasets/efflora/harmonyos-migration-bench-runs)

English | [简体中文](README.md)

**📦 Trajectories & artifacts: [huggingface.co/datasets/efflora/harmonyos-migration-bench-runs](https://huggingface.co/datasets/efflora/harmonyos-migration-bench-runs)**

</div>

---

## Overview

**Efflora HarmonyOS Migration Bench** is an open-source benchmark for HarmonyOS app migration, started by Efflora. It evaluates the end-to-end ability of AI coding agents to **migrate real-world apps from other mobile stacks to HarmonyOS**.

Many existing apps need to move from iOS / Android to HarmonyOS, yet existing coding benchmarks barely cover cross-stack migration. This benchmark sets out to answer one question:

> Given a real iOS / Android project and a migration brief, can an agent migrate the app the way a competent HarmonyOS engineer would, and deliver a functionally equivalent HarmonyOS project that is ready for production?

Every task follows the [Harbor Framework](https://docs.harborframework.com) task format, so any agent/model supported by the `harbor` CLI can be evaluated out of the box.

## Task Categories

| Category | Task id prefix | Input | Agent must | Verification |
| --- | --- | --- | --- | --- |
| **iOS → HarmonyOS** | `ios2hm-` | iOS project (SwiftUI / UIKit) + migration brief, optionally a HarmonyOS scaffold | Migrate the app / module to a functionally equivalent ArkTS project | Build + hidden unit/UI tests |
| **Android → HarmonyOS** | `android2hm-` | Android project (Compose / View) + migration brief, optionally a HarmonyOS scaffold | Migrate the app / module to a functionally equivalent ArkTS project | Build + hidden unit/UI tests |

See [`docs/categories.md`](docs/categories.md) for detailed definitions and task authoring requirements.

## Environment Requirements

All tasks, reference solutions and tests target the following environment (see [`docs/environment.md`](docs/environment.md)):

| Item | Value |
| --- | --- |
| OS version | HarmonyOS 6.1.1(24) |
| API level | 24 |
| Device | Pura 90 Pro |
| Resolution | 1256 × 2760 |
| Time zone | Asia/Shanghai |
| Locale | zh-Hans-CN |
| Harbor | 0.24.0 (`schema_version = "1.4"` in `task.toml`) |

## Repository Layout

```text
efflora-harmonyos-migration-bench/
├── tasks/                       # Tasks (Harbor format), flat; category determined by task id prefix
│   └── <task-id>/                # One self-contained directory per task, e.g. android2hm-catima-full-app
│       ├── instruction.md        # Task instructions for the agent
│       ├── task.toml             # Task configuration and metadata
│       ├── environment/          # Dockerfile and build context (HarmonyOS scaffold, etc.; the source-platform project is fetched at build time)
│       ├── solution/             # Reference solution (solve.sh) to verify task solvability
│       └── tests/                # Verification scripts (test.sh), hidden from the agent during execution
├── docker/
│   └── harmonyos-base/           # Build files for efflora/harmonyos-base (download the toolchain separately)
├── scripts/
│   ├── validate_tasks.py         # Validate task structure and metadata
│   └── validate_submission.py    # Validate evaluation submissions (Harbor job directories)
├── docs/
│   ├── categories.md             # Category definitions and task authoring guidelines
│   ├── environment.md            # Environment requirements (OS / device / time zone / locale / Harbor version)
│   ├── task-format.md            # Harbor task format conventions used in this benchmark
│   ├── emulator-ui-tests.md      # Emulator setup and integration
│   └── evaluation.md             # Evaluation workflow, metrics and result submission
├── CONTRIBUTING.md
├── NOTICE
└── LICENSE
```

**Task naming**: `<category>-<app>-<short-desc>`, all lowercase and hyphen-separated, e.g. `android2hm-catima-full-app` or `android2hm-aegis-full-app`. All tasks live directly under `tasks/<task-id>/`, with categories determined by the prefix; their registered Harbor names are `efflora/<task-id>`.

## Quick Start

### 1. Prepare the Environment

- Docker (at least 4 CPUs and 8 GB of memory recommended)
- [Harbor CLI](https://docs.harborframework.com) **0.24.0**: `uv tool install harbor==0.24.0` (or `pip install harbor==0.24.0`)
- Task base image `efflora/harmonyos-base:latest`: download the HarmonyOS command-line tools `commandline-tools-linux-x64-6.1.1.418.zip` (Command Line Tools 6.1.1 Release, bundled API 24 SDK; `build.sh` checks its SHA-256), then build locally with `docker/harmonyos-base/build.sh`; see [`docker/harmonyos-base/README.md`](docker/harmonyos-base/README.md)
- Tasks with emulator UI tests (those whose `tests/fpoints.json` defines F-points) also require a HarmonyOS emulator meeting the environment requirements, with the `EFFLORA_HDC_DEVICE` environment variable set; see [`docs/emulator-ui-tests.md`](docs/emulator-ui-tests.md)

### 2. Verify Task Solvability (Oracle)

```bash
harbor run -p tasks -a oracle
```

### 3. Evaluate an Agent

```bash
# Evaluate all tasks
harbor run -p tasks -a claude-code -m anthropic/claude-opus-5-5

# Evaluate one category (filter by task id prefix)
harbor run -p tasks -i 'android2hm-*' -a claude-code -m anthropic/claude-opus-5-5

# Evaluate one task
harbor run -p tasks/android2hm-catima-full-app -a claude-code -m anthropic/claude-opus-5-5
```

Results are written to `jobs/` by default. See [`docs/evaluation.md`](docs/evaluation.md) for evaluation metrics and result submission.

## Evaluation Principles

- **Real-world projects**: Source projects come from real open-source iOS / Android apps, and migrations are verified against the source app's behavior.
- **Executable verification**: Evaluation relies on successful builds and hidden tests, with LLM scoring excluded as the primary metric.
- **Reproducibility**: Upstream code is pinned to specific commits, image versions are fixed, and reference solutions must pass all checks.
- **Leakage prevention**: `tests/` and `solution/` are hidden from the agent during execution.

## Contributing

Contributions of new tasks, evaluation environment improvements and issue reports are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

```bash
harbor task init efflora/android2hm-myapp-some-module -p tasks --no-pytest
python3 scripts/validate_tasks.py tasks/android2hm-myapp-some-module
```

## License

Code, task instructions and tests are licensed under [Apache-2.0](LICENSE). Third-party upstream projects used by tasks (the iOS / Android source apps, fetched at build time) keep their original licenses (see each task's `metadata.upstream_license` and [NOTICE](NOTICE)); reference solutions (`solution/`) ported from such a project follow that project's license (e.g. GPL-3.0), see [NOTICE](NOTICE). The HarmonyOS SDK is governed by Huawei's own terms and is not distributed here.

## Citation

```bibtex
@misc{efflora-harmonyos-migration-bench,
  title  = {Efflora HarmonyOS Migration Bench: Benchmarking Coding Agents on Migrating Mobile Apps to HarmonyOS},
  author = {Efflora},
  year   = {2026},
  url    = {https://github.com/Equality-Machine/efflora-harmonyos-migration-bench}
}
```

## Acknowledgments

The design of this benchmark draws on excellent open-source projects including [Harbor Framework](https://github.com/laude-institute/harbor), [Terminal-Bench](https://github.com/laude-institute/terminal-bench) and [SWE-bench](https://github.com/SWE-bench/SWE-bench).
