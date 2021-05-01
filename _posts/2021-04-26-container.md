---
layout: article
tags: Linux
title: Container
mathjax: true
key: Linux
---

## Namespace
```
A namespace wraps a global system resource in an abstraction that
       makes it appear to the processes within the namespace that they
       have their own isolated instance of the global resource.  Changes
       to the global resource are visible to other processes that are
       members of the namespace, but are invisible to other processes.
       One use of namespaces is to implement containers.

Namespace用来封装系统资源，每个Namesapce内的资源，对于Namespace里面的进程来说，是对立的。  
改变Namespace里面的资源对于Namespace里面的进程可见，但是对于Namespace外的进程不可见。

Namespace是将内核的全局资源做封装，使得每个Namespace都有一份独立的资源，因此不同的进程在各自的Namespace内对同一种资源的使用不会互相干扰。  
实际上，Linux内核实现namespace的主要目的就是为了实现轻量级虚拟化（容器）服务。  
在同一个namespace下的进程可以感知彼此的变化，而对外界的进程一无所知。  
这样就可以让容器中的进程产生错觉，仿佛自己置身于一个独立的系统环境中，以此达到独立和隔离的目的。
```

### Linux支持的Namespace类型
```
Namespace Flag            Page                  Isolates
Cgroup    CLONE_NEWCGROUP cgroup_namespaces(7)  Cgroup root directory

IPC       CLONE_NEWIPC    ipc_namespaces(7)     System V IPC, POSIX message queues

Network   CLONE_NEWNET    network_namespaces(7) Network devices,stacks, ports, etc.

Mount     CLONE_NEWNS     mount_namespaces(7)   Mount points

PID       CLONE_NEWPID    pid_namespaces(7)     Process IDs

Time      CLONE_NEWTIME   time_namespaces(7)    Boot and monotonic clocks

User      CLONE_NEWUSER   user_namespaces(7)    T{User and group IDsT}

UTS       CLONE_NEWUTS    uts_namespaces(7)     Hostname and NIS domain name
```

### The namespaces API
```
As well as various /proc files described below, the namespaces
API includes the following system calls:

clone(2)
       The clone(2) system call creates a new process.  If the
       flags argument of the call specifies one or more of the
       CLONE_NEW* flags listed below, then new namespaces are
       created for each flag, and the child process is made a
       member of those namespaces.  (This system call also
       implements a number of features unrelated to namespaces.)

setns(2)
       The setns(2) system call allows the calling process to
       join an existing namespace.  The namespace to join is
       specified via a file descriptor that refers to one of the
       /proc/[pid]/ns files described below.

unshare(2)
       The unshare(2) system call moves the calling process to a
       new namespace.  If the flags argument of the call
       specifies one or more of the CLONE_NEW* flags listed
       below, then new namespaces are created for each flag, and
       the calling process is made a member of those namespaces.
       (This system call also implements a number of features
       unrelated to namespaces.)

ioctl(2)
       Various ioctl(2) operations can be used to discover
       information about namespaces.  These operations are
       described in ioctl_ns(2).

Creation of new namespaces using clone(2) and unshare(2) in most
cases requires the CAP_SYS_ADMIN capability, since, in the new
namespace, the creator will have the power to change global
resources that are visible to other processes that are
subsequently created in, or join the namespace.  User namespaces
are the exception: since Linux 3.8, no privilege is required to
create a user namespace.
```

### The /proc/[pid]/ns/ directory
```
Each process has a /proc/[pid]/ns/ subdirectory containing one
entry for each namespace that supports being manipulated by
setns(2):

    $ ls -l /proc/$$/ns | awk '{print $1, $9, $10, $11}'
    total 0
    lrwxrwxrwx. cgroup -> cgroup:[4026531835]
    lrwxrwxrwx. ipc -> ipc:[4026531839]
    lrwxrwxrwx. mnt -> mnt:[4026531840]
    lrwxrwxrwx. net -> net:[4026531969]
    lrwxrwxrwx. pid -> pid:[4026531836]
    lrwxrwxrwx. pid_for_children -> pid:[4026531834]
    lrwxrwxrwx. time -> time:[4026531834]
    lrwxrwxrwx. time_for_children -> time:[4026531834]
    lrwxrwxrwx. user -> user:[4026531837]
    lrwxrwxrwx. uts -> uts:[4026531838]

Bind mounting (see mount(2)) one of the files in this directory
to somewhere else in the filesystem keeps the corresponding
namespace of the process specified by pid alive even if all
processes currently in the namespace terminate.

Opening one of the files in this directory (or a file that is
bind mounted to one of these files) returns a file handle for the
corresponding namespace of the process specified by pid.  As long
as this file descriptor remains open, the namespace will remain
alive, even if all processes in the namespace terminate.  The
file descriptor can be passed to setns(2).

In Linux 3.7 and earlier, these files were visible as hard links.
Since Linux 3.8, they appear as symbolic links.  If two processes
are in the same namespace, then the device IDs and inode numbers
of their /proc/[pid]/ns/xxx symbolic links will be the same; an
application can check this using the stat.st_dev and stat.st_ino
fields returned by stat(2).  The content of this symbolic link is
a string containing the namespace type and inode number as in the
following example:

    $ readlink /proc/$$/ns/uts
    uts:[4026531838]

The symbolic links in this subdirectory are as follows:

/proc/[pid]/ns/cgroup (since Linux 4.6)
       This file is a handle for the cgroup namespace of the
       process.

/proc/[pid]/ns/ipc (since Linux 3.0)
       This file is a handle for the IPC namespace of the
       process.

/proc/[pid]/ns/mnt (since Linux 3.8)
       This file is a handle for the mount namespace of the
       process.

/proc/[pid]/ns/net (since Linux 3.0)
       This file is a handle for the network namespace of the
       process.

/proc/[pid]/ns/pid (since Linux 3.8)
       This file is a handle for the PID namespace of the
       process.  This handle is permanent for the lifetime of the
       process (i.e., a process's PID namespace membership never
       changes).

/proc/[pid]/ns/pid_for_children (since Linux 4.12)
       This file is a handle for the PID namespace of child
       processes created by this process.  This can change as a
       consequence of calls to unshare(2) and setns(2) (see
       pid_namespaces(7)), so the file may differ from
       /proc/[pid]/ns/pid.  The symbolic link gains a value only
       after the first child process is created in the namespace.
       (Beforehand, readlink(2) of the symbolic link will return
       an empty buffer.)

/proc/[pid]/ns/time (since Linux 5.6)
       This file is a handle for the time namespace of the
       process.

/proc/[pid]/ns/time_for_children (since Linux 5.6)
       This file is a handle for the time namespace of child
       processes created by this process.  This can change as a
       consequence of calls to unshare(2) and setns(2) (see
       time_namespaces(7)), so the file may differ from
       /proc/[pid]/ns/time.

/proc/[pid]/ns/user (since Linux 3.8)
       This file is a handle for the user namespace of the
       process.

/proc/[pid]/ns/uts (since Linux 3.0)
       This file is a handle for the UTS namespace of the
       process.

Permission to dereference or read (readlink(2)) these symbolic
links is governed by a ptrace access mode
PTRACE_MODE_READ_FSCREDS check; see ptrace(2).
```

