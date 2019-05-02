# Docker-compatible Overlay Networks

Container overlay networks 主要用于解决 Docker 容器的跨主机通信问题。Docker 在早前的时候没有考虑跨主机的容器通信，这个特性直到 Docker 1.9 才出现。在此之前，如果希望位于不同主机的容器能够通信，一般有几种方法：

- 使用端口映射：直接把容器的服务端口映射到主机上，主机直接通过映射出来的端口通信（Docker bridge）。
- 把容器放到主机所在的网段：修改 Docker 的 IP 分配网段和主机一致，还要修改主机的网络结构（通过 MACVLAN、SRIOV、Linux bridge 等技术让容器中的网络接口和主机接口处于同一个 L2 广播域）。
- 三方容器 overlay networks，包括 Flannel、Weave 和 Calico。

## Docker network model

![img](assets/cnm-model.jpg)

## Docker overlay network

![1555940372043](assets/1555940372043.png)







## Flannel / Weave / Calico

TODO

## Refs

- An Analysis and Empirical Study of Container Networks. http://ranger.uta.edu/~jrao/papers/INFOCOM18.pdf.