# Uevent

> Uevent是Kobject的一部分，负责通知用户空间Kobject的状态变化，用户控件根据变化做出相应的处理。

## Uevent事件

```c
/*
 * The actions here must match the index to the string array
 * in lib/kobject_uevent.c
 *
 * Do not add new actions here without checking with the driver-core
 * maintainers. Action strings are not meant to express subsystem
 * or device specific properties. In most cases you want to send a
 * kobject_uevent_env(kobj, KOBJ_CHANGE, env) with additional event
 * specific variables added to the event environment.
 */
enum kobject_action {
	KOBJ_ADD,
	KOBJ_REMOVE,
	KOBJ_CHANGE,
	KOBJ_MOVE,
	KOBJ_ONLINE,
	KOBJ_OFFLINE,
	KOBJ_MAX
};
```

## kobj_uevent_env

用于此次时间上报的环境变量

```c
#define UEVENT_HELPER_PATH_LEN		256
#define UEVENT_NUM_ENVP			32	/* number of env pointers */
#define UEVENT_BUFFER_SIZE		2048	/* buffer for the variables */
 
struct kobj_uevent_env {
	char *argv[3];
	char *envp[UEVENT_NUM_ENVP];
	int envp_idx;
	char buf[UEVENT_BUFFER_SIZE];
	int buflen;
};
```

- envp：用于保存环境变量的地址
- envp_idx：用于访问envp的index
- buf：保存环境变量的buffer
- buflen：访问buf的变量

## kset_uevent_ops

```c
struct kset_uevent_ops {
	int (* const filter)(struct kset *kset, struct kobject *kobj);
	const char *(* const name)(struct kset *kset, struct kobject *kobj);
	int (* const uevent)(struct kset *kset, struct kobject *kobj,
		      struct kobj_uevent_env *env);
};
```

- `filter`：kset通过filter过滤掉不希望kobject上报的event
- `name`：返回kset的name，如果kset没有合法的name，那么这个kset下的所有kobject将不允许上报uevent
- `uevent`：kset为kobject同意添加环境变量

## kobject_uevent_env()

默认使用Kmod上传用户事件，如果定义了`CONFIG_NET`关键字，则使用Netlink上报事件。

