---
title: bridge 
categories: [linux, network, docker, bridge]
date: 2022-2-15 9:56:45
---

## 1. Overlook

### 1.1 什么是 bridge

即使你之前没有听说过 bridge，但交换机（switch）总有所耳闻吧。

通常所说的交换机工作在 OSI 参考模型的数据链路层，使用 MAC 地址转发数据，通过学习得到 MAC 地址到目标端口的映射表，依据这张表决定转发的下一个设备，并且有广播和单播等多种能力。

bridge 是 Linux 软件模拟出来的一种**类似**交换机的虚拟设备。

### 1.2 如何使用 bridge

主流 Linux 发行版都提供了 bridge 相关的命令行工具，比如 arch 可以通过 `sudo pacman -S bridge-utils` 安装。
使用方式如下表所示：

| 参数 | 说明 | 示例 |
| :----:| :----: | :----: |
| addbr bridge|创建网桥|brctl addbr br10|
| delbr bridge|删除网桥|brctl delbr br10|
| addif bridge device|将网卡接口接入网桥|brctl addif br10 eth0|
| delif bridge device|删除网桥接入的网卡接口|brctl delif br10 eth0|
| show bridge|查询网桥信息|brctl show br10|
| stp bridge {on|off}|启用禁用| STPbrctl stp br10 off/on|
| showstp bridge|查看网桥 STP 信息|brctl showstp br10|
| setfd bridge time|设置网桥延迟|brctl setfd br10 10|
| showmacs bridge|查看 mac 信息|brctl showmacs br10|

## 2. bridge internal

光练不读源码假把式，还是需要深入到 Linux 源码，来看看 bridge 究竟有没有那么神奇

### 2.1 初始化网桥

万物离不开初始化，bridge 模块通过 `br_init` 完成这项任务

在 `br_init` 中，将 `br_ioctl_hook` 设置为了 `br_ioctl_deviceless_stub`，brctl 的所有指令，都会触发该回调函数，通过不同的 command（cmd），分发不同的操作函数即可

```c
static int __init br_init(void)
{
    int err;

    // 注册 stp protocol
    err = stp_proto_register(&br_stp_proto);
    if (err < 0) {
        pr_err("bridge: can't register sap for STP\n");
        return err;
    }

    // forward database 初始化
    err = br_fdb_init();
    if (err)
        goto err_out;

    // 注册 bridge net sys
    err = register_pernet_subsys(&br_net_ops);
    if (err)
        goto err_out1;
    
    // ...

    // netlink 初始化
    err = br_netlink_init();
    
    // 设置 bridge ioctl 回调函数
    brioctl_set(br_ioctl_deviceless_stub);

    // 错误处理
    // ...
}
```

这里的 `brioctl_set(br_ioctl_deviceless_stub)` 值得一提，通过该函数设置了 bridge 的 ioctl 回调函数，上述的 brctl 最终就是在和 `br_ioctl_deviceless_stub` 打交道

`br_ioctl_deviceless_stub` 逻辑也非常简单，对于不同的 command 派发不同的操作，这里也可以看到 `br_add_bridge` 和 `br_delete_bridge` 两个增伤的操作，是不是上面死板的命令一下子鲜活起来了呢？

```c
int br_ioctl_deviceless_stub(struct net *net, unsigned int cmd, void __user *uarg)
{
    switch (cmd) {
    case SIOCGIFBR:
    case SIOCSIFBR:
        return old_deviceless(net, uarg);

    case SIOCBRADDBR:
    case SIOCBRDELBR:
    {
        char buf[IFNAMSIZ];

        if (!ns_capable(net->user_ns, CAP_NET_ADMIN))
            return -EPERM;

        if (copy_from_user(buf, uarg, IFNAMSIZ))
            return -EFAULT;

        buf[IFNAMSIZ-1] = 0;
        if (cmd == SIOCBRADDBR)
            return br_add_bridge(net, buf);

        return br_del_bridge(net, buf);
    }
    }
    return -EOPNOTSUPP;
}
```

