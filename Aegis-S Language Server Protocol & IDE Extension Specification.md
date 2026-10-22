# Aegis-S Language Server Protocol (LSP) & IDE Extension Specification (`aegis-lsp`)

## 1. Overview of Real-Time IDE Security
Traditional language servers (like rust-analyzer or gopls) focus on syntax autocomplete, type checking, and quick refactoring. In contrast, the **`aegis-lsp`** server integrates active SMT proof checking, lexical entropy tracking, and background `RedTeam-NN` fuzzing directly into the developer's editing loop. 

If a developer writes code that violates formal pre-conditions (`requires`) or introduces potential side-channel leaks, the LSP underlines the code with security diagnostics before the file is even saved or compiled.

---

## 2. Core Diagnostic Channels

### A. Real-Time SMT Proof Status
As the AST updates in memory, background Z3 solver threads evaluate local function contracts. 
* **Green Underline:** Contract invariant is formally proven.
* **Red Strikethrough / Underline:** Unproven verification condition or potential bound violation detected. Clicking the diagnostic suggests valid mathematical adjustments.

### B. Adversarial Vulnerability Warnings
The embedded `RedTeam-NN` engine continuously scans isolated blocks (`isolate`) for non-constant time execution patterns or unmasked state variables, displaying predictive attack vectors right in the editor's "Problems" tab.

---

## 3. VS Code Extension Manifest (`package.json` skeleton)

Below is the configuration structure for the official Aegis-S Visual Studio Code extension.

```json
{
  "name": "aegis-s-vscode",
  "displayName": "Aegis-S Security-First Language Support",
  "version": "0.1.0",
  "publisher": "AegisSecurity",
  "engines": {
    "vscode": "^1.80.0"
  },
  "activationEvents": [
    "onLanguage:aegis"
  ],
  "contributes": {
    "languages": [{
      "id": "aegis",
      "aliases": ["Aegis-S", "aegis"],
      "extensions": [".aegis"],
      "configuration": "./language-configuration.json"
    }],
    "grammars": [{
      "language": "aegis",
      "scopeName": "source.aegis",
      "path": "./syntaxes/aegis.tmLanguage.json"
    }]
  }
}
```

---

## 4. LSP Server Protocol Flow (`aegis-lsp`)

```
[ VS Code / Neovim Editor ] 
         │ (textDocument/didChange)
         ▼
[ Aegis-LSP Daemon (`aegis-lsp`) ]
         │
         ├──► 1. Incremental Lexical Entropy Check
         ├──► 2. Incremental SMT Z3 Solver Evaluation
         └──► 3. Background Red-Team Neural Net Scan
         │
         ▼ (publishDiagnostics)
[ Real-Time Security & Proof Annotations ]
```

### Protocol Method Implementation (Rust Skeleton)
```rust
// aegis_lsp_server.rs - LSP diagnostic emitter for aegis-lsp

pub struct Diagnostic {
    pub range: (usize, usize),
    pub severity: LSPErrorLevel,
    pub message: String,
}

pub enum LSPErrorLevel {
    Error = 1,
    Warning = 2,
    Information = 3,
}

pub struct AegisLspServer;

impl AegisLspServer {
    pub fn analyze_document(&self, source_content: &str) -> Vec<Diagnostic> {
        let mut diagnostics = Vec::new();

        // Check for unverified isolate blocks
        if source_content.contains("isolate") && !source_content.contains("requires") {
            diagnostics.push(Diagnostic {
                range: (0, 10),
                severity: LSPErrorLevel::Error,
                message: String::from("[Aegis-LSP] Isolated execution block lacks formal precondition contract bounds."),
            });
        }

        diagnostics
    }
}
```