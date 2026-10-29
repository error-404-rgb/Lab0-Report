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
  - \hypersetup{colorlinks=true, linkcolor=blue!80!black, urlcolor=blue}
---

\tableofcontents
\newpage

# 实验目的

1. 熟悉 Linux 下 C 语言程序的编译过程与常用工具链（GCC、Makefile）。
2. 掌握 ELF（可执行与可链接格式）文件的基本结构。
3. 能够结合 `GDB` 与 `readelf` 等工具，从二进制层面分析程序在内存与硬盘中的映射关系。

# 一、 实验环境配置过程与截图

本次实验在 VirtualBox 提供的 Kubuntu 虚拟机环境中进行。首先需要确保宿主机与虚拟机的文件互通。

1. **配置共享文件夹**：在 VirtualBox 设置中添加共享文件夹，并在 Ubuntu 终端内通过 `mount` 命令成功挂载。
2. **获取源文件与编译**：将老师提供的源文件从共享文件夹复制到虚拟机的 `~/Lab0` 目录下，使用提供的 Makefile 进行编译。

```bash
cp /mnt/icslab/* .  # 将源文件拷贝至当前目录
make clean all      # 清理并重新编译
```

\begin{center}
\includegraphics[width=0.8\textwidth]{1_ls.png}

\vspace{0.2cm}
\textit{图 1：实验文件就绪}
\end{center}

\begin{center}
\includegraphics[width=0.8\textwidth]{2_make.png}

\vspace{0.2cm}
\textit{图 2：执行 make clean all 编译成功}
\end{center}

\newpage

# 二、 任务 2.1：可重定位目标文件 vs 可执行文件

**1. 实验过程**

为了对比 `.o` 文件和最终可执行文件的区别，我们使用 `objdump` 工具分别生成它们的反汇编代码，并提取前 40 行差异进行对比：

```bash
objdump -S -M intel,i386 hello > hello-1.txt
objdump -S -M intel,i386 hello.o > hello-2.txt
diff hello-1.txt hello-2.txt | head -n 40
```

终端部分输出差异如下（`<` 代表可执行文件，`>` 代表目标文件）：

\begin{tcolorbox}[colback=gray!10, colframe=gray!60, title=\textbf{终端输出差异}]
```text
< Disassembly of section .init:
< 08049000 <_init>:
<  8049000:	53                   	push   ebx
...
---
> Disassembly of section .text:
> 00000000 <test>:
>    0:	55                   	push   ebp
>    1:	89 e5                	mov    ebp,esp
```
\end{tcolorbox}

**2. 结果分析与思考**

观察两份 `objdump` 的输出，最直观的感受是：在 `hello.o` 文件里，代码调用了 `printf`、`malloc` 等外部函数时，`call` 指令后面跟的地址全是 `0`。而在最终生成的 `hello` 文件里，这些位置都填上了真实的虚拟地址（如 `0x804xxxx`）。

\begin{tcolorbox}[colback=blue!5!white, colframe=blue!50!black, title=\textbf{原因分析}]
这是因为我之前理论课上学过的：编译器只负责把 C 代码翻译成汇编，它并不知道外部函数最终会加载到内存的哪里。因此，在生成 `.o` 文件时，它只能先填个 `0` 占位。只有等链接器（Linker）进场，把所有目标文件打包在一起，并分配了最终的运行地址后，才会把这些占位符替换成真实的地址。

另外，`.o` 文件的代码段起始地址通常是从 `0x0` 开始算偏移，而最终生成的可执行文件，由于加了 `-no-pie` 参数，它被固定分配到了 `0x08049000` 这段高的虚拟内存区域。
\end{tcolorbox}

\newpage

# 三、 任务 2.2：调试分析

**1. 使用 GDB 定位字符串内存地址**

本节我们分析 `bar` 文件。进入 GDB 后，通过 `disas main` 查看反汇编代码，可以在 `call puts` 之前观察到一条指令：`push 0x80b4014`。

由于 `puts` 是打印字符串的函数，所以压入栈的 `0x80b4014` 就是 `"Hello World."` 这个字符串在虚拟内存中的首地址。

\begin{center}
\includegraphics[width=0.7\textwidth]{4_gdb.png}

\vspace{0.2cm}
\textit{图 3：GDB 反汇编查找字符串地址}
\end{center}

**2. 使用 readelf 查找程序头表**

知道内存地址后，我们需要搞清楚操作系统是如何把它从硬盘加载进来的。执行 `readelf -l bar` 查看段表：