```c
 /**
 * kobject_uevent_env - send an uevent with environmental data
 *
 * @action: action that is happening
 * @kobj: struct kobject that the action is happening to
 * @envp_ext: pointer to environmental data
 *
 * Returns 0 if kobject_uevent_env() is completed with success or the
 * corresponding error when it fails.
 */
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
		       char *envp_ext[])
{
	struct kobj_uevent_env *env;
	const char *action_string = kobject_actions[action];
	const char *devpath = NULL;
	const char *subsystem;
	struct kobject *top_kobj;
	struct kset *kset;
	const struct kset_uevent_ops *uevent_ops;
	int i = 0;
	int retval = 0;
#ifdef CONFIG_NET
	struct uevent_sock *ue_sk;
#endif
 
	pr_debug("kobject: '%s' (%p): %s\n",
		 kobject_name(kobj), kobj, __func__);
 
	/* search the kset we belong to 知道到该kobj从属的kset*/
	top_kobj = kobj;
	while (!top_kobj->kset && top_kobj->parent)
		top_kobj = top_kobj->parent;    /* 找的方法很简单,若它的kset不存在,则查找其父节点的kset是否存在,不存在则继续查找父父节点.... */
 
	if (!top_kobj->kset) {    
		pr_debug("kobject: '%s' (%p): %s: attempted to send uevent "
			 "without kset!\n", kobject_name(kobj), kobj,
			 __func__);
		return -EINVAL;   /* 最终还没找到就报错 */
	}
 
	kset = top_kobj->kset;        /* 找到与之相关的kset */
	uevent_ops = kset->uevent_ops;
 
	/* skip the event, if uevent_suppress is set*/
	if (kobj->uevent_suppress) {        /* uevent_suppress被置位,则忽略上报uevent */
		pr_debug("kobject: '%s' (%p): %s: uevent_suppress "
				 "caused the event to drop!\n",
				 kobject_name(kobj), kobj, __func__);
		return 0;
	}
	/* skip the event, if the filter returns zero. */
	if (uevent_ops && uevent_ops->filter)       /* 所属的筛选函数存在则筛选,返回0表示被筛掉了,不再上报 */
		if (!uevent_ops->filter(kset, kobj)) {
			pr_debug("kobject: '%s' (%p): %s: filter function "
				 "caused the event to drop!\n",
				 kobject_name(kobj), kobj, __func__);
			return 0;
		}
 
	/* originating subsystem */
	if (uevent_ops && uevent_ops->name)
		subsystem = uevent_ops->name(kset, kobj);    /* name函数存在,则使用kset返回的kset的name */
	else
		subsystem = kobject_name(&kset->kobj);       /* 否则用kset里kobj的name做kset的name */
	if (!subsystem) {        /* kset的name不存在,也不允许上报 */
		pr_debug("kobject: '%s' (%p): %s: unset subsystem caused the "
			 "event to drop!\n", kobject_name(kobj), kobj,
			 __func__);
		return 0;
	}
 
	/* environment buffer */
	env = kzalloc(sizeof(struct kobj_uevent_env), GFP_KERNEL);    /* 分配一个用于此次环境变量的buffer */
	if (!env)
		return -ENOMEM;
 
	/* complete object path */
	devpath = kobject_get_path(kobj, GFP_KERNEL);    /* 根据kobj得到它在sysfs中的路径 */
	if (!devpath) {
		retval = -ENOENT;
		goto exit;
	}
 
	/* default keys 添加当前要上报的行为,path,name到env的buffer中 */
	retval = add_uevent_var(env, "ACTION=%s", action_string);
	if (retval)
		goto exit;
	retval = add_uevent_var(env, "DEVPATH=%s", devpath);
	if (retval)
		goto exit;
	retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
	if (retval)
		goto exit;
 
	/* keys passed in from the caller */
	if (envp_ext) {    /* 我们自己传的外部环境变量要不为空,则解析并添加到env的buffer中 */
		for (i = 0; envp_ext[i]; i++) {
			retval = add_uevent_var(env, "%s", envp_ext[i]);
			if (retval)
				goto exit;
		}
	}
 
	/* let the kset specific function add its stuff */
	if (uevent_ops && uevent_ops->uevent) {    /* 如果uevent_ops中的uevent存在,则调用该接口发送该kobj的env */
		retval = uevent_ops->uevent(kset, kobj, env);
		if (retval) {
			pr_debug("kobject: '%s' (%p): %s: uevent() returned "
				 "%d\n", kobject_name(kobj), kobj,
				 __func__, retval);
			goto exit;
		}
	}
 
	/*
	 * Mark "add" and "remove" events in the object to ensure proper
	 * events to userspace during automatic cleanup. If the object did
	 * send an "add" event, "remove" will automatically generated by
	 * the core, if not already done by the caller.
	 */
	if (action == KOBJ_ADD)    /* 如果action是add或remove的话要更新kobj中的state */
		kobj->state_add_uevent_sent = 1;
	else if (action == KOBJ_REMOVE)
		kobj->state_remove_uevent_sent = 1;
 
	mutex_lock(&uevent_sock_mutex);
	/* we will send an event, so request a new sequence number */
    /* 每次发送一个事件,都要有它的事件号,该事件号不能重复,u64 uevent_seqnum,把它也作为环境变量添加到buffer最后面 */
	retval = add_uevent_var(env, "SEQNUM=%llu", (unsigned long long)++uevent_seqnum);
	if (retval) {
		mutex_unlock(&uevent_sock_mutex);
		goto exit;
	}
 
#if defined(CONFIG_NET)
	/* send netlink message 如果定义了"CONFIG_NET”，则使用netlink发送该uevent */
	list_for_each_entry(ue_sk, &uevent_sock_list, list) {
		struct sock *uevent_sock = ue_sk->sk;
		struct sk_buff *skb;
		size_t len;
 
		if (!netlink_has_listeners(uevent_sock, 1))
			continue;
 
		/* allocate message with the maximum possible size */
		len = strlen(action_string) + strlen(devpath) + 2;
		skb = alloc_skb(len + env->buflen, GFP_KERNEL);
		if (skb) {
			char *scratch;
 
			/* add header */
			scratch = skb_put(skb, len);
			sprintf(scratch, "%s@%s", action_string, devpath);
 
			/* copy keys to our continuous event payload buffer */
			for (i = 0; i < env->envp_idx; i++) {
				len = strlen(env->envp[i]) + 1;
				scratch = skb_put(skb, len);
				strcpy(scratch, env->envp[i]);
			}
 
			NETLINK_CB(skb).dst_group = 1;
			retval = netlink_broadcast_filtered(uevent_sock, skb,
							    0, 1, GFP_KERNEL,
							    kobj_bcast_filter,
							    kobj);
			/* ENOBUFS should be handled in userspace */
			if (retval == -ENOBUFS || retval == -ESRCH)
				retval = 0;
		} else
			retval = -ENOMEM;
	}
#endif
	mutex_unlock(&uevent_sock_mutex);
 
	/* call uevent_helper, usually only enabled during early boot */
	if (uevent_helper[0] && !kobj_usermode_filter(kobj)) {
		char *argv [3];
        
        /* 添加helper和 kset的name */
		argv [0] = uevent_helper;
		argv [1] = (char *)subsystem;
		argv [2] = NULL;
 
        /* 添加了标准环境变量 （HOME=/，PATH=/sbin:/bin:/usr/sbin:/usr/bin）*/
		retval = add_uevent_var(env, "HOME=/");
		if (retval)
			goto exit;
		retval = add_uevent_var(env,
					"PATH=/sbin:/bin:/usr/sbin:/usr/bin");
		if (retval)
			goto exit;
        /* 调用kmod模块提供的call_usermodehelper函数，上报uevent。call_usermodehelper的作用，就是fork一个进程，以uevent为参数，执行uevent_helper */
		retval = call_usermodehelper(argv[0], argv,
					     env->envp, UMH_WAIT_EXEC);
	}
 
exit:
	kfree(devpath);
	kfree(env);
	return retval;
}
```

- `env`：环境变量
- `const char *action_string = kobject_actions[action];`：将enum kobject_action枚举类型转化成字符串
- `subsystem`：kset的name，如果存在`uevent_ops->name`则用其返回值，否则用kset的name（`kobject_name(&kset->kobj)`）
- `devpath`：根据kobj获取它在sysfs下的地址
- `top_kobj`：找到kobj的从属kset，一个kobj必须从属于一个kset才能上报uevent
- `kobj->uevent_suppress`：kobj的uevent忽略置位
- `uevent_ops->filter`：kset存在`filter`，如果`filter`返回0，表示该`uevent`被筛除，不需要上报
- `add_uevent_var`：以格式化字符的形式将action、路径信息、subsystem等信息传给`env`中
- 将传入的`envp`解析并添加到`env`中
- 将`env`传给`uevnet_ops->uevent`
- `KOBJ_ADD`和`KOBJ_REMOVE`需要修改kobj的对应uevent置位
- 每个事件必须有不重复的事件号

