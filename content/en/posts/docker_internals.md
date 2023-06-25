---
title:  "Query Optimization: Tunning indexes for better performance."
date:  2023-06-18
draft:  false
enableToc: true
enableTocContent: true
description: "I talk about AI and NLP in the context of my graduation project."
tags:
- misc
image: "images/docker/docker_internals.png"
---

# Docker Internals

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1674997774330/f1b20304-0c03-4192-b59c-0e0e4b27e924.jpeg)

## Table of Content

- control groups
  - Hierarchical representation
  - Interacting with Cgroups
  - Demo
  - How Docker leverages Cgroups
- Namespaces
  - Network namespaces
  - Mount namespaces
  - procfs namespace
  - creating and entering namespaces
  - persisting namespace
  - Demo
- How Docker Leverages Namespaces

## **Introduction**

This Docker internals series will be divided into 3 parts

1. Definitive Guide to Cgroups, Namespaces

2. Low-Level Layering and Storage [Overlay and storage drivers, UnionFS]

3. Handcrafting containers without Docker using Linux kernel primitives

## Containers

containers are an abstraction over several different Linux technologies, think of containers as a particular way of combining those Linux primitives together

in the end, we will combine what we've learned so far by actually building a container without the need for Docker!

## **control groups**

cgroups is a Linux system for tracking, grouping, and organizing the processes that run

> every process is tracked with cgroups, regardless of whether it's in a container or not!

cgroups are typically used to associate processes with resources with Cgroups you can track how much a group of processes is using for a given kind of resource besides giving you the ability to limit or prioritize what resources are available to a group of processes

the way you interact with cgroups, is with subsystems are concrete implementations that are actually bound to resources there are a bunch of subsystems that are available on Linux :

`blkio` — this subsystem sets limits on input/output access to and from block devices such as physical drives (disk, solid state, or USB).

`cpu` — this subsystem uses the scheduler to provide cgroup tasks access to the CPU.

`cpuacct` — this subsystem generates automatic reports on CPU resources used by tasks in a group.

`cpuset` — this subsystem assigns individual CPUs (on a multicore system) and memory nodes to tasks in a group.

`devices` — this subsystem allows or denies access to devices by tasks in a cgroup. `freezer` — this subsystem suspends or resumes tasks in a cgroup. [used by docker pause]

`memory` — this subsystem sets limits on memory use by tasks in a cgroup and generates automatic reports on memory resources used by those tasks.

`net_cls` — this subsystem tags network packets with a class identifier (classid) that allows the Linux traffic controller (tc) to identify packets originating from a particular cgroup task.

`net_prio` — this subsystem provides a way to set the priority of network traffic per network interface dynamically.

`ns` — the namespace subsystem.

`perf_event` — this subsystem identifies cgroup membership of tasks and can be used for performance analysis.

those subsystems are called "resources controllers", meaning they can track or limit a particular kind of resource for the processes assigned to the group

😁 each of the subsystems is independent, they can organize processes separate from each other for example you could have a single cpu cgroup with two processes, but those two processes can be assigned to two different memory cgroup

### **Hierarchical representation**

All of the cgroup subsystems arrange processes in a hierarchy each subsystem has independent hierarchy every task or process id running on the host is represented in exactly one of the cgroups within a given subsystem hierarchy

Rule 1: A single hierarchy can have one or more subsystems attached to it.

![Alt text](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-6-Resource_Management_Guide-en-US/images/fe94409bf79906ecb380e8fbd8063016/RMG-rule1.png)

Rule 2: Any single subsystem (such as cpu) cannot be attached to more than one hierarchy if one of those hierarchies has a different subsystem attached to it already.

![Alt text](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-6-Resource_Management_Guide-en-US/images/c4b0445881422c88d957e352911bccd8/RMG-rule2.png)

Rule 3:

![Alt text](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-6-Resource_Management_Guide-en-US/images/fb48098033d1c4ccdb5a55516c9cb816/RMG-rule3.png)

Rule 4:

