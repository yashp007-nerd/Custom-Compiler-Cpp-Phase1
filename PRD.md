# Product Requirement Document (PRD)

**Project Name:** Micro-Compiler — Phase 1: Lexical Analysis Engine (Modern C++14 Edition)  
**Repository Reference:** `yashp007-nerd-custom-compiler-cpp-phase1`  
**Document Version:** 1.0.0-PROD  
**Target Release:** Phase 1 Milestone  
**Author:** Principal Product Manager & Technical Architect  

---

## 1. Executive Summary & Objectives

### 1.1 Project Name & High-Level Overview
The **Micro-Compiler** project is a lightweight, high-performance compiler pipeline designed to transform mathematical expressions and source programs into executable structures. **Phase 1: Lexical Analysis (Lexer)** establishes the foundational front-end scanner. 

This phase represents an architectural refactoring from legacy procedural C (`lexer.c` and `lexer.h`) into modern, object-oriented, memory-safe ISO C++14. The Lexer consumes raw source character streams, strips non-semantic whitespace, flags syntax violations, and deterministically produces typed token streams with precise source byte offsets.

### 1.2 Core Problem Statement
Legacy implementations in procedural C suffered from several critical maintainability, safety, and compatibility challenges:
* **Memory Safety & Buffer Overflows:** Fixed-width character buffers (such as `char value[64]`) combined with manual copying (`strncpy`) created severe buffer overflow vulnerabilities when encountering large literals.
* **Manual Lifecycle Management:** Manual heap allocations (`malloc`, `realloc`, and custom free functions like `lexer_free_tokens()`) introduced risks of memory leaks and double-free exceptions.
* **Global Namespace Collisions:** Preprocessor definitions such as standard library `EOF` from `<cstdio>` collided directly with compiler end-of-file token definitions.
* **Toolchain Fragility:** Modern C++17 features (e.g., `std::string_view`) break compatibility on constrained or legacy development environments, such as older MinGW installations running GCC 6+ on Windows.

Phase 1 resolves these issues by delivering an encapsulated, RAII-compliant, non-allocating-leak tokenizer that strictly adheres to the ISO C++14 standard.

### 1.3 Business Goals & Success Metrics (KPIs)
* **Zero Memory Leaks:** 100% dynamic memory safety guaranteed via C++ standard containers (`std::string`, `std::vector`) and RAII.
* **Strict Toolchain Compatibility:** Flawless compilation under GCC 6+ on both POSIX and MinGW/Windows environments using `-Wall -Wextra -Wpedantic -std=c++14`.
* **Linear Time Complexity:** Deterministic single-pass lexical analysis operating in $O(N)$ time complexity relative to the source length $N$.
* **Test Reliability:** 100% pass rate across the comprehensive regression test suite (25/25 passing unit test suites across edge cases, operators, expressions, whitespace, and illegal character handling).

---

## 2. User Personas & User Journeys

### 2.1 Target Audience & Roles
* **Parser / Compiler Engineer (Internal Consumer):** Core engineering consumer integrating Phase 1 with the upcoming Phase 2 Syntactic Parser (AST Builder). Requires deterministic token ordering, precise byte offsets, and sentinel termination.
* **Systems & Tooling Developer:** Developer building CLI linters, syntax highlighters, or diagnostic utilities relying on stream-based or batch tokenization.
* **Build / CI Automation Engineer:** Engineer maintaining cross-platform build pipelines and verification suites via GNU Make and headless test runners.

### 2.2 Core User Journeys