### The /proc/sys/user directory
```
The files in the /proc/sys/user directory (which is present since
Linux 4.9) expose limits on the number of namespaces of various
types that can be created.  The files are as follows:

max_cgroup_namespaces
       The value in this file defines a per-user limit on the
       number of cgroup namespaces that may be created in the
       user namespace.

max_ipc_namespaces
       The value in this file defines a per-user limit on the
       number of ipc namespaces that may be created in the user
       namespace.

max_mnt_namespaces
       The value in this file defines a per-user limit on the
       number of mount namespaces that may be created in the user
       namespace.

max_net_namespaces
       The value in this file defines a per-user limit on the
       number of network namespaces that may be created in the
       user namespace.

max_pid_namespaces
       The value in this file defines a per-user limit on the
       number of PID namespaces that may be created in the user
       namespace.

max_time_namespaces (since Linux 5.7)
       The value in this file defines a per-user limit on the
       number of time namespaces that may be created in the user
       namespace.

max_user_namespaces
       The value in this file defines a per-user limit on the
       number of user namespaces that may be created in the user
       namespace.

max_uts_namespaces
       The value in this file defines a per-user limit on the
       number of uts namespaces that may be created in the user
       namespace.
```

### Namespace lifetime
```
Absent any other factors, a namespace is automatically torn down
when the last process in the namespace terminates or leaves the
namespace.  However, there are a number of other factors that may
pin a namespace into existence even though it has no member
processes.  These factors include the following:

An open file descriptor or a bind mount exists for the
corresponding /proc/[pid]/ns/ file.

The namespace is hierarchical (i.e., a PID or user namespace),
and has a child namespace.

It is a user namespace that owns one or more nonuser
namespaces.

It is a PID namespace, and there is a process that refers to
the namespace via a /proc/[pid]/ns/pid_for_children symbolic
link.

It is a time namespace, and there is a process that refers to
the namespace via a /proc/[pid]/ns/time_for_children symbolic
link.

It is an IPC namespace, and a corresponding mount of an mqueue
filesystem (see mq_overview(7)) refers to this namespace.

It is a PID namespace, and a corresponding mount of a proc(5)
filesystem refers to this namespace.
```

