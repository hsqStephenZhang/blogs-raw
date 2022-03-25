---
title: bridge 
categories: [linux, network, docker, bridge]
date: 2022-2-15 9:56:00
---

## 1. Overlook

### 1.1 什么是 bridge

即使你之前没有听说过 bridge 这种新奇玩意，但凡是学过计算机网络的同学，对交换机（switch）总有所耳闻吧。

通常所说的**交换机**工作在 OSI 参考模型的**数据链路层**，使用 MAC 地址转发数据，通过学习得到 MAC 地址到目标端口的映射表，依据这张表决定转发的下一个设备，并且有广播和单播等多种能力。

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

光练不读源码假把式，还需要深入到 Linux 源码，来看看 bridge 究竟有没有那么神奇

### 2.1 初始化网桥

万物不及初始化，bridge 模块通过 `br_init` 完成这项任务

在 `br_init` 中，将 `br_ioctl_hook` 设置为了 `br_ioctl_deviceless_stub`，brctl 的所有指令，都会触发该回调函数，通过不同的 command（cmd），分发（dispatch）不同的操作函数即可，如果对应到硬件上，相当于多路复用器。

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

    // forwarding database 初始化
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

`br_ioctl_deviceless_stub` 逻辑也非常简单，对于不同的 command 派发（dispatch）不同的操作，比如 `br_add_bridge` 和 `br_delete_bridge` 两个增删的操作，分别对应了 `SIOCBRADDBR` 和 `SIOCBRDELBR`

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

可以先从一个基础的**添加网桥**操作入手，详细解释一下要经历哪些必不可少的步骤

```c
int br_add_bridge(struct net *net, const char *name)
{
    struct net_device *dev;
    int res;

    // 1. 创建设备
    dev = alloc_netdev(sizeof(struct net_bridge), name, NET_NAME_UNKNOWN,
               br_dev_setup);

    if (!dev)
        return -ENOMEM;

    dev_net_set(dev, net);
    // 2. 设置 rtnl_link_ops
    dev->rtnl_link_ops = &br_link_ops;

    // 3. 注册网络设备
    res = register_netdev(dev);
    if (res)
        free_netdev(dev);
    return res;
}
```

`br_ioctl_deviceless_stub` 中可以看到，添加网桥对应 `br_add_bridge`，`br_add_bridge` 主要做了下面几件事：

1. 创建 bridge device，并且通过 `br_dev_setup` 初始化该设备
2. 设置 dev 对应的 `rtnl_link_ops` 为  `br_link_ops`
3. 调用 `register_netdev` 将网桥设备注册到系统中

`br_dev_setup` 用于完成内存分配之后的初始化操作，主要设置了 bridge 设备的一些字段，比如关键的 `dev->netdev_ops`，是网桥设备对上层的接口

```c
static const struct net_device_ops br_netdev_ops = {
	.ndo_open		 = br_dev_open,
	.ndo_stop		 = br_dev_stop,
	.ndo_init		 = br_dev_init,
	.ndo_uninit		 = br_dev_uninit,
	.ndo_start_xmit		 = br_dev_xmit,
	.ndo_get_stats64	 = br_get_stats64,
	.ndo_set_mac_address	 = br_set_mac_address,
	.ndo_set_rx_mode	 = br_dev_set_multicast_list,
	.ndo_change_rx_flags	 = br_dev_change_rx_flags,
	.ndo_change_mtu		 = br_change_mtu,
	.ndo_do_ioctl		 = br_dev_ioctl,
#ifdef CONFIG_NET_POLL_CONTROLLER
	.ndo_netpoll_setup	 = br_netpoll_setup,
	.ndo_netpoll_cleanup	 = br_netpoll_cleanup,
	.ndo_poll_controller	 = br_poll_controller,
#endif
	.ndo_add_slave		 = br_add_slave,
	.ndo_del_slave		 = br_del_slave,
	.ndo_fix_features        = br_fix_features,
	.ndo_fdb_add		 = br_fdb_add,
	.ndo_fdb_del		 = br_fdb_delete,
	.ndo_fdb_dump		 = br_fdb_dump,
	.ndo_fdb_get		 = br_fdb_get,
	.ndo_bridge_getlink	 = br_getlink,
	.ndo_bridge_setlink	 = br_setlink,
	.ndo_bridge_dellink	 = br_dellink,
	.ndo_features_check	 = passthru_features_check,
};
```