```
+-----------------------------------------------------------------------------------+
| JOURNEY 1: Downstream Parser Streaming Token Consumption                          |
|                                                                                   |
|  [Source Code] ---> [Lexer Instance] ---> call nextToken() ---> [Token Produced]  |
|                             ^                                           |         |
|                             |------------- Repeat until EOF ------------+         |
+-----------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------+
| JOURNEY 2: Batch Analysis / Diagnostic Ingestion                                  |
|                                                                                   |
|  [Source Code] ---> Lexer::tokenize(source) ---> std::vector<Token> ---> Output   |
+-----------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------+
| JOURNEY 3: Error Detection & Recovery Flow                                       |
|                                                                                   |
|  [Illegal Char: '$'] ---> Log to stderr (ASCII + Hex) ---> Skip & Tail Recurse    |
|                                                                     |             |
|                                [Emit Next Valid Token: INT] <-------+             |
+-----------------------------------------------------------------------------------+
```

* **Journey 1: Streaming Lexical Analysis (Parser Integration)**
  1. Downstream parser initializes `Lexer` by passing the source string.
  2. The parser enters its parsing loop and invokes `lexer.nextToken()`.
  3. The lexer skips leading whitespace, identifies the lexeme, and returns a concrete `Token` by value.
  4. The parser inspects `Token.type` and advances until `TokenType::END_OF_FILE` is encountered.

* **Journey 2: Batch Inspection and Tooling Ingestion**
  1. A tooling utility invokes the static convenience method `Lexer::tokenize(source)`.
  2. The method internally constructs a lexer, pre-allocates vector storage, drains the token stream until `END_OF_FILE`, and returns an owned `std::vector<Token>` with zero memory leaks via Move Semantics / Named Return Value Optimization (NRVO).

* **Journey 3: Malformed Syntax Diagnosis**
  1. The lexer encounters an unmapped byte (e.g., ASCII `$` or control characters).
  2. The lexer writes a formatted error to `std::cerr` containing the verbatim character, its two-digit hexadecimal representation, and its absolute byte position.
  3. The scanner skips the invalid character and continues scanning to return the next valid token without crashing the engine.

---

## 3. Functional Requirements

### 3.1 Module Breakdown
* **Module 1: Type System & Token Definitions (`TokenType`, `Token`)**
* **Module 2: Core Scanning & Cursor Mechanics (`Lexer`)**
* **Module 3: Numeric Literal Extraction**
* **Module 4: Arithmetic Operator Identification**
* **Module 5: Whitespace Normalization**
* **Module 6: Lexical Diagnostic Subsystem**
* **Module 7: Batch API & Interop Interface**

### 3.2 Traceability Matrix & Detailed Requirements

