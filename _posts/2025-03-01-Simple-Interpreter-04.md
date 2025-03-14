---
layout: post
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第四章"
date:   2025-03-01 21:40:46 +0800
categories: Interpreter
---


### 前言
在阅读前几章时，你是沉浸在知识带来的短暂刺激中，还是真正付诸实践了呢？我期望你选择了后者，因为仅仅追求知识的快感与浏览短视频无异，缺乏深层次的参与和体验。

还记得孔夫子[^1]怎么说的吗？

> 不闻不若闻之，闻之不若见之，见之不若知之，知之不若行之；学至于行之而止矣

![](/assets/images/4.1.png)

![](/assets/images/4.2.png)

![](/assets/images/4.3.png)


在前几章，我们一起探索了如何解析（识别）和解释仅包含加减法的数学表达式，例如 $7 - 3 + 2 - 1$。我们还学习了关于语法图的知识，以及如何用语法图来描绘一种编程语言的结构。

### 上下文无关文法
今天，我们将进一步学习如何解析和解释仅包含乘除法的数学表达式，比如 $7 \times 4 \div 2 \times 3$。本文中，我们讨论的除法是整数除法，也就是说 $9 \div 4 = 2$，而不是 $9 \div 4 = 2.5$。

除此之外，我们还会简单介绍一些**上下文无关文法[^2]（CFG，Context-Free Grammars）**，或者叫**巴科斯范式[^3]（BNF，Backus-Naur Form）** 的内容。这是一种广泛使用的工具，专门用于描述编程语言的语法规则。出于本系列文章的目的，我不会采用纯 [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) 语法，而是选择一种类似于 [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) 的语法。

#### 为什么
下面是一些使用文法（上下文无关文法的简写）的优势：
- 文法能以简洁的方式来描述编程语言的语法，不同于语法图，文法的信息更加紧凑（译者注：可不得紧凑吗，用数学公式表达的）。在后续的文章中，我们会更频繁地接触文法
- 文法本身就是一份优秀的文档
- 对于初学者来说，从文法开始手动编写解析器是一个很好的切入点，一般情况下，你可以通过一系列简单的规则把文法转化为代码
- 如果你比较懒，有一些工具，叫做*解析器生成器*，可以根据文法生成你需要的解析器，后面的文章里我会介绍到这些工具

#### 什么样
现在，我们来看一下文法是什么样的。

下图是一个文法，可以描述类似于 "$7 \times 4 \div 2 \times 3$" 的数学表达式：
![](/assets/images/4.4.png)

文法由一系列的**规则**构成，也有叫**产生式**的，下图中的文法由两个规则构成：
![](/assets/images/4.5.png)

规则由三部分构成：头，冒号和正文。这其中头也可以叫左部，正文也可以叫右部。产生式头，或者说左部，是一个非终结符；而产生式正文，或者说右部，由若干终结符或非终结符构成。如下图所示：
![](/assets/images/4.6.png)
所谓**终结符**，就是类似上图中 MUL，DIV 与 INTEGER 一类的词元，这类词元的特点是无法再继续往下推导。
与此相对的，**非终结符**就是类似上图中 expr，factor 一类的词元，这类词元的特点是还能继续往下推导。
![](/assets/images/4.16.png)

第一个规则的左部叫做**起始符**，它在一个文法中有且仅有一个。在我们的例子中，起始符就是 expr：
![](/assets/images/4.7.png)
你可以把 expr 规则理解为：一个 expr 可以是一个 factor，后面跟着一个 MUL 或者 DIV，再跟一个 factor；而这个 factor 后面又可以跟一个 MUL 或者 DIV，再跟一个 factor，以此类推。

什么是 factor 呢？翻译过来就是因数的意思，只不过在本文的场景中，所有的因数都是整数，这才有了第二个规则 `factor: INTEGER`

