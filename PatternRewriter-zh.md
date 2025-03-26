

# 模式重写：通用 DAG 到 DAG 的重写机制 (Pattern Rewriting : Generic DAG-to-DAG Rewriting)

[目录]

本文档详细介绍了 MLIR 中模式重写（Pattern Rewriting）基础设施的设计与 API，
该基础设施提供了一个通用的 DAG 到 DAG（有向无环图）转换框架。
该框架广泛应用于 MLIR 中，用于标准化（Canonicalization）、转换（Conversion）
以及通用的变换（Transformation）操作。

如果您希望了解 DAG 到 DAG 转换的基本概念以及构建此框架的动机，
建议参考：[通用 DAG 重写机制的原理说明(Generic DAG Rewriter Rationale)](Rationale/RationaleGenericDAGRewriter.md)。



## 简介

模式重写框架在很大程度上可以分解为两个部分：模式定义(Pattern Definition)和模式应用(Pattern Application)。

## 定义模式

模式是通过继承 `RewritePattern` 类来定义的。这个类代表了 MLIR中所有重写模式的基类，并且由以下几个部分组成： 



### 收益(Benefit)

这是应用某个模式后预期能够带来的收益。该收益在模式构建时是静态的，但可以在模式初始化时动态计算。
例如，收益(benefit)可以从特定领域的信息（如目标架构）中派生(derived)。
这种限制使得模式融合(pattern fusion)和将模式编译为高效状态机成为可能。
[Thier, Ertl 和 Krall](https://dl.acm.org/citation.cfm?id=3179501) 的研究表明，
在几乎所有情况下，匹配谓词可以消除对动态计算成本的需求
：您只需为每个可能的成本实例化一次相同的模式，并通过谓词来保护匹配。

### 根操作名称（可选）(Root Operation Name (Optional))

这是模式匹配的根操作名称。如果指定了该名称，
则只有具有该根名称的操作会被传递给 `match` 和 `rewrite` 实现。
如果未指定，则可能会传递任何类型的操作。在可能的情况下，应尽量提供根操作名称，
因为它在应用成本模型(cost model)时能够简化模式的分析。
如果需要匹配任意操作类型(operation type)，则必须使用一个特殊标签来明确意图：`MatchAnyOpTypeTag`。


### `match` and `rewrite` implementation

This is the chunk of code that matches a given root `Operation` and performs a
rewrite of the IR. A `RewritePattern` can specify this implementation either via
separate `match` and `rewrite` methods, or via a combined `matchAndRewrite`
method. When using the combined `matchAndRewrite` method, no IR mutation should
take place before the match is deemed successful. The combined `matchAndRewrite`
is useful when non-trivially recomputable information is required by the
matching and rewriting phase. See below for examples:

### `match` 和 `rewrite` 实现

这是用于匹配给定根 `Operation` 并执行 IR 重写的代码块。
`RewritePattern` 可以通过单独的 `match` 和 `rewrite` 方法来实现，
也可以通过一个组合的 `matchAndRewrite` 方法来实现。
当使用组合的 `matchAndRewrite` 方法时，在匹配成功之前，不应进行任何 IR 的修改。
如果匹配和重写阶段需要非平凡的可重新计算( non-trivially recomputable)的信息，
组合的 `matchAndRewrite` 方法会非常有用。以下是示例：


```c++
class MyPattern : public RewritePattern {
public:
  /// This overload constructs a pattern that only matches operations with the
  /// root name of `MyOp`.
  MyPattern(PatternBenefit benefit, MLIRContext *context)
      : RewritePattern(MyOp::getOperationName(), benefit, context) {}
  /// This overload constructs a pattern that matches any operation type.
  MyPattern(PatternBenefit benefit)
      : RewritePattern(benefit, MatchAnyOpTypeTag()) {}

  /// In this section, the `match` and `rewrite` implementation is specified
  /// using the separate hooks.
  LogicalResult match(Operation *op) const override {
    // The `match` method returns `success()` if the pattern is a match, failure
    // otherwise.
    // ...
  }
  void rewrite(Operation *op, PatternRewriter &rewriter) {
    // The `rewrite` method performs mutations on the IR rooted at `op` using
    // the provided rewriter. All mutations must go through the provided
    // rewriter.
  }

  /// In this section, the `match` and `rewrite` implementation is specified
  /// using a single hook.
  LogicalResult matchAndRewrite(Operation *op, PatternRewriter &rewriter) {
    // The `matchAndRewrite` method performs both the matching and the mutation.
    // Note that the match must reach a successful point before IR mutation may
    // take place.
  }
};
```


#### 限制

在模式的 `match` 部分中，适用以下约束：

* 不允许对 IR 进行任何修改。

在模式的 `rewrite` 部分中，适用以下约束：

* 所有 IR 的修改（包括创建）*必须* 通过给定的 `PatternRewriter` 执行。
  该类提供了执行模式中所有可能修改的钩子(hooks)。
  例如，这意味着不应通过操作的 `erase` 方法来删除操作(operation)。
  要删除操作(operation)，应使用 `PatternRewriter` 的相应钩子(hook)（在本例中为 `eraseOp`）。
* 根操作(root)必须被：原地更新、替换或删除。



### 应用递归(Application Recursion)

递归(Recursion)在模式重写的上下文中是一个重要话题，因为一个模式可能经常适用于其自身的结果。
例如，想象一个从循环操作中剥离单次迭代的模式。如果循环有多个可剥离的迭代，该模式在应用过程中可能会多次适用。
通过查看该模式的实现，递归应用的边界可能是显而易见的，例如，循环中没有可剥离的迭代，
但从模式驱动器(pattern driver)的角度来看，这种递归可能是危险的。
通常情况下，模式的递归应用表明匹配逻辑中存在错误。这类错误通常不会导致崩溃，但会在应用过程中创建无限循环。
鉴于此，模式重写基础设施保守地假设没有模式具有适当的有限递归，并在检测到递归时失败。
已知能够正确处理递归的模式可以在初始化时通过调用 `setHasBoundedRewriteRecursion` 来表明这一点。
这将向模式驱动器发出信号，表明该模式可能会递归应用，并且该模式已准备好安全处理它。

### 调试名称和标签(Debug Names and Labels)

为了辅助调试，模式可以指定：一个调试名称（通过 `setDebugName`），该名称应作为唯一标识特定模式的标识符；
以及一组调试标签（通过 `addDebugLabels`），这些标签对应唯一标识模式组的标识符。
这些信息被各种工具用于辅助模式重写的调试，例如在调试日志中提供模式过滤等功能。以下是一个简单的代码示例：

```c++
class MyPattern : public RewritePattern {
public:
  /// Inherit constructors from RewritePattern.
  using RewritePattern::RewritePattern;

  void initialize() {
    setDebugName("MyPattern");
    addDebugLabels("MyRewritePass");
  }

  // ...
};

void populateMyPatterns(RewritePatternSet &patterns, MLIRContext *ctx) {
  // Debug labels may also be attached to patterns during insertion. This allows
  // for easily attaching common labels to groups of patterns.
  patterns.addWithLabel<MyPattern, ...>("MyRewritePatterns", ctx);
}
```

### 初始化(Initialization)

模式的部分状态需要由模式显式初始化，例如，如果模式能够安全处理递归应用，
则需要设置 `setHasBoundedRewriteRecursion`。
这些模式状态可以在模式的构造函数中初始化，也可以通过工具函数 `initialize` 钩子进行初始化。
使用 `initialize` 钩子可以避免仅为了注入额外的模式状态初始化而重新定义模式构造函数。以下是一个示例：

```c++
class MyPattern : public RewritePattern {
public:
  /// Inherit the constructors from RewritePattern.
  using RewritePattern::RewritePattern;

  /// Initialize the pattern.
  void initialize() {
    /// Signal that this pattern safely handles recursive application.
    setHasBoundedRewriteRecursion();
  }

  // ...
};
```

### 构造(Construction)

构造 `RewritePattern` 应使用静态方法 `RewritePattern::create<T>` 工具方法来完成。
该方法确保模式被正确初始化并准备好插入到 `RewritePatternSet` 中。


## 模式重写器(Pattern Rewriter)

`PatternRewriter` 是一个特殊类，允许模式与模式应用驱动程序进行通信。
如上所述，*所有* IR 的修改（包括创建）都必须通过 `PatternRewriter` 类执行。
这是因为底层模式驱动程序可能具有某些状态，这些状态在修改发生时可能会失效。
以下是一些常用的 `PatternRewriter` API 示例，请参阅
[类文档class documentation](https://github.com/llvm/llvm-project/blob/main/mlir/include/mlir/IR/PatternMatch.h#L235)
以获取最新的可用 API 列表：

*   Erase an Operation : `eraseOp`

This method erases an operation that either has no results, or whose results are
all known to have no uses.

*   Notify why a `match` failed : `notifyMatchFailure`

This method allows for providing a diagnostic message within a `matchAndRewrite`
as to why a pattern failed to match. How this message is displayed back to the
user is determined by the specific pattern driver.

*   Replace an Operation : `replaceOp`/`replaceOpWithNewOp`

This method replaces an operation's results with a set of provided values, and
erases the operation.

*   Update an Operation in-place : `(start|cancel|finalize)OpModification`

This is a collection of methods that provide a transaction-like API for updating
the attributes, location, operands, or successors of an operation in-place
within a pattern. An in-place update transaction is started with
`startOpModification`, and may either be canceled or finalized with
`cancelOpModification` and `finalizeOpModification` respectively. A convenience
wrapper, `modifyOpInPlace`, is provided that wraps a `start` and `finalize`
around a callback.

*  删除操作：`eraseOp`  
该方法删除一个没有结果或其结果已知未被使用的操作(operation)。

*   通知 `match` 失败的原因：`notifyMatchFailure`  
该方法允许在 `matchAndRewrite` 中提供诊断信息，说明模式匹配失败的原因。
该信息如何显示给用户由具体的模式驱动程序决定。

*   替换操作：`replaceOp`/`replaceOpWithNewOp`  
该方法用一组提供的值替换操作的结果，并删除该操作。

*   原地更新操作 ：`(start|cancel|finalize)OpModification`  
这是一组方法，提供类似事务的 API，用于在模式中原地更新操作的属性、位置、操作数或后继操作。
原地更新事务通过 `startOpModification` 开始，
并可以通过 `cancelOpModification` 或 `finalizeOpModification` 分别取消或完成。
还提供了一个便捷的包装函数 `modifyOpInPlace`，它在回调函数周围封装了 `start` 和 `finalize`。

*   OpBuilder API
`PatternRewriter` 继承自 `OpBuilder` 类，因此提供了 `OpBuilder` 中的所有功能。
这包括操作创建，以及许多有用的属性和类型构造方法。


## 模式应用(Pattern Application)

在一组模式被定义后，它们会被收集并提供给特定的驱动程序(driver)进行应用。驱动程序由几个高层次部分组成：

*   输入(Input) `RewritePatternSet`

驱动程序的输入模式以 `RewritePatternSet` 的形式提供。该类为构建模式列表提供了一个简化的 API。

*   Driver-specific `PatternRewriter`

To ensure that the driver state does not become invalidated by IR mutations
within the pattern rewriters, a driver must provide a `PatternRewriter` instance
with the necessary hooks overridden. If a driver does not need to hook into
certain mutations, a default implementation is provided that will perform the
mutation directly.

*  Driver-specific `PatternRewriter` 
为确保驱动程序状态不会因模式重写器中的 IR 修改而失效，驱动程序必须提供一个 `PatternRewriter` 实例，并覆盖必要的钩子。
如果驱动程序不需要干预某些修改，则提供了默认实现，直接执行修改。


*   Pattern Application and Cost Model

Each driver is responsible for defining its own operation visitation order as
well as pattern cost model, but the final application is performed via a
`PatternApplicator` class. This class takes as input the
`RewritePatternSet` and transforms the patterns based upon a provided
cost model. This cost model computes a final benefit for a given pattern, using
whatever driver specific information necessary. After a cost model has been
computed, the driver may begin to match patterns against operations using
`PatternApplicator::matchAndRewrite`.

An example is shown below:

```c++
class MyPattern : public RewritePattern {
public:
  MyPattern(PatternBenefit benefit, MLIRContext *context)
      : RewritePattern(MyOp::getOperationName(), benefit, context) {}
};

/// Populate the pattern list.
void collectMyPatterns(RewritePatternSet &patterns, MLIRContext *ctx) {
  patterns.add<MyPattern>(/*benefit=*/1, ctx);
}

/// Define a custom PatternRewriter for use by the driver.
class MyPatternRewriter : public PatternRewriter {
public:
  MyPatternRewriter(MLIRContext *ctx) : PatternRewriter(ctx) {}

  /// Override the necessary PatternRewriter hooks here.
};

/// Apply the custom driver to `op`.
void applyMyPatternDriver(Operation *op,
                          const FrozenRewritePatternSet &patterns) {
  // Initialize the custom PatternRewriter.
  MyPatternRewriter rewriter(op->getContext());

  // Create the applicator and apply our cost model.
  PatternApplicator applicator(patterns);
  applicator.applyCostModel([](const Pattern &pattern) {
    // Apply a default cost model.
    // Note: This is just for demonstration, if the default cost model is truly
    //       desired `applicator.applyDefaultCostModel()` should be used
    //       instead.
    return pattern.getBenefit();
  });

  // Try to match and apply a pattern.
  LogicalResult result = applicator.matchAndRewrite(op, rewriter);
  if (failed(result)) {
    // ... No patterns were applied.
  }
  // ... A pattern was successfully applied.
}
```

## Common Pattern Drivers

MLIR provides several common pattern drivers that serve a variety of different
use cases.

### Dialect Conversion Driver

This driver provides a framework in which to perform operation conversions
between, and within dialects using a concept of "legality". This framework
allows for transforming illegal operations to those supported by a provided
conversion target, via a set of pattern-based operation rewriting patterns. This
framework also provides support for type conversions. More information on this
driver can be found [here](DialectConversion.md).

### Greedy Pattern Rewrite Driver

This driver processes ops in a worklist-driven fashion and greedily applies the
patterns that locally have the most benefit. The benefit of a pattern is decided
solely by the benefit specified on the pattern, and the relative order of the
pattern within the pattern list (when two patterns have the same local benefit).
Patterns are iteratively applied to operations until a fixed point is reached or
until the configurable maximum number of iterations exhausted, at which point
the driver finishes.

This driver comes in two fashions:

*   `applyPatternsAndFoldGreedily` ("region-based driver") applies patterns to
    all ops in a given region or a given container op (but not the container op
    itself). I.e., the worklist is initialized with all containing ops.
*   `applyOpPatternsAndFold` ("op-based driver") applies patterns to the
    provided list of operations. I.e., the worklist is initialized with the
    specified list of ops.

The driver is configurable via `GreedyRewriteConfig`. The region-based driver
supports two modes for populating the initial worklist:

*   Top-down traversal: Traverse the container op/region top down and in
    pre-order. This is generally more efficient in compile time.
*   Bottom-up traversal: This is the default setting. It builds the initial
    worklist with a postorder traversal and then reverses the worklist. This may
    match larger patterns with ambiguous pattern sets.

By default, ops that were modified in-place and newly created are added back to
the worklist. Ops that are outside of the configurable "scope" of the driver are
not added to the worklist. Furthermore, "strict mode" can exclude certain ops
from being added to the worklist throughout the rewrite process:

*   `GreedyRewriteStrictness::AnyOp`: No ops are excluded (apart from the ones
    that are out of scope).
*   `GreedyRewriteStrictness::ExistingAndNewOps`: Only pre-existing ops (with
    which the worklist was initialized) and newly created ops are added to the
    worklist.
*   `GreedyRewriteStrictness::ExistingOps`: Only pre-existing ops (with which
    the worklist was initialized) are added to the worklist.

Note: This driver listens for IR changes via the callbacks provided by
`RewriterBase`. It is important that patterns announce all IR changes to the
rewriter and do not bypass the rewriter API by modifying ops directly.

Note: This driver is the one used by the [canonicalization](Canonicalization.md)
[pass](Passes.md/#-canonicalize) in MLIR.

### Debugging

To debug the execution of the greedy pattern rewrite driver,
`-debug-only=greedy-rewriter` may be used. This command line flag activates
LLVM's debug logging infrastructure solely for the greedy pattern rewriter. The
output is formatted as a tree structure, mirroring the structure of the pattern
application process. This output contains all of the actions performed by the
rewriter, how operations get processed and patterns are applied, and why they
fail.

Example output is shown below:

```
//===-------------------------------------------===//
Processing operation : 'cf.cond_br'(0x60f000001120) {
  "cf.cond_br"(%arg0)[^bb2, ^bb2] {operandSegmentSizes = array<i32: 1, 0, 0>} : (i1) -> ()

  * Pattern SimplifyConstCondBranchPred : 'cf.cond_br -> ()' {
  } -> failure : pattern failed to match

  * Pattern SimplifyCondBranchIdenticalSuccessors : 'cf.cond_br -> ()' {
    ** Insert  : 'cf.br'(0x60b000003690)
    ** Replace : 'cf.cond_br'(0x60f000001120)
  } -> success : pattern applied successfully
} -> success : pattern matched
//===-------------------------------------------===//
```

This output is describing the processing of a `cf.cond_br` operation. We first
try to apply the `SimplifyConstCondBranchPred`, which fails. From there, another
pattern (`SimplifyCondBranchIdenticalSuccessors`) is applied that matches the
`cf.cond_br` and replaces it with a `cf.br`.

## Debugging

### Pattern Filtering

To simplify test case definition and reduction, the `FrozenRewritePatternSet`
class provides built-in support for filtering which patterns should be provided
to the pattern driver for application. Filtering behavior is specified by
providing a `disabledPatterns` and `enabledPatterns` list when constructing the
`FrozenRewritePatternSet`. The `disabledPatterns` list should contain a set of
debug names or labels for patterns that are disabled during pattern application,
i.e. which patterns should be filtered out. The `enabledPatterns` list should
contain a set of debug names or labels for patterns that are enabled during
pattern application, patterns that do not satisfy this constraint are filtered
out. Note that patterns specified by the `disabledPatterns` list will be
filtered out even if they match criteria in the `enabledPatterns` list. An
example is shown below:

```c++
void MyPass::initialize(MLIRContext *context) {
  // No patterns are explicitly disabled.
  SmallVector<std::string> disabledPatterns;
  // Enable only patterns with a debug name or label of `MyRewritePatterns`.
  SmallVector<std::string> enabledPatterns(1, "MyRewritePatterns");

  RewritePatternSet rewritePatterns(context);
  // ...
  frozenPatterns = FrozenRewritePatternSet(rewritePatterns, disabledPatterns,
                                           enabledPatterns);
}
```

### Common Pass Utilities

Passes that utilize rewrite patterns should aim to provide a common set of
options and toggles to simplify the debugging experience when switching between
different passes/projects/etc. To aid in this endeavor, MLIR provides a common
set of utilities that can be easily included when defining a custom pass. These
are defined in `mlir/RewritePassUtil.td`; an example usage is shown below:

```tablegen
def MyRewritePass : Pass<"..."> {
  let summary = "...";
  let constructor = "createMyRewritePass()";

  // Inherit the common pattern rewrite options from `RewritePassUtils`.
  let options = RewritePassUtils.options;
}
```

#### Rewrite Pass Options

This section documents common pass options that are useful for controlling the
behavior of rewrite pattern application.

##### Pattern Filtering

Two common pattern filtering options are exposed, `disable-patterns` and
`enable-patterns`, matching the behavior of the `disabledPatterns` and
`enabledPatterns` lists described in the [Pattern Filtering](#pattern-filtering)
section above. A snippet of the tablegen definition of these options is shown
below:

```tablegen
ListOption<"disabledPatterns", "disable-patterns", "std::string",
           "Labels of patterns that should be filtered out during application">,
ListOption<"enabledPatterns", "enable-patterns", "std::string",
           "Labels of patterns that should be used during application, all "
           "other patterns are filtered out">,
```

These options may be used to provide filtering behavior when constructing any
`FrozenRewritePatternSet`s within the pass:

```c++
void MyRewritePass::initialize(MLIRContext *context) {
  RewritePatternSet rewritePatterns(context);
  // ...

  // When constructing the `FrozenRewritePatternSet`, we provide the filter
  // list options.
  frozenPatterns = FrozenRewritePatternSet(rewritePatterns, disabledPatterns,
                                           enabledPatterns);
}
```
