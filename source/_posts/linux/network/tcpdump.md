---
title: tcpdump 原理
categories: [linux, network, tcpdump, af_packet]
date: 2022-2-25 9:40:45
---

## 1. tcpdump 简介

tcpdump 类似于 wireshark，是常用的一款抓包工具，其是如何抓到内核态的网络包的呢？如果让你写一个抓包程序，你能完成吗？

## 2. tcpdump internal

### 2.1 tcpdump 抓包点

首先要明确，tcpdump 抓包的入口在何处，这里先讨论收包这种情况：

从软中断到tcp/ip协议栈的收包路径为

```text
net_rx_action
    --> napi_poll
        --> napi_struct->poll
        --> napi_complete_done
            --> netif_receive_skb_list_internal
                --> __netif_receive_skb_list_core
                    --> __netif_receive_skb_core
                    --> ip_list_rcv
```

其中，napi_xxx 是 napi 驱动相关的操作，netif 是设备层面的操作，到了 ip_list_rcv，就要逐步交给上层协议栈去处理了。

tcpdump 的作用位置正是位于 `__netif_receive_skb_core`

```c
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
				    struct packet_type **ppt_prev)
{
	list_for_each_entry_rcu(ptype, &ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}

	list_for_each_entry_rcu(ptype, &skb->dev->ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}
    // ...
}
```

该函数中，遍历了 ptype_all/dev->ptype_all，并且针对其中的每一个 ptype，都会调用 `deliver_skb` 处理。

```c
// include/linux/netdevice.h
struct packet_type {
	__be16			type;	/* This is really htons(ether_type). */
	bool			ignore_outgoing;
	struct net_device	*dev;	/* NULL is wildcarded here	     */
	int			(*func) (struct sk_buff *,
					 struct net_device *,
					 struct packet_type *,
					 struct net_device *);
	void			(*list_func) (struct list_head *,
					      struct packet_type *,
					      struct net_device *);
	bool			(*id_match)(struct packet_type *ptype,
					    struct sock *sk);
	void			*af_packet_priv;
	struct list_head	list;
};

//net/core/dev.c
static inline int deliver_skb(struct sk_buff *skb,
			      struct packet_type *pt_prev,
			      struct net_device *orig_dev)
{
	// ...
	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```

而在 `deliver_skb` 中，直接调用了 `packet_type->func` 进行回调，从而触发 tcpdump 事先设置好的回调函数。

### 2.2 tcpdump 与 ptype_all

上面提到，Linux 在网络收包路径中，已经为我们提供了 hook 点：`ptype_all`，只需要提前注册抓包相关的处理函数，就可以通过回调完成 tcpdump 的基本功能了。那么，就下来就来研究一下 tcpdump 是如何和 ptype_all 关联的。

通过 `trace tcpdump -i eth0` 来查看 tcpdump 的内核函数调用链，发现其源头在于 `socket(AF_PACKET, SOCK_RAW, xxx)`

对于 socket api 有过了解的同学可能知道，socket 相当于一个多路复用器，该函数中的第一个参数 AF_PACKET 表示 packet 类型的协议族，对应注册到系统中的 `packet_family_ops` 

```c
static const struct net_proto_family packet_family_ops = {
	.family =	PF_PACKET,
	.create =	packet_create,
	.owner	=	THIS_MODULE,
};
```

在 socket 对应的系统调用中，会取出该 net_proto_family，并调用 `pf->create()` 来完成 socket 的创建，从而完成该特定类型 sock 的初始化工作

```c
int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	if (family < 0 || family >= NPROTO)
		return -EAFNOSUPPORT;
	if (type < 0 || type >= SOCK_MAX)
		return -EINVAL;

	sock = sock_alloc();
    
    // ...
	pf = rcu_dereference(net_families[family]);
	err = pf->create(net, sock, protocol, kern);
	// ...
}
```

由上可知，AF_PACKET 的关键在于 `packet_create`

```c
static int packet_create(struct net *net, struct socket *sock, int protocol,
			 int kern)
{
    // ...
	po->prot_hook.func = packet_rcv;

	if (sock->type == SOCK_PACKET)
		po->prot_hook.func = packet_rcv_spkt;

	po->prot_hook.af_packet_priv = sk;

	if (proto) {
		po->prot_hook.type = proto;
		__register_prot_hook(sk);
	}

    // ...
}

static void __register_prot_hook(struct sock *sk)
{
	struct packet_sock *po = pkt_sk(sk);

	if (!po->running) {
		if (po->fanout)
			__fanout_link(sk, po);
		else
			dev_add_pack(&po->prot_hook);

		sock_hold(sk);
		po->running = 1;
	}
}
```

该函数中将 `prot_hook.func` 设置为 `packet_rcv`，并且通过 `__register_prot_hook`，将 prot_hook 注册到了 dev 的 ptype_all 链表中。这样，在遍历 dev->ptype_all 链表时，就可以找到 `packet_rcv` 并执行了。

```c
static inline struct list_head *ptype_head(const struct packet_type *pt)
{
	if (pt->type == htons(ETH_P_ALL))
		return pt->dev ? &pt->dev->ptype_all : &ptype_all;
	else
		return pt->dev ? &pt->dev->ptype_specific :
				 &ptype_base[ntohs(pt->type) & PTYPE_HASH_MASK];
}

void dev_add_pack(struct packet_type *pt)
{
	struct list_head *head = ptype_head(pt);

	spin_lock(&ptype_lock);
	list_add_rcu(&pt->list, head);
	spin_unlock(&ptype_lock);
}
EXPORT_SYMBOL(dev_add_pack);
```

之前的解释其实并不太严谨，因为在 `ptype_head` 中，会根据 packet_type 是否设置了 dev 字段，进而判断将 packet_type 挂载到 dev->ptype_all 还是全局的 ptype_all 上。但无论是哪种情况，在 `__netif_receive_skb_core` 都会进行处理

### 2.3 发送数据包

发送数据包的情况有所不同，因为 Linux 中还存在 netfilter 用来过滤数据包，如果数据包在到达 AF_PACKET 发包抓包点之前就已经被丢弃，无论如何也采集不到该数据包的内容。

进入到设备层之后，发包的函数调用链为：

```text
dev_hard_start_xmit
	--> xmit_one
		--> dev_queue_xmit_nit
			--> deliver_skb // 遍历 ptype_all 链表，处理 AF_PACKET 回调 
		--> netdev_start_xmit
			--> __netdev_start_xmit
				--> ops->ndo_start_xmit // 网卡驱动发送数据包
```

在发送数据包之前，会在 `dev_queue_xmit_nit` 中处理 ptype_all 链表上注册的回调函数，之后的处理相信大家都已经了然于胸了。