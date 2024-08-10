# llvm backend lowering学习笔记

[toc]

## 1.lowering简介

### 1.1 什么是`lowering`过程
将输入的<b>IR</b>转换成<b>SelectionDAG</b>的过程被称作lowering, 在lowering之前我们通过<b>IR</b>表示一段程序, 在lowering之后我们使用<b>SelectionDAG</b>来描述同一段程序

### 1.2 为什么需要`lowering`
<b>IR</b>中每个节点只能有一个输出(表示一个操作符/操作数的对象Node本身只有一个输出值), 而<b>SelectionDAG</b>中的节点通常有多个输出, 并且输出不一定是一个参与运算的值. <b>SelectionDAG</b>会包含非数据的依赖关系, 我们可以通过UseList获取显示引用这个操作符/操作数的使用者, 但无法获取非数据依赖的引用关系, 也就是<b>SelectionDAG</b>体现了节点之间的约束关系, 而<b>IR</b>只能表示节点的数据流图.

### 1.3 分析一个简单的例子
1. 首先一个test.c的C程序
``` {.line-numbers}
int add(int a, int b) {
  int c = a + b;
  return c;
}
```

执行:
<b>_`clang -c ./test.c -emit-llvm -S -o test.ll`_</b>

生成[test.ll](./resource/simple_test/test.ll)
``` {.line-numbers}
; ModuleID = './test.c'
source_filename = "./test.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @add(i32 noundef %0, i32 noundef %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %6 = load i32, i32* %3, align 4
  %7 = load i32, i32* %4, align 4
  %8 = add nsw i32 %6, %7
  store i32 %8, i32* %5, align 4
  %9 = load i32, i32* %5, align 4
  ret i32 %9
}

attributes #0 = { noinline nounwind optnone uwtable "frame-pointer"="all" "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }

!llvm.module.flags = !{!0, !1, !2, !3, !4}
!llvm.ident = !{!5}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 7, !"PIC Level", i32 2}
!2 = !{i32 7, !"PIE Level", i32 2}
!3 = !{i32 7, !"uwtable", i32 1}
!4 = !{i32 7, !"frame-pointer", i32 2}
!5 = !{!"Ubuntu clang version 14.0.0-1ubuntu1.1"}
```

执行:
<b>_`llc -march=cpu0 -filetype=asm test.ll -debug-only=isel -o test.s 2>test.isel`_</b>
得到指令选择阶段的输出<b>[test.isel](./resource/simple_test/test.isel)</b>
```

```
