---
layout: post
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第九章"
date:   2025-03-01 21:41:46 +0800
categories: Interpreter
---

我记得自己读大学那会儿（挺久以前了），当时我正在学操作系统开发。那时的我坚信，只有汇编和 C 才是“真正的”语言，而类似于 Pascal 这种高级语言，不过是一帮不愿意探究计算机背后逻辑的应用开发人员才会用的东西。

那会儿的我打死也想不到，未来的我会用 Python 写各种各样的东西（我爱死 Python 了），并且用它来搞钱。并且，我还要为 Pascal 写一个解释器和编译器，原因前面已经说了。

这几天，我觉得自己就跟编程语言中毒了一样，沉迷于搜罗各种编程语言的特点。不过话又说回来，我还是得说，我还是更喜欢用某一个特定的语言去做开发。我确实有偏见，而且我是第一个承认的😊。

这是以前的我：
![](/assets/images/Pasted%20image%2020241124212401.png)

这是现在的我：
![](/assets/images/Pasted%20image%2020241124212411.png)

好了，现在让我们进入正题，下面是你今天要学习的内容：
1. 如何解析并翻译一个 Pascal 代码的定义
2. 如何解析并翻译复合语句
3. 如何计息并翻译赋值语句，包括变量
4. 涉猎一些符号表的知识，关于如何存储以及查找变量

接下来，我们将以如下类似于 Pascal 的代码作为例子来引入新的概念：
```pascal
BEGIN
    BEGIN
        number := 2;
        a := number;
        b := 10 * a + 10 * number / 4;
        c := a - - b
    END;
    x := 11;
END.
```

你可能会觉得，从之前系列的命令行解释器一下子到这里步子跨的有点大，但我希望这样的跨越能带来一些挑战的激情。从现在起，它将不再仅仅是个计算器，我们要严肃起来了，向 Pascial 一样严肃。😁

现在让我们深入来看一下新语言的语法图以及对应的上下文无关文法。
![](/assets/images/Pasted%20image%2020241201150315.png)
![](/assets/images/Pasted%20image%2020241201150327.png)
![](/assets/images/Pasted%20image%2020241201150335.png)

接下来，让我们重新定义一下上下文无关文法。

1. 先从 Pascal 代码的定义开始。`Program` 一般由一个复合语句构成，并使用一个英语句点表示整个代码的结束。如下所示：
	```pascal
	BEGIN  END.
	```
	当然，Pascal 代码的定义不会这么简单，后面还有很多需要进行拓展。
2. **复合语句**（`Compound Statement`）就是一个代码块，由 `BEGIN` 和 `END` 表示开始与结束，开始与结束之间可以包含一个语句列表，或是其他的复合语句。下面是一些例子，注意每个引号都是一个完整的 Pascal 代码（以句点作为结束）：
	```Pascal
	“BEGIN END.”
	“BEGIN a := 5; x := 11 END.”
	“BEGIN a := 5; x := 11; END.”
	“BEGIN BEGIN a := 5 END; x := 11 END.”
	```
3. **语句列表**（`Statement List`）是由0个或多个复合语句中的语句。除了最后一句，语句列表中的每个语句都必须使用分号作为结束符。而代码块的最后一个语句，是否使用分号都是可以的。可以参考上例
4. **语句**（`Statement`）可以是一个复合语句，一个赋值语句或者一个空语句
5. **赋值语句**（`Assignment Statement`）是一个变量后面跟着赋值词元（`:=`），后边跟着一个表达式，如下所示：
	```Pascal
	“a := 11”
	“b := a + 9 - 5 * 2”
	```
6. **变量**（`Variable`）是一个标识符。提前透露一下，为了能够处理变量，我们会为其专门创建一个 ID 词元。这种词元的值是变量的名字，如 `a`，`number` 等等。在下面的代码中，`a` 和 `b` 都是变量
	```Pascal
	"BEGIN a := 11; b := a + 9 - 5 * 2 END"
	```
7. **空语句**（`Empty Statement`）是一个无法继续扩展的语法规则。我们将会在解析器中此单独使用一个 `empty_statement` 语法规则来表示 `statement_list` 的结束，同时也允许一个空的复合语句，如 `BEGIN END`。
8. `factor` 规则需要更新，使之可以处理变量

