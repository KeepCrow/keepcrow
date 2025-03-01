---
layout: framework
sidebar: category-list
title:  "Let's Build a Simple Interpreter 第五章"
date:   2025-03-01 21:41:46 +0800
categories: Interpreter
---


### 前言
你是如何解决问题的呢？我是说难度和创建一个解释器或编译器差不多的问题。刚开始的时候，那种感觉就像是面对一团乱麻，而你的任务是把它们理顺并卷成一个完美的麻线球。

解决方法说起来也很简单，就是一个结一个结解开，自然就能卷出完美的麻线球。虽然有时候你会觉得，没法立马理解一些东西，但你得坚持下去。相信我，只要你一直坚持做这样的事儿，在某一刻，一切都会豁然开朗。（哎呀，如果每次遇到一个不理解的问题就存个10块钱，我很久以前就发财了估计😆）。

关于理解如何创建解释器或编译器，我所能给到你最好的建议就是：读文章，读代码，写代码，多写代码，甚至写重复的代码，直到这些材料和代码对你来说信手拈来，这时候再进行下一个主题。不必心急，慢下来好好理解一些最基本的思想。这个方法在当下看起来很慢，但是在未来你会从中受益。相信我😉

最终你一定会得到那个完美的麻线球。实际上，就算它不完美又怎样呢？总比什么都不做，或者仅仅是快速浏览一遍然后过几天忘掉要好得多

记住：一个一个结地解下去，然后不停写代码，写大量代码来练习你学到的知识：
![](/assets/images/Pasted%20image%2020240506211730.png)

今天我们会运用前面章节中学到的所有知识来编写一个能够运行完整四则运算的解释器，这个解释器能够处理如 $14 + 2 \times 3 — 6 \div 2$ 的数学表达式。

### 结合性与优先级
在深入之前，我们需要先了解一下运算符的**结合性**和**优先级**。

#### 结合性
从小学我们就知道，对于加法来说，$7 + 3 + 1$ 与 $(7 + 3) + 1$ 的结果一样，对于减法来说，$7 - 3 - 1$ 与 $(7 - 3) - 1$ 的结果一样。但是需要注意的是 $7 - 3 - 1$ 却与 $7 - (3 - 1)$ 不一样，前者结果为3，后者结果为5。我们把这种特性称为**左结合**。

那么什么是左结合呢？

假如说在一个表达式中，某个数的左右两边都有一个加号，比如 $7 + 3 + 1$ 中的 $3$。我们就需要统一一个标准或者说约定，这个 $3$ 应该有限与谁结合。是与左边的加号结合，先做 $7 + 3$ 的运算，还是与右边的加号结合，先做 $3 + 1$ 的运算？

当然，答案我们都知道，$3$ 会先与左边的加号结合。那么，我们就称运算符 `+` 是左结合的。同样的逻辑也适用于减号，乘号与除号。

事实上，在一般的算术表达式以及大多数编程语言中，加减乘除都是左结合的：
$7 + 3 + 1 = (7 + 3) + 1$
$7 - 3 - 1 = (7 - 3) - 1$
$8 \times 4 \times 2 = (8 \times 4) \times 2$
$8 \div 4 \div 2 = (8 \div 4) \div 2$

更进一步地说，同种类型的运算符也是左结合的。这里说得同种指的是：
- 加号与减号属于同一种，因为本质上它们都在进行加法运算
- 乘号与除号属于同一种，因为本质上它们都在进行乘法运算

因此你可以看到：
$7 + 3 - 1 = (7 + 3) - 1$
$7 - 3 + 1 = (7 - 3) + 1$
$8 \times 4 \div 2 = (8 \times 4) \div 2$
$8 \div 4 \times 2 = (8 \div 4) \times 2$

#### 优先级
那么如果数的两边含有不同的操作符呢？比如 $7 + 5 \times 2$ 中的 $5$，是先与加号结合还是先与乘号结合呢？用数学的表达方式就是 $7 + 5 \times = (7 + 5) \times 2$ 还是 $7 + 5 \times = 7 + (5 \times 2)$？

在这种情况下，结合律显然已经不适用了，因为结合律仅适用于两边是同种符号的情况。那么，我们就需要另一种约定来解决这种数字两边不是同一种类型符号的情况，这就是**优先级**。

我们说，乘法号会先于加法号于旁边的数结合，也就是说，它有**更高的优先级**。在四则运算中，我们知道，乘法于除法的优先级高于加法与减法。因此 $7 + 5 \times 2$ 等于 $7 + (5 \times 2)$，而 $7 - 8 \div 4$ 等于 $7 - (8 \div 4)$。

那如果说数的两边运算符优先级相同怎么办呢？这不是又回到了前面的结合律了吗，直接左结合走起！

