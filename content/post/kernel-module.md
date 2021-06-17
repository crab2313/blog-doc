+++
title = "Linux在RISC-V平台下的模块实现"
date = 2021-06-12
draft = true


tags = ["kernel", "binary"]
+++

因为上次碰到了模块ABI改变导致的模块加载异常问题，虽然通过分析发现了问题，也认为解决方法是模块跟着ABI走，重新编译，但是认为模块的实现还是要看的。虽然能够想清楚大概的实现方式，但是细节还是要深究一下。很明显内核模块涉及到内核的方方面面，我认为应该从以下几点进行分析：

* 内核模块的资源管理，地址空间以及内存
* 内核模块的编译框架
* 内核模块的relocation机制
* 内核模块的格式细节，即内核模块这个ELF本身，以RISC-V为例
* 内核模块与架构相关代码如何交互，以RISC-V为例
* 内核模块相关的接口，以及内核如何管理内核模块
* 内核模块的签名机制
* 内核模块符号导出机制

这次分析是对我综合能力的一次考验，但是感觉花够时间应该能够完成。下面列出个人认为内核模块主要涉及的知识点：

* ELF文件格式，编译链接原理，relocation原理。
* 内核地址空间管理，链接脚本
* Kbuild源码分析能力
* 特定平台下的Code Model和寻址方式

# 内核模块的生成机制

内核模块本身实际上是一个`relocatable`的ELF文件，其实就是常说的`.o`文件。这一类文件的特点是`relocatable`，也就是其中有用于`relocation`的section和符号表。一般情况下，链接器根据`.o`文件中的`relocation`段中的`relocation`和符号表中的内容决定如何填充代码段中的占位，然后生成一个可执行程序。而对于内核模块，内核会自行根据`relocatable`模块的`relocation`段和符号表的内容，自行进行`relocate`操作，本质上就是自行实现了runtime链接。原先在分析kdump工作原理时见到了类似的实现。

事实上，内核模块的生成原理并不复杂，仅仅简单是生成一个`relocatable`的ELF格式文件，这个文件由两部分组成：

* 模块源码本身生成的`.o`文件
* modpost生成的`<module>.mod.c`文件编译后的`.o`文件

在内核头文件中，与模块相关的定义一般有两套，一套用于生成vmlinux，另一套用于模块。这二者之间通过`MODULE`宏是否有定义进行区分。简单来说，内核用到个各个section在模块中也是适用了，模块的生成是比较简单的，真正复杂的处理在模块装载时体现。在内核的`scripts/mod`文件夹下，存放着`modpost`工具的源码，用于编译出`modpost`工具。modpost工具生成的`<module_name>.mod.c`类似如下：

```c
#include <linux/module.h>
#define INCLUDE_VERMAGIC
#include <linux/build-salt.h>
#include <linux/vermagic.h>
#include <linux/compiler.h>

BUILD_SALT;

MODULE_INFO(vermagic, VERMAGIC_STRING);
MODULE_INFO(name, KBUILD_MODNAME);

__visible struct module __this_module
__section(".gnu.linkonce.this_module") = {
	.name = KBUILD_MODNAME,
	.init = init_module,
#ifdef CONFIG_MODULE_UNLOAD
	.exit = cleanup_module,
#endif
	.arch = MODULE_ARCH_INIT,
};

#ifdef CONFIG_RETPOLINE
MODULE_INFO(retpoline, "Y");
#endif

MODULE_INFO(depends, "wmi");

MODULE_ALIAS("wmi:8C5DA44C-CDC3-46B3-8619-4E26D34390B7");

MODULE_INFO(srcversion, "E60E97D36266BD3884F3208");
```

这其中的一些机制细节，可以在模块装载流程分析时明了。除此之外，内核模块还可能携带签名，内核模块签名并不携带在ELF文件本身里面，而是计算完签名之后，添加到ELF文件尾部。

# 内核模块的装载流程

涉及内核模块的系统调用有三个：

* init_module
* finit_module
* delete_module

其中，`finit_module`是`init_module`的文件描述符形式，可以勉强算作一个。因此，想要理解内核如何装载模块，完成了哪些工作，需要分析`init_module`函数。之所以称勉强算作一个，是因为`finit_module`有一个额外的参数，支持传入flags，即：

