# Go\_汇编plan9学习

## Go Plan9 Learn

Go语言的编译器和汇编器都带了一个`-S`参数，可以查看生成的最终目标代码。通过对比目标代码和原始的Go语言或Go汇编语言代码的差异可以加深对底层实现的理解。

在学习go源代码和原理过程中，需要结合汇编进行学习，所以必要少不了要熟知基础汇编知识。go也提供了工具可以将源代码中底层汇编指令输出，通过`` `go tool compile ``命令用于调用Go语言提供的底层命令工具，其中`-S`参数表示输出汇编格式。

```go
package main

func main() {
    _ = add(3,5)
}

func add(a,b int) int {
    return a+b
}
```

`go tool compile -S demo.go`命令执行查看源代码的汇编指令。

```go
cotox@cotoxdeMacBook-Pro main % go tool compile -S demo.go
"".main STEXT nosplit size=1 args=0x0 locals=0x0
.....
// add函数底层汇编
"".add STEXT nosplit size=19 args=0x18 locals=0x0
        0x0000 00000 (demo.go:9)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $0-24
        0x0000 00000 (demo.go:9)        PCDATA  $0, $-2
        0x0000 00000 (demo.go:9)        PCDATA  $1, $-2
        0x0000 00000 (demo.go:9)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (demo.go:9)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (demo.go:9)        FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (demo.go:10)       PCDATA  $0, $0
        0x0000 00000 (demo.go:10)       PCDATA  $1, $0
        0x0000 00000 (demo.go:10)       MOVQ    "".b+16(SP), AX
        0x0005 00005 (demo.go:10)       MOVQ    "".a+8(SP), CX
        0x000a 00010 (demo.go:10)       ADDQ    CX, AX
        0x000d 00013 (demo.go:10)       MOVQ    AX, "".~r2+24(SP)
        0x0012 00018 (demo.go:10)       RET
        0x0000 48 8b 44 24 10 48 8b 4c 24 08 48 01 c8 48 89 44  H.D$.H.L$.H..H.D
        0x0010 24 18 c3                                         $..
....
```

其中，会根据源代码中`func add`函数的生命定义，底层汇编也会生成方法:

```go
// 第一步声明定义函数
0x0000 00000 (demo.go:9)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $0-24
```

这一行表示定义`add`这个函数，最后的数字`$0-24`，其中`0`表示函数栈帧大小为0；`24`表示参数及返回值的大小：参数是2个int型变量，返回值是1个int型变量，共24字节。

在定义完函数之后，就是函数逻辑处理了，可以从上面看到变量`a,b`的处理：

```go
        0x0000 00000 (demo.go:10)       MOVQ    "".b+16(SP), AX
        0x0005 00005 (demo.go:10)       MOVQ    "".a+8(SP), CX
        0x000a 00010 (demo.go:10)       ADDQ    CX, AX
        0x000d 00013 (demo.go:10)       MOVQ    AX, "".~r2+24(SP)
