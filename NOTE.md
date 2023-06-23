为了实现指针 + 1 实际是加 sizeof(p), 在这里需要引入类型系统的概念
```cpp
int *p;
p = p + 1;
```

rvcc.h
```diff
@@ -52,6 +52,7 @@ Token *tokenize(char *Input);
 // 生成AST（抽象语法树），语法解析
 //
 
+typedef struct Type Type;
 typedef struct Node Node;
 
 // 本地变量
@@ -98,6 +99,7 @@ struct Node {
   NodeKind Kind; // 节点种类
   Node *Next;    // 下一节点，指代下一语句
   Token *Tok;    // 节点对应的终结符
+  Type *Ty;      // 节点中数据的类型
 
   Node *LHS; // 左部，left-hand side
   Node *RHS; // 右部，right-hand side
@@ -119,6 +121,29 @@ struct Node {
 // 语法解析入口函数
 Function *parse(Token *Tok);
 
+//
+// 类型系统
+//
+
+// 类型种类
+typedef enum {
+  TY_INT, // int整型
+  TY_PTR, // 指针
+} TypeKind;
+
+struct Type {
+  TypeKind Kind; // 种类
+  Type *Base;    // 当 TypeKind 为 TY_PTR 的时候，指针所指向的类型
+};
+
+// 声明一个全局变量，定义在type.c中。
+extern Type *TyInt;
+
+// 判断是否为整型
+bool isInteger(Type *TY);
+// 为节点内的所有节点添加类型
+void addType(Node *Nd);
+
 //
 // 语义分析与代码生成
 //
```

为类型系统新增一个 type.c

type.c
```diff
@@ -0,0 +1,69 @@
+#include "rvcc.h"
+
+// (Type){...}构造了一个复合字面量，相当于Type的匿名变量。
+Type *TyInt = &(Type){TY_INT};
+
+// 判断Type是否为int类型
+bool isInteger(Type *Ty) { return Ty->Kind == TY_INT; }
+
+// 指针类型，并且指向基类
+Type *pointerTo(Type *Base) {
+  Type *Ty = calloc(1, sizeof(Type));
+  Ty->Kind = TY_PTR;
+  Ty->Base = Base;
+  return Ty;
+}
+
+// 为节点内的所有节点添加类型
+void addType(Node *Nd) {
+  // 判断 节点是否为空 或者 节点类型已经有值，那么就直接返回
+  if (!Nd || Nd->Ty)
+    return;
+
+  // 递归访问所有节点以增加类型
+  addType(Nd->LHS);
+  addType(Nd->RHS);
+  addType(Nd->Cond);
+  addType(Nd->Then);
+  addType(Nd->Els);
+  addType(Nd->Init);
+  addType(Nd->Inc);
+
+  // 访问链表内的所有节点以增加类型
+  for (Node *N = Nd->Body; N; N = N->Next)
+    addType(N);
+
+  switch (Nd->Kind) {
+  // 将节点类型设为 节点左部的类型
+  case ND_ADD:
+  case ND_SUB:
+  case ND_MUL:
+  case ND_DIV:
+  case ND_NEG:
+  case ND_ASSIGN:
+    Nd->Ty = Nd->LHS->Ty;
+    return;
+  // 将节点类型设为 int
+  case ND_EQ:
+  case ND_NE:
+  case ND_LT:
+  case ND_LE:
+  case ND_VAR:
+  case ND_NUM:
+    Nd->Ty = TyInt;
+    return;
+  // 将节点类型设为 指针，并指向左部的类型
+  case ND_ADDR:
+    Nd->Ty = pointerTo(Nd->LHS->Ty);
+    return;
+  // 节点类型：如果解引用指向的是指针，则为指针指向的类型；否则为int
+  case ND_DEREF:
+    if (Nd->LHS->Ty->Kind == TY_PTR)
+      Nd->Ty = Nd->LHS->Ty->Base;
+    else
+      Nd->Ty = TyInt;
+    return;
+  default:
+    break;
+  }
+}
```