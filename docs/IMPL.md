# Posita Compiler Implementation Guide (IMPL.md)

**Status:** Working draft — intended for compiler contributors.
**Last updated:** 2026-06-29 (updated to reflect current progress and design decisions)

## 1. Overview

This document outlines the architecture, technology choices, and development roadmap for the Posita compiler, **`ponent`**. It is a living guide, evolving alongside the language specification [`SYNTAX.md`](../SYNTAX.md).

### Core principles for the compiler
- **Correctness first**: The compiler must faithfully implement the Posita safety guarantees.
- **Compilation speed is a feature**: Posita's "compile-time first" philosophy means the compiler itself must be fast and predictable.
- **Modular and testable**: Each stage should be independently testable with clear input/output contracts.
- **Written in Rust**: The compiler is implemented in Rust, leveraging its strong type system and existing ecosystem for compiler construction (Cranelift, etc.).

## 2. Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Language** | Rust | Safety, performance, strong ecosystem for compiler construction, seamless Cranelift integration. |
| **Lexer** | `logos` | Fast, zero-allocation lexer generator. Already implemented and tested with full keyword, operator, and literal support, including complex escape sequences in character/string literals. |
| **Parser** | Hand-written recursive descent | Full control over error recovery and diagnostics; Posita grammar is designed to be LL(1) with minimal lookahead. |
| **IRs** | Custom HIR, MIR (Rust structs) | HIR preserves source-level structure; MIR is a typed, SSA-like mid-level representation. The internal IR includes `ConstTypeParam` nodes as a shared foundation for `comptime` type factories and future `const` generics. |
| **Backend** | Cranelift (primary), LLVM (optional future) | Cranelift for fast AOT/JIT compilation; LLVM as an optional high-optimization backend. RVO/NRVO and move elision for non‑`Copy` types are enforced during MIR-to-CLIF lowering, not relying on backend optimizations. |
| **SMT Solver** | Z3 via `z3-rs` crate | For contract verification in strict mode. |
| **Incremental computation** | `salsa` | For fast recompilation and IDE support. |
| **CLI** | `clap` | Standard argument parsing. |
| **Testing** | `cargo test`, snapshot tests, property-based testing | Comprehensive verification of each compiler stage. |

## 3. Compiler Pipeline

