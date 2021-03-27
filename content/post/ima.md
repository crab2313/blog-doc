+++
title = "IMA内核框架分析"
date = 2020-11-28
draft = true


tags = ["kernel", "ima"]

+++

## IMA模板管理

根据我的理解，IMA的开发过程中发现，IMA进行度量的数据是需要进行扩充的。比如说，最早的`ima`类型模板是一个固定长度的文件内容哈希值（md5/sha1）与文件路径的组合。后续IMA的开发者需要对其进行扩充，需要定义新的字段，这就设计一个管理问题。

IMA采取的方式是分离IMA模板管理代码。定义了如下两个数据结构：

* IMA模板描述符
* IMA模板字段

前者定义哪些数据需要被度量，并包含在最终的数据中。而后者则定义进行度量的数据本身（如文件路径，文件内容哈希等）。下面以源代码进行举例说明。

```c
static struct ima_template_desc builtin_templates[] = {
	{.name = IMA_TEMPLATE_IMA_NAME, .fmt = IMA_TEMPLATE_IMA_FMT},
	{.name = "ima-ng", .fmt = "d-ng|n-ng"},
	{.name = "ima-sig", .fmt = "d-ng|n-ng|sig"},
	{.name = "ima-buf", .fmt = "d-ng|n-ng|buf"},
	{.name = "ima-modsig", .fmt = "d-ng|n-ng|sig|d-modsig|modsig"},
	{.name = "", .fmt = ""},	/* placeholder for a custom format */
};
```

上面就是IMA内建的模板描述符数组，可以看到描述副定义为`struct ima_template_desc`，且有两个字段。其中name很明显就是描述符的名称，而fmt是一个字符串，由`|`分隔开来，其中每一个元素都是一个模板字段的名称，表示该模板描述符定义的数据格式。例如，IMA默认的描述副`ima-ng`的格式就是`d-ng|n-ng`，分别对应为任意算法哈希值与任意长度事件名称（事件这一说法源自TPM中的event log，在IMA的语境下大部分情况都是文件名称）。

而模板字段的定义如下：

```c
/* IMA template field definition */
struct ima_template_field {
	const char field_id[IMA_TEMPLATE_FIELD_ID_MAX_LEN];
	int (*field_init)(struct ima_event_data *event_data,
			  struct ima_field_data *field_data);
	void (*field_show)(struct seq_file *m, enum ima_show_type show,
			   struct ima_field_data *field_data);
};
```

其中`field_id`就是前面看到的字段名，如`d-ng`与`n-ng`，而`field_init`与`field_show`抽象了字段的两个操作，即字段的计算与显示。以`d-ng`为例，其表示任意算法的文件内容哈希值，`field_init`函数可以根据输入的文件内容计算出文件内容哈希值，并保存到缓冲区中。`field_init`函数的输入为`ima_event_data`，涵盖了用于生成字段输出的全部信息。而`field_show`则可以从缓冲区中解析对应格式的数据，并将其打印到输出缓冲区中。

```c
struct ima_event_data {
	struct integrity_iint_cache *iint;
	struct file *file;
	const unsigned char *filename;
	struct evm_ima_xattr_data *xattr_value;
	int xattr_len;
	const struct modsig *modsig;
	const char *violation;
	const void *buf;
	int buf_len;
};
```

IMA甚至提供了更加灵活的模板描述符创建方式，可以根据`ima_template`和`ima_template_fmt`命令行参数创建或者选择其他模板。

### ima_template_entry

```c
struct ima_template_entry {
	int pcr;
	struct tpm_digest *digests;
	struct ima_template_desc *template_desc; /* template descriptor */
	u32 template_data_len;
	struct ima_field_data template_data[];	/* template related data */
};
```

这个结构体是IMA度量在内存中的表示，包括：

* 应该对TPM中的哪个PCR进行更新，对应的digest
* 根据`ima_template_desc`中定义的fields计算出的数据

`ima_template_entry`被装在`ima_queue_entry`中，并以两种方式组织在内存中：队列与哈希表。

```c
struct ima_queue_entry {
	struct hlist_node hnext;	/* place in hash collision list */
	struct list_head later;		/* place in ima_measurements list */
	struct ima_template_entry *entry;
};

struct ima_h_table ima_htable = {
	.len = ATOMIC_LONG_INIT(0),
	.violations = ATOMIC_LONG_INIT(0),
	.queue[0 ... IMA_MEASURE_HTABLE_SIZE - 1] = HLIST_HEAD_INIT
};

LIST_HEAD(ima_measurements);	/* list of all measurements */
```

