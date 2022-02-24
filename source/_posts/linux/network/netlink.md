---
title: netlink
categories: [linux, network, netlink, ipc]
date: 2022-2-22 10:12:45
---

## 1. netlink 简介

### 1.1 netlink 是什么

netlink 是一种 进程间通信（inter process communication IPC） 机制，为用户空间和内核空间进程之间（当然也可以是两个内核进程之间）提供了一种双向异步的通信方式。

netlink 依托成熟的 socket api 提供服务，增加了 AF_NETLINK 这个协议簇，依靠 sock_ops 对应的各种回调函数（sendmsg，recvmsg）来实现 netlink 的功能。

### 1.2 netlink 优点

相比如其他的 IPC 方式，netlink 有下面几点优势：

1. 不需要通过 poll 操作来获取数据，像普通 socket 编程一样 recvmsg 即可，接口简单
2. 全双工异步通信
3. netlink 支持广播和多播

### 1.3 netlink 示例

前面说到，netlink 通过 socket 向用户提供操作接口，具体应该怎么用呢？下面是其工作示意图

![netlink api](/images/netlink-api.png)

## 2. netlink internal

### 2.1 netlink api

前面说到，可以用 socket api 来操控 netlink，我们就从 `socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)` 的几个参数入手

1. AF_NETLINK，指定该 socket 的协议簇为 NETLINK，这样能通过 `netlink_family_ops.create` 回调函数创建 socket 的内部结构

```c
static const struct net_proto_family netlink_family_ops = {
    .family = PF_NETLINK,
    .create = netlink_create,
    .owner    = THIS_MODULE,    /* for consistency 8) */
};
```

2. SOCK_RAW，与主题无关，暂不讨论
3. NRETLINK_ROUTE，指定 NETLINK 的类型，因为 NETLINK 在内核中被各个模块使用，所以需要通过这个字段区分究竟是哪个模块，而 netlink socket 相当于一个多路复用器，通过在内核中保存的一个针对不同 protocol 的 netlink_table，查表就能获取到对应的回调函数。

 ```c
struct netlink_table *nl_table __read_mostly;
```

```c
static int netlink_create(struct net *net, struct socket *sock, int protocol,
              int kern)
{
    // 1. 查表
    netlink_lock_table();
    if (nl_table[protocol].registered &&
        try_module_get(nl_table[protocol].module))
        module = nl_table[protocol].module;
    else
        err = -EPROTONOSUPPORT;
    cb_mutex = nl_table[protocol].cb_mutex;
    bind = nl_table[protocol].bind;
    unbind = nl_table[protocol].unbind;
    netlink_unlock_table();

    // 2. 赋值
    nlk = nlk_sk(sock->sk);
    nlk->module = module;
    nlk->netlink_bind = bind;
    nlk->netlink_unbind = unbind;
}
```

### 2.2 netlink's protocols

下面来看一下 netlink 处理不同 protocol 的方式

netlink 通过 `netlink_kernel_create` 来初始化一种 protocol，该函数接收三个参数：

1. net：network namespace
2. unit：也就是将 protocol 作为 netlink_table 的下标，
3. cfg：配置选项，提供 input/bind/unbind 等回调函数，以及别的一些配置信息，保存到 nl_table 中

也就是说，netlink 将 protocol 作为 nl_table 的下标，不同的 protocol 提供不同的 cfg 即可，这是一种典型的**策略模式**，下面分别是 NETLINK_ROUTE 和 NETLINK_KOBJECT_UEVENT 两种 protocol 对应的初始化函数，so easy

```c
static int __net_init rtnetlink_net_init(struct net *net)
{
    struct sock *sk;
    struct netlink_kernel_cfg cfg = {
        .groups        = RTNLGRP_MAX,
        .input        = rtnetlink_rcv,
        .cb_mutex    = &rtnl_mutex,
        .flags        = NL_CFG_F_NONROOT_RECV,
        .bind        = rtnetlink_bind,
    };

    sk = netlink_kernel_create(net, NETLINK_ROUTE, &cfg);
    if (!sk)
        return -ENOMEM;
    net->rtnl = sk;
    return 0;
}
```

