# Posita Design Document (DESIGN.md)

**Status:** Working draft  
**Last updated:** 2026-06-19

This document captures the overarching design philosophy of Posita, records key design decisions, and places the language within the broader programming language landscape. It complements the formal syntax specification (`SYNTAX.md`), the implementation plan (`IMPL.md`), and the project roadmap (`PLAN.md`).

## 1. Core Philosophy: Ultra-Static Typing

Posita is built on the principle of **"ultra-static typing"**: every representation detail and behavioral guarantee that *can* be resolved at compile time *must* be resolved at compile time. The language provides first-class mechanisms for programmers to explicitly state their intent and for the compiler to mathematically verify it.

This philosophy manifests in several concrete principles:

- **Explicit over Implicit**: No hidden ABI, no type-erased errors, no null pointers, no implicit overflow, no invisible allocations. All critical semantic effects—compile-time execution (`!`), error propagation (`?`), asynchronous suspension (`await`)—are marked with dedicated syntax at the point of use. Unsafe operations are confined to functions marked `@trusted` at the point of declaration, establishing explicit trust boundaries. Reviewers can see every important behavior without reading function definitions.

- **Compile-time over Run-time**: Error handling, defaults, optimizations, reflection, resource tracking, and even some proofs are resolved statically. Work that can be done once, before deployment, should be.

- **No Undefined Behavior in Safe Code**: In Posita's safe subset, every operation either succeeds with well-defined semantics or is rejected at compile time. This includes eliminating all traditional sources of UB such as uninitialized variables, integer overflow traps, null pointer dereferences, data races, and out-of-bounds access.

- **Auditability over Brevity**: Code is written to be read, reviewed, and certified. Keywords (`def`, `set`, `leave`, `catch`), spec tags (`@spec`, `@requirement`), and explicit contracts (`requires`, `ensures`) make intent visible. The language deliberately rejects syntactic sugar that obscures control flow or performance characteristics.

## 2. The Graded Type System: Theory Meets Practice

Posita's type system can be characterized as a practical fusion of **refinement types** and **graded types**.

- **Refinement Types** describe *what data is*: bit-widths (`Int<13>`), overflow behavior (`with overflow = saturate`), default values (`with default = 0xFF`), and logical invariants (`invariant n != 0`). These allow the programmer to specify the exact physical representation and logical constraints of data.

- **Graded Types** describe *how data is used*: affine ownership (used at most once), linear constraints (used exactly once, planned), and effect annotations (`@pure`, `@io`, `@alloc`). Planned usage-count contracts (`@consume(N)`) extend this to arbitrary compile-time constants.

Unlike full dependent type systems (e.g., Coq, Idris), Posita restricts type-level dependency to **compile-time constants**. This avoids the need for runtime evidence or complex proof terms, keeping the type system decidable and the performance model predictable. The SMT solver is employed as an automated theorem prover for contracts, and proof obligations that cannot be automatically discharged are surfaced for human review via the `capsa audit` toolchain.

## 3. Key Design Decisions

### 3.1 Why Copy Semantics by Default? (Move vs. Copy)

Posita defaults to **copy semantics** for types that implement the `Copy` trait (pure data). This eliminates "moved-from" invalid states and simplifies reasoning, which is crucial for safety-critical systems. Large types that manage resources (implement `Drop`) are not `Copy` and are moved explicitly via the `move` keyword. This design provides the safety of ownership without the cognitive burden of ubiquitous moves for simple values. For a detailed rationale, see the "Move vs. Copy" design note within this document's appendix.

### 3.2 Error Handling: No Exceptions, No Type Erasure

Posita rejects both exceptions and type-erased error interfaces (like `Box<dyn Error>`). Instead, it uses a monomorphic `Result<T, E>` type with:

- `?` for propagation,
- `catch` for localized pattern matching on errors,
- `leave with Err(...)` for structured early return,
- `From` trait implementations for automatic, statically visible error conversion.

This makes every possible error path fully visible in the source code and prevents the silent loss of error type information.

### 3.3 Memory Management: Affine Ownership + RAII + Finally

Resources are managed through a combination of:

- **Affine ownership**: Non-`Copy` types can be used at most once, preventing double-free.
- **`Drop` trait**: Provides deterministic cleanup at the end of a scope.
- **`finally` blocks**: Guarantee infallible cleanup on any exit path. Fallible cleanup must be handled explicitly via `scope_cleanup`.
- **`scope_cleanup`**: Named, composable cleanup actions that can be triggered early.

### 3.4 Concurrency Model: Tasks, Channels, and Async

Posita provides a structured concurrency model suitable for real-time systems:

- **Tasks** are the basic unit of concurrency, with private heaps.
- **Channels** are typed, bounded, and synchronous for safe inter-task communication.
- **`async`/`await`** provides non-blocking operations, with suspension points explicitly marked for reviewers.

