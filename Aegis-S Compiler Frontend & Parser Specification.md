# Aegis-S Compiler Frontend & Parser Specification (`aegis-c` Parser)

## 1. Introduction to the Frontend Pipeline
The `aegis-c` frontend is engineered to parse high-security syntax while enforcing strict cryptographic and mathematical invariants at the syntax tree level. Unlike standard compilers that treat keywords as static strings, the Aegis-S lexer and parser incorporate **Lexical Entropy Mapping**, ensuring that variable bindings, memory qualifiers (`mut`, `Protected`), and formal contracts (`requires`, `ensures`) are rigidly verified before semantic lowering.

---

## 2. Lexer Token Definitions

The lexer scans the raw `.aegis` source code stream and tokenizes it while applying dynamic token shifting to prevent predictable pattern analysis by external tools.

### Core Token Categories
* **Keywords:** `module`, `import`, `secure`, `struct`, `enclave`, `pub`, `func`, `requires`, `ensures`, `isolate`, `let`, `return`
* **Type Qualifiers:** `<Encrypted>`, `<Protected>`, `<ZeroizeOnDrop>`
* **Logical Operators:** `==`, `!=`, `>=`, `<=`, `&&`, `||`, `+`, `-`, `*`, `/`
* **Contract Literals:** `old(...)` expression wrappers for pre/post-state comparisons.

---

## 3. Extended Backus-Naur Form (EBNF) Grammar

Below is the formal EBNF grammar governing the Aegis-S language syntax, specifically focusing on secure structures, functions, and formal contract clauses.

```ebnf
Program ::= ModuleDecl ImportDecl* Item*

ModuleDecl ::= "module" Identifier ";"

ImportDecl ::= "import" Identifier ("." Identifier)* ";"

Item ::= StructDef | FuncDef | EnclaveDef

StructDef ::= "secure" "struct" Identifier "{" FieldDecl ( "," FieldDecl )* "}"

FieldDecl ::= Identifier ":" TypeRef

TypeRef ::= Identifier ("<" TypeArg ( "," TypeArg )* ">")? ( ".." Expression )?

TypeArg ::= Identifier | Literal

FuncDef ::= "pub"? "func" Identifier "(" ParamList? ")" "->" TypeRef ContractClause* Block

ParamList ::= Param ( "," Param )*

Param ::= "mut"? Identifier ":" TypeRef

ContractClause ::= RequiresClause | EnsuresClause

RequiresClause ::= "requires" Expression

EnsuresClause ::= "ensures" Expression

Block ::= "{" Statement* "}"

Statement ::= LetStmt | AssignStmt | IsolateStmt | ReturnStmt | ExprStmt

LetStmt ::= "let" Identifier ":" TypeRef "=" Expression ";"

AssignStmt ::= Identifier AssignOp Expression ";"

AssignOp ::= "=" | "+=" | "-=" | "*=" | "/="

IsolateStmt ::= "isolate" Block

ReturnStmt ::= "return" Expression ";"

ExprStmt ::= Expression ";"

Expression ::= Term (( "+" | "-" | "==" | "!=" | ">=" | "<=" ) Term)*

Term ::= Identifier | Literal | OldExpr | CallExpr

OldExpr ::= "old" "(" Expression ")"

CallExpr ::= Identifier "(" ( Expression ( "," Expression )* )? ")"

Identifier ::= [a-zA-Z_][a-zA-Z0-9_]*

Literal ::= [0-9]+ ("." [0-9]+)? | StringLiteral
```

---

## 4. SMT Contract Binding & AST Translation

When the parser encounters `requires` and `ensures` clauses, it does not treat them as comments or runtime assertions. Instead, it translates them directly into SMT-LIB2 logic formulations that are fed into embedded solvers (such as Z3) during Stage 2 compilation.

### Example Translation Rule
For a post-condition contract:
$$\text{ensures } s.\text{balance} == \text{old}(s.\text{balance}) - \text{amount}$$

The parser emits the following first-order logic assertion to the SMT solver:
$$\forall t_1, t_2: \text{State} \implies (\text{balance}(t_2) = \text{balance}(t_1) - \text{amount})$$

If the solver finds any counterexample where this equation does not hold true under all execution branches, the compiler terminates immediately with a formal proof failure.