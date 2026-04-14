srn:

sanjith ks 
PES1UG24CS613

pradhyut
PES1UG24CS646

# Build, Load, and Run Instructions

```bash
# Build
cd boilerplate
make clean
make

# Load kernel module
sudo insmod monitor.ko

# Verify control device
ls -l /dev/container_monitor

# Start supervisor (Terminal 1)
cd ~/OS-Jackfruit/OS-Jackfruit
sudo ./boilerplate/engine supervisor ./rootfs-base

# Create per-container writable rootfs copies (Terminal 2)
cd ~/OS-Jackfruit/OS-Jackfruit
rm -rf rootfs-alpha rootfs-beta
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

# Start two containers
sudo ./boilerplate/engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./boilerplate/engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96

# List tracked containers
sudo ./boilerplate/engine ps

# Inspect logs
sudo ./boilerplate/engine logs alpha

# Memory monitoring demonstration
cp boilerplate/memory_hog rootfs-alpha/

sudo ./boilerplate/engine start memwarn ./rootfs-alpha /memory_hog --soft-mib 10 --hard-mib 50
sleep 5
sudo dmesg | tail -20

sudo ./boilerplate/engine start memkill ./rootfs-alpha /memory_hog --soft-mib 10 --hard-mib 20
sleep 5
sudo dmesg | tail -20

# Scheduling experiment
cp boilerplate/cpu_hog rootfs-alpha/
cp boilerplate/cpu_hog rootfs-beta/

sudo ./boilerplate/engine start cpu1 ./rootfs-alpha /cpu_hog --nice 0
sudo ./boilerplate/engine start cpu2 ./rootfs-beta /cpu_hog --nice 10

sudo ./boilerplate/engine ps

# Stop containers
sudo ./boilerplate/engine stop alpha
sudo ./boilerplate/engine stop beta

# Stop supervisor and inspect kernel logs
sudo pkill engine
sudo dmesg | tail

# Unload module
cd boilerplate
sudo rmmod monitor





# Engineering Analysis

## 1. Namespace Isolation

The project uses PID, UTS, and mount namespaces to isolate each container from the host system and from other containers. A PID namespace gives every container its own process numbering, so processes inside the container begin at PID 1 and cannot directly observe host processes. A UTS namespace allows each container to have a different hostname, which makes the containers appear independent even though they run on the same kernel. A mount namespace isolates the filesystem view of the container, allowing each container to use its own root filesystem and `/proc` mount.

This design demonstrates how Linux containers achieve isolation without requiring a separate virtual machine. The host and all containers still share the same kernel, but namespaces provide the illusion that each container is running on its own system.

## 2. Why a Supervisor Process is Needed

The supervisor exists because the operating system must keep track of every container after it has been launched. Without a long-lived parent process, there would be no single place to maintain metadata such as container IDs, host PIDs, states, and memory limits.

The supervisor also performs process reaping. When a container exits, Linux temporarily keeps the process entry until its parent collects the exit status with `waitpid()`. If the supervisor did not do this, zombie processes would accumulate. The project demonstrates this by showing that containers can be started, stopped, and cleaned up without leaving defunct processes behind.

## 3. IPC and Logging Design

The project uses two different IPC mechanisms because control commands and log traffic have different requirements. Control commands such as `start`, `stop`, `ps`, and `logs` are sent through a UNIX-domain socket. This allows short request-response messages between the CLI and the supervisor.

Container output is handled differently. Standard output and standard error from each container are redirected into a pipe. The supervisor reads from the pipe and places the data into a bounded buffer. A separate logging thread consumes the buffer and writes the data into per-container log files.

Using two IPC paths avoids interference between control traffic and log traffic. The bounded buffer also demonstrates the producer-consumer synchronization pattern used by operating systems.

## 4. Memory Monitoring and Enforcement

The kernel module periodically checks the resident memory usage (RSS) of each monitored process. Two thresholds are used. Exceeding the soft limit generates a warning but allows the process to continue. Exceeding the hard limit causes the module to terminate the process.

The distinction between soft and hard limits reflects how operating systems often separate monitoring from enforcement. A soft limit is useful because it warns that a process is consuming too many resources while still allowing it to continue. A hard limit is necessary to protect the system from runaway processes that could exhaust memory and destabilize the machine.

The project demonstrates that user-space alone is not always sufficient for resource control. Because the monitor runs inside the kernel, it can observe memory usage more accurately and immediately enforce limits.

## 5. Linux Scheduling Behavior

The scheduling experiment demonstrates how Linux distributes CPU time between competing processes. Two CPU-bound workloads were run with different nice values. The container with the lower nice value received higher scheduling priority and therefore completed sooner.

The experiment with `cpu_hog` and `io_pulse` also demonstrates the difference between CPU-bound and I/O-bound workloads. The CPU-bound process continuously requests CPU time, while the I/O-bound process frequently blocks waiting for I/O. Linux uses this behavior to keep interactive and I/O-heavy workloads responsive even when CPU-intensive processes are running at the same time.

These experiments show that Linux scheduling is not purely round-robin. Instead, it adapts to process priority and workload type in order to balance fairness, responsiveness, and throughput.



# Design Decisions and Tradeoffs

## Namespace Isolation

We decided to use PID, UTS, and mount namespaces for every container. PID namespaces make the processes inside a container look like they are running on their own system, starting from PID 1. UTS namespaces let us give each container a different hostname, and mount namespaces allow every container to have its own filesystem.

The downside is that setting all of this up takes extra work. We had to configure `clone()`, mount `/proc`, and use `chroot()` correctly before the container could run. Even though it was more complicated, it was still the best choice because it gave us proper container-style isolation without needing a full virtual machine.

## Supervisor Architecture

We used one long-running supervisor process to manage every container. The supervisor keeps track of the container ID, PID, state, and memory limits. It also handles stopping containers, collecting logs, and cleaning up exited processes.

The main drawback is that the supervisor becomes a single point of failure. If it crashes, the socket disappears and no more commands can be sent to the containers. We still chose this design because it made the project much easier to manage. Having one place that knows about every container simplified logging, cleanup, and communication with the kernel module.

## IPC and Logging

For communication, we used two different IPC mechanisms. Control commands such as `start`, `stop`, `ps`, and `logs` go through a UNIX-domain socket. Container output is handled separately using a pipe and a bounded buffer.

Using two different mechanisms made the code a little more complicated, because both channels had to be maintained separately. However, it turned out to be the better approach. The control commands stayed responsive even when a container was producing a lot of output, and the bounded buffer made it easy to demonstrate the producer-consumer model.

## Kernel Monitor

We implemented the memory monitor as a kernel module instead of a normal user-space program. The module periodically checks the memory usage of each container and either prints a warning or kills the process if it crosses the configured limit.

The tradeoff is that kernel modules are harder to work with. They require root access, can only run on Linux, and mistakes can crash the system. Even so, this was still the right choice because the kernel has direct access to accurate memory usage information, which made the monitoring much more reliable.

## Scheduling Experiment

For the scheduling experiment, we compared containers with different `nice` values and also compared a CPU-heavy workload with an I/O-heavy workload.

The exact timings changed slightly between runs, since Linux scheduling depends on what else is happening on the system. Even with that variation, the overall pattern stayed the same: the process with the lower nice value finished first, and the I/O-heavy workload stayed responsive.

# Scheduler Experiment Results

We ran two versions of the experiment.

In the first test, both containers ran `cpu_hog`, but one had a nice value of `0` and the other had a nice value of `10`.



From this, we could clearly see that Linux gives more CPU time to processes with a lower nice value.

In the second test, we ran `cpu_hog` together with `io_pulse`.



This experiment showed that Linux does not treat every process in exactly the same way. CPU-bound processes keep using the processor, while I/O-bound processes often get quick bursts of CPU time whenever they wake up. That is why the `io_pulse` container still felt responsive even while the CPU-heavy container was running.# os_submission