```c
static int uevent_net_init(struct net *net)
{
    struct uevent_sock *ue_sk;
    struct netlink_kernel_cfg cfg = {
        .groups    = 1,
        .input = uevent_net_rcv,
        .flags    = NL_CFG_F_NONROOT_RECV
    };
    // ...

    ue_sk->sk = netlink_kernel_create(net, NETLINK_KOBJECT_UEVENT, &cfg);
    
    // ...
}
```

### 2.3 deeper into netlink's api

前面提到，通过 socket api 可以直接操作 netlink，那具体到每种操作，对应怎样的内核执行路径呢？下面来进行探究

1. sendmsg 系统调用

```text
___sys_sendmsg
    --> sock_sendmsg
        --> sock->ops->sendmsg(sock, msg, msg_data_left(msg)) = netlink_sendmsg
            --> netlink_unicast 
                --> netlink_unicast_kernel
                    --> nlk->netlink_rcv = cfg->input
                        --> rtnetlink_rcv
```

sys_sendmsg 最终会调用到 `sock_ops->sendmsg`，从而触发 AF_NETLINK 对应的 `netlink_sendmsg` 回调函数。

前面也说到，netlink 可以对应单播或者广播，这里以 unicast，也就是单播的执行路径为例，最终触发了 `cfg->input` 中传入的 rev 函数，对于 rtnetlink 而言，就会调用 `rtnetlink_rcv`。 

```text
rtnetlink_rcv
    --> netlink_rcv_skb
        --> rtnetlink_rcv_msg
            --> rtnl_get_link(family, type)
                --> netlink_dump_start(rtnl, skb, nlh, &c)
                    --> netlink_lookup
                    --> netlink_dump
                        --> alloc_skb
                        --> skb_reserve
                        --> netlink_skb_set_owner_r
                        --> nlk->dump_done_errno = cb->dump(skb, cb) // 执行 rtnl_dump_ifinfo/rtnl_dump_all 回调函数
                        --> __netlink_sendskb(sk, skb)
                            --> netlink_deliver_tap(sock_net(sk), skb)
                            --> skb_queue_tail(&sk->sk_receive_queue, skb) //将skb加入到用户态 sock的 sk_receive_queue 队列
                            --> sk->sk_data_ready(sk) = sock_def_readable // 唤醒 netlink_recvmsg 进行接收
                               -->  wake_up_interruptible_sync_poll
```

2. recvmsg 系统调用

```text
__sys_recvmmsg
    -->___sys_recvmsg
        --> sock->ops->recvmsg(socket, msg, msg_data_left(msg))
            --> netlink_recvmsg
                --> skb_recv_datagram
                    --> __skb_recv_datagram
                        --> __skb_try_recv_datagram or __skb_wait_for_more_packets
                            --> __skb_try_recv_from_queue // 从 sk->sk_receive_queue 中取出 skb
```

在 rtnl_get_link 这一步，会根据传入的下标取出 `rtnl_msg_handlers` 数组中的 link，从而执行 `link->doit` 回调函数

而该数组中的内容是系统初始化时就设置好的，在 `rtnetlink_init` 中，通过 `rtnl_register` 初始化了一系列操作对应的处理函数

```c
void __init rtnetlink_init(void)
{
    if (register_pernet_subsys(&rtnetlink_net_ops))
        panic("rtnetlink_init: cannot initialize rtnetlink\n");

    register_netdevice_notifier(&rtnetlink_dev_notifier);

    rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_getlink,
              rtnl_dump_ifinfo, 0);
    rtnl_register(PF_UNSPEC, RTM_SETLINK, rtnl_setlink, NULL, 0);
    rtnl_register(PF_UNSPEC, RTM_NEWLINK, rtnl_newlink, NULL, 0);
    rtnl_register(PF_UNSPEC, RTM_DELLINK, rtnl_dellink, NULL, 0);

    rtnl_register(PF_UNSPEC, RTM_GETADDR, NULL, rtnl_dump_all, 0);
    rtnl_register(PF_UNSPEC, RTM_GETROUTE, NULL, rtnl_dump_all, 0);
    rtnl_register(PF_UNSPEC, RTM_GETNETCONF, NULL, rtnl_dump_all, 0);

    rtnl_register(PF_UNSPEC, RTM_NEWLINKPROP, rtnl_newlinkprop, NULL, 0);
    rtnl_register(PF_UNSPEC, RTM_DELLINKPROP, rtnl_dellinkprop, NULL, 0);

    rtnl_register(PF_BRIDGE, RTM_NEWNEIGH, rtnl_fdb_add, NULL, 0);
    rtnl_register(PF_BRIDGE, RTM_DELNEIGH, rtnl_fdb_del, NULL, 0);
    rtnl_register(PF_BRIDGE, RTM_GETNEIGH, rtnl_fdb_get, rtnl_fdb_dump, 0);

    rtnl_register(PF_BRIDGE, RTM_GETLINK, NULL, rtnl_bridge_getlink, 0);
    rtnl_register(PF_BRIDGE, RTM_DELLINK, rtnl_bridge_dellink, NULL, 0);
    rtnl_register(PF_BRIDGE, RTM_SETLINK, rtnl_bridge_setlink, NULL, 0);

    rtnl_register(PF_UNSPEC, RTM_GETSTATS, rtnl_stats_get, rtnl_stats_dump,
              0);
}
```

