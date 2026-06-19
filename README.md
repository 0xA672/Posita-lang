# Posita

**Explicit. Static. Safe.**

Posita is a systems programming language designed for safety‑critical domains where every bit of representation, every overflow policy, and every error path must be explicitly stated and statically verified. It combines Ada’s precision, Rust’s modern type system, and compile‑time verification powered by SMT solvers to eliminate undefined behaviour without runtime overhead.

The language syntax has stabilised, the design philosophy is documented in depth, and compiler implementation is now underway. A fully tested lexer covering all keywords, literals, escape sequences, and operator tokens is complete and serves as the first concrete piece of the compiler frontend.

See [docs/SYNTAX.md](docs/SYNTAX.md) for the full language specification. See [PLAN.md](PLAN.md) for the project roadmap.
