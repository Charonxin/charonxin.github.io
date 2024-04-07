---
title: "llvm tutorials - 2-1"
categories:
  - blog
tags:
  - llvm
---

# llvm tutorials - $2-1$ Kaleidoscope: Kaleidoscope Introduction and the Lexer

---------

## 1. Kaleidoscope 语言

Kaleidscope 是一种过程语言，允许您定义函数、使用条件、数学等。在本教程中，我们将扩展 Kaleidscope 以支持 if/then/else 结构、for 循环、用户定义的运算符、JIT使用简单的命令行界面、调试信息等进行编译

我们想让事情变得简单，因此 Kaleidscope 中唯一的数据类型是 64 位浮点类型（在 C 语言中也称为“double”）。因此，所有值都是隐式双精度的，并且该语言不需要类型声明。这为该语言提供了非常漂亮且简单的语法。

## 1.2 词法分析器

词法分析器返回的每个标记都包括标记代码和可能的一些元数据（例如数字的数值）。首先，我们定义可能性：

```c++
// The lexer returns tokens [0-255] if it is an unknown character, otherwise one
// of these for known things.
enum Token {
  tok_eof = -1,

  // commands
  tok_def = -2,
  tok_extern = -3,

  // primary
  tok_identifier = -4,
  tok_number = -5,
};

static std::string IdentifierStr; // Filled in if tok_identifier
static double NumVal;             // Filled in if tok_number
```

我们的词法分析器返回的每个标记要么是 Token 枚举值之一，要么是像“+”这样的“未知”字符，它作为 **ASCII** 值返回。如果当前标记是标识符，则 `IdentifierStr`**全局变量**保存标识符的名称。如果当前标记是数字文字（如 1.0），则`NumVal`保留其值。为了简单起见，我们使用全局变量，但这不是真正语言实现的最佳选择:)。

词法分析器的实际实现是一个名为 的函数 `gettok`。`gettok`调用该函数以从标准输入返回下一个标记。它的定义开始如下：

```C++
/// gettok - Return the next token from standard input.
static int gettok() {
  static int LastChar = ' ';

  // Skip any whitespace.
  while (isspace(LastChar))
    LastChar = getchar();
```

`gettok`其工作原理是调用 C`getchar()`函数从标准输入中一次读取一个字符。它在识别它们时将其吃掉，并将最后读取但未处理的字符存储在 LastChar 中。它要做的第一件事是忽略标记之间的空格。这是通过上面的循环完成的。

接下来`gettok`需要做的是识别标识符和特定关键字，例如“def”。万花筒通过这个简单的循环来做到这一点：

```C++
if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
  IdentifierStr = LastChar;
  while (isalnum((LastChar = getchar())))
    IdentifierStr += LastChar;

  if (IdentifierStr == "def")
    return tok_def;
  if (IdentifierStr == "extern")
    return tok_extern;
  return tok_identifier;
}
```

请注意，每当此代码`IdentifierStr`对标识符进行词法分析时，都会设置“ ”全局变量。另外，由于语言关键字由同一循环匹配，因此我们在这里内联处理它们。数值类似：

```C++
if (isdigit(LastChar) || LastChar == '.') {   // Number: [0-9.]+
  std::string NumStr;
  do {
    NumStr += LastChar;
    LastChar = getchar();
  } while (isdigit(LastChar) || LastChar == '.');

  NumVal = strtod(NumStr.c_str(), 0);
  return tok_number;
}
```

这些都是用于处理输入的非常简单的代码。从输入读取数值时，我们**使用 C`strtod`函数将其转换为存储在 中的数值`NumVal`**。请注意，这并没有进行足够的错误检查：它将错误地读取“1.23.45.67”并像您键入“1.23”一样处理它。请随意扩展它！接下来我们处理评论：

```C++
if (LastChar == '#') {
  // Comment until end of line.
  do
    LastChar = getchar();
  while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

  if (LastChar != EOF)
    return gettok();
}
```

我们通过跳到行尾来处理注释，然后返回下一个标记。最后，如果输入与上述情况之一不匹配，则它要么是“+”等运算符字符，要么是文件结尾。这些都是用这段代码处理的：

```C++
  // Check for end of file.  Don't eat the EOF.
  if (LastChar == EOF)
    return tok_eof;

  // Otherwise, just return the character as its ascii value.
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```

