---
title: veth 
categories: [linux, network, namespace, veth]
date: 2022-2-13 13:37:45
---

## veth 是什么

veth ，又名虚拟网络设备对，主要是用于解决不同网络命名空间之间的通信。

说起网络名称空间（network namespace），大家应该都不陌生，这是 Linux 用来隔离容器网络环境的一项技术，主要隔离的资源有：

1. iptables
2. 路由规则表
3. 网络设备列表

虽然不同 namespace 之间是隔离的，但也有办法让它们之间完成通信，veth 就是其中的一种比较常见的解决方式。

可以将 veth 看成是两块通过网线连接的网卡，只要将其中之一放置到网络命名空间 A，另一个放置到网络命名空间 B，那么两个不同的网络命名空间就能够通信。

众所周知，网线是一个冷酷无情的传输设备，从一端发送的数据会沿着线路传输到另一端，交付给另一端的设备，这一切是由硬件来完成的，不需要我们去干预，不过对于 veth 这种软件模拟的方式来说，就需要动一番脑子了。

常用的 veth 的使用方式为：

```bash
# 创建一对 veth
ip link add veth0 type veth peer name veth1
# 启动设备
ip link set dev veth0 up
ip link set dev veth1 up
# 设置 namespace
ip link set veth0 netns netns0
ip link set veth1 netns netns1
# 设置 veth0/1 的 ip
ip netns exec ns0 ip a a 10.1.1.2/24 dev veth0
ip netns exec ns1 ip a a 10.1.1.3/24 dev veth1
# ping 测试
ip netns exec ns0 ping -I veth0 10.1.1.3
```

![communicate between network namespaces through veth](/images/sysnet-veth.png)

## 1. veth internal

### 1.1 创建 veth pair

使用 ip 命令创建一对 veth 时，会触发 `rtnl_link_ops.newlink` 回调函数来完成这项任务，对应 `veth_newlink`

```c
static int veth_newlink(struct net *src_net, struct net_device *dev,
            struct nlattr *tb[], struct nlattr *data[],
            struct netlink_ext_ack *extack)
{
    /*
     * 首先设置和注册 peer
     */

    // ...

    net = rtnl_link_get_net(src_net, tbp);
    if (IS_ERR(net))
        return PTR_ERR(net);

    peer = rtnl_create_link(net, ifname, name_assign_type,
                &veth_link_ops, tbp, extack);
    
    // ...

    err = register_netdevice(peer);
    put_net(net);
    net = NULL;
    if (err < 0)
        goto err_register_peer;

    netif_carrier_off(peer);

    err = rtnl_configure_link(peer, ifmp);
    if (err < 0)
        goto err_configure_peer;

    /*
     * 之后注册 dev
     */
    
    //...

    err = register_netdevice(dev);

    netif_carrier_off(dev);

    /*
     * 将 dev 和 peer 关联起来
     */

    priv = netdev_priv(dev);
    rcu_assign_pointer(priv->peer, peer);

    priv = netdev_priv(peer);
    rcu_assign_pointer(priv->peer, dev);

    return 0;

    // 错误处理 ...
}
```

veth 由两个设备组成，其中 dev 在调用 `veth_newlink` 之前已经创建完成了，接下来首先需要通过 `rtnl_create_link` 创建另一个设备 peer，并通过 `register_netdevice` 分别将 peer 和 dev 注册到 netdevice 列表当中。

之后的这步是 veth 的核心：通过 priv 字段关联 dev 和 peer

```c
static inline void *netdev_priv(const struct net_device *dev)
{
    return (char *)dev + ALIGN(sizeof(struct net_device), NETDEV_ALIGN);
}
```

这里有必要解释一下 `net_device` 的结构，以及 `netdev_priv` 的作用

Linux 内核使用 `net_device` 来表示一个网络设备，但是不同厂商的设备规格有一定差别，为了最大程度**求同存异**，在 net_device 结构体中保留了私有数据的存储空间，一般保存在 net_device 对象结束之后的位置（不难从 `netdev_priv` 中的 `sizeof(struct net_device` 得到证实）

```text

        |------------|\
        | net_device | \
        |            | / sizeof net_device
        |------------|/
        |------------|\
        | xx_private | \
        |            | / sizeof private struct
        |------------|/
```

而对于 veth 来说，这个 private 字段就是 `struct veth_priv`

```c
struct veth_priv {
    struct net_device __rcu    *peer;
    atomic64_t        dropped;
    struct bpf_prog        *_xdp_prog;
    struct veth_rq        *rq;
    unsigned int        requested_headroom;
};
```

因此，通过 peer 字段就能关联另一个 `net_device`，`veth_newlink` 中 dev 与 peer 的配对过程也就很清晰了。

### 1.2 初始化 veth

创建好 veth 之后，还需要通过 `rtnl_link_ops.setup` 回调函数进行初始化，对应 `veth_setup`

该函数中主要设置了 veth 对应 net_device 的一些字段，最关键的莫过于 `netdev_ops` 了

```c
static void veth_setup(struct net_device *dev)
{
    ether_setup(dev);
    dev->netdev_ops = &veth_netdev_ops;

    ...
}
```

### 1.3 veth_xmit

Linux 提供框架，具体机制需要特殊对待。在网络包的发送路径中，经过 ip 和 neighbor 系统之后，会调用具体网络设备的  `ndo_start_xmit` 回调函数发送数据包，如果是真实的网卡驱动，一般会将 skb 放到自己的缓冲区并发送数据，之后通知 CPU......，对于 veth 这种虚拟设备而言则大不相同，请看 `veth_xmit`

```c
static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct veth_priv *rcv_priv, *priv = netdev_priv(dev);
    rcu_read_lock();
    rcv = rcu_dereference(priv->peer);
    
    if (likely(veth_forward_skb(rcv, skb, rq, rcv_xdp) == NET_RX_SUCCESS)) {
        if (!rcv_xdp)
            dev_lstats_add(dev, length);
    } else {
drop:
        atomic64_inc(&priv->dropped);
    }

    if (rcv_xdp)
        __veth_xdp_flush(rq);

    rcu_read_unlock();

    return NETDEV_TX_OK;
}
```

首先获取到了 `veth_priv` 结构中的 peer，也就是与当前设备关联的另一个虚拟网络设备，接下来调用 `veth_forward_skb`

该函数的命名还是非常贴切的：转发由 veth 对端设备发来的 skb 数据包

```c
static int veth_forward_skb(struct net_device *dev, struct sk_buff *skb,
                struct veth_rq *rq, bool xdp)
{
    return __dev_forward_skb(dev, skb) ?: xdp ?
        veth_xdp_rx(rq, skb) :
        netif_rx(skb);
}
```

`veth_forward_skb` 有两条路径来处理数据包，暂且不论第一条 xdp 处理路径，更常用的是后面的 `netif_rx` 函数，其将 skb 放到了 cpu backlog 队列中，提供给上层协议使用，mission complete！

```c
static int netif_rx_internal(struct sk_buff *skb)
{
    int ret;
    ...
    trace_netif_rx(skb);
    ret = enqueue_to_backlog(skb, get_cpu(), &qtail);
    ...
    return ret;
}
```
