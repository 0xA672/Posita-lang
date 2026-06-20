# Posita Language Syntax
**Document revision: 2026-06-20** (working draft, not a frozen specification)

> [!NOTE]
> This document version tracks its own edits. It does **not** correspond to a language specification release.
> Posita itself is in pre-alpha; the syntax is under active design and may change without notice.

## Design Philosophy
Posita is a **ultra-static, systems programming language** where the programmer explicitly *posits* every representation detail: bit widths, pointer sizes, default values, error paths, overflow behavior, and even resource consumption protocols. All decisions are made visible in source and enforced at compile time with zero runtime overhead.

- **Explicit over implicit**: No hidden ABI, no type-erased errors, no null pointers, no implicit overflow, no invisible allocations. All critical semantic effects—compile-time execution (`!`), error propagation (`?`), asynchronous suspension (`await`)—are marked with dedicated syntax at the point of use. Unsafe operations are confined to functions marked `@trusted` at the point of declaration, establishing explicit trust boundaries. Reviewers can see every important behavior without reading function definitions.
- **Compile-time over run-time**: Error handling, defaults, optimizations, reflection, and resource tracking are resolved statically. A function may optionally defer contract checking to runtime via `@runtime_check`.
- **Readable as documentation**: English keywords (`def`, `set`, `leave`, `catch`), not cryptic symbols. Specification tags (`@spec`, `@requirement`, `@rationale`) link code directly to system requirements.
- **No undefined behavior in safe code**: Every operation either succeeds with defined semantics or is rejected at compile time.

---

## Lexical Structure

### Keywords
`def`, `set`, `type`, `with`, `default`, `return`, `if`, `else`, `for`, `in`, `while`, `loop`, `leave`,
`comptime`, `import`, `as`, `true`, `false`, `auto`, `and`, `or`, `not`, `sizeof`, `alignof`,
`catch`, `panic`, `unsafe`, `let`, `finally`,
`where`, `requires`, `ensures`, `invariant`, `constraint`, `move`, `dyn`, `by`, `copy`, `ref`, `mut`, `wrap`, `saturate`, `trap`, `Self`, `no_default`, `extern`, `pub`, `edition`, `deprecated`, `experimental`, `endian`, `bit_order`, `align`, `pad`, `packed`, `async`, `await`, `task`, `channel`, `linear`, `consume`, `pure`, `io`, `trusted`, `ghost`, `scope_cleanup`, `trigger`, `validate`, `missing_match`, `apply_lemma`, `exists`, `trait`, `impl`, `decreases`, `terminates`, `cfg`, `isolate`, `hint`, `must_use`, `must_handle`, `link_proof`, `exhaustive`, `no_alloc_error`, `no_panic`, `debug_info`, `old`, `audit_log`, `interrupt`

`Int`, `UInt`, `Ptr`, `Str`, `String`, `Result`, `Option`, `usize`, `Float` are built-in type constructors, not reserved words.  
`linear`, `consume` are planned keywords; `dyn`, `by` are reserved for future use.

### Identifiers
`[a-zA-Z_][a-zA-Z0-9_]*`

### Literals
- Integers: `42`, `0xFF`, `0b1010`
- Integer suffixes for explicit bit-width: `42i32` (equivalent to `42: Int<32>`), `0xFFu8` (equivalent to `0xFF: UInt<8>`). The suffix is syntactic sugar and the type is fully checked.
- Floats: `3.14`, `2.5e-3` (type `Float<64>` by default)
- Characters: `'a'` (UTF-8, type `UInt<8>`)
- Byte strings: `b"hello\n"` (type `&[Byte]`, see Escape Sequences)
- Strings: `"hello"` (type `&Str`, guaranteed valid UTF-8, see Escape Sequences)
- Booleans: `true`, `false`

### Escape Sequences

The following escape sequences are recognized in string literals, byte string literals, and character literals. They are resolved by the lexer at compile time and replaced with the corresponding byte or Unicode scalar value.

| Sequence | Name | Byte Value |
|----------|------|------------|
| `\n` | Line Feed | 0x0A |
| `\r` | Carriage Return | 0x0D |
| `\t` | Horizontal Tab | 0x09 |
| `\\` | Backslash | 0x5C |
| `\"` | Double Quote | 0x22 |
| `\'` | Single Quote | 0x27 |
| `\0` | Null Character | 0x00 |
| `\xNN` | Hex Byte (2 digits) | 0xNN |
| `\u{NNNNNN}` | Unicode Scalar (1-6 hex digits) | UTF-8 encoded bytes |

In character literals (`'...'`), the escape must resolve to exactly one valid Unicode scalar value.

In byte strings (`b"..."`) and byte characters, `\u{...}` is not valid (byte strings contain raw bytes, not UTF-8).

The sequences `\a`, `\b`, `\v`, `\f`, and `\?` are not recognized in Posita string literals. If needed, use the equivalent `\xNN` form.

### Comments
- Line comment: `// ...`
- Block comment: `/* ... */`
- Documentation comment: `/// ...` (Markdown, code examples are automatically verified as `@comptime_test`)
- Module-level documentation comment: `//! ...` (applies to the enclosing module)

---

## Specification and Traceability

Posita integrates specification directly into the codebase via documentation comments, enabling reviewers and auditors to trace system requirements to their implementations without leaving the source file.

### Spec Tags

The following tags are used inside `///` documentation comments:

- **`@spec`** – references an external standard or system requirement document.
- **`@requirement`** – cites a specific requirement identifier and its description.
- **`@rationale`** – explains why the implementation satisfies the requirement.
- **`@trace`** – establishes a traceability link from requirement to implementation.
- **`@safety_integrity`** – marks the ASIL/DO‑178C/IEC 61508 level associated with the function.

Example:
```posita
/// @spec TCAS‑II v7.1, Section 3.2.1
/// @requirement REQ‑SAFE‑004: If intruder altitude is within ±1000 ft
///   and vertical separation is projected to be lost within 25 seconds,
///   a Resolution Advisory (RA) must be issued.
/// @rationale This function is the core of the collision avoidance logic.
/// @trace REQ‑SAFE‑004 → issue_resolution_advisory → `requires`/`ensures`
/// @safety_integrity DO‑178C Level A
def issue_resolution_advisory(intruder: AircraftState, own: AircraftState) -> Result<RA, Error>
    requires abs(intruder.altitude - own.altitude) <= 1000
    ensures result.is_ok() implies ra_issued()
{
    // ...
}
```

The `capsa spec` command collects all these tags and generates a traceability matrix suitable for certification submissions.

---

## Type System

**Note on semicolons:** A semicolon after a type definition is optional. It is typically omitted for simple one‑line definitions and may be used for clarity after complex definitions.

### Value Semantics
Posita defaults to **copy semantics** for all types that implement the `Copy` trait. A type automatically derives `Copy` if it consists only of integers, floats, other `Copy` types, and does **not** implement `Drop`. This ensures that "copy is a trivial bitwise replication" and eliminates "moved‑from" invalid states. Large types like `Vector<T>` are **not** `Copy` because they manage resources (they implement `Drop`). Explicit `move` semantics are available via the `move` keyword for ownership transfer optimization.

To prevent automatic `Copy` derivation for a type that would otherwise qualify, either implement `Drop` (even with an empty body) or use the `@no_copy` attribute:
```posita
@no_copy
type LargeStruct = struct { data: [Int<8>; 1024] };
```

Manual implementation of `Copy` is allowed for types where bitwise replication is semantically sound, even if large. By doing so, the programmer explicitly accepts the performance characteristics.

### Bit‑width Parameterized Integers
- Signed: `Int<bits>`, `bits` must be compile‑time constant, 1..64.
- Unsigned: `UInt<bits>`.
- Example: `Int<13>`, `UInt<8>`.

### Floating‑Point Types
- `Float<bits>`: IEEE 754‑compliant floating‑point type with the specified bit width. `bits` must be 32 or 64. `Float<32>` corresponds to single precision, `Float<64>` to double precision.
- Float literals (`3.14`, `2.5e-3`) have a default type of `Float<64>` unless explicitly annotated.

### Fixed‑Precision Rational Numbers
- `Rational<p, q>`: A fixed‑point rational type with `p` integer bits and `q` fractional bits. Arithmetic is performed exactly over the rational domain for contracts. The default overflow policy is `saturate`; `with overflow = trap` may be specified. Conversion to floating‑point is explicit (`as Float<64>`), using round‑to‑nearest ties‑to‑even by default.