现在让我们快速过一遍文法中用到的符号，以及它们的含义。
- `|`，或，表示符号两边都可以选择，所以 MUL | DIV 就表示 MUL 或 DIV
- `(...)`，括号括住的部分，表示里面的部分是一组，一般后面会跟上其他符号表示更多的意思
- `(...)*`，这里的 `*` 就表示括号里这一组可以重复出现，最少 0 次

如果你使用过正则表达式，那么对上面这些符号应该非常熟悉。

一下子说了一堆概念，如果你感觉到被它们冲昏了头脑，不妨站起来走走，看看窗外的风景，再回来从文法的概念看起。相信我，你会好很多。

#### 怎么用
文法通过定义语言中的句子应该长什么样来定义语言本身。拿我们的算术表达式来说，你可以通过文法*推导*出任意一个合法的数学表达式：
1. 首先从起始符 expr 开始；
2. 将非终结符用其对应的规则的正文替换掉；
3. 如果句子中还有非终结符，重复步骤 2；
4. 如果句子中没有非终结符，结束推导。
而所有通过以上步骤推导出的句子组合在一起，就是由文法所定义的那个*语言*。

如果有这么一个表达式，无法通过文法推导出来，比如 $3 + 2$，那么就证明文法不支持这个表达式。或者说，文法所定义的语言中没有这样的句子。与此对应的，如果你用解析器去解析这个句子，那么解析器一定会报错。

说到这里，我认为有必要举几个例子来解释一下具体的推导过程。下图是从文法推导出算术表达式 $3$ 的过程：
![](/assets/images/4.8.png)

下图是从文法推导出算术表达式 $3 \times 7$ 的过程：
![](/assets/images/4.9.png)

下图是从文法推导出算术表达式 $3 \times 7 \div 2$ 的过程：
![](/assets/images/4.10.png)

哇，这部分的理论真是有趣，是不是！

当我第一次读到上下文无关文法以及相关名词的时候，当时的感觉就像这样：
![](/assets/images/4.11.png)

至少绝对不是像下面这样：
![](/assets/images/4.12.png)

我花了一些时间才适应这种符号，它的工作原理以及它与解析器和词法分析器之间的关系。但我要说的是，从长远来看，投入时间去学习它是绝对值得的。因为这个概念几乎遍布每一个编译器，解释器以及相关文献中，你在未来的某个时刻一定会遇到它。所以，早学晚学都是学，不如现在就学👀。

### 文法翻译为代码的指导原则
那么，是时候把文法变成代码了！

下面列出了一些指导原则，用于将文法转化为代码。只要你遵守这些原则，就可以经文法正确地翻译成一个能够正常运行的解析器。
1. 对于在文法中定义的任意规则 **R**，将其翻译成同名的函数。而在其他规则中对于 R 的引用则翻译成对其同名函数的调用 `R()`。函数体则按照规则正文进行翻译，也是遵循同样的指导原则
2. 对于类似 **(alt1 | alt2 | alt3)** 一类的可选项，翻译成 `if-elif-else` 语句
3. 对于类似 **(...)\*** 的重复组，翻译成 `while` 循环，可以执行 0 次或多次那种
4. 对于任意一个词元 **T** 的引用，都翻译为对 `eat` 函数的调用：`eat(T)`。`eat` 函数的工作原理就是当入参 T 的类型与当前正在处理的词元类型一致时，“消化”掉当前词元，并从词法解析器中获取下一个词元将其赋值给 `current_token`，也就是当前正在处理的词元

如果用图来展示的话，应该如下图所示：
![](/assets/images/4.13.png)

理论介绍完毕，下面就来着手转换吧！🎨

我们的文法中有两个规则：一个是 `expr` 规则，一个是 `factor` 规则。我们先从简单的 `facotr` 规则（产生式）开始。按照指导原则，一需要创建一个函数，名为 `factor`（原则 1）。函数中仅调用一个 `eat` 函数用于处理 INTEGER 类型的词元（原则 4）。
```python
def factor(self):
	self.eat(INTEGER)
```