删除设备的源码也留给读者自己研读。

### 2.3 添加网桥 slave 设备

网桥设备创建完成后需要在其下面添加 slave 设备才能使网桥设备正常工作起来，主要任务在 `ndo_add_slave` 对应的 `br_add_slave` 方法中完成，调用链为

```c
br_netdev_ops->ndo_add_slave = br_add_slave
    --> br_add_if
        --> br_get_rx_handler
        --> netdev_rx_handler_register
```

最终来到了 `netdev_rx_handler_register` 中，将从设备的 `dev->rx_handler` 实例赋值为 `br_handle_frame` 函数

```c
int netdev_rx_handler_register(struct net_device *dev,
			       rx_handler_func_t *rx_handler,
			       void *rx_handler_data)
{
	if (netdev_is_rx_handler_busy(dev))
		return -EBUSY;

	if (dev->priv_flags & IFF_NO_RX_HANDLER)
		return -EINVAL;

	/* Note: rx_handler_data must be set before rx_handler */
	rcu_assign_pointer(dev->rx_handler_data, rx_handler_data);
	rcu_assign_pointer(dev->rx_handler, rx_handler);

	return 0;
}
EXPORT_SYMBOL_GPL(netdev_rx_handler_register);
```
   
### 2.4 网桥与网络收发包路径

之所以强调了上一步的 `dev->rx_handler`，是因为在收包的关键处理函数 `__netif_receive_skb_core` 中，对于附加在网桥上的**从属设备**而言，会获取并调用 `skb->dev->rx_handler` ，也就是 `br_add_if` 时设置的 `br_handle_frame`

```c
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
				    struct packet_type **ppt_prev)
{
    // ...
    rx_handler = rcu_dereference(skb->dev->rx_handler);
	if (rx_handler) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		switch (rx_handler(&skb)) {
		case RX_HANDLER_CONSUMED:
			ret = NET_RX_SUCCESS;
			goto out;
		case RX_HANDLER_ANOTHER:
			goto another_round;
		case RX_HANDLER_EXACT:
			deliver_exact = true;
		case RX_HANDLER_PASS:
			break;
		default:
			BUG();
		}
	}
    // ...
}
```

反之，如果不是附加在网桥上的 net_device，就不存在 `dev->rx_handler`，也就不会经历上述流程。

接下来就探究 `br_handle_frame` 究竟做了什么。

首先通过 perf/funcgraph 等工具，可以查看 `br_handle_frame` 的调用链，大致如下

```text
br_handle_frame
    --> br_handle_frame_finish
        --> br_forward
            --> __br_forward
                --> br_forward_finish
                    --> br_dev_queue_push_xmit
                        --> dev_queue_xmit

...
br_handle_frame
    --> br_handle_frame_finish
        --> br_flood
            --> __br_forward
                --> br_forward_finish
                    --> br_dev_queue_push_xmit
                        --> dev_queue_xmit
            --> __br_forward
                --> br_forward_finish
                    --> br_dev_queue_push_xmit
                        --> dev_queue_xmit
            --> __br_forward
                --> br_forward_finish
                    --> br_dev_queue_push_xmit
                        --> dev_queue_xmit

...
br_handle_frame
    --> br_handle_frame_finish
        --> br_pass_frame_up
            --> br_netif_receive_skb
                --> netif_receive_skb
                    --> netif_receive_skb_internal
                        --> __netif_receive_skb_list
                            --> __netif_receive_skb_list_core
                                --> __netif_receive_skb_core
                                    --> 
```

在 `br_handle_frame_finish` 中，有四条处理数据包的路径，也是网桥的几个**核心能力**，分别是：

1. br_forward
2. br_flood
3. br_multicast_flood
4. br_pass_frame_up

其中最后一条路径是处理发向本地的数据包，不是发往本地的数据包，若能在 fdb 中找到对应的表项，则调用 br_forward，否则进行洪泛，通过一些标志位的判断走不通的路径。

#### 2.4.1 br_forward

