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

- 左结合的运算符从左向右结合，例如这个表达式：`2 - 3 - 4`。减法是左结合的，它会按照从左到右的顺序结合，因此表达式会被解析为 `(2 - 3) - 4`，而不是 `2 - (3 - 4)`。
- 右结合的运算符从右向左结合，例如这个表达式：`2 ^ 3 ^ 4`。如果指数运算是右结合的，它会按照从右到左的顺序结合。因此表达式会被解析为 `2 ^ (3 ^ 4)`，而不是 `(2 ^ 3) ^ 4`。

Pratt parsing 算法通过将操作符根据它们的优先级和结合性进行分类，并使用不同的解析函数处理不同类型的操作符，从而解决了这个问题。这种方法使得解析器能够根据操作符的优先级和结合性正确地解析表达式，而不需要使用大量的上下文信息。

以上内容由 ChatGPT 强力驱动。一句话总结就是：Pratt Parsing 是将程序表达式或者语句解析为抽象语法树的一种简单易懂的算法。「简单易懂」是作者 Vaughan Pratt 说的：

> The approach described below is very simple to understand, trivial to implement, easy to use, extremely efficient in practice if not in theory, yet flexible enough to meet most reasonable syntactic needs of users in both categories (i) and (ii) above. (What is "reasonable" is addressed in more detail below). Moreover, it deals nicely with error detection.