`rtnl_register` 函数原型为:

```c
void rtnl_register(int protocol, int msgtype,
           rtnl_doit_func doit, rtnl_dumpit_func dumpit,
           unsigned int flags);
```

几个参数的含义分别为：

1. protocol：具体的协议
2. msgtype：消息类型，后面会提到
3. doit：注册到 rtnl_msg_handlers 中的 link->doit
4. dumpit：注册到 rtnl_msg_handlers 中的 link->dumpit
5. flags：doit/dumpit 的参数

rtnetlink 字面意思是 `route table netlink`，其实它包含的范围更大，邻居子系统也是通过 rtnetlink 暴露出去的，都可以从上面的 msgtype 参数可以看出来。

这样解释还是有一些抽象，但其实 Linux 中一些网络相关的常用指令都是以 rtnetlink 为基础，就比如 `ip addr show`

这条指令，会显示所有网络设备的地址信息，背后就是调用了 msgtype 为 RTM_GETADDR 的 `rtnl_dump_all` ，通过 [funcgraph](https://github.com/brendangregg/perf-tools/blob/master/kernel/funcgraph) 可以查看详细的内核函数调用链，从而验证这一点。

事实上，NETLINK_ROUTE 可以按照下面进行划分：

1. LINK (network interfaces)
2. ADDR (network addresses)
3. ROUTE (network messages)
4. NEIGH (neighbouring subsystem messages)
5. RULE (policy rouing rules)
6. QDISC (queueing disciplines)
7. TCLASS (traffic classes)
8. ACTIOn (packet action api)
9. NEIGHTBL (neighbouring table)
10. ADDRLABEL (address labeling)

不仅是路由子系统，几乎所有 L3 和 L2 的功能，都通过 rtnetlink 暴露出去

## 3. generic netlink protocol

netlink 协议簇数最大 32 个（MAX_LINKS），为支持更多的协议簇，开发了通用 netlink 簇 NETLINK_GENERIC。generic netlink 以 netlink 协议为基础，多做了一层多路复用，从而能够支持更多的子系统（netlink 本身已经完成了一次多路复用）。

同理，generic netlink 也需要通过 `netlink_kernel_create` 初始化：

```c
static int __net_init genl_pernet_init(struct net *net)
{
    struct netlink_kernel_cfg cfg = {
        .input        = genl_rcv,
        .flags        = NL_CFG_F_NONROOT_RECV,
    };

    net->genl_sock = netlink_kernel_create(net, NETLINK_GENERIC, &cfg);

    if (!net->genl_sock && net_eq(net, &init_net))
        panic("GENL: Cannot initialize generic netlink\n");

    if (!net->genl_sock)
        return -ENOMEM;

    return 0;
}
```

也就是说，generic netlink socket 的 `nlk->netlink_rcv` 会对应 `genl_rcv`，在这个函数中，主要是查找对应的 generic family，然后调用其对应的 `doit` 回调函数。

这一点其实不难理解，netlink 本身支持多种协议簇，所以维护了一张 nl_table，保存了所有协议对应的处理方式，以 protocol 作为下标，取出对应的协议回调函数即可，这是第一层**多路复用**。

而 generic netlink 在这个基础上进一步维护了多路复用，同理也是维护一张表格，通过二级协议类型 `genl_family` 作为下标，道理其实非常相似。

genl_family 可以通过 genl_register_family/genl_unregister_family 进行注册和注销。

只不过有一点不大一样，genl_family 用到了 `struct idr` 这个结构，相比数组而言更加灵活，这里不过多赘述。