* MODULE_INIT_IGNORE_MODVERSIONS
* MODULE_INIT_IGNORE_VERMAGIC

而`init_module`则不允许。二者的本质都是在做完权限检查（CAP_SYS_MODULE和`modules_disabled`内核命令行）之后，分配空间存放内核模块ELF文件，然后调用`load_module`函数。`load_modules`的参数有三个，其中最关键的是`load_info`结构体。

```c
struct load_info {
        const char *name;
        /* pointer to module in temporary copy, freed at end of load_module() */
        struct module *mod;
        Elf_Ehdr *hdr;
        unsigned long len;
        Elf_Shdr *sechdrs;
        char *secstrings, *strtab;
        unsigned long symoffs, stroffs, init_typeoffs, core_typeoffs;
        struct _ddebug *debug;
        unsigned int num_debug;
        bool sig_ok;
#ifdef CONFIG_KALLSYMS
        unsigned long mod_kallsyms_init_off;
#endif
        struct {
                unsigned int sym, str, mod, vers, info, pcpu;
        } index;
};
```

在传入`load_module`时，只有`hdr`和`len`字段，被填写为了存放ELF文件内容的地址与长度。对模块装载流程的分析转变称了对`load_module`函数的分析。

## 合法性检查

合法性检查不多说，主要有如下几点：

* 模块签名检查，由`mod_sig_check`函数完成。这里多说一句，模块签名是加在模块ELF文件尾部的，模块最尾部有一个`magic string`，即一个特殊的字符串，内核通过该字符串确定模块是否带有签名。除此之外模块的ELF文件部分不会因为签名而变动。
* ELF文件合法性检查，简单检查ELF的文件结构是否合法。
* 获取模块名字之后，检查其是否在`module_blacklist`内核命令行参数，如果是，则拒绝加载。

## setup_load_info

随后函数调用`setup_load_info`，从前面看到传入的`load_info`还是空的，这个函数负责填充它。注意该函数并不是填充所有的字段，而是主要填充`index`字段，其他部分后面会继续处理。之所以是叫`index`，是因为其内部字段记录的都是模块ELF文件的section header的索引。函数的主要行为如下：

* 遍历搜索`.modinfo`段，并将其索引保存在`index.info`字段。从这里可以看出这个section的内容为多个`KEY=VALUE`的`NULL`结尾字符串。
* 从`.modinfo`段中可以找到`name`关键字，将其值作为模块名称保存在`load_info->name`中。
* 遍历搜索模块ELF文件的符号表以及字符串表，将其索引保存至`index.sym`和`index.str`。并将`load_info->strtab`指向ELF文件的字符串表位置。
* 遍历搜索`.gnu.linkonce.this_module`段，并将其索引保存在`index.mod`中。并将`load_info->mod`指向该段的首地址。如果前面从`.modinfo`段中没有找到模块名称，则可以从这个`struct module`结构体中获取。
* 如果调用`load_module`函数时设置了`MODULE_INIT_IGNORE_MODVERSIONS`标志，则将`index.vers`设置为0。反之，则遍历查找`__versions`段，并设置索引。
* 遍历查找`.data..percpu`段，并设置`index.pcpu`索引。

## rewrite_section_headers

该函数对模块ELF文件的section header做一些简单处理，包含如下操作：

* 将第一个section的`sh_addr`设置为0
* 将其他所有的section的`sh_addr`指向其当前位于的虚拟地址
* unset掉`__versions`段和`.modinfo`段的`SHF_ALLOC`标志

稍微做一下解释，在ELF标准中，一个section的`sh_addr`属性记录section的第一个字节应该被装载的虚拟地址。内核认为模块的第一个section是特殊的，遍历搜索section时的代码都会跳过第一个，即0号section。最后内核遍历section的代码也会跳过所有没有`SHF_ALLOC`标志的section，即无视没有标为`SHF_ALLOC`的section。但是很明显，我们最终没有必要将`__versions`和`.modinfo`装载至内存中，所以这里unset掉这个标志。

## check_modstruct_versions

详见模块版本机制分析。

## layout_and_allocate

函数首先调用`check_modinfo`检查模块的一些基本信息，包括：

