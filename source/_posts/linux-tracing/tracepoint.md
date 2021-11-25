---
title: tracepoint
---



# TraceEvent





## 1. TraceEvent 是什么



要搞明白 TraceEvent，首先要去了解一下 Tracepoint。



### 1.1 tracepoint 

Tracepoint 是内核中一种探测的机制。放置在代码中的 tracepoint（跟踪点）提供了一个钩子来调用您可以在运行时提供的函数（probe）。 tracepoint 可以是 "on"（开启）或 "off"（关闭）。 当跟踪点关闭时，它没有任何影响，除了添加一个微小的时间损失（检查分支条件）和空间损失（函数调用的指令所占空间）。 当 tracepoint 处于 "on" 状态时，每次执行 tracepoint 时都会在调用者的执行上下文中调用您提供的函数。 当提供的函数结束其执行时，直接返回给调用者。



简单来说，可以在一些关键函数中增加一个 tracepoint 调用，在这个调用中，会去检查是否开启了该 tracepoint，如果开启，才回去执行上面注册的所有回调函数。



再概括来说，tracepoint 也是基于 **注册 + 回调** 的思想，类似于 **发布者/订阅者** 模式，如果需要在某个函数上执行 trace 逻辑，就需要将回调函数注册上去，并且保证该 tracepoint 是开启状态。

和 kprobe 这种运行期动态修改指令的方式不同，tracepoint 是静态的，保存开启状态+回调函数，就可以轻松地完成 trace 的功能。



### 1.2 tracepoint 的实现



tracepoint 的原理并不是那么复杂，但是为了方便开发者使用，内核大神们可是动用了各种**宏**里面的黑魔法



首先提供了一个**声明trace**  的宏 `DECLARE_TRACE` ，这个宏其实是给别的几个宏包装了一下，核心在于 `__DECLARE_TRACE`

```c
#define DECLARE_TRACE(name, proto, args)				\
	__DECLARE_TRACE(name, PARAMS(proto), PARAMS(args),		\
			cpu_online(raw_smp_processor_id()),		\
			PARAMS(void *__data, proto),			\
			PARAMS(__data, args))
```



在 `__DECLARE_TRACE` 中，主要是按照注册的 tracepoint 名字，添加了几个函数，通过名字可以大致推测出其功能，大致有：

1. do action                        `trace##name`
2. register/unregister       `register_trace_##name`/`unregister_trace_##name`
3. enable/disable              `trace_##name##_enabled`



因为大部分都是重复的函数，所以用一个宏来帮开发者解决了这个问题，只不过对于内核开发人员，以及我们这些想要通过源码一探究竟的好奇宝宝来说，就不是那么友好了

```c
#define __DECLARE_TRACE(name, proto, args, cond, data_proto, data_args) \
	extern int __traceiter_##name(data_proto);			\
	DECLARE_STATIC_CALL(tp_func_##name, __traceiter_##name);	\
	extern struct tracepoint __tracepoint_##name;			\
	static inline void trace_##name(proto)				\
	{								\	
		if (static_key_false(&__tracepoint_##name.key))		\
			__DO_TRACE(name,				\
				TP_PROTO(data_proto),			\
				TP_ARGS(data_args),			\
				TP_CONDITION(cond), 0);			\
		if (IS_ENABLED(CONFIG_LOCKDEP) && (cond)) {		\
			rcu_read_lock_sched_notrace();			\
			rcu_dereference_sched(__tracepoint_##name.funcs);\
			rcu_read_unlock_sched_notrace();		\
		}							\
	}								\
	__DECLARE_TRACE_RCU(name, PARAMS(proto), PARAMS(args),		\
		PARAMS(cond), PARAMS(data_proto), PARAMS(data_args))	\
	static inline int						\
	register_trace_##name(void (*probe)(data_proto), void *data)	\
	{								\
		return tracepoint_probe_register(&__tracepoint_##name,	\
						(void *)probe, data);	\
	}								\
	static inline int						\
	register_trace_prio_##name(void (*probe)(data_proto), void *data,\
				   int prio)				\
	{								\
		return tracepoint_probe_register_prio(&__tracepoint_##name, \
					      (void *)probe, data, prio); \
	}								\
	static inline int						\
	unregister_trace_##name(void (*probe)(data_proto), void *data)	\
	{								\
		return tracepoint_probe_unregister(&__tracepoint_##name,\
						(void *)probe, data);	\
	}								\
	static inline void						\
	check_trace_callback_type_##name(void (*cb)(data_proto))	\
	{								\
	}								\
	static inline bool						\
	trace_##name##_enabled(void)					\
	{								\
		return static_key_false(&__tracepoint_##name.key);	\
	}
```



上面提到过，tracepoint 是有状态的，需要记录是否开启，以及所有注册的回调函数，这些信息都记录在 `struct tracepoint ` 当中

对应到上面的各个难懂的宏，每个字段的功能分为是：

1. key 用来记录 tracepoint 是否开启
2. regfunc 也就是注册回调函数的方法
3. unregfunc 也就是取消注册的方法

```c
struct tracepoint {
	const char *name;		/* Tracepoint name */
	struct static_key key;
	struct static_call_key *static_call_key;
	void *static_call_tramp;
	void *iterator;
	int (*regfunc)(void);
	void (*unregfunc)(void);
	struct tracepoint_func __rcu *funcs;
};
```



再来看一下系统启动的时候，tracepoint 是如何注册到内核当中的。



Todo!()

