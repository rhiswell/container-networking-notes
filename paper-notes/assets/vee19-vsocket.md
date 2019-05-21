# PaperNote: vSocket

## Background & Motivation

公有云对虚拟化的一些需求：

- Network security: the connection establishment must comply with the security rules.
- Network isolation: different tenants' traffic must be isolated by the network overlay technique such as VxLAN.
- Live migration: VMs can be deployed with flexibility and lively migrated.
- Statistics: the traffic should be counted and monitored if needed.

|  Solution   | Virt. Method | Net. Isolation | Net. Security | Live Migration | Statistics | Cost of Maintenance |
| :---------: | :----------: | :------------: | :-----------: | :------------: | :--------: | :-----------------: |
| RNIC-VF+VMA |    SR-IOV    |     X @R1      |       X       |       X        |     X      |          -          |
|  vRDMA+VMA  |      PV      |       X        |       X       |       -        |     -      |          -          |
|   HyV+VMA   |      PV      |       X        |       X       |       -        |     -      |      High @R2       |

Reasons:

1. The NIC hardware does not support complicated overlay techniques such as VxLAN.
2. HyV has to maintain a device/vendor-dependent driver in both the frontend and the backend, making it difficult for the cloud provider to upgrade physical devices. 

vSocket 是如何满足这些需求的呢？

1. vSocket 的 control path 还是走传统 TCP/IP 协议栈（control 操作包括**建立连接**和**销毁连接**），故可以使用基于 TCP/IP 协议栈的防火墙技术；
2. vSocket 方案中，因为 host 可以获取和监管所有的流量和虚拟设备，故可以支持 statistics（可用于计费和数据分析）；
3. 至于 vSocket 如何满足 network isolation、live migration，还需要讨论。

## Design and Implementation

Pass

## Evaluation

### Layouts

![image-20180923214143108](./assets/eval-architectures.png)

### Concerned problems

![](./assets/micro-bm-p0.png)

现象一：在 micro-benchmarks，VM/DPDK-OVS 的性能最差（见上图：相比 VM/kernel-OVS，latency 高了 3.4x，throughput 低了4.5x）。师兄认为是 DPDK Mellanox ConnectX-4 VPI 设备驱动的 batching delay 造成的。若去除该因素，则 VM/DPDK-OVS 的延迟为 43 us，略好于 VM/Kernel-OVS 的 61 us。但该问题反应了 VM/DPDK-OVS 网络性能的不确定性（non-deterministic）。例如，实际场景下，在少量连接和大量连接两个不同情况下，网络的 latency 分布不一致，如下图蓝线。

![](./assets/redis-bm-p0.png)

现象二：如下图，SRIOV-VMA/vSocket（黑线和粉线）比其它网络性能好很多，说明 VM kernel 对 VM 网络性能影响较大，故在 VM 中使用基于 kernel-bypass 的网络可以给 VM 网络带来很好的性能提升。

![](./assets/redis-bm-tp.png)

