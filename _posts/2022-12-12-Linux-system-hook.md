---
layout: post
title:  "Linux System Hook"
date:   2022-12-12 15:15:54
categories: Linux-Kernel
excerpt: kernel linux syscall hook
---

* content
{:toc}

---


# hook linux syscall on x86_64

## hook的方法





## Search and Patch Blindly 

### 获取syscall_table

---

获取syscall_table是hook syscall的第一步，但是linux内核版本更新很快，v5.7.0之后的版本已经不再export `kallsyms_lookup_name`这个api了，不过通过下面的方法仍然可以获得`kallsyms_lookup_name`函数地址。

``` 
    #include <linux/init.h>
    #include <linux/module.h>
    #include <linux/kallsyms.h>
    #include <linux/version.h>

    typedef asmlinkage long (*sys_call_ptr_t)(const struct pt_regs *);
    static sys_call_ptr_t *sys_call_table;


    #if LINUX_VERSION_CODE >= KERNEL_VERSION(5,7,0)
    #define KPROBE_LOOKUP 1
    #include <linux/kprobes.h>
    static struct kprobe kp = {
        .symbol_name = "kallsyms_lookup_name";
    };
    #endif

    static int __init hook_init(void)
    {
    #ifdef KPROBE_LOOKUP
        typedef unsigned long (*kallsyms_lookup_name_t)(const char *name);
        kallsyms_lookup_name_t kallsyms_lookup_name;
        register_kprobe(&kp);    
        kallsyms_lookup_name = (kallsyms_lookup_name_t)kp.addr;
        unregister_kprobe(&kp);
    #endif

        sys_call_table = (sys_call_ptr_t *)kallsyms_lookup_name("sys_call_table");

        printk("[ifno] %s, execve:0x%llx\n"， __func__, sys_call_table[__NR_execve]);

        return 0;
    }


    ...

```



### Rip-relative指令

---

相对于x86来说，x86_64指令集最大的变化就是`long mode`下开始更大范围的支持`RIP-relative addressing`新的寻址方式,它可以更加方便的实现`position-independent` code. 在之前的x86中，只有`control transfer`指令（call, jmp, soforth）才支持RIP寻址。

`RIP-relative addressing`就是通过一个相对于当前指令指针的`signed int` 32bit的偏移量进行寻址的模式。

>  If one used flat addressing on x86, references to global variables typically required hardcoding the absolute address of the global in question, assuming the module loads at its preferred base address. If the module then could not be loaded at the preferred base address at runtime, the loader had to perform a set of base relocations that essentially rewrite all instructions that had an absolute address operand component to refer to take into account the new address of the module.

使用RIP寻址可以很好的解决上面这些问题。

下面通过实例来说明RIP寻址的过程。

``` 
    //test_rip.c

    #include <stdio.h>

    static void foo()
    {
        int a = 666;
    }

    int main()
    {
        foo();

        return 0;
    }

    $gcc -g -O0 test_rip.c
    $gdb ./a.out
    (gdb)disassemble /r

```

![kernel_hook_0.png](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/kernel_hook_0.png)



### patch hook实现方法





# hook linux library


# hook in docker env


# TOCTTOU攻击