### Compile‑Time Regular Expressions
- `Regex<"...">` is a type that validates the pattern at compile time and compiles it into a deterministic automaton. It supports safe, zero‑overhead string matching at runtime. Regex types cannot appear in contracts directly; they are for runtime use only.

### Type‑level Constraint Shorthand
Instead of `exists n: Int<32> invariant n > 0`, you may write:
```posita
type PositiveInt = Int<32> where value > 0;
```
This is syntactic sugar with identical semantics.

### Zero‑Size Types (ZST)
Posita guarantees that any `struct` composed entirely of ZST fields has size zero, and `[ZST; N]` also has size zero. ZST fields in `@packed` structs occupy no space. When `@align(N)` is applied to a ZST type, the alignment is honored but the size remains zero.

### Platform Word Type
`usize` is a built‑in type alias for the unsigned integer whose width equals the target platform's pointer width. On 64‑bit targets it is `UInt<64>`, on 32‑bit targets `UInt<32>`, etc. It is intended for array indexing and pointer‑sized values. Programmers may always write `UInt<64>` or `UInt<32>` explicitly when portability is not a concern.

### Overflow Behavior
Integer overflow is never undefined. The default overflow policy is `trap` (compile‑time error in strict mode, runtime panic otherwise). Programmers can override at the type level:
```posita
type WrapCount = Int<32> with overflow = wrap;     // two's complement wrap
type SatCount  = Int<32> with overflow = saturate;  // saturation
type StrictCount = Int<32> with overflow = trap;    // trap (default)
```
Or at the operator level with suffixes:
- `+%`, `-%`, `*%` : wrap
- `+?`, `-?`, `*?` : saturate
- `+!`, `-!`, `*!` : trap (assert no overflow)

Division (`/`) and remainder (`%`) do not accept overflow suffixes; division‑by‑zero is handled by contract (`requires b != 0`) or runtime panic.
For signed division, `MIN / -1` (where `MIN` is the most negative value of the type) always traps regardless of the type's overflow policy. This is a representability issue, not a standard overflow, and the compiler will attempt to prove this case unreachable via range analysis.

The compiler uses range analysis and type invariants to statically eliminate overflow checks where possible.

### Pointers and References
- Raw pointer type: `Ptr<size = SizeType, pointee = PointeeType>`
  - `size`: type that determines the pointer's own width (e.g., `UInt<16>`).
  - `pointee`: the type it points to.
- Syntactic sugar: `*T` is a platform‑word‑sized pointer to `T`.
- References: `&T` (immutable), `&mut T` (mutable). References are checked at compile time and do not support pointer arithmetic. No null references are allowed; use `Option<&T>` for nullable semantics.
- **Exclusive borrow**: `&mut T` is an exclusive, non‑copyable borrow. While it is live, the original variable is frozen—neither readable nor writable—preventing data races statically.

### Explicit Lifetime Parameters
When the compiler cannot infer lifetimes, you may annotate them explicitly:
```posita
def process<'a>(x: &'a mut Data, y: &'a Data) -> &'a mut Result { ... }
```
Lifetime annotations are verified by the borrow checker; mismatches cause compile errors. They serve only for disambiguation.

### Reference‑Counted Type
`Rc<T>` provides shared ownership. The compiler verifies that every `clone()` is paired with a corresponding `drop()` along all control‑flow paths, preventing leaks. The net change in reference count across any function boundary can be specified in contracts.

### Strings and Byte Slices
Posita distinguishes between UTF‑8 text and arbitrary bytes:
- **`Str`**: A built‑in UTF‑8 string slice type. It is an immutable view, guaranteed to contain valid UTF‑8.
- **`String`**: A standard‑library provided, mutable, owning UTF‑8 string type.
- **`[Byte]`**: A byte slice that makes no encoding guarantees.
- Literals: `"hello"` has type `&Str`, `b"hello"` has type `&[Byte]`. Conversion requires explicit casts.

### Type‑level Default Values
Every type can declare a default value that is automatically assigned to any variable of that type that is not explicitly initialized. This eliminates all "uninitialized variable" bugs.
```posita
type MyInt = Int<8> with default = 1;
```
The default value **must satisfy any type invariants**; otherwise the compiler will reject the type definition.

**Semantically Sensitive Defaults and `no_default`**:
For types where a "blank" default is semantically dangerous, you can forbid implicit initialization with `with no_default`:
```posita
type OwnedFd = Int<32>
    invariant n >= 0
    with no_default;   // must explicitly initialize
```
Declaring `set fd: OwnedFd;` will then be a compile‑time error.

### Construction Validation
A type may specify a validation function that is automatically called after every explicit construction.
```posita
type Config = struct {
    timeout: UInt<32>,
} with validate = |c: &Config| -> Result<(), Error> {
    if c.timeout > 1000 {
        return Err(Error::TimeoutTooLarge);
    }
    Ok(())
};
```
The compiler inserts a call to the `validate` closure immediately after any `Config { ... }` expression, and a compile‑time error is raised if the result is not handled.

### Self‑Referential Types
A type may refer to itself through `Self` inside its definition:
```posita
type ListNode = struct {
    value: Int<32>,
    next: Option<Ptr<size=UInt<64>, pointee=Self>>,
};
```

### Type Invariants
A type may define an `invariant` clause that all valid instances must satisfy. The compiler verifies or enforces the invariant at every construction point.
```posita
type NonZeroInt = exists n: Int<32>
    invariant n != 0;
```
The `exists` keyword introduces a name for the value being constrained. This name can be used in the invariant expression and is erased at runtime.

**Implicit invariant propagation**: All functions automatically inherit `requires` that their parameters satisfy type invariants, and `ensures` that their return values satisfy type invariants. This eliminates redundant contract repetition.

### Composite Types
- Arrays: `[T; N]` (fixed size), `[T]` (slice, usually behind a reference).
- Tuples: `(T1, T2, ...)`
- **Empty Tuple (`()`)**: The empty tuple `()` is both a type and a value. It serves as Posita's unit type—a concrete, constructible value that carries no information. It is used as a generic placeholder when a type parameter must be instantiated but no meaningful value is needed (e.g., `Iterator<Item = ()>`). Unlike the `!` (never) type, `()` is a normal value that can be passed, returned, and stored.
- Structs:
  ```posita
  type Point = struct {
      x: Int<32>,
      y: Int<32> with default = 0,
  }
  ```
- Enums (algebraic):
  ```posita
  type Option<T> = enum {
      None,
      Some(T),
  }
  ```
  An enum can provide a custom error message that is emitted when a `match` does not cover all variants:
  ```posita
  type State = enum {
      Init,
      Running,
      Stopped,
  } with missing_match = "You must handle all three states: Init, Running, and Stopped.";
  ```
  The `@exhaustive` attribute on an enum forces all `match`, `if let`, and `while let` sites to be exhaustive, preventing future variants from being silently ignored.

### Layout Control Attributes
Fine‑grained control for hardware registers and protocols:
- **Packing**: `@packed` removes all padding between fields, ensuring minimal memory footprint. **Restriction:** `@packed` cannot be applied to structs that contain reference types (`&T`, `&mut T`, `&[T]`, `&mut [T]`). If you need a compact layout with indirection, use `Ptr` instead.
- **Endianness**: `@endian(little)` or `@endian(big)`.
- **Bit order**: `@bit_order(lsb_to_msb)` or `@bit_order(msb_to_lsb)`. This attribute only applies to bit‑fields inside `@packed` structs.
- **Alignment**: `@align(N)` overrides natural alignment. `N` must be a power of two.
- **Padding**: `@pad(byte_count)` inserts explicit padding bytes.

Example:
```posita
@packed @endian(big) @bit_order(msb_to_lsb)
type IPv4Header = struct {
    version: UInt<4>,
    ihl:     UInt<4>,
    dscp:    UInt<6>,
    ecn:     UInt<2>,
};
```
Posita natively supports bit‑fields (`UInt<4>`, etc.) inside `@packed` structs. The `@bit_order` attribute controls the order in which bit‑fields fill bytes.

### Language Attributes
The following attributes are not layout‑specific but affect language semantics or tooling behavior:

| Attribute | Applies to | Description |
|-----------|------------|-------------|
| `@no_copy` | Type | Prevents automatic `Copy` derivation |
| `@deprecated("msg")` | Function, Type | Marks an API as deprecated |
| `@experimental` | Function, Type | Marks a feature as experimental |
| `@inline` | Function | Forces inlining at call sites; compile error if not possible |
| `@noinline` | Function | Prevents inlining of the function |
| `@must_use` | Function, Type | Compiler warns if the return value is silently discarded. `Result` and `Option` are implicitly `@must_use`. |
| `@must_handle(Variant1, ...)` | Function (returning `Result`) | Compiler warns if the caller does not explicitly match or catch the listed error variants. Provides finer‑grained error accountability. |
| `@tailrec` | Function | Verifies that all recursive calls are in tail position and enforces tail‑call optimization; compile error if not possible |
| `@lemma` | Function | Marks a `comptime` proof helper that returns assertions for the SMT solver |
| `@apply_lemma(...)` | Function | Applies a lemma (by name) to supply the SMT solver with additional assertions during verification |
| `@trusted` | Function | Establishes a trust boundary for `unsafe` operations; requires `requires`/`ensures` contracts |
| `@link_proof(path, hash)` | Function (`@trusted`) | References an external formal proof (e.g., Coq/ATS script) and records its SHA‑256 hash, supplementing the verification of `@trusted` code. `capsa audit` verifies the file exists and matches the hash. |
| `@pure` | Function | No side effects; result depends only on arguments |
| `@io(read)` | Function | May perform input operations |
| `@io(write)` | Function | May perform output operations |
| `@io` | Function | May perform any I/O (equivalent to `@io(read, write)`) |
| `@alloc` | Function | May perform dynamic memory allocation |
| `@no_alloc` | Function | Guarantees no dynamic allocation |
| `@no_alloc_error` | Function | Guarantees no allocation on error paths, including `From` conversions |
| `@no_panic` | Function | Guarantees the function never panics; compiler verifies no `trap`, bounds checks, or calls to non‑`@no_panic` functions |
| `@runtime_check` | Function | Defers contract checking to runtime, even if arguments are compile‑time known. Overrides strict mode locally. |
| `@cfg(condition)` | Module, Function | Conditional compilation with `all`, `any`, `not` combinators. The condition may refer to target platform, features, etc. Only paths that compile under the configuration are permitted in strict mode. |
| `@hint(assertion)` | Function, Loop | Provides a hint to the SMT solver to guide proof search. Hints must be accompanied by a meta‑contract that proves the hint itself is valid. |
| `@exhaustive` | Enum | Requires all `match`, `if let`, and `while let` on the enum to be exhaustive. |
| `@debug_info` | Function, Module | Controls which symbols are emitted into debug information. Supports minimal exposure for safety‑critical deployments. |
| `@audit_log` | Function | Marks a function whose runtime contract violations must be written to an immutable audit log. The storage backend is defined by the standard library; tamper‑evident integrity (e.g., hash chains) is strongly recommended. |
| `@interrupt(irq, priority?)` | Function | Marks an interrupt handler; implicitly `@no_alloc`, `@no_panic`, return type `!`. |

**Attribute compatibility and precedence**: When multiple attributes are combined, the compiler follows a strict ordering:
1. `@cfg` is evaluated first (determines existence of the item).
2. Effect annotations (`@pure`, `@io`, `@alloc`, `@no_alloc`, `@no_alloc_error`, `@no_panic`) are checked for consistency.
3. Contract‑related attributes (`@trusted`, `@runtime_check`, `@link_proof`, `@lemma`) are processed.
4. Code‑generation attributes (`@inline`, `@noinline`, `@tailrec`) are applied last.

Incompatible combinations (e.g., `@pure` + `@runtime_check`, `@pure` + `@io`, `@runtime_check` + `@lemma`, `@trusted` + `@runtime_check`) are rejected at compile time.

### Type Attributes
Access compile‑time properties using `'`:
- `x'len` – length of array/slice
- `x'size` – bit width
- `x'align` – alignment
- `x'first`, `x'last` – first/last index (for arrays)
- `T'default` – the default value of type `T` (usable in `comptime`)

### Compile‑Time Layout Reflection
The built‑in function `layout_of!(T)` returns a compile‑time `LayoutDescriptor` for type `T`, describing the exact size, alignment, and field offsets. This is essential for `comptime` code that verifies or generates hardware‑specific memory layouts.
```posita
comptime {
    set desc = layout_of!(IPv4Header);
    // desc.size, desc.align, desc.fields[0].offset, etc.
}
```
`layout_of!` is a `comptime`‑only operation and thus requires `!`.

### Type Inference and Capture
- `set a = 42;` infers `a` as `Int<32>` (default integer width).
- **Type capture**:
  ```posita
  set auto[<T>..] = expression;
  ```
  Binds the type of `expression` to the compile‑time name `T`. The `..` is a placeholder for future extensions.

### Ghost Variables
Ghost variables are declared with the `ghost` keyword and exist only at compile time. They participate in contracts and invariants but are completely erased from runtime code. Their scope follows normal block scoping; they cannot affect runtime control flow (e.g., `if ghost_var` is illegal outside contracts).
```posita
def get_adult_names(users: &[User]) -> Vector<String>
    requires users'len > 0
    requires exists user in users where user.age >= 18
    ensures result'len >= 1
{
    set mut names = Vector<String>::new();
    ghost set mut found_adult: bool = false;

    for i in 0..users'len
        invariant if found_adult { names'len >= 1 } else { true }
        invariant (exists u in users[0..i] where u.age >= 18) implies found_adult
    {
        set user = users[i];
        if user.age >= 18 {
            names.push(user.name.clone());
            found_adult = true;
        }
    }
    return names;
}
```
Note: When iterating over arrays/slices, prefer the explicit index form (`for i in 0..arr'len`) if you need to refer to the index in contracts or invariants.

---

## Safety Guarantees, Effect Annotations, and `unsafe`

### The "No Undefined Behavior" Promise
Posita eliminates language‑level undefined behavior in safe code: no uninitialized variables, no integer overflow UB, no null pointer dereference, no out‑of‑bounds access, no data races, no invalid enum values, no misaligned or invalid pointer casts.

### Effect Annotations
Functions may be annotated with fine‑grained effect markers to describe their side effects. These annotations are checked by the compiler and serve as auditable documentation for reviewers.

- **`@pure`**: The function has no side effects and its result depends only on its arguments.
- **`@io(read)`**: The function may perform input operations (reading files, network, etc.).
- **`@io(write)`**: The function may perform output operations.
- **`@io(read, write)`** or simply **`@io`**: The function may perform any I/O.
- **`@alloc`**: The function may perform dynamic memory allocation.
- **`@no_alloc`**: The function guarantees no dynamic allocation.
- **`@no_alloc_error`**: The function guarantees no allocation on any error path. All `From` conversions reachable via `?` in error paths must also be `@no_alloc`.
- **`@no_panic`**: The function never panics. The compiler statically verifies the absence of overflow traps, bounds‑check failures, or calls to non‑`@no_panic` functions. Callers are not required to be `@no_panic` themselves; the guarantee is internal to the function body.
- **`@trusted`**: The function contains `unsafe` operations and establishes a trust boundary; it must carry `requires`/`ensures` contracts.
- **`@audit_log`**: The function must write any runtime contract violation to an immutable audit log. The log storage backend is provided by the standard library; tamper‑evident integrity (e.g., hash chains) is strongly recommended.

In strict mode the compiler verifies that effect annotations are consistent across call chains. A function calling an `@io(write)` function must itself be annotated at least `@io(write)`.

**Closure effect inference**: A closure's effect annotation is the union of the effects of its captured variables and its body. The compiler automatically derives this union and annotates the closure type.

### The `unsafe` Keyword
`unsafe` allows escape hatches (inline assembly, raw pointer manipulation, C FFI). All `unsafe` operations must be encapsulated in an `@trusted` function with `requires`/`ensures` contracts. The compiler trusts these contracts as axioms, and they are auditable via `capsa audit`.

In Strict Mode, `unsafe` blocks are completely forbidden, guaranteeing UB‑free by construction.

### `@trusted` Nesting
`@trusted` functions may call other `@trusted` functions, but the entire call chain is tracked by `capsa audit`. In strict mode, additional checks apply:
- `@link_proof` may be required to reference an external formal proof (e.g., a Coq file with its SHA‑256 hash).
- `@comptime_test` can provide concrete test cases that are executed at compile time.
- If neither is possible, the dependency must be explicitly marked as `trusted = true` in `posita.toml`, indicating that it has been manually audited.

### `extern "C"` ABI Rules
All `extern "C"` function declarations are **inherently unsafe** and can only be called inside `unsafe` blocks or `@trusted` functions. When a reference type (`&T`, `&[T]`, `&mut T`, `&mut [T]`) appears in the signature of an `extern "C"` function, the compiler automatically converts it to the corresponding raw C pointer for the call:
- `&T` and `&[T]` become `*const T`
- `&mut T` and `&mut [T]` become `*mut T`
The length component of slices is discarded. The caller is responsible for ensuring that the data satisfies any additional C‑side requirements (e.g., null termination for `puts`). This conversion is deterministic and does not constitute an implicit runtime check.