## IMA策略管理

IMA子系统需要一个明确的策略来决定是否对特定文件采取特定行为（measure| appraise）。IMA提供了几个默认的策略，可以通过`ima_policy`内核命令行参数进行选择，我们最常见的就是`appraise_tcb`和`tcb`。

注意`policy_setup`函数中对`ima_policy`命令行参数的处理，很明显可以发现处理了一个历史遗留问题：`ima_tcb`参数。在早期内核版本中，并没有`ima_policy`参数的存在，IMA策略只能通过`ima_tcb`和`ima_appraise`等独立参数进行指定。新的内核提供了`ima_policy`参数的同时并没有删除掉`ima_tcb`的支持。事实上，`ima_tcb`与`ima_policy=tcb`的行为还是有区别的，这两个参数在内核中通过`ima_policy`变量的值进行区分：

```c
enum policy_types { ORIGINAL_TCB = 1, DEFAULT_TCB }
```

后面的分析中会具体分析其不同。除此之外，各个策略其实是可以叠加的，从对对应参数的处理就可以发现：

```c
		if ((strcmp(p, "tcb") == 0) && !ima_policy)
			ima_policy = DEFAULT_TCB;
		else if (strcmp(p, "appraise_tcb") == 0)
			ima_use_appraise_tcb = true;
		else if (strcmp(p, "secure_boot") == 0)
			ima_use_secure_boot = true;
		else if (strcmp(p, "fail_securely") == 0)
			ima_fail_unverifiable_sigs = true;
```

每一个变量都独立记录了是否进行特定行为，如下：

* ima_policy - 是否进行measure操作
* ima_use_appraise_tcb - 是否进行appraise操作
* ima_use_secure_boot - 是否位于secure boot模式
* ima_fail_unverifiable_sigs - 是否强制验证`SB_I_UNVERIFIABLE_SIGNATURE`标志的文件系统

事实上上面几个变量都是`__init`数据，只在初始化时才使用。IMA策略实际上的实现并没有这么简单，很明显，要实现自定义策略，这些是远远不够的。真正的IMA策略子系统初始化由`ima_init`函数中调用的`ima_init_policy`实现。

### IMA策略内部表示

IMA策略最后会转化为IMA规则表，格式如下：

```c
struct ima_rule_entry {
	struct list_head list;
	int action;
	unsigned int flags;
	enum ima_hooks func;
	int mask;
	unsigned long fsmagic;
	uuid_t fsuuid;
	kuid_t uid;
	kuid_t fowner;
	bool (*uid_op)(kuid_t, kuid_t);    /* Handlers for operators       */
	bool (*fowner_op)(kuid_t, kuid_t); /* uid_eq(), uid_gt(), uid_lt() */
	int pcr;
	struct {
		void *rule;	/* LSM file metadata specific */
		char *args_p;	/* audit value */
		int type;	/* audit type */
	} lsm[MAX_LSM_RULES];
	char *fsname;
	struct ima_rule_opt_list *keyrings; /* Measure keys added to these keyrings */
	struct ima_template_desc *template;
};
```

系统已经定义好了多个默认的规则列表：

* dont_measure_rules
* orignal_measurement_rules
* default_mesaurement_rules
* default_appraise_rules
* build_appraise_rules

首先明确一个规则最重要的部分就是`flags`，这个字段直接告诉规则执行引擎这个规则中哪些字段是需要比对的，已经定义的flags如下：

```c
#define IMA_FUNC	0x0001
#define IMA_MASK	0x0002
#define IMA_FSMAGIC	0x0004
#define IMA_UID		0x0008
#define IMA_FOWNER	0x0010
#define IMA_FSUUID	0x0020
#define IMA_INMASK	0x0040
#define IMA_EUID	0x0080
#define IMA_PCR		0x0100
#define IMA_FSNAME	0x0200
#define IMA_KEYRINGS	0x0400
```

`ima_rule_entry`的`action`字段标志该规则需要进行的操作，有如下选项：

```c
#define UNKNOWN		0
#define MEASURE		0x0001	/* same as IMA_MEASURE */
#define DONT_MEASURE	0x0002
#define APPRAISE	0x0004	/* same as IMA_APPRAISE */
#define DONT_APPRAISE	0x0008
#define AUDIT		0x0040
#define HASH		0x0100
#define DONT_HASH	0x0200
```

