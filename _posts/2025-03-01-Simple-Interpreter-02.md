---
layout: framework
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第二章"
date:   2025-03-01 21:40:46 +0800
categories: Interpreter
---

《高效思维的五大要素》一书中有这样一个故事，内容为国际知名小号演奏家托尼-普洛格所举办的大师班，其学生多为小有成就的小号演奏家。

在课堂上，老师要求学生们先演奏复杂的旋律，他们表现得很好。然而，当老师要求学生们演奏非常基本、简单的音符时，他们却表现得和新手差不多。这之后，老师也演奏了同样的音符，但他的演奏并不生涩。

托尼说，只有掌握了简单音符的演奏方法，才能以更强的控制力去演奏更加复杂的乐曲。这道理老生常谈了，不积跬步无以至千里，没有良好的基础，就没法达到至高点。

同样的道理，放到编程中也适用。对于一些工具，掌握使用方法与框架固然重要，但了解其背后的原理也同样重要。永远不要忽视深入研究最基础的想法的重要性，正如拉尔夫·瓦尔多·爱默生所说：
*“只学习方法，你会被方法所束缚。但如果学习原则，你就能设计出自己的方法”*

大道理到此为止，让我们回到解释器与编译器的学习中来。

这一章中，我们将会看到新版本的计算器，它增加了以下功能：
- 能够处理输入字符串中出现在任意位置，任意数量的空格字符
- 能够处理多位数（目前仅支持一位数）
- 能够处理整数减法（目前仅支持加法）

以下是第二版计算器的源码：
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
        # 客户端字符串输入, 如"3 + 5", "12 - 5"，等等
        self.text = text
        # self.pos是self.text的下标
        self.pos = 0
        # 当前正在处理的词元
        self.current_token = None
        self.current_char = self.text[self.pos]

    def error(self):
        raise Exception('Error parsing input')

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

    def eat(self, token_type):
        # 将传入的词元类型于当前的词元类型进行比较，如果匹配成功，
        # 那么就“吃掉”当前的词元，然后把下一个词元赋值给self.current_token
        # 否则报错
        if self.current_token.type == token_type:
            self.current_token = self.get_next_token()
        else:
            self.error()

    def expr(self):
        """Parser / Interpreter

        expr -> INTEGER PLUS INTEGER
        expr -> INTEGER MINUS INTEGER
        """
        # 把获取到的第一个词元赋值给当前的词元
        self.current_token = self.get_next_token()

        # 我们期望当前的词元的类型应该是整数类型
        left = self.current_token
        self.eat(INTEGER)

        # 我们期望当前的词元的类型应该是加法或减法类型
        op = self.current_token
        if op.type == PLUS:
            self.eat(PLUS)
        else:
            self.eat(MINUS)

        # 我们期望当前的词元的类型应该是整数类型
        right = self.current_token
        self.eat(INTEGER)
        # 在上述流程结束后，self.current_token应该设置为EOF 词元

        # 此时，INTEGER PLUS INTEGER 或 INTEGER MINUS INTEGER 
        # 符号序列已成功找到，方法只需返回两个整数相加或相减的结果，
        # 从而有效地解释了客户端输入
        if op.type == PLUS:
            result = left.value + right.value
        else:
            result = left.value - right.value
        return result

def main():
    while True:
        try:
            # 想要在Python3下运行，把`raw_input`替换为`input`
            text = raw_input('calc> ')
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

将上述代码保存到 calc2.py 文件中，或直接从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part2/calc2.py) 下载。尝试运行一下，看看它是否满足上面提到的三个需求：处理空格，多位整数以及减法的能力。

下面是我在笔记本电脑上运行的一个实例：
```bash
$ python calc2.py
calc> 27 + 3
30
calc> 27 - 7
20
calc>
```

相对于[第一章](第一章.md)的版本，代码主要有以下改动：
1. 重构了 `get_next_token` 函数，将递增 `pos` 指针的逻辑提取为一个独立的函数；
2. 增加了两个函数：`skip_whitespace` 函数用于跳过空格字符，`integer` 函数用于处理输入中的多位整数；
3. 修改了 `expr` 函数，现在除了能识别加法短语 INTEGER -> PLUS -> INTEGER，它还能识别减法短语 INTEGER -> MINUS -> INTEGER，并能够在识别后对其进行解释，求出结果 

在[第一章](第一章.md)中，我们提到了两个重要概念：词元与词法分析器。今天，我们聊一下**词素**、**解析**与**解析器**。

首先来看看**词素（Lexeme）**，词素是由一系列字符组成的，下图是一些与词素的示例。可以将理解为类，而词素则是一个个具体的对象。
![图2.1](/assets/images/2.1.png)

还记得 `expr` 函数吗？就是解释算术表达式的地方。在解释表达式之前，它需要先识别其属于哪种短语，是加法还是减法。其实这也是 `expr` 函数的工作原理：从 `get_next_token` 函数获得流，找出想要的短语结构，然后解释识别出的短语，最终生成算术表达式的结果。

这个在流中寻找短语，或者说识别短语的过程，我们定义为**解析（Parsing）**，解释器或者编译器中负责这部分工作的模块成为**解析器（Parser）**。

好了，现在我们可以对 `expr` 函数重新定义了，它其实就是我们的简单解释器中负责解析和解释的部分，也就是解析器。其负责在中识别（解析）出短语，要么是 INTEGER -> PLUS -> INTEGER，要么是 INTEGER -> MINUS -> INTEGER。总之，在识别了以后，就对短语进行解释，也就是计算两个整数相加或相减的结果，然后返回。

这一章较为简单，但又学到了两个新的概念。现在，练习时间到！
![图2.2](/assets/images/2.2.png)

1. 扩展计算器，使之能够处理整数除法
2. 扩展计算器，使之能够处理整数乘法
3. 扩展计算器，使之能够处理两则运算式（不包括乘除法），例如 "9 - 5 + 3 + 11"。

检查一下概念的掌握程度：
1. 什么是词素？
2. 在流中寻找结构的过程是什么？换句话说，在流中识别某个短语的过程是什么？
3. 解释器（编译器）中进行解析的部分叫什么？

希望你喜欢今天的内容。在本系列的下一篇文章中，你将扩展你的计算器，以处理更复杂的算术表达式。敬请期待。