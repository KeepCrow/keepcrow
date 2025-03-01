---
layout: framework
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第七章"
date:   2025-03-01 21:41:46 +0800
categories: Interpreter
---

如前文所述，今天我将向大家介绍一个至关重要的数据结构，它将在接下来的系列中扮演核心角色。

## 中间表示
迄今为止，我们的解释器与解析器代码是紧密耦合的，解析器在识别到语言的特定结构（如加法，减法，乘法或除法）的同时，解释器会即时对其进行计算。这类解释器被称为**语法导向解释器（Syntax-Directed Interpreters）**，通常适用于只需对输入进行一次处理的简单语言。

为了能够处理像 Pascal 这样复杂语言的结构，我们需要引入中间表示（IR，Intermediate Representation）。有了它，我们就可以多次快捷地处理用户输入的文本。接下来，我们将修改解析器，让它可以把输入转化为 IR。

在众多数据结构中，树是非常适合 IR 的。因为我们可以使用子树表示子表达式，使用树的层级表示优先级，使用树的遍历表示求值过程等等，这些后面都会介绍到。
![](/assets/images/Pasted%20image%2020240715194749.png)

先来回顾一下树的相关概念：
- 树是一种数据结构，由一个或多个组织成层次结构的节点组成
- 树有一个根节点，位于最顶部
- 除了根节点，每个节点都有父节点
- 下图中标签为 `*` 的节点为标签为2和7的父节点，与之对应的，二者是它的子节点；可以看到，两个子节点从左往右进行排列
- 没有子节点的节点称为叶节点
- 既有父节点又有子节点的节点称为内部节点
- 一个子树可以是完全子树。在下图中，`+` 节点的左节点（标签为 `*`）就是一个完全子树
- 在计算机领域中，我们从根节点开始向下绘制树

下图是表达式 $2 \times 7 + 3$ 的树，并附上了注释：
![](/assets/images/Pasted%20image%2020240715195709.png)

## 解析树
后续的系列中，我们将会一直使用的 IR 叫做**抽象语法树（AST，Abstract Syntax Tree）**。但是在深入 AST 之前，让我们先简单聊一下**解析树（Parse Tree）**。

尽管我们不打算在后续的开发中使用解析树，但它能直观地展示解释器的工作流程，帮助我们更好地理解解析器如何处理输入数据。此外，我们还会对比解析树与AST，以此阐明为何AST更适合用于中间表示（IR）

所以，什么是解析树？解析树是一种树，它可以根据我们的文法定义把语言的结构表示出来。基本上来说，它可以把你的解释器如何识别语言结构的过程展示出来。或者用前文提到的知识，解析树可以展示出你所定义的文法的起始符是如何推导出语言中一个特定的短语的。

在代码中，解析器在尝试解析一个特定的语言结构时，会通过调用栈的形式自动构建出解析树。

让我们来看一下表达式 $2 \times 7 + 3$ 的解析树：
![](/assets/images/Pasted%20image%2020240715201028.png)

