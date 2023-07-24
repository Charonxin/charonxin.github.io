---
title: "Some tricks in gcc"
date: 2023-07-24T15:34:30-04:00
categories:
  - blog
tags:
  - gcc
  - operating system
---

# Gcc 编译器的那些坑

-------


在进行到真相还原那本书的第七章时，遇到的一些小问题。


## 问题复现

目录结构：

```shell
#log
charon@charon-vm:~/MyOS$ tree .
.
├── boot
│   ├── include
│   │   └── boot.inc
│   ├── loader.S
│   └── mbr.S
├── build.sh
├── kernel
│   ├── global.h
│   ├── init.c
│   ├── init.h
│   ├── interrupt.c
│   ├── interrupt.h
│   ├── io.h
│   ├── kernel.S
│   └── main.c
├── lib
│   ├── kernel
│   │   ├── print.h
│   │   └── print.S
│   └── stdint.h
└── README.md
```

执行命令

```shell
#!/bin/sh

mkdir build

echo "Compiling..."
nasm -I boot/include/ -o build/mbr.bin boot/mbr.S
nasm -I boot/include/ -o build/loader.bin boot/loader.S
gcc  -I lib/ -I kernel/ -I lib/kernel/ -c -fno-builtin -o build/main.o kernel/main.c
nasm -f elf -o build/print.o lib/kernel/print.S
nasm -f elf -o build/kernel.o kernel/kernel.S
gcc  -I lib/ -I kernel/ -I lib/kernel/ -c -fno-builtin -o build/interrupt.o kernel/interrupt.c
gcc  -I lib/ -I kernel/ -I lib/kernel/ -c -fno-builtin -o build/init.o kernel/init.c
ld   -Ttext 0xc0001500 -e main -o build/kernel.bin build/main.o build/init.o build/interrupt.o build/print.o build/kernel.o
```

错误报告

```shell
#log
ld: i386 architecture of input file `build/print.o' is incompatible with i386:x86-64 output
ld: i386 architecture of input file `build/kernel.o' is incompatible with i386:x86-64 output
ld: build/interrupt.o: in function `idt_init':
interrupt.c:(.text+0x3dc): undefined reference to `__stack_chk_fail'
interrupt.c:(.text+0x3dc): relocation truncated to fit: R_X86_64_PLT32 against undefined symbol `__stack_chk_fail'
```

## 尝试解决

在程序的链接阶段，发生错误i386输入文件的体系结构与i386-x86_64不兼容。

查阅[1]，可以为链接的ld指令加上-m参数为其指定模拟仿真链接器，使用`ld -m -v`查看：

```shell
charon@charon-vm:~/MyOS$ ld -m -V
Supported emulations: elf_x86_64 elf32_x86_64 elf_i386 elf_iamcu elf_l1om elf_k1om i386pep i386pe
```

尝试在`ld`指令后加上`-m elf_i386`参数，再次尝试编译链接：

```shell
#log
ld: i386:x86-64 architecture of input file `build/main.o' is incompatible with i386 output
ld: i386:x86-64 architecture of input file `build/init.o' is incompatible with i386 output
ld: i386:x86-64 architecture of input file `build/interrupt.o' is incompatible with i386 output
ld: build/interrupt.o: in function `idt_init':
interrupt.c:(.text+0x3dc): undefined reference to `__stack_chk_fail'
```

给我们返回了栈检查不通过的报错，并且gcc在编译时生成的.o文件也要符合i386:x86-64架构。

查阅资料了解到，我们需要加上`fno-stack-protector`将编译时的栈检查关掉，但我有些好奇，因为我没有在gcc9.4的文档中直接找到这个参数的解释，下面是gcc文档中有关编译时栈保护的部分：

