目前支持的是单字母的变量

# 词法分析
增加 TK_IDENT 类型
```cpp
// 解析标记符
    if ('a' <= *P && *P <= 'z') {
      Cur->Next = newToken(TK_IDENT, P, P + 1);
      Cur = Cur->Next;
      ++P;
      continue;
    }
```

# 语法分析

struct Node 上新增了一个 Name 成员，用来保存变量名
```cpp
struct Node {
  NodeKind Kind; // 节点种类
  Node *Next;    // 下一节点，指代下一语句
  Node *LHS;     // 左部，left-hand side
  Node *RHS;     // 右部，right-hand side
  char Name;     // 存储ND_VAR的字符串
  int Val;       // 存储ND_NUM种类的值
};
```

针对新增加的 TK_IDENT 的 node 进入 newVarNode 处理

```cpp
  // ident
  if (Tok->Kind == TK_IDENT) {
    Node *Nd = newVarNode(*Tok->Loc);
    *Rest = Tok->Next;
    return Nd;
  }
```

newVarNode 中会将变量的名称记录下来
```cpp
static Node *newVarNode(char Name) {
  Node *Nd = newNode(ND_VAR);
  Nd->Name = Name;
  return Nd;
}
```

# Codegen

测试例子：`a=3;a+1`

```cpp
  case ND_ASSIGN:
    // 左部是左值，保存值到的地址
    genAddr(Nd->LHS);
    push();
    // 右部是右值，为表达式的值
    genExpr(Nd->RHS);
    pop("a1");
    printf("  sd a0, 0(a1)\n");
    return;
```

先处理 `a=3;` 这种赋值语句，左值（“=” 的左边）必须是一个地址，右边是一个值。

先在栈中为 26 个字母分配空间，就是我们的 local 变量，local 变量是放在 stack 里的

同时注意 Prologue 中利用 fp，Epilog 中要对 fp 进行恢复。

`genAddr()` 根据左值对应的Node 的 Name 字段保存的字母在 local 变量的 stack 中找到对应的变量的地址，
然后 push() 将左值压栈

a+1 这种表达式不是赋值，所以走的逻辑和上面的不同，走入 genExpr() 的缺省处理逻辑

```cpp
  // 递归到最右节点
  genExpr(Nd->RHS);
  // 将结果压入栈
  push();
  // 递归到左节点
  genExpr(Nd->LHS);
  // 将结果弹栈到a1
  pop("a1");
```

a 的值可以从前面保存在 local stack 中 a 对应的地址获取

```cpp
  case ND_VAR:
    // 计算出变量的地址，然后存入a0
    genAddr(Nd);
    // 访问a0地址中存储的数据，存入到a0当中
    printf("  ld a0, 0(a0)\n");
    return;
```