其他字段的分析留在分析规则执行引擎的时候再进行。现在随便找个默认规则列表分析一下，就分析`default_appraise_rules`：

```c
static struct ima_rule_entry default_appraise_rules[] __ro_after_init = {
	{.action = DONT_APPRAISE, .fsmagic = PROC_SUPER_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = SYSFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = DEBUGFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = TMPFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = RAMFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = DEVPTS_SUPER_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = BINFMTFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = SECURITYFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = SELINUX_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = SMACK_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = NSFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = EFIVARFS_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = CGROUP_SUPER_MAGIC, .flags = IMA_FSMAGIC},
	{.action = DONT_APPRAISE, .fsmagic = CGROUP2_SUPER_MAGIC, .flags = IMA_FSMAGIC},
#ifdef CONFIG_IMA_WRITE_POLICY
	{.action = APPRAISE, .func = POLICY_CHECK,
	.flags = IMA_FUNC | IMA_DIGSIG_REQUIRED},
#endif
#ifndef CONFIG_IMA_APPRAISE_SIGNED_INIT
	{.action = APPRAISE, .fowner = GLOBAL_ROOT_UID, .fowner_op = &uid_eq,
	 .flags = IMA_FOWNER},
#else
	/* force signature */
	{.action = APPRAISE, .fowner = GLOBAL_ROOT_UID, .fowner_op = &uid_eq,
	 .flags = IMA_FOWNER | IMA_DIGSIG_REQUIRED},
#endif
};
```

该规则列表中开头一大串的`DONT_APPRAISE`很明显是为虚拟文件系统定义的，很明显这些位于系统内存中、且内容是动态生成的虚拟文件系统是不需要进行`appraise`操作的。随后在`IMA_WRITE_POLICY`为true，即允许多次写入ima_policy的情况下新添了一条针对`POLICY_CHECK`的操作的`appraise`。最后才根据`IMA_APPRAISE_SIGNED_INIT`是否设置，定义一个针对root拥有的文件的`appraise`规则。

### ima_rules

IMA内部有一个名为`add_rules`的接口函数，用于将一个规则列表导入到IMA中。从中可以发现IMA规则列表有两个属性：

```c
enum policy_rule_list { IMA_DEFAULT_POLICY = 1, IMA_CUSTOM_POLICY }
```

即默认规则和定制规则，分别定义为内核定义的规则和用户自行定制的规则。同时也可以看到IMA中存在三个规则列表：

```c
static LIST_HEAD(ima_default_rules);
static LIST_HEAD(ima_policy_rules);
static LIST_HEAD(ima_temp_rules);
static struct list_head *ima_rules = &ima_default_rules;
```

这三个规则列表的同时存在是为了实现用户态的自定义策略。事实上，IMA使用`ima_rules`指针指向当前的规则列表。系统启动时，默认的规则列表为`ima_default_rules`，一旦用户要求内核使用自定义策略（通过写入securityfs导出的policy文件），IMA会调用`ima_update_policy`函数将`ima_rules`修改为指向`ima_policy_rules`的指针，同时更新`ima_policy_flags`。

### ima_init_policy

上面的分析完毕之后，可以看到这个函数就是根据内核的命令行参数向IMA的策略管理部分灌输规则列表。具体来说：

* ima_tcb对应于`original_measurement_rules`，`ima_policy=tcb`对应于`default_measurement_rules`
* appraise_tcb对应于`default_appraise_rules`
* secure_boot对应于`secure_boot_rules`，且会使`build_appraise_rules`不加入到`DEFAULT_POLICY`中

函数末尾调用了`ima_update_policy_flag`更新`ima_policy_flag`变量。

### ima_policy_flag

这个的思想非常简单，就是优化，类似于`vm_area`的优化。`ima_policy_flag`中的标志位缓存了现有规则要进行的操作（action字段）。据此，IMA可以简单的判断一些情况，如：没有规则要求`appraise`操作，据此来跳过特性行为，提升执行速度。

### ima_match_policy

该函数实际上就是规则引擎决策的入口。事实上整个函数，或者说整个IMA最费解的就是标志位的使用了，这一点在函数开头就能发现。函数开头有一个非常费解的操作：

```c
	int action = 0, actmask = flags | (flags << 1);
```

这与整个IMA的标志使用有关，需要正确理解，才能看懂接下来的实现。`security/integrity/integrity.h`文件的开头定义了大量的标志位，这些标志位在IMA子系统的大部分地方是通用的，其中有一些被用于描述IMA进行的行为：