小菜一碟，下一位！

`expr` 规则翻译为名为 `expr` 函数（原则 1）。规则正文中，对于 `factor` 的引用翻译为对函数 `factor` 的调用。可选组 `(...)*` 翻译为 `while` 循环，而其中的 `(MUL | DIV)` 则翻译为其循环体中的 `if-elif-else` 语句。把上述内容组合在一起，我们就得到了完整的 `expr` 函数：
```python
def expr(self):
	self.factor()
	
	while self.current_token.type in (MUL, DIV):
		token = self.current_token
		if token.type == MUL:
			self.eat(MUL)
			self.factor()
		elif token.type == DIV:
			self.eat(DIV)
			self.factor()
```

请务必投入足够的时间来深入理解上述的转换流程，因为掌握这些知识对于后续的学习和应用将起到关键作用。

为了方便，我把上面的代码放到了 `parser.py` 文件中，这个文件还包含了词法解析器与语法解析器，加上上面的解释代码，刚好组成一个解释器。你可以直接从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part4/parser.py) 上下载下来，然后玩一下。

代码跑起来后有交互提示，当你输入表达时候就可以看到其是否符合语法。不符合会报错，符合就会输出正确的计算结果。

下面是我在自己的电脑上跑的结果：
```shell
$ python parser.py
calc> 3
calc> 3 * 7
calc> 3 * 7 / 2
calc> 3 *
Traceback (most recent call last):
  File "parser.py", line 155, in <module>
    main()
  File "parser.py", line 151, in main
    parser.parse()
  File "parser.py", line 136, in parse
    self.expr()
  File "parser.py", line 130, in expr
    self.factor()
  File "parser.py", line 114, in factor
    self.eat(INTEGER)
  File "parser.py", line 107, in eat
    self.error()
  File "parser.py", line 97, in error
    raise Exception('Invalid syntax')
Exception: Invalid syntax
```

自己试一下吧！

我又要再说一遍语法图了，下面的语法图实际上就对应着 `expr` 规则：
![](/assets/images/4.14.png)