### 2.2 添加网桥

接下来详细解释一下，**添加网桥**要经历哪些必不可少的步骤

```c
int br_add_bridge(struct net *net, const char *name)
{
    struct net_device *dev;
    int res;

    dev = alloc_netdev(sizeof(struct net_bridge), name, NET_NAME_UNKNOWN,
               br_dev_setup);

    if (!dev)
        return -ENOMEM;

    dev_net_set(dev, net);
    dev->rtnl_link_ops = &br_link_ops;

    res = register_netdev(dev);
    if (res)
        free_netdev(dev);
    return res;
}
```

`br_add_bridge` 主要做了下面几件事：

1. 创建 bridge device，并且通过 `br_dev_setup` 初始化该设备
2. 设置 dev 对应的 `rtnl_link_ops` 为  `br_link_ops`
3. 调用 `register_netdev` 将网桥设备注册到系统中

值得一提的是，在 `br_dev_setup` 函数当中完成了 bridge 的一些初始化操作。主要设置了 bridge 设备的一些字段，比如关键的 `dev->netdev_ops` `dev->ethtool_ops`

### 2.3 net_bridge 结构

从 `br_add_bridge` 中可以看出，Linux 使用 `net_bridge` 来描述一个网桥设备，定义如下

```c
struct net_bridge {
    spinlock_t            lock;
    spinlock_t            hash_lock;
    struct list_head        port_list;
    struct net_device        *dev;
    ...
    struct hlist_head        fdb_list;
};
```

几个关键字段的作用为：

1. port_list：端口列表，保存所有接入网桥的网络接口
2. fdb_list：以网络接口的 MAC 地址为值，网桥端口为键的哈希表

```text
Tips:
    之所以 fdb_list 以 MAC 地址为值，网桥端口为键，是因为网桥是一个二层设备，需要根据目标的 MAC 地址进行广播或者单播，当目标的 MAC 地址在网桥中对应一个端口，就表明此次为单播，否则为广播
```

### 2.4 发送数据

之前提到，`br_dev_setup` 中设置了 bridge 相关的网络驱动，这不，发送数据包不就用到了 `dev->netdev_ops.ndo_start_xmit` 吗。

对于 bridge 而言，发送 skb 的回调函数 `ndo_start_xmit` 即为 `br_dev_xmit`

```c
netdev_tx_t br_dev_xmit(struct sk_buff *skb, struct net_device *dev)
{
    ...
    dest = eth_hdr(skb)->h_dest;
    if (is_broadcast_ether_addr(dest)) {
        br_flood(br, skb, BR_PKT_BROADCAST, false, true);
    } else if (is_multicast_ether_addr(dest)) {
        if (unlikely(netpoll_tx_running(dev))) {
            br_flood(br, skb, BR_PKT_MULTICAST, false, true);
            goto out;
        }

        ...
        if ((mdst || BR_INPUT_SKB_CB_MROUTERS_ONLY(skb)) &&
            br_multicast_querier_exists(br, eth_hdr(skb)))
            br_multicast_flood(mdst, skb, false, true);
        else
            br_flood(br, skb, BR_PKT_MULTICAST, false, true);
    } else if ((dst = br_fdb_find_rcu(br, dest, vid)) != NULL) {
        br_forward(dst->dst, skb, false, true);
    } else {
        br_flood(br, skb, BR_PKT_UNICAST, false, true);
    }
out:
    rcu_read_unlock();
    return NETDEV_TX_OK;
}
```

`br_dev_xmit` 主要是根据 skb 中的目标 MAC 地址决定数据包发送到哪些网络端口。通过 `eth_hdr` 获取到 skb 的目标 MAC 地址之后，有下面几种可能的操作：

1. br_foward             -- 转发
2. br_flood(MULTICAST)   -- 多播
3. br_flood(BROADCAST)   -- 广播
4. br_flood(UNICAST)     -- 单播

#### 2.4.1 br_forward

在 `br_dev_xmit` 的判定逻辑中，如果本次发送方式不为 broadcast/multicast，并且可以在 forward_database(fdb) 当中找到该端口，就表明本次发送是**转发**，对应 `br_forward`