引用自 [Top Down Operator Precedence](https://tdop.github.io/)

## Pratt Parser 实现

本文将用 go 语言描述 Pratt Parser 的实现，出于简单考虑，假设需要解析的 Token 定义为单个数字，基本算数运算符和括号。

最终目的是实现以下效果：

```
input: (1 + (2 + 3) * 4) / (-(5 + 6!))
output: ((1 + ((2 + 3) * 4)) / (-(5 + (6!))))
```

整个解析过程如下，输入的字符串通过 Lexer 分解为 Tokens，然后通过 Parser 解析为 AST。

```
input string --Lexer--> tokens --Parser--> AST
```

### 定义 Token

Token 类型分为三类：

- NUMBER 表示单字节的字母或者数字
- PLUS MINUS ASTERISK SLASH BANG LPAREN RPAREN 分别对应符号 + - * / ! ( )
- EOF 表示输入字符串的结束

```go
package main

type TokenType string

const (
	// Single Number
	NUMBER = "NUMBER"

	// Operators
	PLUS     = "+"
	MINUS    = "-"
	ASTERISK = "*"
	SLASH    = "/"
	BANG     = "!"
	LPAREN   = "("
	RPAREN   = ")"

	// End Of File
	EOF = "EOF"
)

type Token struct {
	Type    TokenType
	Literal string
}

func NewToken(tokenType TokenType, ch byte) Token {
	return Token{Type: tokenType, Literal: string(ch)}
}
```

### 定义 Lexer

Lexer 用于解析输入字符串，生成 Tokens，主要提供了 `NextToken()` 方法，用于获取下一个 Token。

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
	case '!':
		tok = NewToken(BANG, l.ch)
	case '(':
		tok = NewToken(LPAREN, l.ch)
	case ')':
		tok = NewToken(RPAREN, l.ch)
	case 0:
		tok.Literal = ""
		tok.Type = EOF
	default:
		tok = NewToken(NUMBER, l.ch)
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

`position` 和 `readPosition` 两个游标参数用于定位当前处理的 Token 位置。

在 `readChar()` 方法中通过赋值语句 `l.ch = 0` 标记输入字符串的结束。

### 定义 AST

AST 是由节点（Node）构成的树状结构，因此定义一个基本接口 `Node`。

同时我们需要解析的都是表达式（Expression），注意表达式和语句（statement）的区别：参考 rust 语言里面对这两者的定义，表达式返回值，而语句没有返回值。

最基本的表达式：

```
1
```

单个数字就是一个表达式，因为它代表返回值 1，在这里把这种单字节的数字或者字母定义为 `Number` 类型，实现了 `Expression` 接口。

```go
package main

// The base Node interface
type Node interface {
	String() string
}

// All expression nodes implement this
type Expression interface {
	Node
}

type Number struct {
	Token Token
	Value string
}

func (i *Number) String() string { return i.Value }
```

### 解析单字符表达式

现在开始编写一个最简单版本的 Parser，只能够解析上面的单个数字的表达式。

```go
package main

type Parser struct {
	l *Lexer

	curToken  Token
	peekToken Token
}

func NewParser(l *Lexer) *Parser {
	p := &Parser{
		l: l,
	}
	// Read two tokens, so curToken and peekToken are both set
	p.nextToken()
	p.nextToken()

	return p
}

func (p *Parser) nextToken() {
	p.curToken = p.peekToken
	p.peekToken = p.l.NextToken()
}

func (p *Parser) Parse() string {
	switch p.curToken.Type {
	case NUMBER:
		return p.parseNumberExpression().String()
	default:
		return ""
	}
}

func (p *Parser) parseNumberExpression() Expression {
	return &Number{Token: p.curToken, Value: p.curToken.Literal}
}
```

在 `Parse()` 方法中简单匹配当前 Token 的类型，因为单个数字定义为 `NUMBER` 类型，所以调用 `parseNumberExpression()` 方法，生成 `Number` 表达式。

编写测试用例：

```go
package main

import (
	"testing"
)

func TestNumberParsing(t *testing.T) {
	tests := []struct {
		input    string
		expected string
	}{
		{
			"",
			"",
		},
		{
			"1",
			"1",
		},
	}

	for _, tt := range tests {
		l := NewLexer(tt.input)
		p := NewParser(l)
		actual := p.Parse()

		if actual != tt.expected {
			t.Errorf("expected=%q, got=%q", tt.expected, actual)
		}
	}
}
```

执行测试命令，通过测试：

```
$ go test -v .
=== RUN   TestNumberParsing
--- PASS: TestNumberParsing (0.00s)
PASS
ok      github.com/zhengtianbao/pratt   0.001s
```

### 解析前缀表达式

考虑以下表达式：

```
-5 * 3
```

其中的 - 优先级高于 *，所以应该解析为 ((-5) * 3)，像这里的 -5 就是前缀表达式，为了避免引起优先级歧义这里将解析结果用括号括起来。

前缀表达式可以由以下形式表示：

```
<prefix operator> <expression>
```

定义前缀表达式类型：

```go
import (
	"bytes"
)

type PrefixExpression struct {
	Token    Token // The prefix token, e.g. -
	Operator string
	Right    Expression
}

func (pe *PrefixExpression) String() string {
	var out bytes.Buffer

	out.WriteString("(")
	out.WriteString(pe.Operator)
	out.WriteString(pe.Right.String())
	out.WriteString(")")

	return out.String()
}
```

不同与表示单个数字的 `NumberExpression` 直接输出数字，`PrefixExpression` 将结果用括号括起来，方便表示优先级。

现在需要解析的表达式长度变长了，我们将 `Parse()` 方法稍作修改：

```go
func (p *Parser) Parse() string {
	expr := p.parseExpression()
	if expr == nil {
		return ""
	}
	return expr.String()
}

func (p *Parser) parseExpression() Expression {
	var leftExp Expression
	switch p.curToken.Type {
	case NUMBER:
		leftExp = p.parseNumberExpression()
	case MINUS:
		leftExp = p.parsePrefixExpression()
	default:
		leftExp = nil
	}
	return leftExp
}

func (p *Parser) parsePrefixExpression() Expression {
	expression := &PrefixExpression{
		Token:    p.curToken,
		Operator: p.curToken.Literal,
	}

	p.nextToken()
	expression.Right = p.parseExpression()
	return expression
}
```

解析 `-5` 表达式的过程为，先解析 - 代表的 Token，在 `parseExpression()` 方法中判断 `p.curToken.Type` 为 `MINUS`，因此构造 `PrefixExpression` 类型的表达式，在方法 `parsePrefixExpression()` 中调用 `p.nextToken()` 将 `p.curToken` 向右移，`expression.Right` 递归调用了 `p.parseExpression()` 来解析 5 这个 Token，最终形成了一个 `PrefixExpression`。

编写测试用例：

```go
func TestPrefixParsing(t *testing.T) {
	tests := []struct {
		input    string
		expected string
	}{
		{
			"-1",
			"(-1)",
		},
		{
			"--1",
			"(-(-1))",
		},
	}

	for _, tt := range tests {
		l := NewLexer(tt.input)
		p := NewParser(l)
		actual := p.Parse()

		if actual != tt.expected {
			t.Errorf("expected=%q, got=%q", tt.expected, actual)
		}
	}
}
```

OK，测试顺利通过：

```
$ go test -v .
=== RUN   TestNumberParsing
--- PASS: TestNumberParsing (0.00s)
=== RUN   TestPrefixParsing
--- PASS: TestPrefixParsing (0.00s)
PASS
ok      github.com/zhengtianbao/pratt   0.001s
```

### 解析中缀表达式

考虑以下表达式：

```
a + b
```

可以表示为以下形式：

```
<expression> <infix operator> <expression>
```

定义中缀表达式的类型：

```go
type InfixExpression struct {
	Token    Token // The operator token, e.g. +
	Left     Expression
	Operator string
	Right    Expression
}

func (oe *InfixExpression) String() string {
	var out bytes.Buffer

	out.WriteString("(")
	out.WriteString(oe.Left.String())
	out.WriteString(" " + oe.Operator + " ")
	out.WriteString(oe.Right.String())
	out.WriteString(")")

	return out.String()
}
```

与前缀表达式 `PrefixExpression` 一样，中缀表达式 `InfixExpression` 的输出也用括号括起来。

在解析前缀表达式中，我们可以根据首先碰到的 Token 类型来判断是不是一个前缀表达式，例如碰到 - 代表的 Token `MINUS`，那么就能推定必然是个前缀表达式。但是如何确定中缀表达式呢？想象一下在解析表达式 `1 + 2` 过程中首先碰到的是数字 1 代表的 Token `NUMBER`，只有提前知道下一个 Token 是 + 代表的 `PLUS` 时才能作为中缀表达式进行解析，因此在解析中缀表达式中，`Parser` 中的 `peekToken` 就体现出作用了，可以用来提前知道下一个字符的 Token 类型。

扩展 `parseExpression()` 方法：

```go
func (p *Parser) parseExpression() Expression {
	var leftExp Expression
	switch p.curToken.Type {
	case NUMBER:
		leftExp = p.parseNumberExpression()
	case MINUS:
		leftExp = p.parsePrefixExpression()
	default:
		leftExp = nil
	}

	for p.peekToken.Type != EOF {
		switch p.peekToken.Type {
		case PLUS, MINUS, ASTERISK, SLASH:
			p.nextToken()
			leftExp = p.parseInfixExpression(leftExp)
		default:
			return leftExp
		}
	}
	return leftExp
}

func (p *Parser) parseInfixExpression(left Expression) Expression {
	expression := &InfixExpression{
		Token:    p.curToken,
		Operator: p.curToken.Literal,
		Left:     left,
	}

	p.nextToken()
	expression.Right = p.parseExpression()
	return expression
}
```

注意，在 `parseExpression()` 中使用了 `for p.peekToken.Type != EOF {}` 而不是 `if p.peekToken.Type != EOF {}`，这意味着程序可以解析 `a + b + c` 这样的表达式。

编写测试用例：

```go
func TestInfixParsing(t *testing.T) {
	tests := []struct {
		input    string
		expected string
	}{
		{
			"1 + 2",
			"(1 + 2)",
		},
		{
			"1 + -2",
			"(1 + (-2))",
		},
		{
			"1 + 2 + 3",
			"((1 + 2) + 3)",
		},
	}

	for _, tt := range tests {
		l := NewLexer(tt.input)
		p := NewParser(l)
		actual := p.Parse()

		if actual != tt.expected {
			t.Errorf("expected=%q, got=%q", tt.expected, actual)
		}
	}
}
```

运行测试，结果报错：

```
$ go test -v .
=== RUN   TestNumberParsing
--- PASS: TestNumberParsing (0.00s)
=== RUN   TestPrefixParsing
--- PASS: TestPrefixParsing (0.00s)
=== RUN   TestInfixParsing
    parser_test.go:84: expected="((1 + 2) + 3)", got="(1 + (2 + 3))"
--- FAIL: TestInfixParsing (0.00s)
FAIL
FAIL    github.com/zhengtianbao/pratt   0.001s
FAIL
```

很显然，在解析 `1 + 2 + 3` 这个表达式的时候，没有按我们预期的顺序进行结合，按结果看是右结合的，我们需要增加一个判断，如果右边的运算符优先级比左边的高，才能进行右结合，目前的代码中还没有对优先级进行处理。

先定义优先级：

```go
const (
	_ int = iota
	LOWEST
	SUM     // +
	PRODUCT // *
	PREFIX  // -X
	POSTFIX // X!
)

var precedences = map[TokenType]int{
	PLUS:     SUM,
	MINUS:    SUM,
	ASTERISK: PRODUCT,
	SLASH:    PRODUCT,
	BANG:     POSTFIX,
}
```

修改 `parseExpression()` 方法，增加参数 `precedence`，用于表示当前解析表达式的优先级

```go
func (p *Parser) Parse() string {
	expr := p.parseExpression(LOWEST)
	if expr == nil {
		return ""
	}
	return expr.String()
}

func (p *Parser) parseExpression(precedence int) Expression {
	var leftExp Expression
	switch p.curToken.Type {
	case NUMBER:
		leftExp = p.parseNumberExpression()
	case MINUS:
		leftExp = p.parsePrefixExpression()
	default:
		leftExp = nil
	}

	for p.peekToken.Type != EOF {
		switch p.peekToken.Type {
		case PLUS, MINUS, ASTERISK, SLASH:
			if precedence >= p.peekPrecedence() {
				return leftExp
			}
			p.nextToken()
			leftExp = p.parseInfixExpression(leftExp)
		default:
			return leftExp
		}
	}
	return leftExp
}

func (p *Parser) parsePrefixExpression() Expression {
	expression := &PrefixExpression{
		Token:    p.curToken,
		Operator: p.curToken.Literal,
	}

	p.nextToken()
	expression.Right = p.parseExpression(PREFIX)
	return expression
}

func (p *Parser) parseInfixExpression(left Expression) Expression {
	expression := &InfixExpression{
		Token:    p.curToken,
		Operator: p.curToken.Literal,
		Left:     left,
	}
	precedence := p.curPrecedence()
	p.nextToken()
	expression.Right = p.parseExpression(precedence)

	return expression
}

func (p *Parser) curPrecedence() int {
	if p, ok := precedences[p.curToken.Type]; ok {
		return p
	}
	return LOWEST
}

func (p *Parser) peekPrecedence() int {
	if p, ok := precedences[p.peekToken.Type]; ok {
		return p
	}
	return LOWEST
}
```

增加了两个方法 `curPrecedence()` 和 `peekPrecedence()` 用于获取当前运算符优先级以及下一个字符的优先级，数字类型 `NUMBER` 是最低优先级的。

增加一个测试案例：

```go
func TestInfixParsing(t *testing.T) {
	tests := []struct {
		input    string
		expected string
	}{
		{
			"1 + 2",
			"(1 + 2)",
		},
		{
			"1 + -2",
			"(1 + (-2))",
		},
		{
			"1 + 2 + 3",
			"((1 + 2) + 3)",
		},
		{
			"1 + 2 * 3 + 4 / 5 - 6",
			"(((1 + (2 * 3)) + (4 / 5)) - 6)",
		},
	}

	for _, tt := range tests {
		l := NewLexer(tt.input)
		p := NewParser(l)
		actual := p.Parse()

		if actual != tt.expected {
			t.Errorf("expected=%q, got=%q", tt.expected, actual)
		}
	}
}
```

OK，测试通过：

```
$ go test -v .
=== RUN   TestNumberParsing
--- PASS: TestNumberParsing (0.00s)
=== RUN   TestPrefixParsing
--- PASS: TestPrefixParsing (0.00s)
=== RUN   TestInfixParsing
--- PASS: TestInfixParsing (0.00s)
PASS
ok      github.com/zhengtianbao/pratt   0.001s
```

### 解析后缀表达式

考虑以下表达式：

```
5!
```

所以后缀表达式可以由以下形式表示：

```
<expression> <postfix operator> 
```

定义后缀表达式类型

```go
type PostfixExpression struct {
	Token    Token // The postfix token, e.g. !
	Operator string
	Left     Expression
}

func (pe *PostfixExpression) String() string {
	var out bytes.Buffer

	out.WriteString("(")
	out.WriteString(pe.Left.String())
	out.WriteString(pe.Operator)
	out.WriteString(")")

	return out.String()
}
```

修改 `parseExpression()` 方法

```go
func (p *Parser) parseExpression(precedence int) Expression {
	var leftExp Expression
	switch p.curToken.Type {
	case NUMBER:
		leftExp = p.parseNumberExpression()
	case MINUS:
		leftExp = p.parsePrefixExpression()
	default:
		leftExp = nil
	}

	for p.peekToken.Type != EOF {
		switch p.peekToken.Type {
		case PLUS, MINUS, ASTERISK, SLASH:
			if precedence >= p.peekPrecedence() {
				return leftExp
			}
			p.nextToken()
			leftExp = p.parseInfixExpression(leftExp)
		case BANG:
			if precedence >= p.peekPrecedence() {
				return leftExp
			}
			p.nextToken()
			leftExp = p.parsePostExpression(leftExp)
		default:
			return leftExp
		}
	}
	return leftExp
}

func (p *Parser) parsePostExpression(left Expression) Expression {
	expression := &PostfixExpression{
		Token:    p.curToken,
		Operator: p.curToken.Literal,
		Left:     left,
	}
	return expression
}
```

后缀表达式跟中缀表达式的区别就是没有右侧表达式 `Right`， 因此不需要调用 `p.nextToken()`。

增加后缀表达式的测试：

```go
func TestPostfixParsing(t *testing.T) {
	tests := []struct {
		input    string
		expected string
	}{
		{
			"1!",
			"(1!)",
		},
		{
			"1 + 2!",
			"(1 + (2!))",
		},
		{
			"1 + 2! + 3",
			"((1 + (2!)) + 3)",
		},
	}

	for _, tt := range tests {
		l := NewLexer(tt.input)
		p := NewParser(l)
		actual := p.Parse()

		if actual != tt.expected {
			t.Errorf("expected=%q, got=%q", tt.expected, actual)
		}
	}
}
```

测试通过：

```
$ go test -v .
=== RUN   TestNumberParsing
--- PASS: TestNumberParsing (0.00s)
=== RUN   TestPrefixParsing
--- PASS: TestPrefixParsing (0.00s)
=== RUN   TestInfixParsing
--- PASS: TestInfixParsing (0.00s)
=== RUN   TestPostfixParsing
--- PASS: TestPostfixParsing (0.00s)
PASS
ok      github.com/zhengtianbao/pratt   0.001s
```

### 解析包含括号的表达式

最后，将要实现能够正确解析包含括号的表达式。

编写测试用例：

```go
func TestParenParsing(t *testing.T) {
	tests := []struct {
		input    string
		expected string
	}{
		{
			"(1 + 2)",
			"(1 + 2)",
		},
		{
			"(1 + 2) + 3",
			"((1 + 2) + 3)",
		},
		{
			"1 + (2 + 3) + 4",
			"((1 + (2 + 3)) + 4)",
		},
		{
			"(5 + 5) * 2",
			"((5 + 5) * 2)",
		},
		{
			"2 / (5 + 5)",
			"(2 / (5 + 5))",
		},
		{
			"(5 + 5) * 2 * (5 + 5)",
			"(((5 + 5) * 2) * (5 + 5))",
		},
		{
			"1 + (2 + 3) * 4",
			"(1 + ((2 + 3) * 4))",
		},
		{
			"-(5 + 5)",
			"(-(5 + 5))",
		},
	}

	for _, tt := range tests {
		l := NewLexer(tt.input)
		p := NewParser(l)
		actual := p.Parse()

		if actual != tt.expected {
			t.Errorf("expected=%q, got=%q", tt.expected, actual)
		}
	}
}
```

修改 `parseExpression()` 方法，当遇到 `LPAREN` 类型时调用 `parseParenExpression()`：

```go
func (p *Parser) parseExpression(precedence int) Expression {
	var leftExp Expression
	switch p.curToken.Type {
	case NUMBER:
		leftExp = p.parseNumberExpression()
	case MINUS:
		leftExp = p.parsePrefixExpression()
	case LPAREN:
		leftExp = p.parseParenExpression()
	default:
		leftExp = nil
	}

	for p.peekToken.Type != EOF {
		switch p.peekToken.Type {
		case PLUS, MINUS, ASTERISK, SLASH:
			if precedence >= p.peekPrecedence() {
				return leftExp
			}
			p.nextToken()
			leftExp = p.parseInfixExpression(leftExp)
		case BANG:
			if precedence >= p.peekPrecedence() {
				return leftExp
			}
			p.nextToken()
			leftExp = p.parsePostExpression(leftExp)
		default:
			return leftExp
		}
	}
	return leftExp
}

func (p *Parser) parseParenExpression() Expression {
	p.nextToken()

	exp := p.parseExpression(LOWEST)

	if p.peekToken.Type == RPAREN {
		p.nextToken()
	}
	return exp
}
```

在 `parseParenExpression()` 方法中很简单，就是跳过左括号，解析括号内的表达式，然后再跳过对应的右括号。

执行测试：

```
$ go test -v .
=== RUN   TestNumberParsing
--- PASS: TestNumberParsing (0.00s)
=== RUN   TestPrefixParsing
--- PASS: TestPrefixParsing (0.00s)
=== RUN   TestInfixParsing
--- PASS: TestInfixParsing (0.00s)
=== RUN   TestPostfixParsing
--- PASS: TestPostfixParsing (0.00s)
=== RUN   TestParenParsing
--- PASS: TestParenParsing (0.00s)
PASS
ok      github.com/zhengtianbao/pratt   0.001s
```

至此，已经实现了一个简单版本的 Pratt Parser。

详细代码请查看：<https://github.com/zhengtianbao/pratt>

## 参考链接

<https://interpreterbook.com/>

<https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html>