```

代码片段中的第1行，将第2个参数`b`搬到`AX`寄存器；第2行将1个参数`a`搬到寄存器`CX`；第3行将`a`和`b`相加，相加的结果搬到`AX`；最后一行，将结果搬到返回参数的地址。

上面add函数的栈帧大小为0，其实更一般的调用者与被调用者的栈帧示意图如下：![image](https://s1.ax1x.com/2020/09/07/wuHuGt.md.png)

最后，执行`RET`指令。这一步把被调用函数`add`栈帧清零,接着，弹出栈顶的`返回地址`，把它赋给指令寄存器`rip`，而`返回地址`就是`main`函数里调用`add`函数的下一行。

## plan9 assembly学习

### 基础操作

*   栈调整 plan9 中没有 push 和 pop，栈的调整是通过对硬件 SP 寄存器进行运算来实现的

    ```go
    SUBQ $0x18, SP // 对 SP 做减法，为函数分配函数栈帧
    ...               // 省略无用代码
    ADDQ $0x18, SP // 对 SP 做加法，清除函数栈帧
    ```
*   数据搬运 常数在 plan9 汇编用 $num 表示，可以为负数，默认情况下为十进制。可以用 $0x123 的形式来表示十六进制数。 `B`: 一个字节，`W`:两个字节，`D`：4个字节，`Q`: 8个字节

    ```go
    MOVB $1, DI      // 1 byte
    MOVW $0x10, BX   // 2 bytes
    MOVD $1, DX      // 4 bytes
    MOVQ $-10, AX     // 8 bytes
    ```

    可以看到，搬运的长度是由 MOV 的后缀决定的。plan9 的汇编的操作数的方向是和 intel 汇编相反的，与 AT\&T 类似。

    ```go
    MOVQ $0x10, AX ===== mov rax, 0x10
           |    |------------|      |
           |------------------------|
    ```
*   常见计算指令

    ```go
    ADDQ  AX, BX   // BX += AX
    SUBQ  AX, BX   // BX -= AX
    IMULQ AX, BX   // BX *= AX
    ```

    类似数据搬运指令，同样可以通过修改指令的后缀来对应不同长度的操作数。例如 ADDQ/ADDW/ADDL/ADDB。
*   跳转

    ```go
    // 无条件跳转
    JMP addr   // 跳转到地址，地址可为代码中的地址，不过实际上手写不会出现这种东西
    JMP label  // 跳转到标签，可以跳转到同一函数内的标签位置
    JMP 2(PC)  // 以当前指令为基础，向前/后跳转 x 行
    JMP -2(PC) // 同上

    // 有条件跳转
    JNZ target // 如果 zero flag 被 set 过，则跳转
    ```

### 寄存器

在汇编运算中，也是需要有容器来存放数据的，就是使用寄存器来保存数据（地址数据，值数据，都是二进制数据，但是可以代表不同的概念）

*   通用寄存器

    ```go
    (lldb) reg read
    General Purpose Registers:
           rax = 0x0000000000000005
           rbx = 0x000000c420088000
           rcx = 0x0000000000000000
           rdx = 0x0000000000000000
           rdi = 0x000000c420088008
           rsi = 0x0000000000000000
           rbp = 0x000000c420047f78
           rsp = 0x000000c420047ed8
            r8 = 0x0000000000000004
            r9 = 0x0000000000000000
           r10 = 0x000000c420020001
           r11 = 0x0000000000000202
           r12 = 0x0000000000000000
           r13 = 0x00000000000000f1
           r14 = 0x0000000000000011
           r15 = 0x0000000000000001
           rip = 0x000000000108ef85  int`main.main + 213 at int.go:19
        rflags = 0x0000000000000212
            cs = 0x000000000000002b
            fs = 0x0000000000000000
            gs = 0x0000000000000000
    ```

    在 plan9 汇编里都是可以使用的，应用代码层面会用到的通用寄存器主要是:`rax, rbx, rcx, rdx, rdi, rsi, r8~r15` 这 14 个寄存器，虽然 rbp 和 rsp 也可以用，不过 bp 和 sp 会被用来管理栈顶和栈底，最好不要拿来进行运算。

    plan9 中使用寄存器不需要带 r 或 e 的前缀，例如 rax，只要写 AX 即可:

    ```go
    MOVQ $101, AX = mov rax, 101
    ```
*   伪寄存器

    Go 的汇编还引入了 4 个伪寄存器，援引官方文档的描述：

    > * `FP`: Frame pointer: arguments and locals.
    > * `PC`: Program counter: jumps and branches.
    > * `SB`: Static base pointer: global symbols.
    > * `SP`: Stack pointer: top of stack.

    我们这里对容易混淆的几点简单进行说明：

    1. 伪 SP 和硬件 SP 不是一回事，在手写代码时，伪 SP 和硬件 SP 的区分方法是看该 SP 前是否有 symbol。如果有 symbol，那么即为伪寄存器，如果没有，那么说明是硬件 SP 寄存器。
    2. SP 和 FP 的相对位置是会变的，所以不应该尝试用伪 SP 寄存器去找那些用 FP + offset 来引用的值，例如函数的入参和返回值。
    3. 官方文档中说的伪 SP 指向 stack 的 top，是有问题的。其指向的局部变量位置实际上是整个栈的栈底(除 caller BP 之外)，所以说 bottom 更合适一些。
    4. 在 go tool objdump/go tool compile -S 输出的代码中，是没有伪 SP 和 FP 寄存器的，我们上面说的区分伪 SP 和硬件 SP 寄存器的方法，对于上述两个命令的输出结果是没法使用的。在编译和反汇编的结果中，只有真实的 SP 寄存器。
    5. FP 和 Go 的官方源代码里的 framepointer 不是一回事，源代码里的 framepointer 指的是 caller BP 寄存器的值，在这里和 caller 的伪 SP 是值是相等的
