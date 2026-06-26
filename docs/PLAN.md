# Posita Project Plan (PLAN.md)

**Status:** Living document  
**Last updated:** 2026-06-26

## 1. Vision

Posita aims to become the premier ultra-static systems programming language for safety‑critical domains—avionics, medical devices, autonomous vehicles, and industrial control—where undefined behavior is eliminated by construction and every safety property is either statically proven or explicitly documented as an auditable trust boundary.

## 2. Project Roadmap

The project is organized into six stages. Stage 0 begins immediately; subsequent stages are gated on the completion of the previous stage.

| Stage | Name | Primary Goal |
|-------|------|--------------|
| 0 | Bootstrapping | Parse `def main() {}`, generate executable via Cranelift |
| 1 | Core language | Integers, floats, arrays, control flow, functions, basic modules |
| 2 | Error handling | `Result<T,E>`, `?`, `catch`, `leave with`, `finally`, contracts (runtime only) |
| 3 | Advanced types | Generics, traits, constraints, `comptime` type factories, `const` generics, reflection, layout control |
| 4 | Full verification | SMT‑based contract proving, strict mode, `@trusted`, `@lemma`, `@comptime_test` |
| 5 | Concurrency & FFI | Tasks, channels, `async`/`await`, `extern "C"`, `unsafe`, interrupt handling |
| 6 | Tooling & ecosystem | LSP, package manager (`capsa`), debugger, formatter, documentation generator |

## 3. Current Status (June 2026)

We have completed the initial design phase. The language syntax (`SYNTAX.md`) and core philosophy (`DESIGN.md`) are stable enough to begin compiler implementation. The package manager design (`CAPSA_DESIGN.md`) and compiler implementation guide (`IMPL.md`) are also complete.

The compiler team is implementing Stage 0. A type ID system has been adopted for internal compiler use (monomorphization, vtable generation, debug info); it is strictly internal and never exposed to user code. An internal `ConstTypeParam` IR node has been introduced in the HIR/MIR design to serve as the shared foundation for both `comptime` type factories and future `const` generics.

**Stage 0 is in progress.**

## 4. Short‑Term Goals (Stage 0–2)

### Stage 0 — Bootstrapping the Compiler Shell
- Set up `ponent` Rust project with `clap` CLI.
- Implement lexer using `logos`.
- Implement minimal parser that handles `def main() { }`.
- Integrate Cranelift to produce an executable for this minimal program.
- Implement internal type ID system (compiler‑only, not user‑exposed).
- **Deliverable**: `ponent run hello.ps` compiles and runs a trivial program.

### Stage 1 — Core Type System and Expressions
- All primitive types (`Int<N>`, `UInt<N>`, `Bool`, `Float`, arrays, slices).
- Integer literals with bit‑width inference.
- Type‑annotated literals (`42: Int<32>`, `1: PositiveInt`).
- Arithmetic, bitwise, logical operators with overflow control.
- Variable declarations (`set`, `set mut`), assignment.
- Control flow: `if`, `else`, `while`, `for`, `loop`, `leave`, `continue`, labels.
- Functions, `return`, basic modules and imports.
- RVO/NRVO for non‑`Copy` types, move elision guarantees.
- **Deliverable**: Compile and run non‑trivial algorithms (fib, sorting) with integers and arrays.

### Stage 2 — Error Handling and Basic Contracts
- `Result<T,E>`, `?`, `catch`, `leave with`, `finally`.
- `From` trait for automatic error conversion.
- Runtime contract checking for `requires`/`ensures` (no SMT yet).
- Type‑level defaults (`with default`, `no_default`).
- `scope_cleanup` with `trigger`, `propagates`, `overrides`.
- Ghost variables for conditional cleanup.
- **Deliverable**: Robust error handling; programs can use `?` and `catch`.

## 5. Medium‑Term Goals (Stage 3–5)

### Stage 3 — Advanced Type System
- Generics with `where` clauses, traits, `constraint` blocks.
- `comptime` type factories, `@typeInfo`, `auto<…>` capture.
- **Compile‑time type parameters (`const` generics)**: independent language feature enabling types parameterized by compile‑time constants (e.g., `Mod<1000000007>`). `const` generic parameters are visible in contracts and can be used for symbolic reasoning across type instances, providing capabilities beyond what `comptime` factories alone can express.
- Layout control (`@packed`, `@endian`, etc.).
- Enum set aliases with `|` operator for error aggregation.
- `generate` blocks for declarative code generation.
- **Deliverable**: Type‑level programming possible; standard library can begin.

### Stage 4 — Compile‑Time Execution and Full Contracts
- `comptime` blocks/functions with budget limits.
- Static contract verification using Z3 (strict mode).
- `@trusted`, `@lemma`, `@comptime_test`.
- `@runtime_check`, `@ieee_contracts`, `@diverges` attributes.
- May‑be and must‑be error analysis.
- **Deliverable**: Strict Mode can verify simple contracts automatically.

### Stage 5 — Concurrency and System Programming
- Tasks, channels, `async`/`await`.
- `extern "C"` FFI with contractual wrappers.
- `unsafe` blocks with boundary checks.
- Interrupt handling (`@interrupt`).
- Dynamic dispatch (`dyn Trait`) in `@trusted` contexts.
- **Deliverable**: Real‑time and concurrent programs compile and run.

## 6. Long‑Term Goals (Stage 6 and beyond)