| Req ID | Module | Feature Name | Description | User Story | Priority |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FR-01** | Module 1 | Scoped Token Enumeration | System must define `TokenType` as an `enum class` containing `INT`, `PLUS`, `MINUS`, `MULT`, `DIV`, `END_OF_FILE`, and `ILLEGAL` to prevent macro/namespace pollution. | As a Compiler Engineer, I need strongly-typed token classifications so that tokens do not collide with C standard macros like `EOF`. | **P0** |
| **FR-02** | Module 1 | Dynamic Token Container | `Token` struct must encapsulate `TokenType type`, `std::string value`, and `int position`. Fixed-width character buffers are strictly prohibited. | As an Engineer, I need arbitrary-length lexemes stored safely without buffer overrun risks. | **P0** |
| **FR-03** | Module 1 | Token Stringification | System must provide `tokenTypeName(TokenType)` returning human-readable C-string representations of token types. | As a Tooling Developer, I want string names of tokens for logging and diagnostic dumps. | **P1** |
| **FR-04** | Module 2 | Cursor Encapsulation | `Lexer` must encapsulate input string copy `m_source`, current read index `m_pos`, and input boundary `m_length`. | As an Architect, I need the internal scanning state fully shielded from external modification. | **P0** |
| **FR-05** | Module 2 | Lookahead / Peeking | Lexer must implement `currentChar()` returning the character at `m_pos`, or null-terminator `'\0'` when `m_pos >= m_length` without out-of-bounds indexing. | As the Lexer Engine, I need safe 1-character lookahead to branch parsing logic. | **P0** |
| **FR-06** | Module 2 | Cursor Advancement | Lexer must implement `advance()` to safely increment `m_pos` strictly within bounds (`m_pos < m_length`). | As the Lexer Engine, I need controlled position advancement across the input buffer. | **P0** |
| **FR-07** | Module 3 | Multi-Digit Integer Lexing | When `currentChar()` matches `std::isdigit`, the lexer must aggregate contiguous ASCII digits into a single `TokenType::INT` token via `scanNumber()`. | As an End User, I want integer literals (e.g., `42`, `12345`) recognized as single integer tokens with accurate starting offsets. | **P0** |
| **FR-08** | Module 4 | Single-Character Operator Lexing | The lexer must map `+`, `-`, `*`, and `/` to `TokenType::PLUS`, `TokenType::MINUS`, `TokenType::MULT`, and `TokenType::DIV` respectively, advancing the cursor by 1 byte. | As a Downstream Parser, I need arithmetic operators categorized distinctly. | **P0** |
| **FR-09** | Module 5 | Whitespace Stripping | `skipWhitespace()` must discard ASCII space (`0x20`) and horizontal tab (`0x09`) characters without emitting tokens. | As a Parser, I need insignificant whitespace ignored so expression formatting does not affect syntax analysis. | **P0** |
| **FR-10** | Module 2 | Stream Termination Sentinel | When the cursor reaches or exceeds `m_length`, the lexer must emit a sentinel `Token` of type `TokenType::END_OF_FILE`, with an empty value (`""`) and `position == m_length`. | As a Downstream Parser, I need a deterministic stream terminator to conclude parsing loops. | **P0** |
| **FR-11** | Module 6 | Illegal Character Handling | Upon encountering an unmapped byte, the lexer must write an error message to `std::cerr` formatted as `[Lexer Error] Illegal character '<c>' (0x<HEX>) at position <pos> — skipping.` and tail-recurse to return the next valid token. | As a Developer, I need clear error reports for invalid characters while allowing the scanner to recover gracefully. | **P1** |
| **FR-12** | Module 7 | Static Batch Tokenization | `Lexer::tokenize(const std::string&)` must return a `std::vector<Token>` containing all scanned tokens up to and including the terminal `END_OF_FILE`. | As an Integration Client, I need a one-line call to convert a source string into an array of tokens. | **P0** |
| **FR-13** | Module 1 | Aligned Console Output | `Token::print()` must print aligned representations matching: `Token { type: <type:-12>, value: "<val:-10>", position: <pos> }`. | As a CLI User, I need formatted tabular output for token verification. | **P2** |

---

## 4. Technical Architecture & System Integration

### 4.1 System Topology & Class Diagrams

```
+-------------------------------------------------------------------------+
|                              TokenType                                  |
|-------------------------------------------------------------------------|
| + INT                                                                   |
| + PLUS                                                                  |
| + MINUS                                                                 |
| + MULT                                                                  |
| + DIV                                                                   |
| + END_OF_FILE                                                           |
| + ILLEGAL                                                               |
+-------------------------------------------------------------------------+
                                     ^
                                     | uses
+------------------------------------+------------------------------------+
|                               Token                                     |
|-------------------------------------------------------------------------|
| + type     : TokenType                                                  |
| + value    : std::string                                                |
| + position : int                                                        |
|-------------------------------------------------------------------------|
| + print() const : void                                                  |
+-------------------------------------------------------------------------+
                                     ^
                                     | produces
+------------------------------------+------------------------------------+
|                               Lexer                                     |
|-------------------------------------------------------------------------|
| - m_source : std::string                                                |
| - m_pos    : int                                                        |
| - m_length : int                                                        |
|-------------------------------------------------------------------------|
| + explicit Lexer(const std::string& source)                             |
| + nextToken() : Token                                                   |
| + static tokenize(const std::string& source) : std::vector<Token>       |
| - currentChar() const noexcept : char                                   |
| - advance() noexcept : void                                             |
| - skipWhitespace() noexcept : void                                      |
| - scanNumber() : Token                                                  |
+-------------------------------------------------------------------------+
```

