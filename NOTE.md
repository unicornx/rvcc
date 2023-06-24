rvcc.h

重点：
- Obj 增加一个属性是 Ty，用于设置一个变量的类型

```diff
@@ -45,6 +45,7 @@ void errorTok(Token *Tok, char *Fmt, ...);
 // 判断Token与Str的关系
 bool equal(Token *Tok, char *Str);
 Token *skip(Token *Tok, char *Str);
+bool consume(Token **Rest, Token *Tok, char *Str);
 // 词法分析
 Token *tokenize(char *Input);
 
@@ -60,6 +61,7 @@ typedef struct Obj Obj;
 struct Obj {
   Obj *Next;  // 指向下一对象
   char *Name; // 变量名
+  Type *Ty;   // 变量类型
   int Offset; // fp的偏移量
 };
 
@@ -133,7 +135,12 @@ typedef enum {
 
 struct Type {
   TypeKind Kind; // 种类
-  Type *Base;    // 指向的类型
+
+  // 指针
+  Type *Base; // 指向的类型
+
+  // 变量名？
+  Token *Name;
 };
 
 // 声明一个全局变量，定义在type.c中。
@@ -141,6 +148,8 @@ extern Type *TyInt;
 
 // 判断是否为整型
 bool isInteger(Type *TY);
+// 构建一个指针类型，并指向基类
+Type *pointerTo(Type *Base);
 // 为节点内的所有节点添加类型
 void addType(Node *Nd);
```

type.c
```diff
@@ -48,20 +48,22 @@ void addType(Node *Nd) {
   case ND_NE:
   case ND_LT:
   case ND_LE:
-  case ND_VAR:
   case ND_NUM:
     Nd->Ty = TyInt;
     return;
+  // 将节点类型设为 变量的类型
+  case ND_VAR:
+    Nd->Ty = Nd->Var->Ty;
+    return;
   // 将节点类型设为 指针，并指向左部的类型
   case ND_ADDR:
     Nd->Ty = pointerTo(Nd->LHS->Ty);
     return;
-  // 节点类型：如果解引用指向的是指针，则为指针指向的类型；否则为int
+  // 节点类型：如果解引用指向的是指针，则为指针指向的类型；否则报错
   case ND_DEREF:
-    if (Nd->LHS->Ty->Kind == TY_PTR)
-      Nd->Ty = Nd->LHS->Ty->Base;
-    else
-      Nd->Ty = TyInt;
+    if (Nd->LHS->Ty->Kind != TY_PTR)
+      errorTok(Nd->Tok, "invalid pointer dereference");
+    Nd->Ty = Nd->LHS->Ty->Base;
     return;
   default:
     break;
```

tokenize.c
```diff
@@ -68,6 +68,18 @@ Token *skip(Token *Tok, char *Str) {
   return Tok->Next;
 }
 
+// 消耗掉指定Token
+bool consume(Token **Rest, Token *Tok, char *Str) {
+  // 存在
+  if (equal(Tok, Str)) {
+    *Rest = Tok->Next;
+    return true;
+  }
+  // 不存在
+  *Rest = Tok;
+  return false;
+}
+
 // 返回TK_NUM的值
 static int getNumber(Token *Tok) {
   if (Tok->Kind != TK_NUM)
@@ -116,7 +128,7 @@ static int readPunct(char *Ptr) {
 // 判断是否为关键字
 static bool isKeyword(Token *Tok) {
   // 关键字列表
-  static char *Kw[] = {"return", "if", "else", "for", "while"};
+  static char *Kw[] = {"return", "if", "else", "for", "while", "int"};
 
   // 遍历关键字列表匹配
   for (int I = 0; I < sizeof(Kw) / sizeof(*Kw); ++I) {
```

parse.c
语法分析部分改动较大，主要是增加了对 declaration 的处理

