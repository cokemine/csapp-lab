## BombLab

### Preparation

需要留下几张图以备随时翻阅。

1. x86-64 常见寄存器及用途

![https://i.imgur.com/d9KWNdb.png](https://i.imgur.com/d9KWNdb.png)

主要是以下几个寄存器：

- `%rdi` 保存第一个参数

- `%rsi` 保存第二个参数

- `%rdx` 保存第三个参数

- `%rcx` 保存第四个参数

- `%rsp` 指向栈顶

- `%rip` 程序计数器，指向当前执行代码的位置

2. 常见指令

2. 常用 gdb 调试指令

通过观察 bomb.c 的源代码，可以发现该程序执行的逻辑上读入数据（可以从文件读入，也可以从 STDIN 读入）依次有 6 个炸弹：phase_(1-6) 通过最后的注释可以发现还有一个隐藏炸弹。

不过我们能看到的只有 main 函数的源码，看不到各个炸弹函数（phase_(1-6)）的源码（要是能看到那还需要拆炸弹吗.....）

先使用 objdump 工具反汇编 bomb 二进制文件。

```bash
objdump -d bomb > bomb.asm
```

### phase_1

先 run 一下程序，输出如下

> ❯ ./bomb
> Welcome to my fiendish little bomb. You have 6 phases with
> which to blow yourself up. Have a nice day!

随便输入一个字符串，程序表示炸弹被引爆了。Emmm... 接下来我们应该输入的是第一个炸弹的密码。

由于 main 函数的逻辑直接看源码就能搞明白，我们直接看反汇编后的 phase_1 这个函数的汇编代码。值得注意的是从源码看出在 main 函数中，我们输入的字符串被传入到了每一个 phase_x 函数的第一个参数，那么应该在进入 phase_x 函数前，将我们输入的字符串的地址放到 `%rdi` 寄存器中。实际上从反汇编的结果来看也是这样做的。

```assembly
400e32:	e8 67 06 00 00       	call   40149e <read_line> # 调用 read_line 函数，并将返回值保存到 %rdi 寄存器
400e37:	48 89 c7             	mov    %rax,%rdi
```

phase_1 函数汇编代码：

```assembly
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi # 将一个地址放到了 %esi 寄存器，也就是放第二个传参的寄存器
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal> # 调用 strings_not_equal 函数并判断返回值是否是 0
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17> #是 0 则跳转 非 0 则调用 explode_bomb 函数导致炸弹引爆
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	ret   
```

这个炸弹还是很容易的，可以推测 strings_not_equal 有两个参数，第一个参数是我们传入的参数（一直在 `%rdi` 寄存器里）第二个参数是 `0x402400`这个地址。这个函数的作用应该是判断传入两个字符串是否相同。如果不同则返回 1，相同则返回 0。通过后面的汇编代码也可以看到如果返回值不是 0 就会调用 explode_bomb 函数导致炸弹被引爆。那么我们只需要看看 `0x402400` 这个地址或程序执行`0x400ee9`时，`%esi` 到存的字符串是什么就好了。

顺便一提这个炸弹还不让人随便退出...

> So you think you can stop the bomb with ctrl-c, do you?

~~由于 objdump 反汇编得到的结果是虚拟地址，需要 run 一下才能看到数据真正存放的地址在哪里。~~

上面这句话是我本地测试学习 gdb 指令得出的结论，但结果发现 bomb 这个程序每次运行时地址是固定的。 stackoverflow 上搜了一下大概是链接器的原因。

基于上述结论，我们可以直接在 不打断点不执行 run 的情况下拿到这个值.....

```assembly
(gdb) x/s 0x402400
0x402400:       "Border relations with Canada have never been better."
```

**答案**：Border relations with Canada have never been better.

### phase_2

先观察一下 phase_2 的汇编代码：

```assembly
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp # 调用者保存栈
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp # 开辟 28 个字节新的栈空间
  400f02:	48 89 e6             	mov    %rsp,%rsi # 栈顶指向的地址放到第二个参数
  400f05:	e8 52 05 00 00       	call   40145c <read_six_numbers> # 调用函数
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	call   40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	ret    
```

phase_2 没有见到有使用我们输入的字符串（存放在 `%rdi` 中），显然 read_six_numbers 函数第一个参数是我们输入的字符串，第二个参数是栈顶指向的地址，根据这个函数的名称推测第二个参数应该是个数组，应该是从我们输入的字符串中得到了 6 个数字然后放到了数组里。

看一下 read_six_numbers 的代码

```assembly
000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
  401474:	48 89 04 24          	mov    %rax,(%rsp)
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	call   400bf0 <__isoc99_sscanf@plt> #调用了 sscanf 函数
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	call   40143a <explode_bomb>
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	ret    
```

看似比较复杂，不过在其中调用了 sscanf 函数，而且使用了不少参数寄存器用于给 sscanf 传参。合理推测它用 sscanf 函数格式化了我们输入的字符串并从中获得了 6 个数字，由于 sscanf 函数第二个参数是格式字符串。可以看看 `0x4025c3` 地址存放的格式字符串是什么。

```assembly
(gdb) x/s 0x4025c3
0x4025c3:       "%d %d %d %d %d %d"
```

显然我们输入的数据的格式应为6个整数，格式是以空格分割。再分析调用 sscanf 时传入参数的次序，字符串转数组后数组的第一个元素应该对应我们六个数字中的第一个数。

接下来回头看一看 phase_2 的代码。phase_2 用了很多跳转指令，可以依照它的逻辑来分析（

```assembly
400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp) # 判断栈顶元素（栈顶地址所存整数）是否为 1，因为栈顶就是我们所创建数组的第一个元素，从这里可以看出我们第一个整数是 1
400f0e:	74 20                	je     400f30 <phase_2+0x34> # 如果是 1 就跳转到 0x0f30 则引爆炸弹，那我们跟着他跳转
400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx # %rbx = 数组的第二个元素（的地址）
400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp # %rbp 存放 %rsp + 24 因为一共有 6 个数，一个整型 4 字节，推测目的是防止数组越界
400f3a:	eb db                	jmp    400f17 <phase_2+0x1b> # 跳转到 0x400f17
# 为什么空一行因为下面是循环体
400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax # %eax = 数组的第二个元素（的地址）- 0x4 = 数组第一个元素（的地址）
400f1a:	01 c0                	add    %eax,%eax # %eax = %eax + %eax = 数组第一个元素 * 2 = 2
400f1c:	39 03                	cmp    %eax,(%rbx) # 比较 %eax（数组第一个元素的2倍） 和 %rbx（数组第二个元素）
400f1e:	74 05                	je     400f25 <phase_2+0x29> # 如果相等就跳转到 0x400f25 否则引爆炸弹，从这可以发现第二个整数应该是 2
400f25:	48 83 c3 04          	add    $0x4,%rbx # %rbx = %rbx + 0x4，那么 %rbx 现在指向的应该是数组里的第三个元素
400f29:	48 39 eb             	cmp    %rbp,%rbx # 比较  %rbx 和 %rbp，如果都减去一开始的栈顶地址那应该比较的是 8 和 24，上面的猜想得到了验证
400f2c:	75 e9                	jne    400f17 <phase_2+0x1b> # 不相等跳转到 0x400f17 可以看到我们已经执行过一次这个指令了，说明开始了循环

400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40> #相等的话退出循环 phase_2 执行完成。
```

上面的注释我个人认为写的非常清楚了，本质上这个递推就是
$$
a[0] = 1; a[i] = a[i - 1] * 2\\
i \in [1,6)
$$
**答案**：1 2 4 8 16 32

### phase_3

先观察一下 phase_3 的汇编代码：

```assembly
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>  #调用了 sscanf 函数
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	ret    
```

phase_3 又调用了 sscanf 函数，说明我们输入的字符串又经过格式化被转换成了数字，还是得看下格式字符串是什么。

```assembly
(gdb) x/s 0x4025cf
0x4025cf:       "%d %d"
```

显然我们输入的应该为两个整型数组，以空格分隔。

从参数的顺序可以看到格式化后的第一个数字放到了 `%rsp + 0x8`，第二个数字被放到了 `%rsp + 0xc`

```assembly
400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
```

观察到 `0x400f6a` 如果第一个数字大于 7 就会调用炸弹爆炸函数。说明第一个数字应该是小于等于 7 的。

```assembly
400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
# 下面一行是 0x400fad
400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
```

这里有一个跳转表，推测可能是用了 switch 语句。在看到后续代码可以判断。根据第一个数的不同，第二个数的答案也不同

```assembly
400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8) # %rax*8+0x402470 的地址
# 退出 switch 后比较输入第二个数字和 %eax 是否相等，switch分支内针对每一个不同的第一个数，%eax 也被赋予了不同的值
400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
```

Anyway，其实也不需要知道这是个 switch 语句，更简单的方法是已经知道第一个数是 0 - 7 了，只需要先把第一个数确定下来，也不用管跳转表，因为第一个数确定后跳转到哪一个指令也是确定的。然后打上断点 `stepi` 单步调试，偷窥一下 `%eax` 寄存器里保存的是什么就可以了。

**答案**： 

```
0 207
1 311
2 707
3 256
4 389
5 206
6 682
7 327
```

### phase_4

先看汇编代码

```assembly
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt> # 调用 sscanf
  401029:	83 f8 02             	cmp    $0x2,%eax  # sscanf 返回值为格式化的组数，从这也可以看出来我们格式化后应为两组数据（后续观察格式字符串可知应为两个数字）
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp) # a[0] 和 0xe 比较
  401033:	76 05                	jbe    40103a <phase_4+0x2e> # a[0]应该小于等于 0xe 否则炸弹爆炸
  401035:	e8 00 04 00 00       	call   40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx # %edx = 0xe # 第三个参数为 0xe （推测下面为调用另一个函数做准备）
  40103f:	be 00 00 00 00       	mov    $0x0,%esi # %edx = 0x0 # 第二个参数为 0x0
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi # %edi = a[0] 第一个参数为我们输入的字符串被格式化后的第一个数字
  401048:	e8 81 ff ff ff       	call   400fce <func4> #调用 func4
  40104d:	85 c0                	test   %eax,%eax # 判断 func4 函数返回值是否为 0，非 0 则炸弹爆炸
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp) #判断 a[1] 是否为 0 非 0 则炸弹爆炸
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	ret    
```

它又调用了 sscanf，看了看格式字符串，说明这个炸弹的密码还是两个数字

```assembly
(gdb) x/s 0x4025cf
0x4025cf:       "%d %d"
```

继续分析可以得知（见注释）第二个数字一定是 0，但是第一个数字一定小于 14，具体还无从得知。

~~其实第一个数可以从0开始试到14，反正肯定能试出来~~

为了分析第一个数字为何值，需要看看它调用的函数 func4，调用 func4 应该为 func4(a[0], 0, 14)，从这个函数名看不出什么有效信息，还是得深入看看。

显然这是一个递归。假设这个函数是 func func4(x, y, z int) (ret int)

```assembly
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax # ret = z | ret = 14
  400fd4:	29 f0                	sub    %esi,%eax # ret = ret - y | ret = 14 - 0 = 14
  400fd6:	89 c1                	mov    %eax,%ecx # %ecx = ret | %ecx = 14
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx # %ecx = %ecx >> 31 | %ecx = 0
  400fdb:	01 c8                	add    %ecx,%eax # ret = ret + z | ret = 0 + 14 = 14
  400fdd:	d1 f8                	sar    %eax # ret = ret >> 1 | ret = 7
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx # %ecx = ret + y * 1 | %ecx = 7
  400fe2:	39 f9                	cmp    %edi,%ecx # 比较 x 和 %ecx | 比较 a[0] 和 7
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24> # 如果 a[0] <= 7 跳转
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx # 否则 z = %ecx - 1 | z = 6
  400fe9:	e8 e0 ff ff ff       	call   400fce <func4> # 递归调用 func4(x, y, z - 1) | func4(a[0], 0, 6)
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax # 设置返回值为 0
  400ff7:	39 f9                	cmp    %edi,%ecx # 比较 x 和 %ecx | 比较 a[0] 和 7
  400ff9:	7d 0c                	jge    401007 <func4+0x39> # 如果 x >= %ecx 退出递归 | a[0] 应大于等于 7
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi # 否则 y = %ecx + 1
  400ffe:	e8 cb ff ff ff       	call   400fce <func4> # func(x, %ecx + 1, z)
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	ret    
```

递归比较难分析，真正让我自己根据汇编代码反编译出来 C Code 有些难度。但最好的情况下我们让这个函数 func4 只执行一次。通过对汇编代码的分析，可以知道第一次执行 func4，当$$7 <= x <= 7$$ 时就不会递归执行 func4 而是直接返回 0

那本体最优解即为 `7 0`（因为 func4 只执行了一次，这也是最快的解）当然如果深入探究这个递归函数或者暴力枚举 1-14 的话

**答案**： 

```
0 0
1 0
3 0
7 0
```

### phase_5

还是先看汇编代码

```assembly
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx # %rbx = %rdi = 输入的字符串
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	call   40131b <string_length> # 调用了 string_length 函数 第一个参数是输入的字符串，推测是判断我们输入字符串的长度
  40107f:	83 f8 06             	cmp    $0x6,%eax # 我们输入的字符串长度必须为 6 否则炸弹爆炸
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70> # 跳转到 0x4010d2 后将 %eax 设置为 0 又跳回 0x40108b 了
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx # %ecx=%rbx+%rax %rax此时为 0，%ecx 为输入字符串的第一个字符
  40108f:	88 0c 24             	mov    %cl,(%rsp) # %cl 就是 %ecx
  401092:	48 8b 14 24          	mov    (%rsp),%rdx # 和上面的语句一起看，就是把 %ecx 的值赋值给了 %rdx
  401096:	83 e2 0f             	and    $0xf,%edx # %edx = %edx & 0xf。其实就是取 %edx 的低四位（就是第一个字符 ASCII码 的第四位）
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx # 将 0x4024b0 + %edx 放到 %edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1) # 存放所得结果到 %rsp+0x10+%rax
  4010a4:	48 83 c0 01          	add    $0x1,%rax # %rax 自增
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax # 比较 %rax 和 6
  4010ac:	75 dd                	jne    40108b <phase_5+0x29> # 跳转
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi # 把 0x40245e 移到 %esi第一个参数的位置）
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi # 把 0x10 + %rsp 移到 %esi 第二个参数的位置
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal> # 判断两个字符串是否相等
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00 
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	call   400b30 <__stack_chk_fail@plt>
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	ret    
```

phase_5 中调用 string_length 并判断字符串长度是否为 6，同时又有以下语句

```assembly
4010a4:	48 83 c0 01          	add    $0x1,%rax # %rax 自增
4010a8:	48 83 f8 06          	cmp    $0x6,%rax # 比较 %rax 和 6
4010ac:	75 dd                	jne    40108b <phase_5+0x29> # 跳转
```

这是一个显然的循环结构，翻译到 C 语言即为

```c
while(i < 6) {
    ...
    i++;
}
```

从汇编代码中可以推测看到 `0x4024b0` 和 `0x40245e` 地址和存放的是两个固定的字符串。

```assembly
(gdb) x/s 0x4024b0
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
(gdb) x/s 0x40245e
0x40245e:       "flyers"
```

由于调用 `strings_not_equal`函数时，比较的是 `0x10(%rsp)` 和 "flyers" 两个字符串的大小，那 `0x10(%rsp)` 从哪里来呢？明显循环体的意义就是在构造在 `0x10(%rsp)` 存放的字符串。所以可以很简单的翻译出 C 语言代码如下

```c
char* dict = "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?";
char* ans = "flyers";
int phase_5(char* input) {
    char s[6];
    int i = 0;
    while(i != 6) {
        s[i] = dict[input[i] & 0xf];
        i++;
    }
    return !strings_not_equal(ans, s);
}
```

也就是说我们输入的6个字符的第四位作为偏移量，用 `maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?` 这段字符串作为字典去构造一个新的字符串，需要使得构造出的字符串为 flyers。根据字典。我们输入的 6 位字符串每一个字符的低四位依次为 `9 F E 5 6 7` 然后查表找合适的字符就可以了，有多种可能性。

我认为这个题目是前 5 个里除了第一个之外最简单的了，毕竟循环结构要比递归和 switch 好理解多了

**答案**： 低四位依次为 `9 F E 5 6 7` 构造出的任意一个由 6 个字符组成的字符串

### phase_6

下面是最后一个炸弹，汇编代码比较多，分开来看比较好。

首先是一些初始化操作。包括调用者保存寄存器、开辟栈空间。读入了6个数字，进行第一次判定。

```assembly
00000000004010f4 <phase_6>:
  4010f4:	41 56                	push   %r14 # 调用者保存寄存器
  4010f6:	41 55                	push   %r13
  4010f8:	41 54                	push   %r12
  4010fa:	55                   	push   %rbp
  4010fb:	53                   	push   %rbx
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13 # %r13 = %rsp
  401103:	48 89 e6             	mov    %rsp,%rsi # %rsi = %rsp
  401106:	e8 51 03 00 00       	call   40145c <read_six_numbers> # 又是读入 6 个数字，说明这次的密码还是 6 数字
  40110b:	49 89 e6             	mov    %rsp,%r14 # %r14 = %rsp
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d # %r12d = 0x0
  401114:	4c 89 ed             	mov    %r13,%rbp # %rbp = %r13
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax # %eax = a[0]
  40111b:	83 e8 01             	sub    $0x1,%eax  # %eax = a[0] - 1;
  40111e:	83 f8 05             	cmp    $0x5,%eax # %eax = a[0] - 1 <= 5 -> a[0] <= 6;
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	call   40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d # r12d = 0 + 1 = 1;
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d # r12d 需要小于 6
  401130:	74 21                	je     401153 <phase_6+0x5f>
```

这一段是一个典型的循环结构，结束条件是跳转到 0x401153 说明一直到 0x401153 都是循环体的部分，说明了我们输入的第每个数必须小于等于6

```assembly
  401132: mov    %r12d,%ebx # %ebx = %r12d = 1
  # 下面是内层循环的部分 %ebx 在循环最后部分会自增
  401135: movslq %ebx,%rax  # %rax = %ebx = j
  401138: mov    (%rsp,%rax,4),%eax # %eax = a[j]
  40113b: cmp    %eax,0x0(%rbp) # a[j] != a[0]
  40113e: jne    401145 <phase_6+0x51>
  401140: callq  40143a <explode_bomb>
  401145: add    $0x1,%ebx # ebx 也是循环变量 %ebx < 5
  401148: cmp    $0x5,%ebx
  40114b: jle    401135 <phase_6+0x41> # 如果不相等就跳转到 0x401135 （继续内循环）否则更新 %r13 结束内循环跳转到 0x401114 开始下一轮外循环
  40114d: add    $0x4,%r13 # %r13 指向下一个元素 因为外层循环中有将 %rbp 赋值为 %r13 的操作，所以下一次内循环开始的时候 %rbp 指向 a[i]
  401151: jmp    401114 <phase_6+0x20> 
```

可以看到这里 $\%r12d$ 代表外层循环变量 $i (i \in [0,6)$， $\%ebx$ 代表内层循环变量 $j (j \in [i + 1,6)$，要求 $(a[i] > 6)\ \&\&\ a[i] != a[j]$ 。所以这一段代码的作用为：输入的 6 个数必须都小于等于 6 并互不相等。接下来从 0x401153 开始看，又是一个循环。

%r14 从上面汇编代码可以看到，%r14 一直指向的 a[0] 没有变化

```assembly
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi # %rsi = %rsp + 0x18
  401158:	4c 89 f0             	mov    %r14,%rax # %rax = %r14 = a[i]
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx # $ecx = 7
  401160:	89 ca                	mov    %ecx,%edx # $edx = 7
  401162:	2b 10                	sub    (%rax),%edx  # $edx = 7 - %rax
  401164:	89 10                	mov    %edx,(%rax) # %rax = %edx
  401166:	48 83 c0 04          	add    $0x4,%rax # %rax 指向 a[i + 1]
  40116a:	48 39 f0             	cmp    %rsi,%rax # 判断数组是否越界
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
```

这段代码比较好理解，就是我们输入的 6 个数字被处理了一下 $a[i]=7-a[i]$。因为先前已经确定 $a[i] <= 6$ 了，所以处理后的 $a[i]$ 的范围仍然是 $[1,6]$

```assembly
  40116f:	be 00 00 00 00       	mov    $0x0,%esi # %esi = 0
  401174:	eb 21                	jmp    401197 <phase_6+0xa3> # 跳到 0x401197
  #内层循环
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx # 由后续分析可知，这里取的是链表下一个元素的地址（跳8字节跳了两个int）
  40117a:	83 c0 01             	add    $0x1,%eax # %eax 自增。从下方看，%ecx 是数组中的某个元素 a[i]
  40117d:	39 c8                	cmp    %ecx,%eax # j < a[i]
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94> # 跳出内层循环
  # 内层循环结束
  # 上面这个循环其实比较抽象，看起来我们输入的数据被当成了索引，但其实一想比较合理，毕竟前面确保了我们输入的数必须在 [0, 6] 并且互不相同，符合索引的性质
  # 经过上方的循环，%rdx 应该指向了 nodes[a[i] - 1]
  # a[i] == 1 的情况，跳过了上面的循环，比较合理，因为是取 nodes[0]
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx # %edx = nodes[0]
  # 如果进行过上方内循环，会跳过上方指令，毕竟我们好不容易拿到了 nodes[a[i] - 1]，上方指令又给置成 nodes[0]了
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2) # rdx 放到了 %rsp+0x20+2i，a 本身占 24 字节，这里应该是把这个地址赋值给了新数组，因为地址 8 字节所以是 2i（看到这里我已经对自己的猜想完全没有信心了）
  40118d:	48 83 c6 04          	add    $0x4,%rsi # i++
  401191:	48 83 fe 18          	cmp    $0x18,%rsi # i < 6
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx # %ecx= %rsp + %rsi 此处是取数组第一个元素
  40119a:	83 f9 01             	cmp    $0x1,%ecx # 比较 %ecx 和 1
  40119d:	7e e4                	jle    401183 <phase_6+0x8f> # 小于等于 1 跳转到 0x401183 因为 a[i] >0 所以只有可能为 1
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax # 大于等于 1: %eax = 0x1; %edx = 0x6032d0
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82> #回到被跳过的 0x401176
```

出现的 `0x6032d0` 这个地址让人困惑，如果按字符串输出显然不是一个合法的字符串。因为这一章学过的结构体在这个 lab 中还没有出现，推测这应该是一个结构体。因为有符号表的存在，可以看出这确实是一个自定义结构体，而且从名字看出这应该是一个链表

```assembly
(gdb) x 0x6032d0
0x6032d0 <node1>:       0x000000010000014c

(gdb) x/24w 0x6032d0
0x6032d0 <node1>:       0x0000014c      0x00000001      0x006032e0      0x00000000
0x6032e0 <node2>:       0x000000a8      0x00000002      0x006032f0      0x00000000
0x6032f0 <node3>:       0x0000039c      0x00000003      0x00603300      0x00000000
0x603300 <node4>:       0x000002b3      0x00000004      0x00603310      0x00000000
0x603310 <node5>:       0x000001dd      0x00000005      0x00603320      0x00000000
0x603320 <node6>:       0x000001bb      0x00000006      0x00000000      0x00000000
```

通过观察后面几个字节，基本可以确定链表结构如下

```assembly
struct node {
	int val; // 4字节
	int id; // 4字节
	node *next; // 8字节
}
```

循环部分在注释已经提到了。这样循环下来 %rsp+0x20 这往后又存放了一个数组，我们叫他 $addr$ 吧。$addr$ 的类型是 $(int*)[]$（我想起视频里的数组指针和指针数组的噩梦了，我记不住还是加个括号吧）。$addr[i] = nodes[a[i] - 1]$ 这一部分也就算了个 $addr$ 数组，剩下的看 0x4011ab 开始。

```assembly
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx # %rbx 指向 addr[0]
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax # %rax 指向 addr[1]
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi # %rsi 指向 addr + 6 | 0x50-0x20=0x30=48=8*6
  4011ba:	48 89 d9             	mov    %rbx,%rcx # %rcx 指向 addr[0]
  
  4011bd:	48 8b 10             	mov    (%rax),%rdx # %rdx = addr[1]
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx) # %addr[0].next = addr[1]
  4011c4:	48 83 c0 08          	add    $0x8,%rax # % %rax 指向 addr[2]
  4011c8:	48 39 f0             	cmp    %rsi,%rax # %rsp 和 %rsi 防止越界
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx) # addr[5] = NULL
  4011d9:	00 
```

这里比较好理解，做了一步 $addr[i].next=addr[i+1]$比较好理解，因为这是一个链表，我们的数组是有序的，但是还需要调整链表内部的顺序。综合上面的步骤推测它将我们输入的6个数字作为索引，重新将链表进行了排序 。既然我们已经为链表排好序了，说明接下来是要遍历这个链表了。

设当前节点为 $node$ 实际上接下来 $node=\%rbx=node1=addr[0]$

```assembly
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp # %ebp=0x5
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax # %rax 指向 node.next
  4011e3:	8b 00                	mov    (%rax),%eax # %rax = node.next
  4011e5:	39 03                	cmp    %eax,(%rbx) # node.next.value > node.value 否则炸弹爆炸 cmp 默认取前 4 个字节
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	call   40143a <explode_bomb>
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx # %rbx 指向 node.next
  4011f2:	83 ed 01             	sub    $0x1,%ebp # %ebp从 5 循环到 0
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
```

完工，说明我们输入的6个数字用7减去后的索引值要让数列 $[332, 168, 924, 691, 477,443]$依次递减。

先依次递减后，按索引排好序：$[3, 4, 5, 6, 1, 2]$，再用 7 取依次减去得到我们应输入的6个数字 $[4,3,2,1,6,5]$

**答案**： `4 3 2 1 6 5`
