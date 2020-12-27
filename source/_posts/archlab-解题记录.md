---
title: archlab 解题记录
date: 2020-12-01 20:58:49
tags: archlab csapp
keywords:
categories: csapp解题记录
description: 最近心血来潮，打算把csapp课程中，除了课内要求的bomb lab和attack lab以外的几个lab做掉，现在面对的第一个就是实验四——arch lab
summary_img:
---

临近期末，不如来点好玩的吧

<!-- more -->

## 实验说明

这个 lab 的实验说明就比较劝退，我先看了开头的 Part A 部分，大致意思是，课程设置了一种新的指令集：`Y86-64`，相对于 `x86` 指令集精简了很多，
以用来进行实验，幸好的是，经过前两个 lab 的摧残，已经对汇编代码有~~抗性~~较好的认识了。

## 事前准备

首先使用 `tar -zvf archlab-handout.tar` 解压实验文件压缩包，然后运行

```bash
cd sim
make clean; make
```

编译工具链，期间遇到了 `/usr/bin/ld: cannot find -ltk /usr/bin/ld: cannot find -ltcl` 找不到两个库的错误，看了下这两个库应该是 gui 相关的，理论上来说可以直接在 makefile 里关掉，或者也可以下载安装

```bash
apt install -y tk-dev tcl-dev
```

### Part A

进入正题，part a 主要要求我们使用 Y86 指令翻译一些 C 语言程序指令，其中需要翻译的 `examples.c` 如下：

```C
typedef struct ELE {
    long val;
    struct ELE *next;
} *list_ptr;

/* sum_list - Sum the elements of a linked list */
long sum_list(list_ptr ls)
{
    long val = 0;
    while (ls) {
	val += ls->val;
	ls = ls->next;
    }
    return val;
}

/* rsum_list - Recursive version of sum_list */
long rsum_list(list_ptr ls)
{
    if (!ls)
	    return 0;
    else {
        long val = ls->val;
        long rest = rsum_list(ls->next);
        return val + rest;
    }
}

/* copy_block - Copy src to dest and return xor checksum of src */
long copy_block(long *src, long *dest, long len)
{
    long result = 0;
    while (len > 0) {
        long val = *src++;
        *dest++ = val;
        result ^= val;
        len--;
    }
    return result;
}
/* $end examples */

```

#### sum_list

为了测试，我们首先编写运行框架

```x86asm
.pos 0
    irmovq stack, %rsp
    call main
    halt

.align 8
ele1:
.quad 0x00a
.quad ele2
ele2:
.quad 0x0b0
.quad ele3
ele3:
.quad 0xc00
.quad 0


main:
    irmovq ele1, %rdi
    ret

.pos 0x200
    stack:

```

接下来，编写 `sum` 函数：

```x86asm
sum:
    irmovq $0, %rax
    andq %rdi, %rdi
    jmp test
loop:
    mrmovq (%rdi), %r10
    addq %r10, %rax
    mrmovq 8(%rdi), %rdi
    andq %rdi, %rdi
test:
    jne loop
    ret

```

函数逻辑很简单，不多做解释，有几个要注意的点是：

- Y86 指令集只支持偏移寻址，即 `instant(%reg)`
- Y86 指令集不支持直接使用内存地址来进行算数运算

#### rsum_list

使用递归的方法遍历链表并且将其中的值相加

```x86asm
rsum:
    xorq %rax, %rax
    andq %rdi, %rdi
    je return
    mrmovq (%rdi), %r10
    addq %r10, %rax
    pushq %rax
    mrmovq 8(%rdi), %rdi
    call rsum
    popq %r10
    addq %r10, %rax
return:
    ret
```

需要注意的点：%rax 寄存器为调用者保存寄存器(caller saved register)，如果要在本函数中继续使用，需要将其 push 入栈，然后再 pop。
代码中的%r10 寄存器理论上也是调用者保存的，但是由于接下来的代码中并没有需要其保存值使用的，因此并没有将其入栈。

### 2020/12/6 Updated

#### copy_block

用于从一个块中复制指定长度的内容到另一个块中，同样是使用循环结构，但是这一次需要加上条件判断，这里使用到的有：

`subq $0, %rdx`：（pseudo）将%rdx 中的内容减去 0，设置 CF（最高位是否进位^是否是减法）、SF（结果是否为负数）、ZF（结果是否为 0），用于跳转使用（或者可以简单理解为`cmp $0, %rdx`，然鹅 Y86 并没有支持）

`jg body`：如果%rdx > 0，则跳转，是否跳转取决于`~(SF^OF)&~ZF`

下面贴上 Y86 代码：

```x86asm
copy_block:
    # src:%rdi dest:%rsi len:%rdx
    xorq %rax, %rax
    jmp tst
body:
    irmovq $8, %r10
    mrmovq (%rdi), %r11
    addq %r10, %rdi
    rmmovq %r11, (%rsi)
    addq %r10, %rsi
    xorq %r11, %rax
    irmovq $1, %r10
    subq %r10, %rdx
tst:
    xorq %r10, %r10
    subq %r10, %rdx
    jg body
    ret
```

至此，Part A 告一段落，下面进入 Part B

### Part B

阅读题面：

Your task in Part B is to extend the SEQ processor to support the `iaddq`.

很明显，这道题需要我们使用 `HCL` 语言来实现 `iaddq` 指令，即允许直接给一个寄存器加上一个立即数。参照 CSAPP 的图 4-18 我们可以做出 iaddq 的整个流程：

| 阶段    | `Opq rA, rB`                                                                    | `iaddq V, rB`                                                                                          |
| ------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| 取指    | icode: ifun ← M<sub>1</sub>[PC] <br>rA:rB←M<sub>1</sub>[PC+1]<br><br>valP← PC+2 | icode: ifun ← M<sub>1</sub>[PC]<br>rA:rB←M<sub>1</sub>[PC+1]<br>valC←M<sub>8</sub>[PC+2]<br>valP←PC+10 |
| 译码    | valA←R[rA]<br>valA←R[rA]                                                        | valB←R[rB]                                                                                             |
| 执行    | valE←valB OP valA                                                               | valE←valB + valC                                                                                       |
| 访存    |                                                                                 |                                                                                                        |
| 写回    | R[rB]←valE                                                                      | R[rB]←valE                                                                                             |
| 更新 PC | PC←valP                                                                         | PC←valP                                                                                                |

**_valC、valE、valP_**：分别代表运算所 fetch 的常数、alu 的运算结果、命令执行完后下一条命令的地址。

这么一看，iaddq 其实和 irmovq 指令是基本完全相同的，但是要指出以下两个不同点：

1. iaddq 指令需要获取 rB 寄存器中的值参与运算，而 irmovq 则不需要。
2. iaddq 指令属于运算指令，需要设置 Flag 的值改变，而 irmovq 不需要。

因此总的算下来，我们在`seq-full.hcl`中需要区别于 IIRMOVQ 的只有两个地方：

```bash
################ Decode Stage    ###################################

## What register should be used as the B source?
word srcB = [
	icode in { IOPQ, IRMMOVQ, IMRMOVQ ,IIADDQ } : rB;
	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
	1 : RNONE;  # Don't need register
];
```

```bash
## Select input B to ALU
word aluB = [
	icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL,
		      IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
	icode in { IRRMOVQ, IIRMOVQ } : 0;
	# Other instructions don't need ALU
];
```

结果就是将立即数与寄存器中的值相加，存回寄存器中，按照实验指导书上的指令 pass 掉了所有的 benchmark，进入下一个阶段。

### Part C

此坑待填