现在，来欣赏一下完整的上下文无关文法：
```BNF
    program : compound_statement DOT

    compound_statement : BEGIN statement_list END

    statement_list : statement
                   | statement SEMI statement_list

    statement : compound_statement
              | assignment_statement
              | empty

    assignment_statement : variable ASSIGN expr

    empty :

    expr : term ((PLUS | MINUS) term)*

    term : factor ((MUL | DIV) factor)*

    factor : PLUS factor
           | MINUS factor
           | INTEGER
           | LPAREN expr RPAREN
           | variable

    variable: ID
```
作为对比，我把之前的文法放在这里，可以对照着看一下：
```BNF
expr : term ((PLUS | MINUS) term)*

term : factor ((MUL | DIV) factor)*

factor : PLUS factor
       | MINUS factor 
       | INTEGER 
       | LPAREN expr RPAREN
```

你可能已经注意到了，在新的文法中，我没有在 `compound_satement` 中使用 `*` 来表示零个或多个重复，而是用了另一种方式：明确地指定了 `statement_list` 规则。后面你会发现这种表达方式更加接近于 PLY 这类 Parser 生成器的语法。除此之外，我还把 `(PLUS | MINUS) factor` 的子规则分成了两个独立的规则。

为了能够支持这些新的语法，我们需要对前面做的词法解析器，语法解析器以及解释器做一些修改。现在，让我们逐一看一下这些修改。

这里是我们的词法解析器需要做出的修改的总结：
![](/assets/images/Pasted%20image%2020241209204412.png)

1. 为了0...支持 Pascal 代码的定义，组合语句，赋值语句以及变量，我们的词法解析器需要能够返回新的词元：
	1. `BEGIN`，用于标识组合语句的开始
	2. `END`，用于标识组合语句的结束
	3. `DOT`，Pascal 代码定义需要的词元，特指一个英文句点 `.`
	4. `ASSIGN`，两个字符组成的字符序列 `:=`。在 Pascal 中，赋值操作符与其他语言不太一样，它们大部分都是用单个等号字符 `=` 来表示赋值。
	5. `SEMI`，一个分号字符 `;`，用于表示组合语句内的语句结束了
	6. `ID`，用于标识符的词元。标识符的首字符为字母，后续是任意数量的字母或者数字。
2. 有时候，为了能够在一开始就区分开首字符相同，但实际不同的词元（如 `:` 与 `:=`，或者 `==` 与 `=>`）。我们需要多看一个字符，但是不会真的多消耗一个字符（指针不会前移）。出于这样的目的，我引入了一个 `peek` 函数来帮我们解析赋值语句。实际上这个方法也不是目前必须要有的，只不过我觉得早点引入比较好，而且这个函数可以让我们的 `get_next_token` 函数也更清晰一些。它做的事儿就是返回输入文本的下一个字符，但是不会增加 `self.pos` 变量的值。这里是它的代码：
	```python
	def peek(self):
	    peek_pos = self.pos + 1
	    if peek_pos > len(self.text) - 1:
	        return None
	    else:
	        return self.text[peek_pos]
	```
3. 由于 Pascal 的变量和保留字都是标识符，我们把它们的处理集中到了一个函数中：`called_id`。它的工作原理就是，处理一系列的字母和数字字符，然后判断是不是保留字。如果是保留字，它就返回一个提前构造好的保留字词元。如果不是保留字，就返回一个新 ID 的词元，该词元的值是刚获取到的字符串。说到这儿，你肯定已经开始迷糊了，不如直接来看看代码，让代码解释它自己：
	```Python
	RESERVED_KEYWORDS = {
	    'BEGIN': Token('BEGIN', 'BEGIN'),
	    'END': Token('END', 'END')
	}

    def __id(self):
        """处理标识符和保留字"""
        result = ''
        while self.current_char is not None and self.current_char.isalnum():
            result += self.current_char
            self.advance()

        token = RESERVED_KEYWORDS.get(result, Token(ID, result))
        return token
	```
