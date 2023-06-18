010 中的 local var 采用的是单字母，比较简单，所以直接在 stack 中为 26 个字母直接开辟空间
011 中需要对 local var 支持多字母类型，更加贴近实际情况，我感觉这里的处理明显是已经开始引入 函数的概念了，而且由于无法固定开辟 local var 空间，所以引入了各种队列

rvcc.h 中

```cpp
// 本地变量
typedef struct Obj Obj;
struct Obj {
  Obj *Next;  // 指向下一对象
  char *Name; // 变量名
  int Offset; // fp 的偏移量，指向了每个 Obj 对应的值在 stack 中的位置
};

// 函数
typedef struct Function Function;
struct Function {
  Node *Body;    // 函数体
  Obj *Locals;   // 本地变量的链表，链表的节点是 Obj 对象
  int StackSize; // 栈大小，因为变量的个数是可变的，所以我们会根据变量的个数开辟 stack
};
```

虽然这里还没有正式引入函数，但是我们可以认为例如 `foo=3;foo;`  这样的语句实际就是一个函数体

所以我们看到 011 中 的 parse 和 codegen 函数也做了相应的改变：

```cpp
// 语法解析入口函数
Function *parse(Token *Tok);

// 代码生成入口函数
void codegen(Function *Prog);
```

其他部分和 010 区别不大。

一些想法，010 貌似没什么必要，011 可以一步到位。