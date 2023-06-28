我感觉这里定义函数应该是有限制的，具体限制是什么感觉课程没有说清楚，需要再看看 FIXME

rvcc.h

新增 Function 链表

```diff
@@ -68,9 +68,11 @@ struct Obj {
 // 函数
 typedef struct Function Function;
 struct Function {
-  Node *Body;    // 函数体
-  Obj *Locals;   // 本地变量
-  int StackSize; // 栈大小
+  Function *Next; // 下一函数
+  char *Name;     // 函数名
+  Node *Body;     // 函数体
+  Obj *Locals;    // 本地变量
+  int StackSize;  // 栈大小
 };
 
 // AST的节点种类
@@ -134,8 +136,9 @@ Function *parse(Token *Tok);
 
 // 类型种类
 typedef enum {
-  TY_INT, // int整型
-  TY_PTR, // 指针
+  TY_INT,  // int整型
+  TY_PTR,  // 指针
+  TY_FUNC, // 函数
 } TypeKind;
 
 struct Type {
@@ -146,6 +149,9 @@ struct Type {
 
   // 变量名
   Token *Name;
+
+  // 函数类型
+  Type *ReturnTy; // 函数返回的类型
 };
 
 // 声明一个全局变量，定义在type.c中。
@@ -157,6 +163,8 @@ bool isInteger(Type *TY);
 Type *pointerTo(Type *Base);
 // 为节点内的所有节点添加类型
 void addType(Node *Nd);
+// 函数类型
+Type *funcType(Type *ReturnTy);
 
 //
 // 语义分析与代码生成
```

type.c
```diff
@@ -14,6 +14,14 @@ Type *pointerTo(Type *Base) {
   return Ty;
 }
 
+// 函数类型，并赋返回类型
+Type *funcType(Type *ReturnTy) {
+  Type *Ty = calloc(1, sizeof(Type));
+  Ty->Kind = TY_FUNC;
+  Ty->ReturnTy = ReturnTy;
+  return Ty;
+}
+
 // 为节点内的所有节点添加类型
 void addType(Node *Nd) {
   // 判断 节点是否为空 或者 节点类型已经有值，那么就直接返回
```

parse.c

新增解析 functionDefinition

```diff
@@ -3,12 +3,15 @@
 // 在解析时，全部的变量实例都被累加到这个列表里。
 Obj *Locals;
 
-// program = "{" compoundStmt
+// program = functionDefinition*
+// functionDefinition = declspec declarator "{" compoundStmt*
+// declspec = "int"
+// declarator = "*"* ident typeSuffix
+// typeSuffix = ("(" ")")?
+
 // compoundStmt = (declaration | stmt)* "}"
 // declaration =
 //    declspec (declarator ("=" expr)? ("," declarator ("=" expr)?)*)? ";"
-// declspec = "int"
-// declarator = "*"* ident
 // stmt = "return" expr ";"
 //        | "if" "(" expr ")" stmt ("else" stmt)?
 //        | "for" "(" exprStmt expr? ";" expr? ")" stmt
@@ -26,8 +29,10 @@ Obj *Locals;
 // primary = "(" expr ")" | ident func-args? | num
 
 // funcall = ident "(" (assign ("," assign)*)? ")"
-static Node *compoundStmt(Token **Rest, Token *Tok);
+static Type *declspec(Token **Rest, Token *Tok);
+static Type *declarator(Token **Rest, Token *Tok, Type *Ty);
 static Node *declaration(Token **Rest, Token *Tok);
+static Node *compoundStmt(Token **Rest, Token *Tok);
 static Node *stmt(Token **Rest, Token *Tok);
 static Node *exprStmt(Token **Rest, Token *Tok);
 static Node *expr(Token **Rest, Token *Tok);
@@ -112,7 +117,18 @@ static Type *declspec(Token **Rest, Token *Tok) {
   return TyInt;
 }
 
-// declarator = "*"* ident
+// typeSuffix = ("(" ")")?
+static Type *typeSuffix(Token **Rest, Token *Tok, Type *Ty) {
+  // ("(" ")")?
+  if (equal(Tok, "(")) {
+    *Rest = skip(Tok->Next, ")");
+    return funcType(Ty);
+  }
+  *Rest = Tok;
+  return Ty;
+}
+
+// declarator = "*"* ident typeSuffix
 static Type *declarator(Token **Rest, Token *Tok, Type *Ty) {
   // "*"*
   // 构建所有的（多重）指针
@@ -122,10 +138,11 @@ static Type *declarator(Token **Rest, Token *Tok, Type *Ty) {
   if (Tok->Kind != TK_IDENT)
     errorTok(Tok, "expected a variable name");
 
+  // typeSuffix
+  Ty = typeSuffix(Rest, Tok->Next, Ty);
   // ident
-  // 变量名
+  // 变量名 或 函数名
   Ty->Name = Tok;
-  *Rest = Tok->Next;
   return Ty;
 }
 
@@ -581,15 +598,34 @@ static Node *primary(Token **Rest, Token *Tok) {
   return NULL;
 }
 
+// functionDefinition = declspec declarator "{" compoundStmt*
+static Function *function(Token **Rest, Token *Tok) {
+  // declspec
+  Type *Ty = declspec(&Tok, Tok);
+  // declarator? ident "(" ")"
+  Ty = declarator(&Tok, Tok, Ty);
+
+  // 清空全局变量Locals
+  Locals = NULL;
+
+  // 从解析完成的Ty中读取ident
+  Function *Fn = calloc(1, sizeof(Function));
+  Fn->Name = getIdent(Ty->Name);
+
+  Tok = skip(Tok, "{");
+  // 函数体存储语句的AST，Locals存储变量
+  Fn->Body = compoundStmt(Rest, Tok);
+  Fn->Locals = Locals;
+  return Fn;
+}
+
 // 语法解析入口函数
-// program = "{" compoundStmt
+// program = functionDefinition*
 Function *parse(Token *Tok) {
-  // "{"
-  Tok = skip(Tok, "{");
+  Function Head = {};
+  Function *Cur = &Head;
 
-  // 函数体存储语句的AST，Locals存储变量
-  Function *Prog = calloc(1, sizeof(Function));
-  Prog->Body = compoundStmt(&Tok, Tok);
-  Prog->Locals = Locals;
-  return Prog;
+  while (Tok->Kind != TK_EOF)
+    Cur = Cur->Next = function(&Tok, Tok);
+  return Head.Next;
 }
```

