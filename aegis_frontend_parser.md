# Aegis-S Frontend Parser & Lexer Specification and Implementation

## 1. Introduction to the Frontend Pipeline
This document describes a proposed grammar and includes an illustrative Rust lexer/parser skeleton. The code only tokenizes a subset of the syntax and parses a module declaration; it is not a complete or validated compiler frontend. Security properties must be enforced and tested in later compiler stages, not assumed from parsing alone.

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

The following is a draft EBNF grammar for the proposed syntax. The Rust skeleton later in this document supports only a subset of it.

```ebnf
Program ::= ModuleDecl ImportDecl* Item*

ModuleDecl ::= "module" Identifier ";"

ImportDecl ::= "import" Identifier ("." Identifier)* ";"

Item ::= StructDef | FuncDef | EnclaveDef

EnclaveDef ::= "enclave" Identifier Block

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

LetStmt ::= "let" Identifier ( ":" TypeRef )? "=" Expression ";"

AssignStmt ::= Identifier AssignOp Expression ";"

AssignOp ::= "=" | "+=" | "-=" | "*=" | "/="

IsolateStmt ::= "isolate" Block

ReturnStmt ::= "return" Expression ";"

ExprStmt ::= Expression ";"

Expression ::= Term (( "+" | "-" | "==" | "!=" | ">=" | "<=" ) Term)*

Term ::= Identifier ( "." Identifier )* | Literal | OldExpr | CallExpr

OldExpr ::= "old" "(" Expression ")"

CallExpr ::= Identifier ( "::" Identifier )* "(" ( Expression ( "," Expression )* )? ")"

Identifier ::= [a-zA-Z_][a-zA-Z0-9_]*

Literal ::= [0-9]+ ("." [0-9]+)?
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

## 5. Lexer & Parser Implementation Skeleton

The following Rust example is a partial skeleton. It supports tokenizing a subset of the grammar and parsing only a module declaration; it does not parse structures, functions, contracts, or statements.

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Token {
	Module, Import, Secure, Struct, Func, Pub, Mut, Requires, Ensures, Isolate, Let, Return,
	Identifier(String), Literal(String), Arrow, Colon, Semicolon, Comma, Equal, EqEq, NotEq,
	GreaterEq, LessEq, Plus, Minus, Star, Slash, LBrace, RBrace, LParen, RParen, LT, GT, Dot, EOF,
}

pub struct Lexer {
	input: Vec<char>,
	position: usize,
}

impl Lexer {
	pub fn new(input: &str) -> Self {
		Self { input: input.chars().collect(), position: 0 }
	}

	pub fn next_token(&mut self) -> Token {
		self.skip_whitespace();
		if self.position >= self.input.len() {
			return Token::EOF;
		}

		let current = self.input[self.position];
		if current.is_alphabetic() || current == '_' {
			let start = self.position;
			while self.position < self.input.len()
				&& (self.input[self.position].is_alphanumeric() || self.input[self.position] == '_')
			{
				self.position += 1;
			}
			let identifier: String = self.input[start..self.position].iter().collect();
			return match identifier.as_str() {
				"module" => Token::Module,
				"import" => Token::Import,
				"secure" => Token::Secure,
				"struct" => Token::Struct,
				"func" => Token::Func,
				"pub" => Token::Pub,
				"mut" => Token::Mut,
				"requires" => Token::Requires,
				"ensures" => Token::Ensures,
				"isolate" => Token::Isolate,
				"let" => Token::Let,
				"return" => Token::Return,
				_ => Token::Identifier(identifier),
			};
		}

		if current.is_ascii_digit() {
			let start = self.position;
			while self.position < self.input.len()
				&& (self.input[self.position].is_ascii_digit() || self.input[self.position] == '_')
			{
				self.position += 1;
			}
			if self.position + 1 < self.input.len()
				&& self.input[self.position] == '.'
				&& self.input[self.position + 1].is_ascii_digit()
			{
				self.position += 1;
				while self.position < self.input.len()
					&& (self.input[self.position].is_ascii_digit() || self.input[self.position] == '_')
				{
					self.position += 1;
				}
			}
			return Token::Literal(self.input[start..self.position].iter().collect());
		}

		self.position += 1;
		match current {
			':' => Token::Colon,
			';' => Token::Semicolon,
			',' => Token::Comma,
			'{' => Token::LBrace,
			'}' => Token::RBrace,
			'(' => Token::LParen,
			')' => Token::RParen,
			'+' => Token::Plus,
			'-' if self.input.get(self.position) == Some(&'>') => {
				self.position += 1;
				Token::Arrow
			}
			'-' => Token::Minus,
			'*' => Token::Star,
			'/' => Token::Slash,
			'=' if self.input.get(self.position) == Some(&'=') => {
				self.position += 1;
				Token::EqEq
			}
			'=' => Token::Equal,
			'!' if self.input.get(self.position) == Some(&'=') => {
				self.position += 1;
				Token::NotEq
			}
			'>' if self.input.get(self.position) == Some(&'=') => {
				self.position += 1;
				Token::GreaterEq
			}
			'>' => Token::GT,
			'<' if self.input.get(self.position) == Some(&'=') => {
				self.position += 1;
				Token::LessEq
			}
			'<' => Token::LT,
			'.' => Token::Dot,
			_ => panic!("Unknown character: {}", current),
		}
	}

	fn skip_whitespace(&mut self) {
		loop {
			while self.position < self.input.len() && self.input[self.position].is_whitespace() {
				self.position += 1;
			}
			if self.position + 1 < self.input.len()
				&& self.input[self.position] == '/'
				&& self.input[self.position + 1] == '/'
			{
				while self.position < self.input.len() && self.input[self.position] != '\n' {
					self.position += 1;
				}
			} else {
				break;
			}
		}
	}
}

pub struct Parser {
	lexer: Lexer,
	current_token: Token,
}

impl Parser {
	pub fn new(mut lexer: Lexer) -> Self {
		let current_token = lexer.next_token();
		Self { lexer, current_token }
	}

	pub fn consume(&mut self) {
		self.current_token = self.lexer.next_token();
	}

	pub fn parse_module_decl(&mut self) {
		assert_eq!(self.current_token, Token::Module);
		self.consume();
		if let Token::Identifier(name) = &self.current_token {
			println!("Parsed Module: {}", name);
			self.consume();
		} else {
			panic!("Expected module identifier");
		}
		assert_eq!(self.current_token, Token::Semicolon);
		self.consume();
	}
}
```