* `.modinfo`中的`vermagic`，如果`MODULE_INIT_IGNORE_VERMAGIC`标志设置时，不检查这个`vermagic`。否则，当`vermagic`不匹配时阻止加载。
* `.modinfo`中的`intree`。如果没有找到的话，说明这个模块是out of tree的，照例给个警告
* `.modinfo`中的`staging`。如果找到的话，说明模块处于staging阶段，照例给个警告
* `.modinfo`中的`livepatch`。找到的话，如果内核支持live patch，那么标记一下，并给个警告，否则禁止加载
* `.modinfo`中的`retpoline`。检查模块是否启用`retpoline`，这个是`Spectre V2`的检查，目前只有x86有。
* `.modinfo`中的`license`。检查并设置模块的`license`，如果与GPL2不兼容，则打出一条警告。

随后函数调用一个平台相关的函数`module_frob_arch_sections`，该函数一般会进行一些架构相关的操作，比如处理GOT和PLT，我们后面分析RISC-V的实现。在`CONFIG_STRICT_MODULE_RWX`配置选项开启时，函数还会检查是否存在即可写又可执行的section，如果存在即阻止模块加载。

由于模块会对`.data..pcpu`段进行特殊处理，所以这里unset掉了其`SHF_ALLOC`标志。同时，函数设置了`.data..ro_after_init`与`__jump_label`段的`SHF_RO_AFTER_INIT`标志。

随后函数连续调用了三个功能比较大的函数：

* layout_sections
* layout_symtab
* move_module

其本质上就是准备好接下来需要装载的section的信息，然后申请空间，进行装载，我们拆开来看。

### layout_sections

这个函数本质上就是进行最终装载到内存中的sections的布局工作，本质上就是选出需要的section，然后计算出其应该处于的位置，这个位置本质上就是个偏移量。这个操作是ELF文件装载最常见的操作，对于一个section，首先将上一个排放好的section的结尾偏移量做一个对齐操作，即为该section的开头，在将这个开头偏移量加上section的大小，即为section的结尾偏移量。这个操作可以总结成如下helper函数：

```c
/* Update size with this section: return offset. */
static long get_offset(struct module *mod, unsigned int *size,
                       Elf_Shdr *sechdr, unsigned int section)
{
        long ret;

        *size += arch_mod_section_prepend(mod, section);
        ret = ALIGN(*size, sechdr->sh_addralign ?: 1);
        *size = ret + sechdr->sh_size;
        return ret;
}

/* Additional bytes needed by arch in front of individual sections */
unsigned int __weak arch_mod_section_prepend(struct module *mod,
                                             unsigned int section)
{
        /* default implementation just returns zero */
        return 0;
}
```

回到`layout_sections`函数，函数一共初始化了两个`struct module_layout`，分别名为`core_layout`与`init_layout`。其中`init_layout`对应着模块的`.init`段，后面可以进行释放操作。而`core_layout`即为模块代码空间本身的layout。`struct module_layout`的结构如下：

```c
struct module_layout {
        /* The actual code + data. */
        void *base;
        /* Total size. */
        unsigned int size;
        /* The size of the executable code.  */
        unsigned int text_size;
        /* Size of RO section of the module (text+rodata) */
        unsigned int ro_size;
        /* Size of RO after init section */
        unsigned int ro_after_init_size;

#ifdef CONFIG_MODULES_TREE_LOOKUP
        struct mod_tree_node mtn;
#endif
};
```

`layout_sections`函数使用一组mask对所有的section进行两次（是否为`.init`段），成功匹配的section会被加入对应layout中。这组mask如下：

```c
        static unsigned long const masks[][2] = {
                /*
                 * NOTE: all executable code must be the first section
                 * in this array; otherwise modify the text_size
                 * finder in the two loops below
                 */
                { SHF_EXECINSTR | SHF_ALLOC, ARCH_SHF_SMALL },
                { SHF_ALLOC, SHF_WRITE | ARCH_SHF_SMALL },
                { SHF_RO_AFTER_INIT | SHF_ALLOC, ARCH_SHF_SMALL },
                { SHF_WRITE | SHF_ALLOC, ARCH_SHF_SMALL },
                { ARCH_SHF_SMALL | SHF_ALLOC, 0 }
        };
```