### 4.2 Scanner Finite State Machine (FSM) Transition Diagram

```
                 +-------------------+
                 |    START STATE    |
                 +-------------------+
                           |
            +--------------+--------------+
            |                             |
      [isspace()]                   [currentChar()]
            |                             |
            v                             |
     +--------------+                     |
     | skipWhite-   |                     |
     |    space()   |                     |
     +--------------+                     |
            |                             |
            +------------>+<--------------+
                          |
      +-------------------+-------------------+-------------------+
      |                   |                   |                   |
 [m_pos>=m_length]    [isdigit()]     ['+','-','*','/']      [default]
      |                   |                   |                   |
      v                   v                   v                   v
+------------+     +-------------+     +-------------+     +-------------+
| Emit EOF   |     | scanNumber  |     | Emit OP     |     | Log Error   |
| Token      |     | (Loop digits|     | advance()   |     | to stderr   |
+------------+     +-------------+     +-------------+     +-------------+
                          |                                       |
                          v                                       v
                   +-------------+                         +-------------+
                   |  Emit INT   |                         | Tail-Recurse|
                   |    Token    |                         | nextToken() |
                   +-------------+                         +-------------+
```

### 4.3 Data Entities & Memory Layout
* **`Token` Structure:**
  * `TokenType type`: 4-byte enum class representation.
  * `std::string value`: Dynamic string holding the exact character lexeme.
  * `int position`: 4-byte signed integer storing the 0-indexed byte offset from the start of the source string.
* **`Lexer` Structure:**
  * `std::string m_source`: Complete copy of the input string ensuring reference validity over the instance lifecycle.
  * `int m_pos`: Current scanning index.
  * `int m_length`: Cached byte size of `m_source` (`static_cast<int>(source.size())`).

### 4.4 Header Specification Contract (`Lexer.hpp`)
```cpp
#ifndef LEXER_HPP
#define LEXER_HPP

#include <string>
#include <vector>

enum class TokenType {
    INT,
    PLUS,
    MINUS,
    MULT,
    DIV,
    END_OF_FILE,
    ILLEGAL
};

const char* tokenTypeName(TokenType type) noexcept;

struct Token {
    TokenType   type;
    std::string value;
    int         position;

    void print() const;
};

class Lexer {
public:
    explicit Lexer(const std::string& source);
    Token nextToken();
    static std::vector<Token> tokenize(const std::string& source);

private:
    std::string m_source;
    int         m_pos;
    int         m_length;

    char currentChar() const noexcept;
    void advance() noexcept;
    void skipWhitespace() noexcept;
    Token scanNumber();
};

#endif // LEXER_HPP
```

### 4.5 Dependencies & Build Toolchain
* **Compiler Target:** ISO C++14 standard compliant.
* **Toolchain Compatibility:** GCC 6.x+ via MinGW (Windows) or native GCC (Linux/POSIX).
* **Flags:** `-Wall -Wextra -Wpedantic -std=c++14 -g`.
* **Third-Party Libraries:** Zero third-party runtime or testing dependencies. Relies exclusively on ISO C++ standard headers (`<cctype>`, `<iomanip>`, `<iostream>`, `<cstdio>`, `<string>`, `<vector>`, `<functional>`).
* **Build Targets (Makefile):**
  * `make` / `make all`: Compiles `main_cpp.exe` from `Lexer.cpp` and `main.cpp`.
  * `make run`: Builds and executes the demo binary.
  * `make test`: Compiles `test_lexer.exe` from `Lexer.cpp` and `test_lexer.cpp` and runs the test harness.
  * `make clean`: Removes binary artifacts (`main_cpp.exe`, `test_lexer.exe`).

---

## 5. Non-Functional Requirements (NFRs)

