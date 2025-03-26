
# 第一章：Toy语言与抽象语法树（AST）

[目录]

## 语言简介

本教程将通过一种我们称之为“Toy”的玩具语言进行演示（命名总是很难...）。Toy 是一种基于张量的语言，
允许你定义函数、进行一些数学计算并打印结果。

为了保持简单，Toy 的代码生成将限制为秩小于等于 2 的张量，并且 Toy 中唯一的数据类型是 64 位浮点类型
（在 C 语言中称为 'double'）。因此，所有的数值都是隐式的双精度，
`Values` 是不可变的（即每个操作(operation)返回一个新分配的value），并且内存的回收是自动管理的。
关于这些的描述就先到这里；没有什么比通过一个例子更能帮助我们理解的了：

```toy
def main() {

  # 定义一个形状为 <2, 3> 的变量 `a`，并进行初始化。
  # 形状是根据提供的值推断出来的。
  var a = [[1, 2, 3], [4, 5, 6]];

  # b 与 a 相同，值张量被隐式重塑：定义新变量是重塑张量的方式（元素数量必须匹配）。
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # transpose() 和 print() 是唯一的内建(builtin)函数，以下代码将转置 a 和 b，
  # 并执行逐元素(element-wise)乘法，然后打印结果。
  print(transpose(a) * transpose(b));

}
```


类型检查通过类型推导在静态阶段完成；只有在需要明确张量形状时，语言才要求进行类型声明。
函数是泛型的：它们的参数是无秩的（换句话说，我们知道这些参数是张量，但并不知道其维度）。
在调用处，每遇到一个新的参数签名，函数就会被特化一次。
我们可以通过添加一个用户自定义函数来重新审视前面的例子：

```toy

# 用户自定义的泛型函数，可作用于形状未知的参数。

def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  # 定义一个形状为 <2, 3> 的变量 `a`，并进行初始化。
  var a = [[1, 2, 3], [4, 5, 6]];
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # 此调用将在两个参数上将 `multiply_transpose` 特化为 <2, 3>，
  # 并在初始化变量 `c` 时推导出返回类型为 <3, 2>。

  var c = multiply_transpose(a, b);

  # 第二次调用 `multiply_transpose`，两个参数都为 <2, 3>，
  # 将重用先前专门化并推导出的版本，并返回 <3, 2>。

  var d = multiply_transpose(b, a);

  # 使用 <3, 2>（而不是 <2, 3>）作为两个维度的新调用将
  # 触发 `multiply_transpose` 的另一次专门化。
  var e = multiply_transpose(c, d);

  # 最后，使用不兼容的形状（<2, 3> 和 <3, 2>）调用 `multiply_transpose`，
  # 将触发形状推导错误。
  var f = multiply_transpose(a, c);
}
```

## The AST

上述代码的抽象语法树（AST）相对简单；以下是它的输出：

```
Module:
  Function 
    Proto 'multiply_transpose' @test/Examples/Toy/Ch1/ast.toy:4:1
    Params: [a, b]
    Block {
      Return
        BinOp: * @test/Examples/Toy/Ch1/ast.toy:5:25
          Call 'transpose' [ @test/Examples/Toy/Ch1/ast.toy:5:10
            var: a @test/Examples/Toy/Ch1/ast.toy:5:20
          ]
          Call 'transpose' [ @test/Examples/Toy/Ch1/ast.toy:5:25
            var: b @test/Examples/Toy/Ch1/ast.toy:5:35
          ]
    } // Block
  Function 
    Proto 'main' @test/Examples/Toy/Ch1/ast.toy:8:1
    Params: []
    Block {
      VarDecl a<> @test/Examples/Toy/Ch1/ast.toy:11:3
        Literal: <2, 3>[ <3>[ 1.000000e+00, 2.000000e+00, 3.000000e+00], <3>[ 4.000000e+00, 5.000000e+00, 6.000000e+00]] @test/Examples/Toy/Ch1/ast.toy:11:11
      VarDecl b<2, 3> @test/Examples/Toy/Ch1/ast.toy:15:3
        Literal: <6>[ 1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00] @test/Examples/Toy/Ch1/ast.toy:15:17
      VarDecl c<> @test/Examples/Toy/Ch1/ast.toy:19:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:19:11
          var: a @test/Examples/Toy/Ch1/ast.toy:19:30
          var: b @test/Examples/Toy/Ch1/ast.toy:19:33
        ]
      VarDecl d<> @test/Examples/Toy/Ch1/ast.toy:22:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:22:11
          var: b @test/Examples/Toy/Ch1/ast.toy:22:30
          var: a @test/Examples/Toy/Ch1/ast.toy:22:33
        ]
      VarDecl e<> @test/Examples/Toy/Ch1/ast.toy:25:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:25:11
          var: c @test/Examples/Toy/Ch1/ast.toy:25:30
          var: d @test/Examples/Toy/Ch1/ast.toy:25:33
        ]
      VarDecl f<> @test/Examples/Toy/Ch1/ast.toy:28:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:28:11
          var: a @test/Examples/Toy/Ch1/ast.toy:28:30
          var: c @test/Examples/Toy/Ch1/ast.toy:28:33
        ]
    } // Block
```


您可以通过以下方式复现该结果并运行示例：进入`examples/toy/Ch1/`目录，
执行命令`path/to/BUILD/bin/toyc-ch1 test/Examples/Toy/Ch1/ast.toy -emit=ast`。

词法分析器代码结构简明，全部实现在单个头文件`examples/toy/Ch1/include/toy/Lexer.h`中。
语法分析器位于`examples/toy/Ch1/include/toy/Parser.h`，采用递归下降分析法实现。
若您不熟悉此类词法/语法分析器，其实现方式与LLVM Kaleidoscope教程前两章
（参见[Kaleidoscope教程](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl02.html)）所述高度相似。

[下一章](Ch-2.md)将演示如何将该AST转换为MLIR格式。