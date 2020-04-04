# Linux：网络协议栈性能优化机制

[TOC]

## 1 NAPI

​		new api：将网卡收包事件通知内核的方式由硬件中断改为轮询（poll）。

​		轮询是基于中断的处理的替代方法。内核可以定期检查传入的网络数据包是否到达而不会被中断，从而消除了中断处理的开销。但是，建立最佳轮询频率很重要。过于频繁的轮询会通过反复检查尚未到达的传入数据包来浪费CPU资源。另一方面，轮询频率太低又会降低系统对传入数据包的反应从而引入延迟，并且如果传入数据包缓冲区在处理之前已满，则可能导致数据包丢失。

​		作为一种折衷，Linux内核默认使用中断驱动模式，并且仅在传入数据包的流量超过某个阈值（称为网络接口的“权重”）时才切换到轮询模式。

​		兼容NAPI接口的网卡驱动程序将按以下方式工作：

+ 数据包接收中断被禁用。

- 网卡驱动程序为内核提供了一种轮询方法。该方法将获取网卡缓冲区上所有可用的传入数据包，以便内核随后将它们处理。
- 允许时，内核调用设备轮询方法以尽可能一次处理许多数据包。

## 2 TSO和GSO

​		tcp segment offload：将tcp分段功能从内核协议栈卸载到网卡执行，需要网卡同时支持tso和checksum offload。

​		generic segment offload：对于不支持tso的网卡，尽可能推迟tcp分段的执行也是有好处的，因此gso将tcp分段功能的执行推迟到调用网卡驱动程序的xmit()前。

## 3 LRO和GRO

​		large receive offload：它通过将多个 tcp 数据包聚合在一个 skb 结构，在稍后的某个时刻作为一个大数据包交付给上层的网络协议栈，以减少上层协议栈处理 skb 的开销，提高系统接收 TCP 数据包的能力。合并了多个 skb 的超级 skb，能够一次性通过网络协议栈，而不是多次，这对 CPU 负荷的减轻是显然的。

​		generic receive offload：GRO提供的接口和LRO提供的接口非常的类似，但更加的简洁。

参考文献

[https://www.ibm.com/developerworks/cn/linux/l-cn-network-pt/index.html](https://www.ibm.com/developerworks/cn/linux/l-cn-network-pt/index.html)