### 代码
那么，万事俱备，是时候开始编写新的算术表达式解释器代码了。一下是计算器的源码，它能够处理合法的算术表达式。这里所说的合法，指的是符合上述文法的表达式，即仅包含乘除法。我把词法解析器提取为了一个独立的类 `Lexer` ，并修改 `Intepreter` 类，将 `Lexer` 类的对象作为一个参数传入其构造函数：
```python
# 词元类型
# EOF (end-of-file) 词元用于表示后续没有可分析的内容
INTEGER, MUL, DIV, EOF = 'INTEGER', 'MUL', 'DIV', 'EOF'


class Token(object):
    def __init__(self, type, value):
        # 词元类型: INTEGER, MUL, DIV, 或 EOF
        self.type = type
        # 词元值: 非负整数, '*', '/', 或 None
        self.value = value

    def __str__(self):
        """词元对象的字符串描述

        例:
            Token(INTEGER, 3)
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
        # 命令行字符串输入, e.g. "3 * 5", "12 / 3 * 4", 等
        self.text = text
        # self.pos 是 self.text 的下标
        self.pos = 0
        self.current_char = self.text[self.pos]

    def error(self):
        raise Exception('非法字符')

    def advance(self):
        """前移 `pos` 指针，并设置 `current_char` 变量."""
        self.pos += 1
        if self.pos > len(self.text) - 1:
            self.current_char = None  # 表示输入的结尾
        else:
            self.current_char = self.text[self.pos]

    def skip_whitespace(self):
        while self.current_char is not None and self.current_char.isspace():
            self.advance()

    def integer(self):
        """返回一个多位数整数"""
        result = ''
        while self.current_char is not None and self.current_char.isdigit():
            result += self.current_char
            self.advance()
        return int(result)

	def get_next_token(self):
        """词法分析器 (也叫 scanner 或 tokenizer)
		函数负责将输入分解成词元，每次调用返回一个词元
        """
        while self.current_char is not None:

            if self.current_char.isspace():
                self.skip_whitespace()
                continue

            if self.current_char.isdigit():
                return Token(INTEGER, self.integer())

            if self.current_char == '*':
                self.advance()
                return Token(MUL, '*')

            if self.current_char == '/':
                self.advance()
                return Token(DIV, '/')

            self.error()

        return Token(EOF, None)


class Interpreter(object):
    def __init__(self, lexer):
        self.lexer = lexer
        # 把从输入解析出的第一个词元赋值给当前词元
        self.current_token = self.lexer.get_next_token()

    def error(self):
        raise Exception('语法错误')

    def eat(self, token_type):
        # 判断当前词元的类型是否符合预期：如果符合就“吃掉”当前词元，
        # 然后把下一个词元赋值给 self.current_token；否则报错
        if self.current_token.type == token_type:
            self.current_token = self.lexer.get_next_token()
        else:
            self.error()

    def factor(self):
        """返回一个 INTEGER 类型的词元的值

        factor : INTEGER
        """
        token = self.current_token
        self.eat(INTEGER)
        return token.value

    def expr(self):
        """算术表达式解析器/解释器.

        expr   : factor ((MUL | DIV) factor)*
        factor : INTEGER
        """
        result = self.factor()

        while self.current_token.type in (MUL, DIV):
            token = self.current_token
            if token.type == MUL:
                self.eat(MUL)
                result = result * self.factor()
            elif token.type == DIV:
                self.eat(DIV)
                result = result / self.factor()

        return result


def main():
    while True:
        try:
            text = input('calc> ')
        except EOFError:
            break
        if not text:
            continue
        lexer = Lexer(text)
        interpreter = Interpreter(lexer)
        result = interpreter.expr()
        print(result)


if __name__ == '__main__':
    main()
```

把上述代码保存到 `calc4.py` 中，或者直接从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part4/calc4.py) 下载。按照惯例，你可以自己先试试代码是否能够正常运行。

下面是我在自己电脑上运行的结果：
```shell
$ python calc4.py
calc> 7 * 4 / 2
14
calc> 7 * 4 / 2 * 3
42
calc> 10 * 4  * 2 * 3 / 8
30
```

### 练习
我知道你已经等不及这部分了😁，今天的练习来了：
![](/assets/images/4.15.png)
- 自己写一个支持完整四则运算的上下文无关文法。通过这个文法可以推导出类似 $2 + 7 \times 4$，$7 - 8 \div 4$，$14 + 2 \times 3 - 6 \div 2$，一类的表达式。
- 根据你编写的上下文无关文法，写一个能够支持完整四则运算的解释器。你的解释器应该可以处理类似 $2 + 7 \times 4$，$7 - 8 \div 4$，$14 + 2 \times 3 - 6 \div 2$，一类的表达式。
- 如果你完成了上述练习，记得放松一下😉

### 自测

带着今天学到的的上下文无关文法回答下面这些问题，可以参考下图：
![](/assets/images/4.17.png)

- 什么是上下文无关文法？
- 上图的文法中有几个规则（产生式）？
- 什么是终结符？找出上图中的所有终结符
- 什么是非终结符？找出上图中的所有非终结符
- 什么是规则的头？找出上图中的所有头（左部）
- 什么是规则的正文？找出上图中的所有正文（右部）
- 什么是文法的起始符？

你真的读完了！这篇文章塞了太多理论知识了，能把它读完的你真的很了不起🥇。

不久后我会带着新的文章回来，尽情期待，别忘了做练习，那玩意儿真的很有用。

### 注释

[^1]: 这句话实际上是荀子说的，只不过以讹传讹了
[^2]: 一种可以用推导的方式来产生语言的文法，后面会详细介绍
[^3]: 与上下文无关文法是同一种东西，只不过由不同的人（John Backus）提出