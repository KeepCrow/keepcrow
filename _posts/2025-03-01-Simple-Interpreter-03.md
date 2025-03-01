---
layout: framework
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第三章"
date:   2025-03-01 21:40:46 +0800
categories: Interpreter
---

今早醒来，我心血来潮问我自己："为什么我们会觉得新技能这么难学呢？"

我认为不仅仅是因为我们需要付出艰苦地努力，还有一个原因是我们把大量时间花在了学习理论知识，却没有通过练习把知识转化为技能。以游泳为例，即使你花费几百个小时去看游泳的专业书籍，观看游泳教学视频，甚至和专业游泳教练交流，第一次下水，你还是会跟个石头一样沉下去😏

老话说得好，*经验大似学问*。不管你觉得自己对一门学科有多了解，都应该动手去做，把知识转化为真正的技能。为了让你达成这一目标，我在[第一章](第一章.md)和[第二章](第二章.md)中加入了练习。并且我保证，在未来的文章中，你会看到更多的练习🙂

好了，现在正式开始今天的内容。

到目前为止，你已经学会了如何解释两个整数相加或相减的算术表达式，如 "7 + 3 "或 "12 - 9"。今天，我将介绍如何解析（识别）和解释包含任意数量加减运算符的算术表达式，例如 "7 - 3 + 2 - 1"。
本文中的算术表达式可以用下面的语法图表示：

到目前为止，我们已经学习了如何解释如"7 + 3"和"12 + 9"一类的整数加减法表达式。今天，我们将学习如何解释四则运算表达式……中的加减法表达式，比如"7 - 3 + 2 - 1"（乘除法改日再说）。

理论上来说，四则运算表达式中的项出现的次数应该是不限次的，加减符号也是如此。如果我们用流程图来表示的话，大概如图所示：
![图3.1](/assets/images/3.1.png)

你可以随便写一个表达式（不要有乘除法和括号谢谢），它一定可以用上面这个图表达：
1. 第一个肯定是项，也就是图中的 term；
2. 后面可能没有其他的运算，例如表达式"3"就只有一个项，也就是图中最上面的路线
3. 或者后面跟着其他运算，也就是会有运算符号和另一个项
4. 再然后后面可能没有其他运算，例如表达式"3 + 4"就只有两个项一个运算符
5. 当然也可能还有其他的运算，例如表达式"7 - 3 + 2 — 1"，就是走图中最下面的路线

上面这种流程图的专业名词叫**语法图**，所谓语法图，就是能够表达编程语言语法规则的图。

为什么要介绍这么个东西？主要有以下三个原因：
1. 语法图能以图形的方式准确直观地告诉读者，我们的编程语言支持哪些语法，不支持哪些语法。比如说，目前我们的解释器（计算器）仅能识别包含加减法的数学表达式。而别人看到这个语法图就知道，如果在项后面跟上一个乘法号，或者在表达式的开头给个加法号，我们的解释器都是无法进行解释的。
2. 语法图容易理解：你只要跟着图中的箭头走就行，有分叉的地方表示有多种可能，有回头的地方就表示有循环。
3. 最重要的原因是，语法图是编写解释器的重要一步。有了语法图，你几乎可以无脑地把它映射到你的代码中，完成语义解析的工作。

前面我们介绍到，在词元流中识别短语的过程叫做**解析**。在解释器中，负责这部分工作的模块叫做**解析器（Parser）**。实际上解析又被叫做**语法分析**，因为它做的事就是在分析语法是否符合预期。那么同理，解析器也可以叫做**语法分析器**。

因为每个编程语言的算术表达式的语法都是类似的，所以你可以暂时在 Python 中"测试"一下我们的语法图。在命令行里面启动 Python，然后自己写几个符合我们语法图的数学表达式：
```shell
>>> 3
3
>>> 3 + 4
7
>>> 7 - 3 + 2 - 1
5
```

嗯，Python 同学算得不错😎。

算术式"3+"不是一个合法的数学表达式，因为按照语法图（图3.1）的语法来看，加法号后面应该紧跟着一个项，否则就是语法错误。在 Ptyon 中测试一下，你也可以看到关于语法错误的提示：
```shell
>>> 3 +
  File "<stdin>", line 1
    3 +
      ^
SyntaxError: invalid syntax
```

用 Python 来进行测试固然很爽，但毕竟不是我们的目的。接下来，我们就把语法图转化为代码，然后用我们自己的解释器来进行测试，期待一下吧！😼

