---
layout: post
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第六章"
date:   2025-03-01 21:41:46 +0800
categories: Interpreter
---


今天，就是我们为数学表达式的讨论收尾的吉日了😊。
我们会为它增加处理括号的功能，并且还能解释带有嵌套括号的数学表达式，例如 $7 + 3 \times (10 \div (12 \div (3 + 1) - 1))$。

首先，我们来修改一下[上下文无关文法](第四章%20上下文无关文法.md#上下文无关文法)，使其能够支持带有括号的数学表达式。还记得[第五章](第五章.md)的内容吗，`factor` 规则用于生成表达式的最基础的单元。在那篇文章中，基础单元只有一种可能，就是整数。今天我们将会增加另一种基础单元：带括号的表达式。

下图式更新后的文法：
![](/assets/images/Pasted%20image%2020240601214811.png)

生成式 `expr` 与 `term` 与[第五章](第五章.md)完全一样，只有 `factor` 生成式做了一点修改，增加了新的拓展选项。其中：
- 终结符 `LPAREN` 表示左括号
- 终结符 `RPAREN` 表示右括号
- 非终结符 `expr` 代表就是指生成式 `expr`

下图是更新后的 `factor` 的语法图，相对于以前的版本，多了一个选项：
![](/assets/images/Pasted%20image%2020240601215157.png)

由于 `expr` 与 `term` 的文法没有变，因此它们的语法图和[第五章](第五章.md)相同：
![](/assets/images/Pasted%20image%2020240705220749.png)

我们的新文法有一个有趣的特点：它是递归的。如果你尝试推导出表达式 $2 \times (7 + 3)$，那么你将会从起始符 `expr` 开始推导，经过一系列的推导，又回到 `expr` 以推导 $7 + 3$

现在让我们根据文法来分解表达式 $2 \times (7 + 3)$，用更直观的方式来看一下：
![](/assets/images/Pasted%20image%2020240705221244.png)

> [!note] 顺带说一句
> 如果你想要复习一下递归，可以看看 Daniel P. Friedman 和 Matthias Felleisen 的 The Little Schemer 一书，讲得真的很棒

好了，让我们开始把新的文法转化为代码吧。

下面是代码的主要更新点：
- `Lexer` 类能够多识别两个词元：左括号的词元 `LPAREN` 与右括号的词元 `RPAREN`
- `Interpreter` 类的 `factor` 函数稍微更新了一下，现在它能解析带括号的表达式了

下面就是完整的代码。只要表达式中仅有整数运算，且符合四则运算标准，不管表达式有多长，括号嵌入多少层，它都能正确计算出结果：
```python
# 词元类型
# EOF (end-of-file) 词元用于表示后续无内容可供分析
INTEGER, PLUS, MINUS, MUL, DIV, LPAREN, RPAREN, EOF = (
    'INTEGER', 'PLUS', 'MINUS', 'MUL', 'DIV', '(', ')', 'EOF'
)


class Token(object):
    def __init__(self, type, value):
        self.type = type
        self.value = value
        
    def __str__(self):
        """词元对象的字符串描述
        例:
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
        # 命令行字符串输入, e.g. "3 * 5", "12 / 3 * 4", 等
        self.text = text
        # self.pos 是 self.text 的下标
        self.pos = 0
        self.current_char = self.text[self.pos]

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
        """factor : INTEGER | LPAREN expr RPAREN"""
        token = self.current_token
        if token.type == INTEGER:
            self.eat(INTEGER)
            return token.value
        elif token.type == LPAREN:
            self.eat(LPAREN)
            result = self.expr()
            self.eat(RPAREN)
            return result

    def term(self):
        """term : factor ((MUL | DIV) factor)*"""
        result = self.factor()

        while self.current_token.type in (MUL, DIV):
            token = self.current_token
            
            if token.type == MUL:
                self.eat(MUL)
                result = result * self.factor()
            elif token.type == DIV:
                self.eat(DIV)
                divisor = self.factor()
                if divisor == 0:
                    self.error('语法错误：除数不能为0')
                result = result / divisor

        return result

    def expr(self):
        """算术表达式解析器/解释器.
  
        calc>  14 + 2 * 3 - 6 / 2
        17

        expr   : term ((PLUS | MINUS) term)*
        term   : factor ((MUL | DIV) factor)*
        factor : INTEGER
        """
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

        lexer = Lexer(text)
        interpreter = Interpreter(lexer)
        result = interpreter.expr()
        print(result)
        

if __name__ == '__main__':
    main()
```

把上述代码保存到 `calc6.py` 文件中，或者直接从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part6/calc6.py) 上下载，然后在自己的电脑上跑一下，看看是否和你预期的结果一致。下面是我在自己的电脑上跑的一个会话：
```shell
$ python calc6.py
calc> 3
3
calc> 2 + 7 * 4
30
calc> 7 - 8 / 4
5
calc> 14 + 2 * 3 - 6 / 2
17
calc> 7 + 3 * (10 / (12 / (3 + 1) - 1))
22
calc> 7 + 3 * (10 / (12 / (3 + 1) - 1)) / (2 + 3) - 5 - 3 + (8)
10
calc> 7 + (((3 + 2)))
12
```


下面是今天的练习：
1. 按照本文所述编写自己的解释器，重复是学习任何技能的最佳方法

恭喜！读到这里，相信你已经知道了如何创建一个简单的递归下降的解析器/解释器，能够处理非常复杂的数学表达式。实际上，如果你刚刚完成了练习，那么你不仅知道了，还掌握了如何写这样的解释器

下一篇文章，我们将更深入地讨论递归下降解释器。同时还会学到一种在编译原理中频繁使用的数据结构，后续的学习都会在这个数据结构上展开

在那之前，请继续努力提升你的解释器知识，最重要的是：享受学习的过程
