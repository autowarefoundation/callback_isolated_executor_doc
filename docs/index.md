# CallbackIsolatedExecutor

`CallbackIsolatedExecutor` is a ROS 2 executor — and a matching component container — that assigns a
**dedicated OS thread to each CallbackGroup**, enabling per-callback scheduling control (policy,
priority, and CPU affinity) handled entirely by the Linux kernel.

It is a standalone `rclcpp`-based package: it requires no changes to `rcl` or `rclcpp` and can be
installed and used on its own.

## Why

Standard ROS 2 executors share a pool of threads across callbacks — you cannot control which callback
runs on which CPU or at what priority.

```text
Standard Executor:                  CallbackIsolatedExecutor:

  CallbackGroup A ─┐                 CallbackGroup A → Thread (FIFO, prio=90, CPU0)
  CallbackGroup B ─┼→ Thread Pool    CallbackGroup B → Thread (FIFO, prio=80, CPU1)
  CallbackGroup C ─┘                 CallbackGroup C → Thread (CFS,  nice=0)

  No per-callback control            Full OS-level scheduling control
```

With `CallbackIsolatedExecutor`, each CallbackGroup maps to exactly one OS thread. Scheduling is then
delegated to the Linux kernel: you assign `SCHED_FIFO`, `SCHED_RR`, `SCHED_DEADLINE`, or CFS
(`SCHED_OTHER` / `SCHED_BATCH`) parameters per CallbackGroup through a single YAML file, applied by the
`cie_thread_configurator`.

For the design rationale and evaluation, see the RTAS 2025 paper
[*Middleware-Transparent Callback Enforcement in Commoditized Component-Oriented Real-Time Systems*](https://arxiv.org/pdf/2505.06546).

This package continues the development of [`sykwer/callback_isolated_executor`](https://github.com/sykwer/callback_isolated_executor), the original implementation accompanying that paper.

## Supported Environments

| Category | Supported Versions / Notes |
|----------|----------------------------|
| ROS 2 | Humble, Jazzy (only with the `rclcpp` client library) |
| Linux Distribution | Ubuntu 22.04 (Jammy Jellyfish), Ubuntu 24.04 (Noble Numbat) |

## Install

The packages are released on the ROS 2 build farm:

```bash
sudo apt install ros-$ROS_DISTRO-callback-isolated-executor
```

See the [Integration Guide](integration-guide.md) for building from source and for the remaining steps.

## Pages

- [Integration Guide](integration-guide.md) — Build, install, switch the executor, and set up the thread configurator
- [YAML Specification](yaml-specification.md) — Complete reference for the thread configuration file
- [Non-ROS Threads](non-ros-threads.md) — Managing scheduling for threads outside the ROS 2 executor
- [Tutorial](tutorial.md) — End-to-end walkthrough with the sample application
- [Comparison with Other Executors](comparison-with-other-executors.md) — Real-time properties versus other ROS 2 executors

## Adoption in Autoware

Most of Autoware's major nodes already run on `CallbackIsolatedExecutor`, and the remaining nodes are
scheduled to follow (see the
[Autoware discussion](https://github.com/orgs/autowarefoundation/discussions/6813) and the
[migration issue](https://github.com/autowarefoundation/autoware_core/issues/968)). In TIER IV's
robotaxi, applying `CallbackIsolatedExecutor` with tuned scheduling attributes to the five bottleneck
DAGs (top LiDAR, localization, perception, planning, and control) improved the measured worst-case
response time by roughly 5x compared with default CFS scheduling.

## Relation to Agnocast

[Agnocast](https://github.com/autowarefoundation/agnocast) provides an Agnocast-compatible port of this
executor (`agnocast::CallbackIsolatedAgnocastExecutor`) that additionally schedules Agnocast's zero-copy
callbacks. If you use Agnocast, refer to the
[CallbackIsolatedExecutor section of the Agnocast documentation](https://autowarefoundation.github.io/agnocast_doc/callback-isolated-executor/).
This site documents the original, `rclcpp`-only `CallbackIsolatedExecutor`.

## Citation

If you find `CallbackIsolatedExecutor` useful in your research, please consider citing:

> T. Ishikawa-Aso, A. Yano, T. Azumi, and S. Kato, "Work in Progress: Middleware-Transparent Callback
> Enforcement in Commoditized Component-Oriented Real-Time Systems," in Proc. of 31st IEEE Real-Time
> and Embedded Technology and Applications Symposium (RTAS), 2025, pp. 426–429.

??? note "BibTeX"

    ```bibtex
    @inproceedings{ishikawa2025work,
      title={Work in Progress: Middleware-Transparent Callback Enforcement in Commoditized Component-Oriented Real-Time Systems},
      author={Ishikawa-Aso, Takahiro and Yano, Atsushi and Azumi, Takuya and Kato, Shinpei},
      booktitle={Proc. of 31st IEEE Real-Time and Embedded Technology and Applications Symposium (RTAS)},
      pages={426--429},
      year={2025},
      organization={IEEE}
    }
    ```
