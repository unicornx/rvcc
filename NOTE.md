rvcc.h
```diff
@@ -83,6 +83,7 @@ typedef enum {
   ND_LE,        // <=
   ND_ASSIGN,    // 赋值
   ND_RETURN,    // 返回
+  ND_IF,        // "if"，条件判断
   ND_BLOCK,     // { ... }，代码块
   ND_EXPR_STMT, // 表达式语句
   ND_VAR,       // 变量
@@ -93,8 +94,14 @@ typedef enum {
 struct Node {
   NodeKind Kind; // 节点种类
   Node *Next;    // 下一节点，指代下一语句
-  Node *LHS;     // 左部，left-hand side
-  Node *RHS;     // 右部，right-hand side
+
+  Node *LHS; // 左部，left-hand side
+  Node *RHS; // 右部，right-hand side
+
+  // "if"语句
+  Node *Cond; // 条件内的表达式
+  Node *Then; // 符合条件后的语句
+  Node *Els;  // 不符合条件后的语句
 
   // 代码块
   Node *Body;
```

tokenize.c
```diff
@@ -113,10 +113,24 @@ static int readPunct(char *Ptr) {
   return ispunct(*Ptr) ? 1 : 0;
 }
 
+// 判断是否为关键字
+static bool isKeyword(Token *Tok) {
+  // 关键字列表
+  static char *Kw[] = {"return", "if", "else"};
+
+  // 遍历关键字列表匹配
+  for (int I = 0; I < sizeof(Kw) / sizeof(*Kw); ++I) {
+    if (equal(Tok, Kw[I]))
+      return true;
+  }
+
+  return false;
+}
+
 // 将名为“return”的终结符转为KEYWORD
 static void convertKeywords(Token *Tok) {
   for (Token *T = Tok; T->Kind != TK_EOF; T = T->Next) {
-    if (equal(T, "return"))
+    if (isKeyword(T))
       T->Kind = TK_KEYWORD;
   }
 }
```

parse.c
```diff
index 7af2ffa..748a34a 100644
--- a/parse.c
+++ b/parse.c
@@ -5,7 +5,10 @@ Obj *Locals;
 
 // program = "{" compoundStmt
 // compoundStmt = stmt* "}"
-// stmt = "return" expr ";" | "{" compoundStmt | exprStmt
+// stmt = "return" expr ";"
+//        | "if" "(" expr ")" stmt ("else" stmt)?
+//        | "{" compoundStmt
+//        | exprStmt
 // exprStmt = expr? ";"
 // expr = assign
 // assign = equality ("=" assign)?
@@ -84,7 +87,10 @@ static Obj *newLVar(char *Name) {
 }
 
 // 解析语句
-// stmt = "return" expr ";" | "{" compoundStmt | exprStmt
+// stmt = "return" expr ";"
+//        | "if" "(" expr ")" stmt ("else" stmt)?
+//        | "{" compoundStmt
+//        | exprStmt
 static Node *stmt(Token **Rest, Token *Tok) {
   // "return" expr ";"
   if (equal(Tok, "return")) {
@@ -93,6 +99,23 @@ static Node *stmt(Token **Rest, Token *Tok) {
     return Nd;
   }
 
+  // 解析if语句
+  // "if" "(" expr ")" stmt ("else" stmt)?
+  if (equal(Tok, "if")) {
+    Node *Nd = newNode(ND_IF);
+    // "(" expr ")"，条件内语句
+    Tok = skip(Tok->Next, "(");
+    Nd->Cond = expr(&Tok, Tok);
+    Tok = skip(Tok, ")");
+    // stmt，符合条件后的语句
+    Nd->Then = stmt(&Tok, Tok);
+    // ("else" stmt)?，不符合条件后的语句
+    if (equal(Tok, "else"))
+      Nd->Els = stmt(&Tok, Tok->Next);
+    *Rest = Tok;
+    return Nd;
+  }
+
   // "{" compoundStmt
   if (equal(Tok, "{"))
     return compoundStmt(Rest, Tok->Next);
```

codegen.c
这里注意在 codegen 阶段会为 if else 跳转的位置设置 tag
N 为行号
.L.else.N
.L.end.N
这里注意 tag 的作用，属于 block 内部的 tag

```diff
@@ -3,6 +3,12 @@
 // 记录栈深度
 static int Depth;
 
+// 代码段计数
+static int count(void) {
+  static int I = 1;
+  return I++;
+}
+
 // 压栈，将结果临时压入栈中备用
 // sp为栈指针，栈反向向下增长，64位下，8个字节为一个单位，所以sp-8
 // 当前栈指针的地址就是sp，将a0的值压入栈
@@ -131,6 +137,28 @@ static void genExpr(Node *Nd) {
 // 生成语句
 static void genStmt(Node *Nd) {
   switch (Nd->Kind) {
+  // 生成if语句
+  case ND_IF: {
+    // 代码段计数
+    int C = count();
+    // 生成条件内语句
+    genExpr(Nd->Cond);
+    // 判断结果是否为0，为0则跳转到else标签
+    printf("  beqz a0, .L.else.%d\n", C);
+    // 生成符合条件后的语句
+    genStmt(Nd->Then);
+    // 执行完后跳转到if语句后面的语句
+    printf("  j .L.end.%d\n", C);
+    // else代码块，else可能为空，故输出标签
+    printf(".L.else.%d:\n", C);
+    // 生成不符合条件后的语句
+    if (Nd->Els)
+      genStmt(Nd->Els);
+    // 结束if语句，继续执行后面的语句
+    printf(".L.end.%d:\n", C);
+    return;
+  }
+
   // 生成代码块，遍历代码块的语句链表
   case ND_BLOCK:
     for (Node *N = Nd->Body; N; N = N->Next)
```