在前面的文章里（[第一章](第一章.md)与[第二章](第二章.md)），`expr` 函数既是解析器也是解释器，因为它要先从词元流中解析出特定的语法（解析器），然后再根据语法解释出最终的结果（解释器）。

下面的代码片段展示了语法图对应的代码。为了能和语法图一一对应，我们用 `term` 函数来负责 term 矩形框中的工作：解析整数。而 `expr` 函数则仅负责执行语法图的主流程：
```python
def term(self):
    self.eat(INTEGER)

def expr(self):
    # 将当前词元设置为从输入解析出的第一个词元
    self.current_token = self.get_next_token()

    self.term()
    while self.current_token.type in (PLUS, MINUS):
        token = self.current_token
        if token.type == PLUS:
            self.eat(PLUS)
            self.term()
        elif token.type == MINUS:
            self.eat(MINUS)
            self.term()
```

可以看到，`expr` 函数先调用了 `term` 函数，然后用一个 `while` 循环来做后续的运算，这个 `while` 循环会执行0到多次。在循环体中，解析器基于词元的值（加法号或减法号）做出不同的动作。你可以花点时间确认一下，代码是否真的实现了语法图表示的语法。

不过上述代码作为解析器并没有解释的功能，如果输入的语法正确，它啥也不干，如果语法不正确，它会报错。所以，我们可以修改一下 `expr` 函数，增加解释的代码，让它具有解释器的功能：
```python
def term(self):
    """返回一个 INTEGER 词元的值"""
    token = self.current_token
    self.eat(INTEGER)
    return token.value

def expr(self):
    """解析器 / 解释器 """
    # 将当前词元设置为从输入解析出的第一个词元
    self.current_token = self.get_next_token()

    result = self.term()
    while self.current_token.type in (PLUS, MINUS):
        token = self.current_token
        if token.type == PLUS:
            self.eat(PLUS)
            result = result + self.term()
        elif token.type == MINUS:
            self.eat(MINUS)
            result = result - self.term()

    return result
```

考虑到解释器要计算表达式的值，所以修改了 `term`，使其可以返回一个整型；然后修改了一下 `expr` 函数，使其可以在合适的地方进行加减法运算，并在函数结束的时候返回结果。虽然代码看起来十分简单易懂，但我还是建议你花点时间仔细看看💖。

好了，接下来让我们看一下完整代码。

下面是新版本的计算器源码，不管算术表达式有多长，只要合法，它都能正确解释：
```python
# 词元类型
# EOF (end-of-file)表示词法分析已经结束
INTEGER, PLUS, MINUS, EOF = 'INTEGER', 'PLUS', 'MINUS', 'EOF'

class Token(object):
    def __init__(self, type, value):
        # 词元类型: INTEGER, PLUS, or EOF
        self.type = type
        # 词元取值: 非负整数, '+', '-', or None
        self.value = value

    def __str__(self):
        """词元对象的字符串表达.
        例:
            Token(INTEGER, 3)
            Token(PLUS, '+')
        """
        return 'Token({type}, {value})'.format(
            type=self.type,
            value=repr(self.value)
        )

    def __repr__(self):
        return self.__str__()

class Interpreter(object):
    def __init__(self, text):
        # 命令行输入的字符串，例如："3 + 5", "12 - 5 + 3"
        self.text = text
        # self.pos是self.text的下标
        self.pos = 0
        # 当前正在处理的词元
        self.current_token = None
        self.current_char = self.text[self.pos]

    ##########################################################
    # 词法解析器代码                                           #
    ##########################################################
    def error(self):
        raise Exception('Invalid syntax')

    def advance(self):
        """'pos'指针前移，并为'current_char'赋值"""
        self.pos += 1
        if self.pos > len(self.text) - 1:
            self.current_char = None  # Indicates end of input
        else:
            self.current_char = self.text[self.pos]

    def skip_whitespace(self):
        while self.current_char is not None and self.current_char.isspace():
            self.advance()

    def integer(self):
        """返回一个从输入中获取的多位整数"""
        result = ''
        while self.current_char is not None and self.current_char.isdigit():
            result += self.current_char
            self.advance()
        return int(result)

    def get_next_token(self):
        """词法解析器（别名，tokenizer）
        该函数负责将字符串分解成一个个词元
        每次调用返回一个词元
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

            self.error()

        return Token(EOF, None)

    ##########################################################
    # 解析器/解释器代码                                        #
    ##########################################################
    def eat(self, token_type):
        # 将传入的词元类型于当前的词元类型进行比较，如果匹配成功，
        # 那么就"吃掉"当前的词元，然后把下一个词元赋值给self.current_token
        # 否则报错
        if self.current_token.type == token_type:
            self.current_token = self.get_next_token()
        else:
            self.error()

    def term(self):
        """返回一个 INTEGER 词元的值."""
        token = self.current_token
        self.eat(INTEGER)
        return token.value

    def expr(self):
        """算术表达式解析器 / 解释器."""
	    # 将当前词元设置为从输入解析出的第一个词元
        self.current_token = self.get_next_token()

        result = self.term()
        while self.current_token.type in (PLUS, MINUS):
            token = self.current_token
            if token.type == PLUS:
                self.eat(PLUS)
                result = result + self.term()
            elif token.type == MINUS:
                self.eat(MINUS)
                result = result - self.term()

        return result

def main():
    while True:
        try:
            text = input('calc> ')
        except EOFError:
            break
        if not text:
            continue
        interpreter = Interpreter(text)
        result = interpreter.expr()
        print(result)

if __name__ == '__main__':
    main()
```

