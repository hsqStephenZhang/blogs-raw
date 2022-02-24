---
title: veth 
categories: [linux, network, namespace, veth]
date: 2022-2-13 13:37:45
---

## 1. veth 是什么

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

## 2. veth internal

下面分别从 创建，初始化，发送数据 三个角度来剖析 veth 源码。

### 2.1 创建 veth pair

使用 ip 命令创建一对 veth 时，会通过 netlink 触发 `rtnl_link_ops.newlink` 回调函数，如果感兴趣，可以看我的另一篇文章 {% post_link linux/network/netlink [netlink 机制] %} ，对应 `veth_newlink`

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
     * 关联 dev 和 peer
     */

    priv = netdev_priv(dev);
    rcu_assign_pointer(priv->peer, peer);

    priv = netdev_priv(peer);
    rcu_assign_pointer(priv->peer, dev);

    return 0;

    // 错误处理 ...
}
```

veth 由两个设备组成，其中 dev 在调用 `veth_newlink` 之前已经创建完成了，接下来首先需要通过 `rtnl_create_link` 创建另一个设备 peer，并通过 `register_netdevice` 分别将 peer 和 dev 注册到 netdevice 列表当中

下一步是 veth 的核心：通过 priv 字段关联 dev 和 peer

这里有必要解释一下 `net_device` 的结构，以及 `netdev_priv` 的作用

Kernel 使用 `net_device` 来表示一个网络设备，但是不同厂商的设备规格有一定差别，所以 net_device 结构体采用了**面向对象**的思想，不仅有所有设备共有的结构（基类），还保留了私有数据的存储空间（子类），私有结构一般保存在 net_device 对象结束之后的位置（不难从 `netdev_priv` 中的 `sizeof(struct net_device` 得到证实），示意图如下所示：

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

```c
static inline void *netdev_priv(const struct net_device *dev)
{
    return (char *)dev + ALIGN(sizeof(struct net_device), NETDEV_ALIGN);
}
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

### 2.2 初始化 veth

创建好 veth 之后，还需要通过 `rtnl_link_ops.setup` 进行初始化才能使用，对应 `veth_setup`

该函数中主要设置了 veth 对应 net_device 的一些字段

```c
static void veth_setup(struct net_device *dev)
{
    ether_setup(dev);
    dev->netdev_ops = &veth_netdev_ops;

    ...
}
```

关注 `dev->netdev_ops = &veth_netdev_ops;`，因为我们都知道，内核最终发送 skb 数据包就是通过回调 `netdev_ops->ndo_start_xmit`，这样就进入到了 veth 的掌握之中。

同时，在 veth 模块的初始化操作中，还将 veth_netdev_ops 注册到了 rtnetlink 中，从而可以通过 netlink 向外提供服务

```c
static __init int veth_init(void)
{
	return rtnl_link_register(&veth_link_ops);
}
```

像 `ip link add veth0 type veth peer name veth1` 这些和 veth 相关的网络命令，都会通过 netlink socket，最终交由 veth_link_ops 中的回调函数处理。

### 2.3 发送数据

在驱动层，内核通过 `netdev_ops->ndo_start_xmit` 发送数据包，前面说到，`veth_setup` 函数中已经将 netdev_ops 设置为 veth_netdev_ops，其中的 ndo_start_xmit 对应下面的 `veth_xmit`

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

来看该函数的逻辑：首先获取到了 `veth_priv` 结构中的 peer，也就是与当前设备关联的另一个虚拟网络设备，接下来调用 `veth_forward_skb`

该函数的命名还是非常贴切的：将 skb 数据包转发到 veth 的 peer 上

```c
static int veth_forward_skb(struct net_device *dev, struct sk_buff *skb,
                struct veth_rq *rq, bool xdp)
{
    return __dev_forward_skb(dev, skb) ?: xdp ?
        veth_xdp_rx(rq, skb) :
        netif_rx(skb);
}
```

`veth_forward_skb` 有两条路径来处理数据包，暂且不论第一条 xdp 处理路径，`netif_rx` 才是热路径，其最终调用了 `netif_rx_internal`，将 skb 放到了 cpu backlog 队列中，并且发送软中断进行处理，之后就进入到网络收包路径上，最终调用 peer 的收包回调函数进行处理（dev 和 peer 的 net_dev_ops 分别在 veth_setup 和 veth_newlink 中设置，不过都为 veth_netdev_ops）

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
