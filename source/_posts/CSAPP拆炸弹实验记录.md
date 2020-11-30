---
title: CSAPP拆炸弹实验记录
date: 2020-09-17 09:18:50
tags: CSAPP bomblab 拆炸弹
keywords: CSAPP bomblab 拆炸弹
categories: CSAPP
description: CSAPP作业，拆炸弹实验记录
summary_img:
---
环境选择`ubuntu 20.04 LTS`，使用`objdump`工具进行反汇编

```bash
objdump -M intel -d bomb > bomb.asm.txt
```

这里记得需要指定目标格式为`intel`的汇编标准，否则默认得到的是`AT&T`标准的汇编码。

### Glance

我们可以首先看一下`bomb.c`的源代码，从中可以摘抄出如下的主体代码：

```c
input = read_line();
phase_1(input);
phase_defused();
printf("Phase 1 defused. How about the next one?\n");
input = read_line();
phase_2(input);
phase_defused();
printf("That's number 2.  Keep going!\n");
input = read_line();
phase_3(input);
phase_defused();
printf("Halfway there!\n");
input = read_line();
phase_4(input);
phase_defused();
printf("So you got that one.  Try this one.\n");
input = read_line();
phase_5(input);
phase_defused();
printf("Good work!  On to the next...\n");
input = read_line();
phase_6(input);
phase_defused();

```

可以看到，每个phase都会进行一次`read_line`，然后交由对应的函数处理，理解了这些就可以开始做题了。

### 0x1 phase_1

```assembly
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    rsp,0x8
  400ee4:	be 00 24 40 00       	mov    esi,0x402400
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
  400eee:	85 c0                	test   eax,eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    rsp,0x8
  400efb:	c3                   	ret    
```

phase_1是最简单的一个阶段：首先将读入的字符串与目标字符串相比较，如果返回值不为0（寄存器test自己等于判断是否为0）则成功拆弹，反之则拆弹失败，其中目标字符串没法直接通过反汇编得到，于是在`gdb`中在`0x400ee0`处打断点，使用`x/s 0x402400`查看目标字符串，具体不再赘述。



