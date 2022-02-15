---
title: bridge 
categories: [linux, network, docker, bridge]
date: 2022-2-15 9:56:45
---

## 1. 什么是 bridge

bridge 是 Linux 虚拟出来的一种设备，类似于交换机，可以将系统中多个网络设备连接起来。

换句话说，从一个网络设备中发送的数据包，会转发/广播到连接在 bridge 的其余设备上

## 2. bridge internal

### 2.1 操控 bridge 设备

在 `br_init` 这个初始化函数中，将 `br_ioctl_hook` 设置为了 `br_ioctl_deviceless_stub`，brctl 的所有指令，都会触发该回调函数，通过不同的 command（cmd），分发不同的操作函数即可

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

比如当操作执行为 `SIOCBRDELBR` 的时候，就是增删 bridge 设备，触发 `br_add_bridge/br_delete_bridge` 

### 2.2 添加网桥

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

1. 创建 dev，并且通过 `br_dev_setup` 初始化该设备
2. 设置 dev 对应的 `rtnl_link_ops` 为  `br_link_ops`
3. 调用 `register_netdev` 将网桥设备注册到系统中

值得一提的是，在 `br_dev_setup` 函数当中完成了 bridge 的一些初始化操作。主要设置了 bridge 设备的一些字段，比如关键的 `dev->netdev_ops` `dev->ethtool_ops`，也就是具体的**机制**，与 Linux 庞大的网络框架相结合

### 2.3 net_bridge 结构

从 `br_add_bridge` 中可以看出，Linux 使用 `net_bridge` 来描述一个网桥设备，定义如下

```c
struct net_bridge {
	spinlock_t			lock;
	spinlock_t			hash_lock;
	struct list_head		port_list;
	struct net_device		*dev;
    ...
	struct hlist_head		fdb_list;
};
```

几个关键字段的作用为：

1. port_list：端口列表，保存所有接入网桥的网络接口
2. fdb_list：以网络接口的 MAC 地址为值，网桥端口为键的哈希表

### 2.4 发送数据

之前提到，`br_dev_setup` 中设置了 bridge 相关的网络操作回调函数，这不，发送数据包不就用到了吗。 

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

`br_dev_xmit` 主要是根据 skb 中的目标地址决定数据包发送到哪里。通过 `eth_hdr` 获取到 skb 的目标地址之后，一共有下面几种操作：

1. br_foward
2. br_flood(MULTICAST)
3. br_flood(BROADCAST)
4. br_flood(UNICAST)

#### 2.4.1 br_forward

根据上面的判定，如果不是 broadcast/multicast，并且可以在 forward_database(fdb) 当中找到该端口，就表明本次发送是 one-to-one 的转发，对应 `br_forward`

`br_forward` 函数最终会通过`dev_queue_xmit` 函数将 skb 发送出去，从这里开始即为**网络包收发**的主路径。

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

可以看出，`br_flood` 操作还是比较清晰的：遍历 `port_list`，对于其中的每一个 port，判断是否需要发送，如果不需要，直接跳过，如果需要，通过 `maybe_deliver` 发送数据包

`maybe_deliver` 则是间接调用了 `deliver_clone`，将 skb 复制一份并调用 `__bf_forward` 转发到特定的端口设备上，就不重复解释了
