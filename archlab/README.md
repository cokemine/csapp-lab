# ArchLab

### Preparation

1. Y86-64 指令集

   ![](https://i.imgur.com/qVXanbM.png)

2. Y86-64 指令集的功能码

   ![](https://i.imgur.com/zQhwF7q.png)

3. Y86-64 寄存器标识符

   ![](https://i.imgur.com/1tIkwCl.png)

4. 由于这个 lab 代码时间久远，可能会存在一些依赖问题（如 tcl/tk 版本问题）以及 gcc 版本问题也可能导致需要修改部分编译参数或是代码才可以成功编译，建议遇到错误去搜索解决办法，因为在这里解释也有一定时效性，不确定过几年对于新版本系统/编译器还能否运行。~~随便改改能编译就行了，谁知道为什么能跑起来~~

### Part A

#### 1. sum.ys: Iteratively sum linked list elements

第一题，要求我们用 Y86-64 汇编代码实现循环对链表进行求和，我们需要实现的是 sum_list  函数，对应的 C 语言代码已给出。

参考 CSAPP 一书中图 4-7 对于计算数组和的编写，也可以尝试把 C 程序编译后再反编译为汇编代码，但 gcc 大概率会夹杂大量私货优化和安全保护，在编译时可能需要添加不少参数才能使汇编代码转变成我们方便改写的形式。

```assembly
# CSAPP 图 4-7 的初始化操作
    .pos 0
    irmovq stack,%rsp # 初始化栈指针
    call main # 调用 main 函数
    halt

# Sample linked list 测试数据，复制于 archlab.pdf
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
	irmovq ele1,%rdi # 将链表头指针移到第一个参数
    call sum_list # 调用 sum_list 函数
    ret

sum_list:
    irmovq $0,%rax # 设置返回值为 0，%rax 存放返回值即 ans
    jmp test # 跳转到 test 判断 while 循环条件
loop:	
	mrmovq 0(%rdi),%rsi # 取链表当前节点指针到 %rsi
    addq %rsi,%rax # ans += node->value
    mrmovq 8(%rdi),%rsi # node = node->next
    rrmovq %rsi,%rdi
test:   
	andq %rdi,%rdi # 判断当前所指节点是否为 NULL，即是否遍历完链表
    jne loop
    ret

# 栈地址从 0x100 开始向下增长
    .pos 0x100
stack:
```

编译并运行。观察结果 `%rax` 结果是否为测试链表的节点和。

```bash
./yas sum.ys
./yis sum.yo
```

#### 2. rsum.ys: Recursively sum linked list elements  

第二题，要求我们用 Y86-64 汇编代码实现递归对链表进行求和，我们需要实现的是 rsum_list 函数，对应的 C 语言代码已给出。

```assembly
# CSAPP 图 4-7 的初始化操作
    .pos 0
    irmovq stack,%rsp # 初始化栈指针
    call main # 调用 main 函数
    halt

# Sample linked list 测试数据，复制于 archlab.pdf
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
	irmovq ele1,%rdi # 将链表头指针移到第一个参数
    call rsum_list # 调用 rsum_list 函数
    ret

rsum_list:
    pushq %rcx         # 调用者保存的寄存器
    andq %rdi,%rdi # 如果当前节点为 NULL 则返回
    je  end
    mrmovq (%rdi),%rcx # rest = node->value
    mrmovq 8(%rdi),%rdi # node = node->next 传参
    call rsum_list
    addq %rcx,%rax # return val+rest;
end:
    popq %rcx
    ret

# 栈地址从 0x100 开始向下增长
    .pos 0x100
stack:
```

编译并运行。观察结果 `%rax` 结果是否为测试链表的节点和。

```bash
./yas rsum.ys
./yis rsum.yo
```

#### copy.ys: Copy a source block to a destination block  

实现数组复制，并返回原数组的异或和。

```assembly
# CSAPP 图 4-7 的初始化操作
    .pos 0
    irmovq stack,%rsp # 初始化栈指针
    call main # 调用 main 函数
    halt

# 测试数据，复制于 archlab.pdf
.align 8
# Source block
src:
    .quad 0x00a
    .quad 0x0b0
    .quad 0xc00
# Destination block
dest:
    .quad 0x111
    .quad 0x222
    .quad 0x333

main:	
    irmovq src,%rdi
    irmovq dest,%rsi
    irmovq $3,%rdx
    call copy_block # copy_block 接收三个参数：原数组、被复制数组、数组长度
    ret
    
copy_block:
    irmovq $1,%r13 # 存放一些常数，因为算术运算不能使用立即数
    irmovq $8,%r14
    irmovq $0,%rax # result = 0
    jmp test
loop:
    mrmovq 0(%rdi),%r12 # val = src
    addq %r14,%rdi # src++
    rmmovq %r12,(%rsi) # dest = val
    addq %r14,%rsi # dest++
    xorq %r12,%rax # result ^= val
    subq %r13,%rdx # len--
test:
    andq %rdx,%rdx # 判断 len 是否为 0 退出循环
    jg loop
    ret

# 栈地址从 0x100 开始向下增长
    .pos 0x100
stack:
```

编译并运行。观察结果 `%rax` 结果是否为`src`数组的异或和，并且 `dest` 数组是否被 `src` 数组覆盖。

```bash
./yas rsum.ys
./yis rsum.yo
```
### Part B

根据书中课后作业 4.51、4.52 实现 `iaddq` 指令。

![](https://i.imgur.com/nsJXa3u.png)

这一部分主要是书中 4.3 的内容，每一个阶段一个一个修改就可以了。

1. 取值阶段

   *instr_valid*：这个字节对应于一个合法的指令吗？显然我们要在这里插入我们需要添加的指令 `IIADDQ`

   ```c
   bool instr_valid = icode in 
   	{ INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
   	       IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };
   ```

   need_regids：这个指令包含一个寄存器指示符字节吗？当然包括。

   ```c
   bool need_regids =
   	icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
   		     IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };
   ```

   need_valC：这个指令包括一个常数字吗？当然包括。

   ```c
   bool need_valC =
   	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };
   ```

2. 译码和写回阶段。

   只需要修改寄存器 B 的部分，因为这个指令我们不需要寄存器 A 取而代之的是立即数。

   srcB：操作数 B（valB）在哪里取？这里表示一个是取自寄存器 B 还是寄存器 `%rsp` 或是不取

   ```c
   word srcB = [
   	icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ  } : rB;
   	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
   	1 : RNONE;  # Don't need register
   ];
   ```

   dstE：计算结果写回到哪里，肯定我们需要写回到寄存器 B

   ```c
   word dstE = [
   	icode in { IRRMOVQ } && Cnd : rB;
   	icode in { IIRMOVQ, IOPQ, IIADDQ } : rB;
   	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
   	1 : RNONE;  # Don't write any register
   ];
   ```

3. 执行阶段

   aluA：ALU 的 A 输入端，应该输入的是立即数

   ```c
   word aluA = [
   	icode in { IRRMOVQ, IOPQ } : valA;
   	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
   	icode in { ICALL, IPUSHQ } : -8;
   	icode in { IRET, IPOPQ } : 8;
   	# Other instructions don't need ALU
   ];
   ```

   aluB：ALU 的 B 输入端，应该输入的是 valB

   ```c
   word aluB = [
   	icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
   		      IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
   	icode in { IRRMOVQ, IIRMOVQ } : 0;
   	# Other instructions don't need ALU
   ];
   ```

   set_cc：是否需要设置条件码

   ```c
   bool set_cc = icode in { IOPQ, IIADDQ };
   ```

访存阶段和更新 PC 阶段都不需要额外修改内容，因为这个指令不涉及访存。

运行测试

```shell
./ssim -t ../y86-code/asumi.yo
(cd ../y86-code; make testssim)
(cd ../ptest; make SIM=../seq/ssim) 
(cd ../ptest; make SIM=../seq/ssim TFLAGS=-i)
```

## Part C

修改 `ncopy.ys` 和 `pipe-full.hcl` 两个文件，使得 `ncopy.ys` 运行得尽可能快。

Todo