```diff
@@ -4,7 +4,11 @@
 Obj *Locals;
 
 // program = "{" compoundStmt
-// compoundStmt = stmt* "}"
+// compoundStmt = (declaration | stmt)* "}"
+// declaration =
+//    declspec (declarator ("=" expr)? ("," declarator ("=" expr)?)*)? ";"
+// declspec = "int"
+// declarator = "*"* ident
 // stmt = "return" expr ";"
 //        | "if" "(" expr ")" stmt ("else" stmt)?
 //        | "for" "(" exprStmt expr? ";" expr? ")" stmt
@@ -21,6 +25,7 @@ Obj *Locals;
 // unary = ("+" | "-" | "*" | "&") unary | primary
 // primary = "(" expr ")" | ident | num
 static Node *compoundStmt(Token **Rest, Token *Tok);
+static Node *declaration(Token **Rest, Token *Tok);
 static Node *stmt(Token **Rest, Token *Tok);
 static Node *exprStmt(Token **Rest, Token *Tok);
 static Node *expr(Token **Rest, Token *Tok);
@@ -81,15 +86,91 @@ static Node *newVarNode(Obj *Var, Token *Tok) {
 }
 
 // 在链表中新增一个变量
-static Obj *newLVar(char *Name) {
+static Obj *newLVar(char *Name, Type *Ty) {
   Obj *Var = calloc(1, sizeof(Obj));
   Var->Name = Name;
+  Var->Ty = Ty;
   // 将变量插入头部
   Var->Next = Locals;
   Locals = Var;
   return Var;
 }
 
+// 获取标识符
+static char *getIdent(Token *Tok) {
+  if (Tok->Kind != TK_IDENT)
+    errorTok(Tok, "expected an identifier");
+  return strndup(Tok->Loc, Tok->Len);
+}
+
+// declspec = "int"
+// declarator specifier
+static Type *declspec(Token **Rest, Token *Tok) {
+  *Rest = skip(Tok, "int");
+  return TyInt;
+}
+
+// declarator = "*"* ident
+static Type *declarator(Token **Rest, Token *Tok, Type *Ty) {
+  // "*"*
+  // 构建所有的（多重）指针
+  while (consume(&Tok, Tok, "*"))
+    Ty = pointerTo(Ty);
+
+  if (Tok->Kind != TK_IDENT)
+    errorTok(Tok, "expected a variable name");
+
+  // ident
+  // 变量名
+  Ty->Name = Tok;
+  *Rest = Tok->Next;
+  return Ty;
+}
+
+// declaration =
+//    declspec (declarator ("=" expr)? ("," declarator ("=" expr)?)*)? ";"
+static Node *declaration(Token **Rest, Token *Tok) {
+  // declspec
+  // 声明的 基础类型
+  Type *Basety = declspec(&Tok, Tok);
+
+  Node Head = {};
+  Node *Cur = &Head;
+  // 对变量声明次数计数
+  int I = 0;
+
+  // (declarator ("=" expr)? ("," declarator ("=" expr)?)*)?
+  while (!equal(Tok, ";")) {
+    // 第1个变量不必匹配 ","
+    if (I++ > 0)
+      Tok = skip(Tok, ",");
+
+    // declarator
+    // 声明获取到变量类型，包括变量名
+    Type *Ty = declarator(&Tok, Tok, Basety);
+    Obj *Var = newLVar(getIdent(Ty->Name), Ty);
+
+    // 如果不存在"="则为变量声明，不需要生成节点，已经存储在Locals中了
+    if (!equal(Tok, "="))
+      continue;
+
+    // 解析“=”后面的Token
+    Node *LHS = newVarNode(Var, Ty->Name);
+    // 解析递归赋值语句
+    Node *RHS = assign(&Tok, Tok->Next);
+    Node *Node = newBinary(ND_ASSIGN, LHS, RHS, Tok);
+    // 存放在表达式语句中
+    Cur->Next = newUnary(ND_EXPR_STMT, Node, Tok);
+    Cur = Cur->Next;
+  }
+
+  // 将所有表达式语句，存放在代码块中
+  Node *Nd = newNode(ND_BLOCK, Tok);
+  Nd->Body = Head.Next;
+  *Rest = Tok->Next;
+  return Nd;
+}
+
 // 解析语句
 // stmt = "return" expr ";"
 //        | "if" "(" expr ")" stmt ("else" stmt)?
@@ -172,16 +253,21 @@ static Node *stmt(Token **Rest, Token *Tok) {
 }
 
 // 解析复合语句
-// compoundStmt = stmt* "}"
+// compoundStmt = (declaration | stmt)* "}"
 static Node *compoundStmt(Token **Rest, Token *Tok) {
   Node *Nd = newNode(ND_BLOCK, Tok);
 
   // 这里使用了和词法分析类似的单向链表结构
   Node Head = {};
   Node *Cur = &Head;
-  // stmt* "}"
+  // (declaration | stmt)* "}"
   while (!equal(Tok, "}")) {
-    Cur->Next = stmt(&Tok, Tok);
+    // declaration
+    if (equal(Tok, "int"))
+      Cur->Next = declaration(&Tok, Tok);
+    // stmt
+    else
+      Cur->Next = stmt(&Tok, Tok);
     Cur = Cur->Next;
     // 构造完AST后，为节点添加类型信息
     addType(Cur);
@@ -446,8 +532,7 @@ static Node *primary(Token **Rest, Token *Tok) {
     Obj *Var = findVar(Tok);
     // 如果变量不存在，就在链表中新增一个变量
     if (!Var)
-      // strndup复制N个字符
-      Var = newLVar(strndup(Tok->Loc, Tok->Len));
+      errorTok(Tok, "undefined variable");
     *Rest = Tok->Next;
     return newVarNode(Var, Tok);
   }
```