---

## Traits and Implementations

Posita uses traits to define shared behavior, similar to Rust traits or Haskell typeclasses. Traits may contain function signatures, associated types (with optional defaults), and default method implementations.

### Defining a Trait
```posita
trait Add<Rhs = Self> {
    type Output;
    def add(self: &Self, rhs: &Rhs) -> Self::Output;
}

trait Iterator {
    type Item;
    type Distance = usize;  // associated type with default value
    fn next(&mut self) -> Option<Self::Item>;
}

trait Default {
    def default() -> Self;
}

trait Drop {
    def drop(&mut self);
}

trait Copy: Clone { }  // marker trait, no methods

trait Clone {
    def clone(&self) -> Self;
}
```
Within a trait definition, `type Name = DefaultType;` declares an associated type with a default value. Implementations may override the default or use it as‑is.

### Implementing a Trait
```posita
impl Add for Int<32> {
    type Output = Int<32>;
    def add(self: &Int<32>, rhs: &Int<32>) -> Int<32> { /* … */ }
}

impl Drop for UniqueToken {
    def drop(&mut self) { /* release token */ }
}
```

### Automatic Clone for Copy Types
When `Copy` is automatically derived (or manually implemented), the compiler also automatically derives `Clone` with `fn clone(&self) -> Self { *self }`.

### Associated Types and Projections
Associated types are accessed via the `::` operator: `T::Output`. In `where` clauses they are used to constrain relationships between types.

### Dynamic Dispatch
When static dispatch is not possible (e.g., heterogeneous collections), the `dyn` keyword creates a trait object: `dyn Trait`. Trait objects use dynamic dispatch via a vtable and may incur a heap allocation. Their use is explicit to ensure reviewers can identify runtime dispatch points.
```posita
let handlers: [dyn Fn(Int<32>) -> Int<32>; 10];
```

### Built‑in Traits
The following traits are defined by the language and automatically implemented for applicable types:
- `Add`, `Sub`, `Mul`, `Div`, `Rem` – arithmetic operators
- `Eq`, `Ord` – comparison operators
- `Copy` – bitwise copy semantics
- `Clone` – explicit duplication
- `Default` – default value construction
- `Drop` – destructor
- `Display` – formatting
- `Serialize`, `Write` – I/O traits (standard library)

---

## Concurrency Model

### Tasks
Basic concurrency unit:
```posita
set worker = task { /* code */ };
```

### Channels
Typed, bounded, synchronous channels:
```posita
set (sender, receiver) = Channel<Int<32>>::new(16);
```

### `async`/`await`
For non‑blocking operations, functions return `Future<T>`:
```posita
async def fetch_data(url: &[Byte]) -> Result<Data, Error> { ... }
def main() -> Result<(), Error> {
    set data = await fetch_data("...")?;
}
```
The `await` keyword marks a suspension point that must be visible to reviewers.

### Interrupts
Interrupt handlers are modeled as special, highly constrained tasks:
```posita
@interrupt(irq = 14)
def timer_handler() -> ! {
    // must be @no_alloc, @no_panic; cannot have custom parameters or return a value
}
```
- Interrupt handler functions must have the return type `!` (never return).
- They are implicitly `@no_alloc` and `@no_panic`.
- The compiler automatically generates the interrupt vector table from all `@interrupt` annotations, respecting platform ABI and optional `@layout` attributes.

### Task Isolation
The `isolate` block guarantees that the enclosed code does not access any external mutable state. The compiler verifies this property statically, enabling safe concurrent access patterns.
```posita
isolate {
    // only local variables and immutable external references are accessible
    set x = compute_safe();
}
```
`isolate` blocks can be called from multiple interrupt contexts without data‑race risks.

---

## Modules and Imports

### Visibility
All symbols are **private by default**. Use `pub` to export.

### Importing
- `import std::io;` — qualified access `io::puts(...)`.
- `import std::io as my_io;` — alias.
- `from std::io import { puts, File };` — selective.
- `import std::{io, fs};` — nested paths.
- **Wildcard import is prohibited** (`import *` is illegal).

### Package Layout
Project defined by `posita.toml`. Compiler resolves full dependency graph.

---

## Language Versioning and Deprecation

### Editions
Each file declares its edition:
```posita
edition = "2026";
```

### Deprecation
```posita
@deprecated("Use `new_method` instead")
def old_method() { ... }
```

### Experimental Features
```posita
@experimental
type NewInteger = Int<128>;
```
Requires `--enable-experimental` flag.

### Conditional Compilation
Modules, functions, or type definitions can be conditionally compiled using the `@cfg` attribute with the `all`, `any`, `not` combinators:
```posita
@cfg(all(target_os = "linux", target_arch = "x86_64"))
def platform_specific() { ... }

@cfg(any(feature = "logging", debug))
def maybe_log(msg: &Str) { ... }

@cfg(not(target_os = "windows"))
def unix_only() { ... }
```
In strict mode, all compilation paths must be provably reachable under at least one configuration.

---

## Compile‑Time Execution

Posita does not support procedural macros or any form of syntax‑level code generation outside the `comptime` system. All compile‑time code generation is performed by `comptime` functions, which are type‑checked, sandboxed, and subject to resource limits. This ensures that all code—whether executed at runtime or at compile time—is visible, auditable, and subject to the same safety guarantees.

**Safety restrictions:** `comptime` code runs in a sandboxed environment. It is **prohibited** from performing any of the following:
- File system writes
- Network access
- Spawning processes
- Calling `@io(write)` functions or any `@trusted` functions that perform I/O

Violations are detected at compile time and cause a hard error. This guarantees that `comptime` evaluation is purely a deterministic, side‑effect‑free transformation of the source program.

**Proof obligations for generated code**: Code generated by `comptime` is subject to the same verification standards as hand‑written code. Any `@trusted` function produced by `comptime` must include corresponding `@link_proof` or `@comptime_test` evidence; otherwise, compilation fails in strict mode.

### comptime Blocks
```posita
comptime { /* compile‑time code */ }
```

### comptime Functions
```posita
comptime def eval_polynomial(...) -> Int<64> { ... }
```
Calls to `comptime` functions **must** be marked with `!` to make the compile‑time nature explicit at the call site:
```posita
set result = eval_polynomial!(coeffs, x);
```
This ensures that reviewers can immediately identify where compile‑time evaluation occurs.

### Comptime Tests
`@comptime_test` blocks execute at compile time and are used to validate `@trusted` code against concrete inputs:
```posita
@comptime_test
def test_trusted_function() {
    let result = some_trusted_function(test_input);
    assert(result == expected_output);
}
```
Test failures cause a compile error. `@comptime_test` functions are stripped from the final binary. Contract‑verification counterexamples can be automatically converted into such tests.

### Type Factories
`comptime` functions can return `type`:
```posita
comptime def make_vector(Elem: type, N: usize) -> type {
    return [Elem; N];
}
type Vec4f = make_vector!(Float<32>, 4);
```
`type` is a first‑class value in `comptime` contexts and can be passed, returned, and stored.
Execution is bounded by step/memory limits to prevent non‑termination.

### Compile‑Time Reflection (`@typeInfo`)
```posita
comptime {
    set info = @typeInfo!(MyStruct);
    for field in info.fields { /* generate code */ }
}
```
Public reflection (`@typeInfo!`) only exposes `pub` items. Internal reflection (`@typeInfo!` with full access) is available only inside `@trusted` comptime blocks.

### Built‑in Compile‑Time Utilities

The following utilities are available in `comptime` contexts:

- **`assert(condition)`**: Evaluates `condition` at compile time. If the condition is `false`, compilation halts with an error message. `assert` is stripped from the final binary and cannot be used for runtime checks.
- **`@compile_error("msg")`**: Unconditionally halts compilation with the given message. Typically used in `comptime` to reject invalid code generation or type combinations.

### Optimization Hooks (advanced)
```posita
comptime {
    set plan = optimize(target = my_function, strategy = q_learn(config));
    plan.apply();
}
```

### Lemma Functions
Lemma functions are `comptime` functions that return auxiliary assertions to assist the SMT solver:
```posita
@lemma
comptime def pow2_induction_hint(n: Int<32>) -> [Assertion; 2] { ... }
```
They are invoked by placing **`@apply_lemma(pow2_induction_hint)`** on the target function's definition, which causes the compiler to inject the returned assertions into the SMT solver's context during verification of that function.

---

## Statements and Expressions

