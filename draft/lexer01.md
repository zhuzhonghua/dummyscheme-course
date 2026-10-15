# 词法分析

> 道可道，非常道。
> 名可名，非常名。
> 无，名天地之始。
> 有，名万物之母。
> 故常无，欲以观其妙；
> 常有，欲以观其徼。
> 此两者，同出而异名。
> 同谓之玄。
> 玄之又玄，众妙之门。
> ——老子《道德经》

字符流 --> token --> ValueT

---

## 先看ValueT
```
struct ValueT {
  ValueU v;
  byte t;
};
```
v 是所有类型的union, 基本类型(num, char, ...)，指针类型(ref)
```
union ValueU {
  RefPtr p;
  BasicNum num;
  uchar c;
};
union BasicNum {
  scm_int i; // integer
  double r; // real
};
struct RefObject {
  ...
  RefPtr gcnxt;
};
typedef RefObject* RefPtr;
typedef uchar scm_char;
typedef long scm_int;
```
RefObject::gcnxt是为了GC用的，所有new出来的对象都会通过这个字段用单向链表挂起来

ValueT::t表示类型，根据不同类型决定ValueU中哪个字段有意义
```
enum ValueTEnum {
  ...
  VT_NULL, // nil or null
  VT_NUM_INTEGER,
  VT_NUM_REAL,
  VT_CHAR,
  VT_REF,
  ...
};
```
---
## 字符流
抽象一个Reader类型，用于支持字符串，REPL，文件，甚至将来的网络数据
```
class Reader {
public:
  virtual ~Reader() {}
  virtual int nextc() = 0;
  ...
};

class ReaderS : public Reader {
public:
  ReaderS(const char* p): buff(p), n(0) {
    size = strlen(p);
  }

  virtual int nextc() { return n < size ? buff[n++] : -1; }
  ...

  const char* buff;
  int n;
  int size;
};
```
以ReaderS为例，ReaderS::size是buff的长度，ReaderS::n下一个要读的字符，返回-1表示末尾，
ReaderS::buff指向字符串，外部调用方需要负责它的生命周期。
不过使用过程一般是
```
const char* p = "(define a 2)";
Reader S(p);
...
```
一个局部变量即可，不需要特别在意，除非要做异步，现在不用考虑。

## 接下来重中之重，字符流 -> Token
需要借助一个类Lexer配合上面的Reader来完成词法分析
```
class Lexer {
public:
  ...
  void nextc() { curchar = reader->nextc(); }
  int dLex();
  ...
  Reader* r;
  int curchar;
  ...
};
```
Lexer::curchar是当前读到的字符，Lexer::dLex需要用它判断当前是什么token类型(int, string, symbol, char, ...)。
关于Token类型
```
enum TokenType {
  TOKEN_UNKNOWN,
  TOKEN_END,
  ...
  TOKEN_DOT,
  ...
  TOKEN_LEFT_PAREN,
  TOKEN_RIGHT_PAREN,
  ...
  TOKEN_MAX,
};
```
当前只需要关心\( \) 还有 \. 即可。
Lexer::dLex就是一个巨大的switch case
```
int Lexer::dLex()
{
  int token = TOKEN_UNKNOWN;
 loop:
  switch(curchar) {
  case READ_END:
    return TOKEN_END;
  case '\n': case '\r':
    plusline();
    goto loop;
  case ' ': case '\f': case '\t': case '\v':
    nextc();
    goto loop;
  case ';':
    do {
      nextc();
    } while(curchar != READ_END && curchar != '\n' && curchar != '\r');
    goto loop;
  case '(':
    token = TOKEN_LEFT_PAREN;
    ...
    nextc();
    break;
  case ')':
	token = TOKEN_RIGHT_PAREN;
    nextc();
    break;
  }
}
#define READ_END -1
```
如果遇到了READ_END(-1)，那就返回TOKEN_END，\; 是Lisp/Scheme的注释，从当前位置到行尾都是注释。
遇到\( \)设置token类型，然后向前读取一个字符。
非换行的空白就继续往前探索，关于换行有必要特殊说明一下，很好玩（参考Lua）
```
void Lexer::plusline()
{
  ...
  int old = curchar;
  nextc();
  // \n\r or \r\n
  // not \n\n
  if ((curchar == '\n' || curchar == '\r') && curchar != old)
    nextc();
}
```
```
\r\n   -> 行号 +1
\n\r   -> 行号 +1
\n\n   -> 行号 +2
\r\r   -> 行号 +2
```