4. 现在，来看一眼词法解析器主函数 `get_next_token` 的改动：
	```Python
	def get_next_token(self):
	    while self.current_char is not None:
	        ...
	        if self.current_char.isalpha():
	            return self._id()
	
	        if self.current_char == ':' and self.peek() == '=':
	            self.advance()
	            self.advance()
	            return Token(ASSIGN, ':=')
	
	        if self.current_char == ';':
	            self.advance()
	            return Token(SEMI, ';')
	
	        if self.current_char == '.':
	            self.advance()
	            return Token(DOT, '.')
	        ...
	```

现在，是时候全方位欣赏我们的新词法解析器了。从 [Github](https://github.com/rspivak/lsbasi/blob/master/part9/python) 上下载源代码，然后用你的 Python 来执行它：
```shell
>>> from spi import Lexer
>>> lexer = Lexer('BEGIN a := 2; END.')
>>> lexer.get_next_token()
Token(BEGIN, 'BEGIN')
>>> lexer.get_next_token()
Token(ID, 'a')
>>> lexer.get_next_token()
Token(ASSIGN, ':=')
>>> lexer.get_next_token()
Token(INTEGER, 2)
>>> lexer.get_next_token()
Token(SEMI, ';')
>>> lexer.get_next_token()
Token(END, 'END')
>>> lexer.get_next_token()
Token(DOT, '.')
>>> lexer.get_next_token()
Token(EOF, None)
>>>
```

再来看解析器的改动。

这里时解释器改动的总结：
![](/assets/images/Pasted%20image%2020241215102401.png)

1. 让我们从新增加的 AST 节点开始：
	- `Copound` AST 节点类表示复合语句。它的静态属性里就有一个 `Statement` 节点的列表。
	```Python
	class Compound(AST):
	    """表示一个 'BEGIN .. END' 代码块"""
	    def __init__(self):
	        self.children = []
	```
	- `Assign` AST 节点表示一个赋值语句。它的 `left` 变量用于存储 `Var` 节点，`right` 变量用于存储 `expr` 函数返回的节点：
	```Python
	class Assign(AST):
	    def __init__(self, AST: Var, op: Token, right: AST):
	        self.left = left
	        self.token = self.op = op
	        self.right = right
	```
	- `Var` AST 节点用于表示一个变量，但是请注意，`self.value` 存储的是变量名，而不是变量的值（因为是从 `Var` 类的角度来看的）
	```Python
	class Var(AST):
	    """Var 节点是由 ID 词元构造而成的"""
	    def __init__(self, token):
	        self.token = token
	        self.value = token.value
	```
	- `NoOp` AST 节点用于表示一个空语句，例如 `BEGIN END` 就是一个合法的复合语句，但是其中没有包含其他语句
	```Python
	class NoOp(AST):
	    pass
	```
2. 你应该还记得，上下文无关文法中的每一个规则，都对应着一个函数，因此这次我们需要增加7个新的函数。这些函数负责解析新语言结构，并返回新的 AST 节点，代码也很直接：
	```Python
	def program(self):
	     """program: compound_statement DOT"""
	     node = self.compound_statement()
	     self.eat(DOT)
	     return node
	 
	 def compound_statement(self):
	     """compund_statement: BEGIN statement_list END"""
	     self.eat(BEGIN)
	     nodes = self.statement_list()
	     self.eat(END)
	 
	     root = Compound()
	     for node in nodes:
	         root.children.append(node)
	 
	     return root
	 
	 def statement_list(self):
	     """
	     statement_list: statement
	                     | statement SEMI statement_list
	     """
	     node = self.statement()
	 
	     results = [node]
	     while self.current_token.type == SEMI:
	         self.eat(SEMI)
	         results.append(self.statement())
	 
	     if self.current_token.type == ID:
	         self.error()
	 
	     return results
	 
	 def statement(self):
	     """
	     statement: compound_statement
	              | assignment_statement
	              | empty
	     """
	     if self.current_token.type == BEGIN:
	         node = self.compound_statement()
	     elif self.current_type == ID:
	         node = self.assignment_statement()
	     else:
	         node = self.empty()
	     return node
	 
	 def variable(self):
	     """
	     variable: ID
	     """
	     node = Var(self.current_token)
	     self.eat(ID)
	     return node
	 
	 def empty(self):
	     """AN empty production"""
	     return NoOp()
		```
3. 我们同样需要更新 `factor` 函数来解析变量：
```Python
def factor(self):
    """
    factor: PLUS factor
          | MINUS factor
          | INTEGER
          | LPAREN expr RPAREN
          | variable
    """
    token = self.current_token
    if token.type == PLUS:
        self.eat(PLUS)
        node = UnaryOp(token, self.factor())
        return node
    ...
    else:
        node = self.variable()
        return node
```
4. 解析器的 `parse` 函数则需要更新，以 `Program` 规则开始解析：
```Python
def parse(self):
    node = self.program()
    if self.current_token.type != EOF:
        self.error()
    return node
```

我们整个 Pascal 代码的例子来测试一下：
```Pascal
BEGIN
    BEGIN
        number := 2;
        a := number;
        b := 10 * a + 10 * number / 4;
        c := a - - b
    END;
END.
```

先用 [genastdot.py](https://github.com/rspivak/lsbasi/blob/master/part9/python/genastdot.py) 把 Pascal 的代码可视化一下（简洁起见，`Varnode` 仅展示节点的变量名，`Assign` 节点则用 `:=` 展示）：
```Shell
$ python genastdot.py assignments.txt > ast.dot && dot /assets/images/-Tpng -o ast.png ast.dot
```
![](/assets/images/Pasted%20image%2020241215211940.png)

最后，是解释器类的改动：
![](/assets/images/Pasted%20image%2020241215212039.png)

为了能够访问新的 AST 节点，我们需要为解释器新增一些对应的访问函数。下面是需要新增的4个访问函数：
- `visit_Compound`
- `visit_Assign`
- `visit_Var`
- `visit_NoOp`

`Compound` 和 `NoOp` 访问函数都比较简单。`visit_Compound` 函数就是遍历它的子节点，而 `visit_NoOp` 函数则什么都不做。
```Python
def visit_Compound(self, node):
    for child in node.children:
        self.visit(child)

def visit_NoOp(self, node):
    pass
```

`Assign` 和 `Var` 访问函数就需要考虑多一些。

当你为一个变量赋值的时候，实际上是想在未来某个地方用到它的值，所以这才是 `visit_Assign` 函数真正需要解决的问题：
```Python
def visit_Assign(self, node):
    var_name = node.left.value
    self.GLOBAL_SCOPE[var_name] = self.visit(node.right)
```

这个函数把赋值语句的变量名作为哈希表的键，赋值语句的值作为哈希表的值，存储到 `GLOBAL_SCOPE` 这个全局变量中。当然，在编译原理中，它有个更高级的名字，叫符号表（Symbol Table）。

什么是符号表？
**符号表**是一个用于跟踪源码中各个符号的抽象数据类型（ADT，Abstract Data Type）。不过目前我们的符号表中仅有一种类型的符号，那就是变量。这里为了方便，我们是使用 Python 的字典来实现符号表的。

我只想说，在这篇文章里，符号表的实现方法相当的“临时抱拂脚”：没有用独立的类和特殊的函数，而仅仅使用 Python 的字典来实现，同时还负责变量的内存空间。在未来的文章里，我们会更加深入地讨论符号表，而同时我们也会移除所有的“权宜之计”。

先来看一下赋值语句 `a := 3` 的 AST，同时也看看在执行 `visit_Assign` 前后符号表的变化：
![](/assets/images/Pasted%20image%2020241215221550.png)

然后，我们再来看看含有变量 `a` 的赋值语句 `b := a + 7;` 的 AST：
![](/assets/images/Pasted%20image%2020241215221652.png)

如你所见，赋值语句的右边（`a + 7`）使用到了变量 `a`。因此在为 `a + 7` 求值之前，我们需要知道 `a` 的值是多少。而这，就是 `visit_Var` 函数要做的事儿：
```Python
def visit_Var(self, node):
    var_name = node.value
    val = self.GLOBAL_SCOPE.get(var_name)
    if val is None:
        raise NameError(repr(var_name))
    else:
        return val
```

当使用这个函数去访问上面 AST 图中的 `Var` 节点时，它会先获取到节点的值，也就是变量名。接着，再用变量名去访问符号表（`GLOBAL_SCOPE` 字典），来获取变量的值：
- 如果可以获取到，那么就直接返回变量的值；
- 如果获取不到就报一个 `NameError` 的错误
这是求值赋值语句 `b := a + 7` 前后符号表的变化：
![](/assets/images/Pasted%20image%2020241215222441.png)

以上，就是为了能让我们的编译器运行起来所作的所有改动。在主流程的结尾，我们简单的输出一下符号表 `GLOBAL_SCOPE` 到标准输出。

现在是时候在 Python 命令行中测试我们更新后的解释器了。在测试开始之前，确保你已经下载了最新的解释器源码，以及 [assignments.txt](https://github.com/rspivak/lsbasi/blob/master/part9/python/assignments.txt)：
启动 Python 命令行：
```Shell
$ python
>>> from spi import Lexer, Parser, Interpreter
>>> text = """\
... BEGIN
...
...     BEGIN
...         number := 2;
...         a := number;
...         b := 10 * a + 10 * number / 4;
...         c := a - - b
...     END;
...
...     x := 11;
... END.
... """
>>> lexer = Lexer(text)
>>> parser = Parser(lexer)
>>> interpreter = Interpreter(parser)
>>> interpreter.interpret()
>>> print(interpreter.GLOBAL_SCOPE)
{'a': 2, 'x': 11, 'c': 27, 'b': 25, 'number': 2}
```

然后在命令行把 Pascal 代码的源文件输入到我们的解释器：
```Shell
$ python spi.py assignments.txt
{'a': 2, 'x': 11, 'c': 27, 'b': 25, 'number': 2}
```

如果你还没有尝试运行过，现在就运行一下，看看解释器能否正常运行。

现在，让我们总结一下，为了把计算器扩展为 Pascal 解释器，这篇文章中我们做了啥：
1. 为上下文无关文法增加了新的规则
2. 为上下文无关文法的新规则增加对应的函数，如果有需要，更新已有的函数（说的就是你，`factor` ）
3. 为词法解析器增加了新的词元、以及对应的函数并修改了 `get_next_token` 函数
4. 为语法解析器增加了新的 AST 节点，用于处理新的语法结构
5. 为解释器增加新的访问函数
6. 增加一个字典，用于存储变量以及寻找它们

这部分我引入了一些“权宜之计”的手段，随着整个系列的推进，我们会把它们全部干掉：
![](/assets/images/Pasted%20image%2020241217195713.png)

1. `Program` 规则是不完整的，后续我们会增加新的元素来拓展它
2. Pascal 是一种静态类型语言，在使用一个变量之前，你得先定义它。但是你也看到了，目前我们不是这样搞的。
3. 目前没有类型检查。目前来说不是啥大问题，我只是想把它提出来。一旦解释器支持的类型多起来的时候，就得考虑输出一些错误信息了（比如往字符串上加整型）。
4. 目前的符号表是一个简单的字典，而且同时负责内存管理的功能。不过别担心，符号表是一个很重要的内容，后续我们会写几篇文章专门讨论它。而内存管理（运行时管理）就是其中的一个讨论点。
5. 在前面的简单计算其中，我们使用了一个斜杠字符 `/` 来表示整数除法。但是在 Pascal 中，必须使用 `div` 保留字来表示整数除法。
6. 除此之外还有一个问题没有提出来，我是希望你能在练习题中把它解决：在 Pascal 中所有的保留字和标识符都是不区分大小写的，但是我们的解释器是区分大小写的。

关系这么好了，送你一些练习题做做：
![](/assets/images/Pasted%20image%2020241217201137.png)

1. 不同于其他的编程语言，Pascal 的变量和保留字都是忽略大小写的。因此 `BEGIN`，`begin` 和 `BeGin` 都是合法的，且表示同一种意思。试着修改一下我们的解释器，使它可以忽略变量和保留字的大小写，并用下面的代码测试一下：
```Pascal
BEGIN

    BEGIN
        number := 2;
        a := NumBer;
        B := 10 * a + 10 * NUMBER / 4;
        c := a - - b
    end;

    x := 11;
END.
```
2. 我在“权益之计”那部分提到了，之前我们用斜杠字符 `/` 表示整数除法，但是在 Pascal 应该使用保留字 `div` 来表示。修改一下解释器，让它可以用 `div` 保留字来做整数除法。这样就能解决掉一个“权宜之计”了
3. 修改解释器，让它可以接受以下划线开始的变量名，如 `_num := 5`

今天的内容到此为止，真是酣畅淋漓的一天啊，回头见！