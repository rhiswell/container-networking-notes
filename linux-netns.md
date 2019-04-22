# Linux Network Namespaces

## Overview on design and implementation

关于 Linux namespaces 和 cgroups 的实现细节可以参见 Rami Rosen 分享的 [slide](http://www.haifux.org/lectures/299/netLec7.pdf)。根据 Rosen 的介绍，namespaces 是一种轻量级的进程虚拟化方案（lightweight process virtualization）。Namespace 提供的隔离性（isolation）enable a process (or serveral processes) to have different views of the system than other processes。相比 OS virtualization 如 KVM、Xen，Linux namespace 省去了 hypervisor layer，所以很 lightweight。

目前，Linux namespaces 包括 mnt (mount points, filesystems) / pid (processes) / net (network stack) / ipc (System V IPC) / uts (hostname) / user (UIDs)。关于实现，Linux 添加了若干个 CLONE_NEW* flags，并结合三个系统调用 clone() / unshare() / setns() 实现。CLONE_NEW* flags 包括 CLONE_NEWNS / CLONE_NEWUTS / CLONE_NEWIPC / CLONE_NEWPID / CLONE_NEWNET / CLONE_NEWUSER。另外，三个系统调用的区别如下：

- `clone()` - 创建一个新 process 和一些新的 namespaces，且将该进程加入该 namespaces 中。进程创建和进程终止方法 `fork()` 和 `exit()` 方法也被 patched 以处理 namespace 相关的 CLONE_NEW * 标志。
- `unshare()` - 创建新的 namespaces，然后将当前进程加入该 namespaces 中。
- `setns()` - 这是一个新的系统调用，让当前进程加入指定的 namespaces 中。

需要注意的是，通过 `clone()` 创建新的进程和新的 namespaces 时，并没有指定 namespaces 的名字（nameless namespaces），那么系统如何知道两个进程是否在同一个 namespace 中？系统在创建 namespace 时会为其分配一个全局 inode 并链接到 `/proc/<pid>/ns` 目录下（每一个 namespace 对应一个 entry），所以通过对比两个进程 `/proc/<pid>/ns` 目录下 namespace 的 inode 值就可以确定两个进程是否在同一个 namespace 中。

```bash
$ sudo ls -l /proc/15702/ns/
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 net -> 'net:[4026532001]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 user -> 'user:[4026531837]'
lrwxrwxrwx 1 rh rh 0 Apr 22 16:20 uts -> 'uts:[4026531838]'
```

Linux 在进程描述符 task_struct 中添加了新成员 `nsproxy`，并通过函数 `task_nsproxy(struct task_struct *tsk)` 访问指定进程所在的 namespaces。

Linux 提供了一些用户程序来操作 Linux namespaces，包括：iproute2 中的 `ip netns` 用于操作 network namespace、util-linux、`unshare` for all the six namespaces 和 `nsenter`, a wrapper around `setns()`, to join a specific namespace。

## Network namespace

逻辑上，一个 network namespace 就是 network stack 的一份拷贝，包括独立的 routes、firewall rules 和 network devices。其在内核中的描述符号为 `struct net`，属性包括：loopback device、SNMP stats (netns_mib)、all network tables (routing / neighboring / etc.)、all sockets、/procfs 和 /sysfs entries。 

A network device belongs to exactly one network namespace and also a socket. All network namespaces are inserted into a system wide linked list named `net_namespace_list`. 最初的 network namespace 包含了所有的网络设备，包括一个 loopback 设备、所有的物理设备和 networking tables 等等。而新创建的 network namespace 只会包含一个 loopback 设备，不包含其它资源（e.g. sockets）。

下面通过一些 examples 说明 network namespace 的行为。

```bash
# Create two namespaces, called "myns1" and "myns2"
$ ip netns add myns1
$ ip netns add myns2
```

This triggers:

- creation of `/var/run/netns/myns1`, `/var/run/netns/myns2` empty folders.
- calling the `unshare()` system call with CLONE_NEWNET. The flag has the same effect as the `clone()` CLONE_NEWNET flag. Unshare the network namespace, so that the calling process is moved into a new network namespace which is not shared with any previously existing process. Use of CLONE_NEWNET requires the CAP_SYS_ADMIN capability. 需要注意的是，因为创建 myns[1-2] 的程序，i.e. `ip netns add ...`，创建完 network namespace 后就拜拜了，其 `/proc/<pid>/` 也就没了，所以需要另外安排一个地方来保存新建的 netns，i.e. `/var/run/netns/`。

```bash
# Delete a namespace by
$ ip netns del myns1
```

This will unmounts and removes `/var/run/netns/myns1`. However, it won't delete a network namespace if there is one or more processes attached to it.

```bash
# List created network namespaces
$ ip netns list

# Assign vxlan interface to myns1
$ ip link add myvlan type vxlan id 233
$ ip link set myvxlan netns myns1

# Find the pid (or list of pids) in a specified netns
$ ip netns pids <ns_name>
# Find the netns of a specified pid, however, it cannot display a nameless netns
$ ip netns identify <pid>
# Monitor addition/removal of netns
$ ip netns monitor
# Run command in a specific netns
$ ip netns exec myns1 bash
# Or
# \note nsenter use `setns` to join current process to a network namespace specified by
# fd.
$ nsenter --net=/var/run/netns/myns1 bash
# Or directly run in a new netns and will be auto removed when exit.
# \note Unlike `ip netns exec ...`, when use `unshare` to run a new program, 
# it wont create an entry in /var/run/netns/ and will be removed when caller exits.
$ unshare --net bash
```

## Refs

- Resource management: Linux kernel Namespace and cgroups. http://www.haifux.org/lectures/299/netLec7.pdf.