codegen.c
```diff
@@ -4,6 +4,8 @@
 static int Depth;
 // 用于函数参数的寄存器们
 static char *ArgReg[] = {"a0", "a1", "a2", "a3", "a4", "a5"};
+// 当前的函数
+static Function *CurrentFn;
 
 static void genExpr(Node *Nd);
 
@@ -273,8 +275,8 @@ static void genStmt(Node *Nd) {
     genExpr(Nd->LHS);
     // 无条件跳转语句，跳转到.L.return段
     // j offset是 jal x0, offset的别名指令
-    printf("  # 跳转到.L.return段\n");
-    printf("  j .L.return\n");
+    printf("  # 跳转到.L.return.%s段\n", CurrentFn->Name);
+    printf("  j .L.return.%s\n", CurrentFn->Name);
     return;
   // 生成表达式语句
   case ND_EXPR_STMT:
@@ -289,75 +291,83 @@ static void genStmt(Node *Nd) {
 
 // 根据变量的链表计算出偏移量
 static void assignLVarOffsets(Function *Prog) {
-  int Offset = 0;
-  // 读取所有变量
-  for (Obj *Var = Prog->Locals; Var; Var = Var->Next) {
-    // 每个变量分配8字节
-    Offset += 8;
-    // 为每个变量赋一个偏移量，或者说是栈中地址
-    Var->Offset = -Offset;
+  // 为每个函数计算其变量所用的栈空间
+  for (Function *Fn = Prog; Fn; Fn = Fn->Next) {
+    int Offset = 0;
+    // 读取所有变量
+    for (Obj *Var = Fn->Locals; Var; Var = Var->Next) {
+      // 每个变量分配8字节
+      Offset += 8;
+      // 为每个变量赋一个偏移量，或者说是栈中地址
+      Var->Offset = -Offset;
+    }
+    // 将栈对齐到16字节
+    Fn->StackSize = alignTo(Offset, 16);
   }
-  // 将栈对齐到16字节
-  Prog->StackSize = alignTo(Offset, 16);
 }
 
 // 代码生成入口函数，包含代码块的基础信息
 void codegen(Function *Prog) {
   assignLVarOffsets(Prog);
-  printf("  # 定义全局main段\n");
-  printf("  .globl main\n");
-  printf("\n# =====程序开始===============\n");
-  printf("# main段标签，也是程序入口段\n");
-  printf("main:\n");
 
-  // 栈布局
-  //-------------------------------// sp
-  //              ra
-  //-------------------------------// ra = sp-8
-  //              fp
-  //-------------------------------// fp = sp-16
-  //             变量
-  //-------------------------------// sp = sp-16-StackSize
-  //           表达式计算
-  //-------------------------------//
+  // 为每个函数单独生成代码
+  for (Function *Fn = Prog; Fn; Fn = Fn->Next) {
+    printf("\n  # 定义全局%s段\n", Fn->Name);
......
```