[namespace man page](https://man7.org/linux/man-pages/man7/namespaces.7.html)
{:.info}

[another doc](https://segmentfault.com/a/1190000016357628)
{:.info}

### Demo
```
# 查看network Namespace
[root@mlxsw-sn2100-01 mycontainer]# ll /var/run/netns/
total 0
-r--r--r-- 1 root root 0 Apr 25 02:29 cni-2ecaf863-139a-d6cd-ce9d-dde9f820fc6e
-r--r--r-- 1 root root 0 Apr 25 04:54 ns1

# 查看进程的Namespace handler
[root@mlxsw-sn2100-01 ~]# ll /proc/self/ns/
total 0
lrwxrwxrwx 1 root root 0 Apr 25 22:13 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Apr 25 22:13 uts -> 'uts:[4026531838]'

# 创建container Namespace handler的符号连接，这样就可以通过ip命令管理netns，（如果需要可以自己创建，podman会自动创建的）
[localhost #] namespace=$(docker inspect — format=’{{ .State.Pid }}’ docker02)
[localhost #] ln -sfT /proc/$namespace/ns/net /var/run/netns/docker02

# 把网卡绑定到容器中
[root@hp-dl380pg8-15 ~]# podman inspect centos | grep cni
            "SandboxKey": "/run/netns/cni-eb6b0df5-0328-958f-9da8-59b6027ec4be",
[root@hp-dl380pg8-15 ~]# ip link set ens2f0v0 netns cni-eb6b0df5-0328-958f-9da8-59b6027ec4be
[root@hp-dl380pg8-15 ~]# podman ps
CONTAINER ID  IMAGE                         COMMAND    CREATED        STATUS            PORTS   NAMES
27dae744e635  quay.io/centos/centos:latest  /bin/bash  3 minutes ago  Up 3 minutes ago          centos
[root@hp-dl380pg8-15 ~]# podman attach centos
[root@27dae744e635 /]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0@if291: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 0e:98:05:0d:1b:e6 brd ff:ff:ff:ff:ff:ff link-netnsid 0
20: ens2f0v0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 00:90:fa:73:47:19 brd ff:ff:ff:ff:ff:ff

# 查看Namespace
[root@mlxsw-sn2100-01 mycontainer]# lsns -o NS,TYPE,PATH,NPROCS,PID,PPID,COMMAND,UID,USER,NETNSID,NSFS
        NS TYPE   PATH              NPROCS   PID  PPID COMMAND                                                            UID USER      NETNSID NSFS
4026531835 cgroup /proc/1/ns/cgroup    138     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root              
4026531836 pid    /proc/1/ns/pid       137     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root              
4026531837 user   /proc/1/ns/user      138     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root              
4026531838 uts    /proc/1/ns/uts       137     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root              
4026531839 ipc    /proc/1/ns/ipc       137     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root              
4026531840 mnt    /proc/1/ns/mnt       132     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root              
4026531860 mnt    /proc/37/ns/mnt        1    37     2 kdevtmpfs                                                            0 root              
4026531992 net    /proc/1/ns/net       137     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root   unassigned 
4026532222 mnt    /proc/788/ns/mnt       1   788     1 /usr/lib/systemd/systemd-udevd                                       0 root              
4026532241 mnt    /proc/911/ns/mnt       1   911     1 /sbin/auditd                                                         0 root              
4026532242 mnt    /proc/939/ns/mnt       1   939     1 /usr/sbin/NetworkManager --no-daemon                                 0 root              
4026532252 mnt    /proc/949/ns/mnt       1   949     1 /usr/sbin/chronyd                                                  995 chrony            
4026532254 net    /proc/1926/ns/net      1  1926  1914 /bin/sh                                                              0 root            0 /run/netns/cni-2ecaf863-139a-d6cd-ce9d-dde9f820fc6e
4026532407 mnt    /proc/1926/ns/mnt      1  1926  1914 /bin/sh                                                              0 root              
4026532408 uts    /proc/1926/ns/uts      1  1926  1914 /bin/sh                                                              0 root              
4026532409 ipc    /proc/1926/ns/ipc      1  1926  1914 /bin/sh                                                              0 root              
4026532410 pid    /proc/1926/ns/pid      1  1926  1914 /bin/sh                                                              0 root              
```

## cgroups

### v1
```
cgroups用来控制进程对资源的访问，比如限制cpu、内存、blockio的使用量。  
其中每一种资源叫一种subsystem或者resource controller。  

v1版本的cgroup包括下面的一些controller：  
       cpu (since Linux 2.6.24; CONFIG_CGROUP_SCHED)
              Cgroups can be guaranteed a minimum number of "CPU shares"
              when a system is busy.  This does not limit a cgroup's CPU
              usage if the CPUs are not busy.  For further information,
              see Documentation/scheduler/sched-design-CFS.rst (or
              Documentation/scheduler/sched-design-CFS.txt in Linux 5.2
              and earlier).

              In Linux 3.2, this controller was extended to provide CPU
              "bandwidth" control.  If the kernel is configured with
              CONFIG_CFS_BANDWIDTH, then within each scheduling period
              (defined via a file in the cgroup directory), it is
              possible to define an upper limit on the CPU time
              allocated to the processes in a cgroup.  This upper limit
              applies even if there is no other competition for the CPU.
              Further information can be found in the kernel source file
              Documentation/scheduler/sched-bwc.rst (or
              Documentation/scheduler/sched-bwc.txt in Linux 5.2 and
              earlier).

       cpuacct (since Linux 2.6.24; CONFIG_CGROUP_CPUACCT)
              This provides accounting for CPU usage by groups of
              processes.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/cpuacct.rst (or
              Documentation/cgroup-v1/cpuacct.txt in Linux 5.2 and
              earlier).

       cpuset (since Linux 2.6.24; CONFIG_CPUSETS)
              This cgroup can be used to bind the processes in a cgroup
              to a specified set of CPUs and NUMA nodes.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/cpusets.rst (or
              Documentation/cgroup-v1/cpusets.txt in Linux 5.2 and
              earlier).

       memory (since Linux 2.6.25; CONFIG_MEMCG)
              The memory controller supports reporting and limiting of
              process memory, kernel memory, and swap used by cgroups.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/memory.rst (or
              Documentation/cgroup-v1/memory.txt in Linux 5.2 and
              earlier).

       devices (since Linux 2.6.26; CONFIG_CGROUP_DEVICE)
              This supports controlling which processes may create
              (mknod) devices as well as open them for reading or
              writing.  The policies may be specified as allow-lists and
              deny-lists.  Hierarchy is enforced, so new rules must not
              violate existing rules for the target or ancestor cgroups.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/devices.rst (or
              Documentation/cgroup-v1/devices.txt in Linux 5.2 and
              earlier).

       freezer (since Linux 2.6.28; CONFIG_CGROUP_FREEZER)
              The freezer cgroup can suspend and restore (resume) all
              processes in a cgroup.  Freezing a cgroup /A also causes
              its children, for example, processes in /A/B, to be
              frozen.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/freezer-subsystem.rst
              (or Documentation/cgroup-v1/freezer-subsystem.txt in Linux
              5.2 and earlier).

       net_cls (since Linux 2.6.29; CONFIG_CGROUP_NET_CLASSID)
              This places a classid, specified for the cgroup, on
              network packets created by a cgroup.  These classids can
              then be used in firewall rules, as well as used to shape
              traffic using tc(8).  This applies only to packets leaving
              the cgroup, not to traffic arriving at the cgroup.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/net_cls.rst (or
              Documentation/cgroup-v1/net_cls.txt in Linux 5.2 and
              earlier).

       blkio (since Linux 2.6.33; CONFIG_BLK_CGROUP)
              The blkio cgroup controls and limits access to specified
              block devices by applying IO control in the form of
              throttling and upper limits against leaf nodes and
              intermediate nodes in the storage hierarchy.

              Two policies are available.  The first is a proportional-
              weight time-based division of disk implemented with CFQ.
              This is in effect for leaf nodes using CFQ.  The second is
              a throttling policy which specifies upper I/O rate limits
              on a device.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/blkio-controller.rst
              (or Documentation/cgroup-v1/blkio-controller.txt in Linux
              5.2 and earlier).

       perf_event (since Linux 2.6.39; CONFIG_CGROUP_PERF)
              This controller allows perf monitoring of the set of
              processes grouped in a cgroup.

              Further information can be found in the kernel source
              files

       net_prio (since Linux 3.3; CONFIG_CGROUP_NET_PRIO)
              This allows priorities to be specified, per network
              interface, for cgroups.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/net_prio.rst (or
              Documentation/cgroup-v1/net_prio.txt in Linux 5.2 and
              earlier).

       hugetlb (since Linux 3.5; CONFIG_CGROUP_HUGETLB)
              This supports limiting the use of huge pages by cgroups.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/hugetlb.rst (or
              Documentation/cgroup-v1/hugetlb.txt in Linux 5.2 and
              earlier).

       pids (since Linux 4.3; CONFIG_CGROUP_PIDS)
              This controller permits limiting the number of process
              that may be created in a cgroup (and its descendants).

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/pids.rst (or
              Documentation/cgroup-v1/pids.txt in Linux 5.2 and
              earlier).

       rdma (since Linux 4.11; CONFIG_CGROUP_RDMA)
              The RDMA controller permits limiting the use of RDMA/IB-
              specific resources per cgroup.

              Further information can be found in the kernel source file
              Documentation/admin-guide/cgroup-v1/rdma.rst (or
              Documentation/cgroup-v1/rdma.txt in Linux 5.2 and
              earlier).

要使用每一种controller都要先把它mount到pseudo-filesystem上。  
v1版本的cgroup，不同的controller可以mount到不同到目录，但是一个controller只能mount到一个目录。  
      mount -t cgroup -o cpu none /sys/fs/cgroup/cpu  
      mount -t cgroup -o cpu,cpuacct none /sys/fs/cgroup/cpu,cpuacct  
```

### v1 Demo
```
# 默认cgroup mount point
[root@hp-dl380pg8-15 ~]# ll /sys/fs/cgroup/
total 0
dr-xr-xr-x 6 root root  0 Apr 21 00:54 blkio
lrwxrwxrwx 1 root root 11 Apr 21 00:54 cpu -> cpu,cpuacct
dr-xr-xr-x 6 root root  0 Apr 21 00:54 cpu,cpuacct
lrwxrwxrwx 1 root root 11 Apr 21 00:54 cpuacct -> cpu,cpuacct
dr-xr-xr-x 3 root root  0 Apr 21 00:54 cpuset
dr-xr-xr-x 6 root root  0 Apr 21 00:54 devices
dr-xr-xr-x 3 root root  0 Apr 21 00:54 freezer
dr-xr-xr-x 3 root root  0 Apr 21 00:54 hugetlb
dr-xr-xr-x 6 root root  0 Apr 21 00:54 memory
lrwxrwxrwx 1 root root 16 Apr 21 00:54 net_cls -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Apr 21 00:54 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Apr 21 00:54 net_prio -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Apr 21 00:54 perf_event
dr-xr-xr-x 6 root root  0 Apr 21 00:54 pids
dr-xr-xr-x 2 root root  0 Apr 21 00:54 rdma
dr-xr-xr-x 6 root root  0 Apr 21 00:54 systemd

[root@hp-dl380pg8-15 ~]# cat /proc/mounts |grep cgroup
tmpfs /sys/fs/cgroup tmpfs ro,nosuid,nodev,noexec,mode=755 0 0
cgroup /sys/fs/cgroup/systemd cgroup rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd 0 0
cgroup /sys/fs/cgroup/pids cgroup rw,nosuid,nodev,noexec,relatime,pids 0 0
cgroup /sys/fs/cgroup/net_cls,net_prio cgroup rw,nosuid,nodev,noexec,relatime,net_cls,net_prio 0 0
cgroup /sys/fs/cgroup/cpuset cgroup rw,nosuid,nodev,noexec,relatime,cpuset 0 0
cgroup /sys/fs/cgroup/hugetlb cgroup rw,nosuid,nodev,noexec,relatime,hugetlb 0 0
cgroup /sys/fs/cgroup/memory cgroup rw,nosuid,nodev,noexec,relatime,memory 0 0
cgroup /sys/fs/cgroup/devices cgroup rw,nosuid,nodev,noexec,relatime,devices 0 0
cgroup /sys/fs/cgroup/blkio cgroup rw,nosuid,nodev,noexec,relatime,blkio 0 0
cgroup /sys/fs/cgroup/rdma cgroup rw,nosuid,nodev,noexec,relatime,rdma 0 0
cgroup /sys/fs/cgroup/cpu,cpuacct cgroup rw,nosuid,nodev,noexec,relatime,cpu,cpuacct 0 0
cgroup /sys/fs/cgroup/freezer cgroup rw,nosuid,nodev,noexec,relatime,freezer 0 0
cgroup /sys/fs/cgroup/perf_event cgroup rw,nosuid,nodev,noexec,relatime,perf_event 0 0

# /sys/fs/cgroup/目录下每一个子目录代表一个controller或者controller的组合。

# 每一个controller下面有相应的配置文件，用来做资源限制
[root@hp-dl380pg8-15 ~]# ll /sys/fs/cgroup/cpu/
total 0
-rw-r--r-- 1 root root 0 Apr 25 23:15 cgroup.clone_children
-rw-r--r-- 1 root root 0 Apr 21 00:54 cgroup.procs
-r--r--r-- 1 root root 0 Apr 25 23:15 cgroup.sane_behavior
-rw-r--r-- 1 root root 0 Apr 21 04:56 cpu.cfs_period_us
-rw-r--r-- 1 root root 0 Apr 21 04:56 cpu.cfs_quota_us
-rw-r--r-- 1 root root 0 Apr 21 04:56 cpu.rt_period_us
-rw-r--r-- 1 root root 0 Apr 21 04:56 cpu.rt_runtime_us
-rw-r--r-- 1 root root 0 Apr 21 04:56 cpu.shares
-r--r--r-- 1 root root 0 Apr 25 23:15 cpu.stat
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.stat
-rw-r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage_all
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage_percpu
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage_percpu_sys
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage_percpu_user
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage_sys
-r--r--r-- 1 root root 0 Apr 25 23:15 cpuacct.usage_user
drwxr-xr-x 2 root root 0 Apr 21 04:55 init.scope
drwxr-xr-x 5 root root 0 Apr 21 04:55 machine.slice
-rw-r--r-- 1 root root 0 Apr 25 23:15 notify_on_release
-rw-r--r-- 1 root root 0 Apr 25 23:15 release_agent
drwxr-xr-x 2 root root 0 Apr 21 04:55 system.slice
-rw-r--r-- 1 root root 0 Apr 25 23:15 tasks
drwxr-xr-x 2 root root 0 Apr 21 04:55 user.slice

# 当你创建一个子目录后，会自动创建配置文件
[root@hp-dl380pg8-15 ~]# mkdir /sys/fs/cgroup/cpu/cpu2
[root@hp-dl380pg8-15 ~]# ll /sys/fs/cgroup/cpu/cpu2
total 0
-rw-r--r-- 1 root root 0 Apr 25 23:21 cgroup.clone_children
-rw-r--r-- 1 root root 0 Apr 25 23:21 cgroup.procs
-rw-r--r-- 1 root root 0 Apr 25 23:21 cpu.cfs_period_us
-rw-r--r-- 1 root root 0 Apr 25 23:21 cpu.cfs_quota_us
-rw-r--r-- 1 root root 0 Apr 25 23:21 cpu.rt_period_us
-rw-r--r-- 1 root root 0 Apr 25 23:21 cpu.rt_runtime_us
-rw-r--r-- 1 root root 0 Apr 25 23:21 cpu.shares
-r--r--r-- 1 root root 0 Apr 25 23:21 cpu.stat
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.stat
-rw-r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage_all
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage_percpu
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage_percpu_sys
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage_percpu_user
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage_sys
-r--r--r-- 1 root root 0 Apr 25 23:21 cpuacct.usage_user
-rw-r--r-- 1 root root 0 Apr 25 23:21 notify_on_release
-rw-r--r-- 1 root root 0 Apr 25 23:21 tasks

# 把进程加入某个cgroup
echo $pid > /sys/fs/cgroup/cpu/cpu2/cgroup.procs

```

### v2
```
v2版本的cgroup，所有的controller都在一个目录中。  
In cgroups v2, all mounted controllers reside in a single unified
       hierarchy.  While (different) controllers may be simultaneously
       mounted under the v1 and v2 hierarchies, it is not possible to
       mount the same controller simultaneously under both the v1 and
       the v2 hierarchies.

一个系统可以同时使用v1和v2，但是一个controller只能在一个版本中使用。  
A cgroup v2 controller is available only if it is not currently
       in use via a mount against a cgroup v1 hierarchy.  Or, to put
       things another way, it is not possible to employ the same
       controller against both a v1 hierarchy and the unified v2
       hierarchy.  This means that it may be necessary first to unmount
       a v1 controller (as described above) before that controller is
       available in v2.  Since systemd(1) makes heavy use of some v1
       controllers by default, it can in some cases be simpler to boot
       the system with selected v1 controllers disabled.  To do this,
       specify the cgroup_no_v1=list option on the kernel boot command
       line; list is a comma-separated list of the names of the
       controllers to disable, or the word all to disable all v1
       controllers.  (This situation is correctly handled by systemd(1),
       which falls back to operating without the specified controllers.)

       Note that on many modern systems, systemd(1) automatically mounts
       the cgroup2 filesystem at /sys/fs/cgroup/unified during the boot
       process.

       cpu (since Linux 4.15)
              This is the successor to the version 1 cpu and cpuacct
              controllers.

       cpuset (since Linux 5.0)
              This is the successor of the version 1 cpuset controller.

       freezer (since Linux 5.2)
              This is the successor of the version 1 freezer controller.

       hugetlb (since Linux 5.6)
              This is the successor of the version 1 hugetlb controller.

       io (since Linux 4.5)
              This is the successor of the version 1 blkio controller.

       memory (since Linux 4.5)
              This is the successor of the version 1 memory controller.

       perf_event (since Linux 4.11)
              This is the same as the version 1 perf_event controller.

       pids (since Linux 4.5)
              This is the same as the version 1 pids controller.

       rdma (since Linux 4.11)
              This is the same as the version 1 rdma controller.

       There is no direct equivalent of the net_cls and net_prio
       controllers from cgroups version 1.  Instead, support has been
       added to iptables(8) to allow eBPF filters that hook on cgroup v2
       pathnames to make decisions about network traffic on a per-cgroup
       basis.

       The v2 devices controller provides no interface files; instead,
       device control is gated by attaching an eBPF (BPF_CGROUP_DEVICE)
       program to a v2 cgroup.

mount -t cgroup2 none /mnt/cgroup2

```

### files
```
/proc files
    /proc/cgroups (since Linux 2.6.24)
           This file contains information about the controllers that
           are compiled into the kernel.  An example of the contents
           of this file (reformatted for readability) is the
           following:

               #subsys_name    hierarchy      num_cgroups    enabled
               cpuset          4              1              1
               cpu             8              1              1
               cpuacct         8              1              1
               blkio           6              1              1
               memory          3              1              1
               devices         10             84             1
               freezer         7              1              1
               net_cls         9              1              1
               perf_event      5              1              1
               net_prio        9              1              1
               hugetlb         0              1              0
               pids            2              1              1

           The fields in this file are, from left to right:

           1. The name of the controller.

           2. The unique ID of the cgroup hierarchy on which this
              controller is mounted.  If multiple cgroups v1
              controllers are bound to the same hierarchy, then each
              will show the same hierarchy ID in this field.  The
              value in this field will be 0 if:

                a) the controller is not mounted on a cgroups v1
                   hierarchy;

                b) the controller is bound to the cgroups v2 single
                   unified hierarchy; or

                c) the controller is disabled (see below).

           3. The number of control groups in this hierarchy using
              this controller.

           4. This field contains the value 1 if this controller is
              enabled, or 0 if it has been disabled (via the
              cgroup_disable kernel command-line boot parameter).

    /proc/[pid]/cgroup (since Linux 2.6.24)
           This file describes control groups to which the process
           with the corresponding PID belongs.  The displayed
           information differs for cgroups version 1 and version 2
           hierarchies.

           For each cgroup hierarchy of which the process is a
           member, there is one entry containing three colon-
           separated fields:

               hierarchy-ID:controller-list:cgroup-path

           For example:

               5:cpuacct,cpu,cpuset:/daemons

           The colon-separated fields are, from left to right:

           1. For cgroups version 1 hierarchies, this field contains
              a unique hierarchy ID number that can be matched to a
              hierarchy ID in /proc/cgroups.  For the cgroups version
              2 hierarchy, this field contains the value 0.

           2. For cgroups version 1 hierarchies, this field contains
              a comma-separated list of the controllers bound to the
              hierarchy.  For the cgroups version 2 hierarchy, this
              field is empty.

           3. This field contains the pathname of the control group
              in the hierarchy to which the process belongs.  This
              pathname is relative to the mount point of the
              hierarchy.

/sys/kernel/cgroup files
    /sys/kernel/cgroup/delegate (since Linux 4.15)
           This file exports a list of the cgroups v2 files (one per
           line) that are delegatable (i.e., whose ownership should
           be changed to the user ID of the delegatee).  In the
           future, the set of delegatable files may change or grow,
           and this file provides a way for the kernel to inform
           user-space applications of which files must be delegated.
           As at Linux 4.15, one sees the following when inspecting
           this file:

               $ cat /sys/kernel/cgroup/delegate
               cgroup.procs
               cgroup.subtree_control
               cgroup.threads

    /sys/kernel/cgroup/features (since Linux 4.15)
           Over time, the set of cgroups v2 features that are
           provided by the kernel may change or grow, or some
           features may not be enabled by default.  This file
           provides a way for user-space applications to discover
           what features the running kernel supports and has enabled.
           Features are listed one per line:

               $ cat /sys/kernel/cgroup/features
               nsdelegate
               memory_localevents

           The entries that can appear in this file are:

           memory_localevents (since Linux 5.2)
                  The kernel supports the memory_localevents mount
                  option.

           nsdelegate (since Linux 4.15)
                  The kernel supports the nsdelegate mount option.
```

[cgroups man page](https://man7.org/linux/man-pages/man7/cgroups.7.html)
{:.info}

## OCI & CRI

OCI是容器标准
```
The Open Container Initiative is an open governance structure for the express purpose of creating open industry standards around container formats and runtimes.

Established in June 2015 by Docker and other leaders in the container industry, the OCI currently contains two specifications: 
the Runtime Specification (runtime-spec) and the Image Specification (image-spec). 
The Runtime Specification outlines how to run a “filesystem bundle” that is unpacked on disk. 
At a high-level an OCI implementation would download an OCI Image then unpack that image into an OCI Runtime filesystem bundle. 
At this point the OCI Runtime Bundle would be run by an OCI Runtime.
```

CRI是Kubernetes的容器运行时接口
```
Each container runtime has it own strengths, and many users have asked for Kubernetes to support more runtimes. 
In the Kubernetes 1.5 release, we are proud to introduce the Container Runtime Interface (CRI) -- a plugin interface which enables kubelet to use a wide variety of container runtimes, 
without the need to recompile. 
```

## runc

[runc](https://www.docker.com/blog/runc/)

runc是基于OCI标准的一种容器运行时，与Namespace，cgroups打交道。
```
Docker is a platform to build, ship and rundistributed applications – meaning that it runs applications in a distributed fashion across many machines, often with a variety of hardware and OS configurations. For this to be possible, it needs a sandboxing environment capable of abstracting the specifics of the underlying host (for portability), without requiring a complete rewrite of the application (for ubiquity), and without introducing excessive performance overhead (for scale).

Over the last 5 years Linux has gradually gained a collection of features which make this kind of abstraction possible. Windows, with its upcoming version 10, is adding similar features as well. Those individual features have esoteric names like “control groups”, “namespaces”, “seccomp”, “capabilities”, “apparmor” and so on. But collectively, they are known as “OS containers” or sometimes “lightweight virtualization”.

Docker makes heavy use of these features and has become famous for it. Because “containers” are actually an array of complicated, sometimes arcane system features, we have integrated them into a unified low-level component which we simply call runC. And today we are spinning out runC as a standalone tool, to be used as plumbing by infrastructure plumbers everywhere.

runC is a lightweight, portable container runtime. It includes all of the plumbing code used by Docker to interact with system features related to containers. It is designed with the following principles in mind:

• Designed for security.
• Usable at large scale, in production, today.
• No dependency on the rest of the Docker platform: just the container runtime and nothing else.

Popular runC features include:

• Full support for Linux namespaces, including user namespaces
• Native support for all security features available in Linux: Selinux, Apparmor, seccomp, control groups, capability drop, pivot_root, uid/gid dropping etc. If Linux can do it, runC can do it.
• Native support for live migration, with the help of the CRIU team at Parallels
• Native support of Windows 10 containers is being contributed directly by Microsoft engineers
• Planned native support for Arm, Power, Sparc with direct participation and support from Arm, Intel, Qualcomm, IBM, and the entire hardware manufacturers ecosystem.
• Planned native support for bleeding edge hardware features – DPDK, sr-iov, tpm, secure enclave, etc.
• Portable performance profiles, contributed by Google engineers based on their experience deploying containers in production.
• A formally specified configuration format, governed by the Open Container Project  under the auspices ofthe Linux Foundation. In other words: it’s a real standard.

The goal of runC is to make standard containers available everywhere

In fact, we have decided to donate the code of runC itself to the OCP foundation. Because OCP is designed to work just like the Linux Foundation, we expect that the maintainers – a blend of employees from various container-focused companies and hobbyists – will be largely left alone, and will continue to write the most awesome software possible.

runC is available today and is already under active development. Because it is based on the battle-tested plumbing used by Docker, you can use it in production today, either as part of a Docker deployment or in your own custom platform. We look forward to your contributions!
```

### deploy container with runc
```
dnf install -y golang libseccomp-devel

git clone https://github.com/opencontainers/runc
cd runc
make
sudo make install

# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -

runc spec
```

```
[root@mlxsw-sn2100-01 ~]# cat /mycontainer/config.json
{
	"ociVersion": "1.0.2-dev",
	"process": {
		"terminal": true,
		"user": {
			"uid": 0,
			"gid": 0
		},
		"args": [
			"sh"
		],
		"env": [
			"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
			"TERM=xterm"
		],
		"cwd": "/",
		"capabilities": {
			"bounding": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE",
				"CAP_NET_ADMIN",
				"CAP_NET_BROADCAST",
				"CAP_NET_RAW"
			],
			"effective": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE",
				"CAP_NET_ADMIN",
				"CAP_NET_BROADCAST",
				"CAP_NET_RAW"
			],
			"inheritable": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE",
				"CAP_NET_ADMIN",
				"CAP_NET_BROADCAST",
				"CAP_NET_RAW"
			],
			"permitted": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE",
				"CAP_NET_ADMIN",
				"CAP_NET_BROADCAST",
				"CAP_NET_RAW"
			],
			"ambient": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE",
				"CAP_NET_ADMIN",
				"CAP_NET_BROADCAST",
				"CAP_NET_RAW"
			]
		},
		"rlimits": [
			{
				"type": "RLIMIT_NOFILE",
				"hard": 1024,
				"soft": 1024
			}
		],
		"noNewPrivileges": true
	},
	"root": {
		"path": "rootfs",
		"readonly": true
	},
	"hostname": "runc",
	"mounts": [
		{
			"destination": "/proc",
			"type": "proc",
			"source": "proc"
		},
		{
			"destination": "/dev",
			"type": "tmpfs",
			"source": "tmpfs",
			"options": [
				"nosuid",
				"strictatime",
				"mode=755",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/pts",
			"type": "devpts",
			"source": "devpts",
			"options": [
				"nosuid",
				"noexec",
				"newinstance",
				"ptmxmode=0666",
				"mode=0620",
				"gid=5"
			]
		},
		{
			"destination": "/dev/shm",
			"type": "tmpfs",
			"source": "shm",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"mode=1777",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/mqueue",
			"type": "mqueue",
			"source": "mqueue",
			"options": [
				"nosuid",
				"noexec",
				"nodev"
			]
		},
		{
			"destination": "/sys",
			"type": "sysfs",
			"source": "sysfs",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"ro"
			]
		},
		{
			"destination": "/sys/fs/cgroup",
			"type": "cgroup",
			"source": "cgroup",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"relatime",
				"ro"
			]
		}
	],
	"linux": {
		"resources": {
			"devices": [
				{
					"allow": false,
					"access": "rwm"
				}
			]
		},
		"namespaces": [
			{
				"type": "pid"
			},
			{
				"type": "network",
				"path": "/var/run/netns/ns1"
			},
			{
				"type": "ipc"
			},
			{
				"type": "uts"
			},
			{
				"type": "mount"
			}
		],
		"maskedPaths": [
			"/proc/acpi",
			"/proc/asound",
			"/proc/kcore",
			"/proc/keys",
			"/proc/latency_stats",
			"/proc/timer_list",
			"/proc/timer_stats",
			"/proc/sched_debug",
			"/sys/firmware",
			"/proc/scsi"
		],
		"readonlyPaths": [
			"/proc/bus",
			"/proc/fs",
			"/proc/irq",
			"/proc/sys",
			"/proc/sysrq-trigger"
		]
	}
}

把container绑定到network Namespace /var/run/netns/ns1 之后，
就可以把网卡加入container
# ip netns add ns1
# ip link set $nic netns ns1
# 在container里面可以看到网卡了
[root@mlxsw-sn2100-01 mycontainer]# runc run liali
/ # ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop qlen 1000
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
23: veth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue qlen 1000
    link/ether 3a:41:f5:1d:42:8a brd ff:ff:ff:ff:ff:ff
/ # ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop qlen 1000
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
23: veth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue qlen 1000
    link/ether 3a:41:f5:1d:42:8a brd ff:ff:ff:ff:ff:ff
    inet 199.111.1.1/24 scope global veth0
       valid_lft forever preferred_lft forever
    inet6 fe80::3841:f5ff:fe1d:428a/64 scope link
       valid_lft forever preferred_lft forever

# 需要赋予container CAP_NET_ADMIN权限才能ping
/ # ping 199.111.1.2
PING 199.111.1.2 (199.111.1.2): 56 data bytes
64 bytes from 199.111.1.2: seq=0 ttl=64 time=0.090 ms
64 bytes from 199.111.1.2: seq=1 ttl=64 time=0.073 ms
^C
--- 199.111.1.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.073/0.081/0.090 ms

# 此时可以看到ns1是和container进程关联的（container在使用ns1这个network Namespace）
[root@mlxsw-sn2100-01 mycontainer]# lsns -o NS,TYPE,PATH,NPROCS,PID,PPID,COMMAND,UID,USER,NETNSID,NSFS
        NS TYPE   PATH               NPROCS   PID  PPID COMMAND                                                            UID USER      NETNSID NSFS
4026531835 cgroup /proc/1/ns/cgroup     141     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root
4026531836 pid    /proc/1/ns/pid        139     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root
4026531837 user   /proc/1/ns/user       141     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root
4026531838 uts    /proc/1/ns/uts        139     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root
4026531839 ipc    /proc/1/ns/ipc        139     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root
4026531840 mnt    /proc/1/ns/mnt        134     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root
4026531860 mnt    /proc/37/ns/mnt         1    37     2 kdevtmpfs                                                            0 root
4026531992 net    /proc/1/ns/net        139     1     0 /usr/lib/systemd/systemd --switched-root --system --deserialize 17   0 root   unassigned
4026532222 mnt    /proc/788/ns/mnt        1   788     1 /usr/lib/systemd/systemd-udevd                                       0 root
4026532241 mnt    /proc/911/ns/mnt        1   911     1 /sbin/auditd                                                         0 root
4026532242 mnt    /proc/939/ns/mnt        1   939     1 /usr/sbin/NetworkManager --no-daemon                                 0 root
4026532252 mnt    /proc/949/ns/mnt        1   949     1 /usr/sbin/chronyd                                                  995 chrony
4026532254 net    /proc/1926/ns/net       1  1926  1914 /bin/sh                                                              0 root            0 /run/netns/cni-2ecaf863-139a-d6cd-ce9d-dde9f820fc6e
4026532407 mnt    /proc/1926/ns/mnt       1  1926  1914 /bin/sh                                                              0 root
4026532408 uts    /proc/1926/ns/uts       1  1926  1914 /bin/sh                                                              0 root
4026532409 ipc    /proc/1926/ns/ipc       1  1926  1914 /bin/sh                                                              0 root
4026532410 pid    /proc/1926/ns/pid       1  1926  1914 /bin/sh                                                              0 root
4026532412 net    /proc/24416/ns/net      1 24416 24405 sh                                                                   0 root            1 /run/netns/ns1
4026532480 mnt    /proc/24416/ns/mnt      1 24416 24405 sh                                                                   0 root
4026532481 uts    /proc/24416/ns/uts      1 24416 24405 sh                                                                   0 root
4026532482 ipc    /proc/24416/ns/ipc      1 24416 24405 sh                                                                   0 root
4026532483 pid    /proc/24416/ns/pid      1 24416 24405 sh                                                                   0 root
[root@mlxsw-sn2100-01 mycontainer]# ps 24416
    PID TTY      STAT   TIME COMMAND
  24416 pts/0    Ss+    0:00 sh
[root@mlxsw-sn2100-01 mycontainer]# ps 24405
    PID TTY      STAT   TIME COMMAND
  24405 pts/1    Sl+    0:00 runc run liali
[root@mlxsw-sn2100-01 mycontainer]# ll /proc/24416/ns/net 
lrwxrwxrwx 1 root root 0 Apr 26 02:07 /proc/24416/ns/net -> 'net:[4026532412]'

# 再看一下进程的cgroup, 发现再每个cgroup controller目录下都为container liali创建了一个子目录
[root@mlxsw-sn2100-01 mycontainer]# cat /proc/24416/cgroup
12:memory:/user.slice/user-0.slice/session-5.scope/liali
11:perf_event:/liali
10:cpu,cpuacct:/user.slice/liali
9:devices:/user.slice/liali
8:blkio:/user.slice/liali
7:pids:/user.slice/user-0.slice/session-5.scope/liali
6:rdma:/
5:freezer:/liali
4:hugetlb:/liali
3:net_cls,net_prio:/liali
2:cpuset:/liali
1:name=systemd:/user.slice/user-0.slice/session-5.scope/liali
[root@mlxsw-sn2100-01 mycontainer]# cat /proc/24405/cgroup
12:memory:/user.slice/user-0.slice/session-5.scope
11:perf_event:/
10:cpu,cpuacct:/user.slice
9:devices:/user.slice
8:blkio:/user.slice
7:pids:/user.slice/user-0.slice/session-5.scope
6:rdma:/
5:freezer:/
4:hugetlb:/
3:net_cls,net_prio:/
2:cpuset:/
1:name=systemd:/user.slice/user-0.slice/session-5.scope

```

## podman

```
Podman is a daemonless, open source, Linux native tool designed to make it easy to find, run, build, share and deploy 
applications using Open Containers Initiative (OCI) Containers and Container Images. 
Podman provides a command line interface (CLI) familiar to anyone who has used the Docker Container Engine. 
Most users can simply alias Docker to Podman (alias docker=podman) without any problems. 
Similar to other common Container Engines (Docker, CRI-O, containerd), Podman relies on an OCI compliant Container Runtime (runc, crun, runv, etc) 
to interface with the operating system and create the running containers. 
This makes the running containers created by Podman nearly indistinguishable from those created by any other common container engine.
```

```
yum install -y jq podman
podman login quay.io
podman pull quay.io/liali/do180-todonodejs
podman tag do180-todonodejs localhost/do180-todonodejs
podman build -t localhost/apache .
podman build -t localhost/httpd .
podman run -p 8082:8080 httpd
podman stop 1a398d79818fe3256c86f2cfe42d0754fe420b43892e437d95c0e4200725c121
podman container list
podman image list
podman ps
podman ps -a
podman inspect ad2f156de35e
podman inspect ad2f156de35e | grep IPAddress\":
podman ps ad2f156de35e
podman top ad2f156de35e
podman login
podman inspect --format='{{ .State.Pid }}' centos
podman network create -d macvlan -o parent=enp1s0np7 macvlan1
podman network connect macvlan1 centos
podman container attach 591d854d2ed7
podman container exec -it c98ac2f2cf07 pwd
podman container exec -it -u root c98ac2f2cf07 python3 -V
podman run -it --name centos centos
podman attach test1
podman rm -f centos
podman inspect 5a4cc57fa466|grep -i cgroup
podman export $(podman create busybox) | tar -C rootfs -xvf -
podman exec d8bac1691c99 top
```
[podman doc](http://docs.podman.io/en/latest/Introduction.html)  
[dockerfile reference](https://docs.docker.com/engine/reference/builder/#entrypoint)  
[dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)  
[create you own base image](https://github.com/moby/moby/tree/master/contrib)  
[create you own base image](https://docs.docker.com/develop/develop-images/baseimages/)  