从上图中你可以看到：
- 解析树记录了一系列用于识别输入的[规则](第四章%20上下文无关文法.md#什么样)
- 解析树的根节点标签是文法的起始符
- 每个内部节点的标签都是非终止符，如：`expr`，`term` 或 `factor`
- 每个叶节点的标签都是词元

如我前文所言，我们不打算手动构造解析树用于我们的解释器，但是解析树确实可以通过可视化解析器的调用关系来帮助我们理解。

为此我迅速写了一个脚本可以用来把不同的表达式转化为解析树的小工具：[genptdot.py](https://github.com/rspivak/lsbasi/blob/master/part7/python/genptdot.py)/assets/images/。想要使用这个工具，你需要先安装 Graphviz 包，然后在命令行中按照如下命令执行工具，就可以在 `parsetree.png` 文件中看到根据你输入的表达式生成的解析树了：
```shell
$ python genptdot.py "14 + 2 * 3 - 6 / 2" > \
  parsetree.dot && dot /assets/images/-Tpng -o parsetree.png parsetree.dot
```

下图是上面命令行输出的结果：
![](/assets/images/Pasted%20image%2020240715204455.png)

可以试着用一下这个工具，输入不同的表达式，然后看看它们的解析树长什么样。

## 抽象语法树
现在，让我们再来讨论一下抽象语法树，AST。AST 在系列的剩余内容中将扮演者重要的角色，是我们的解释器和编译器项目的核心数据结构。

为了区分 AST 与解析树，让我们来看一下表达式 $2 \times 7 + 3$ 的 AST 与解析树：
![](/assets/images/Pasted%20image%2020240715205625.png)

从上图中你可以看到如下不同点：
- AST 使用运算符作为根节点和内部节点，而运算数都是运算符的子节点
- AST 不使用内部节点来表示文法的规则，这是解析树的特点
- AST 并不会把语法的所有细节都表达出来（这就是为什么叫它的名字里带抽象）：比如没有规则节点，没有括号节点
- AST 相对于解析树来说，信息密度更高

所以，什么是抽象语法树？**抽象语法树是一种树，它可以抽象地表示一个语言结构，其内部节点和根节点用于表示操作符，操作符的子节点用于表示操作数**

刚刚提到，AST 比解析树的信息密度更高。举个很简单的例子，通过表达式 $7 + ((2 + 3))$ 的两种表达方式，可以看到 AST 比解析树更小，但是它依旧保留了表达式的本质计算逻辑：
![](/assets/images/Pasted%20image%2020240715211709.png)

到目前为止都很顺利，但是我们要如何在 AST 中表达优先级呢？说起来比较简单，在 AST 中，如果你想要表达运算符的优先级，或者说想表达“X 发生于 Y 之前”，只需要把 X 的位置放得比 Y 低即可。实际上前面的图已经有所展示。

现在让我们看一个更加简单易懂的例子。

下图中，左边的树是表达式 $2 \times 7 + 3$ 的 AST，显然乘法应该先于加法执行，因此在树中 `*` 位于 `+` 的下层。那么让我们把 $7 + 3$ 放进括号，以提高加法的优先级，也就是说，表达式变成了 $2 \times (7 + 3)$，右边的树就是表达式的 AST。可以看到，此时 `+` 位于 `*` 的下层：
![](/assets/images/Pasted%20image%2020240717183548.png)

下图是表达式 $1+2+3+4+5$ 的 AST：
![](/assets/images/Pasted%20image%2020240717184005.png)

从上图中可以看到，具有更高优先级的运算符最终在树的结构中处于更低的位置。

#### AST 实现
好了，现在让我们写一些代码来实现各种 AST 的节点类型，然后把我们的解析器改造成生成可以生成 AST 的样子。

首先，我们创建一个名为 AST 的基类用于其他节点类继承：
```python
class AST(object):
	pass
```

这里没啥好说的。目前为止，我们已经有了4种操作符和整数操作数，其中操作符包括加减乘除。虽然我们可以创建4个节点类，但对他们进行抽象，我们可以得到一个二元操作符类 `BinOp`。所谓二元操作符，就是指可以作用于两个操作数的操作符。
```python
class BinOp(AST):
	def __init__(self, left, op, right):
	    self.left = left
	    self.token = self.op = op
	    self.right = right
```

构造函数的参数包括：`left`，`op` 以及 `right`。其中 `left` 与 `right` 表示两个操作数节点，`op` 则存储了操作符本身这个词元：对于加法操作符来说就是 `Token(PLUS, '+')`，对于减法操作符来说就是 `Token(MINUS, '-')`，等等。

为了能在 AST 中表示整数，我们需要定义一个 `Num` 类用于存储一个 INTEGER 类型的词元以及词元的值：
```python
class Num(AST):
    def __init__(self, token):
        self.token = token
        self.value = token.value
```

你应该已经注意到了，所有的节点类都把用于创造他们的词元存储了下来，这主要是为了以后要用的时候方便取用。

现在回想一下表达式 $2 \times 7 + 3$ 的 AST，现在我们要用 python 手动创建它了：
```shell
>>> from spi import Token, MUL, PLUS, INTEGER, Num, BinOp
>>>
>>> mul_token = Token(MUL, '*')
>>> plus_token = Token(PLUS, '+')
>>> mul_node = BinOp(
...     left=Num(Token(INTEGER, 2)),
...     op=mul_token,
...     right=Num(Token(INTEGER, 7))
... )
>>> add_node = BinOp(
...     left=mul_node,
...     op=plus_token,
...     right=Num(Token(INTEGER, 3))
... )
```

下图展示了用我们定义的类所构造出的 AST 长什么样，以及它的构造过程：
![](/assets/images/Pasted%20image%2020240717193837.png)

下面是修改后的解析器代码，它可以识别输入（一个数学表达式），然后构造并返回对应的 AST：
```python
class AST(object):
    pass

class BinOp(AST):
    def __init__(self, left, op, right):
        self.left = left
        self.token = self.op = op
        self.right = right

class Num(AST):
    def __init__(self, token):
        self.token = token
        self.value = token.value

class Parser(object):
    def __init__(self, lexer):
        self.lexer = lexer
        # 把从输入解析的第一个词元赋值为当前词元
        self.current_token = self.lexer.get_next_token()

    def error(self):
        raise Exception('Invalid syntax')

    def eat(self, token_type):
        # 判断当前词元的类型是否符合预期：如果符合就“吃掉”当前词元，
        # 然后把下一个词元赋值给 self.current_token；否则报错
        if self.current_token.type == token_type:
            self.current_token = self.lexer.get_next_token()
        else:
            self.error()

    def factor(self):
        """factor : INTEGER | LPAREN expr RPAREN"""
        token = self.current_token
        if token.type == INTEGER:
            self.eat(INTEGER)
            return Num(token)
        elif token.type == LPAREN:
            self.eat(LPAREN)
            node = self.expr()
            self.eat(RPAREN)
            return node

	def term(self):
        """term : factor ((MUL | DIV) factor)*"""
        node = self.factor()
  
        while self.current_token.type in (MUL, DIV):
            token = self.current_token
            if token.type == MUL:
                self.eat(MUL)
            elif token.type == DIV:
                self.eat(DIV)
            node = BinOp(node, token, self.factor())
  
        return node

	def expr(self):
		"""
		expr   : term ((PLUS | MINUS) term)*
		term   : factor ((MUL | DIV) factor)*
		factor : INTEGER
		"""
		node = self.term()
		
		while self.current_token.type in (PLUS, MINUS):
		    token = self.current_token
		
		    if token.type == PLUS:
		        self.eat(PLUS)
		    elif token.type == MINUS:
		        self.eat(MINUS)
            node = BinOp(node, token, self.term())

		return node

    def parse(self):
        return self.expr()
```

让我们再来复习一下数学表达式的 AST 的构建过程。

如果你看了上面的代码，就会发现它构建 AST 的方法为：每一个 `BinOp` 节点直接把当前的 `node` 节点当作左节点来使用，然后再调用 `term` 或 `factor`，把返回值当作右节点。

因此在生成的过程中，优先级更高的会被往左下方推，$1 +2+3+4+5$ 就是个例子。下图是表达式 $1 +2+3+4+5$ 的 AST 的可视化构造过程，你可以看到高优先级的操作符是怎么一步步被推到最下面的（前文提到，AST 越靠下的节点优先级越高）：
![](/assets/images/Pasted%20image%2020240717201007.png)

为了能让你看到不同的表达式的 AST，我写了一个简单的工具，输入数学表达式就可以生成一个 DOT 文件，然后再使用 dot 工具处理一下就可以绘制一个 AST 出来（dot 是 Graphviz 的一部分，你需要安装以后才能使用 dot 命令）。

下面是我使用小工具为表达式 $7 + 3 \times (10 / (12 / (3 + 1) - 1))$ 生成 AST 的命令以及结果：
```shell
$ python genastdot.py "7 + 3 * (10 / (12 / (3 + 1) - 1))" > \
  ast.dot && dot /assets/images/-Tpng -o ast.png ast.dot
```
![](/assets/images/Pasted%20image%2020240717204215.png)

我觉得你很有必要写下一些数学表达式，手绘出他们的 AST，然后再通过 [genastdot.py](https://github.com/rspivak/lsbasi/blob/master/part7/python/genastdot.py) 工具生成图片来验证你的理解知否正确。这可以帮助你更好地理解数学表达式的 AST 是如何通过解析器构造出来的。

### 计算 AST
好了，现在这里有个表达式 $2 \times 7 + 3$ 的 AST：
![](/assets/images/Pasted%20image%2020240717204519.png)

你要如何使用这个树来计算出表达式的结果呢？显而易见，是[后序遍历](202407242209%20后序遍历：TODO.md)，一种特殊的[深度优先遍历](202407242208%20深度优先遍历：TODO.md)。这种算法需要先递归访问树的所有子节点，最后再访问根节点。后序遍历会尽快地远离根节点，直到不能远离才会开始访问。比如上图中的访问顺序为：`2 -> 7 -> * -> 3 -> +`。

下图是后续遍历的伪代码，其中 `《postorder actions》` 表示执行加减乘除一类的操作，并将处理结果以节点的形式返回上一层。
![](/assets/images/Pasted%20image%2020240726225245.png)

之所以要用后序遍历，主要有以下几个原因：
1. 我们需要先获取一个节点的子树的结果，因为它代表着表达式中更高优先级的计算过程
2. 我们需要先获取操作数，才能根据操作符进行计算（AST 中一个节点的左右子节点要么是操作数，要么是一个计算过程，最终也可以得到操作数）
从下图中可以看到，后序遍历的情况下我们会先计算 $2 \times 7$ 得到 $14$，然后再计算 $14 + 3$ 得到最终的结果：$17$
![](/assets/images/Pasted%20image%2020240726230012.png)

出于完整性的考虑，我不得不提一嘴，深度优先遍历有三种算法：前序遍历，中序遍历，以及刚提到的后序遍历。三种遍历方法的称呼来自于你何时访问当前节点：
- 如果先访问当前节点，再访问子节点：前序遍历
- 如果先访问左子节点，再访问当前节点，最后访问右节点：中序遍历
- 如果先访问子节点，再访问当前节点：后序遍历

下面的伪代码可能更加直观一些：
![](/assets/images/Pasted%20image%2020240728000206.png)

然而，有的时候，你会在三处都有一些特定的操作（前序，中序和后序）。我们的文章中所提到的代码就是个例子。

好了，现在让我们写一些代码来遍历并计算解析器所构建的 AST 吧。

下面是源码，值得一提的是，它应用了[访问者模式](https://refactoringguru.cn/design-patterns/visitor)：
```python
class NodeVisitor(object):
    def visit(self, node):
        method_name = 'visit_' + type(node).__name__
        visitor = getattr(self, method_name, self.generic_visit)
        return visitor(node)

    def generic_visit(self, node):
        raise Exception('No visit_{} method'.format(type(node).__name__))
```

下面是 `Interpreter` 类的源码，它继承了 `NodeVisitor`，并实现了一些形式为 `visit_NodeType` 的访问函数。其中，`NodeType` 可以替换为各种节点类型，如访问 `BinOp` 的函数就叫 `visit_BinOp`，等等
```python
class Interpreter(NodeVisitor):
    def __init__(self, parser):
        self.parser = parser

    def visit_BinOp(self, node):
        if node.op.type == PLUS:
            return self.visit(node.left) + self.visit(node.right)
        elif node.op.type == MINUS:
            return self.visit(node.left) - self.visit(node.right)
        elif node.op.type == MUL:
            return self.visit(node.left) * self.visit(node.right)
        elif node.op.type == DIV:
            return self.visit(node.left) / self.visit(node.right)

    def visit_Num(self, node):
        return node.value
```

有两个有趣的点值得注意一下。

首先是操作 AST 节点的访问者代码与 AST 节点本身是解耦的。可以看到所有的 AST 节点类中都没有提供任何用于操作自身数据的函数，这些操作是 `Interpreter` 类从 `NodeVisitor` 类中继承过来的，也就是说，它封装于 `NodeVisitor` 类中。

第二，`NodeVisitor` 的 `visit` 函数不是一个巨大的 `if` 块，而是下面这种：
```python
def visit(node):
    node_type = type(node).__name__
    if node_type == 'BinOp':
        return self.visit_BinOp(node)
    elif node_type == 'Num':
        return self.visit_Num(node)
    elif ...
    # ...
```
或者这种：
```python
def visit(node):
    if isinstance(node, BinOp):
        return self.visit_BinOp(node)
    elif isinstance(node, Num):
        return self.visit_Num(node)
    elif ...
```

`NodeVisitor` 的 `visit` 函数是非常通用的，它可以根据节点的类型调用对应的访问函数。正如前面提到的，为了能够使用 `visit` 函数，`Interpreter` 类从 `NodeVisitor` 类中将其继承了过来，并实现了其需要的函数，如 `visit_Num`，`visit_BinOp` 等等。

当调用 `visit` 函数，并传入一个 `BinOp` 类型的节点，那么 `visit` 函数会将根据它的类型把它交给 `visit_BinOp` 函数进行访问。同理，如果传入的时 `Num` 类型的节点，`visit` 函数会把它交给 `visit_Num` 函数进行访问，等等。

你可以花点时间好好理解一下这个设计模式，因为未来我们将会用这种模式为解释器拓展更多的节点类型，实现更多的 `visit_NodeType`。顺带一提，Python 官方的 `ast` 模块也采用了这种机制来实现节点遍历。

`generic_visit` 函数是最后的备选方案。即当 `Interpreter` 类中没有实现节点类型对应的 `visit_NodeType` 函数时，就调用 `generic_visit` 函数，表示解释过程出现了错误。

现在，让我们一起手动为 $2 \times 7 + 3$ 创建 AST，并将其输入给我们的解释器，来看看它的 `visit` 函数能不能把表达式的结果计算出来。下面是你在 python shell 里可以做的事情：
```python
>>> from spi import Token, MUL, PLUS, INTEGER, Num, BinOp
>>>
>>> mul_token = Token(MUL, '*')
>>> plus_token = Token(PLUS, '+')
>>> mul_node = BinOp(
...     left=Num(Token(INTEGER, 2)),
...     op=mul_token,
...     right=Num(Token(INTEGER, 7))
... )
>>> add_node = BinOp(
...     left=mul_node,
...     op=plus_token,
...     right=Num(Token(INTEGER, 3))
... )
>>> from spi import Interpreter
>>> inter = Interpreter(None)
>>> inter.visit(add_node)
17
```

你可以看到，我们把表达式树的根节点传入了 `visit` 函数。然后根节点就根据类型传给了 `Interpreter` 类的 `visit_BinOp` 函数去访问，并生成结果。

## 完整代码

好了，下面是完整代码：
```python
""" SPI - Simple Pascal Interpreter """

###############################################################################
#                                                                             #
#  LEXER                                                                      #
#                                                                             #
###############################################################################

# Token types
#
# EOF (end-of-file) token is used to indicate that
# there is no more input left for lexical analysis
INTEGER, PLUS, MINUS, MUL, DIV, LPAREN, RPAREN, EOF = (
    'INTEGER', 'PLUS', 'MINUS', 'MUL', 'DIV', '(', ')', 'EOF'
)

class Token(object):
    def __init__(self, type, value):
        self.type = type
        self.value = value

    def __str__(self):
        """String representation of the class instance.

        Examples:
            Token(INTEGER, 3)
            Token(PLUS, '+')
            Token(MUL, '*')
        """
        return 'Token({type}, {value})'.format(
            type=self.type,
            value=repr(self.value)
        )

    def __repr__(self):
        return self.__str__()

class Lexer(object):
    def __init__(self, text):
        # client string input, e.g. "4 + 2 * 3 - 6 / 2"
        self.text = text
        # self.pos is an index into self.text
        self.pos = 0
        self.current_char = self.text[self.pos]

    def error(self):
        raise Exception('Invalid character')

    def advance(self):
        """Advance the `pos` pointer and set the `current_char` variable."""
        self.pos += 1
        if self.pos > len(self.text) - 1:
            self.current_char = None  # Indicates end of input
        else:
            self.current_char = self.text[self.pos]

    def skip_whitespace(self):
        while self.current_char is not None and self.current_char.isspace():
            self.advance()

    def integer(self):
        """Return a (multidigit) integer consumed from the input."""
        result = ''
        while self.current_char is not None and self.current_char.isdigit():
            result += self.current_char
            self.advance()
        return int(result)

    def get_next_token(self):
        """Lexical analyzer (also known as scanner or tokenizer)

        This method is responsible for breaking a sentence
        apart into tokens. One token at a time.
        """
        while self.current_char is not None:

            if self.current_char.isspace():
                self.skip_whitespace()
                continue

            if self.current_char.isdigit():
                return Token(INTEGER, self.integer())

            if self.current_char == '+':
                self.advance()
                return Token(PLUS, '+')

            if self.current_char == '-':
                self.advance()
                return Token(MINUS, '-')

            if self.current_char == '*':
                self.advance()
                return Token(MUL, '*')

            if self.current_char == '/':
                self.advance()
                return Token(DIV, '/')

            if self.current_char == '(':
                self.advance()
                return Token(LPAREN, '(')

            if self.current_char == ')':
                self.advance()
                return Token(RPAREN, ')')

            self.error()

        return Token(EOF, None)

###############################################################################
#                                                                             #
#  PARSER                                                                     #
#                                                                             #
###############################################################################

class AST(object):
    pass

class BinOp(AST):
    def __init__(self, left, op, right):
        self.left = left
        self.token = self.op = op
        self.right = right

class Num(AST):
    def __init__(self, token):
        self.token = token
        self.value = token.value

class Parser(object):
    def __init__(self, lexer):
        self.lexer = lexer
        # set current token to the first token taken from the input
        self.current_token = self.lexer.get_next_token()

    def error(self):
        raise Exception('Invalid syntax')

    def eat(self, token_type):
        # compare the current token type with the passed token
        # type and if they match then "eat" the current token
        # and assign the next token to the self.current_token,
        # otherwise raise an exception.
        if self.current_token.type == token_type:
            self.current_token = self.lexer.get_next_token()
        else:
            self.error()

    def factor(self):
        """factor : INTEGER | LPAREN expr RPAREN"""
        token = self.current_token
        if token.type == INTEGER:
            self.eat(INTEGER)
            return Num(token)
        elif token.type == LPAREN:
            self.eat(LPAREN)
            node = self.expr()
            self.eat(RPAREN)
            return node

    def term(self):
        """term : factor ((MUL | DIV) factor)*"""
        node = self.factor()

        while self.current_token.type in (MUL, DIV):
            token = self.current_token
            if token.type == MUL:
                self.eat(MUL)
            elif token.type == DIV:
                self.eat(DIV)

            node = BinOp(left=node, op=token, right=self.factor())

        return node

    def expr(self):
        """
        expr   : term ((PLUS | MINUS) term)*
        term   : factor ((MUL | DIV) factor)*
        factor : INTEGER | LPAREN expr RPAREN
        """
        node = self.term()

        while self.current_token.type in (PLUS, MINUS):
            token = self.current_token
            if token.type == PLUS:
                self.eat(PLUS)
            elif token.type == MINUS:
                self.eat(MINUS)

            node = BinOp(left=node, op=token, right=self.term())

        return node

    def parse(self):
        return self.expr()

###############################################################################
#                                                                             #
#  INTERPRETER                                                                #
#                                                                             #
###############################################################################

class NodeVisitor(object):
    def visit(self, node):
        method_name = 'visit_' + type(node).__name__
        visitor = getattr(self, method_name, self.generic_visit)
        return visitor(node)

    def generic_visit(self, node):
        raise Exception('No visit_{} method'.format(type(node).__name__))

class Interpreter(NodeVisitor):
    def __init__(self, parser):
        self.parser = parser

    def visit_BinOp(self, node):
        if node.op.type == PLUS:
            return self.visit(node.left) + self.visit(node.right)
        elif node.op.type == MINUS:
            return self.visit(node.left) - self.visit(node.right)
        elif node.op.type == MUL:
            return self.visit(node.left) * self.visit(node.right)
        elif node.op.type == DIV:
            return self.visit(node.left) / self.visit(node.right)

    def visit_Num(self, node):
        return node.value

    def interpret(self):
        tree = self.parser.parse()
        return self.visit(tree)

def main():
    while True:
        try:
            try:
                text = raw_input('spi> ')
            except NameError:  # Python3
                text = input('spi> ')
        except EOFError:
            break
        if not text:
            continue

        lexer = Lexer(text)
        parser = Parser(lexer)
        interpreter = Interpreter(parser)
        result = interpreter.interpret()
        print(result)

if __name__ == '__main__':
    main()
```

今天，你学习了解析树，抽象语法树，如何构造抽象语法树以及如何计算抽象语法树。你还修改了解析器与解释器，并将它们分离开来。当前词法分析器，语法分析器以及解释器之间的接口大概如下图所示：
![](/assets/images/Pasted%20image%2020240728195700.png)

你可以把它理解为“语法解析器以从词法解析器获取到的词元作为输入，以通用的 AST 作为输出；解释器以 AST 作为输入，遍历并求出结果。”

以上就是今天的全部内容了，但在结束之前，我想简单说一下**递归下降解释器（Recursive-Decent Parser）**。这里知识简单地给个定义，因为上次我承诺说要更深入地讨论它们。递归下降解释器是一种[自上而下的解释器](202407282009%20自上而下解释器：TODO.md)，它用一系列的递归调用来处理输入。自上而下的意思就是，解释器是从最顶部的节点开始构造，然后再去构造更低的节点。

## 练习
练习时间到🤺！
![](/assets/images/Pasted%20image%2020240728201351.png)

- 写一个转换器（注：节点访问器），输入数学表达式，可以输出其后缀表达式（Postfix Notation），或者叫逆波兰表达式（Reverse Polish Notation ，RPN）。举例来说，如果给转换器输入的表达式为 `(5 + 3) * 12 / 3`，那么输出应该是 `5 3 + 12 * 3 /`。可以参考这里的[答案](https://github.com/rspivak/lsbasi/blob/master/part7/python/ex1.py)，但请务必自己先试着解决一下
- 写一个转换器（注：节点访问器），输入数学表达式可以输出其 [LISP风格表达式](202407282159%20LISP风格表示法.md)。也就是说，当你输入 `2 + 3` 时，输出就是 `(+ 2 3)`；而当你输入 `2 + 3 * 5` 时，输出就是 `(+ 2 (* 3 5))`。同样，有[答案](https://github.com/rspivak/lsbasi/blob/master/part7/python/ex2.py)，但是最好还是先自己尝试解决一下

下一次见面，我们会为 Pascal 解释器增加赋值和一元运算符的功能。在那之前，祝你和编译器玩的开心。