分别对应于：可执行，只读数据，`.data..ro_after_init`和`__jump_label`（INIT后只读数据），可写数据。注意每个mask有两部分，前一部分是匹配项，必须匹配上，后一部分是不匹配项，必须不匹配上。这组mask最后会将有`ARCH_SHF_SMALL`标志的section放到layout最后面，而`ARCH_SHF_SMALL`由架构自行定义。

函数使用了一个trick，即利用section header中的`sh_entsize`作为临时区域存储偏移量。这是因为`sh_entsize`一般是用于保存拥有固定元素大小的类似数组的section的元素大小的字段，而这里处理的section很明显没有这样的section。同时，`sh_entsize`的MSB一位作为一个标志位使用，存储了该section到底是属于`core_layout`还是`init_layout`这一信息。

### layout_symtab

该函数仅当`CONFIG_KALLSYMS`时有定义，否则为空函数。函数工作总结如下：

* 将模块ELF文件的符号表放到`init_layout`的末尾
* 找出core symbol，并在`core_layout`上预留其对应的位置。这里还预留了core symbol使用的字符串表中字符串的空间
* 将字符串表放到`init_layout`末尾
* `init_layout`末尾预留一个`struct mod_kallsyms`的空间

### move_module

该函数简单通过`module_alloc`函数申请`core_layout`与`init_layout`所占的空间，然后设置对应的`core_layout->base`与`init_layout->base`。根据前面所做的标记得到需要拷贝的section的源地址与目标地址，进行拷贝，很明显`SHT_NOBITS`标记的section（例如.bss）是不用拷贝的。拷贝完成后，函数更新section header的`sh_addr`字段，指向拷贝后的地址。

## percpu_modalloc

```c
        mod->percpu = __alloc_reserved_percpu(pcpusec->sh_size, align);
        if (!mod->percpu) {
                pr_warn("%s: Could not allocate %lu bytes percpu data\n",
                        mod->name, (unsigned long)pcpusec->sh_size);
                return -ENOMEM;
        }
        mod->percpu_size = pcpusec->sh_size;
```

函数很简单，唯一需要记住的就是模块的`percpu`数据的处理方式。Linux直接通过`__alloc_reserved_percpu`接口申请了模块需要的percpu数据区域，并保存在`struct module`结构体中。

## find_module_sections

该函数查找特定的section，并初始化`struct module`的特定字段，比较多，这里不一一列举。

## simplify_symbols

这个函数的目的是解决所有没有定义的symbol，也就是`st_shndx`定义为`SHN_UNDEF`的符号。简单来说，函数遍历符号表，找到`SHN_UNDEF`类型的符号，然后要求内核resolve该符号。得到该符号的地址后，函数将其填入symbol的`st_value`字段内。内核符号的机制后面尽心分析，这里只需要知道要求内核对符号进行了解析，并返回了地址。

除此之外，这个函数还对percpu变量进行了特殊处理。

## apply_relocations

该函数遍历所有的section，然后对以下三种进行处理：

* 有SHF_RELA_LIVEPATCH标志的
* SHT_REL类型
* SHT_RELA类型

这里会调用架构相关的函数`apply_relocate`和`apply_relocate_add`对这个可执行程序根据`relocation`的类型进行`relocate`操作。这个操作比较常规，就是linker比较常见的行为，后面分析RISC-V下的实现。总之，`relocation`完成之后，`core_layout`上的`relocation`完成了`relocate`操作，代码段可以正常执行了，对symbol的引用被指向了正确的地址。

## do_module_init

在处理完一些杂项之后，`do_module_init`函数将完整最后的操作。简单来说，这个函数将会调用`mod->init`函数指针，也就是模块自己实现`init_module`函数。之后，完成一些通知操作，并释放`init_layout`所占用的空间。

这里可以思考一个问题，即`mod->init`指针是谁设置的？从前面看到`mod`，即一个`struct module`类型的变量是被定义在`.gnu.linkonce.this_module`段中的。整个`load_module`函数操作过程中，`mod`指针都是指向这个段的，由于模块ELF是一个`relocatable`的ELF文件，所以进行`relocation`时，`__this_module`上的`init`指针会被`relocation`重写，指向`init_module`装载后的虚拟地址。使得我们调用`mod->init`时，可以调用模块自己实现的`init_module`函数。