### 5.1 Performance & Complexity
* **Time Complexity:** The scanning process must strictly run in $O(N)$ linear time where $N$ is the length of `m_source`. Each character is inspected at most twice (once during peeking/dispatch and once during advance/number scanning).
* **Space Complexity:** Dynamic memory overhead for token generation must be bounded. `Lexer::tokenize` shall pre-reserve an initial capacity of at least 32 tokens (`tokens.reserve(32)`) to eliminate dynamic reallocations for standard expressions.
* **Return Semantics:** Lexer outputs must support NRVO and C++11 move semantics during token collection returns to guarantee zero redundant heap copies.

### 5.2 Reliability & Safety
* **Zero Pointer Arithmetic:** Elimination of raw character pointers (`char*`) and manual pointer arithmetic in public interfaces to prevent use-after-free and dangling pointer bugs.
* **Safe Character Casting:** All character inspection operations involving `<cctype>` functions (e.g., `std::isdigit`) must explicitly cast characters to `unsigned char` (`static_cast<unsigned char>(c)`) to prevent undefined behavior with negative char values on platforms where `char` is signed.
* **Hex Formatting Safety:** Use of `std::snprintf` with a fixed-length stack buffer (`char hexBuf[8]`) when formatting hex dumps for illegal characters to eliminate format string vulnerabilities.

### 5.3 Portability & Maintainability
* **Standard Conformance:** Code must cleanly compile under `-std=c++14` without vendor-specific extensions.
* **Language Feature Restraint:** Explicit avoidance of C++17 library additions (such as `std::string_view` or `<filesystem>`) to maintain backward compatibility with legacy MinGW toolchains.
* **Self-Contained Verification:** The test runner must execute autonomously without external testing frameworks (such as GoogleTest or Catch2), enabling automated testing in minimal shell environments.

---

## 6. Wireframes, UI/UX, & Interface Layouts

While the Lexer is a non-visual systems component, it defines strict standard I/O interface formatting for diagnostic output and developer verification.

### 6.1 Standard Console Diagnostic Formatting (`Token::print()`)
Token streaming and debugging require strict column-aligned formatting:
* `type`: Left-aligned in a 12-character field (`std::left << std::setw(12)`).
* `value`: Quoted string, left-aligned in a 10-character field (`std::setw(10)`).
* `position`: Trailing integer.

```
Token { type: INT         , value: "15"       , position: 0 }
Token { type: PLUS        , value: "+"        , position: 3 }
Token { type: INT         , value: "3"        , position: 5 }
Token { type: END_OF_FILE , value: ""         , position: 6 }
```

### 6.2 Standard Error Diagnostic Output (`std::cerr`)
Illegal characters generate structured diagnostic warnings to standard error:
```
[Lexer Error] Illegal character '<CHAR>' (0x<HEX>) at position <POS> — skipping.
```

#### Example Output:
```
[Lexer Error] Illegal character '$' (0x24) at position 3 — skipping.
[Lexer Error] Illegal character '@' (0x40) at position 0 — skipping.
```

### 6.3 Test Suite CLI Dashboard (`test_lexer.cpp`)
Execution of `make test` outputs categorized, color-independent status reports:

```
=============================================================
  Micro-Compiler — Phase 1 Lexer  |  Unit Tests
=============================================================

[ Edge Cases ]
  [PASS] Empty source
  [PASS] Single integer
  [PASS] Single digit
  [PASS] Multi-digit integer

[ Operators ]
  [PASS] Plus operator
  [PASS] Minus operator
  [PASS] Multiply operator
  [PASS] Divide operator

[ Expressions ]
  [PASS] Simple addition
  [PASS] Subtraction
  [PASS] Multiplication
  [PASS] Division
  [PASS] All four operators

[ Whitespace Handling ]
  [PASS] Leading whitespace
  [PASS] Trailing whitespace
  [PASS] Multiple spaces
  [PASS] Tab whitespace
  [PASS] Whitespace-only source

[ Illegal Characters ]
  [PASS] Illegal char skipped
  [PASS] Multiple illegal chars
  [PASS] Illegal between valid

[ Correctness ]
  [PASS] Token positions
  [PASS] EOF value is empty
  [PASS] tokenTypeName() strings
  [PASS] nextToken() == tokenize()

=============================================================
  Results: 25 / 25 tests passed  ✓  ALL PASSED
=============================================================
```

