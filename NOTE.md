为了实现指针 + 1 实际是加 sizeof(p), 在这里需要引入类型系统的概念
```cpp
int *p;
p = p + 1;
```

对 codegen 没有影响变化


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

addType 中的操作是不是体现了表达式都是有类型的概念？FIXME

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
+  // 这里是值多个 compoundStatment 吗？
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
+  // 这里的左值是什么意思？
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

parse.c
在语法分析过程中会调用 addType，如下地方目前会调用 addType
- compoundStmt 中每构造完一个 statement 对应的 AST，则对这个 statement 对应的 node 调用 addType，这会触发内部递归的动作发生。为啥要在 statment 解析完之后再统一处理？FIXME
- 涉及 +/- 操作的处理中，参考 add(), 会触发调用 newAdd/newSub


```diff
@@ -21,6 +21,7 @@ Obj *Locals;
 // unary = ("+" | "-" | "*" | "&") unary | primary
 // primary = "(" expr ")" | ident | num
 static Node *compoundStmt(Token **Rest, Token *Tok);
+static Node *stmt(Token **Rest, Token *Tok);
 static Node *exprStmt(Token **Rest, Token *Tok);
 static Node *expr(Token **Rest, Token *Tok);
 static Node *assign(Token **Rest, Token *Tok);
@@ -182,6 +183,8 @@ static Node *compoundStmt(Token **Rest, Token *Tok) {
   while (!equal(Tok, "}")) {
     Cur->Next = stmt(&Tok, Tok);
     Cur = Cur->Next;
+    // 构造完AST后，为节点添加类型信息
+    addType(Cur);
   }
 
   // Nd的Body存储了{}内解析的语句
@@ -292,6 +295,64 @@ static Node *relational(Token **Rest, Token *Tok) {
   }
 }
 
+// 解析各种加法
+static Node *newAdd(Node *LHS, Node *RHS, Token *Tok) {
+  // 为左右部添加类型
+  addType(LHS);
+  addType(RHS);
+
+  // num + num
+  if (isInteger(LHS->Ty) && isInteger(RHS->Ty))
+    return newBinary(ND_ADD, LHS, RHS, Tok);
+
+  // 不能解析 ptr + ptr
+  if (LHS->Ty->Base && RHS->Ty->Base)
+    errorTok(Tok, "invalid operands");
+
+  // 将 num + ptr 转换为 ptr + num
+  if (!LHS->Ty->Base && RHS->Ty->Base) {
+    Node *Tmp = LHS;
+    LHS = RHS;
+    RHS = Tmp;
+  }
+
+  // ptr + num
+  // 指针加法，ptr+1，这里的1不是1个字节，而是1个元素的空间，所以需要 ×8 操作
+  RHS = newBinary(ND_MUL, RHS, newNum(8, Tok), Tok);
+  return newBinary(ND_ADD, LHS, RHS, Tok);
+}
+
+// 解析各种减法
+static Node *newSub(Node *LHS, Node *RHS, Token *Tok) {
+  // 为左右部添加类型
+  addType(LHS);
+  addType(RHS);
+
+  // num - num
+  if (isInteger(LHS->Ty) && isInteger(RHS->Ty))
+    return newBinary(ND_SUB, LHS, RHS, Tok);
+
+  // ptr - num
+  if (LHS->Ty->Base && isInteger(RHS->Ty)) {
+    RHS = newBinary(ND_MUL, RHS, newNum(8, Tok), Tok);
+    addType(RHS);
+    Node *Nd = newBinary(ND_SUB, LHS, RHS, Tok);
+    // 节点类型为指针
+    Nd->Ty = LHS->Ty;
+    return Nd;
+  }
+
+  // ptr - ptr，返回两指针间有多少元素
+  if (LHS->Ty->Base && RHS->Ty->Base) {
+    Node *Nd = newBinary(ND_SUB, LHS, RHS, Tok);
+    Nd->Ty = TyInt;
+    return newBinary(ND_DIV, Nd, newNum(8, Tok), Tok);
+  }
+
+  errorTok(Tok, "invalid operands");
+  return NULL;
+}
+
 // 解析加减
 // add = mul ("+" mul | "-" mul)*
 static Node *add(Token **Rest, Token *Tok) {
@@ -304,13 +365,13 @@ static Node *add(Token **Rest, Token *Tok) {
 
     // "+" mul
     if (equal(Tok, "+")) {
-      Nd = newBinary(ND_ADD, Nd, mul(&Tok, Tok->Next), Start);
+      Nd = newAdd(Nd, mul(&Tok, Tok->Next), Start);
       continue;
     }
 
     // "-" mul
     if (equal(Tok, "-")) {
-      Nd = newBinary(ND_SUB, Nd, mul(&Tok, Tok->Next), Start);
+      Nd = newSub(Nd, mul(&Tok, Tok->Next), Start);
       continue;
     }
```