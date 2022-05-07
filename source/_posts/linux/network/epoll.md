---
title: epoll
categories: [linux, network, epoll]
date: 2022-3-26 10:05:00
---

## 1. 介绍

关于 epoll 的分析，网上的分析多如牛毛，也不乏高质量的文章，因此这里抛砖引玉，读者可先耐心看完下面几篇博客，再来探讨后面的问题。

- [深入揭秘 epoll 是如何实现 IO 多路复用的！](https://zhuanlan.zhihu.com/p/361750240)
- [epoll的原理和实现](https://tqr.ink/2017/10/05/implementation-of-epoll/)
- [linux网络编程之epoll源码重要部分详解](https://zhuanlan.zhihu.com/p/147549069)

优先级从高到低

## 2. epoll 编程模型

有下面几个要点需要掌握

1. LT VS ET 触发方式
2. PingPong 模式
3. 错误处理

### 2.1 epoll 触发方式

大家都说 edge trigger 是边缘触发，发生的事件只会触发一次，level trigger 是水平触发，发生的事件会一直触发。背后的原理是什么？

这得从 epoll_wait 说起

epoll_wait 的调用路径为

```text
sys_epoll_wait
    --> do_epoll_wait
        --> ep_poll
            --> schedule_hrtimeout_range (context switch)
            --> ep_send_events
                --> ep_scan_ready_list
                    --> ep_send_events_proc
```

`ep_scan_ready_list` 函数中，将 eventpoll 结构中的 rdllist 取了出来，放到一个链表中，交给了 `ep_send_events_proc` 进行处理，将就绪的 epoll_item 发送到用户态。

```c
static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
			       void *priv)
{
	// ...

	list_for_each_entry_safe(epi, tmp, head, rdllink) {
		if (esed->res >= esed->maxevents)
			break;
		ws = ep_wakeup_source(epi);
		if (ws) {
			if (ws->active)
				__pm_stay_awake(ep->ws);
			__pm_relax(ws);
		}

		list_del_init(&epi->rdllink);

		revents = ep_item_poll(epi, &pt, 1);
		if (!revents)
			continue;

		if (__put_user(revents, &uevent->events) ||
		    __put_user(epi->event.data, &uevent->data)) {
			list_add(&epi->rdllink, head);
			ep_pm_stay_awake(epi);
			if (!esed->res)
				esed->res = -EFAULT;
			return 0;
		}
		esed->res++;
		uevent++;
		if (epi->event.events & EPOLLONESHOT)
			epi->event.events &= EP_PRIVATE_BITS;
		else if (!(epi->event.events & EPOLLET)) {
			/*
			 * If this file has been added with Level
			 * Trigger mode, we need to insert back inside
			 * the ready list, so that the next call to
			 * epoll_wait() will check again the events
			 * availability. At this point, no one can insert
			 * into ep->rdllist besides us. The epoll_ctl()
			 * callers are locked out by
			 * ep_scan_ready_list() holding "mtx" and the
			 * poll callback will queue them in ep->ovflist.
			 */
			list_add_tail(&epi->rdllink, &ep->rdllist);
			ep_pm_stay_awake(epi);
		}
	}

	return 0;
}
```

这个函数有两点需要注意：
1. 遍历就绪链表时，若 `ep_item_poll` 返回 0，直接跳过该结点
2. 当 epoll_item 的 events 不为 EPOLLET 时，将当前结点重新插入 rdllist

大家对于 LT 和 ET 的误区就在此：rdllist 中的结点并不一定有 EPOLLIN/EPOLLOUT 等事件发生，只是需要在下一次 epoll_wait 调用中，去检查一下是否有事件发生。对于默认的 EPOLLLT 来说，一旦发生了事件，就会在每次 epoll_wait 过程中重复检查，直到某一次没有任何事件发生。

而针对每一个 socket，具体发生了什么事件由 `ep_item_poll` 来决定，最终会在 `tcp_poll` 中，根据 sock 的状态来设置。

```text
ep_item_poll
    --> vfs_poll
        --> tcp_poll
```

```c
__poll_t tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
	__poll_t mask;
	struct sock *sk = sock->sk;
	const struct tcp_sock *tp = tcp_sk(sk);
	int state;

	sock_poll_wait(file, sock, wait);

	state = inet_sk_state_load(sk);
	if (state == TCP_LISTEN)
		return inet_csk_listen_poll(sk);

	mask = 0;

	if (sk->sk_shutdown == SHUTDOWN_MASK || state == TCP_CLOSE)
		mask |= EPOLLHUP;
	if (sk->sk_shutdown & RCV_SHUTDOWN)
		mask |= EPOLLIN | EPOLLRDNORM | EPOLLRDHUP;

	if (state != TCP_SYN_SENT &&
	    (state != TCP_SYN_RECV || rcu_access_pointer(tp->fastopen_rsk))) {
		int target = sock_rcvlowat(sk, 0, INT_MAX);

		if (READ_ONCE(tp->urg_seq) == READ_ONCE(tp->copied_seq) &&
		    !sock_flag(sk, SOCK_URGINLINE) &&
		    tp->urg_data)
			target++;

		if (tcp_stream_is_readable(tp, target, sk))
			mask |= EPOLLIN | EPOLLRDNORM;

		if (!(sk->sk_shutdown & SEND_SHUTDOWN)) {
			if (__sk_stream_is_writeable(sk, 1)) {
				mask |= EPOLLOUT | EPOLLWRNORM;
			} else {  /* send SIGIO later */
				sk_set_bit(SOCKWQ_ASYNC_NOSPACE, sk);
				set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
				smp_mb__after_atomic();
				if (__sk_stream_is_writeable(sk, 1))
					mask |= EPOLLOUT | EPOLLWRNORM;
			}
		} else
			mask |= EPOLLOUT | EPOLLWRNORM;

		if (tp->urg_data & TCP_URG_VALID)
			mask |= EPOLLPRI;
	} else if (state == TCP_SYN_SENT && inet_sk(sk)->defer_connect) {
		mask |= EPOLLOUT | EPOLLWRNORM;
	}
	/* This barrier is coupled with smp_wmb() in tcp_reset() */
	smp_rmb();
	if (sk->sk_err || !skb_queue_empty_lockless(&sk->sk_error_queue))
		mask |= EPOLLERR;

	return mask;
}
EXPORT_SYMBOL(tcp_poll);
```