把上述代码保存到`cal3.py`文件中，或者直接从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part3/calc3.py) 中下载。自己试一下，看看它能不能处理符合前面语法图的表达式。

下面是我在自己的电脑上进行的会话：
```shell
$ python calc3.py
calc> 3
3
calc> 7 - 4
3
calc> 10 + 5
15
calc> 7 - 3 + 2 - 1
5
calc> 10 + 1 + 2 - 3 + 4 + 6 - 15
5
calc> 3 +
Traceback (most recent call last):
  File "calc3.py", line 147, in <module>
    main()
  File "calc3.py", line 142, in main
    result = interpreter.expr()
  File "calc3.py", line 123, in expr
    result = result + self.term()
  File "calc3.py", line 110, in term
    self.eat(INTEGER)
  File "calc3.py", line 105, in eat
    self.error()
  File "calc3.py", line 45, in error
    raise Exception('Invalid syntax')
Exception: Invalid syntax
```

还记得我在开头的时候提到的练习吗？我遵守了我的承诺，你也会吗（第一章结束）😋？
![图3.2](/assets/images/3.2.png)

- 绘制一个仅识别乘除法的语法图，例如"7 * 4 / 2 * 3"，我真的强烈建议你动手在纸上画一个出来。
- 修改源代码，使其符合你绘制的语法图
- 从头编写一个可以处理加减法数学表达式的解释器，不看任何示例，自己编写，就用你用着最舒服的语言。在做的过程中，时刻想着那些关键模块：
	- 词法解析器：将输入的字符串解析为词元流
	- 解析器：从词法解析器输出的词元流中识别出固定格式的语句
	- 解释器：根据解析器识别的短语生成最终结果
	然后把这些模块组合起来。
相信我，花些时间把你获取到的知识转化为一个可用的数学表达式解释器会很有用。

检查一下自己的理解：
1. 什么是语法图
2. 什么是语法分析
3. 什么是语法分析器

文章就要结束了，感谢你能坚持到这里，不要忘记做练习哈😁。我很快就会带着新的文章归来，敬请期待。

下面的书单可能会对你学习解释器与编译器有帮助：
1. [Language Implementation Patterns: Create Your Own Domain-Specific and General Programming Languages (Pragmatic Programmers)](http://www.amazon.com/gp/product/193435645X/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=193435645X&linkCode=as2&tag=russblo0b-20&linkId=MP4DCXDV6DJMEJBL)
2. [Writing Compilers and Interpreters: A Software Engineering Approach](http://www.amazon.com/gp/product/0470177071/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0470177071&linkCode=as2&tag=russblo0b-20&linkId=UCLGQTPIYSWYKRRM)
3. [Modern Compiler Implementation in Java](http://www.amazon.com/gp/product/052182060X/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=052182060X&linkCode=as2&tag=russblo0b-20&linkId=ZSKKZMV7YWR22NMW)
4. [Modern Compiler Design](http://www.amazon.com/gp/product/1461446988/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1461446988&linkCode=as2&tag=russblo0b-20&linkId=PAXWJP5WCPZ7RKRD)
5. [Compilers: Principles, Techniques, and Tools (2nd Edition)](http://www.amazon.com/gp/product/0321486811/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321486811&linkCode=as2&tag=russblo0b-20&linkId=GOEGDQG4HIHU56FQ)