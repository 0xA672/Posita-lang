# Posita Compiler Implementation Guide
**Status:** Working draft — intended for compiler contributors.
**Last updated:** 2026-06-11

## 1. Overview

This document outlines the architecture, technology choices, and development roadmap for the Posita compiler. It is a living guide, evolving alongside the language specification.

### Core principles for the compiler
- **Correctness first**: The compiler must faithfully implement the Posita safety guarantees.
- **Compilation speed is a feature**: Posita’s “compile-time first” philosophy means the compiler itself must be fast and predictable.
- **Modular and testable**: Each stage should be independently testable with clear input/output contracts.
- **Written in Rust**: The compiler is implemented in Rust, leveraging its strong type system and existing ecosystem for compilers (Cranelift, etc.).

## 2. Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Language** | Rust | Safety, performance, strong ecosystem for compiler construction, seamless Cranelift integration. |
| **Lexer** | `logos` | Fast, zero-allocation lexer generator with great error messages. |
| **Parser** | Hand-written recursive descent | Full control over error recovery and diagnostics; Posita grammar is designed to be LL(1) with minimal lookahead. |
| **IRs** | Custom HIR, MIR (Rust structs) | HIR preserves source-level structure; MIR is a typed, SSA-like mid-level representation. |
| **Backend** | Cranelift (primary), LLVM (optional future) | Cranelift for fast AOT/JIT compilation; LLVM as an optional high-optimization backend. |
| **SMT Solver** | Z3 via `z3-rs` crate | For contract verification in strict mode. |
| **Incremental computation** | `salsa` | For fast recompilation and IDE support. |
| **CLI** | `clap` | Standard argument parsing. |
| **Testing** | `cargo test`, snapshot tests, property-based testing | Comprehensive verification of each compiler stage. |

## 3. Compiler Pipeline

```
Source file (.ps)
    │
    ▼
Lexer (logos) ──► Token stream
    │
    ▼
Parser (hand-written) ──► Concrete Syntax Tree (CST) / AST
    │
    ▼
AstLowering ──► High-level IR (HIR) with resolved names and types
    │
    ▼
Semantic Analysis (type checking, borrow checking, contract expansion)
    │
    ▼
MirLowering ──► Mid-level IR (MIR) ──► Optional: Contract verification (SMT)
    │
    ▼
Codegen ──► Cranelift IR (CLIF) ──► Object file (.o)
    │
    ▼
Linker (system ld) ──► Executable
```

All stages maintain diagnostic information (source locations, spans) for error reporting. The compiler supports both single-file and whole-project compilation.

## 4. Implementation Stages

### Stage 0: Bootstrapping the Compiler Shell
- Set up Rust project with `clap` CLI.
- Implement basic lexer using `logos` (keywords, operators, literals, comments).
- Implement a minimal parser that can handle a “hello world” program: `def main() { }`.
- Integrate Cranelift to produce an executable for this minimal program.
- **Goal**: `posita run hello.ps` compiles and runs a trivial program.

### Stage 1: Core Type System and Expressions
- Implement AST nodes for all primitive types: `Int<N>`, `UInt<N>`, `Bool`, `Float`, etc.
- Implement integer literals with bit-width inference (`set x = 42` → `Int<32>`).
- Basic arithmetic, bitwise, logical operators, with overflow control.
- Variable declarations (`set`, `set mut`), assignment.
- Control flow: `if`, `else`, `while`, `for`, `loop`, `leave`, `continue`, labels.
- Function definitions and calls, `return`.
- Simple modules: file-based, `import` and `from ... import`.
- **Goal**: Can compile and run non-trivial algorithms (e.g., fib, sorting) with integers and arrays.

### Stage 2: Error Handling and Basic Contracts
- Implement `Result<T,E>`, `?` operator, `catch` blocks, `leave with`.
- Implement `finally` blocks and enforce their restrictions.
- `From` trait for automatic error conversion.
- Runtime contract checking for `requires` and `ensures` (no SMT yet).
- Type-level defaults (`with default`, `no_default`).
- **Goal**: Robust error handling works; programs can use `?` and `catch`.

