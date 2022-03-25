---
title: tun/tap
categories: [linux, network, tun, tap]
date: 2022-3-25 10:09:00
---

## 1. 什么是 tun/tap

tun、tap 是 Linux 提供的两种可收发数据的虚拟网卡设备。

它们除了不具备物理网卡的硬件功能外，和物理网卡的功能是一样的，那这两者之间有什么区别和联系呢？

- tun 是三层设备，其封装的外层是 IP 头
- tap 是二层设备，其封装的外层是以太网帧(frame)头
- tun 是 PPP 点对点设备，没有 MAC 地址
- tap 是以太网设备，有 MAC 地址
- tap 比 tun 更接近于物理网卡，可以认为，tap 设备等价于去掉了硬件功能的物理网卡

基于这些特点，它们的应用场景也不尽相同：tap 设备通常用来连接其它网络设备(它更像网卡)，tun 设备通常用来结合用户空间程序实现再次封装。

换句话说，tap 设备通常接入到虚拟交换机(bridge)上作为局域网的一个节点，qemu 就是基于 tap 和 网桥与外界进行数据交互，tun 设备通常用来实现三层的 ip 隧道（VTun）


### 1.2 使用方式

tun，tap 通过 `/dev/net/tun` 设备向用户提供操作接口，可以通过 `open/read/write` 等标准方法来操作这个字符设备，大体操作方式如下：

1. open("/dev/net/tun", O_RDWR)。这一步要求用户对于 `/dev/net/tun` 有读写权限，否则无法成功。
2. ioctl(fd, TUNSETIFF, &ifr)。传入上一步获取的文件描述符，并且设置一些 flags，决定创建的是 tun 还是 tap 设备（IFF_TUN | IFF_TAP）。
3. 通过 read/write 来操控 tun/tap 设备



## 2. 底层实现


### 2.1 收包路径

```c
sys_read
    --> vfs_read
        --> __vfs_read
            --> tun_chr_read_iter
```

### 2.2 发包路径

```c
sys_write
    --> vfs_write
        --> do_readv_writev
            --> tun_get_user
                --> netif_rx_ni
                    --> netif_rx_internal
```

只会会进入 net_rx_action，触发真正的收包流程。

如果某个 tap 设备连接在了网桥上，则会在接下来的流程中触发 `br_handle_frame` 的逻辑，如果对网桥的处理路径不太熟悉，可以参考之前的一篇{% post_link linux/network/bridge 文章 %}


## 3. 参考资料

- [云计算底层技术-虚拟网络设备(tun/tap,veth)](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/)
- [Linux虚拟网络设备之tun/tap](https://segmentfault.com/a/1190000009249039)
- [linux net bridge技术分析](https://cloud.tencent.com/developer/article/1087504)