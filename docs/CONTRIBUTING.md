# Contributing to Posita

Thank you for your interest in contributing to **Posita**!  
Posita is in its earliest design phase — there is no compiler code yet, but we are actively shaping the language syntax, type system, and safety guarantees. Your input at this stage is incredibly valuable.

## Code of Conduct

We follow the [Contributor Covenant](https://www.contributor-covenant.org/).  
Be kind, respectful, and constructive. We aim to build a community where everyone feels safe to ask questions and propose ideas.

## How You Can Help Right Now

### 1. Discuss the Language Design
- Read the [language syntax document](SYNTAX.md).
- Join [GitHub Discussions](https://github.com/0xA672/Posita-lang/discussions) to ask questions, critique the design, or suggest improvements.
- Look for open issues labeled `design-discussion`.

### 2. Propose Changes to the Specification
- Open an issue to propose a new feature or change to the syntax / semantics.
- Follow the **RFC (Request for Comments)** template if provided (we'll add one soon).
- Be prepared to explain how your proposal aligns with Posita’s core philosophy: **explicit, compile-time, ultra-static typing with no undefined behavior**.

### 3. Improve Documentation
- Fix typos, unclear wording, or missing examples in `docs/SYNTAX.md` or other docs.
- Add design rationale documents (e.g., in `design/`).
- Write tutorials or examples (even in pseudocode) to help others understand Posita.

### 4. Share Your Domain Expertise
- Do you work in embedded systems, safety-critical software, or formal verification?  
  Share real‑world requirements that Posita should handle.

## Future Contributions (Compiler Development)

The compiler will be implemented in **Rust**, using **Cranelift** as the primary backend.  
Once implementation starts, we’ll need help with:

- Lexer / parser (hand‑written recursive descent)
- Semantic analysis (type checking, borrow checking, contract expansion)
- MIR (mid‑level IR) design and lowering
- Code generation (Cranelift / LLVM)
- Comptime interpreter and SMT integration
- Standard library design
- Testing infrastructure (unit, snapshot, fuzzing)

Keep an eye on the [issue tracker](https://github.com/0xA672/Posita-lang/issues) for tasks labeled `good-first-issue`.

## Communication Channels

- **Primary**: GitHub Discussions  
- **Real-time chat**: *(to be decided – we may set up a Zulip or Discord server)*
- **Announcements**: Watch the repository for releases and major updates.

## Getting Started (for compiler work, later)

1. Install [Rust](https://rustup.rs) (stable, latest version).
2. Clone the repository.
3. Build with `cargo build`.
4. Run tests with `cargo test`.

We’ll add detailed `COMPILER.md` and setup instructions once the compiler work begins.

## Pull Request Process

1. Open an issue first for non‑trivial changes. Discuss the approach.
2. Fork the repo and create a feature branch.
3. Ensure your changes follow the language style and pass all tests.
4. Update relevant documentation if necessary.
5. Submit a PR referencing the issue.  
   One of the maintainers will review it.

## Licensing

Posita is distributed under the MIT license (or Apache 2.0, to be confirmed).  
By contributing, you agree that your work will be licensed under the same terms.

---

We are excited to build a language that puts safety, precision, and compile‑time power first — and we’d love for you to be part of it from the very beginning.
