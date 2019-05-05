# Docker-compatible Overlay Networks

Container overlay networks 主要用于解决 Docker 容器的跨主机通信问题。Docker 在早前的时候没有考虑跨主机的容器通信，这个特性直到 Docker 1.9 才出现。在此之前，如果希望位于不同主机的容器能够通信，一般有几种方法：

- 使用端口映射：直接把容器的服务端口映射到主机上，主机直接通过映射出来的端口通信（Docker bridge）。
- 把容器放到主机所在的网段：修改 Docker 的 IP 分配网段和主机一致，还要修改主机的网络结构（通过 MACVLAN、SRIOV、Linux bridge 等技术让容器中的网络接口和主机接口处于同一个 L2 广播域）。
- 三方容器 overlay networks，包括 Flannel、Weave 和 Calico。

## Docker network model (CNM)

![img](assets/cnm-model.jpg)

CNM 中有一些高层的抽象，不依赖于 OS 和基础设施，所以应用无需考虑底层的软件栈。包括：

- **Sandbox** — A Sandbox contains the configuration of a container's network stack. This includes management of the container's interfaces, routing table, and DNS settings. An implementation of a Sandbox could be a `Linux Network Namespace`, a `FreeBSD Jail`, or other similar concept. A Sandbox may contain many endpoggints from multiple networks.
- **Endpoint** — An Endpoint joins a Sandbox to a Network. The Endpoint construct exists so the actual connection to the network can be abstracted away from the application. This helps maintain portability so that a service can use different types of network drivers without being concerned with how it's connected to that network.
- **Network** — The CNM does not specify a Network in terms of the OSI model. An implementation of a Network could be a Linux bridge, a VLAN, etc. A Network is a collection of endpoints that have connectivity between them. Endpoints that are not connected to a network will not have connectivity on a Network.

## Docker overlay network

![1555940372043](assets/1555940372043.png)

## Flannel / Weave / Calico

TODO

## Refs

- An Analysis and Empirical Study of Container Networks. http://ranger.uta.edu/~jrao/papers/INFOCOM18.pdf.