```c
/* iint action cache flags */
#define IMA_MEASURE		0x00000001
#define IMA_MEASURED		0x00000002
#define IMA_APPRAISE		0x00000004
#define IMA_APPRAISED		0x00000008
/*#define IMA_COLLECT		0x00000010  do not use this flag */
#define IMA_COLLECTED		0x00000020
#define IMA_AUDIT		0x00000040
#define IMA_AUDITED		0x00000080
#define IMA_HASH		0x00000100
#define IMA_HASHED		0x0000020
```

注意他们是成对出现的，且一对标志使用相邻的两位。以`IMA_MEASURED`举例，`IMA_MEASURE`表示进行`measure`行为，而对应的`IMA_MEASURED`标志着对象已经完成了`measure`。这样定义是因为这些标志位也被用作`iint cache`（inode的完整性缓存）中，用以标志inode的状态。前面看到`ima_rule_entry`的action字段的可取值，也是基于这几个`action`定义的。只不过，`IMA_MEASURED`这样的标志被重新定义成了`DONT_MEASURE`，表示不进行`measure`。

有了上面的分析，接下来就可以看懂代码了。`actmask`本质上就是`action mask`的缩写，是`ima_rule_entry.action`字段的一个mask，可以同时匹配action本身与对应的`don't action`。函数接下来的操作仅仅就是遍历`ima_rules`列表，检查`entry->action`是否落于`actmask`中。如果是，则调用`ima_match_rules`对`ima_rule_entry`进行进一步匹配，匹配成功后才进行下一步操作。

接下来的操作涉及两个mask：`IMA_ACTION_FLAGS`和`IMA_DO_MASK`：

```c
#define IMA_DO_MASK		(IMA_MEASURE | IMA_APPRAISE | IMA_AUDIT | \
				 IMA_HASH | IMA_APPRAISE_SUBMASK)

#define IMA_ACTION_FLAGS	0xff000000
#define IMA_DIGSIG_REQUIRED	0x01000000
#define IMA_PERMIT_DIRECTIO	0x02000000
#define IMA_NEW_FILE		0x04000000
#define EVM_IMMUTABLE_DIGSIG	0x08000000
#define IMA_FAIL_UNVERIFIABLE_SIGS	0x10000000
#define IMA_MODSIG_ALLOWED	0x20000000
#define IMA_CHECK_BLACKLIST	0x40000000
```

很明显，`IMA_DO_MASK`定义为IMA对特定文件采取的行为。而`IMA_ACTION_FLAGS`就用的比较取巧了，从后面可以看到`ima_rule_entry.flags`最大几位被定义成`IMA_ACTION_FLAGS`，用以描述规则匹配上时的额外行为，而函数接下的操作就是将这几位取下，加入到`action`变量中：

```c
		action |= entry->flags & IMA_ACTION_FLAGS;
```

除此之外，也应该将`ima_rule_entry.action`加入到`action`变量中：

```c
		action |= entry->action & IMA_DO_MASK;
```

注意`appraise`操作实际上是划分为更小的操作的，主要是通过hook的位置决定：

```c
		if (entry->action & IMA_APPRAISE) {
			action |= get_subaction(entry, func);
			action &= ~IMA_HASH;
			if (ima_fail_unverifiable_sigs)
				action |= IMA_FAIL_UNVERIFIABLE_SIGS;
		}
```

因为`actmask`实际上在这次已经匹配上了，所以清除对应的标志位：

```c
		if (entry->action & IMA_DO_MASK)
			actmask &= ~(entry->action | entry->action << 1);
		else
			actmask &= ~(entry->action | entry->action >> 1);
```

注意循环的结尾，只有`actmask`被彻底清空之后循环才会结束，否则会一直进行匹配。

### ima_match_rules

`ima_match_rules`只区分对待一种情况，即传入的`func=KEY_CHECK`时。此时直接调用`ima_match_keyring`并返回。随后，函数会对inode进行多次匹配，只有完全匹配之后，才能返回true，即认为匹配上了。

## IMA初始化

### init_ima

IMA的初始化位于`security/integrity/ima_main.c`的`init_ima`函数中。函数首先初始化支持的模板，简单来说就是将`builtin_templates`拷贝到`defined_templates`。注意如果通过内核命令行`ima_template_fmt`创建了新模板，则新建的模板也会进行拷贝。

随后调用`hash_setup`设置默认的哈希算法，默认的哈希算法为`sha1`且被写入`ima_hash_algo`变量中。注意当哈希算法初始化失败时，会fallback一个哈希算法。

