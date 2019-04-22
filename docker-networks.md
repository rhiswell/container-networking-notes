# Docker Networks

[TOC]

## Docker concepts

Docker 提供了一个统一的平台来开发、携带和运行应用程序。Docker 允许我们将应用程序和底层平台设施分开，从而提升软件的交付速度。Balala。See [Docker overview](https://docs.docker.com/engine/docker-overview/) for more details.

### Docker Engine

Docker Engine is a client-server application with these major components: dockerd, REST API and CLI.

![Docker Engine Components Flow](assets/engine-components-flow.png)

### What can I use Docker for?

#### Fast, consistent delivery of your applications

Docker streamlines the development lifecycle by allowing developers to work in standardized environments using local containers which provide your applications and services. Containers are great for continuous integration and continuous delivery (CI/CD) workflows.

Consider the following example scenario:

- Your developers write code locally and share their work with their colleagues using Docker containers.
- They use Docker to push their applications into a test environment and execute automated and manual tests.
- When developers find bugs, they can fix them in the development environment and redeploy them to the test environment for testing and validation.
- When testing is complete, getting the fix to the customer is as simple as pushing the updated image to the production environment.

#### Responsive deployment and scaling

Docker’s container-based platform allows for highly portable workloads. Docker containers can run on a developer’s local laptop, on physical or virtual machines in a data center, on cloud providers, or in a mixture of environments.

Docker’s **portability** and **lightweight** nature also make it easy to dynamically manage workloads, scaling up or tearing down applications and services as business needs dictate, in near real time.

#### Running more workloads on the same hardware

Docker is **lightweight** and fast. It provides a viable, cost-effective alternative to hypervisor-based virtual machines, so you can use more of your compute capacity to achieve your business goals. Docker is perfect for high density environments and for small and medium deployments where you need to do more with fewer resources.

### The underlying technology

Docker 是用 Go 写的，同时基于 Linux kernel 的许多特性构建了其功能。

#### Namespaces

Docker 使用一种叫 namespaces 的技术来提供隔离的环境（isolated environment or isolation），即容器（container）。当你创建一个 container，Docker 为其创建了一系列的 namespaces，从而限定容器中的应用只能访问这些 namespaces 中的资源。

目前，Docker Engine 使用了 Linux namespaces 中的五种：

- The pid namespace: Process isolation.
- The net namespace: Manaing network interfaces.
- The ipc namespace: Managing access to IPC resources.
- The mnt namespace: Managing filesystem mount points.
- The uts namespace: Isolating kernel and version identifiers. (UTS: Unix Timesharing System).

其中，network namespaces 用于隔离和 networking 相关的资源，包括 network devices、IPv4 and IPv6 protocol stacks、IP routing tables、firewall rules、the `/proc/net` directory (which is a symbolic link to `/proc/PID/net`)、the `/sys/class/net` directory、various files under `/proc/sys/net` 和 port numbers（sockets）等等。另外，network namespaces 还隔离了 UNIX domain abstract socket namespace。

一个物理网络设备只能出现在一个 network namespace 中。如果某物理网络设备所在的 network namespace 被释放了，则该设备将被移回最初的 network namespace，而不是归还给父进程。

veth 设备用于连接两个不同的 network namespaces，如桥接处在不同 network namespace 中的物理网络设备 。当一个 namespace 被释放后，其拥有的 veth 设备将被销毁。

#### Control groups

Docker Engine 依赖的另一种技术叫 control groups (`cgroups`)。一个 `cgroup` 用于限制应用可用的资源。借助 `cgroups`，Docker Engine 可以选择性地限制 container 的硬件资源，如限定某 container 的内存使用量。

#### Union file systems

Union file systems, or UnionFS, are file systems that operate by creating layers, making them very lightweight and fast. Docker Engine uses UnionFS to provide the building blocks for containers. Docker Engine can use multiple UnionFS variants, including AUFS, btrfs, vfs, and DeviceMapper.

#### Container format

Docker Engine combines the namespaces, control groups, and UnionFS into a wrapper called a container format. The default container format is `libcontainer`. In the future, Docker may support other container formats by integrating with technologies such as BSD Jails or Solaris Zones.

## Swarm mode concetps

See [Swarm mode key concepts](https://docs.docker.com/engine/swarm/key-concepts/#/services-and-tasks) for more details.

## Learn network deployment models by example

See [Tutorial Application: The Pets APP](https://github.com/docker/labs/blob/master/networking/tutorials.md#overlayarch) for more details.

文中涉及 single/multi-host bridge driver deployment、overlay driver  deployment 和 MACVLAN bridge mode，共四种部署方式。其中，相比 bridge，MACVLAN 直接在网络接口上虚拟出若干个 L2 设备，然后 host、containers 分别绑定一个虚拟网络接口来接入网络。因为 MACVLAN 省去了 bridge 设备，所以性能更好，但是丧失了一定的 portability。

![Linux Bridge](assets/linux-bridge.png)

![Linux macvlan](assets/linux-macvlan.png)

需要注意的是，docker bridge mode 和 Linux bridge 不是同一个概念。Docker bridge mode 是 Docker network 的一种，基于 NAT 和 Linux bridge 实现。所以，在 Docker 中，我们无法通过 bridge mode 让 host 和 containers 处于同一个网段，而 macvlan 可以。

![e4](assets/e4.png)

![e5](assets/e5.png)

## Operate Docker networking

```bash
$ docker network ls/inspect/...
# Show supported network plugins
$ docker info 
```

## Challenges of networking containers and microservices

Docker has developed a new way of delivering applications, and with that, containers have also changed some aspects of how we approach networking. The following topics are common design themes for containerized applications:

- **Portability**
  - *How do I guarantee maximum portability across diverse network environments while taking advantage of unique network characteristics?*
- **Service Discovery**
  - *How do I know where services are living as they are scaled up and down?*
- **Load Balancing**
  - *How do I share load across services as services themselves are brought up and scaled?*
- **Security**
  - *How do I segment to prevent the right containers from accessing each other?*
  - *How do I guarantee that a container with application and cluster control traffic is secure?*
- **Performance**
  - *How do I provide advanced network services while minimizing latency and maximizing bandwidth?*
- **Scalability**
  - *How do I ensure that none of these characteristics are sacrificed when scaling applications across many hosts?*

## The container networking model

![Container Networking Model](assets/cnm.png)

Docker 的网络架构构建在一系列的 interfaces 之上，我们称这些 interfaces 的集合为 Container Networking Model (CNM)。CNM 的原则是在不同的基础设施之上为应用提供 portability。

CNM 中有一些高层的抽象，不依赖于 OS 和基础设施，所以应用无需考虑底层的软件栈。包括：

- **Sandbox** — A Sandbox contains the configuration of a container's network stack. This includes management of the container's interfaces, routing table, and DNS settings. An implementation of a Sandbox could be a Linux Network Namespace, a FreeBSD Jail, or other similar concept. A Sandbox may contain many endpoints from multiple networks.
- **Endpoint** — An Endpoint joins a Sandbox to a Network. The Endpoint construct exists so the actual connection to the network can be abstracted away from the application. This helps maintain portability so that a service can use different types of network drivers without being concerned with how it's connected to that network.
- **Network** — The CNM does not specify a Network in terms of the OSI model. An implementation of a Network could be a Linux bridge, a VLAN, etc. A Network is a collection of endpoints that have connectivity between them. Endpoints that are not connected to a network will not have connectivity on a Network.

## Linux network fundamentals

Linux kernel 已经包含了一个成熟且高效的 TCP/IP 协议栈，包括 DNS 和 VXLAN。Docker networking 在这些基础上（low level primitives）构建了高层的 network drivers。简而言之，Docker 网络就是 Linux 网络。

现有 Linux 内核功能的这种实现确保了高性能和健壮性。 最重要的是，它提供了多发行版间的可移植性，从而增强了应用程序的可移植性。

Docker 使用若干 Linux 网络模块来实现其内置的 CNM 网络驱动程序，包括 Linux bridge，network namespaces，veth pair 和 iptables。The combination of these tools implemented as network drivers provide the forwarding rules, network segmentation, and management tools for complex network policy.

### bridge

Linux 网桥是 L2 设备，它是在 Linux 内核中实现的虚拟交换机。它根据通过检查流量动态学习 MAC 地址从而进行流量转发。Linux bridge 广泛用于 Docker 网络驱动程序中。Linux 桥不应与 Docker bridge network 驱动程序混淆，后者是 Linux bridge 的高层实现。

### netns

Linux network namepsace 是内核中独立的网络协议栈，包括独立的接口（interfaces）、路由（routes）和防火墙规则（firewall rules）。 它属于容器和 Linux 安全方面之一，用于隔离容器。 在网络术语中，它们类似于 VRF，它将主机内的 network control plane 和 data plane 分离。Network namespace 确保同一主机上的两个容器无法相互通信，甚至无法与主机本身通信，除非通过 Docker 网络进行配置。通常，CNM 网络驱动程序为每个容器实现单独的命名空间。但是，容器可以共享相同的网络命名空间，甚至可以是主机网络命名空间的一部分。主机网络命名空间包含主机接口和主机路由表。此网络命名空间称为全局网络命名空间。

### veth

虚拟以太网设备或 veth 是一个 Linux 网络接口，充当两个网络命名空间之间的连接线。 veth 是一个全双工链接，每个命名空间中都有一个接口。 一个接口中的流量被引导出另一个接口。Docker 网络驱动程序利用 veths 在创建Docker网络时提供名称空间之间的显式连接。当容器连接到 Docker 网络时，veth 的一端放在容器内（通常被视为 ethX 接口），而另一端连接到 Docker 网络。

### iptables

iptables 是本机包过滤系统，自 2.4 版本以来一直是 Linux 内核的一部分。它是一个功能丰富的 L3 / L4 防火墙，通过 rule chains 为数据包提供 marking，masquerading 和 dropping 操作。内置的 Docker 网络驱动程序广泛使用 iptables 来分割网络流量，提供主机端口映射，并标记流量以实现负载平衡决策。

## Overlay driver network architecture

内置的 Docker overlay network driver 从根本上简化了多主机网络中的许多挑战。 With the overlay driver, multi-host networks are first-class citizens inside Docker without external provisioning or components. 在大规模的集群中，Overlay 使用 Swarm 分布式控制提供 centralized management，stability 和 security。

### VXLAN data plane

覆盖驱动程序使用行业标准的 VXLAN 数据平面，将容器网络与底层物理网络（底层）分离。Docker overlay network 将容器流量封装在 VXLAN 标头中，允许流量穿过物理 L2 或者 L3 网络。无论底层物理拓扑如何，overlay 使得网络分段可动态变化且易于控制。 使用标准 IETF VXLAN 标头可促进标准工具检查和分析网络流量。

自 3.7 版本以来，VXLAN 一直是 Linux 内核的一部分，而 Docker 使用内核的本机 VXLAN 功能来创建 overlay network。Docker overlay 数据路径完全在内核空间中。 这样可以减少上下文切换，减少CPU开销，并在应用程序和物理 NIC 之间实现低延迟和直接的流量路径。

IETF VXLAN（RFC 7348）是一种数据层封装格式，它通过第 3 层网络覆盖第 2 层网段。VXLAN 旨在在标准 IP 网络和共享物理网络基础架构上支持大规模多租户设计。现有的内部部署和基于云的网络可以透明地支持 VXLAN。VXLAN 定义为 MAC-in-UDP 封装，将容器第 2 层帧放置在底层 IP/UDP 头中。底层 IP / UDP 报头提供底层网络上主机之间的传输。The overlay is the stateless VXLAN tunnel that exists as point-to-multipoint connections between each host participating in a given overlay network. 由于覆盖层独立于底层拓扑，因此应用程序变得更加便携。因此，无论是在本地，在开发人员桌面上还是在公共云中，网络策略和连接都对应用程序透明。

![Packet Flow for an Overlay Network](./assets/packetwalk.png)

In this diagram we see the packet flow on an overlay network. Here are the steps that take place when `c1` sends `c2`packets across their shared overlay network:

- `c1` does a DNS lookup for `c2`. Since both containers are on the same overlay network the Docker Engine local DNS server resolves `c2` to its overlay IP address `10.0.0.3`.
- An overlay network is a L2 segment so `c1` generates an L2 frame destined for the MAC address of `c2`.
- The frame is encapsulated with a VXLAN header by the `overlay` network driver. The distributed overlay control plane manages the locations and state of each VXLAN tunnel endpoint so it knows that `c2` resides on `host-B` at the physical address of `192.168.1.3`. That address becomes the destination address of the underlay IP header.
- Once encapsulated the packet is sent. The physical network is responsible of routing or bridging the VXLAN packet to the correct host.
- The packet arrives at the `eth0` interface of `host-B` and is decapsulated by the `overlay` network driver. The original L2 frame from `c1` is passed to the `c2`'s `eth0` interface and up to the listening application.

### Overlay driver internal architecture

Docker Swarm 控制平面可自动完成 overlay network 的所有配置。不需要额外配置 VXLAN 或 Linux networking。数据平面加密是可选功能，也可以在创建网络时由 overlay 驱动程序自动配置。用户或网络管理员只需定义网络（docker network create -d overlay ...）并将容器附加到该网络即可。

![Overlay Network Created by Docker Swarm](assets/overlayarch.png)

在覆盖网络创建期间，Docker Engine 会在每台主机上创建覆盖所需的网络基础架构。 每个覆盖创建一个 Linux 桥及其关联的 VXLAN 接口。 仅当在主机上安排连接到该网络的容器时，Docker Engine 才会智能地在主机上实例化 overlay network。 这可以防止不存在连接容器的 overlay network 蔓延。

In the following example we create an overlay network and attach a container to that network. We'll then see that Docker Swarm/UCP automatically creates the overlay network.

```
# Create an overlay named "ovnet" with the overlay driver
$ docker network create -d overlay ovnet

# Create a service from an nginx image and connect it to the "ovnet" overlay network
$ docker service create --network ovnet --name container nginx
```

When the overlay network is created, you will notice that several interfaces and bridges are created inside the host.

```
# Run the "ifconfig" command inside the nginx container
$ docker exec -it container ifconfig

# docker_gwbridge network
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:04
          inet addr:172.18.0.4  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe12:4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

# overlay network
eth1      Link encap:Ethernet  HWaddr 02:42:0A:00:00:07
          inet addr:10.0.0.7  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::42:aff:fe00:7/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)
     
# container loopback
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:48 errors:0 dropped:0 overruns:0 frame:0
          TX packets:48 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:4032 (3.9 KiB)  TX bytes:4032 (3.9 KiB)
```

Two interfaces have been created inside the container that correspond to two bridges that now exist on the host. On overlay networks, each container will have at least two interfaces that connect it to the `overlay` and the `docker_gwbridge`.

| Bridge              | Purpose                                                      |
| ------------------- | ------------------------------------------------------------ |
| **overlay**         | The ingress and egress point to the overlay network that VXLAN encapsulates and (optionally) encrypts traffic going between containers on the same overlay network. It extends the overlay across all hosts participating in this particular overlay. One will exist per overlay subnet on a host, and it will have the same name that a particular overlay network is given. |
| **docker_gwbridge** | The egress bridge for traffic leaving the cluster. Only one `docker_gwbridge` will exist per host. Container-to-Container traffic is blocked on this bridge allowing ingress/egress traffic flows only. |

Docker overlay 驱动程序自 Docker Engine 1.9 以来就已存在，并且需要外部 KV 存储来管理网络状态。Docker 1.12 将控制平面状态集成到 Docker Engine 中，因此不再需要外部存储。 1.12 还引入了一些新功能，包括加密和服务负载平衡。引入的网络功能需要支持它们的 Docker Engine 版本，并且不兼容旧版本的 Docker Engine。

## Refs

- Docker overview. https://docs.docker.com/engine/docker-overview/.
- `man namespaces`. http://man7.org/linux/man-pages/man7/namespaces.7.html.
- `man network_namespaces`. http://man7.org/linux/man-pages/man7/network_namespaces.7.html.
- Swarm mode key concepts. https://docs.docker.com/engine/swarm/key-concepts/#/services-and-tasks.
- Docker networking. https://github.com/docker/labs/tree/master/networking.
- Bridge vs macvlan. http://hicu.be/bridge-vs-macvlan.
- Docker bridge vs linux bridge. http://blog.daocloud.io/docker-bridge/.
- Docker Networking with Linux. http://www.i3s.unice.fr/~urvoy/docs/VICC/3_vicc.pdf.