*   变量声明

    在汇编里所谓的变量，一般是存储在 .rodata 或者 .data 段中的只读值。对应到应用层的话，就是已初始化过的全局的 const、var、static 变量/常量。

    使用 DATA 结合 GLOBL 来定义一个变量。DATA 的用法为:

    ```go
    DATA    symbol+offset(SB)/width, value
    ```

    大多数参数都是字面意思，不过这个 offset 需要稍微注意。其含义是该值相对于符号 symbol 的偏移，而不是相对于全局某个地址的偏移。
*   函数声明 我们来看看一个典型的 plan9 的汇编函数的定义：

    ```go
    // func add(a, b int) int
    //   => 该声明定义在同一个 package 下的任意 .go 文件中
    //   => 只有函数头，没有实现
    TEXT pkgname·add(SB), NOSPLIT, $0-8
        MOVQ a+0(FP), AX
        MOVQ a+8(FP), BX
        ADDQ AX, BX
        MOVQ BX, ret+16(FP)
        RET
    ```

    为什么要叫 TEXT ？如果对程序数据在文件中和内存中的分段稍有了解的同学应该知道，我们的代码在二进制文件中，是存储在 .text 段中的，这里也就是一种约定俗成的起名方式。实际上在 plan9 中 TEXT 是一个指令，用来定义一个函数。除了 TEXT 之外还有前面变量声明说到的 DATA/GLOBL。

    ```go
                                  参数及返回值大小,入参数+返回参数总字节
                                      | 
     TEXT pkgname·add(SB),NOSPLIT,$32-32
           |        |               |
          包名     函数名         栈帧大小(局部变量+可能需要的额外调用函数的参数空间的总大小，但不包括调用其它函数时的 ret address 的大小)
    ```
*   栈结构 下面是一个典型的函数的栈结构图:

    ```go
                                               -----------------                                           
                           current func arg0                                           
                           ----------------- <----------- FP(pseudo FP)                
                            caller ret addr                                            
                           +---------------+                                           
                           | caller BP(*)  |                                           
                           ----------------- <----------- SP(pseudo SP，实际上是当前栈帧的 BP 位置)
                           |   Local Var0  |                                           
                           -----------------                                           
                           |   Local Var1  |                                           
                           -----------------                                           
                           |   Local Var2  |                                           
                           -----------------                -                          
                           |   ........    |                                           
                           -----------------                                           
                           |   Local VarN  |                                           
                           -----------------                                           
                           |               |                                           
                           |               |                                           
                           |  temporarily  |                                           
                           |  unused space |                                           
                           |               |                                           
                           |               |                                           
                           -----------------                                           
                           |  call retn    |                                           
                           -----------------                                           
                           |  call ret(n-1)|                                           
                           -----------------                                           
                           |  ..........   |                                           
                           -----------------                                           
                           |  call ret1    |                                           
                           -----------------                                           
                           |  call argn    |                                           
                           -----------------                                           
                           |   .....       |                                           
                           -----------------                                           
                           |  call arg3    |                                           
                           -----------------                                           
                           |  call arg2    |                                           
                           |---------------|                                           
                           |  call arg1    |                                           
                           -----------------   <------------  hardware SP 位置           
                           | return addr   |                                           
                           +---------------+
    ```

    图上的 caller BP，指的是 caller 的 BP 寄存器值，有些人把 caller BP 叫作 caller 的 frame pointer，实际上这个习惯是从 x86 架构沿袭来的。Go 的 asm 文档中把伪寄存器 FP 也称为 frame pointer，但是这两个 frame pointer 根本不是一回事。

    如果编译器在最终的汇编结果中没有插入 caller BP(源代码中所称的 frame pointer)的情况下，伪 SP 和伪 FP 之间只有 8 个字节的 caller 的 return address，而插入了 BP 的话，就会多出额外的 8 字节。也就说伪 SP 和伪 FP 的相对位置是不固定的，有可能是间隔 8 个字节，也有可能间隔 16 个字节。并且判断依据会根据平台和 Go 的版本有所不同。

    图上可以看到，FP 伪寄存器指向函数的传入参数的开始位置，因为栈是朝低地址方向增长，为了通过寄存器引用参数时方便，所以参数的摆放方向和栈的增长方向是相反的，即：

    ```go
                                  FP
    high ----------------------> low
    argN, ... arg3, arg2, arg1, arg0
    ```

    > 假设所有参数均为 8 字节，这样我们就可以用 symname+0(FP) 访问第一个 参数，symname+8(FP) 访问第二个参数，以此类推。用伪 SP 来引用局部变量，原理上来讲差不多，不过因为伪 SP 指向的是局部变量的底部，所以 symname-8(SP) 表示的是第一个局部变量，symname-16(SP)表示第二个，以此类推。当然，这里假设局部变量都占用 8 个字节。

    因为官方文档本身较模糊，我们来一个函数调用的全景图，来看一下这些真假 SP/FP/BP 到底是个什么关系:

    ```go
                                           caller                                                                                 
                                     +------------------+                                                                         
                                     |                  |                                                                         
           +---------------------->  --------------------                                                                         
           |                         |                  |                                                                         
           |                         | caller parent BP |                                                                         
           |           BP(pseudo SP) --------------------                                                                         
           |                         |                  |                                                                         
           |                         |   Local Var0     |                                                                         
           |                         --------------------                                                                         
           |                         |                  |                                                                         
           |                         |   .......        |                                                                         
           |                         --------------------                                                                         
           |                         |                  |                                                                         
           |                         |   Local VarN     |                                                                         
                                     --------------------                                                                         
     caller stack frame              |                  |                                                                         
                                     |   callee arg2    |                                                                         
           |                         |------------------|                                                                         
           |                         |                  |                                                                         
           |                         |   callee arg1    |                                                                         
           |                         |------------------|                                                                         
           |                         |                  |                                                                         
           |                         |   callee arg0    |                                                                         
           |                         ----------------------------------------------+   FP(virtual register)                       
           |                         |                  |                          |                                              
           |                         |   return addr    |  parent return address   |                                              
           +---------------------->  +------------------+---------------------------    <-------------------------------+         
                                                        |  caller BP               |                                    |         
                                                        |  (caller frame pointer)  |                                    |         
                                         BP(pseudo SP)  ----------------------------                                    |         
                                                        |                          |                                    |         
                                                        |     Local Var0           |                                    |         
                                                        ----------------------------                                    |         
                                                        |                          |                                              
                                                        |     Local Var1           |                                              
                                                        ----------------------------                            callee stack frame
                                                        |                          |                                              
                                                        |       .....              |                                              
                                                        ----------------------------                                    |         
                                                        |                          |                                    |         
                                                        |     Local VarN           |                                    |         
                                      SP(Real Register) ----------------------------                                    |         
                                                        |                          |                                    |         
                                                        |                          |                                    |         
                                                        |                          |                                    |         
                                                        |                          |                                    |         
                                                        |                          |                                    |         
                                                        +--------------------------+    <-------------------------------+         

                                                                  callee
    ```