```
Source file (.ps)
    │
    ▼
Lexer (logos) ──► Token stream   [DONE]
    │
    ▼
Parser (hand-written) ──► Concrete Syntax Tree (CST) / AST   [DONE]
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
- [x] Set up Rust project with `clap` CLI.
- [x] Implement lexer using `logos` (keywords, operators, literals, comments). All token types defined and covered by 55+ tests. Includes correct handling of escape sequences, integer overflow errors, and apostrophe/attribute-access disambiguation.
- [x] Implement a minimal parser that can handle a "hello world" program: `def main() { }`. AST, error recovery, and Pratt expression parsing are in place. Supported statements: `set`, `let`, `if`, `while`, `for`, `loop`, `leave`, `continue`, `return`, `comptime`, `scope_cleanup`, `trigger`, imports, type definitions, contracts, attributes. All tokens and significant syntax are covered.
- [ ] Integrate Cranelift to produce an executable for this minimal program.
- **Goal**: `ponent run hello.ps` compiles and runs a trivial program.

### Stage 1: Core Type System and Expressions
- Implement AST nodes for all primitive types: `Int<N>`, `UInt<N>`, `Bool`, `Float`, etc.
- Implement integer literals with bit-width inference (`set x = 42` → `Int<32>`).
- Type‑annotated literals (`42: Int<32>`, `1: PositiveInt`).
- Basic arithmetic, bitwise, logical operators, with overflow control.
- Variable declarations (`set`, `set mut`), assignment.
- Control flow: `if`, `else`, `while`, `for`, `loop`, `leave`, `continue`, labels.
- Function definitions and calls, `return`.
- Simple modules: file-based, `import` and `from ... import`.
- RVO/NRVO for non‑`Copy` types, move elision guarantees.
- **Goal**: Can compile and run non-trivial algorithms (e.g., fib, sorting) with integers and arrays.

### Stage 2: Error Handling and Basic Contracts
- Implement `Result<T,E>`, `?` operator, `catch` blocks, `leave with`.
- Implement `finally` blocks and enforce their restrictions.
- `From` trait for automatic error conversion.
- Runtime contract checking for `requires` and `ensures` (no SMT yet).
- Type-level defaults (`with default`, `no_default`).
- `scope_cleanup` with `trigger`, `propagates`, `overrides`.
- Ghost variables for conditional cleanup.
- **Goal**: Robust error handling works; programs can use `?` and `catch`.

### Stage 3: Advanced Type System Features
- Generic functions and types with `where` clauses.
- Traits (similar to Rust traits) for `Add`, `Display`, etc.
- `constraint` blocks.
- `comptime` type factories (functions returning `type`).
- `@typeInfo` compile-time reflection (limited to type structure).
- `auto<T>` type capture.
- Layout control attributes (`@packed`, `@endian`, etc.).
- Enum set aliases with `|` operator for error aggregation.
- `@auto_deref` attribute for controlled method‑call auto‑dereferencing. Built‑in references (`&T` / `&mut T`) auto‑dereference unconditionally; standard library types like `Rc<T>` and `Box<T>` require `@auto_deref` on their `Deref` implementation.
- `generate` blocks for declarative code generation.
- Internal `ConstTypeParam` IR node introduced for future `const` generics.
- **Goal**: Expressive type-level programming possible; standard library can begin.

### Stage 4: Compile-Time Execution and Full Contracts
- `comptime` blocks and functions with budget limits.
- Cycle detection and safe termination.
- Static contract verification using Z3 (strict mode).
- `@trusted` and `@experimental` attributes.
- `invariant` on types and loops (verification hooks).
- `@runtime_check`, `@ieee_contracts`, `@diverges` attributes.
- May‑be and must‑be error analysis.
- **Goal**: Strict Mode can verify simple contracts automatically.

### Stage 5: Concurrency and System Programming
- Tasks and channels.
- `async`/`await` with executor.
- `extern "C"` FFI with contractual wrappers.
- `unsafe` blocks with full checks on boundaries.
- Interrupt handling (`@interrupt`).
- Dynamic dispatch (`dyn Trait`) in `@trusted` contexts.
- **Goal**: Real-time and concurrent programs compile and run.

### Stage 6: Tooling and Ecosystem
- LSP server for IDE support.
- Package manager (`capsa`).
- Debugger integration (DWARF support).
- Documentation generator.
- Code formatter (`ponent fmt`).
- **Goal**: Posita is usable for production projects.

## 5. Key Design Decisions for the Compiler

### 5.1 Memory Management
Posita does not use a garbage collector. The compiler will implement:
- **Copy semantics** by default for `Copy` types.
- **Move semantics** for non-`Copy` types, with the help of a **borrow checker** (similar to Rust's NLL). The MIR will contain explicit `move` and `copy` operations. The compiler will track ownership and insert drops at the end of scopes.
- **RVO/NRVO and move elision** for non‑`Copy` types are enforced by the frontend during MIR-to-CLIF lowering, rather than relying on backend optimizations.

### 5.2 Integer Representation
`Int<N>` and `UInt<N>` are mapped to Cranelift's `iN` types where available, or legalized to the next larger supported integer with appropriate truncation/extension. Overflow checks are inserted according to the declared policy (wrapping, saturating, trapping).

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

### 5.6 Internal Type ID System
The compiler uses type IDs internally for monomorphization, vtable generation, debug info, incremental compilation, and metadata caching. Type IDs are based on structural hashing of type content and are stable across compilation units. They are strictly internal and never exposed to user code.

### 5.7 Constant Evaluation Memory Quotas
Both `comptime` JIT execution and the `const` evaluator share a unified compiler‑internal memory quota (`--compiler-memory-limit`, default 512 MB). Any compile‑time allocation that would exceed this limit causes a clean compiler termination with a diagnostic, preventing CI‑hostile OOM kills.

### 5.8 Instance Resolution: Orphan Rules and Overlap Prohibition
Posita enforces **orphan rules** for trait implementations: an `impl Trait for Type` is allowed only if `Trait` or `Type` is defined in the current crate. Overlapping instance declarations are **prohibited** in all contexts—the compiler rejects any pair of `impl` blocks that could match the same constraint. The combination of these two rules guarantees that every constraint has at most one matching instance, preserving coherence and auditability.

### 5.9 Conversion Checking (Definitional Equality)
The MIR will include a definitional equality checker (`are_def_eq`) that determines when two terms are considered identical for the purposes of contract verification and optimization. The checker will be **type-directed** (consuming type information for η‑rules) and **bidirectional** (synthesizing types for neutral terms, checking against known types for canonical forms). This is essential for sound SMT encoding of dependent types and `old()` expressions in contracts. η‑equivalence is restricted to the MIR layer and not exposed in source‑level semantics, preserving the language's "explicit over implicit" philosophy.

### 5.10 `ConstTypeParam` in HIR/MIR
The HIR and MIR contain a `ConstTypeParam` node that represents compile‑time constant values used as type parameters. This node serves as the shared representation for both `comptime` type factories (Stage 3) and future `const` generics, ensuring that adding `const` generics does not require architectural changes to the type checker or backend.

## 6. Testing Strategy

- **Unit tests** for each compiler stage (lexer, parser, type checker, codegen).
- **Snapshot tests** for error messages and IR output.
- **Integration tests** that compile and run programs, checking exit codes and output.
- **Property-based tests** for overflow semantics, type invariants, etc.
- **Fuzzing** of the parser and codegen.
- **Benchmarks** for compilation speed and generated code performance.

## 7. Contributing to the Compiler

We welcome contributions! Please read `CONTRIBUTING.md` for details. The compiler source is in the `ponent/` repository under the `Posita-lang` organization. Each stage has its own module with clear entry points. Start by picking an issue labeled `good-first-issue` related to Stage 0.

## 8. Repository Structure

| Repository | Content | Status |
|------------|---------|--------|
| `Posita-lang/Posita-lang` | Language specification, design docs, discussions | Active |
| `Posita-lang/Ponent` | Compiler source (Rust) | Active |
| `Posita-lang/capsa` | Package manager source (Rust) | Planned (Stage 6) |
| `Posita-lang/std` | Standard library (Posita) | Planned (Stage 3+) |
| `Posita-lang/vscode-posita` | VS Code extension | Planned (Stage 6) |
| `Posita-lang/playground` | Web playground (WASM) | Planned (Stage 6) |

## 9. Current Status & Short-term Roadmap

- Lexer: Complete (all tokens defined, 59 tests passing, integer overflow errors, proper escape handling). See `src/lexer.rs`.
- Parser: Core implementation complete. Supports functions, variables, control flow, types, imports, attributes, contracts, and expressions via Pratt parsing. 3 parser tests passing. See `src/parser.rs`.
- AST: Fully defined in `src/ast.rs` with all major Posita syntax structures.
- HIR: Initial lowering and type structures implemented. Basic type context, symbol table, and builtin type registration.
- Type Checker: Core constraint generation and unification implemented. Automatic dereference for `&T`/`&mut T` and `Ptr` types, with `builtin_deref_ty` cleaning up hard-coded `Box`/`Rc` logic. `MethodInfo` extended with `has_auto_deref` field for future `@auto_deref` support.
- CLI: Skeleton exists using `clap`, supports `ponent lex` and `ponent parse` commands.

The immediate next step is integrating Cranelift to generate executable code for a minimal program (`def main() {}`), completing Stage 0.

## 10. Conclusion

This roadmap aims to deliver a working, safe, and efficient Posita compiler incrementally, with a focus on the language's unique guarantees. The initial stages will produce a usable subset quickly, allowing community experimentation and feedback while the full vision is realized. Let's bring ultra-static typing to reality.
