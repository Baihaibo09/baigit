# 代码优化
## clang代码优化
使用clang的Ox来编译优化代码
```
clang -S -O1 -mllvm --x86-asm-syntax=intel test.cpp
clang -S -mllvm --x86-asm-syntax=intel test.cpp
```
```
代码生成选项
-O0, -O1, -O2, -O3, -Ofast, -Os, -Oz, -Og, -O, -O4
指定要使用的优化级别：

-O0表示“无优化”：此级别编译速度最快，生成的代码可调试性最高。
-O1介于-O0和之间-O2。
-O2中等优化级别，可实现大多数优化。
-O3像-O2，除了它启用需要更长时间执行或可能生成更大代码的优化（试图使程序运行得更快）。
-Ofast启用所有优化-O3以及其他可能违反严格遵守语言标准的积极优化。
-Os就像-O2额外的优化以减少代码大小一样。
-Oz喜欢-Os（因此-O2），但进一步减少了代码大小。
-Og喜欢-O1。在未来的版本中，此选项可能会禁用不同的优化以提高可调试性。
-O相当于-O1。
-O4及更高,目前相当于-O3
```
参考链接:优化选项<https://gcc.gnu.org/onlinedocs/gcc-6.3.0/gcc/Optimize-Options.html>
Clang/LLVM 有一个开关，用于描述在编译运行期间使用了哪些优化。Opt.txt 包含在多次传递中尝试的所有优化的详细信息。
![1650287304525.png](./img/1650287304525.png)
![20220425162205](https://github.com/Baihaibo09/image/blob/main/20220425162205.png)
![2copy.png](https://github.com/Baihaibo09/image/blob/main/20220425162309.png)
![new](https://github.com/Baihaibo09/image/blob/main/20220425162951.png)
![1.png](https://github.com/Baihaibo09/image/blob/main/1650287304525.png)
```
clang -O3 -foptimization-record-file=Opt.txt test.cpp
```
## OPT代码优化：消除死代码、循环优化
1. 使用OPT对IR进行优化，消除死代码。
```
./opt --mem2reg --adce --bdce /data/test/1.bc -o /data/test/2.bc 
```
![1650282127577.png](./img/1650282127577.png)

通过对比看出，对y变量的操作已经没有了。
2. 接下来使用OPT继续优化，增加“循环展开”策略优化。

        ./opt --mem2reg --adce --bdce --loop-unroll /data/test/2.bc -o /data/test/3.bc

![1650282290011.png](./img/1650282290011.png)

可以看到，for循环已经没有了，变成了连续3次的调用。
## 实验
### 原始代码
如果您不应用任何优化通道 (-O0)，Clang 会将 C++ isEven函数编译为直接汇编（以 LLVM IR 的形式）。
```
; Function Attrs: noinline nounwind ssp uwtable
define zeroext i1 @_Z6isEveni(i32 %0) #0 {
    %2 = alloca i32, align 4 ; number
    %3 = alloca i32, align 4 ; numberCompare
    %4 = alloca i8, align 1 ; even
    store i32 %0, i32* %2, align 4 ; store function argument in number
    store i32 0, i32* %3, align 4 ; store 0 in numberCompare
    store i8 1, i8* %4, align 1 ; store 1 (true) in even
    br label %5
5:                                                ; preds = %9, %1
    %6 = load i32, i32* %2, align 4
    %7 = load i32, i32* %3, align 4
    %8 = icmp ne i32 %6, %7
    br i1 %8, label %9, label %16
9:                                                ; preds = %5
    %10 = load i8, i8* %4, align 1
    %11 = trunc i8 %10 to i1
    %12 = xor i1 %11, true
    %13 = zext i1 %12 to i8
    store i8 %13, i8* %4, align 1
    %14 = load i32, i32* %3, align 4
    %15 = add nsw i32 %14, 1
    store i32 %15, i32* %3, align 4
    br label %5
16:                                               ; preds = %5
    %17 = load i8, i8* %4, align 1
    %18 = trunc i8 %17 to i1
    ret i1 %18
}
```
LLVM IR 是一种低级（接近机器）中间表示形式，具有单一静态分配形式（SSA）的特殊性。每条指令总是产生一个新值，而不是重新分配前一个值。
静态单赋值形式（static single assignment form),通常简写为SSA form或是SSA。
实验代码CFG图如下：
![1650288526752.png](./img/1650288526752.png)
* 第一个块（numeroted %1）初始化函数内部各种变量的内存（两个 4 字节整数用于number和numberCompare，一个字节用于even）。
* 第二个块（numeroted %5）是循环检查，以确定我们是应该进入循环还是退出循环。
* 第三个块 (numeroted %9) 是循环体，它递增numberCompare ( %15) 并切换布尔偶数( %12)。
* 第四个块是函数的返回，它将结果转换为布尔值（%18）并将其返回给调用者。

### 优化代码
此时，代码只是我们原始 C++ 代码的汇编 SSA 版本。让我们对其运行一些 LLVM 优化传递，看看它是如何演变的。

#### 1. 注册内存
我们运行的第一遍称为Memory to Register ( mem2reg)：它的目标是将变量从内存（在 RAM 中）移动到抽象寄存器（直接在 CPU 内部），以使其更快（内存延迟约为 100ns）。
![1650288718617.png](./img/1650288718617.png)
我们看到所有与内存相关的指令（alloca, load, store）都已被优化器删除，现在所有操作（add, xor）都直接在 CPU 寄存器上完成。

4 个块仍然存在，但略有不同（它们看起来更接近原始 C++ 代码）：
* 初始化块现在是空的。
* 循环条件块已更改，它包含 2 个称为phi节点的指令。这些是特殊节点，根据先前执行的块（称为前驱块）取值 2 。例如，该行%.01 = phi i32 [ 0, %1 ], [ %8, %4 ]表示变量%.01%（在我们的 C++ 代码中表示numberCompare ）如果我们来自 作为函数开头的基本块，则应该取值0 ，或者如果我们来自基本块，则取值这是循环的主体。%1%8%4
### 2. 指令结合
将几条指令组合成一条更简单/更快的指令。例如，同一变量上的 2 次连续加法可以减少为一条；或者乘以 8 可以更改为左移 3，等等……
![1650288946033.png](./img/1650288946033.png)
在我们的代码中，此通道进行了一些更改：
* even不再是一个字节（i8），而是直接存储为一个位（i1），因此删除了几个转换指令。
* 循环条件已从 a not equal切换为equal，并且 2 个块已适应保持语义完整。
### 3.循环和归纳变量
我们在前面的代码上运行 pass （ -simplifycfg），以删除任何无用的分支。在最后一次instruction combine pass后，我们获得了完全优化的O(1)复杂性代码。
```
; Function Attrs: noinline nounwind ssp uwtable
define zeroext i1 @_Z6isEveni(i32 %0) #0 {
  %2 = trunc i32 %0 to i1
  %3 = add i1 %2, true
  ret i1 %3
}
```
Recursive version
```
bool isEvenRec(int number)
{
  if (number == 0) return true;

  return !isEvenRec(number-1);
}
```
参考链接：
1. <https://llvm.org/docs/Passes.html#indvars-canonicalize-induction-variables>
2. <https://blog.matthieud.me/2020/exploring-clang-llvm-optimization-on-programming-horror/>