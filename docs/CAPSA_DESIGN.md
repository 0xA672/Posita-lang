# Capsa: The Posita Package Manager

**Status:** Draft  
**Last updated:** 2026-06-19

## 1. Design Philosophy

`capsa` (Latin for "box, container") is the official package manager and build tool for the Posita language. It serves as the primary interface for developers to manage dependencies, build projects, run tests, and ensure the security and correctness of their code.

`capsa` extends Posita's core philosophy into the toolchain:

- **Explicit over implicit**: All dependencies, versions, feature flags, and security policies are declared in the manifest. No global or hidden state.
- **Closed-World Assumption (CWA)**: The full dependency graph is resolved and locked before building. In strict mode, any code that cannot be statically verified (e.g., unverified `unsafe` blocks) is flagged or rejected.
- **Safety by proof**: `capsa` deeply integrates with Posita's contract system, `@trusted` boundaries, effect annotations, and formal verification capabilities. It can require that dependencies pass contract verification and generate auditable security evidence.

## 2. The Manifest (`posita.toml`)

Every Posita project (a "box") contains a `posita.toml` file that defines metadata, dependencies, and build configuration.

```toml
[package]
name = "my_safety_app"
version = "0.1.0"
edition = "2026"
strict = true                     # Enforce strict mode globally

[dependencies]
libmath = { version = "1.2", path = "../libmath" }
libhal  = { version = "0.3", registry = "default" }

[dev-dependencies]
posita-test = "0.5"

[build]
target = "arm-none-eabi"          # Cross-compilation target
optimization = "size"             # Options: speed, size, none
```

### Key fields

| Field | Description |
|-------|-------------|
| `[package].edition` | The language edition. Ensures consistent behavior across compiler versions. |
| `[package].strict` | Enables strict mode project-wide. All contracts must be proven, `@trusted` blocks require evidence. |
| `[dependencies]`   | Supports local paths, Git repositories, and the official registry. |
| `[build].target`   | Explicitly sets the cross-compilation target. |
| `[build].optimization` | `speed`, `size`, or `none`. In strict mode, defaults to `size` for deterministic performance. |

## 3. Dependency Resolution and CWA

`capsa` uses a SAT-based version resolution algorithm, similar to `cargo`, but enhanced for the closed-world assumption.

- **Full graph resolution**: All transitive dependencies are flattened and checked for compatibility. Unresolvable conflicts (e.g., diamond dependencies) produce a clear error requiring manual intervention.
- **Lock file (`posita.lock`)**: Records exact versions, sources, and integrity hashes (SHA-256). Ensures fully reproducible builds.
- **CWA enforcement**: In strict mode, `capsa` scans the entire dependency tree. Any dependency containing unverified `unsafe` blocks, or `extern "C"` calls without contractual wrappers, is flagged. The build can be configured to either warn or fail based on security requirements.

## 4. Security and Safety Integration

This is where `capsa` fundamentally differs from general-purpose package managers.

### 4.1 Contract Audit Dependencies (`@audit`)

Security policies can be applied per-dependency in `posita.toml`:

```toml
[dependencies]
# Trust this package; no further audit required
libmath = { version = "1.2", trusted = true }

# Require this package to pass full contract verification (strict mode)
libcrypto = { version = "0.8", strict-prove = true }
```

If `libcrypto` fails to prove its own `requires` and `ensures` contracts, `capsa` will refuse to build.

### 4.2 Effect Annotation Consistency

`capsa` verifies that effect annotations (`@pure`, `@io(read)`, `@io(write)`, `@alloc`, `@no_alloc`) are consistent across the dependency graph. A function annotated `@pure` cannot call an `@io` function without an explicit `@trusted` bridge.

### 4.3 `unsafe` Isolation and Auditing

The `capsa audit` command scans the entire project and its dependencies to:

- List all modules containing `unsafe` blocks.
- Verify that every `unsafe` block is encapsulated in an `@trusted` function with `requires`/`ensures` contracts.
- Compute the percentage of `@trusted` code relative to total code.
- Flag `extern "C"` declarations that lack safety wrappers.