`br_forward` 函数最终会通过 `dev_queue_xmit` 将 skb 发送出去。相信大家对于 `dev_queue_xmit` 已经不陌生了，这可以说是承接 ip 层和设备驱动层的一个关键函数，从这里往下执行，最终会调用网络设备的 `ndo_start_xmit` 回调函数发送 skb。

值得一提的是，`br_foward` 中也存在一个 `NF_HOOK`，名称为 `NFPROTO_BRIDGE`，可以通过编译选项开启。

#### 2.4.2 br_flood

flood 的字面意思为洪水。那么 `br_flood` 的作用就是将数据包发送到网桥上的**所有**接口设备上

```c
void br_flood(struct net_bridge *br, struct sk_buff *skb,
          enum br_pkt_type pkt_type, bool local_rcv, bool local_orig)
{
    struct net_bridge_port *prev = NULL;
    struct net_bridge_port *p;

    list_for_each_entry_rcu(p, &br->port_list, list) {
        // 三种 flood 类型
        switch (pkt_type) {
        case BR_PKT_UNICAST:
            if (!(p->flags & BR_FLOOD))
                continue;
            break;
        case BR_PKT_MULTICAST:
            if (!(p->flags & BR_MCAST_FLOOD) && skb->dev != br->dev)
                continue;
            break;
        case BR_PKT_BROADCAST:
            if (!(p->flags & BR_BCAST_FLOOD) && skb->dev != br->dev)
                continue;
            break;
        }

        if (p->flags & BR_PROXYARP)
            continue;
        if ((p->flags & (BR_PROXYARP_WIFI | BR_NEIGH_SUPPRESS)) &&
            BR_INPUT_SKB_CB(skb)->proxyarp_replied)
            continue;

        // 发送数据包
        prev = maybe_deliver(prev, p, skb, local_orig);
        if (IS_ERR(prev))
            goto out;
    }

    ...
}
```

可以看出，`br_flood` 操作还是比较清晰的：遍历连接在网桥设备上的每一个端口，判断是否符合当前广播的类型 UNICAST/MULTICAST/BROADCAST，不符合则直接跳过，否则通过 `maybe_deliver` 发送数据包

`maybe_deliver` 则是间接调用了 `deliver_clone`，将 skb 复制一份并调用 `__bf_forward` 转发到特定的端口设备上，就不重复解释了

### 2.5 MAC 地址

作为一个与交换机功能类似的虚拟设备，bridge 自然也需要实现和 OSI 数据链路层协议相关的功能。

这里回顾一下二层交换机的功能：

1. 收到某网段（设为A）MAC 地址为 X 的计算机发给 MAC 地址为 Y 的计算机的数据包。交换机从而记下了 MAC 地址 X 在网段 A。这称为学习（learning）。
2. 交换机还不知道MAC地址Y在哪个网段上，于是向除了 A 以外的所有网段转发该数据包。这称为泛洪（flooding）。
3. MAC地址Y的计算机收到该数据包，向 MAC 地址 X 发出确认包。交换机收到该包后，从而记录下 MAC 地址 Y 所在的网段。
4. 交换机向MAC地址X转发确认包。这称为转发（forwarding）。
5. 交换机收到一个数据包，查表后发现该数据包的来源地址与目的地址属于同一网段。交换机将不处理该数据包。这称为过滤（filtering）。
6. 交换机内部的MAC地址-网段查询表的每条记录采用时间戳记录最后一次访问的时间。早于某个阈值（用户可配置）的记录被清除。这称为老化（aging）。

上面已经介绍了 flood 和 forward 的具体机制，而和 MAC 地址相关一些 learning，aging 操作在[源码](https://github.com/torvalds/linux/blob/master/net/bridge/br_fdb.c)当中也有对应的实现，详情可以参考 `br_fdb_change_mac_address` `br_fdb_cleanup` `br_fdb_flush` `br_fdb_delete_by_port` 等函数
