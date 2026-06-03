# Non-ROS Threads

The thread configurator can also manage scheduling for threads that are not part of any ROS 2 executor
— for example, sensor driver threads or custom worker threads. Threads created through the helper below
report their information to `cie_thread_configurator` automatically, without requiring a ROS node.

## Usage

Use `cie_thread_configurator::spawn_non_ros2_thread` instead of `std::thread` to create the thread:

```cpp
#include "cie_thread_configurator/cie_thread_configurator.hpp"

// Instead of:
//   std::thread t(worker_function, arg1, arg2);

// Use:
auto t = cie_thread_configurator::spawn_non_ros2_thread(
    "worker_thread_name",   // unique thread name (used as the YAML id)
    worker_function,        // thread function
    arg1, arg2);            // optional arguments

t.join();
```

The thread name passed as the first argument must be unique among all threads managed by the
configurator, and must match the `id` field in the `non_ros_threads` section of the YAML configuration.

## CMake

Add `cie_thread_configurator` as a dependency:

```cmake
find_package(cie_thread_configurator REQUIRED)
ament_target_dependencies(your_target cie_thread_configurator)
```

## YAML Configuration

Configure non-ROS threads under the `non_ros_threads` section. The fields are the same as
`callback_groups`, except `id` is the thread name instead of an auto-generated CallbackGroup ID. All
scheduling policies, including `SCHED_DEADLINE`, are supported.

```yaml
non_ros_threads:
  - id: worker_thread_name
    policy: SCHED_FIFO
    priority: 85
    affinity:
      - 3
```

See the [YAML Specification](yaml-specification.md#non_ros_threads) for the full field reference.

## Prerun Mode

Non-ROS threads created with `spawn_non_ros2_thread` are also discovered by the prerun node and
included in the generated `template.yaml`.

!!! tip
    The `cie_sample_application` package includes `sample_non_ros_process`, which demonstrates using
    `spawn_non_ros2_thread` in a standalone process with no ROS 2 node:

    ```bash
    ros2 run cie_sample_application sample_non_ros_process
    ```
