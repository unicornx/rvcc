支持语句后，语法分析树就不是一棵树，而是多棵树的一个链表，这个通过 `struct Node` 中的 Next 成员来表示。每个子树表示一个语句。

```cpp
struct Node {
  NodeKind Kind; // 节点种类
  Node *Next;    // 下一节点，指代下一语句
  Node *LHS;     // 左部，left-hand side
  Node *RHS;     // 右部，right-hand side
  int Val;       // 存储ND_NUM种类的值
};
```