This report serves as a security certification artifact for standards like DO-178C or ISO 26262.

### 4.4 External Proof Verification (`@link_proof`)

If a `@trusted` function references an external formal proof (e.g., a Coq file via `@link_proof`), `capsa` verifies the existence and hash of the proof file during both publishing and installation. The proof is distributed alongside the package, enabling independent verification by auditors.

## 5. Publishing and the Registry

### 5.1 The Official Registry

`capsa` connects to `registry.posita-lang.org` by default. This registry is similar to `crates.io` but with a focus on safety metadata.

### 5.2 Publishing Workflow (`capsa publish`)

Before a package can be published, `capsa` enforces the following:

1. **Full Strict Mode Build**: The entire dependency tree (unless `trusted = true` is set) is compiled in strict mode. All contracts must be proven, all `@trusted` blocks must carry contracts, and effect annotations must be consistent.
2. **Security Evidence Generation**: A "Security Evidence Bundle" is created, containing:
   - Contract verification reports.
   - `unsafe` usage statistics.
   - External proof references and their hashes.
   - A signed summary from the registry's verification service.
3. **Tamper-proofing**: The evidence bundle is cryptographically signed and linked to the exact source code hash. Any mismatch during installation triggers a security warning.

### 5.3 Safety Levels

Packages in the registry are classified into four safety levels:

| Level | Name | Description |
|-------|------|-------------|
| **3** | **Certified** | All contracts proven, all `@trusted` blocks backed by `@link_proof`, full strict mode pass. |
| **2** | **Verified** | All contracts proven, `unsafe` is encapsulated in `@trusted` functions with contracts. |
| **1** | **Audited** | `unsafe` is encapsulated in `@trusted` functions, but contracts may rely on manual trust. |
| **0** | **Unverified** | No safety guarantees. This level is used for early-stage or experimental packages and is clearly marked in search results. |

These levels are prominently displayed on the registry and in `capsa search` output.

## 6. The `capsa` CLI

`capsa` provides a familiar, Cargo-like experience:

```bash
# Creating a new project
capsa new my_project

# Building the project (calls `ponent`)
capsa build

# Building and running
capsa run

# Building in strict mode
capsa build --strict

# Running tests (including @comptime_test and test blocks)
capsa test

# Auditing the project and its dependencies for safety
capsa audit

# Generating a traceability matrix from @spec tags
capsa spec

# Updating dependencies to their latest compatible versions
capsa update

# Formatting code
capsa fmt

# Installing a published binary
capsa install
```

## 7. Private Registries and Enterprise Policies

`capsa` supports configuring private registries for internal or proprietary use. Enterprises can enforce additional policies:

- **Mandatory review signatures**: All packages must be signed by an authorized reviewer.
- **License compliance checks**: Ensure all dependencies use approved licenses.
- **Vulnerability scanning**: Integrate with external vulnerability databases.

These policies are defined in a `capsa.toml` configuration file at the project or organization level.

## 8. Comparison with Cargo

| Feature | Cargo | Capsa |
|---------|-------|-------|
| **Basic workflow** | `cargo build`, `run`, `test` | Identical, lowering the learning curve |
| **Dependency resolution** | SAT-based version solving | Same, but enforced CWA |
| **Security auditing** | External tool (`cargo audit`) | **Built-in `capsa audit`** |
| **Contract verification** | None | **Strict mode with full SMT-based contract proving** |
| **Effect consistency** | None | **Verification of `@pure`, `@io`, `@alloc` annotations** |
| **`unsafe` isolation** | None | **Automated reporting and enforcement** |
| **External proof integration** | None | **`@link_proof` distribution and verification** |
| **Registry safety levels** | None | **Certified, Verified, Audited, Unverified** |

## 9. Future Plans

- **`capsa verify`**: A standalone command that runs a full strict-mode verification of the entire project and generates a printable "Safety Certificate" suitable for regulatory submission.
- **Binary distribution**: Allow publishing pre-compiled, contract-signed binaries for closed-source or confidential projects.
- **Workspace support**: Enhanced management of multi-crate workspaces for large-scale safety-critical systems.