> **Statement termination:** Every statement in Posita must be terminated with a semicolon. There is no implicit semicolon insertion, and the compiler will reject any statement that lacks a terminating `;`. The only exception is the final expression in a block that is used as an expression (e.g., the tail expression of an `if` expression or `match` arm), which may omit the semicolon to denote that its value is yielded to the enclosing expression. Function return values must always be produced via an explicit `return` statement; a trailing expression without a semicolon at the end of a function body does not constitute an implicit return.

### Variable Declaration
```posita
set identifier : Type = expression;   // full form
set identifier = expression;          // type inference
set identifier : Type;                // uses type's default (unless no_default)
```
Variables are immutable by default. Use `mut` for mutability:
```posita
set mut x = 0;
x = x + 1;
```

### `let` Bindings
`let` is a restricted, immutable‑only variant of `set` that additionally supports pattern destructuring. A `let` binding is always immutable; there is no `let mut`. It must always be explicitly initialized and cannot rely on a type's default value.

- Simple binding: `let x = expr;` (equivalent to `set x = expr;`)
- Tuple destructuring: `let (x, y) = tuple_expr;`
- Struct destructuring: `let Point { x, y } = point_expr;`
- Enum variant destructuring with mandatory `else`:
  ```posita
  let Some(value) = opt_expr else { return Err(Error::None); };
  ```
  The `else` block must diverge (via `return`, `leave with`, `panic`, etc.).

### Assignment
`x = value` (requires `mut`). Compound assignments: `+=`, `-=`, `*=`, etc.

### Control Flow
- **Conditional**: `if condition { ... } else { ... }`
- **`if` as an expression**: `if` is an expression that returns a value. All branches must return the same type, and the `else` branch is mandatory when used as an expression:
  ```posita
  set msg = if x > 5 { b"big" } else { b"small" };
  ```
- **if let**: `if let Some(val) = opt { ... }`
- **while let**: `while let Some(item) = iter.next() { ... }`
- **Loops**: `for item in iterable { ... }`, `while condition { ... }`, `loop { ... }`
- **Leaving loops**: `leave;` (exits `for`, `while`, or `loop`), `leave 'label;`, `continue;`
- **Never type** (`!`): The `!` type represents a computation that never returns (e.g., `panic`, infinite loops, or divergent control flow). It can be used as the return type of functions that do not terminate normally, enabling the compiler to perform control‑flow analysis.
**`!` vs `()`**: The `!` type represents the absence of *any* value and is uninhabited—no expression can produce a value of type `!`. The `()` type is a unit value, representing an *empty* but existing value. Use `!` for functions that never return or whose return value must never be used; use `()` when a generic instantiation requires a concrete, ignorable value. This distinction eliminates the need for a separate `unit` keyword.

### Functions
```posita
def function_name(param1: Type1, param2: Type2) -> ReturnType { ... }
```
Default parameter values: `def f(x: Int<32> = 0) { ... }`

### Pattern Matching
```posita
match value {
    pattern1 | pattern2 if guard_condition => expression1,
    pattern3 => { statements },
    _ => default,
}
```
- **Or patterns**: `pattern1 | pattern2` matches if either pattern matches. Both patterns must bind the same set of variables with compatible types.
- **Guards**: The `if guard_condition` clause filters matches. Guard expressions must be `@pure` and free of side effects.

### Named Scope Cleanup
Multiple cleanup actions can be declared with `scope_cleanup` and explicitly triggered before the end of scope:
```posita
def process() -> Result<(), Error> {
    set file = File.open("data.bin")?;
    scope_cleanup @close_file { file.close(); }

    // ... operations ...
    trigger @close_file;  // explicitly close before normal return
    return Ok(());
} // at scope exit, any remaining scope_cleanup actions are executed
```
`scope_cleanup` actions are executed in LIFO order. Unlike `defer`, they are named and can be invoked early.

### Finally Blocks
```posita
def process() -> Result<(), Error> {
    set file = File.open("data.bin")?;
    return Ok(());
} finally {
    file.release_buffer();  // infallible cleanup: no ?, leave with, return, or panic allowed
}
```
`finally` blocks are reserved for infallible cleanup. For fallible cleanup, use `scope_cleanup` and handle errors explicitly.

### Expressions
Arithmetic: `+`, `-`, `*`, `/`, `%`. The operators `+`, `-`, `*` accept optional overflow suffixes (`%`, `?`, `!`).
Bitwise: `&`, `|`, `^`, `<<`, `>>`, `~`
Logical: `and`, `or`, `not`
Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
Dereference: `*ptr`
Address‑of: `&var`
Cast: `value as NewType` (safe), `value as! NewType` (bitcast)
Rounding suffixes for float‑to‑int conversion: `round`, `trunc` (default), `ceil`, `floor`.

**Operator precedence** (highest to lowest):

| Precedence | Operators | Associativity |
|------------|-----------|---------------|
| 1 (highest) | `*`, `/`, `%` | left-to-right |
| 2 | `+`, `-` | left-to-right |
| 3 | `<<`, `>>` | left-to-right |
| 4 | `&` | left-to-right |
| 5 | `^` | left-to-right |
| 6 | `\|` | left-to-right |
| 7 | `==`, `!=`, `<`, `>`, `<=`, `>=` | left-to-right |
| 8 | `and` | left-to-right |
| 9 | `or` | left-to-right |
| 10 | `not` | right-to-left (prefix) |
| 11 (lowest) | `..`, `..=` | left-to-right |

**`as!` layout compatibility**: The compiler verifies that the source and target types have the same size and alignment via `layout_of!`, or, in the case of truncation, that the truncated value does not violate the target type's `invariant`. All uses of `as!` are flagged by `capsa audit` for mandatory human review.

### Move Semantics
The `move` keyword explicitly transfers ownership of a non‑`Copy` value. It may be used in:
- Assignments: `set target = move source;`
- Function arguments: `consume(move value);`
- Closure captures: `let closure = |...| capture value by move { ... };`
After a move, the source variable is invalidated and any subsequent use is a compile‑time error. The compiler will not insert a `drop` call for the moved‑from variable.

---

## Error Handling

### The `Result` Type
`Result<T, E>` is a built‑in enum:
```posita
type Result<T, E> = enum { Ok(T), Err(E) }
```

### Propagating Errors with `?`
```posita
def read_file(path: &[Byte]) -> Result<String, FileError> {
    set file = File.open(path)?;   // propagates FileError
    // ...
}
```
No type‑erased errors; fully monomorphized, zero overhead.

When a function is annotated `@no_alloc_error`, the compiler verifies that every `?` propagation path uses only `From` implementations that are themselves `@no_alloc`. If any conversion in the error path could allocate, the `?` operator is rejected. Simple enum‑to‑enum conversions without payload transformations automatically satisfy this requirement.

### Handling Errors Locally with `catch`
A `catch` expression has the type `T` where the preceding expression has type `Result<T, E>`. Each branch of `catch` must either diverge (via `leave with`, `return`, `panic`, etc.) or produce a value of type `T`. The patterns in `catch` branches are the enum variant names directly (e.g., `|IoError| { ... }`), not qualified paths, because the error type is already known from the expression.

Example with divergence:
```posita
def process() -> Result<(), ProcessError> {
    set data = fetch() catch {
        |NetworkError as e| { log(e); leave with Err(ProcessError::NetworkFail); }
        |ParseError => { return Err(ProcessError::BadData); }
    };
    return Ok(());
}
```
- **`as` binding**: Binds the error value to a local variable for inspection or logging.
- **Arrow shorthand** (`=>`): When a branch body is a single expression (e.g., a return value), the `=>` syntax can replace curly braces for brevity.

Example with a fallback value (non‑diverging):
```posita
def fetch_or_default() -> Data {
    set data = fetch() catch {
        |NetworkError| { cached_default }
    };
    return data;
}
```

### Early Return with `leave with`
`leave with` is a structured, non‑local exit that returns an error from the current function. The expression after `leave with` must be of type `Err(E)` where `E` matches the error type of the enclosing function's `Result`.

### Explicit Error Paths and `From` Conversions
`From` implementations allow automatic error conversion via `?`. All conversions are statically known.

### Fatal Errors: `panic`
Only within `unsafe` contexts or `comptime` blocks.