![Alt text](https://access.redhat.com/webassets/avalon/d/Red_Hat_Enterprise_Linux-6-Resource_Management_Guide-en-US/images/67e2c07808671294692acde9baf0b452/RMG-rule4.png)

what does this dependability offer us? it offers an expressive segmentation across resource types, for example, you could have two processes share the total amount of memory that they consume, but give one process more cpu time than the other

another note: some resource controllers apply settings from the parent level to the child levels, while others consider each level in the hierarchy independently! when a new process is started it begins in the same cgroup as its parent

### **Interacting with cgroups**

you can interact with cgroups, via a virtual file system typically mounted at `/sys/fs/cgroup` a bunch of files and folders are here, but they are just interfaces into the kernel's data structures for cgroups

- each directory inside a given "subsystem" root, represents a cgroups, in each directory, you'll see a file called "task"

This file holds all of the processes is for processes assigned to that particular cgroup

### **Demo**

```bash
$ ls /sys/fs/cgroup/devices
cgroup.clone_children  devices.allow  devices.list       tasks
cgroup.procs           devices.deny   notify_on_release
```

here you can see the files that control how the cgroup subsystems itself are configured which prefixes with cgroup and controlling devices prefixed with devices and task file

:question: what if I want to see which cgroup any process belongs to? we can do this by using the proc virtual file system

the `proc` file system contains a directory that corresponds to each process id

```bash
$ cat /sys/fs/cgroup/devices/tasks
1
16
$ echo $$
1
$ ls /proc
1          crypto       ioports      loadavg       partitions   timer_list
17         devices      irq          locks         sched_debug  tty
acpi       diskstats    kallsyms     mdstat        schedstat    uptime
buddyinfo  dma          kcore        meminfo       self         version
bus        driver       key-users    misc          softirqs     vmallocinfo
cgroups    execdomains  keys         modules       stat         vmstat
cmdline    filesystems  kmsg         mounts        swaps        zoneinfo
config.gz  fs           kpagecgroup  mtrr          sys
consoles   interrupts   kpagecount   net           sysvipc
cpuinfo    iomem        kpageflags   pagetypeinfo  thread-self
$ cat /proc/1/cgroup

27:name=systemd:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
26:rdma:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
25:pids:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
24:hugetlb:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
23:net_prio:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
22:perf_event:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
21:net_cls:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
20:freezer:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
19:devices:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
18:memory:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
17:blkio:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
16:cpuacct:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
15:cpu:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
14:cpuset:/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
0::/docker/ef9bd7c7f5b6f74b67814d4fe694256a6f397ea00a607184ca8e7a7bcbfb6cc8
```

now let's see how to move a process to a cgroup and change some settings on that cgroup

we can make a new cgroup by creating a new directory

```bash
❯ echo $$
3147

❯ cat /proc/3147/cgroup 
13:rdma:/
12:blkio:/user.slice
11:perf_event:/
10:pids:/user.slice/user-1000.slice/user@1000.service
9:freezer:/
8:memory:/user.slice/user-1000.slice/user@1000.service
7:cpu,cpuacct:/user.slice
6:hugetlb:/
5:devices:/user.slice
4:cpuset:/
3:net_cls,net_prio:/
2:misc:/
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope
0::/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope



❯ sudo mkdir /sys/fs/cgroup/pids/lfnw


❯ ls /sys/fs/cgroup/pids/lfnw
cgroup.clone_children  notify_on_release  pids.events  tasks
cgroup.procs        pids.current   pids.max

# let's write this current shell into the tasks file so the shell is moved into the cgroup
❯ echo 3147 | sudo tee /sys/fs/cgroup/pids/lfnw/tasks
3147

❯ cat /proc/3147/cgroup                             
13:rdma:/
12:blkio:/user.slice
11:perf_event:/
10:pids:/lfnw
9:freezer:/
8:memory:/user.slice/user-1000.slice/user@1000.service
7:cpu,cpuacct:/user.slice
6:hugetlb:/
5:devices:/user.slice
4:cpuset:/
3:net_cls,net_prio:/
2:misc:/
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope
0::/user.slice/user-1000.slice/user@1000.service/apps.slice/apps-org.gnome.Terminal.slice/vte-spawn-8a2ac531-aa46-4222-aedf-2d9ef54b37af.scope


❯ cat /sys/fs/cgroup/pids/lfnw/tasks      
3147
3602


❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3629


❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3653


❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3677


❯ cat /sys/fs/cgroup/pids/lfnw/tasks
3147
3701

## notice that with every time we cat the file, a new process id is found?
## from where comes the new one?
# because as we've mentioned before, with the child process here (cat) inherit from parent process (bash) the cgroup
```

the pid cgroup controls the number of processes that can run, and limit the impact of mistakes or attacks

the limit is controlled by pids.max file let's see an example

```bash
echo 2 | sudo tee /sys/fs/cgroup/pids/lfnw/pids.max
```

now try to cat the file again `bash $(cat /sys/fs/cgroup/pids/lfnw/tasks 1>&2)`

### **How Docker uses Cgroup?**

we're going to run a container with a cpu resource control

```bash
> docker run --name demo --cpu-shares 256 -d --rm amazonlinux sleep 600
```

Let's take a look at the container subsystem

```sh
❯ docker exec demo ls /sys/fs/cgroup/cpu


❯ docker exec demo cat /sys/fs/cgroup/cpu/cpu.shares
256
# It's looks like our settings got applied to the root c group
## but that's just from the container perspective
```

Let's look at what it looks like outside the container as well.

Docker likes to organize container cgroups inside a docker directory

```sh
❯ ls /sys/fs/cgroup/cpu/docker

> cat /sys/fs/cgroup/cpu/docker/b9229671392a2c9fa60ff357611c394b6428010441f62a0876c5bba9e1365131/cpu.shares
256
```

## **Namespaces**

- Namespaces are concerned with controlling resource visibility and access control [isolation mechanism]

- Namespaces can make it appear to the process that it has its own isolated copy of a given resource

- Namespaces can map resources outside a namespace to a resource inside a namespace while changing some properties like names and permissions ⛔ changes to resources within namespaces can be invisible outside the namespace, you can say that

> the process that's making the changes has its own private copy of the resource

⛔ Just like with cgroups, processes can be in any combination of namespaces the processes can share different sets of namespaces with each other, for example, You could have a mount namespace that is shared between most processes but want to run one of them in a separate network or pid namespaces or you can run processes with its own namespaces much like a container

## **** Frequently used namespaces**

image.png

### **Network namespaces**

- Giving a process-separated view of the network with different network interfaces and routing rules

- Network namespaces can be connected together using [veth] (a virtual ethernet device)

> docker uses a separate network namespace per container

### **Mount Namespaces**

```bash
mount

output:
```

### **Procfs virtual filesystem**

```bash
readlink /proc/$$/ns/*


output:
```

### **Creating Namespaces**

### **Persisting Namespaces**

### **Entering Namespaces**

### **Demo**

`docker0` a Linux bridge, this is what docker uses for container networking

I am going to use a command utility called unshare to create a new namespace and run `ip link` inside

```sh
sudo unshare --net ip link
```

we can see the different output here, instead of three interfaces, we see only a one interface `lo` it's actually a different `lo` than the one that we saw before, as this is inside the new network namespace

as we talked before out `persistance of namespace` since we created a new network namespace and run `ip link` and exits, the namespace is gone so let's see how can we make this network ns be persistent

we talked about the symlink files in the `proc` file system, so let's look at those

```sh
# we are now operating inside a new shell inside the new network namespace
sudo unshare --net bash
> ip link
# you'll see the same lo interface as before
> readlink /proc/$$/ns/*
# we can make the namespace persist by making a bind mount
> touch /var/run/netns/lfnw
> mount --bind /proc/$$/ns/net /var/run/netns/lfnw

# ip netns list # can list the persistent namespace
> ip netns list
lfnw
# up netns identify # identify if we are currently running this namespace
> ip netns identify
lfnw
> exit

# now we existed the namespace and inside the original shell, let's see if the namespace actually has been persisted

> ip netns list
lfnw
> ip netns identify
# nothing


> sudo ip netns exec lfnw ip link
# you'll see the same output just as before
```

### **Namespaces with Docker**

we're going to see how docker sets up the Namespaces

```sh
docker run --name redis -d redis

ps ax | grep redis
sudo readlink /proc/1990/ns/net
# now we know the identifier of the namesapce

# let's grap what docker thinks the ip address is
docker inspect redis | jq .[0].NetworkSettings.IPAddress
"ip bla bla"

# now let's look inside the namespace to see if that's right
> sudo nsenter --target <pid> --net ifconfig eth0

# you can see they match up
```

we know the docker connects the container to the docker zero bridge with a v-eth pair

```sh
ip link | grep veth

ifconfig veth443600a
```

### **Demo**

in this demo, I want to use a binary from the container and run it on the host

we ran redis from a container, redis here isn't installed on the host!

but we can use the redis binary from the container and run it inside all of the other host namespaces

to enter a namespace, we need one of those special files in proc we can use nsenter with targt and mount options to enter the mount namespace

```sh
> docker inspect --format "{{ .NetworkSettings.IPAddress}}" redis
172.17.0.2

telnet 172.17.0.2 6379


ps ax | grep redis
8526 ?        Ssl    0:18 redis-server *:6379
  10803 pts/1    S+     0:00 grep redis

sudo nsenter --target 8526 --mount /usr/local/bin/redis-server
```

you can see that redis-server starts up we can switch to another terminal and try hitting that as well

- Access binaries in your container with the mount namespace

---

## Resources