最后调用`ima_init`函数进行初始化操作，一旦`ima_init`调用成功，则注册LSM事件监听函数`ima_lsm_policy_change`函数，并调用`ima_update_policy_flag`函数更新`ima_policy_flag`变量。

### ima_init

函数会获取当前默认的TPM设备，一旦失败，则打印相关信息并进入`TPM bypass`模式，该模式下一些需要TPM才能进行的操作会被取消。相应的，系统的安全性会下降。

TODO

## IMA内核钩子

IMA子系统的内核钩子主要集中在`ima_main.c`文件中，经过观察可以发现，`measure`和`appraise`都是由`process_measurement`函数实现的，直接搜索这个函数名就可以找到调用了该函数的内核钩子实现：

* ima_file_mmap
* ima_bprm_check
* ima_file_check
* ima_read_file
* ima_post_read_file

从名字基本就能看出其对应的位置，如后后续对具体实现有需要，可以进一步进行分析。`enum ima_hooks`定义了钩子的类型，使`process_mesasurement`函数，以及后续的规则处理能够获取到是哪一个钩子调用的`process_mesasurement`函数。

```c
#define __ima_hooks(hook)				\
	hook(NONE, none)				\
	hook(FILE_CHECK, file)				\
	hook(MMAP_CHECK, mmap)				\
	hook(BPRM_CHECK, bprm)				\
	hook(CREDS_CHECK, creds)			\
	hook(POST_SETATTR, post_setattr)		\
	hook(MODULE_CHECK, module)			\
	hook(FIRMWARE_CHECK, firmware)			\
	hook(KEXEC_KERNEL_CHECK, kexec_kernel)		\
	hook(KEXEC_INITRAMFS_CHECK, kexec_initramfs)	\
	hook(POLICY_CHECK, policy)			\
	hook(KEXEC_CMDLINE, kexec_cmdline)		\
	hook(KEY_CHECK, key)				\
	hook(MAX_CHECK, none)

#define __ima_hook_enumify(ENUM, str)	ENUM,
#define __ima_stringify(arg) (#arg)
#define __ima_hook_measuring_stringify(ENUM, str) \
		(__ima_stringify(measuring_ ##str)),

enum ima_hooks {
	__ima_hooks(__ima_hook_enumify)
};
```

## IMA度量与验证

### integrity_iint_cache

IMA一直有一个核心问题需要解决：性能。抛开IMA解决的问题本身，一旦IMA的性能影响了操作系统的正常运行，那么IMA本身是没有意义的。经过简单思考之后，一般能够想到以下优化手段：

* 将磁盘中保存的`security.ima`属性缓存到内存中以加速读取
* 对没有被修改过的文件不进行重复哈希值计算
* 缓存特定文件的规则匹配状态，减少规则匹配产生的开销

这些操作实质上要求IMA实现一个缓存，类似一个哈希表，通过inode作为key搜索对应的信息。IMA的缓存实现在`security/integrity/iint.c`中，名为`iint cache`，本质上是一个红黑树。所有的缓存单元通过SLAB进行管理，并在内核启动时进行初始化：

```c
	iint_cache =
	    kmem_cache_create("iint_cache", sizeof(struct integrity_iint_cache),
			      0, SLAB_PANIC, init_once);
```

缓存内的元素定义非常简单，如下：

```c
/* integrity data associated with an inode */
struct integrity_iint_cache {
	struct rb_node rb_node;	/* rooted in integrity_iint_tree */
	struct mutex mutex;	/* protects: version, flags, digest */
	struct inode *inode;	/* back pointer to inode in question */
	u64 version;		/* track inode changes */
	unsigned long flags;
	unsigned long measured_pcrs;
	unsigned long atomic_flags;
	enum integrity_status ima_file_status:4;
	enum integrity_status ima_mmap_status:4;
	enum integrity_status ima_bprm_status:4;
	enum integrity_status ima_read_status:4;
	enum integrity_status ima_creds_status:4;
	enum integrity_status evm_status:4;
	struct ima_digest_data *ima_hash;
};
```

整个`integrity_iint_cache`作为一个红黑树的元素，被组织起来，它的`inode`字段用于保存其对应的inode，这使得红黑树是可以进行遍历查找的。IMA内部可以通过`integrity_iint_cache`对红黑树进行遍历，注意这里有一个优化：有对应缓存的inode的`i_flags`里有一位`S_IMA`，可以直接确定该inode没有对应的缓存。而外界最常用的API就是`integrity_inode_get`了，该函数首先会查找inode对应的缓存，一旦发现没有就会创建缓存并插入到红黑树中。

