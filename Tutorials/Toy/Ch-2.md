# 第二章：生成基础MLIR

[TOC]

既然我们已经熟悉了编程语言及其抽象语法树(AST)，现在让我们看看MLIR如何帮助编译Toy语言。

## MLIR简介：多级中间表示

传统编译器（如LLVM，参见[Kaleidoscope教程](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html)）
通常提供一组固定的预定义类型和（通常是低层级/RISC风格的）指令集。
语言前端需要自行处理所有语言特定的类型检查、分析和转换工作，然后才能生成LLVM IR。
例如，Clang不仅会利用其AST进行静态分析，还会执行诸如通过AST克隆和重写来实现C++模板实例化等转换操作。
对于比C/C++更高级的语言，从AST到LLVM IR的转换过程往往更加复杂。

这种情况导致不同前端不得不重复实现大量相似的基础设施来支持这些分析和转换需求。
MLIR通过可扩展的设计解决了这个问题——它几乎没有预定义指令（MLIR术语中称为*操作operations*）或类型。


## Interfacing with MLIR

[Language Reference](../../LangRef.md)

MLIR is designed to be a completely extensible infrastructure; there is no
closed set of attributes (think: constant metadata), operations, or types. MLIR
supports this extensibility with the concept of
[Dialects](../../LangRef.md/#dialects). Dialects provide a grouping mechanism for
abstraction under a unique `namespace`.

In MLIR, [`Operations`](../../LangRef.md/#operations) are the core unit of
abstraction and computation, similar in many ways to LLVM instructions.
Operations can have application-specific semantics and can be used to represent
all of the core IR structures in LLVM: instructions, globals (like functions),
modules, etc.

Here is the MLIR assembly for the Toy `transpose` operations:


## 与MLIR的交互接口

[语言参考 Language Reference](../../LangRef.md)

MLIR被设计为完全可扩展的基础架构，不存在封闭(closed)的属性(attributes)集（即常量元数据）、操作(operations)或类型(types)。
MLIR通过[方言(Dialects)](../../LangRef.md/#dialects)的概念实现这种可扩展性。
Dialect在唯一的`namespace`下为抽象提供分组机制。

在MLIR中，[`操作(Operations)`](../../LangRef.md/#operations)是抽象和计算的核心单元，
在许多方面类似于LLVM指令。Operations可具有应用特定(application-specific)的语义，
并能表示LLVM中所有的核心IR结构：指令(instructions)、全局对象(globals)（如函数）、模块(modules)等。

以下是Toy语言`transpose`操作对应的MLIR汇编(assembly)代码：

```mlir
%t_tensor = "toy.transpose"(%tensor) {inplace = true} : (tensor<2x3xf64>) -> tensor<3x2xf64> loc("example/file/path":12:1)
```

让我们解析这个MLIR operation 的组成结构：

-   `%t_tensor`

    *   operation定义的结果的名称（包括[前缀符号以避免冲突](../../LangRef.md/#identifiers-and-keywords)）。
        一个operation可以定义零个或多个结果（在 Toy 的上下文中，我们将operation限制为单结果操作），
        这些结果是 SSA 值。该名称在解析过程中使用，但并不是持久的（例如，它不会在 SSA 值的内存表示中被跟踪）。

-   `"toy.transpose"`

    *   operation的名称。预计它是一个唯一的字符串，且在“`.`”之前会加上方言的命名空间(namespace)。
        可以将其理解为 `toy` dialect中的 `transpose` operation。

-   `(%tensor)`

    *   一组零个或多个输入操作数(operands)（或参数(arguments)），这些operands是由其他operation定义的 SSA 值，
        或是引用了块参数(block arguments)。

-   `{ inplace = true }`

    *  一个包含零个或多个属性(attributes)的字典(dictionary)，这些属性(attribute)是特殊的操作数(operands)，
      始终是常量。在这里，我们定义了一个名为 `inplace` 的布尔属性，其常量值为 `true`。

-   `(tensor<2x3xf64>) -> tensor<3x2xf64>`

    *  这指的是operation的类型(type)，以函数形式表示，参数的类型写在括号中，返回值的类型则写在后面。

-   `loc("example/file/path":12:1)`

    *   这是该operation来源于源代码中的位置。

这里展示的是operation的一般形式。如上所述，MLIR 中的operation集是可扩展的。
operation是使用一小组概念来建模的，这些概念使得operation能够以通用的方式进行推理和操作。这些概念包括：

-   operation的名称。
-   一组 SSA 操作数值 operand values。
-   一组[属性(attributes)](../../LangRef.md/#attributes)。
-   一组结果值的[类型(types)](../../LangRef.md/#type-system)。
-   用于调试目的的[源位置(source location)](../../Diagnostics.md/#source-locations)。
-   一组后继[块(blocks)](../../LangRef.md/#blocks)（主要用于分支）。
-   一组[区域(regions)](../../LangRef.md/#regions)（用于像函数这样的结构化操作）。


在 MLIR 中，每个operation都有一个强制关联的源位置(source location)。
与 LLVM 不同，在 LLVM 中，调试信息的位置是元数据，可以被丢弃，而在 MLIR 中，位置是核心要求，
API 依赖并操作它。因此，丢弃位置是一个显式的选择，不会意外发生。

为了提供一个说明：如果某个转换将一个operation替换为另一个operation，
那么新的operation仍然必须附带位置。这使得追踪该operation来源成为可能。

值得注意的是，`mlir-opt` 工具——一个用于测试编译器通道的工具——默认情况下不在输出中包括位置。
`-mlir-print-debuginfo` 标志指定要包含位置。（运行 `mlir-opt --help` 获取更多选项。）


### Opaque API

MLIR is designed to allow all IR elements, such as attributes, operations, and
types, to be customized. At the same time, IR elements can always be reduced to
the above fundamental concepts. This allows MLIR to parse, represent, and
[round-trip](../../../getting_started/Glossary.md/#round-trip) IR for *any*
operation. For example, we could place our Toy operation from above into an
`.mlir` file and round-trip through *mlir-opt* without registering any `toy`
related dialect:

### 不透明 API(Opaque API)

MLIR 设计允许所有 IR 元素，如属性、操作和类型(attributes, operations, and
types)，进行自定义。
与此同时，IR 元素始终可以还原为上述基本概念。这使得 MLIR 能够解析、表示并为
 *任何(any)* operation进行[往返(round-trip)](../../../getting_started/Glossary.md/#round-trip) IR。
例如，我们可以将上面的 Toy operation放入一个 `.mlir` 文件中，
并通过 *mlir-opt* 进行round-trip，而无需注册任何与 `toy` 相关的方言：


```mlir
func.func @toy_func(%tensor: tensor<2x3xf64>) -> tensor<3x2xf64> {
  %t_tensor = "toy.transpose"(%tensor) { inplace = true } : (tensor<2x3xf64>) -> tensor<3x2xf64>
  return %t_tensor : tensor<3x2xf64>
}
```

In the cases of unregistered attributes, operations, and types, MLIR will
enforce some structural constraints (e.g. dominance, etc.), but otherwise they
are completely opaque. For instance, MLIR has little information about whether
an unregistered operation can operate on particular data types, how many
operands it can take, or how many results it produces. This flexibility can be
useful for bootstrapping purposes, but it is generally advised against in mature
systems. Unregistered operations must be treated conservatively by
transformations and analyses, and they are much harder to construct and
manipulate.

This handling can be observed by crafting what should be an invalid IR for Toy
and seeing it round-trip without tripping the verifier:

对于未注册的属性、操作和类型，MLIR 会强制执行一些结构约束（例如，支配关系等），
但除此之外，它们是完全不透明的。例如，MLIR 对未注册的操作是否可以操作特定数据类型、
它可以接受多少操作数或产生多少结果几乎没有信息。这种灵活性对于引导工作流程是有用的，
但在成熟的系统中通常不建议使用。未注册的操作必须由转换和分析以保守的方式处理，并且它们更难构建和操作。

可以通过构造一个本应是无效的 Toy IR 来观察这种处理方式，并查看它如何在不触发验证器的情况下进行round-trip：

```mlir
func.func @main() {
  %0 = "toy.print"() : () -> tensor<2x3xf64>
}
```



这里有多个问题：`toy.print` 操作不是终结符；它应该接受一个操作数；并且不应该返回任何值。
在下一节中，我们将把我们的方言和操作注册到 MLIR 中，接入验证器(verifier)，并添加更友好的 API 来操作我们的operations。

## 定义一个 Toy 方言(Toy Dialect)

为了有效地与 MLIR 进行接口交互，我们将定义一个新的 Toy 方言。这个方言将建模 Toy 语言的结构，
并提供一种便捷的途径用于high-level分析和转换。

```c++
/// 这是 Toy 方言的定义。一个方言继承自 mlir::Dialect，
/// 并注册自定义的属性、操作和类型。它还可以重写virtual方法以更改一些通用行为，相关内容将在教程的后续章节中演示。

class ToyDialect : public mlir::Dialect {
public:
  explicit ToyDialect(mlir::MLIRContext *ctx);


  /// 提供一个访问方言命名空间的实用程序访问器(accessor)。
  static llvm::StringRef getDialectNamespace() { return "toy"; }

  /// 从 ToyDialect 构造函数中调用的initializer，用于在 Toy 方言中注册属性、操作、类型等内容。
  void initialize();
};
```


这是方言的 C++ 定义，但 MLIR 也支持通过 [tablegen](https://llvm.org/docs/TableGen/ProgRef.html) 声明性地定义方言。
使用声明性规范要更简洁，因为它消除了在定义新方言时需要大量样板代码的需求。
它还使得方言文档的生成变得更加容易，文档可以直接与方言一起描述。在这种声明性格式中，Toy 方言将被指定为：


```tablegen
// 在 ODS 框架中提供 'toy' dialect的定义，以便我们能够定义我们的operations。
def Toy_Dialect : Dialect {
  // 我们方言的命名空间，这与我们在 `ToyDialect::getDialectNamespace` 中提供的字符串一一对应。
  let name = "toy";

  // 我们方言的简短总结。
  let summary = "用于分析和优化 Toy 语言的高级方言";

  // 我们方言的更长描述。
  let description = [{
    Toy 语言是一种基于张量的语言，允许你定义函数、执行一些数学计算并打印结果。这个方言提供了该语言的表示形式，有利于分析和优化。
  }];

  // 方言类定义所在的 C++ 命名空间。
  let cppNamespace = "toy";
}
```



为了查看这会生成什么，我们可以运行 `mlir-tblgen` 命令，并使用 `gen-dialect-decls` 操作，如下所示：

```shell
${build_root}/bin/mlir-tblgen -gen-dialect-decls ${mlir_src_root}/examples/toy/Ch2/include/toy/Ops.td -I ${mlir_src_root}/include/
```

在方言定义完成后，它现在可以加载到 MLIRContext 中：

```c++
  context.loadDialect<ToyDialect>();
```

默认情况下，`MLIRContext` 只加载 [Builtin Dialect](../../Dialects/Builtin.md)，
该方言提供了一些核心的 IR 组件，这意味着其他方言，如我们的 `Toy` 方言，必须显式加载。



## 定义 Toy 操作 Toy Operations

现在我们已经有了一个 `Toy` dialect，我们可以开始定义operations。这将允许提供语义信息，
供系统的其他部分使用。例如，让我们通过创建一个 `toy.constant` 操作来进行演示。这个操作将表示 Toy 语言中的常量值。

```mlir
 %4 = "toy.constant"() {value = dense<1.0> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
```


This operation takes zero operands, a
[dense elements](../../Dialects/Builtin.md/#denseintorfpelementsattr) attribute named
`value` to represent the constant value, and returns a single result of
[RankedTensorType](../../Dialects/Builtin.md/#rankedtensortype). An operation class
inherits from the [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)
`mlir::Op` class which also takes some optional [*traits*](../../Traits) to
customize its behavior. `Traits` are a mechanism with which we can inject
additional behavior into an Operation, such as additional accessors,
verification, and more. Let's look below at a possible definition for the
constant operation that we have described above:

该操作接受零个操作数，一个名为 `value` 的 [dense elements](../../Dialects/Builtin.md/#denseintorfpelementsattr) 属性，
用于表示常量值，并返回一个 [RankedTensorType](../../Dialects/Builtin.md/#rankedtensortype) 类型的单个结果。
operation类继承自 [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) 的 `mlir::Op` 类，
该类还可以接受一些可选的 [*traits*](../../Traits)，以自定义其行为。
`Traits` 是一种机制，我们可以通过它将额外的行为注入到operation中，例如额外的访问器、验证等。下面我们来看一下可能的常量操作定义：


```c++
class ConstantOp : public mlir::Op<
                     /// `mlir::Op` 是一个 CRTP 类，意味着我们将派生类作为模板参数提供。
                     ConstantOp,
                     /// ConstantOp 接受零个输入操作数。
                     mlir::OpTrait::ZeroOperands,
                     /// ConstantOp 返回一个单一结果。
                     mlir::OpTrait::OneResult,
                     /// 我们还提供一个实用的 `getType` 访问器，用于返回单一结果的 TensorType。
                     mlir::OpTraits::OneTypedResult<TensorType>::Impl> {

 public:
  /// 继承自基础 Op 类的构造函数。
  using Op::Op;

  /// 提供此操作的唯一名称。MLIR 将使用此名称注册操作，并在整个系统中唯一标识它。
  /// 这里提供的名称必须以父方言命名空间为前缀，后跟 `.`。
  static llvm::StringRef getOperationName() { return "toy.constant"; }

  /// 通过从属性中获取常量值来返回常量。
  mlir::DenseElementsAttr getValue();

  /// Operations可以提供额外的验证，超出所附 traits 的内容。
  /// 在这里我们将确保常量操作的特定不变式得到遵守，例如，结果类型必须是 TensorType，
  /// 并且与常量 `value` 的类型匹配。
  LogicalResult verifyInvariants();

  /// 提供一个接口，通过一组输入值来构建此操作。
  /// 这个接口被 `builder` 类使用，允许轻松生成此操作的实例：
  ///   mlir::OpBuilder::create<ConstantOp>(...)
  /// 此方法填充 MLIR 用于创建操作的给定 `state`。该 state 是操作可能包含的所有离散元素(discrete elements)的集合。
  /// 使用给定的返回类型和 `value` 属性构建常量。
  static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
                    mlir::Type result, mlir::DenseElementsAttr value);
  /// 通过给定的 'value' 重用类型来构建常量。
  static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
                    mlir::DenseElementsAttr value);
  /// 通过广播给定的 'value' 来构建常量。
  static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
                    double value);
};
```


我们可以在 `ToyDialect` 初始化器(initializer)中注册这个操作：

```c++
void ToyDialect::initialize() {
  addOperations<ConstantOp>();
}
```


### Op 与 Operation：使用 MLIR Operations

现在我们已经定义了一个operation，我们将希望访问并转换它。
在 MLIR 中，与operation相关的主要有两个类：`Operation` 和 `Op`。
`Operation` 类用于通用地建模所有operation。它是“不可知的(opaque)”，意味着它不描述特定operation的属性或操作类型。
相反，`Operation` 类提供了一个通用的 API 来访问operation实例。
另一方面，每种特定类型的operation由一个 `Op` 派生类表示。
例如，`ConstantOp` 表示一个具有零个输入和一个输出的操作，该输出始终设置为相同的值。
`Op` 派生类充当 `Operation*` 的智能指针包装器(smart pointer wrapper)，
提供特定于operation的访问方法和类型安全的操作属性。
这意味着，当我们定义我们的 Toy 操作时，我们只是为构建和与 `Operation` 类交互定义了一个简洁、语义上有用的接口。
这就是为什么我们的 `ConstantOp` 不定义任何类字段；
所有的操作数据都存储在引用的 `Operation` 中。
该设计的副作用是，我们总是“按值”传递 `Op` 派生类，
而不是按引用或指针传递（*按值传递(passing byvalue)* 是 MLIR 中常见的惯用法，也适用于属性、类型等）。
给定一个通用的 `Operation*` 实例，我们总是可以使用 LLVM 的类型转换基础设施来获取特定的 `Op` 实例：


```c++
void processConstantOp(mlir::Operation *operation) {
  ConstantOp op = llvm::dyn_cast<ConstantOp>(operation);

  // 这个操作不是 `ConstantOp` 的实例。
  if (!op)
    return;

  // 获取智能指针包装的内部操作实例。
  mlir::Operation *internalOperation = op.getOperation();
  assert(internalOperation == operation &&
         "这些操作实例是相同的");
}
```



### Using the Operation Definition Specification (ODS) Framework

In addition to specializing the `mlir::Op` C++ template, MLIR also supports
defining operations in a declarative manner. This is achieved via the
[Operation Definition Specification](../../DefiningDialects/Operations.md) framework. Facts
regarding an operation are specified concisely into a TableGen record, which
will be expanded into an equivalent `mlir::Op` C++ template specialization at
compile time. Using the ODS framework is the desired way for defining operations
in MLIR given the simplicity, conciseness, and general stability in the face of
C++ API changes.

Lets see how to define the ODS equivalent of our ConstantOp:

Operations in ODS are defined by inheriting from the `Op` class. To simplify our
operation definitions, we will define a base class for operations in the Toy
dialect.


### 使用操作定义规范（ODS）框架

除了专门化 `mlir::Op` C++ 模板外，MLIR 还支持以声明性方式定义操作。
这是通过 [Operation Definition Specification](../../DefiningDialects/Operations.md) 框架实现的。
Operation的事实被简洁地指定为 TableGen 记录，这些记录将在编译时扩展为等效的 `mlir::Op` C++ 模板实例化。
鉴于其简单性、简洁性以及在面对 C++ API 变化时的普遍稳定性，使用 ODS 框架是定义 MLIR 操作的推荐方式。

让我们看看如何定义与我们 `ConstantOp` 等效的 ODS：

ODS 中的Operation通过继承自 `Op` 类来定义。为了简化我们的Operation定义，我们将在 Toy 方言中为操作定义一个基类。


```tablegen
// Toy 方言操作的基类。该操作继承自 OpBase.td 中的基础 `Op` 类，并提供：
//   * 操作的父方言。
//   * 操作的助记符，或者没有方言前缀的名称。
//   * 操作的特征列表。
class Toy_Op<string mnemonic, list<Trait> traits = []> :
    Op<Toy_Dialect, mnemonic, traits>;

```

定义好所有初步的组件后，我们可以开始定义常量operation。

我们通过继承上面定义的基类 'Toy_Op' 来定义一个 Toy operation。
在这里，我们为operation提供助记符(mnemonic)和一组特征列表。
此处的 [mnemonic](../../DefiningDialects/Operations.md/#operation-name) 与 
`ConstantOp::getOperationName` 中提供的名称相匹配，
但没有方言前缀 `toy.`。在我们的 C++ 定义中缺少的是 `ZeroOperands` 和 `OneResult` 特征；
这些将基于我们稍后定义的 `arguments` 和 `results` 字段自动推导。


```tablegen
def ConstantOp : Toy_Op<"constant"> {
}
```

在这一点上，您可能想了解由 TableGen 生成的 C++ 代码是什么样的。
只需运行 `mlir-tblgen` 命令，并使用 `gen-op-decls` 或 `gen-op-defs` 操作，如下所示：

```shell
${build_root}/bin/mlir-tblgen -gen-op-defs ${mlir_src_root}/examples/toy/Ch2/include/toy/Ops.td -I ${mlir_src_root}/include/
```

根据所选操作，这将打印 `ConstantOp` 类的声明或其实现。
在开始使用 TableGen 时，将此输出与手工编写的实现进行比较是非常有帮助的。

#### Defining Arguments and Results

With the shell of the operation defined, we can now provide the
[inputs](../../DefiningDialects/Operations.md/#operation-arguments) and
[outputs](../../DefiningDialects/Operations.md/#operation-results) to our operation. The
inputs, or arguments, to an operation may be attributes or types for SSA operand
values. The results correspond to a set of types for the values produced by the
operation:


#### 定义输入和输出

在定义了operation的框架后，我们现在可以为operation提供
 [inputs](../../DefiningDialects/Operations.md/#operation-arguments) 和
  [outputs](../../DefiningDialects/Operations.md/#operation-results)。操作的输入，
  或称为参数，可以是属性或用于 SSA 操作数值的类型。输出对应于操作产生的值的类型集合：

```tablegen
def ConstantOp : Toy_Op<"constant"> {

  // 常量操作将属性作为唯一输入。
  // `F64ElementsAttr` 对应于一个 64 位浮点元素属性。
  let arguments = (ins F64ElementsAttr:$value);

  // 常量操作返回一个 TensorType 类型的单一值。
  // F64Tensor 对应于一个 64 位浮点 TensorType。
  let results = (outs F64Tensor);

}
```

通过为参数或结果提供名称，例如 `$value`，
ODS 将自动生成一个匹配的访问器accessor：`DenseElementsAttr ConstantOp::value()`。


#### 添加文档

定义operation后的下一步是为其添加文档。
操作可以提供 [`summary` 和 `description`](../../DefiningDialects/Operations.md/#operation-documentation) 
字段来描述操作的语义。这些信息对于方言的用户非常有用，甚至可以用来自动生成 Markdown 文档。


```tablegen
def ConstantOp : Toy_Op<"constant"> {
  // 为此操作提供摘要和描述。可以用来自动生成我们方言中操作的文档。
  let summary = "constant operation";
  let description = [{
    Constant operation turns a literal into an SSA value. The data is attached
    to the operation as an attribute. For example:

      %0 = "toy.constant"()
         { value = dense<[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]> : tensor<2x3xf64> }
        : () -> tensor<2x3xf64>
  }];

  // 常量操作将属性作为唯一输入。
  // `F64ElementsAttr` 对应于一个 64 位浮点元素属性。
  let arguments = (ins F64ElementsAttr:$value);

  // 通用调用操作返回一个 TensorType 类型的单一值。
  // F64Tensor 对应于一个 64 位浮点 TensorType。
  let results = (outs F64Tensor);
}
```

#### Verifying Operation Semantics

At this point we've already covered a majority of the original C++ operation
definition. The next piece to define is the verifier. Luckily, much like the
named accessor, the ODS framework will automatically generate a lot of the
necessary verification logic based upon the constraints we have given. This
means that we don't need to verify the structure of the return type, or even the
input attribute `value`. In many cases, additional verification is not even
necessary for ODS operations. To add additional verification logic, an operation
can override the [`verifier`](../../DefiningDialects/Operations.md/#custom-verifier-code)
field. The `verifier` field allows for defining a C++ code blob that will be run
as part of `ConstantOp::verify`. This blob can assume that all of the other
invariants of the operation have already been verified:

#### 验证操作语义

到目前为止，我们已经涵盖了大部分原始 C++ 操作定义。接下来需要定义的是验证器(verifier)。
幸运的是，像命名访问器一样，ODS 框架会根据我们给定的约束自动生成大量必要的验证逻辑。
这意味着我们不需要验证返回类型的结构，甚至不需要验证输入属性 `value`。
在许多情况下，ODS 操作甚至不需要额外的验证。为了添加额外的验证逻辑，
operation可以重写 [`verifier`](../../DefiningDialects/Operations.md/#custom-verifier-code) 字段。
`verifier` 字段允许定义一个 C++ 代码块，该代码块将在 `ConstantOp::verify` 中作为一部分运行。
此代码块可以假设操作的其他所有不变式已经通过验证：

```tablegen
def ConstantOp : Toy_Op<"constant"> {
  // Provide a summary and description for this operation. This can be used to
  // auto-generate documentation of the operations within our dialect.
  let summary = "constant operation";
  let description = [{
    Constant operation turns a literal into an SSA value. The data is attached
    to the operation as an attribute. For example:

      %0 = "toy.constant"()
         { value = dense<[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]> : tensor<2x3xf64> }
        : () -> tensor<2x3xf64>
  }];

  // The constant operation takes an attribute as the only input.
  // `F64ElementsAttr` corresponds to a 64-bit floating-point ElementsAttr.
  let arguments = (ins F64ElementsAttr:$value);

  // The generic call operation returns a single value of TensorType.
  // F64Tensor corresponds to a 64-bit floating-point TensorType.
  let results = (outs F64Tensor);

  // Add additional verification logic to the constant operation. Setting this bit
  // to `1` will generate a `::llvm::LogicalResult verify()` declaration on the
  // operation class that is called after ODS constructs have been verified, for
  // example the types of arguments and results. We implement additional verification
  // in the definition of this `verify` method in the C++ source file.
  let hasVerifier = 1;
}
```

#### Attaching `build` Methods

The final missing component here from our original C++ example are the `build`
methods. ODS can generate some simple build methods automatically, and in this
case it will generate our first build method for us. For the rest, we define the
[`builders`](../../DefiningDialects/Operations.md/#custom-builder-methods) field. This field
takes a list of `OpBuilder` objects that take a string corresponding to a list
of C++ parameters, as well as an optional code block that can be used to specify
the implementation inline.


#### 附加 `build` 方法

在我们原始 C++ 示例中，最后一个缺失的部分是 `build` 方法。
ODS 可以自动生成一些简单的构建方法，在这种情况下，它将为我们生成第一个构建方法。
对于其余的部分，我们定义 [`builders`](../../DefiningDialects/Operations.md/#custom-builder-methods) 字段。
此字段接受一个 `OpBuilder` 对象的列表，这些对象接收一个对应于 C++ 参数列表的字符串，以及一个可选的代码块，用于指定内联实现。


```tablegen
def ConstantOp : Toy_Op<"constant"> {
  ...

  // Add custom build methods for the constant operation. These methods populate
  // the `state` that MLIR uses to create operations, i.e. these are used when
  // using `builder.create<ConstantOp>(...)`.
  let builders = [
    // Build a constant with a given constant tensor value.
    OpBuilder<(ins "DenseElementsAttr":$value), [{
      // Call into an autogenerated `build` method.
      build(builder, result, value.getType(), value);
    }]>,

    // Build a constant with a given constant floating-point value. This builder
    // creates a declaration for `ConstantOp::build` with the given parameters.
    OpBuilder<(ins "double":$value)>
  ];
}
```

#### 指定自定义汇编格式

到此为止，我们可以生成我们的“Toy IR”。例如，以下内容：

```toy
# User defined generic function that operates on unknown shaped arguments.
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  var a<2, 3> = [[1, 2, 3], [4, 5, 6]];
  var b<2, 3> = [1, 2, 3, 4, 5, 6];
  var c = multiply_transpose(a, b);
  var d = multiply_transpose(b, a);
  print(d);
}
```

产生以下 IR：

```mlir
module {
  "toy.func"() ({
  ^bb0(%arg0: tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":4:1), %arg1: tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":4:1)):
    %0 = "toy.transpose"(%arg0) : (tensor<*xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:10)
    %1 = "toy.transpose"(%arg1) : (tensor<*xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:25)
    %2 = "toy.mul"(%0, %1) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:25)
    "toy.return"(%2) : (tensor<*xf64>) -> () loc("test/Examples/Toy/Ch2/codegen.toy":5:3)
  }) {sym_name = "multiply_transpose", type = (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>} : () -> () loc("test/Examples/Toy/Ch2/codegen.toy":4:1)
  "toy.func"() ({
    %0 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64> loc("test/Examples/Toy/Ch2/codegen.toy":9:17)
    %1 = "toy.reshape"(%0) : (tensor<2x3xf64>) -> tensor<2x3xf64> loc("test/Examples/Toy/Ch2/codegen.toy":9:3)
    %2 = "toy.constant"() {value = dense<[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00]> : tensor<6xf64>} : () -> tensor<6xf64> loc("test/Examples/Toy/Ch2/codegen.toy":10:17)
    %3 = "toy.reshape"(%2) : (tensor<6xf64>) -> tensor<2x3xf64> loc("test/Examples/Toy/Ch2/codegen.toy":10:3)
    %4 = "toy.generic_call"(%1, %3) {callee = @multiply_transpose} : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":11:11)
    %5 = "toy.generic_call"(%3, %1) {callee = @multiply_transpose} : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":12:11)
    "toy.print"(%5) : (tensor<*xf64>) -> () loc("test/Examples/Toy/Ch2/codegen.toy":13:3)
    "toy.return"() : () -> () loc("test/Examples/Toy/Ch2/codegen.toy":8:1)
  }) {sym_name = "main", type = () -> ()} : () -> () loc("test/Examples/Toy/Ch2/codegen.toy":8:1)
} loc(unknown)
```

One thing to notice here is that all of our Toy operations are printed using the
generic assembly format. This format is the one shown when breaking down
`toy.transpose` at the beginning of this chapter. MLIR allows for operations to
define their own custom assembly format, either
[declaratively](../../DefiningDialects/Operations.md/#declarative-assembly-format) or
imperatively via C++. Defining a custom assembly format allows for tailoring the
generated IR into something a bit more readable by removing a lot of the fluff
that is required by the generic format. Let's walk through an example of an
operation format that we would like to simplify.

需要注意的一点是，我们所有的 Toy operations都使用通用汇编格式打印出来。
这个格式在本章开始时拆解 `toy.transpose` 时就已经展示过了。MLIR 允许operations定义它们自己的自定义汇编格式，
可以通过 [declaratively](../../DefiningDialects/Operations.md/#declarative-assembly-format) 
或通过 C++ 命令式方式定义。定义自定义汇编格式允许通过去除通用格式中许多冗余的内容，
将生成的 IR 调整为更易读的形式。让我们通过一个示例来看如何简化操作格式。

##### `toy.print`

The current form of `toy.print` is a little verbose. There are a lot of
additional characters that we would like to strip away. Let's begin by thinking
of what a good format of `toy.print` would be, and see how we can implement it.
Looking at the basics of `toy.print` we get:

```mlir
toy.print %5 : tensor<*xf64> loc(...)
```

Here we have stripped much of the format down to the bare essentials, and it has
become much more readable. To provide a custom assembly format, an operation can
either override the `hasCustomAssemblyFormat` field for a C++ format, or the
`assemblyFormat` field for the declarative format. Let's look at the C++ variant
first, as this is what the declarative format maps to internally.

```tablegen
/// Consider a stripped definition of `toy.print` here.
def PrintOp : Toy_Op<"print"> {
  let arguments = (ins F64Tensor:$input);

  // Divert the printer and parser to `parse` and `print` methods on our operation,
  // to be implemented in the .cpp file. More details on these methods is shown below.
  let hasCustomAssemblyFormat = 1;
}
```

A C++ implementation for the printer and parser is shown below:

```c++
/// The 'OpAsmPrinter' class is a stream that will allows for formatting
/// strings, attributes, operands, types, etc.
void PrintOp::print(mlir::OpAsmPrinter &printer) {
  printer << "toy.print " << op.input();
  printer.printOptionalAttrDict(op.getAttrs());
  printer << " : " << op.input().getType();
}

/// The 'OpAsmParser' class provides a collection of methods for parsing
/// various punctuation, as well as attributes, operands, types, etc. Each of
/// these methods returns a `ParseResult`. This class is a wrapper around
/// `LogicalResult` that can be converted to a boolean `true` value on failure,
/// or `false` on success. This allows for easily chaining together a set of
/// parser rules. These rules are used to populate an `mlir::OperationState`
/// similarly to the `build` methods described above.
mlir::ParseResult PrintOp::parse(mlir::OpAsmParser &parser,
                                 mlir::OperationState &result) {
  // Parse the input operand, the attribute dictionary, and the type of the
  // input.
  mlir::OpAsmParser::UnresolvedOperand inputOperand;
  mlir::Type inputType;
  if (parser.parseOperand(inputOperand) ||
      parser.parseOptionalAttrDict(result.attributes) || parser.parseColon() ||
      parser.parseType(inputType))
    return mlir::failure();

  // Resolve the input operand to the type we parsed in.
  if (parser.resolveOperand(inputOperand, inputType, result.operands))
    return mlir::failure();

  return mlir::success();
}
```

With the C++ implementation defined, let's see how this can be mapped to the
[declarative format](../../DefiningDialects/Operations.md/#declarative-assembly-format). The
declarative format is largely composed of three different components:

*   Directives
    -   A type of builtin function, with an optional set of arguments.
*   Literals
    -   A keyword or punctuation surrounded by \`\`.
*   Variables
    -   An entity that has been registered on the operation itself, i.e. an
        argument(attribute or operand), result, successor, etc. In the `PrintOp`
        example above, a variable would be `$input`.

A direct mapping of our C++ format looks something like:

```tablegen
/// Consider a stripped definition of `toy.print` here.
def PrintOp : Toy_Op<"print"> {
  let arguments = (ins F64Tensor:$input);

  // In the following format we have two directives, `attr-dict` and `type`.
  // These correspond to the attribute dictionary and the type of a given
  // variable represectively.
  let assemblyFormat = "$input attr-dict `:` type($input)";
}
```

The [declarative format](../../DefiningDialects/Operations.md/#declarative-assembly-format) has
many more interesting features, so be sure to check it out before implementing a
custom format in C++. After beautifying the format of a few of our operations we
now get a much more readable:

```mlir
module {
  toy.func @multiply_transpose(%arg0: tensor<*xf64>, %arg1: tensor<*xf64>) -> tensor<*xf64> {
    %0 = toy.transpose(%arg0 : tensor<*xf64>) to tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:10)
    %1 = toy.transpose(%arg1 : tensor<*xf64>) to tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:25)
    %2 = toy.mul %0, %1 : tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:25)
    toy.return %2 : tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":5:3)
  } loc("test/Examples/Toy/Ch2/codegen.toy":4:1)
  toy.func @main() {
    %0 = toy.constant dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64> loc("test/Examples/Toy/Ch2/codegen.toy":9:17)
    %1 = toy.reshape(%0 : tensor<2x3xf64>) to tensor<2x3xf64> loc("test/Examples/Toy/Ch2/codegen.toy":9:3)
    %2 = toy.constant dense<[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00]> : tensor<6xf64> loc("test/Examples/Toy/Ch2/codegen.toy":10:17)
    %3 = toy.reshape(%2 : tensor<6xf64>) to tensor<2x3xf64> loc("test/Examples/Toy/Ch2/codegen.toy":10:3)
    %4 = toy.generic_call @multiply_transpose(%1, %3) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":11:11)
    %5 = toy.generic_call @multiply_transpose(%3, %1) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":12:11)
    toy.print %5 : tensor<*xf64> loc("test/Examples/Toy/Ch2/codegen.toy":13:3)
    toy.return loc("test/Examples/Toy/Ch2/codegen.toy":8:1)
  } loc("test/Examples/Toy/Ch2/codegen.toy":8:1)
} loc(unknown)
```

Above we introduce several of the concepts for defining operations in the ODS
framework, but there are many more that we haven't had a chance to: regions,
variadic operands, etc. Check out the
[full specification](../../DefiningDialects/Operations.md) for more details.

## Complete Toy Example

We can now generate our "Toy IR". You can build `toyc-ch2` and try yourself on
the above example: `toyc-ch2 test/Examples/Toy/Ch2/codegen.toy -emit=mlir
-mlir-print-debuginfo`. We can also check our RoundTrip: `toyc-ch2
test/Examples/Toy/Ch2/codegen.toy -emit=mlir -mlir-print-debuginfo 2>
codegen.mlir` followed by `toyc-ch2 codegen.mlir -emit=mlir`. You should also
use `mlir-tblgen` on the final definition file and study the generated C++ code.

At this point, MLIR knows about our Toy dialect and operations. In the
[next chapter](Ch-3.md), we will leverage our new dialect to implement some
high-level language-specific analyses and transformations for the Toy language.
