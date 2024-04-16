---
layout: post
title: "Pratt Parsing 算法"
date: 2024-04-15 15:21:26 +0800
categories: ["2024"]
tags: [algorithm]
---

## 什么是 Pratt Parsing 算法

Pratt parsing 算法是一种通用的解析方法，用于将程序代码或其他结构化文本转换为抽象语法树（Abstract Syntax Tree）。主要用于解析算术表达式、布尔表达式和类似的语言结构。算法的核心思想是将操作符根据它们的优先级（Precedence）和结合性（Associativity）来分类，然后使用不同的解析函数处理不同类型的操作符。

### 优先级

优先级决定了不同运算符之间的执行顺序。具有较高优先级的运算符会在表达式中较低优先级的运算符之前进行求值。例如，在表达式 `2 + 3 * 4` 中，乘法的优先级高于加法，所以先计算 `3 * 4`，然后再加上 `2`。

### 结合性

结合性指定了相同优先级的运算符在表达式中的结合顺序。运算符可以是左结合（left-associative）、右结合（right-associative）。

- 左结合的运算符从左向右结合，例如这个表达式：`a - b - c`。减法是左结合的，它会按照从左到右的顺序结合，因此表达式会被解析为 `(a - b) - c`，而不是 `a - (b - c)`。
- 右结合的运算符从右向左结合，例如这个表达式：`a ^ b ^ c`。如果指数运算是右结合的，它会按照从右到左的顺序结合。因此表达式会被解析为 `a ^ (b ^ c)`，而不是 `(a ^ b) ^ c`。

Pratt parsing 算法通过将操作符根据它们的优先级和结合性进行分类，并使用不同的解析函数处理不同类型的操作符，从而解决了这个问题。这种方法使得解析器能够根据操作符的优先级和结合性正确地解析表达式，而不需要使用大量的上下文信息。

以上内容由 ChatGPT 强力驱动。一句话总结就是：Pratt Parsing 是将程序表达式或者语句解析为抽象语法树的一种简单易懂的算法。「简单易懂」是作者 Vaughan Pratt 说的：

> The approach described below is very simple to understand, trivial to implement, easy to use, extremely efficient in practice if not in theory, yet flexible enough to meet most reasonable syntactic needs of users in both categories (i) and (ii) above. (What is "reasonable" is addressed in more detail below). Moreover, it deals nicely with error detection.

引用自 [Top Down Operator Precedence](https://tdop.github.io/)

## Pratt Parser 实现

本文将用 go 语言描述 Pratt Parser 的实现，出于简单考虑，这里的 Token 定义为单个字符，数字和一些基本算数运算符。

整个解析过程如下，输入的字符串通过 Lexer 分解为 Tokens，然后通过 Parser 解析为 AST。

```
input string -> Lexer -> tokens -> Parser -> AST
```

定义 Token 类型，分为三类：

- ATOM 表示单字节的数字或者字母
- PLUS MINUS ASTERISK SLASH 分别对应符号 + - * /
- EOF 表示输入字符串的结束

```go
package main

type TokenType string

const (
	// End Of File
	EOF = "EOF"

	// Single Character or Number
	ATOM = "ATOM"

	// Operators
	PLUS     = "+"
	MINUS    = "-"
	ASTERISK = "*"
	SLASH    = "/"
)

type Token struct {
	Type    TokenType
	Literal string
}

func NewToken(tokenType TokenType, ch byte) Token {
	return Token{Type: tokenType, Literal: string(ch)}
}
```

定义 Lexer，主要提供了 NextToken() 方法，用于获取下一个 Token

```go
package main

type Lexer struct {
	input        string
	position     int  // current position in input (points to current char)
	readPosition int  // current reading position in input (after current char)
	ch           byte // current char under examination
}

func NewLexer(input string) *Lexer {
	l := &Lexer{input: input}
	l.readChar()
	return l
}

func (l *Lexer) NextToken() Token {
	var tok Token

	l.skipWhitespace()

	switch l.ch {
	case '+':
		tok = NewToken(PLUS, l.ch)
	case '-':
		tok = NewToken(MINUS, l.ch)
	case '/':
		tok = NewToken(SLASH, l.ch)
	case '*':
		tok = NewToken(ASTERISK, l.ch)
	case 0:
		tok.Literal = ""
		tok.Type = EOF
	default:
		tok = NewToken(ATOM, l.ch)
	}

	l.readChar()
	return tok
}

func (l *Lexer) skipWhitespace() {
	for l.ch == ' ' || l.ch == '\t' || l.ch == '\n' || l.ch == '\r' {
		l.readChar()
	}
}

func (l *Lexer) readChar() {
	if l.readPosition >= len(l.input) {
		l.ch = 0
	} else {
		l.ch = l.input[l.readPosition]
	}
	l.position = l.readPosition
	l.readPosition += 1
}
```

position 和 readPosition 两个游标用于定位当前处理的 Token 位置。

在 readChar() 方法中通过赋值 l.ch = 0 标记输入字符串的结束。

定义 AST 节点

```go
package main

import (
	"bytes"
)

type Node interface {
	String() string
}

type Expression interface {
	Node
}

type AtomLiteral struct {
	Token Token
	Value string
}

func (i *AtomLiteral) String() string  { return i.Value }

type ExpressionStatement struct {
	Expression Expression
}

func (es *ExpressionStatement) String() string {
	if es.Expression != nil {
		return es.Expression.String()
	}
	return ""
}
```

### 术语定义

**Infix Operator** 在两个操作数中间，像下面这样的表达式就是：`1 + 2`，`a * b`

**Prefix Operator** 在操作数的前面，例如：`++a`

**Postfix Operator** 在操作数的后面，例如：`b++`

### Infix 实现

### Prefix 实现

### Postfix 实现
