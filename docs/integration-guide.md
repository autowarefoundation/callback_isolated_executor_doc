# Integration Guide

## Overview

0. **(Optional) Separate CallbackGroups** so that every callback you want to schedule strictly gets its own thread
1. **Install** the `callback_isolated_executor` packages
2. **Switch the executor** in your application code or launch file
3. **Generate a YAML template** by running the configurator in prerun mode
4. **Edit the YAML** to assign scheduling policies, priorities, and CPU affinities
5. **Grant `CAP_SYS_NICE`** to the thread configurator (one-time system configuration)
6. **Launch the configurator** with your config file, then start your application

See the [Tutorial](tutorial.md) for a concrete walkthrough with the sample application.

## Step 0 (Optional): Separate CallbackGroups

`CallbackIsolatedExecutor` assigns one OS thread per CallbackGroup, so the CallbackGroup is the unit of
scheduling: callbacks in the same group share a thread and therefore share scheduling parameters. Give
every callback whose scheduling you want to control strictly its own CallbackGroup:

```cpp
timer_group_ = create_callback_group(rclcpp::CallbackGroupType::MutuallyExclusive);
timer_ = create_wall_timer(100ms, std::bind(&MyNode::on_timer, this), timer_group_);

sub_group_ = create_callback_group(rclcpp::CallbackGroupType::MutuallyExclusive);
rclcpp::SubscriptionOptions sub_options;
sub_options.callback_group = sub_group_;
subscription_ = create_subscription<Msg>("topic", 10, callback, sub_options);
```

