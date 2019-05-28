# Linux Packet Scheduler

[TOC]

## Queueing Disciplines for Bandwidth Management

## Queues and Queueing Disciplines explained

通过排队，我们决定数据发送的方式。重要的是要意识到我们只能塑造我们发送的数据。

随着互联网的运作方式，我们无法直接控制人们发送给我们的内容。这有点像你家里的（物理！）邮箱。除了联系每个人之外，你无法影响世界来修改他们发送给你的邮件数量。

但是，Internet主要基于TCP / IP，它具有一些可以帮助我们的功能。 TCP / IP无法知道两台主机之间的网络容量，所以它只是开始越来越快地发送数据（“慢启动”），当数据包开始丢失时，因为没有空间发送它们，它将慢一点。事实上它比这更聪明，但我们稍后再说。

这相当于你还没阅读完一半的邮件，并希望人们不再发送给您。区别在于它适用于互联网:-)

如果您有路由器并且希望阻止网络中的某些主机下载速度过快，则需要在路由器的 inner  接口上进行整形，该路由器将数据发送到您自己的计算机。

您还必须确保控制链接的瓶颈。如果你有一个100Mbit网卡并且你有一个256kbit链路的路由器，你必须确保你没有发送比你的路由器可以处理更多的数据。否则，路由器将控制链路并塑造可用带宽。We need to 'own the queue' so to speak, and be the slowest link in the chain. Luckily this is easily possible.

## TODO: Looks too complicated.

## Refs

- Ch9. Queueing Disciplines for Bandwith Management. http://www.tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.qdisc.html. 