
# Toy语言教程 (Toy Tutorial)

本教程将引导您在MLIR之上实现一个基础的Toy语言。本教程的目标是介绍MLIR的核心概念，
特别是如何通过[dialects](../../LangRef.md/#dialects)轻松支持语言特定的结构和转换，
同时提供一条简单的路径将其下降(lower)为LLVM或其他代码生成基础设施。
本教程基于[LLVM Kaleidoscope Tutorial](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html)
的模型。

另一个很好的入门资料是2020年LLVM开发者大会的在线
[recording](https://www.youtube.com/watch?v=Y4SvqTtOIDk)
（[slides](https://llvm.org/devmtg/2020-09/slides/MLIR_Tutorial.pdf)）。

本教程假设您已经克隆并构建了MLIR；如果尚未完成，请参阅[MLIR入门指南](../../../getting_started/)。

本教程分为以下章节：

[MLIR入门指南(Getting started with MLIR)](../../../getting_started/).

教程分为以下几个章节:

-   [Chapter #1](Ch-1.md): Introduction to the Toy language and the definition
    of its AST.
-   [Chapter #2](Ch-2.md): Traversing the AST to emit a dialect in MLIR,
    introducing base MLIR concepts. Here we show how to start attaching
    semantics to our custom operations in MLIR.
-   [Chapter #3](Ch-3.md): High-level language-specific optimization using
    pattern rewriting system.
-   [Chapter #4](Ch-4.md): Writing generic dialect-independent transformations
    with Interfaces. Here we will show how to plug dialect specific information
    into generic transformations like shape inference and inlining.
-   [Chapter #5](Ch-5.md): Partially lowering to lower-level dialects. We'll
    convert some of our high level language specific semantics towards a generic
    affine oriented dialect for optimization.
-   [Chapter #6](Ch-6.md): Lowering to LLVM and code generation. Here we'll
    target LLVM IR for code generation, and detail more of the lowering
    framework.
-   [Chapter #7](Ch-7.md): Extending Toy: Adding support for a composite type.
    We'll demonstrate how to add a custom type to MLIR, and how it fits in the
    existing pipeline.

The [first chapter](Ch-1.md) will introduce the Toy language and AST.


- [第一章](Ch-1.md): Toy语言及其抽象语法树(AST)的定义介绍
- [第二章](Ch-2.md): 通过遍历AST生成MLIR方言，介绍MLIR基础概念。本节将展示如何为MLIR中的自定义操作(operations)附加语义
- [第三章](Ch-3.md): 使用模式重写系统(pattern rewriting system)实现高级语言特定优化
- [第四章](Ch-4.md): 通过接口(Interfaces)编写与方言(dialect)无关的通用转换。演示如何将方言特定信息融入形状推断、内联等通用转换流程
- [第五章](Ch-5.md): 部分降级(lower)至低级方言。将高层语言特定语义转换为面向通用仿射(affine)计算的方言以进行优化
- [第六章](Ch-6.md): 降级至LLVM及代码生成。以LLVM IR为目标输出，并详细说明降级(lowering)框架的实现细节
- [第七章](Ch-7.md): Toy语言扩展：添加复合类型支持。演示如何在MLIR中添加自定义类型，并集成到现有处理流程中

[开篇章节](Ch-1.md)将介绍Toy语言及其抽象语法树(AST)结构。