### Fine‑Grained Error Accountability (`@must_handle`)
By default, `@must_use` ensures that a `Result` is not silently discarded. The `@must_handle` attribute provides finer control, requiring callers to explicitly handle specific error variants:
```posita
@must_handle(NetworkError, ParseError)
def fetch() -> Result<Data, Error> { ... }

// Caller that ignores NetworkError or ParseError will get a compiler warning
let data = fetch() catch {
    |NetworkError as e| { log(e); cached_default }  // handled
    |ParseError => { return Err(...) }               // handled
    // other errors can still fall through
};
```
In strict mode, the warning becomes an error.  
**Interaction with `?`**: Propagating an error via `?` does **not** count as handling for the purposes of `@must_handle`. The caller must explicitly match the variant or use `delegate()` to explicitly pass the responsibility upstream.

---

## Contracts and Constraints

### Function Contracts
- `requires`: a precondition.
- `ensures`: a postcondition. Applies to **all** exit paths unless explicitly qualified with `on Ok(...)` or `on Err(...)`.
- `terminates`: a termination measure. For recursive functions, the argument specified must strictly decrease on each recursive call and have a lower bound, ensuring the recursion always terminates.

Contracts are evaluated in the mathematical integer domain (ideal precision) by default. For floating‑point contracts, `ensures with ieee_precision` switches to IEEE 754 semantics.
```posita
def sqrt_approx(x: Float<64>) -> Float<64>
    requires x >= 0.0
    ensures abs(result * result - x) <= 1e-10
    ensures with ieee_precision
{ ... }
```

**Asynchronous contracts**: For `async` functions, the postcondition may be further qualified with `ensures on_timeout => ...` and `ensures on_cancel => ...`. The `on_timeout` clause describes what holds when the operation times out (the runtime scheduler is responsible for triggering this path). The `on_cancel` clause describes what holds when the operation is cancelled before completion. The compiler verifies that the function body respects these guarantees along each corresponding control‑flow path.

To refer to the value of an argument at function entry, use the `old()` function:
```posita
def increment(x: &mut Int<32>)
    ensures *x == old(*x) + 1
{ *x += 1; }
```

```posita
def divide(a: Int<32>, b: Int<32>) -> Int<32>
    requires b != 0
    ensures result * b == a
{ return a / b; }

def factorial(n: Int<32>) -> Int<32>
    requires n >= 0
    ensures result > 0
    terminates n
{
    if n == 0 { 1 } else { n * factorial(n - 1) }
}
```

### Deferred Contract Checking
When the `@runtime_check` attribute is applied to a function, its `requires` and `ensures` checks are performed at runtime, even if the arguments are compile‑time constants. This is useful during development or when interfacing with untrusted inputs. In strict mode, the compiler emits a warning for each `@runtime_check` function, noting that the function has not been statically verified.
```posita
@runtime_check
def debug_divide(a: Int<32>, b: Int<32>) -> Int<32>
    requires b != 0
    ensures result * b == a
{
    return a / b;   // will trap at runtime if b == 0
}
```

### Loop Invariants and Termination
Loops may specify an `invariant` and a `decreases` clause. The `decreases` expression must be a non‑negative integer that strictly decreases on each iteration, providing a proof that the loop always terminates.
```posita
def sum(arr: &[Int<32>]) -> Int<32>
    ensures result == fold(arr, 0, +)
{
    set mut total = 0;
    for i in 0..arr'len
        invariant total == fold(arr[0..i], 0, +)
        decreases arr'len - i
    { total += arr[i]; }
    return total;
}
```
Invariants and decreases clauses must appear immediately after the loop header, before the opening brace `{`.

### Built‑in Contract Functions
- `fold(arr, init, op)`: represents the left fold of array `arr` starting from `init` using binary operator `op`. It is a pure function used only in contracts and invariants. Its semantics are: `fold(arr, init, op) == op(...op(op(init, arr[0]), arr[1]), ..., arr[arr'len-1])`.
- `abs(x)`: absolute value of integer `x`.
- `old(expr)`: captures the value of `expr` at function entry, usable in `ensures` clauses.
- `forall` and `exists` quantifiers may appear in contracts:
  ```posita
  requires forall i in 0..arr'len: arr[i] > 0
  requires exists i in 0..arr'len: arr[i] == target
  ```

### Type Invariants (see Type System)

### Ghost Variables (see Type System)

### Generic Constraints with `where`
```posita
def serialize<T, S>(value: &T, stream: &mut S) -> Result<(), Error>
    where T: Serialize, S: Write, T::Format: Display
{ ... }
```

### Reusable Constraint Blocks (`constraint`)
```posita
constraint SortableContainer<C> { C: Container, C::Item: Ord, C::Item: Default }
def sort<C>(container: &mut C) where C: SortableContainer { ... }
```
Constraints may be combined with `+`: `def f<T>() where T: A + B { ... }`. Constraints may also reference other constraints, forming a hierarchy.

---

## Standard Library Initialization Philosophy
Posita avoids implicit conversions between literals and dynamic containers. Compile‑time functions provide ergonomic initialization. The `...args: T` syntax declares a variadic parameter; inside the function `args` is accessible as a `&[T]` slice.
```posita
comptime def vec<T>(...args: T) -> Vector<T> {
    set v = Vector::with_capacity(args'len);
    for a in args { v.push(a); }
    return v;
}
set v = vec!(1, 2, 3);  // Creates Vector<Int<32>> via explicit comptime function
```

---

## Compiler Diagnostics

The compiler provides three diagnostic levels for contract verification failures:

| Level | Name | Purpose | Output |
|-------|------|---------|--------|
| **L1** | **Locate** | Quickly find the problem | Source file, line, column, violated clause |
| **L2** | **Explain** | Understand the cause | Counterexample in Posita syntax, data‑flow trace |
| **L3** | **Debug** | Deep analysis | SMT‑LIB script, solver statistics, unsat core |

Developers select the level via `capsa build --diagnostic-level=N`. When a contract fails, the compiler attempts to produce a minimal counterexample and a specific suggestion for how to fix it.

---

## Undefined Behavior Prevention
| UB Category | Prevention Mechanism |
|-------------|----------------------|
| **Uninitialized variables** | Type‑level defaults; `no_default` forces explicit initialization where needed. |
| **Integer overflow (signed)** | Default `trap`; configurable per‑type/operator; compiler range analysis can eliminate checks. |
| **Division by zero** | Contract `requires b != 0` enforced at compile time or via runtime panic. |
| **Null pointer dereference** | References are non‑null; `Option<&T>` enforces checking before use. |
| **Dangling pointers** | Borrow checker; default copy semantics reduce accidental moves. |
| **Data races** | `&mut T` is exclusive, `&T` is shared; compile‑time borrow rules. |
| **Buffer overflow** | Array access checked via contract or runtime bounds; pointer arithmetic constrained to explicit `Ptr` types. |
| **Invalid enum values** | Type invariants. |
| **Misaligned pointers** | `Ptr` type carries alignment; safe casts check alignment; `as!` demands proof of alignment. |
| **Type punning / transmute** | `as!` requires compile‑time verification of layout compatibility. |
| **Non‑terminating loops** | `decreases` clause on loops proves strict decrease of a non‑negative measure. |
| **Non‑terminating recursion** | `terminates` clause on functions proves strict decrease of the argument. |
| **Infinite loops at compile time** | `comptime` execution bounded by step and memory limits. |
| **Static initialization order** | Compile‑time evaluation ensures constant initializers; module‑level variables use zero‑init or type defaults. |

Strict Mode forbids `unsafe` entirely; all contracts must be statically proven.

---