```c
void br_forward(const struct net_bridge_port *to,
		struct sk_buff *skb, bool local_rcv, bool local_orig)
{
	if (unlikely(!to))
		goto out;

	/* redirect to backup link if the destination port is down */
	if (rcu_access_pointer(to->backup_port) && !netif_carrier_ok(to->dev)) {
		struct net_bridge_port *backup_port;

		backup_port = rcu_dereference(to->backup_port);
		if (unlikely(!backup_port))
			goto out;
		to = backup_port;
	}

	if (should_deliver(to, skb)) {
		if (local_rcv)
			deliver_clone(to, skb, local_orig);
		else
			__br_forward(to, skb, local_orig);
		return;
	}

out:
	if (!local_rcv)
		kfree_skb(skb);
}
EXPORT_SYMBOL_GPL(br_forward);
```

上面仅仅是针对 `__br_forward` 的一层封装，关键点在于下面这个函数。首先需要思考：什么是转发？

转发是将数据包从一个设备发送到另一个设备，然后触发网络设备的回调函数，剩下的流程 Linux 已经为我们安排好了，因此**转发**最关键的操作，就是将 `skb->dev` 设置为目标设备，然后通过 `dev_queue_xmit` 交给设备驱动来处理。

```c
static void __br_forward(const struct net_bridge_port *to,
			 struct sk_buff *skb, bool local_orig)
{
    // ...
    // !important
	indev = skb->dev;
	skb->dev = to->dev;
    // ...

	NF_HOOK(NFPROTO_BRIDGE, br_hook,
		net, NULL, skb, indev, skb->dev,
		br_forward_finish);
}
```

#### 2.4.2 br_flood 

```c
void br_flood(struct net_bridge *br, struct sk_buff *skb,
	      enum br_pkt_type pkt_type, bool local_rcv, bool local_orig)
{
	struct net_bridge_port *prev = NULL;
	struct net_bridge_port *p;

	list_for_each_entry_rcu(p, &br->port_list, list) {
		/* Do not flood unicast traffic to ports that turn it off, nor
		 * other traffic if flood off, except for traffic we originate
		 */
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

		/* Do not flood to ports that enable proxy ARP */
		if (p->flags & BR_PROXYARP)
			continue;
		if ((p->flags & (BR_PROXYARP_WIFI | BR_NEIGH_SUPPRESS)) &&
		    BR_INPUT_SKB_CB(skb)->proxyarp_replied)
			continue;

		prev = maybe_deliver(prev, p, skb, local_orig);
		if (IS_ERR(prev))
			goto out;
	}

	if (!prev)
		goto out;

	if (local_rcv)
		deliver_clone(prev, skb, local_orig);
	else
		__br_forward(prev, skb, local_orig);
	return;

out:
	if (!local_rcv)
		kfree_skb(skb);
}
```

该函数也会调用 `__br_forward` 进行处理，只不过遍历了 `bridge->port_list`，依次进行转发（maybe_deliver 中也调用了 __br_forward 函数）

#### 2.4.3 br_pass_frame_up

```c
static int br_pass_frame_up(struct sk_buff *skb)
{
	struct net_device *indev, *brdev = BR_INPUT_SKB_CB(skb)->brdev;

    // ....

	indev = skb->dev;
	skb->dev = brdev;
	skb = br_handle_vlan(br, NULL, vg, skb);
	if (!skb)
		return NET_RX_DROP;

	return NF_HOOK(NFPROTO_BRIDGE, NF_BR_LOCAL_IN,
		       dev_net(indev), NULL, skb, indev, NULL,
		       br_netif_receive_skb);
}
```

最后会重新调用 `netif_receive_skb` ，但此时 skb->dev 已经替换为网桥设备，网桥上没有注册rx_handler，因此不会再次进入 `br_handle_frame`，而是会调用 ptype 协议链上对应的协议处理函数进入上层处理

### 2.5 fdb 

fdb 全称为 forwarding database，也就是维护 MAC 地址到设备端口之间的映射数据表。用于辅助 forward，flood，multicast 等核心功能。

别看它名字很高大上，其实原理非常简答，就是通过增删改查操作维护这张映射表，同时通过一些过期策略维护数据的时效性。这里不多赘述。