---
layout: framework
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第八章"
date:   2025-03-01 21:41:46 +0800
categories: Interpreter
---

今天，我们要聊一下**一元运算符**，具体来说是一元正号（`+`）和一元负号（`-`）运算符。

今天的许多内容都是在之前的基础上进行的拓展，如果你需要回顾之前的知识，可以直接前往[第七章 抽象语法树](第七章%20抽象语法树.md)，再过一遍。正所谓拳不离手，曲不离口，重复是一切学习的基础。

今天你的任务有：
- 拓展上下文无关文法，使之可以处理一元正负号运算符；
- 增加一个新的 AST 类：`UnaryOp`；
- 拓展解析器，使之可以生成带有 `UnaryOp` 节点的 AST；
- 拓展解释器，并增加一个新的函数 `visit_UnaryOp` 用于处理一元运算符。

开搞开搞！

目前为止，我们一直处理的都是二元运算符，如加减乘除，也就是能够对两个运算数进行计算的运算符。那么什么是一元运算符呢？没错，就是只能对一个运算数进行计算的运算符。

下面是一元正负号运算符的规则：
- 一元负号（`-`）运算符生成其操作数的相反数；
- 一元正号（`+`）运算符生成其操作数的原来的值，不做任何改变；
- 一元运算符的优先级要高于二元运算符。

在表达式 `+ - 3` 中，第一个运算符 ` + ` 表示对后续数据进行一元正号运算，第二个运算符 ` - ` 则表示对后续数据进行一元负号运算。因此表达式 `+ - 3` 相当于 `+ (- (3))` ，结果为 `-3`。另一种看法是，你可以直接说表达式中的 `(-3)` 是负数，但是在我们的代码中，一律当作一元负号运算符对 `3` 进行处理：
![](/assets/images/Pasted%20image%2020240904223218.png)

再来看看另一个表达式，`5 - - 2`：
![](/assets/images/Pasted%20image%2020240925211950.png)
在表达式 `5 - - 2` 中，第一个 `-` 表示减法运算，第二个 `-` 则是一元负号运算，取负值。

下面还有更多的例子：
![](/assets/images/Pasted%20image%2020240925212135.png)
![](/assets/images/Pasted%20image%2020240925212139.png)

现在，让我们来更新我们的上下文无关文法，把一元正负号运算囊括进来。由于一元运算符的优先级要高于四则运算运算符，所以我们将会在 `factor` 规则中加入一元运算符。

这是当前的 `factor` 规则：
![](/assets/images/Pasted%20image%2020240925212555.png)
这是修改后的 `factor` 规则：
![](/assets/images/Pasted%20image%2020240925213342.png)

如你所见，我拓展了 `factor` 规则，并使其引用自身，这样它就可以推导出类似 `- - - + - 3` 这种拥有一大堆一元运算符的合法表达式。

下图是能够推导出包含一元正负运算符表达式的上下文无关文法：
![](/assets/images/Pasted%20image%2020241012201616.png)

下一步是增加一个 AST 节点类，用于表示一元运算符。
```python
class UnaryOp(AST):
    def __init__(self, op, expr):
        self.token = self.op = op
        self.expr = expr
```

构造函数有两个入参：`op` 表示一元运算符词元（`PLUS` 或 `MINUS`），`expr` 表示一个 AST 节点。

由于升级后的上下文无关文法对 `factor` 规则做了一些修改，所以我们要修改解析器的 `factor` 函数。具体的做法是在函数里增加一些代码用于处理“(PLUS | MINUS) factor”子规则：
```python
def factor(self):
    """factor : (PLUS | MINUS) factor | INTEGER | LPAREN expr RPAREN"""
    token = self.current_token
    if token.type == PLUS:
        self.eat(PLUS)
        node = UnaryOp(token, self.factor())
        return node
    elif token.type == MINUS:
        self.eat(MINUS)
        node = UnaryOp(token, self.factor())
        return node
    elif token.type == INTEGER:
        self.eat(INTEGER)
        return Num(token)
    elif token.type == LPAREN:
        self.eat(LPAREN)
        node = self.expr()
        self.eat(RPAREN)
        return node
```

接下来我们需要拓展 `Interpreter` 类，增加 `visit_UnaryOp` 函数，用于处理一元节点：
```python
def visit_UnaryOp(self, node):
    op = node.op.type
    if op == PLUS:
        return +self.visit(node.expr)
    elif op == MINUS:
        return -self.visit(node.expr)
```

代码搞定，现在测试一下！

让我们手动为表达式 `5 - - - 2` 构造一个 AST，然后给它传到我们的解释器中，来验证新的 `visit_UnaryOp` 函数是否正确。下面是在 Python 终端中的操作步骤：
```shell
>>> from spi import BinOp, UnaryOp, Num, MINUS, INTEGER, Token
>>> five_tok = Token(INTEGER, 5)
>>> two_tok = Token(INTEGER, 2)
>>> minus_tok = Token(MINUS, '-')
>>> expr_node = BinOp(
...     Num(five_tok),
...     minus_tok,
...     UnaryOp(minus_token, UnaryOp(minus_token, Num(two_tok)))
... )
>>> from spi import Interpreter
>>> inter = Interpreter(None)
>>> inter.visit(expr_node)
3
```

上面的 AST 应该如下图所示：
![](/assets/images/Pasted%20image%2020241012213940.png)

从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part8/python/spi.py) 可以下载本文的完整源码，尝试运行一下，看看你刚刚修改的基于树结构的解释器能否正确地计算包含一元运算符的表达式的值。

下面是一些例子：
```shell
spi> - 3
-3
spi> + 3
3
spi> 5 - - - + - 3
8
spi> 5 - - - + - (3 + 4) - +2
10
```

为了能让 [genastdot.py](https://github.com/rspivak/lsbasi/blob/master/part8/python/genastdot.py) 工具能够处理一元运算符，我对其进行了更新。下面是一些使用示例，输入为包含一元运算符的表达式，输出为 AST 图像。

```shell
$ python genastdot.py "- 3" > ast.dot && dot /assets/images/-Tpng -o ast.png ast.dot
```
![](/assets/images/Pasted%20image%2020241012214910.png)

```shell
$ python genastdot.py "+ 3" > ast.dot && dot /assets/images/-Tpng -o ast.png ast.dot
```
![](/assets/images/Pasted%20image%2020241012214930.png)

```shell
$ python genastdot.py "5 - - - + - 3" > ast.dot && dot /assets/images/-Tpng -o ast.png ast.dot
```
![](/assets/images/Pasted%20image%2020241012214945.png)

```shell
$ python genastdot.py "5 - - - + - (3 + 4) - +2" \
  > ast.dot && dot /assets/images/-Tpng -o ast.png ast.dot
```
![](/assets/images/Pasted%20image%2020241012215008.png)

剩下的就是为你准备的一些新的练习了：
![](/assets/images/Pasted%20image%2020241012215044.png)
- 安装 [Free Pascal](http://www.freepascal.org/)，编译并运行 [testunary.pas](https://github.com/rspivak/lsbasi/blob/master/part8/python/testunary.pas)，验证其结果是否与你的 [spi](https://github.com/rspivak/lsbasi/blob/master/part8/python/spi.py) 解释器一致。

今天的内容到此结束，在下一篇文章中，我们将要处理赋值语句。保持关注，稍后再见！