## Complete Example
```posita
edition = "2026";

type AppError = enum { IoError, ValidationError(&[Byte]) }

extern "C" def puts(s: &[Byte]) -> Int<32>;

@trusted
def safe_puts(msg: &[Byte]) -> Result<(), AppError>
    requires msg'len > 0 && msg[msg'len - 1] == 0
    ensures result == Ok(())
{
    unsafe { puts(msg); }
    Ok(())
}

type EmployeeId = UInt<16> with default = 0;
type Salary = Int<32> with default = 0;
type Age = exists n: UInt<8> invariant n >= 18 && n <= 120 with default = 25;
type OwnedFileDescriptor = exists n: Int<32> invariant n >= 0 with no_default;

type UniqueToken = struct { id: EmployeeId }
impl Drop for UniqueToken { def drop(&mut self) { } }

type Department = enum { Engineering, Sales, HumanResources }

type Employee = struct {
    id: EmployeeId, age: Age, salary: Salary, dept: Department,
    name: &[Byte],
}

type PositiveInt = exists n: Int<32> invariant n > 0 with default = 1;

constraint AddableDefault<T> { T: Add<T, Output = T>, T: Default, T: Copy }

def sum_list<T>(items: &[T]) -> T where T: AddableDefault {
    set mut total: T = T::default();
    for item in items { total = total + *item; }
    return total;
}

def calculate_bonus(salary: Salary, multiplier: PositiveInt) -> Salary
    requires salary >= 0
    ensures result >= salary
{
    set mut result = salary;
    set mut i: Int<32> = 0;
    while i < multiplier
        invariant result == salary + i * salary
        decreases multiplier - i
    { result = result + salary; i = i + 1; }
    return result;
}

def process_employee(emp: &mut Employee, bonus_mult: PositiveInt) -> Result<(), AppError>
    requires emp.salary >= 0
{
    if emp.age > 100 { return Err(AppError::ValidationError(b"Employee too old\0               ")); }
    safe_puts(b"Processing employee...\0") catch {
        |IoError as e| { log(e); leave with Err(AppError::IoError); }
        |ValidationError => { return Err(AppError::IoError); }
    };
    emp.salary = calculate_bonus(emp.salary, bonus_mult);
    return Ok(());
} finally { }

def make_employee_report(emp: &Employee) -> &[Byte] {
    set auto[<BonusType>..] = calculate_bonus(emp.salary, 1);
    comptime {
        set info = @typeInfo!(BonusType);
        match info {
            TypeInfo::Int { bits, .. } => { if bits > 32 { @compile_error!("Bonus type too large"); } }
            _ => {}
        }
    }
    return b"Report generated";
}

def sum_manual(arr: &[Int<32>]) -> Int<32> {
    set mut total: Int<32> = 0;
    set mut idx: usize = 0;
    while idx < arr'len { total += arr[idx]; idx += 1; }
    return total;
}

def main() -> Result<(), AppError> {
    set fd: OwnedFileDescriptor = 3;
    set e1 = Employee { id = 1, age = 30, salary = 50000, dept = Department::Engineering, name = b"Alice" };
    set mut e2 = Employee { id = 2, age = 45, salary = 75000, dept = Department::Sales, name = b"Bob" };
    set bonus_mult: PositiveInt;
    process_employee(&mut e2, bonus_mult) catch { /* ... */ };
    set salaries: [Salary; 3] = [e1.salary, e2.salary, 60000];
    set total_salary = sum_manual(&salaries);
    safe_puts(b"=== Employee Summary ===\0") catch { |_| {} };
    set e3 = e1;
    set token1 = UniqueToken { id = 10 };
    set token2 = move token1;
    return Ok(());
}
```

---

## Relationship to Other Languages
- **From Ada**: explicit representation control, attribute syntax, contract‑based verification, default initialization, traceability.
- **From Rust**: `Result`‑based error handling (without type erasure), `if let`, `match`, trait‑like generics, borrow checker.
- **From Zig**: The `comptime` mechanism and the philosophy of moving work to compile time are direct inspirations. Posita adds the `!` call marker and integrates `comptime` with SMT‑based contract verification, going beyond what Zig's comptime offers.
- **From ATS**: The concept of encoding invariants in types and the ambition to eliminate runtime errors through static proofs have influenced Posita. ATS is a secondary inspiration after Ada, Rust, and Zig.
- **Unique to Posita**: bit‑width parameterized integers with explicit overflow control, orthogonal pointer sizes, type‑level defaults with invariants and `no_default`, `leave`/`leave with`, type capture, fully static error monomorphization, compile‑time type factories, reflection, structured `finally` blocks, systematic UB elimination, optional strict mode, ghost variables, specification tags, named scope cleanup, construction validation, lemma functions, fine‑grained effect annotations, deferred contract checking (`@runtime_check`), layout reflection (`layout_of!`), proof hints (`@hint`), fine‑grained error accountability (`@must_handle`), tiered diagnostics, implicit invariant propagation, `old()` expressions, fixed‑precision rationals, MMIO types, interrupt vector generation, and more.

---

## Design Q&A

**Q: Why "ultra‑static typing"? How is it different from dependent types or refinement types?**
A: Ultra‑static typing resolves all representation details and behavioral guarantees at compile time. Types depend only on compile‑time constants, avoiding runtime evidence. All checks are provable at compile time in strict mode, leaving zero runtime validation overhead.

**Q: Why copy semantics by default? Doesn't it harm performance?**
A: Copy semantics eliminate moved‑from invalid states, crucial for safety‑critical reasoning. Small types copy at register speed. Large types should use references. Explicit `move` is available as an optimization. For detailed rationale, see the "Move vs. Copy" design note in DESIGN.md.

**Q: How does `with default` differ from Rust's `Default`?**
A: Rust's `Default` is a trait requiring explicit call; Posita's is a type‑level attribute that automatically initializes. Defaults must satisfy type invariants. `with no_default` forces explicit initialization.

**Q: What is the role of `unsafe`? Is Posita "pure safe"?**
A: `unsafe` is confined, requiring `@trusted` with contracts. Strict Mode disallows `unsafe` entirely, making the program pure safe and UB‑free.

**Q: How do you handle C ABI?**
A: `extern "C"` functions are inherently unsafe and require `unsafe` blocks or `@trusted` wrappers with contracts. Standard library provides safe wrappers. Strict Mode can forbid them entirely.

**Q: How do you prevent array out‑of‑bounds?**
A: Compiler proves bounds statically using contracts and invariants; failing proof, it either reports an error (strict mode) or inserts explicit runtime checks (never UB).

**Q: What is the compile‑time performance model?**
A: Fast mode (default) skips SMT proofs; strict mode enables them with time budgets. Incremental compilation and caching keep build times manageable.

**Q: How does the module system work?**
A: File‑based, private by default, no wildcard imports. Dependencies are explicitly declared in `posita.toml`, enabling whole‑program analysis.

**Q: How does Posita support formal verification?**
A: Contracts (`requires`/`ensures`), type invariants, loop invariants are verified by SMT solver. In strict mode, all contracts must be proven.

**Q: What about linear types and usage counts?**
A: (Planned) `linear` and `@consume(N)` will allow compile‑time tracking of resource consumption, enabling safe protocols. Currently under design.

**Q: Can I hand‑write proofs in Posita?**
A: Posita provides `@lemma` functions to supply auxiliary assertions that assist the SMT solver, and `@comptime_test` blocks to validate `@trusted` code against concrete inputs at compile time. For external formal proofs, `@link_proof` can reference Coq/ATS files that are distributed with the package and verified by `capsa`.

**Q: What are ghost variables?**
A: Ghost variables (`ghost set mut x = ...`) exist only at compile time and are erased from the final binary. They enable complex proofs without any runtime overhead.

**Q: What happens when data comes from an untrusted runtime source (e.g., a JSON file)?**
A: Posita requires explicit parsing and validation at the boundary. Once the data is converted into a strongly‑typed struct, all subsequent code enjoys full static guarantees.

**Q: Does Posita support runtime contract checking?**
A: Yes, the global `--runtime-contracts` flag and the per‑function `@runtime_check` attribute allow deferring contract checks to runtime. This is useful for debugging or when interfacing with untrusted inputs.

**Q: How does Posita interact with WCET analysis?**
A: Posita generates highly deterministic code and exports detailed metadata (CFG, loop bounds, etc.) via `--emit-timing-info`. Professional WCET tools like aiT consume this metadata to produce precise timing results.

**Q: How does Posita compare to Moonbit's formal verification?**
A: Moonbit's verification is lighter, targeting cloud/Wasm safety. Posita provides hardware‑level precision (bit‑widths, endianness, interrupt safety) and a complete audit chain from requirements (`@spec`) to object code, suitable for DO‑178C and ISO 26262 certification.

**Q: Does Posita have `if` expressions?**
A: Yes. `if` is an expression that returns a value. All branches must return the same type, and the `else` branch is mandatory.

**Q: Does Posita support ternary operators (`?:`)?**
A: No. The `if` expression already serves this purpose, and `?` is reserved for error propagation.

**Q: What is the `!` symbol used for?**
A: `!` marks calls to `comptime` functions, making compile‑time evaluation explicit at the call site. This is consistent with `?` for error propagation and `await` for asynchronous suspension.

**Q: Can `@trusted` functions call each other?**
A: Yes. The entire trust chain is tracked by `capsa audit`. Additional checks (e.g., `@link_proof`, `@comptime_test`) may be required in strict mode to justify the trust.

**Q: How are specification and requirements traced?**
A: Documentation comments with `@spec`, `@requirement`, `@rationale`, and `@trace` link code directly to system requirements. The `capsa spec` command generates a traceability matrix suitable for certification.

**Q: Does Posita support procedural macros?**
A: No. All compile‑time code generation is performed by `comptime` functions, which are type‑checked and sandboxed. This ensures that all code is visible, auditable, and subject to the same safety guarantees.