Interrupt handlers are modeled as special, highly constrained tasks that must not allocate and cannot panic. The compiler automatically generates the interrupt vector table from `@interrupt` annotations.

### 3.5 The `unsafe` Keyword and `@trusted` Boundary

`unsafe` is a block-level construct for operations the compiler cannot verify (inline assembly, raw pointers, C FFI). All `unsafe` code must be encapsulated in `@trusted` functions with `requires`/`ensures` contracts. The compiler trusts these contracts as axioms. This makes the trust boundary explicit, auditable via `capsa audit`, and, in Strict Mode, entirely forbidden.

## 4. Relationship to Other Languages

Posita draws inspiration from several languages and research projects.

### 4.1 Ada (and SPARK)
- **Inherited**: Explicit representation control, attribute syntax (`'Size`, `'Length`), strong typing, contract-based verification (`Pre`/`Post` → `requires`/`ensures`), default initialization, and the philosophy of "code as documentation."
- **Divergence**: Posita adopts a modern, concise syntax. Contracts are deeply integrated into the type system and verified via an SMT solver, rather than relying on a separate external tool.

### 4.2 Rust
- **Inherited**: `Result`-based error handling (extended to be fully monomorphized), `match` expressions, trait-like generics, borrow checking, and the concept of `unsafe` blocks.
- **Divergence**: Posita rejects type erasure (`dyn Trait` is explicit). It defaults to copy semantics for data, integrates SMT-based contract verification, and provides fine-grained effect annotations.

### 4.3 Zig
- **Inherited**: The `comptime` mechanism and the philosophy of moving computation to compile time are direct inspirations. `comptime` functions in Posita serve a similar role, enabling type factories and compile-time reflection.
- **Divergence**: Posita requires compile-time calls to be explicitly marked with `!`. It integrates `comptime` with SMT-based contract verification, going beyond Zig's capabilities. Posita's safety guarantees are far more extensive.

### 4.4 ATS (Applied Type System)
- **Inherited**: The ambition to encode invariants in types and eliminate many classes of runtime errors through static proofs.
- **Divergence**: Posita does not require programmers to write proof terms. It relies on a combination of an SMT solver and user-provided contracts (`requires`, `ensures`, `invariant`) to achieve a similar, though more automated, level of assurance.

### 4.5 Granule
- **Relationship**: Granule's graded modal types provide the theoretical underpinning for our planned usage-count contracts (`@consume(N)`) and effect system. Posita aims to bring these ideas from a research setting into industrial practice.

### 4.6 Dafny / F\*
- **Relationship**: These are primarily verification-focused research languages. Posita is a general-purpose systems language first, with formal verification as an optional, deeply integrated feature. Posita also adds systems-level control (bit-widths, layout, pointers) that these languages abstract away.

## 5. Safety Boundaries: Where the Proofs End

Posita acknowledges that its static guarantees have limits. These boundaries are explicitly defined and managed.

- **C ABI Boundary**: All calls to C are inherently `unsafe` and must be wrapped in `@trusted` functions with explicit contracts. The compiler can be configured to forbid C FFI in strict mode.
- **Microarchitectural State**: Posita does not model caches, TLBs, or branch predictors. It provides deterministic code generation and exports metadata (`--emit-timing-info`) for external WCET analysis tools, rather than trying to guarantee timing internally.
- **Kernel-Level Operations**: Posita can model hardware structures (page tables, registers) with its type system, but inherently unsafe operations (e.g., intentional page faults, self-modifying code) must be confined to `unsafe` blocks and manually verified.

## 6. Compilation and Toolchain Philosophy

- **Compiler (`ponent`)**: A Rust-based implementation that front-loads optimization by leveraging type and contract information. Its backend is modular: Cranelift is the default for fast compilation; an optional LLVM backend is available for peak performance.
- **Package Manager (`capsa`)**: An all-in-one build tool and package manager that deeply integrates safety. It enforces the Closed-World Assumption (CWA), audits `unsafe` code, and generates security evidence for certification.
- **Bootstrapping**: The long-term goal is for `ponent` to be self-hosted. The compiler will eventually be rewritten in Posita, proving the language's maturity and capability.

## 7. Why Posita?

Posita is a direct response to the limitations of existing languages in the safety-critical domain. It offers a unique combination of:

- **Hardware-level precision**: Bit-widths, endianness, and register layouts are first-class citizens, not an afterthought.
- **Mathematically verifiable safety**: Contracts and invariants are checked by an SMT solver at compile time.
- **Full audit trail**: From high-level specifications (`@spec`) down to the generated code, every decision is transparent and traceable.
- **Modern ergonomics**: Pattern matching, traits, channels, and async/await provide a developer experience that rivals modern general-purpose languages, without sacrificing safety.