---

## 7. Risks, Constraints, & Future Phases

### 7.1 Identified Technical Risks & Bottlenecks
1. **Stack Overflow via Tail-Recursion on Malformed Inputs:**
   * *Risk:* In `Lexer::nextToken()`, encountering an illegal character triggers a recursive call: `return nextToken();`. In unoptimized debug builds (`-O0` or `-g`), if an input file contains tens of thousands of consecutive illegal characters, this can exceed the call stack depth and trigger a stack overflow.
   * *Remediation Strategy for Senior Eng:* Replace the tail-recursive call with an iterative loop (`while (true)`) enclosing the token dispatch.
2. **Memory Footprint of Input Source Copies:**
   * *Risk:* `Lexer` constructor accepts `const std::string& source` and stores an owned copy in `m_source`. For massive source files, this doubles memory consumption.
   * *Remediation Strategy:* Maintain this design for Phase 1 to guarantee lifetime safety without raw pointers; plan a migration to a non-owning zero-copy abstraction in Phase 2.
3. **Single-Byte ASCII Assumption:**
   * *Risk:* `currentChar()` treats characters as individual 8-bit `char` types. Multi-byte UTF-8 sequences will fail validation and get processed as multiple consecutive illegal bytes.
   * *Mitigation:* Document ASCII arithmetic expressions as a strict constraint for Phase 1.

### 7.2 Assumptions & Scope Boundaries

#### In-Scope (Phase 1):
* Tokenizing integers (`TokenType::INT`).
* Tokenizing basic arithmetic operators (`+`, `-`, `*`, `/`).
* Skipping spaces (`0x20`) and tabs (`\t` / `0x09`).
* 1D byte-offset position tracking.
* Unit test coverage and automated compilation harness.

#### Out-of-Scope (Explicitly Deferred):
* Floating-point numbers, hexadecimal, or scientific notation literals.
* Parentheses (`(`, `)`) and grouping operators.
* Variables, identifiers, and keyword recognition (`let`, `fn`, `if`, etc.).
* Negative number tokenization at the lexer level (unary minus is deferred to the parser to avoid ambiguity with binary subtraction).
* 2-Dimensional source tracking (Line numbers and Column numbers; current implementation tracks 1D absolute byte offset).
* Abstract Syntax Tree (AST) construction and evaluation.

### 7.3 Phase 2 & Future Roadmap
* **Phase 2 — Syntactic Analysis (Parser):** Implement a Recursive Descent or Pratt Parser consuming `Lexer::nextToken()` to build an Abstract Syntax Tree (AST) supporting operator precedence ($*$, $/$ over $+$, $-$).
* **Source Location Tracking:** Upgrade `Token::position` to a structured `SourceLocation` containing `line`, `column`, and `byteOffset`.
* **C++17 Migration Path:** Transition `m_source` and `Token::value` to `std::string_view` when baseline compiler requirements can be safely elevated across all build environments, achieving zero allocations during scanning.
* **Extended Grammar:** Introduce tokens for parentheses (`TokenType::LPAREN`, `TokenType::RPAREN`), identifiers (`TokenType::IDENT`), and floating-point literals.

---

## 8. Acceptance Criteria for Engineering Sign-Off

Before handing this specification off to the Senior Engineering team for Phase 2 integration, the implementation must meet the following criteria:
1. `make clean && make && make test` executes cleanly with zero warnings under GCC 6+ (`-Wall -Wextra -Wpedantic -std=c++14`).
2. All 25 regression test cases in `test_lexer.cpp` output `[PASS]` and the test harness terminates with exit code `0`.
3. Execution of `main_cpp.exe` processes edge cases (empty strings, all four operators, whitespace variations, and illegal characters) with exact expected formatting and diagnostic reporting on `std::cerr`.
4. Zero static analysis warnings related to memory leaks, dangling references, or uninitialized members.