# 内核符号管理机制

内核提供了一套机制用于控制模块可以访问的符号。这里有一点需要注意，一旦模块被加载并执行，他是有权力访问整个虚拟地址空间的，内核只能控制其对符号对应地址的请求。可以想象，内核维护了一张表，表中记录了内核向模块导出的接口（符号），以及该符号目前对应的虚拟地址。这张表应该是动态的，因为有KASLR等机制会随机化内核的地址偏移量。

## 符号的注册

我们最常见的符号注册机制就是`EXPORT_SYMBOL`宏，这个宏的实现是比较直观的，但是细节很多，可以只抓原理。一般情况下，其定义如下：

```c
#define ___EXPORT_SYMBOL(sym, sec, ns)                                          \
        extern typeof(sym) sym;                                                 \
        extern const char __kstrtab_##sym[];                                    \
        extern const char __kstrtabns_##sym[];                                  \
        __CRC_SYMBOL(sym, sec);                                                 \
        asm("   .section \"__ksymtab_strings\",\"aMS\",%progbits,1      \n"     \
            "__kstrtab_" #sym ":                                        \n"     \
            "   .asciz  \"" #sym "\"                                    \n"     \
            "__kstrtabns_" #sym ":                                      \n"     \
            "   .asciz  \"" ns "\"                                      \n"     \
            "   .previous                                               \n");   \
        __KSYMTAB_ENTRY(sym, sec)

#endif

#define __KSYMTAB_ENTRY(sym, sec)                                       \
        static const struct kernel_symbol __ksymtab_##sym               \
        __attribute__((section("___ksymtab" sec "+" #sym), used))       \
        __aligned(sizeof(void *))                                       \
        = { (unsigned long)&sym, __kstrtab_##sym, __kstrtabns_##sym }

struct kernel_symbol {
        unsigned long value;
        const char *name;
        const char *namespace;
};
#endif

#ifdef DEFAULT_SYMBOL_NAMESPACE
#include <linux/stringify.h>
#define _EXPORT_SYMBOL(sym, sec)        __EXPORT_SYMBOL(sym, sec, __stringify(DEFAULT_SYMBOL_NAMESPACE))
#else
#define _EXPORT_SYMBOL(sym, sec)        __EXPORT_SYMBOL(sym, sec, "")
#endif

#define EXPORT_SYMBOL(sym)              _EXPORT_SYMBOL(sym, "")
#define EXPORT_SYMBOL_GPL(sym)          _EXPORT_SYMBOL(sym, "_gpl")
#define EXPORT_SYMBOL_NS(sym, ns)       __EXPORT_SYMBOL(sym, "", #ns)
#define EXPORT_SYMBOL_NS_GPL(sym, ns)   __EXPORT_SYMBOL(sym, "_gpl", #ns)
```

可以看到`EXPORT_SYMBOL`也是利用section操作实现的。简单来说，当调用`EXPORT_SYMBOL`时：

* 生成`__kstrtab_##sym`，`__kstrtabns_##sym` 两个symbol，其内容位于`__ksymtab_strings`段中。
* 定义一个名为` `的`struct kernel_symbol`，放入`__ksymtab+#sym`或者`__ksymtab_gpl+#sym`段中，这个结构体的内容是symbol位于的地址，以及其两个定义在`__ksymtab_strings`中的两个字符串。后面看一下KASLR的实现，应该会比较有意思。同样，连接脚本里的合并section操作也是个细节。
* 如果`CONFIG_MODVERSIONS`有定义，则`__CRC_SYMBOL`会为symbol定义一个CRC值，用以验证symbol的版本，后面分析。

回到链接脚本的实现，可以发现与`__ksymtab`有关的段都定义在`RODATA`宏里。首先是`__ksymtab`：

```c
        /* Kernel symbol table: Normal symbols */                       \
        __ksymtab         : AT(ADDR(__ksymtab) - LOAD_OFFSET) {         \
                __start___ksymtab = .;                                  \
                KEEP(*(SORT(___ksymtab+*)))                             \
                __stop___ksymtab = .;                                   \
        }                                                               \
```

可以看到，前面定义的`__ksymtab+#sym`段被排序，并合并成了`__ksymtab`段。同理，其他的section也是类似的方式合并。