### Stage 3: Advanced Type System Features
- Generic functions and types with `where` clauses.
- Traits (similar to Rust traits) for `Add`, `Display`, etc.
- `constraint` blocks.
- Type factories (`comptime def` returning `type`).
- `@typeInfo` compile-time reflection (limited to type structure).
- `auto[<T>..]` type capture.
- Layout control attributes (`@packed`, `@endian`, etc.).
- **Goal**: Expressive type-level programming possible, standard library can begin.

### Stage 4: Compile-Time Execution and Full Contracts
- `comptime` blocks and functions with budget limits.
- Cycle detection and safe termination.
- Static contract verification using Z3 (strict mode).
- `@trusted` and `@experimental` attributes.
- `invariant` on types and loops (verification hooks).
- **Goal**: Strict Mode can verify simple contracts automatically.

### Stage 5: Concurrency and System Programming
- Tasks and channels.
- `async`/`await` with executor.
- `extern "C"` FFI with contractual wrappers.
- `unsafe` blocks with full checks on boundaries.
- Interrupt handling (`@interrupt`).
- **Goal**: Real-time and concurrent programs compile and run.

### Stage 6: Tooling and Ecosystem
- LSP server for IDE support.
- Package manager (`capsa`).
- Debugger integration (DWARF support).
- Documentation generator.
- **Goal**: Posita is usable for production projects.

## 5. Key Design Decisions for the Compiler

### 5.1 Memory Management
Posita does not use a garbage collector. The compiler will implement:
- **Copy semantics** by default for `Copy` types.
- **Move semantics** for non-`Copy` types, with the help of a **borrow checker** (similar to Rust’s NLL). The MIR will contain explicit `move` and `copy` operations. The compiler will track ownership and insert drops at the end of scopes.

### 5.2 Integer Representation
`Int<N>` and `UInt<N>` are mapped to Cranelift’s `iN` types where available, or legalized to the next larger supported integer with appropriate truncation/extension. Overflow checks are inserted according to the declared policy (wrapping, saturating, trapping).

### 5.3 Contract Verification
The MIR will be translated into Z3 assertions when strict mode is enabled. The compiler will:
- Unroll loops up to a configured bound and use induction for generalized invariants.
- Use `k-induction` for loop invariant proofs.
- Cache verification results using hashes of the MIR + contracts.
- Fail gracefully with user-friendly error messages indicating which contract failed and suggesting possible invariants.

### 5.4 Comptime Execution
The compiler will include a **MIR interpreter** for `comptime` evaluation. This interpreter will:
- Support all safe MIR operations.
- Enforce step limits and memory limits.
- Produce constant values or types as results.
- Be sandboxed from the host system (no file I/O except for approved `comptime` imports).

### 5.5 Error Diagnostics
All diagnostics will include source location, a clear message, and optionally a suggestion for fixing the error. The compiler will be designed to produce multiple errors before exiting, where possible.

## 6. Testing Strategy

- **Unit tests** for each compiler stage (lexer, parser, type checker, codegen).
- **Snapshot tests** for error messages and IR output.
- **Integration tests** that compile and run programs, checking exit codes and output.
- **Property-based tests** for overflow semantics, type invariants, etc.
- **Fuzzing** of the parser and codegen.
- **Benchmarks** for compilation speed and generated code performance.

## 7. Contributing to the Compiler

We welcome contributions! Please read `CONTRIBUTING.md` for details. The compiler source will be in the `compiler/` directory of the main Posita repository. Each stage has its own module with clear entry points. Start by picking an issue labeled `good-first-issue` related to Stage 0.

## 8. Conclusion

This roadmap aims to deliver a working, safe, and efficient Posita compiler incrementally, with a focus on the language’s unique guarantees. The initial stages will produce a usable subset quickly, allowing community experimentation and feedback while the full vision is realized. Let’s bring ultra-static typing to reality.