## argsize 和 framesize 计算规则

argSize在函数中声明：

```go
 TEXT pkgname·add(SB),NOSPLIT,$16-32
```

前面已经说过 $16-32 表示 $framesize-argsize。Go 在函数调用时，参数和返回值都需要由 caller 在其栈帧上备好空间。callee 在声明时仍然需要知道这个 argsize。argsize 的计算方法是，参数大小求和+返回值大小求和，例如入参是 3 个 int64 类型，返回值是 1 个 int64 类型，那么这里的 argsize = sizeof(int64) \* 4。

不过真实世界永远没有我们假设的这么美好，函数参数往往混合了多种类型，还需要考虑内存对齐问题。

如果不确定自己的函数签名需要多大的 argsize，可以通过简单实现一个相同签名的空函数，然后 `go tool objdump` 来逆向查找应该分配多少空间。

*   framesize

    函数的 framesize 就稍微复杂一些了，手写代码的 framesize 不需要考虑由编译器插入的 caller BP，要考虑：

    1. 局部变量，及其每个变量的 size。
    2. 在函数中是否有对其它函数调用时，如果有的话，调用时需要将 callee 的参数、返回值考虑在内。虽然 return address(rip)的值也是存储在 caller 的 stack frame 上的，但是这个过程是由 CALL 指令和 RET 指令完成 PC 寄存器的保存和恢复的，在手写汇编时，同样也是不需要考虑这个 PC 寄存器在栈上所需占用的 8 个字节的。
    3. 原则上来说，调用函数时只要不把局部变量覆盖掉就可以了。稍微多分配几个字节的 framesize 也不会死。
    4. 在确保逻辑没有问题的前提下，你愿意覆盖局部变量也没有问题。只要保证进入和退出汇编函数时的 caller 和 callee 能正确拿到返回值就可以。

## Demo

可以通过在源代码中声明函数，不进行实现，而是通过汇编plan9方式进行实现函数也是可以运行的：

