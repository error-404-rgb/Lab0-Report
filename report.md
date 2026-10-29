---
title: "计算机系统基础Ⅱ Lab0 - 实验环境配置与基本概念"
author: "姓名：范佳艺"
date: "2026年10月29日"
geometry: margin=2.5cm
fontsize: 12pt
linestretch: 1.5
header-includes:
  - \usepackage{xcolor}
  - \usepackage{tcolorbox}
  - \usepackage{hyperref}
  - \hypersetup{colorlinks=true, linkcolor=blue, urlcolor=blue, filecolor=magenta}
  - \newtcolorbox{codebox}{colback=gray!10!white, colframe=gray!60!black, boxrule=0.5pt, arc=2pt, left=5pt, right=5pt, top=5pt, bottom=5pt}
  - \newtcolorbox{infobox}{colback=blue!5!white, colframe=blue!60!black, boxrule=0.5pt, arc=2pt}
---

\tableofcontents
\newpage

# 一、 实验环境配置过程与截图

1. **虚拟机与系统环境**：使用 VirtualBox 安装了提供的 Kubuntu 镜像。

2. **共享文件夹配置**：在 VirtualBox 设置中添加共享文件夹，并在 Ubuntu 内通过 `mount` 命令成功挂载。

3. **获取源文件与编译**：将老师提供的源文件从共享文件夹复制到虚拟机的 `~/Lab0` 目录下。

![实验文件就绪](1_ls.png)

<p align="center"><i>图 1：实验文件就绪</i></p>

![编译成功](2_make.png)

<p align="center"><i>图 2：make clean all 编译成功</i></p>

\newpage

# 二、 任务 2.1：可重定位目标文件 vs 可执行文件

**1. 实验过程**

使用 `objdump` 工具生成反汇编文件，并使用 `diff` 命令比较差异。输入以下命令：

```bash
objdump -S -M intel,i386 hello > hello-1.txt
objdump -S -M intel,i386 hello.o > hello-2.txt
diff hello-1.txt hello-2.txt | head -n 40
```

终端输出的部分差异结果如下（左侧为可执行文件，右侧为目标文件）：

::: {.codebox}
```text
< Disassembly of section .init:
< 08049000 <_init>:
<  8049000:	53                   	push   ebx
<  8049001:	83 ec 08             	sub    esp,0x8
...
---
> Disassembly of section .text:
> 00000000 <test>:
>    0:	55                   	push   ebp
>    1:	89 e5                	mov    ebp,esp
```
:::

**2. 结果分析**

通过对比反汇编结果，发现可重定位目标文件（`.o`）和最终的可执行文件主要有以下区别：

::: {.infobox}
**分析结论：**
*   **外部函数调用地址不同**：在 `hello.o` 中，调用 `printf`、`malloc` 等外部函数时，`call` 指令后面的地址是 `0`（占位符），因为此时未链接。而在 `hello` 中，链接器已经将这些地址重定位为真实的虚拟内存地址（如 `0x804xxxx`）。
*   **段起始地址不同**：`hello.o` 的 `.text` 代码段起始偏移量通常从 `0x0` 开始；而最终的 `hello` 文件被分配了固定的虚拟地址（如 `0x08049000`）。
:::

\newpage

# 三、 任务 2.2：调试分析

**1. 使用 GDB 定位字符串内存地址**

使用 GDB 调试 `bar`，执行 `disas main` 后，观察到在 `call puts` 之前有指令 `push 0x80b4014`。因此字符串 `"Hello World."` 在内存中的虚拟地址为 **`0x80b4014`**。

![GDB 调试](4_gdb.png)

<p align="center"><i>图 3：GDB 查找字符串内存地址</i></p>

**2. 使用 readelf 查找程序头表**

执行 `readelf -l bar`，找到包含 `0x80b4014` 的 `LOAD` 段。记录该段的以下信息：

::: {.infobox}
*   文件偏移（Offset） = `0x06c000`
*   虚拟地址（VirtAddr） = `0x080b4000`
:::

![readelf 查看段表](5_readelf.png)

<p align="center"><i>图 4：readelf 查看程序头表</i></p>

**3. 计算并验证文件偏移**

计算公式为：`文件偏移 = 段的 Offset + (目标字符串地址 - 段的 VirtAddr)`

代入数据：`0x06c000 + (0x80b4014 - 0x080b4000) = 0x06c000 + 0x14 = 0x06c014`。

使用 `xxd` 命令查看文件特定位置的数据：

```bash
xxd -s 0x06c014 -l 16 bar
```

::: {.codebox}
```text
0006c014: 4865 6c6c 6f20 576f 726c 642e 0063 203d  Hello World..c =
```
:::

![xxd 验证](6_xxd.png)

<p align="center"><i>图 5：xxd 验证成功找到 Hello World</i></p>

\newpage

# 四、 问题与解答

::: {.infobox}
**Q1. 尝试用自己的话简单解释 Program Counter (或称 Instruction Pointer) 是什么？**

**答**：Program Counter（程序计数器，x86 中称 EIP）是 CPU 内部的一个特殊寄存器。它像一个指针，永远保存着下一条将要执行的指令在内存中的地址。CPU 不断地读取 PC 指向的指令、执行，并更新 PC，使程序按顺序流转。
:::

::: {.infobox}
**Q2. 说说段在进程加载中所起到的作用。**

**答**：段在加载时起映射和权限保护的作用。操作系统根据 Program Headers 将文件映射到内存；且不同段有不同属性，如代码段（只读可执行防篡改），数据段（可读写存变量），实现了内存的安全隔离。
:::

::: {.infobox}
**Q3. 二进制中的 4865 6c6c ... 是怎么对应到 Hello World 的？**

**答**：计算机底层使用 ASCII 码存储字符。例如 `H` 的十六进制 ASCII 码是 `0x48`。字符串在内存中是将每个字符的 ASCII 码按顺序连续排列：`48 65 6c 6c 6f 20 57 6f 72 6c 64 2e`，末尾的 `00` 是字符串结束标志。
:::

# 五、 实验总结

通过亲手逆向计算字符串在 ELF 文件中的偏移，我对程序头表、虚拟地址、文件偏移之间的关系有了非常具象的认识，明白了程序运行时的内存数据是如何从硬盘上的二进制文件中“搬运”过来的。