If callbacks were placed in one `MutuallyExclusive` group only to serialize access to shared state,
move them into separate groups and protect the shared state with a mutex instead (see
[Concurrency](#notes-on-adoption)).

## Step 1: Install

### Option A: apt

`callback_isolated_executor` is released on the ROS 2 build farm for Humble and Jazzy:

```bash
sudo apt install ros-$ROS_DISTRO-callback-isolated-executor
# Optional: the sample application used in the Tutorial
sudo apt install ros-$ROS_DISTRO-cie-sample-application
```

If `apt` cannot find the package for your distribution yet, build from source.

### Option B: Build from source

```bash
git clone https://github.com/autowarefoundation/callback_isolated_executor.git
cd callback_isolated_executor
source /opt/ros/$ROS_DISTRO/setup.bash
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```

## Step 2: Switch the Executor

When a node runs inside the `component_container_callback_isolated` container, you do **not** need to
modify the node's implementation — only the launch file (Option 2 below). If you start a node directly from a `main`
function without a component container, change the executor in code (Option 1) and rebuild.

Refer to the [`cie_sample_application`](https://github.com/autowarefoundation/callback_isolated_executor/tree/main/cie_sample_application)
package for a complete working example.

### Option 1: Launch without a ComponentContainer

Declare the dependency in `package.xml`:

```xml
<package format="3">
  ...
  <depend>callback_isolated_executor</depend>
  ...
</package>
```

Link against it in `CMakeLists.txt`:

```cmake
find_package(callback_isolated_executor REQUIRED)
...
ament_target_dependencies(your_executable ... callback_isolated_executor)
```

Then replace your executor in the `main` function:

```cpp
#include "callback_isolated_executor/callback_isolated_executor.hpp"

int main(int argc, char * argv[]) {
  rclcpp::init(argc, argv);

  auto node = std::make_shared<SampleNode>();
  auto executor = std::make_shared<CallbackIsolatedExecutor>();

  executor->add_node(node);
  executor->spin();

  rclcpp::shutdown();
  return 0;
}
```

### Option 2: Launch with a ComponentContainer

Use `component_container_callback_isolated` from the `callback_isolated_executor` package as the
container executable:

```xml
<launch>
  <node_container pkg="callback_isolated_executor" exec="component_container_callback_isolated"
                  name="sample_container" namespace="">
    <composable_node pkg="cie_sample_application" plugin="SampleNode" name="sample_node" namespace="">
      ...
    </composable_node>
  </node_container>
</launch>
```

Alternatively, load a node into an existing container:

```xml
<launch>
  <load_composable_node target="sample_container">
    <composable_node pkg="cie_sample_application" plugin="SampleNode" name="sample_node" namespace="">
    </composable_node>
  </load_composable_node>
</launch>
```

If you modify application source code (Option 1), rebuild before continuing.

## Step 3: Generate a YAML Template

Open two terminals. In the first, start the `prerun` node **before** launching your application; in the
second, launch your application. The prerun node does not need any special privileges.

```bash
# Terminal 1: start the prerun node first
ros2 run cie_thread_configurator prerun_node
# or: ros2 launch cie_thread_configurator thread_configurator.launch.xml prerun:=true

# Terminal 2: then launch your application
ros2 launch your_package your_launch.xml
```

As the application starts, the prerun node logs one entry per CallbackGroup, each showing the
CallbackGroup ID and its OS thread ID. Once all nodes are up and the log output settles, press
`Ctrl+C` in the prerun terminal. A `template.yaml` is created in the current directory, then stop the
target application.

The template captures hardware information from the system (CPU details via `lscpu`) under
`hardware_info`. This is used to validate compatibility when the configuration is later loaded. See the
[YAML Specification](yaml-specification.md) for the full format.

## Step 4: Edit the YAML

Rename and edit the template to configure each CallbackGroup:

```bash
mv template.yaml your_config.yaml
```

For CallbackGroups that do not require configuration, either delete the entry or leave it unchanged —
the defaults in `template.yaml` use the standard CFS scheduler with the default nice value and no
affinity. See the [YAML Specification](yaml-specification.md) for every available option.

## Step 5: Grant CAP_SYS_NICE to the Thread Configurator

The `cie_thread_configurator` applies the scheduling parameters from your YAML file to the threads of
other processes through `sched_setscheduler(2)`, `sched_setattr(2)`, `setpriority(2)`, and
`sched_setaffinity(2)`. These calls require the `CAP_SYS_NICE` capability. There are two ways to grant
it.

### Option 1: systemd service with AmbientCapabilities (easiest)

Ambient capabilities are inherited across `execve`, so `ros2 run` and the node binary receive
`CAP_SYS_NICE` without modifying the binary and without the dynamic-linker restrictions described in
Option 2.

Create `/etc/systemd/system/thread_configurator.service`:

```ini
[Unit]
Description=CIE thread configurator

[Service]
User=<your-user>
AmbientCapabilities=CAP_SYS_NICE
# Match the target application's environment, e.g.:
# Environment=ROS_DOMAIN_ID=0
ExecStart=/bin/bash -c 'source /opt/ros/humble/setup.bash && exec ros2 run cie_thread_configurator thread_configurator_node --ros-args -p config_file:=/absolute/path/to/your_config.yaml'

[Install]
WantedBy=multi-user.target
```

For a source build, source your workspace's `install/setup.bash` instead of `/opt/ros/humble/setup.bash`.
Then load and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl start thread_configurator   # or: sudo systemctl enable --now thread_configurator
journalctl -u thread_configurator -f       # follow the configurator's log
```

### Option 2: setcap on the binary

```bash
sudo setcap cap_sys_nice+ep \
  $(ros2 pkg prefix cie_thread_configurator)/lib/cie_thread_configurator/thread_configurator_node
```

File capabilities are attached to the binary itself, so re-run this after every rebuild or package
upgrade.

#### Configure library paths

After `setcap`, the dynamic linker ignores `LD_PRELOAD` and `LD_LIBRARY_PATH` for security reasons.
Register the required library directories explicitly by creating a file under `/etc/ld.so.conf.d/` whose
name ends in `.conf` — for example `/etc/ld.so.conf.d/callback-isolated-executor.conf`:

```text
/opt/ros/humble/lib
/opt/ros/humble/lib/x86_64-linux-gnu
```

For a source build, also add the `lib` directory of `cie_config_msgs` in your install space, for example
`/path/to/callback_isolated_executor/install/cie_config_msgs/lib`.

Apply the change:

```bash
sudo ldconfig
```

??? note "Why is ldconfig needed?"

    When specific permissions are granted to an ELF binary using `setcap`, environment variables like
    `LD_PRELOAD` and `LD_LIBRARY_PATH` are ignored for security reasons. Setting `RUNPATH` on the binary
    comes to mind as an alternative, but `RUNPATH` does not handle recursive dynamic linking well. In
    such cases, modifying `/etc/ld.so.conf.d/` is the only option.

### Kernel boot parameter (SCHED_DEADLINE only)

According to the [Linux kernel documentation](https://www.kernel.org/doc/Documentation/scheduler/sched-deadline.txt),
setting CPU affinity for `SCHED_DEADLINE` tasks requires cgroup v1. To enable cgroup v1, disable cgroup
v2 by adding `systemd.unified_cgroup_hierarchy=0` to the kernel boot parameters. Edit
`/etc/default/grub`:

```text
GRUB_CMDLINE_LINUX_DEFAULT="... systemd.unified_cgroup_hierarchy=0 ..."
```

Apply and reboot:

```bash
sudo update-grub
sudo reboot
```

## Step 6: Launch with the Scheduler Configuration

Start the configurator node with your config file **before** launching the target application. If you
created the systemd service in Step 5, `systemctl start thread_configurator` already does this;
otherwise run:

```bash
ros2 run cie_thread_configurator thread_configurator_node --ros-args -p config_file:=your_config.yaml
# or: ros2 launch cie_thread_configurator thread_configurator.launch.xml config_file:=your_config.yaml
```

On startup the configurator validates the hardware: it compares `hardware_info` in the config file
against the current system and reports an error on mismatch, for example:

```text
[ERROR] Hardware validation failed with the following mismatches:
  - CPU family: expected '5', got '6'
```

After successful validation, the configurator prints the settings and waits for the application. When
you launch the application, the configurator receives each CallbackGroup's information and applies the
configured policy immediately upon receipt — including `SCHED_DEADLINE` (which uses
`SCHED_FLAG_RESET_ON_FORK` so that forked children reset to `SCHED_OTHER`).

The configurator keeps running after all configurations are applied. This lets it automatically
re-apply settings when the target application restarts (the OS may reuse thread IDs, so thread-ID
equality cannot be used to skip reconfiguration). If your configuration includes `SCHED_DEADLINE`
threads with CPU affinity (configured via cgroup), the cgroup directories are cleaned up when the
configurator node is terminated.

!!! note "SCHED_DEADLINE requires root"
    A thread cannot be set to `SCHED_DEADLINE` with capabilities alone, so if any CallbackGroup uses
    `SCHED_DEADLINE`, the configurator must run as **root** (with the systemd service, omit `User=`).
    If the target application runs under a specific `ROS_DOMAIN_ID`, the configurator must use the same
    domain ID:

```bash
sudo bash -c "export ROS_DOMAIN_ID=[app domain id]; \
  source /path/to/callback_isolated_executor/install/setup.bash; \
  ros2 run cie_thread_configurator thread_configurator_node --ros-args -p config_file:=your_config.yaml"
```

## Notes on Adoption

!!! warning "Concurrency"
    Replacing `rclcpp::executors::SingleThreadedExecutor` with `CallbackIsolatedExecutor` creates a
    dedicated thread per CallbackGroup, enabling parallel execution. This can expose concurrency bugs
    that were previously hidden by serial execution. Before adopting, review whether any shared state
    requires a mutex or other synchronization. Callbacks within the same `MutuallyExclusiveCallbackGroup`
    are still executed serially.

!!! warning "Real-time policies"
    When using real-time policies such as `SCHED_FIFO` or `SCHED_DEADLINE`, carefully consider the
    potential to delay kernel processing and other applications. Configure affinity, priority, and
    [throttling](https://man7.org/linux/man-pages/man7/sched.7.html) appropriately.