> -fstack-protector
>
> ​	Emit extra code to check for buffer overflows, such as stack smashing attacks.
> This is done by adding a guard variable to functions with vulnerable objects.
> This includes functions that call alloca, and functions with buffers larger than
> 8 bytes. The guards are initialized when a function is entered and then checked
> when the function exits. If a guard check fails, an error message is printed and
> the program exits.
>
> -fstack-check
> 	
>   Generate code to verify that you do not go beyond the boundary of the stack.
> You should specify this flag if you are running in an environment with multiple
> threads, but you only rarely need to specify it in a single-threaded environment
> since stack overflow is automatically detected on nearly all systems if there is
> only one stack.
> 	Note that this switch does not actually cause checking to be done; the operating
> system or the language runtime must do that. The switch causes generation of
> code to ensure that they see the stack being extended.
> 	You can additionally specify a string parameter: ‘no’ means no checking,
> ‘generic’ means force the use of old-style checking, ‘specific’ means use the
> best checking method and is equivalent to bare ‘-fstack-check’
>
> -fno-stack-limit
> 	
>   Generate code to ensure that the stack does not grow beyond a certain value,
> either the value of a register or the address of a symbol. If a larger stack is
> required, a signal is raised at run time. For most targets, the signal is raised
> before the stack overruns the boundary, so it is possible to catch the signal
> without taking special precautions.
> 	For instance, if the stack starts at absolute address ‘0x80000000’ and grows
> downwards, you can use the flags ‘-fstack-limit-symbol=__stack_limit’
> and ‘-Wl,--defsym,\__stack_limit=0x7ffe0000’ to enforce a stack limit of
> 128KB. Note that this may only work with the GNU linker.

下面是我对这些参数的一些理解~~（拙见）~~：

- `fstack-protector`：通过向部分（易受攻击）的对象添加保护变量实现，包括调用alloc申请内存的函数以及缓冲区大于8byte的函数，我印象里面好像是将它添加在栈底，==程序执行结束后判断其有没有被修改==，从而检测栈溢出这样常见的bug。
- `fstack-check`：通过生成代码验证是否超出堆栈边界，在多线程环境中需要指定此标志。
- `fno-stack-limit`：生成代码以确保堆栈增长不超过某个值。

着重关注一下`fstack-protector`参数，加上`fno-stack-protector`试试：

```shell
ld: i386:x86-64 architecture of input file `build/main.o' is incompatible with i386 output
ld: i386:x86-64 architecture of input file `build/init.o' is incompatible with i386 output
ld: i386:x86-64 architecture of input file `build/interrupt.o' is incompatible with i386 output
```

可以看到之前的栈保护检查已经没有了，但无法链接的问题仍然存在。

我们可能需要编译符合elf_i386规范的程序传递给ld进行链接。

查阅资料，相关文档如下：

> -m32
> -m64 Generate code for 32-bit or 64-bit ABI.

在编译时加上`m32`试试：

```shell
#log
Writing mbr, loader and kernel to disk...
1+0 records in
1+0 records out
512 bytes copied, 7.2326e-05 s, 7.1 MB/s
2+1 records in
2+1 records out
1348 bytes (1.3 kB, 1.3 KiB) copied, 6.5002e-05 s, 20.7 MB/s
22+1 records in
22+1 records out
11740 bytes (12 kB, 11 KiB) copied, 0.00116321 s, 10.1 MB/s
```

至此，问题解决。

### Ref

1. [Architecture of i386 input file is incompatible with i386:x86-64](https://stackoverflow.com/questions/19200333/architecture-of-i386-input-file-is-incompatible-with-i386x86-64)
2. [ld(1) - Linux man page](https://linux.die.net/man/1/ld)
3. [undefined reference to `__stack_chk_fail'](https://stackoverflow.com/questions/4492799/undefined-reference-to-stack-chk-fail)
4. [-fno-stack-protector is not working](https://stackoverflow.com/questions/50610889/fno-stack-protector-is-not-working)
5. [What is the use of -fno-stack-protector?](https://stackoverflow.com/questions/10712972/what-is-the-use-of-fno-stack-protector)
6. [how to pass -m elf_i386 to gcc?](https://stackoverflow.com/questions/11748970/how-to-pass-m-elf-i386-to-gcc)