### 优先级表
希望你不要觉得我讲这么多运算符的结合性和优先级是想把你无聊死……实际上，学习这些术语的好处是，当你有一个能够展示算术运算符的结合性与优先级的表格后，你就可以通过这个表格构建出一个算术表达式的[上下文无关文法](第四章%20上下文无关文法.md#上下文无关文法)。然后，你就可以通过上一章提到的[指导原则](第四章%20上下文无关文法.md#文法翻译为代码的指导原则)将文法翻译为代码。在此基础上写出的解释器不仅能处理前面就能处理的结合性，还能处理优先级。

下图就是我说的优先级表：
![](/assets/images/Pasted%20image%2020240511212640.png)

从表中可以看出，四种运算符都是左结合的，加号与减号属于同一优先级而乘号与除号属于同一优先级，并且乘号与除号相对于加号与减号具有更高的优先级。

### 构造上下文无关文法
下面是根据优先级表构建上下文无关文法的规则：
1. 每一个优先级定义一个非终结符。其生成式的主体部分应该包含其当前优先级的算术运算符以及下一优先级的非终结符
2. 为表达式的基本单元创建一个额外的非终结符 **factor**，在我们的场景中，基本单元是整数。

根据以上规则，如果你的表达式有 N 个优先级，那么你应该创建 N+1 个非终结符：每个优先级一个非终结符，基本单元一个非终结符。

现在我们把这个规则应用到我们的场景试一下。

根据规则1，我们需要定义两个非终结符：为优先级2创建非终结符 **expr**，为优先级1创建非终结符 **term**。再根据规则2，我们需要为表达式的基本单元，整数，创建一个非终结符 **factor**。

新的文法的**起始符**是 **expr**，并且 **expr** 生成式的主体将会包含优先级2中的运算符，在我们的场景中就是 + 和 -，同时还包含优先级1中的非终结符 **term**：
![](/assets/images/Pasted%20image%2020240518152124.png)

**term** 生成式的主体中会包含优先级1中的运算符，也就是 $\times$ 与 $\div$，同时还包含了运算符的基础单元非终结符 **factor**：
![](/assets/images/Pasted%20image%2020240518163555.png)

**factor** 非终结符如下所示：
![](/assets/images/Pasted%20image%2020240518163619.png)

上述的生成式你在前面应该已经见过了，但是这里我们是把上下文无关文法与优先级和结合性联系在了一起：
![](/assets/images/Pasted%20image%2020240518164055.png)

下图是对应的语法图：
![](/assets/images/Pasted%20image%2020240518164128.png)

上图中的每一个矩形框都是一次“函数调用”，如果你把 $7 + 5 \times 2$ 放入图中进行解析，从最上方的 **expr** 推导到最下方的 **factor**，那么你就不难发现，高优先级的运算符 $\times$ 与 $\div$ 比低优先级的运算符 $+$ 与 $-$ 更先进行了运算。

为了让运算符的优先级更清晰一些，我们来看一下按照上下文无关文法与语法图对表达式 $7 + 5 \times 2$ 的分解结果。实际上这只是用另一种方法来展示高优先级的运算符会先于低优先级运算符执行：
![](/assets/images/Pasted%20image%2020240518165841.png)

### 将文法转化为代码
好了，现在让我们按照上一章所学习的[指导原则](第四章%20上下文无关文法.md#文法翻译为代码的指导原则)把文法转化为代码，然后看看它是如何运行的。

在这之前，再看一眼我们之前设计出的文法：
![](/assets/images/Pasted%20image%2020240518170147.png)

下面是计算器的完整代码，它可以处理任意合法的四则运算表达式。

以下是相对于[第四章 上下文无关文法](第四章%20上下文无关文法.md#代码)，本章代码的主要改动点：
- `Lexer` 类将可以处理加减乘除四种运算符号
- 回忆一下第四章提到的指导原则，对于在文法中的每一个规则 **R**，我们都要将其转化为同名函数：`R()`。因此，`Interpreter` 类将包含以下三个与非终结符同名的函数：`expr`，`term` 以及 `factor`。

代码如下所示：
```python
# 词元类型
# EOF (end-of-file) 词元用于表示后续无内容可供分析
INTEGER, PLUS, MINUS, MUL, DIV, EOF = (
    'INTEGER', 'PLUS', 'MINUS', 'MUL', 'DIV', 'EOF'
)


class Token(object):
    def __init__(self, type, value):
        # 词元类型: INTEGER, PLUS, MINUS, MUL, DIV, 或 EOF
        self.type = type
        # 词元值: 非负整数, '+', '-', '*', '/', 或 None
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
            self.current_char = None  # 表示输入的结尾
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
        """factor : INTEGER"""
        token = self.current_token
        self.eat(INTEGER)
        return token.value

  

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
                result = result / self.factor()

        return result

    def expr(self):
        """算术表达式解析器/解释器.
  
        calc>  14 + 2 * 3 - 6 / 2
        17

        expr   : term ((PLUS | MINUS) term)*
        term   : factor ((MUL | DIV) factor)*
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

把上述代码保存到 `calc5.py` 文件中，或者直接从 [GitHub](https://github.com/rspivak/lsbasi/blob/master/part5/calc5.py) 上下载。然后，和往常一样，在自己的电脑上运行一下，看它能否正确处理你输入的表达式的优先级。

下面是我在自己电脑上运行的会话：
```shell
$ python calc5.py
calc> 3
3
calc> 2 + 7 * 4
30
calc> 7 - 8 / 4
5
calc> 14 + 2 * 3 - 6 / 2
17
```

### 练习
今天的练习来了：
![](/assets/images/Pasted%20image%2020240518181343.png)

- 不看本文的代码，按照本文的描述的那些规则，自己写一个解释器出来。记得写一些测试用例保证你的代码正确无误
- 拓展你的解释器，使其可以支持嵌套的括号，例如 $7 + 3 \times (10 \div (12 \div (3 + 1) - 1))$

### 概念自测
- 运算符的**左结合**是什么？
- 加号与减号是**左结合**还是**右结合**？乘号和除号呢？
- 加号与乘号谁的优先级更高？

### 结语
欢迎你来到文末，这很值得庆贺一下。下次我会带来新的文章，敬请期待，别忘记练习！