```go
package main

import "fmt"

func add(a,b int) int  // 汇编函数声明
//func sub(a,b int) int
//func mul(a,b int) int

func main() {
    fmt.Println(add(10,11))
    //fmt.Printf(sub(99,15))
    //fmt.Printf(mul(11,12))
}



// 汇编plan9  math.s文件
#include "textflag.h"

// func add(a,b int) int
// plan9汇编使用TEXT声明函数
TEXT ·add(SB), NOSPLIT, $0-24
    MOVQ a+0(FP),AX  // 参数a
    MOVQ b+8(FP),BX
    ADDQ BX,AX
    MOVQ AX,ret+16(FP)
    RET
```

将上述源代码与汇编文件同放在一个目录下，执行`go build`就能看到生成的二进制执行文件。通过`./main`运行即可。

**伪寄存器 SP 、伪寄存器 FP 和硬件寄存器 SP**

spspfp.s:

```go
#include "textflag.h"

// func output(int) (int, int, int)
TEXT ·output(SB), $8-48
    MOVQ 24(SP), DX // 不带 symbol，这里的 SP 是硬件寄存器 SP
    MOVQ DX, ret3+24(FP) // 第三个返回值
    MOVQ perhapsArg1+16(SP), BX // 当前函数栈大小 > 0，所以 FP 在 SP 的上方 16 字节处
    MOVQ BX, ret2+16(FP) // 第二个返回值
    MOVQ arg1+0(FP), AX
    MOVQ AX, ret1+8(FP)  // 第一个返回值
    RET
```

spspfp.go:

```go
package main

import (
    "fmt"
)

func output(int) (int, int, int) // 汇编函数声明

func main() {
    a, b, c := output(987654321)
    fmt.Println(a, b, c)
}

// 执行输出
987654321 987654321 987654321
```

和代码结合思考，可以知道我们当前的栈结构是这样的:

```go
------
ret2 (8 bytes)
------
ret1 (8 bytes)
------
ret0 (8 bytes)
------
arg0 (8 bytes)
------ FP
ret addr (8 bytes)
------
caller BP (8 bytes)
------ pseudo SP
frame content (8 bytes)
------ hardware SP
```

本小节例子的 framesize 是大于 0 的，读者可以尝试修改 framesize 为 0，然后调整代码中引用伪 SP 和硬件 SP 时的 offset，来研究 framesize 为 0 时，伪 FP，伪 SP 和硬件 SP 三者之间的相对位置。

本小节的例子是为了告诉大家，伪 SP 和伪 FP 的相对位置是会变化的，手写时不应该用伪 SP 和 >0 的 offset 来引用数据，否则结果可能会出乎你的预料.

## 寻址模式

汇编语言的一个很重要的概念就是它的寻址模式，Plan 9 汇编也不例外，它支持如下寻址模式：

> 伪寄存器不是真正的寄存器，而是由工具链维护的虚拟寄存器，例如帧指针。
>
> FP, Frame Pointer：帧指针，参数和本地 PC, Program Counter: 程序计数器，跳转和分支 SB, Static Base: 静态基指针, 全局符号 SP, Stack Pointer: 当前栈帧开始的地方
>
> 所有用户定义的符号都作为偏移量写入伪寄存器 FP 和 SB。

```go
R0              数据寄存器
A0              地址寄存器
F0              浮点寄存器
CAAR, CACR, 等  特殊名字
$con            常量 
$fcon           浮点数常量
name+o(SB)      外部符号
name<>+o(SB)    局部符号
name+o(SP)      自动符号
name+o(FP)      实际参数
$name+o(SB)     外部地址
$name<>+o(SB)   局部地址
(A0)+           间接后增量
-(A0)           间接前增量
o(A0)           
o()(R0.s)
```

**汇编调用非汇编函数**

output.s:

```go
#include "textflag.h"

// func output(a,b int) int
TEXT ·output(SB), NOSPLIT, $24-24
    MOVQ a+0(FP), DX // arg a
    MOVQ DX, 0(SP) // arg x
    MOVQ b+8(FP), CX // arg b
    MOVQ CX, 8(SP) // arg y
    CALL ·add(SB) // 在调用 add 之前，已经把参数都通过物理寄存器 SP 搬到了函数的栈顶
    MOVQ 16(SP), AX // add 函数会把返回值放在这个位置
    MOVQ AX, ret+16(FP) // return result
    RET
```

output.go:

```go
package main

import "fmt"

func add(x, y int) int {
    return x + y
}

func output(a, b int) int

func main() {
    s := output(10, 13)
    fmt.Println(s)
}
```

拷贝博客资源：[plan9 assembly 完全解析](https://github.com/cch123/golang-notes/blob/master/assembly.md)