\begin{tcolorbox}[colback=gray!10, colframe=gray!60, title=\textbf{Program Headers (部分)}]
```text
Type   Offset   VirtAddr   PhysAddr   FileSiz  MemSiz   Flg Align
LOAD   0x06c000 0x080b4000 0x080b4000 0x31710  0x31710  R   0x1000
```
\end{tcolorbox}

\begin{tcolorbox}[colback=blue!5!white, colframe=blue!50!black, title=\textbf{数据提取}]
找到包含目标地址 `0x80b4014` 的 `LOAD` 段后，我们得出：
\begin{itemize}
    \item 文件偏移（Offset） = \texttt{0x06c000}
    \item 虚拟地址（VirtAddr） = \texttt{0x080b4000}
\end{itemize}
\end{tcolorbox}

\begin{center}
\includegraphics[width=0.8\textwidth]{5_readelf.png}

\vspace{0.2cm}
\textit{图 4：readelf 查看程序头表}
\end{center}

**3. 计算并验证文件偏移**

根据映射公式，我们要找的字符串在硬盘文件中的实际位置为：
`文件偏移 = 段的 Offset + (目标字符串地址 - 段的 VirtAddr)`

代入我们的数据：
`0x06c000 + (0x80b4014 - 0x080b4000) = 0x06c000 + 0x14 = 0x06c014`

最后，使用 `xxd` 命令去文件中验证这个位置的内容：

```bash
xxd -s 0x06c014 -l 16 bar
```

\begin{tcolorbox}[colback=gray!10, colframe=gray!60, title=\textbf{xxd 验证结果}]
```text
0006c014: 4865 6c6c 6f20 576f 726c 642e 0063 203d  Hello World..c =
```
\end{tcolorbox}

右侧的 ASCII 字符成功显示出 `Hello World.`，说明我们从内存地址逆向计算文件物理偏移的思路完全正确！

\begin{center}
\includegraphics[width=0.8\textwidth]{6_xxd.png}

\vspace{0.2cm}
\textit{图 5：xxd 成功验证字符串位置}
\end{center}

\newpage

# 四、 问题与解答

\begin{tcolorbox}[colback=cyan!5!white, colframe=cyan!50!black, title=\textbf{Q1. 尝试用自己的话简单解释 Program Counter (或称 Instruction Pointer) 是什么？}]
\textbf{答：}在 x86 架构中它通常被称为 EIP 寄存器。你可以把它理解为一个永远指向“下一条指令”的指针。CPU 执行程序的过程，就是不停地按照 EIP 指向的地址去内存里把指令取出来并执行。每执行完一条指令，EIP 会自动加上该指令的长度，跳到下一条指令的位置。
\end{tcolorbox}

\begin{tcolorbox}[colback=cyan!5!white, colframe=cyan!50!black, title=\textbf{Q2. 说说段在进程加载中所起到的作用。}]
\textbf{答：}段（Segment）起到了“按需搬运”和“权限隔离”的作用。操作系统根据 Program Headers 里的标记，把文件里的内容映射到内存中。比如只读的代码段会被赋予 `R E`（可读可执行）权限，防止程序运行时被恶意篡改；而数据段会被赋予 `RW`（可读写）权限，用来存放我们定义的全局变量。
\end{tcolorbox}

\begin{tcolorbox}[colback=cyan!5!white, colframe=cyan!50!black, title=\textbf{Q3. 二进制中的 4865 6c6c ... 是怎么对应到 Hello World 的？}]
\textbf{答：}计算机底层存储的都是二进制（即字节）。为了方便显示，`xxd` 用十六进制把它们展示了出来。字符 `H` 在 ASCII 码表中对应的十进制是 72，十六进制就是 `48`。同理，`e` 是 `65`。因此，连续的字母在文件中就是连续排列的 ASCII 码字节流：`48 65 6c 6c 6f ...`，最后的 `00` 是 C 语言中约定的字符串结束符 `\0`。
\end{tcolorbox}

# 五、 实验总结

这次实验让我对平时经常写的 C 语言代码有了全新的认识。以前只知道代码点一下运行就能出结果，现在明白了从 `.c` 源码到 `.o` 文件，再到内存里的进程，中间经历了极其严谨的重定位和映射过程。

尤其是最后用 `xxd` 在二进制文件里成功挖出 `Hello World` 字符串的那一刻，非常有成就感。这让我真正理解了程序头表（Program Headers）、虚拟地址（VirtAddr）和文件偏移（Offset）之间的物理联系，也为后续的逆向分析与调试打下了扎实的基础。
