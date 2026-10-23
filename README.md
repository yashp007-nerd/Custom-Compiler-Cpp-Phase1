# Micro-Compiler — Phase 1: Lexical Analysis Engine

A lightweight, robust, and zero-leak lexical analysis engine (scanner) implemented in modern **C++14**. 

This repository marks **Phase 1** of the Micro-Compiler project, refactoring a legacy procedural C lexer into a clean, object-oriented, RAII-compliant C++ implementation engineered for predictable execution, cross-platform compatibility, and integration with downstream syntactic parsers (AST builders).

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Grammar & Supported Lexemes](#grammar--supported-lexemes)
- [Architecture & Design Decisions](#architecture--design-decisions)
- [Project Structure](#project-structure)
- [Prerequisites & Toolchain](#prerequisites--toolchain)
- [Building and Running](#building-and-running)
- [API Reference & Usage](#api-reference--usage)
- [Testing & Verification](#testing--verification)
- [Phase 2 Roadmap](#phase-2-roadmap)
- [License](#license)

---

## Overview

The Phase 1 Lexer operates as a deterministic, single-pass character scanner that translates raw mathematical expression strings into typed `Token` sequences. It maintains strict $O(N)$ linear time complexity, performs automatic whitespace normalization, provides clear byte-offset diagnostics for malformed inputs, and manages memory safely without raw pointer arithmetic.

---

## Key Features

- **Memory Safe & Zero Leaks:** Replaces legacy fixed-width character buffers (`char[64]`) and manual dynamic allocations (`malloc`/`free`) with standard C++ abstractions (`std::string`, `std::vector`) governed by RAII.
- **Cross-Platform Toolchain Compatibility:** Targets **ISO C++14** (GCC 6+) without dependencies on C++17 features (e.g., `std::string_view`), guaranteeing seamless builds on legacy Windows/MinGW environments as well as POSIX Linux distributions.
- **Macro & Namespace Isolation:** Employs scoped enumerations (`enum class TokenType`) to eliminate global identifier collisions with standard library macros such as `<cstdio>`'s `EOF`.
- **Dual Ingestion Paradigms:**
  - **Streaming Mode (`nextToken()`):** On-demand iterative consumption designed for downstream recursive-descent parsers.
  - **Batch Mode (`Lexer::tokenize()`):** Static one-pass processing optimized with Named Return Value Optimization (NRVO) and vector capacity reservation.
- **Fault-Tolerant Error Handling:** Unmapped characters are reported to `std::cerr` with both character literal and 2-digit uppercase hexadecimal byte representations, followed by tail-recursive scanning recovery.
- **Zero-Dependency Test Suite:** Built-in test harness with 25 test cases covering operators, edge cases, whitespace variations, and syntax violations.

---

## Grammar & Supported Lexemes

Phase 1 focuses on lexical tokenization of fundamental integer arithmetic:

| Lexeme Pattern | Token Type (`enum class`) | Description | Example |
| :--- | :--- | :--- | :--- |
| `[0-9]+` | `TokenType::INT` | Contiguous sequence of ASCII decimal digits | `0`, `42`, `12345` |
| `+` | `TokenType::PLUS` | Addition operator | `+` |
| `-` | `TokenType::MINUS` | Subtraction operator | `-` |
| `*` | `TokenType::MULT` | Multiplication operator | `*` |
| `/` | `TokenType::DIV` | Division operator | `/` |
| `\0` (EOF) | `TokenType::END_OF_FILE` | End-of-input terminal sentinel | `""` (at `pos == len`) |
| Unmapped bytes | `TokenType::ILLEGAL` | Logged to `stderr` and skipped | `$`, `@`, `#`, `%` |

*Note: Insignificant whitespace (ASCII space `0x20` and horizontal tab `\t` / `0x09`) is stripped automatically by the scanner.*

---

## Architecture & Design Decisions

### 1. Data Structures & Layout

```
                  +-----------------------------------------+
                  |               TokenType                 |
                  |-----------------------------------------|
                  | INT, PLUS, MINUS, MULT, DIV, EOF, ...   |
                  +-----------------------------------------+
                                       ^
                                       |
                  +--------------------+--------------------+
                  |                 Token                   |
                  |-----------------------------------------|
                  | + type     : TokenType                  |
                  | + value    : std::string                |
                  | + position : int                        |
                  |-----------------------------------------|
                  | + print() const : void                  |
                  +-----------------------------------------+
                                       ^
                                       | returns
                  +--------------------+--------------------+
                  |                 Lexer                   |
                  |-----------------------------------------|
                  | - m_source : std::string                |
                  | - m_pos    : int                        |
                  | - m_length : int                        |
                  |-----------------------------------------|
                  | + nextToken() : Token                   |
                  | + static tokenize(src) : vector<Token>  |
                  +-----------------------------------------+
```

### 2. Design Rationale

- **Owned Input Copy (`std::string m_source`):** The lexer stores a direct copy of the input string during construction. This prevents dangling reference hazards when token streams outlive temporary string arguments passed by callers.
- **Pre-allocation Optimization:** `Lexer::tokenize()` invokes `tokens.reserve(32)` to avoid repeated dynamic heap reallocations when tokenizing standard arithmetic expressions.
- **Signed Character Safety:** Direct calls to `<cctype>` functions (such as `std::isdigit`) explicitly cast characters via `static_cast<unsigned char>(c)` to avoid undefined behavior on platforms where native `char` is signed and negative values can occur.

---

## Project Structure

```
yashp007-nerd-custom-compiler-cpp-phase1/
├── Makefile            # Unified build definitions (GCC/g++, targets for demo & tests)
├── Lexer.hpp           # Header: TokenType enum class, Token struct, Lexer interface
├── Lexer.cpp           # Implementation: scanning logic, number parsing, error logging
├── main.cpp            # Demonstration CLI entry point with sample expressions
└── test_lexer.cpp      # Standalone 25-case regression test harness
```

---

## Prerequisites & Toolchain

- **Compiler:** `g++` (GCC 6.0+ recommended) with ISO C++14 support.
- **Build Tool:** GNU Make.
- **Supported Environments:**
  - Windows (via MinGW, MSYS2, or Cygwin).
  - Linux / POSIX systems.
  - macOS (via AppleClang or Homebrew GCC).

Verify your compiler supports C++14 prior to compiling:
```bash
g++ --version
```

---

## Building and Running

A structured `Makefile` is provided with targets for compilation, execution, testing, and cleanup.

### 1. Build the Demo Binary
Compiles the main interactive demonstration executable (`main_cpp.exe`):
```bash
make
```

### 2. Run the Demo
Builds (if required) and executes the demo suite:
```bash
make run
```

#### Demo Output Sample:
```text
=============================================================
  Micro-Compiler — Phase 1 Lexer  |  C++ Demo
=============================================================

--- Simple addition ---
Source: "15 + 3"

Token { type: INT         , value: "15"       , position: 0 }
Token { type: PLUS        , value: "+"        , position: 3 }
Token { type: INT         , value: "3"        , position: 5 }
Token { type: END_OF_FILE , value: ""         , position: 6 }

--- Illegal character '$' ---
Source: "10 $ 5"

[Lexer Error] Illegal character '$' (0x24) at position 3 — skipping.
Token { type: INT         , value: "10"       , position: 0 }
Token { type: INT         , value: "5"        , position: 5 }
Token { type: END_OF_FILE , value: ""         , position: 6 }
```

### 3. Run the Unit Test Suite
Builds and executes the test harness binary (`test_lexer.exe`):
```bash
make test
```

### 4. Clean Build Artifacts
Removes all compiled `.exe` files and build objects:
```bash
make clean
```

---

## API Reference & Usage

### 1. Streaming Tokenization (Parser Integration)
Ideal for memory-constrained environments or streaming syntactic analysis:

```cpp
#include "Lexer.hpp"
#include <iostream>

int main() {
    std::string source = "100 - 45 * 2";
    Lexer lexer(source);

    while (true) {
        Token tok = lexer.nextToken();
        tok.print();

        if (tok.type == TokenType::END_OF_FILE) {
            break;
        }
    }
    return 0;
}
```

### 2. Batch Tokenization (One-Pass / Diagnostic Dumps)
Ideal for tooling, static analysis, or immediate vector processing:

```cpp
#include "Lexer.hpp"
#include <vector>

int main() {
    std::string source = "42 + 7";
    std::vector<Token> tokens = Lexer::tokenize(source);

    for (const auto& token : tokens) {
        std::cout << "Token at position " << token.position 
                  << " has type " << tokenTypeName(token.type) 
                  << " with lexeme '" << token.value << "'\n";
    }
    return 0;
}
```

---

## Testing & Verification

The suite in `test_lexer.cpp` implements a zero-dependency test runner. It returns exit code `0` on total success and `1` on failure, making it ready for integration into CI/CD pipelines.

The 25 test cases evaluate six key functional domains:

1. **Edge Cases:** Empty source strings, single-digit integers, multi-digit integers.
2. **Operators:** Individual assertion of `+`, `-`, `*`, `/`.
3. **Compound Expressions:** Standard addition, subtraction, multiplication, division, and mixed expressions without spaces (e.g., `1+2-3*4/5`).
4. **Whitespace Handling:** Leading whitespace, trailing whitespace, multi-space delimiters, tab characters (`\t`), and pure whitespace sources.
5. **Illegal Characters:** Single illegal character recovery, consecutive illegal characters (`@#!`), and illegal characters embedded within valid expressions (`3 % 4`).
6. **Correctness & Invariants:** Verification that byte offsets are accurate, EOF contains an empty value, string conversion names match, and `nextToken()` matches batch `tokenize()`.

```text
=============================================================
  Results: 25 / 25 tests passed  ✓  ALL PASSED
=============================================================
```

---

## Phase 2 Roadmap

With Phase 1 complete and verified, the next milestones in the micro-compiler pipeline include:

- [ ] **Phase 2 — Syntactic Analysis (Parser):**
  - Implement an Abstract Syntax Tree (AST) node hierarchy (`BinaryExprAST`, `NumberExprAST`).
  - Implement a Recursive Descent or Pratt Parser to handle operator precedence ($*$ and $/$ evaluated before $+$ and $-$).
- [ ] **2D Source Location:** Transition from 1D byte-offset position tracking to structured `SourceLocation { size_t line; size_t column; }`.
- [ ] **Extended Syntax:**
  - Grouping parentheses (`(`, `)`).
  - Floating-point numeric literals (`3.14159`).
  - Variables and assignment syntax (`let x = 10`).
- [ ] **C++17 Upgrades:** Conditionally introduce `std::string_view` for zero-allocation scanning on modern toolchains.

---

## License

This project is open-source and released under the [MIT License](LICENSE).
