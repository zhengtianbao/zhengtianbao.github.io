---
layout: post
title: "Pratt Parsing 算法"
date: 2024-04-15 15:21:26 +0800
categories: ["2024"]
tags: [algorithm]
---

## 什么是 Partt Parsing 算法

Pratt parsing 算法是一种通用的解析方法，用于将程序代码或其他结构化文本转换为抽象语法树（Abstract Syntax Tree）。主要用于解析算术表达式、布尔表达式和类似的语言结构。算法的核心思想是将操作符根据它们的优先级（Precedence）和结合性（Associativity）来分类，然后使用不同的解析函数处理不同类型的操作符。

### 优先级

优先级决定了不同运算符之间的执行顺序。具有较高优先级的运算符会在表达式中较低优先级的运算符之前进行求值。例如，在表达式 `2 + 3 * 4` 中，乘法的优先级高于加法，所以先计算 `3 * 4`，然后再加上 `2`。

### 结合性

结合性指定了相同优先级的运算符在表达式中的结合顺序。运算符可以是左结合（left-associative）、右结合（right-associative）。

- 左结合的运算符从左向右结合，例如这个表达式：`a - b - c`。减法是左结合的，它会按照从左到右的顺序结合，因此表达式会被解析为 `(a - b) - c`，而不是 `a - (b - c)`。
- 右结合的运算符从右向左结合，例如这个表达式：`a ^ b ^ c`。如果指数运算是右结合的，它会按照从右到左的顺序结合。因此表达式会被解析为 `a ^ (b ^ c)`，而不是 `(a ^ b) ^ c`。

Pratt parsing 算法通过将操作符根据它们的优先级和结合性进行分类，并使用不同的解析函数处理不同类型的操作符，从而解决了这个问题。这种方法使得解析器能够根据操作符的优先级和结合性正确地解析表达式，而不需要使用大量的上下文信息。

以上内容由 ChatGPT 强力驱动。一句话总结就是：Parrt Parsing 是将程序表达式或者语句解析为抽象语法树的一种简单易懂的算法。「简单易懂」是作者 Vaughan Pratt 说的：

> The approach described below is very simple to understand, trivial to implement, easy to use, extremely efficient in practice if not in theory, yet flexible enough to meet most reasonable syntactic needs of users in both categories (i) and (ii) above. (What is "reasonable" is addressed in more detail below). Moreover, it deals nicely with error detection.

引用自 [Top Down Operator Precedence](https://tdop.github.io/)
