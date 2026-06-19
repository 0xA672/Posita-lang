# Posita Project Plan (PLAN.md)

**Status:** Living document  
**Last updated:** 2026-06-19

## 1. Vision

Posita aims to become the premier ultra-static systems programming language for safety‑critical domains—avionics, medical devices, autonomous vehicles, and industrial control—where undefined behavior is eliminated by construction and every safety property is either statically proven or explicitly documented as an auditable trust boundary.

## 2. Project Roadmap

The project is organized into six stages. Stage 0 begins immediately; subsequent stages are gated on the completion of the previous stage.

| Stage | Name | Primary Goal |
|-------|------|--------------|
| 0 | Bootstrapping | Parse `def main() {}`, generate executable via Cranelift |
| 1 | Core language | Integers, floats, arrays, control flow, functions, basic modules |
| 2 | Error handling | `Result<T,E>`, `?`, `catch`, `leave with`, `finally`, contracts (runtime only) |
| 3 | Advanced types | Generics, traits, constraints, `comptime` type factories, reflection, layout control |
| 4 | Full verification | SMT‑based contract proving, strict mode, `@trusted`, `@lemma`, `@comptime_test` |
| 5 | Concurrency & FFI | Tasks, channels, `async`/`await`, `extern "C"`, `unsafe`, interrupt handling |
| 6 | Tooling & ecosystem | LSP, package manager (`capsa`), debugger, formatter, documentation generator |

## 3. Current Status (June 2026)

We have completed the initial design phase. The language syntax (`SYNTAX.md`) and core philosophy (`DESIGN.md`) are stable enough to begin compiler implementation. The package manager design (`CAPSA_DESIGN.md`) and compiler implementation guide (`IMPL.md`) are also complete.

**Stage 0 is about to begin.**

## 4. Short‑Term Goals (Stage 0–2)

### Stage 0 — Bootstrapping the Compiler Shell
- Set up `ponent` Rust project with `clap` CLI.
- Implement lexer using `logos`.
- Implement minimal parser that handles `def main() { }`.
- Integrate Cranelift to produce an executable for this minimal program.
- **Deliverable**: `ponent run hello.ps` compiles and runs a trivial program.

### Stage 1 — Core Type System and Expressions
- All primitive types (`Int<N>`, `UInt<N>`, `Bool`, `Float`, arrays, slices).
- Integer literals with bit‑width inference.
- Arithmetic, bitwise, logical operators with overflow control.
- Variable declarations (`set`, `set mut`), assignment.
- Control flow: `if`, `else`, `while`, `for`, `loop`, `leave`, `continue`, labels.
- Functions, `return`, basic modules and imports.
- **Deliverable**: Compile and run non‑trivial algorithms (fib, sorting) with integers and arrays.

### Stage 2 — Error Handling and Basic Contracts
- `Result<T,E>`, `?`, `catch`, `leave with`, `finally`.
- `From` trait for automatic error conversion.
- Runtime contract checking for `requires`/`ensures` (no SMT yet).
- Type‑level defaults (`with default`, `no_default`).
- **Deliverable**: Robust error handling; programs can use `?` and `catch`.

## 5. Medium‑Term Goals (Stage 3–5)

### Stage 3 — Advanced Type System
- Generics with `where` clauses, traits, `constraint` blocks.
- `comptime` type factories, `@typeInfo`, `auto[<T>..]` capture.
- Layout control (`@packed`, `@endian`, etc.).
- **Deliverable**: Type‑level programming possible; standard library can begin.

### Stage 4 — Compile‑Time Execution and Full Contracts
- `comptime` blocks/functions with budget limits.
- Static contract verification using Z3 (strict mode).
- `@trusted`, `@lemma`, `@comptime_test`.
- `@runtime_check` attribute.
- **Deliverable**: Strict Mode can verify simple contracts automatically.

### Stage 5 — Concurrency and System Programming
- Tasks, channels, `async`/`await`.
- `extern "C"` FFI with contractual wrappers.
- `unsafe` blocks with boundary checks.
- Interrupt handling (`@interrupt`).
- **Deliverable**: Real‑time and concurrent programs compile and run.

## 6. Long‑Term Goals (Stage 6 and beyond)

- Full LSP server for IDE support.
- `capsa` package manager with safety‑level enforcement and audit trails.
- Debugger integration (DWARF).
- Documentation generator.
- Self‑hosting: rewrite `ponent` in Posita.
- Formal verification backend (long‑term research).
- Certification support (DO‑178C, ISO 26262) via `capsa certify`.

## 7. Technology Stack (Compiler)

| Component | Choice |
|-----------|--------|
| Implementation language | Rust |
| Lexer | `logos` |
| Parser | Hand‑written recursive descent |
| IRs | Custom HIR / MIR (Rust structs) |
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

| Repository | Content |
|------------|---------|
| `posita-lang/posita-lang` | Language specification, design docs, discussions |
| `posita-lang/ponent` | Compiler source (Rust) |
| `posita-lang/capsa` | Package manager source (Rust) |
| `posita-lang/std` | Standard library (Posita) |
| `posita-lang/vscode-posita` | VS Code extension |
| `posita-lang/playground` | Web playground (WASM) |

## 10. Community Involvement

- **RFC Process**: All significant language and toolchain changes will follow an RFC process. Proposals are submitted as GitHub Discussions, reviewed by the community and core team, and accepted or rejected with public rationale.
- **Contributing**: See `CONTRIBUTING.md` for guidelines on code, documentation, and design contributions.
- **Communication**: Primary communication channels are GitHub Discussions and the project's chat server (to be determined).
- **Code of Conduct**: We follow the Contributor Covenant.

## 11. Adopted but Not Yet Implemented Features

The following features are part of the language design but will be implemented in later stages (Stage 3+):

- `linear` and `@consume(N)` (graded types)
- Typestate (`File<Open>` / `File<Closed>`)
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

These are fully specified in `SYNTAX.md` and will be incrementally added as the compiler matures.

## 12. Immediate Next Steps

1. Initialize the `posita-lang/ponent` repository with a Rust project.
2. Set up CI/CD for the compiler.
3. Implement Stage 0 (lexer + parser for `def main() {}`).
4. Begin implementing Stage 1 (core types and expressions).
5. Publish the first alpha release when Stage 1 is complete.

---

We are building the foundation for a new generation of safety‑critical software. Every line of code we write will itself be subject to the same rigorous standards we aim to provide for our users. Let's begin.