### process_measurement

前面提到了所有的钩子函数都调用了该函数，所以该函数是比较复杂的，其原型如下：

```c
static int process_measurement(struct file *file, const struct cred *cred,
			       u32 secid, char *buf, loff_t size, int mask,
			       enum ima_hooks func);
```

其中`file`、`cred`和`secid`的用途与内核中其他地方无二。`buf`缓冲区的作用需要从代码中寻找，参数`mask`文件系统中的标准权限位，`ima_hooks`则传入了调用该函数的钩子位置。

接下来分析函数行为，函数首先检查当前的`ima_policy_flag`，一旦发现其为0（即没有设置任何一个标志位）则直接退出，直接放行。同样地，由于IMA负责检验文件的一致性，所以这几个钩子函数都针对普通文件，一旦文件不是普通文件，则钩子也直接放行。

随后，函数调用`ima_get_action`得到需要对该文件进行的行为，本质上调用了`ima_match_policy`查找了规则表。如果规则表认为对该文件不用进行任何操作（无action和violation_check），则会直接退出。

* 获取inode对应的integrity_iint_cache

TODO

### ima_collect_measurement

该函数计算文件内容哈希值，并更新i_version和哈系值到inode对应的`iint cache`中。

### ima_store_measurement

该函数实际上是比较简单的，其本质就是根据模板模板描述符生成一个文件的度量值，然后将其添加到`ima_measurement`队列和`ima_htable`中。如果系统中存在TPM，则将度量值extend到对应的PCR中。

### ima_appraise_measurement

函数首先调用`evm_verifyxattr`校验`security.ima`的值（即检查保存在磁盘上的IMA度量基准值是否被人改动过），如果校验通过，则对比度量基准值与文件内容真正的哈希值。

## IMA Modsig

出了支持基于哈希算法的验证，IMA也支持基于签名的验证。该功能主要用于内核模块，可以在相关的secure boot的policy中看到相关的设定。

## Keyring管理

从内核配置中可以看到有一个关于keyring的选项：`INTEGRITY_TRUSTED_KEYRING`。我们从这个选项入手进行分析。首先可以看到，整个codebase中只有两个地方用到了该配置选项：

```c
static struct key *keyring[INTEGRITY_KEYRING_MAX];

static const char *keyring_name[INTEGRITY_KEYRING_MAX] = {
#ifndef CONFIG_INTEGRITY_TRUSTED_KEYRING
	"_evm",
	"_ima",
#else
	".evm",
	".ima",
#endif
	"_module",
};

#ifdef CONFIG_INTEGRITY_TRUSTED_KEYRING
static bool init_keyring __initdata = true;
#else
static bool init_keyring __initdata;
#endif
```

可以看到，该选项开启后唯一造成的作用就是默认keyring列表的名称发生了改变，且`init_keyring`变量被设置成了`true`。而`init_keyring`变量的值则会影响`integrity_init_keyring`函数的行为：

```c
int __init integrity_init_keyring(const unsigned int id)
{
	const struct cred *cred = current_cred();
	struct key_restriction *restriction;
	int err = 0;

	if (!init_keyring)
		return 0;
```

可以看到，如果`init_keyring`为`false`，即配置没有被选中，则函数会直接退出，没有接下来的操作。否则，函数调用`keyring_alloc`创建`keyring_name`数组中对应名称的keyring，并配置该keyring的restriction，要求加入到该keyring的key必须被内核信任的证书签过名。

再来看`integrity_digsig_verify`函数的实现：

```c
	if (!keyring[id]) {
		keyring[id] =
			request_key(&key_type_keyring, keyring_name[id], NULL);
		if (IS_ERR(keyring[id])) {
			int err = PTR_ERR(keyring[id]);
			pr_err("no %s keyring: %d\n", keyring_name[id], err);
			keyring[id] = NULL;
			return err;
		}
	}
```

从这里可以看到，函数首先检查`keyring`数组中又没有保存对应的keyring，没有的话就会调用`request_key`向内核请求该keyring。如果请求失败，此时函数就直接报错退出了。这里看明白了内核的行为，也就是说，无论如何，函数都会向内核请求keyring，但是这个keyring在`INTEGRITY_TRUSTED_KEYRING`选项开启时是由内核自行创建，并强制制定了规则的。反之，该keyring可以由用户态进行创建。