- Full LSP server for IDE support.
- `capsa` package manager with safety‑level enforcement and audit trails.
- Debugger integration (DWARF).
- Documentation generator.
- Self‑hosting: rewrite `ponent` in Posita.
- Formal verification backend (long‑term research).
- Certification support (DO‑178C, ISO 26262) via `capsa certify`.
- `linear` and `@consume(N)` (graded types).
- Typestate (`File<Open>` / `File<Closed>`).
- Session‑typed channels (standard library pattern, not built‑in syntax).
- seL4 platform support as reference target.

## 7. Technology Stack (Compiler)

| Component | Choice |
|-----------|--------|
| Implementation language | Rust |
| Lexer | `logos` |
| Parser | Hand‑written recursive descent |
| IRs | Custom HIR / MIR (Rust structs) with `ConstTypeParam` node |
| AOT backend | `cranelift-object` |
| JIT backend | `cranelift-jit` (for `comptime` execution) |
| SMT solver | Z3 via `z3-rs` |
| Incremental computation | `salsa` |
| CLI | `clap` |
| Diagnostics | `codespan-reporting` or `miette` |
| Testing | `cargo test`, `insta` (snapshot), `proptest` |

## 8. Toolchain

| Tool | Name | Description |
|------|------|-------------|
| Compiler | `ponent` | Compiles Posita source to native code |
| Package manager | `capsa` | Dependency management, auditing, publishing |
| Formatter | `capsa fmt` | Deterministic, semantics‑preserving code formatting |
| Language server | `posita-lsp` | IDE support with contract‑aware diagnostics |

## 9. Repository Structure

| Repository | Content | Status |
|------------|---------|--------|
| `0xA672/Posita-lang` | Language specification, design docs, discussions | Active |
| `0xA672/Ponent` | Compiler source (Rust) | Active |
| `posita-lang/capsa` | Package manager source (Rust) | Planned (Stage 6) |
| `posita-lang/std` | Standard library (Posita) | Planned (Stage 3+) |
| `posita-lang/vscode-posita` | VS Code extension | Planned (Stage 6) |
| `posita-lang/playground` | Web playground (WASM) | Planned (Stage 6) |

## 10. Community Involvement

- **RFC Process**: All significant language and toolchain changes will follow an RFC process. Proposals are submitted as GitHub Discussions, reviewed by the community and core team, and accepted or rejected with public rationale.
- **Contributing**: See `CONTRIBUTING.md` for guidelines on code, documentation, and design contributions.
- **Communication**: Primary communication channels are GitHub Discussions and the project's chat server (to be determined).
- **Code of Conduct**: We follow the Contributor Covenant.

## 11. Adopted but Not Yet Implemented Features

The following features are part of the language design but will be implemented in later stages (Stage 3+):

- `linear` and `@consume(N)` (graded types)
- Typestate (`File<Open>` / `File<Closed>`)
- `const` generics (compile‑time type parameters) — planned as an independent language feature, not syntactic sugar over `comptime` factories. `const` generic parameters are first‑class compile‑time values that can appear in types, contracts, and `where` clauses, enabling symbolic reasoning across type instances.
- `Rational<p, q>` fixed‑precision rationals
- `Regex<"...">` compile‑time regex
- MMIO types and interrupt vector generation
- `@audit_log` attribute
- `@link_proof` external proof integration
- Asynchronous contract qualifiers (`on_timeout`, `on_cancel`)
- `@no_alloc_error`, `@no_panic` effects
- `old()` expressions in contracts
- `layout_of!` compile‑time reflection
- Tiered diagnostics (L1/L2/L3)
- `capsa certify` for certification report generation
- `generate` blocks for declarative code generation

These are fully specified in `SYNTAX.md` and will be incrementally added as the compiler matures.

## 12. Design Decisions

The following architectural decisions have been made and are recorded here for reference:

### 12.1 Compile‑Time Type Parameters: `comptime` Factories First, `const` Generics as Independent Feature

Posita needs the ability to parameterize types by compile‑time constants (e.g., `Mod<1000000007>`). We have decided to implement `comptime` type factories in Stage 3 as the first mechanism for compile‑time type generation, while planning `const` generics as a separate, more powerful language feature for a later stage.

- **`comptime` type factories** use the existing `comptime` execution engine to generate concrete types. They require no new compiler infrastructure and cover the majority of use cases (hardware registers, protocol frames, cryptographic parameters). However, they cannot express cross‑instance relationships or provide symbolic type parameters for contract reasoning.
- **`const` generics** will be implemented as a distinct feature that introduces first‑class compile‑time value parameters into the type system. These parameters are visible in contracts, `where` clauses, and SMT solver contexts, enabling proofs about entire families of types (e.g., “for all M > 0, Mod<M> addition is closed”) and cross‑instance dependencies (e.g., `widen<N, M> where N <= M`). An internal `ConstTypeParam` IR node has been introduced early to ensure both approaches share the same backend representation.
- When both are available, `comptime` factories will serve complex compile‑time metaprogramming, while `const` generics will serve type‑level abstraction with contract‑level symbolic reasoning.

### 12.2 Internal Type ID System

The compiler uses type IDs internally for monomorphization, vtable generation, debug info, incremental compilation, and metadata caching. Type IDs are based on structural hashing of type content and are stable across compilation units. They are strictly internal and never exposed to user code.

### 12.3 Default Backend

The default compiler backend is Cranelift, chosen for fast compilation. An optional LLVM backend may be added later for peak performance. RVO/NRVO and move elision for non‑`Copy` types are enforced by the frontend during MIR-to-CLIF lowering, rather than relying on backend optimizations.
