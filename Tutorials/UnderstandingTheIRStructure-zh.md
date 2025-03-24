# 理解 IR 结构

MLIR 语言参考描述了[High Level Structure](../LangRef.md/#high-level-structure)，
本文档通过示例说明了该结构，并同时介绍了用于操作它的 C++ API。

我们将实现一个遍历任何 MLIR 输入并打印 IR 内部实体（entity）的[pass](../PassManagement.md/#operation-pass)。
一个 pass（或者通常来说几乎任何 IR 片段）始终以一个操作（operation）为根。
大多数情况下，顶层操作是 `ModuleOp`，而 MLIR 的 `PassManager` 实际上仅限于对顶层 `ModuleOp` 进行操作。
因此，一个 pass 以操作（operation）开始，我们的遍历也将如此：

```
  void runOnOperation() override {
    Operation *op = getOperation();
    resetIndent();
    printOperation(op);
  }
```

## 遍历 IR 的嵌套结构

IR 结构是递归嵌套的（recursively nested），一个 `Operation` 可以包含一个或多个嵌套的 `Region`，
每个 `Region` 实际上是 `Block` 的列表，而每个 `Block` 又包含 `Operation` 的列表。
我们的遍历将遵循这一结构，并使用以下三种方法：`printOperation()`、`printRegion()` 和 `printBlock()`。

第一种方法 `printOperation()` 主要用于检查操作（operation）的属性，然后遍历其嵌套的 `Region` 并逐个打印它们：

```c++
  void printOperation(Operation *op) {
    // Print the operation itself and some of its properties
    printIndent() << "visiting op: '" << op->getName() << "' with "
                  << op->getNumOperands() << " operands and "
                  << op->getNumResults() << " results\n";
    // Print the operation attributes
    if (!op->getAttrs().empty()) {
      printIndent() << op->getAttrs().size() << " attributes:\n";
      for (NamedAttribute attr : op->getAttrs())
        printIndent() << " - '" << attr.getName() << "' : '"
                      << attr.getValue() << "'\n";
    }

    // Recurse into each of the regions attached to the operation.
    printIndent() << " " << op->getNumRegions() << " nested regions:\n";
    auto indent = pushIndent();
    for (Region &region : op->getRegions())
      printRegion(region);
  }
```


一个 `Region` 除了包含一个 `Block` 列表外，不包含其他任何内容。

```c++
  void printRegion(Region &region) {
    // A region does not hold anything by itself other than a list of blocks.
    printIndent() << "Region with " << region.getBlocks().size()
                  << " blocks:\n";
    auto indent = pushIndent();
    for (Block &block : region.getBlocks())
      printBlock(block);
  }
```

最后，一个 `Block` 拥有一个参数列表，并且包含一个 `Operation` 列表。

```c++
  void printBlock(Block &block) {
    // Print the block intrinsics properties (basically: argument list)
    printIndent()
        << "Block with " << block.getNumArguments() << " arguments, "
        << block.getNumSuccessors()
        << " successors, and "
        // Note, this `.size()` is traversing a linked-list and is O(n).
        << block.getOperations().size() << " operations\n";

    // A block main role is to hold a list of Operations: let's recurse into
    // printing each operation.
    auto indent = pushIndent();
    for (Operation &op : block.getOperations())
      printOperation(&op);
  }
```

该pass的代码可以在[仓库](https://github.com/llvm/llvm-project/blob/main/mlir/test/lib/IR/TestPrintNesting.cpp)中找到，并且可以通过 `mlir-opt -test-print-nesting` 来进行测试。

### 示例

上一节中介绍的 Pass 可以应用于以下 IR。使用如下命令即可执行：

```bash
mlir-opt -test-print-nesting -allow-unregistered-dialect llvm-project/mlir/test/IR/print-ir-nesting.mlir
```


```mlir
"builtin.module"() ( {
  %0:4 = "dialect.op1"() {"attribute name" = 42 : i32} : () -> (i1, i16, i32, i64)
  "dialect.op2"() ( {
    "dialect.innerop1"(%0#0, %0#1) : (i1, i16) -> ()
  },  {
    "dialect.innerop2"() : () -> ()
    "dialect.innerop3"(%0#0, %0#2, %0#3)[^bb1, ^bb2] : (i1, i32, i64) -> ()
  ^bb1(%1: i32):  // pred: ^bb0
    "dialect.innerop4"() : () -> ()
    "dialect.innerop5"() : () -> ()
  ^bb2(%2: i64):  // pred: ^bb0
    "dialect.innerop6"() : () -> ()
    "dialect.innerop7"() : () -> ()
  }) {"other attribute" = 42 : i64} : () -> ()
}) : () -> ()
```


并将产生如下输出结果：

```
visiting op: 'builtin.module' with 0 operands and 0 results
 1 nested regions:
  Region with 1 blocks:
    Block with 0 arguments, 0 successors, and 2 operations
      visiting op: 'dialect.op1' with 0 operands and 4 results
      1 attributes:
       - 'attribute name' : '42 : i32'
       0 nested regions:
      visiting op: 'dialect.op2' with 0 operands and 0 results
       2 nested regions:
        Region with 1 blocks:
          Block with 0 arguments, 0 successors, and 1 operations
            visiting op: 'dialect.innerop1' with 2 operands and 0 results
             0 nested regions:
        Region with 3 blocks:
          Block with 0 arguments, 2 successors, and 2 operations
            visiting op: 'dialect.innerop2' with 0 operands and 0 results
             0 nested regions:
            visiting op: 'dialect.innerop3' with 3 operands and 0 results
             0 nested regions:
          Block with 1 arguments, 0 successors, and 2 operations
            visiting op: 'dialect.innerop4' with 0 operands and 0 results
             0 nested regions:
            visiting op: 'dialect.innerop5' with 0 operands and 0 results
             0 nested regions:
          Block with 1 arguments, 0 successors, and 2 operations
            visiting op: 'dialect.innerop6' with 0 operands and 0 results
             0 nested regions:
            visiting op: 'dialect.innerop7' with 0 operands and 0 results
             0 nested regions:
       0 nested regions:
```


## 其他 IR 遍历方法

在许多情况下，手动展开 IR 的递归结构可能会比较繁琐，此时您可能会希望使用其他辅助工具来简化操作。



### Filtered iterator(筛选迭代器)：`getOps<OpTy>()`

例如，`Block` 类提供了一个方便的模板方法 `getOps<OpTy>()`，该方法可以返回一个经过筛选的迭代器。以下是一个示例：


```c++
  auto varOps = entryBlock.getOps<spirv::GlobalVariableOp>();
  for (spirv::GlobalVariableOp gvOp : varOps) {
     // process each GlobalVariable Operation in the block.
     ...
  }
```


类似地，`Region` 类也提供了相同的 `getOps` 方法，它会遍历该区域（Region）中的所有块（Block）。



### Walkers（遍历器）

虽然 `getOps<OpTy>()` 方法在遍历某个块（Block）或某个区域（Region）中直接列出的特定类型操作时非常有用，
但在实际应用中，通常更关注以嵌套方式遍历整个 IR 结构。
为此，MLIR 提供了 `walk()` 辅助方法，可用于 `Operation`、`Block` 和 `Region` 对象。

`walk()` 方法接受一个参数：一个回调函数，该函数会被递归地应用于所提供实体下的所有嵌套操作（Operation）。


```c++
  // Recursively traverse all the regions and blocks nested inside the function
  // and apply the callback on every single operation in post-order.
  getFunction().walk([&](mlir::Operation *op) {
    // process Operation `op`.
  });
```

所提供的回调函数可以被特化，用于筛选特定类型的 Operation。
例如，以下代码仅会将回调函数应用于函数体中嵌套的 `LinalgOp` 类型的操作：

```c++
  getFunction().walk([](LinalgOp linalgOp) {
    // process LinalgOp `linalgOp`.
  });
```


最后，回调函数可以通过返回 `WalkResult::interrupt()` 值来选择性地停止遍历。
例如，以下遍历将查找嵌套在函数中的所有 `AllocOp`，并且如果其中任何一个不满足特定条件，则会中断遍历：

```c++
  WalkResult result = getFunction().walk([&](AllocOp allocOp) {
    if (!isValid(allocOp))
      return WalkResult::interrupt();
    return WalkResult::advance();
  });
  if (result.wasInterrupted())
    // One alloc wasn't matching.
    ...
```

## 遍历定义-使用链（def-use chains）

IR 中的另一个重要关系是将 `Value` 与其使用者（users）连接起来的关系。
正如在[语言参考 (language reference)](../LangRef.md/#high-level-structure)中定义的那样，
每个 `Value` 要么是一个 `BlockArgument`，
要么是一个 `Operation` 的结果（一个 `Operation` 可以有多个结果，每个结果都是一个单独的 `Value`）。
`Value` 的使用者是 `Operation`，它们通过参数来引用：每个 `Operation` 参数都引用一个单独的 `Value`。

以下是一个代码示例，展示了如何检查一个 `Operation` 的操作数（operands），并打印一些关于它们的信息：

```c++
  // Print information about the producer of each of the operands.
  for (Value operand : op->getOperands()) {
    if (Operation *producer = operand.getDefiningOp()) {
      llvm::outs() << "  - Operand produced by operation '"
                   << producer->getName() << "'\n";
    } else {
      // If there is no defining op, the Value is necessarily a Block
      // argument.
      auto blockArg = operand.cast<BlockArgument>();
      llvm::outs() << "  - Operand produced by Block argument, number "
                   << blockArg.getArgNumber() << "\n";
    }
  }
```

类似地，以下代码示例遍历一个 `Operation` 产生的结果 `Value`，
并且对于每个结果，都会遍历这些结果的使用者（users），并打印有关它们的信息：

```c++
  // Print information about the user of each of the result.
  llvm::outs() << "Has " << op->getNumResults() << " results:\n";
  for (auto indexedResult : llvm::enumerate(op->getResults())) {
    Value result = indexedResult.value();
    llvm::outs() << "  - Result " << indexedResult.index();
    if (result.use_empty()) {
      llvm::outs() << " has no uses\n";
      continue;
    }
    if (result.hasOneUse()) {
      llvm::outs() << " has a single use: ";
    } else {
      llvm::outs() << " has "
                   << std::distance(result.getUses().begin(),
                                    result.getUses().end())
                   << " uses:\n";
    }
    for (Operation *userOp : result.getUsers()) {
      llvm::outs() << "    - " << userOp->getName() << "\n";
    }
  }
```


该 Pass 的示例代码可以在
[仓库中的此处](https://github.com/llvm/llvm-project/blob/main/mlir/test/lib/IR/TestPrintDefUse.cpp)找到，
并可通过如下命令运行测试：

```bash
mlir-opt -test-print-defuse
```

`Value` 与其使用者之间的链接关系可以通过如下示意图理解：

![Index Map Example](/includes/img/DefUseChains.svg)

此外，`Value` 的所有使用位置（无论是 `OpOperand` 还是 `BlockOperand`）都通过一个双向链表进行连接，
这在将一个 `Value` 替换为另一个新 `Value`（即 “RAUW”，Replace All Uses With）时非常实用：

![Index Map Example](/includes/img/Use-list.svg)