**Q: How are layout attributes used?**
A: `@packed` removes padding, `@endian` controls byte order, `@bit_order` controls bit field order, `@align` overrides alignment, and `@pad` inserts explicit padding. These are essential for hardware‑facing code.

**Q: Is `@trusted` marked at the call site?**
A: No. `@trusted` is a declaration‑site attribute on function definitions. Call sites remain clean; the trust boundary is established once, at the function definition, and tracked by `capsa audit`.

**Q: What is the difference between `set` and `let`?**
A: `set` is the general variable declaration, defaulting to immutability but allowing `set mut`. `let` is a restricted, always‑immutable form that additionally supports pattern destructuring (tuples, structs, enum variants with mandatory `else`). `let` always requires an explicit initializer and cannot use a type's default value. Use `set` for general purposes; use `let` when you need destructuring or want to enforce immutability at a glance.

**Q: What are the fine‑grained effect annotations?**
A: `@pure`, `@io(read)`, `@io(write)`, `@io`, `@alloc`, `@no_alloc`, `@no_alloc_error`, `@no_panic`, `@audit_log`, and `@trusted` describe a function's side effects. The compiler checks these annotations for consistency, giving reviewers a precise summary of what a function can do without reading its body.

**Q: How are slices passed to `extern "C"` functions?**
A: `&[T]` and `&mut [T]` are automatically converted to `*const T` and `*mut T` at the ABI boundary, with the length component discarded. This is a deterministic, compiler‑enforced rule; the programmer must ensure the data meets the C function's expectations (e.g., null termination).

**Q: What is `usize`?**
A: `usize` is a built‑in type alias for `UInt<N>` where `N` is the target platform's pointer width. It exists for convenience when writing code that deals with array indices and pointer‑sized values, but explicit `UInt<32>` or `UInt<64>` may always be used instead.

**Q: What happens with `MIN / -1` for signed integers?**
A: This case always traps regardless of the type's overflow policy. It is a representability issue, not a standard overflow. The compiler attempts to statically prove this case unreachable when possible.

**Q: How do I define and implement a trait?**
A: Use `trait TraitName { ... }` to define methods and associated types. Use `impl TraitName for MyType { ... }` to provide implementations. See the "Traits and Implementations" section for details.

**Q: What does `exists` mean in a type definition?**
A: `exists n: BaseType invariant P(n)` introduces a name for the value being constrained. It is required when the invariant expression refers to the value itself. The bound name is erased at runtime.

**Q: How do I use `@apply_lemma`?**
A: Place `@apply_lemma(lemma_fn_name)` on a function definition. The compiler will call the `comptime` lemma function and inject its returned assertions into the SMT solver during verification of that function.

**Q: What are `...` parameters?**
A: The `...args: T` syntax declares a variadic parameter. Inside the function body, `args` is accessible as a `&[T]` slice containing all supplied arguments. This is typically used in `comptime` functions for container initializers.

**Q: How does `move` work?**
A: `move` explicitly transfers ownership of a non‑`Copy` value in assignments, function arguments, or closure captures. After a move, the source variable is invalidated and any subsequent use is a compile‑time error. The compiler will not call `drop` on the moved‑from variable.

**Q: Are `extern "C"` functions safe to call?**
A: No. All `extern "C"` functions are inherently unsafe and can only be called inside `unsafe` blocks or `@trusted` functions. This ensures all FFI calls are auditable.

**Q: How should `catch` patterns be written?**
A: `catch` branches use the enum variant names directly (e.g., `|IoError| { ... }`), without qualifying them with the enum type. The error type is already known from the expression being caught. The `as` keyword can bind the error value to a local variable. For branches consisting of a single expression, the arrow shorthand `=>` can replace curly braces: `|ParseError => return Err(...)`.

**Q: How do I enforce that callers handle specific errors?**
A: Annotate the function with `@must_handle(Variant1, ...)`. The compiler will warn if a caller does not explicitly match or catch those variants. This keeps critical errors visible without forcing exhaustive matching of all variants.

**Q: What is dynamic dispatch and how do I use it?**
A: Dynamic dispatch is available via `dyn Trait` objects, which use a vtable for method calls. Use them explicitly when static dispatch is not possible, such as storing heterogeneous types in a collection. This makes runtime dispatch visible to reviewers.

**Q: How do `decreases` and `terminates` work?**
A: `decreases expr` on a loop guarantees termination by requiring `expr` to be a non‑negative integer that strictly decreases each iteration. `terminates arg` on a recursive function requires the specified argument to strictly decrease on each recursive call with a lower bound. Both are used by the compiler to prove termination, which is essential for safety‑critical systems and WCET analysis.

**Q: How do I write integer literals with a specific bit width?**
A: Use the `42i32` suffix for `Int<32>` or `0xFFu8` for `UInt<8>`. This is syntactic sugar for `42: Int<32>` and `0xFF: UInt<8>`, respectively. The type is fully checked by the compiler.

**Q: What is the `isolate` block?**
A: An `isolate` block guarantees that the enclosed code does not access any external mutable state. The compiler verifies this statically, enabling safe concurrent execution from multiple interrupt or task contexts.

**Q: What is conditional compilation (`@cfg`)?**
A: The `@cfg(condition)` attribute allows modules, functions, or type definitions to be conditionally included based on target platform, features, or other compile‑time predicates. Conditions can be combined with `all`, `any`, `not`. In strict mode, all compilation paths must be reachable under some configuration.

**Q: What is `layout_of!`?**
A: `layout_of!(T)` is a compile‑time reflection function that returns a `LayoutDescriptor` detailing the exact size, alignment, and field offsets of type `T`. It is essential for verifying hardware‑facing layouts in `comptime` code.

**Q: How do `@hint` attributes help with proofs?**
A: `@hint(assertion)` provides a suggestion to the SMT solver during contract verification. It guides the solver's search without affecting runtime behavior or safety guarantees. The hint itself must be proven valid (meta‑contract) before being injected.

**Q: What are `@must_handle` and how is it different from `@must_use`?**
A: `@must_use` warns if the entire return value is ignored. `@must_handle` is more specific: it lets a library author name particular error variants that the caller must explicitly handle (via `catch` or `match`), even if other variants are handled by a wildcard. This ensures critical failure modes never slip through silently.

**Q: What is `@exhaustive`?**
A: `@exhaustive` on an enum definition forces all `match`, `if let`, and `while let` on that enum to be exhaustive. This prevents new variants added during evolution from being silently ignored in existing code.

**Q: How do contracts interact with error paths?**
A: By default, `ensures` applies to all exit paths. You can specialize with `ensures on Ok(result) => ...` and `ensures on Err(error) => ...` to make guarantees specific to success or failure returns.

**Q: How does the compiler help me debug contract failures?**
A: The compiler provides three diagnostic levels (L1: locate, L2: explain with counterexamples, L3: full SMT‑LIB dump). Use `capsa build --diagnostic-level=N` to select the depth of information.

**Q: When is `as!` safe to use?**
A: The compiler verifies that source and target types have the same size and alignment, or that a truncation does not violate the target type's `invariant`. All uses of `as!` are flagged for human review by `capsa audit`.

**Q: How is audit logging handled?**
A: Functions marked `@audit_log` must write contract violations to an immutable audit log. The storage backend is provided by the standard library; tamper‑evident mechanisms (e.g., hash chains) are strongly recommended.

**Q: How are `Rational` values converted to `Float`?**
A: Conversion is explicit (`as Float<64>`) and uses round‑to‑nearest ties‑to‑even by default. This prevents silent precision loss.

**Q: Why can't I use `Regex` types in contracts?**
A: SMT solvers have limited string theory capabilities, making automatic proof unreliable. `Regex` types are for runtime use only; contracts should use simpler predicates.

**Q: What effects do MMIO operations have?**
A: MMIO reads and writes are implicitly `@io(read)` and `@io(write)`, respectively. The compiler ensures they are only used in appropriate effect contexts.

**Q: How are interrupt handlers constrained?**
A: Interrupt handlers must have the return type `!`, are implicitly `@no_alloc` and `@no_panic`, and cannot have custom parameters. The compiler enforces these rules and generates the interrupt vector table from `@interrupt` annotations.

**Q: Why does Posita have `!` but no `unit` type?**
A: Posita reuses the empty tuple `()` as its unit type. `()` is a regular value that can be constructed and passed around, satisfying generic placeholders. The `!` type is reserved for the true absence of a value—it is uninhabited and signals that a computation never completes normally. This separation keeps the type system orthogonal (no special `unit` keyword) while making the semantics of “no value” explicit.
