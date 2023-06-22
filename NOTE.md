rvcc.h
这里我们需要注意的是，for 循环其实是 if-else 的一种扩展，所以在 Node 中 for 语句复用了 if-else
的一部分成员。讲解时可以考虑一下如何将 for 和 if-else 进行对比体现出相互之间的关系。

另外可以用流程图画一下，可能更形象


```diff
@@ -84,6 +84,7 @@ typedef enum {
   ND_ASSIGN,    // 赋值
   ND_RETURN,    // 返回
   ND_IF,        // "if"，条件判断
+  ND_FOR,       // "for"，循环
   ND_BLOCK,     // { ... }，代码块
   ND_EXPR_STMT, // 表达式语句
   ND_VAR,       // 变量
@@ -98,10 +99,12 @@ struct Node {
   Node *LHS; // 左部，left-hand side
   Node *RHS; // 右部，right-hand side
 
-  // "if"语句
+  // "if"语句 或者 "for"语句
   Node *Cond; // 条件内的表达式
   Node *Then; // 符合条件后的语句
   Node *Els;  // 不符合条件后的语句
+  Node *Init; // 初始化语句
+  Node *Inc;  // 递增语句
 
   // 代码块
   Node *Body;
```

词法分析没有什么

parse.c
```diff
@@ -7,6 +7,7 @@ Obj *Locals;
 // compoundStmt = stmt* "}"
 // stmt = "return" expr ";"
 //        | "if" "(" expr ")" stmt ("else" stmt)?
+//        | "for" "(" exprStmt expr? ";" expr? ")" stmt
 //        | "{" compoundStmt
 //        | exprStmt
 // exprStmt = expr? ";"
@@ -89,6 +90,7 @@ static Obj *newLVar(char *Name) {
 // 解析语句
 // stmt = "return" expr ";"
 //        | "if" "(" expr ")" stmt ("else" stmt)?
+//        | "for" "(" exprStmt expr? ";" expr? ")" stmt
 //        | "{" compoundStmt
 //        | exprStmt
 static Node *stmt(Token **Rest, Token *Tok) {
@@ -116,6 +118,32 @@ static Node *stmt(Token **Rest, Token *Tok) {
     return Nd;
   }
 
+  // "for" "(" exprStmt expr? ";" expr? ")" stmt
+  if (equal(Tok, "for")) {
+    Node *Nd = newNode(ND_FOR);
+    // "("
+    Tok = skip(Tok->Next, "(");
+
+    // exprStmt
+    Nd->Init = exprStmt(&Tok, Tok);
+
+    // expr?
+    if (!equal(Tok, ";"))
+      Nd->Cond = expr(&Tok, Tok);
+    // ";"
+    Tok = skip(Tok, ";");
+
+    // expr?
+    if (!equal(Tok, ")"))
+      Nd->Inc = expr(&Tok, Tok);
+    // ")"
+    Tok = skip(Tok, ")");
+
+    // stmt
+    Nd->Then = stmt(Rest, Tok);
+    return Nd;
+  }
+
   // "{" compoundStmt
   if (equal(Tok, "{"))
     return compoundStmt(Rest, Tok->Next);
```

codegen.c
和 if-else 相比多了 .L.begin.N 这个 tag，体现了循环的概念，还会回来到 begin 处。

```diff
@@ -158,7 +158,33 @@ static void genStmt(Node *Nd) {
     printf(".L.end.%d:\n", C);
     return;
   }
-
+  // 生成for循环语句
+  case ND_FOR: {
+    // 代码段计数
+    int C = count();
+    // 生成初始化语句
+    genStmt(Nd->Init);
+    // 输出循环头部标签
+    printf(".L.begin.%d:\n", C);
+    // 处理循环条件语句
+    if (Nd->Cond) {
+      // 生成条件循环语句
+      genExpr(Nd->Cond);
+      // 判断结果是否为0，为0则跳转到结束部分
+      printf("  beqz a0, .L.end.%d\n", C);
+    }
+    // 生成循环体语句
+    genStmt(Nd->Then);
+    // 处理循环递增语句
+    if (Nd->Inc)
+      // 生成循环递增语句
+      genExpr(Nd->Inc);
+    // 跳转到循环头部
+    printf("  j .L.begin.%d\n", C);
+    // 输出循环尾部标签
+    printf(".L.end.%d:\n", C);
+    return;
+  }
   // 生成代码块，遍历代码块的语句链表
   case ND_BLOCK:
     for (Node *N = Nd->Body; N; N = N->Next)
```