# Posita Language Syntax
**Document revision: 2026-06-10** (working draft, not a frozen specification)

> [!NOTE]
> This document version tracks its own edits. It does **not** correspond to a language specification release.
> Posita itself is in pre-alpha; the syntax is under active design and may change without notice.

## Design Philosophy
Posita is a **ultra-static, systems programming language** where the programmer explicitly *posits* every representation detail: bit widths, pointer sizes, default values, error paths, and even overflow behavior. All decisions are made visible in source and enforced at compile time with zero runtime overhead.

- **Explicit over implicit**: No hidden ABI, no type-erased errors, no null pointers, no implicit overflow.
- **Compile-time over run-time**: Error handling, defaults, optimizations, and reflection are resolved statically.
- **Readable as documentation**: English keywords (`def`, `set`, `leave`, `catch`), not cryptic symbols.
- **No undefined behavior**: Every operation either succeeds with defined semantics or is rejected at compile time.

---

## Lexical Structure

### Keywords
`def`, `set`, `type`, `with`, `default`, `return`, `if`, `else`, `for`, `in`, `while`, `loop`, `leave`,
`comptime`, `import`, `as`, `true`, `false`, `auto`, `and`, `or`, `not`, `sizeof`, `alignof`,
`Result`, `Option`, `catch`, `panic`, `unsafe`, `let`, `finally`,
`where`, `requires`, `ensures`, `invariant`, `constraint`, `move`, `dyn`, `by`, `copy`, `ref`, `mut`, `wrap`, `saturate`, `trap`, `Self`

`Int`, `UInt`, `Ptr` are built-in type constructors, not reserved words.

### Identifiers
`[a-zA-Z_][a-zA-Z0-9_]*`

### Literals
- Integers: `42`, `0xFF`, `0b1010`
- Floats: `3.14`, `2.5e-3`
- Characters: `'a'` (UTF-8, type `UInt<8>`)
- Byte strings: `b"hello\n"`
- Strings: `"hello"` (type `[Byte]`)
- Booleans: `true`, `false`

### Comments
- Line comment: `// ...`
- Block comment: `/* ... */`

---

## Type System

### Value Semantics
Posita defaults to **copy semantics** for all types that implement the `Copy` trait. A type automatically derives `Copy` if it consists only of integers, floats, other `Copy` types, and does **not** implement `Drop`. This ensures that “copy is a trivial bitwise replication” and eliminates “moved-from” invalid states. Large types like `Vector<T>` are **not** `Copy` because they manage resources (they implement `Drop`). Explicit `move` semantics are available via the `move` keyword for ownership transfer optimization.

To prevent automatic `Copy` derivation for a type that would otherwise qualify, either implement `Drop` (even with an empty body) or use the `@no_copy` attribute:
```plaintext
@no_copy
type LargeStruct = struct { data: [Int<8>; 1024] };
```

Manual implementation of `Copy` is allowed for types where bitwise replication is semantically sound, even if large. By doing so, the programmer explicitly accepts the performance characteristics.

### Bit-width Parameterized Integers
- Signed: `Int<bits>`, `bits` must be compile-time constant, 1..64.
- Unsigned: `UInt<bits>`.
- Example: `Int<13>`, `UInt<8>`.

### Overflow Behavior
Integer overflow is never undefined. The default overflow policy is `trap` (compile-time error in strict mode, runtime panic otherwise). Programmers can override at the type level:
```plaintext
type WrapCount = Int<32> with overflow = wrap;     // two's complement wrap
type SatCount  = Int<32> with overflow = saturate;  // saturation
type StrictCount = Int<32> with overflow = trap;    // trap (default)
```
Or at the operator level with suffixes:
- `+%`, `-%`, `*%` : wrap
- `+?`, `-?`, `*?` : saturate
- `+!`, `-!`, `*!` : trap (assert no overflow)

The compiler uses range analysis and type invariants to statically eliminate overflow checks where possible.

### Pointers and References
- Raw pointer type: `Ptr<size = SizeType, pointee = PointeeType>`
  - `size`: type that determines the pointer's own width (e.g., `UInt<16>`).
  - `pointee`: the type it points to.
- Syntactic sugar: `*T` is a platform-word-sized pointer to `T`.
- References: `&T` (immutable), `&mut T` (mutable). References are checked at compile time and do not support pointer arithmetic. No null references are allowed; use `Option<&T>` for nullable semantics.

### Type-level Default Values
Every type can declare a default value that is automatically assigned to any variable of that type that is not explicitly initialized. This eliminates all “uninitialized variable” bugs.
```plaintext
type MyInt = Int<8> with default = 1;
```
When you write `set x: MyInt;`, the compiler automatically initializes `x` to `1`. The default value **must satisfy any type invariants**; otherwise the compiler will reject the type definition. For large aggregates (arrays, structs), the default initializes every element/member. This guarantee extends to all memory, making Posita programs free of undefined behavior from uninitialized data.

### Self-Referential Types
A type may refer to itself through `Self` inside its definition, enabling linked structures without infinite compile-time recursion:
```plaintext
type ListNode = struct {
    value: Int<32>,
    next: Option<Ptr<size=UInt<64>, pointee=Self>>,
};
```
The `Self` keyword resolves to the enclosing type during semantic analysis. For mutually recursive types, separate definitions with forward aliases work as usual.

### Type Invariants
A type may define an `invariant` clause that all valid instances must satisfy. The compiler verifies or enforces the invariant at every construction point, including default initialization.
```plaintext
type NonZeroInt = exists n: Int<32>
    invariant n != 0;

type PositiveArray = exists arr: [Int<32>; N]
    invariant forall i in 0..arr'len: arr[i] > 0;
```
When a value of such a type is constructed, the compiler statically proves the invariant or requires a runtime check (depending on safety mode). Once established, the invariant is assumed to hold for subsequent uses, enabling optimizations like eliding redundant checks.

### Composite Types
- Arrays: `[T; N]` (fixed size), `[T]` (slice, usually behind a reference).
- Tuples: `(T1, T2, ...)`
- Structs:
  ```plaintext
  type Point = struct {
      x: Int<32>,
      y: Int<32> with default = 0,
  }
  ```
- Enums (algebraic):
  ```plaintext
  type Option<T> = enum {
      None,
      Some(T),
  }
  ```

### Type Attributes
Access compile-time properties using `'`:
- `x'len` – length of array/slice
- `x'size` – bit width
- `x'align` – alignment
- `x'first`, `x'last` – first/last index (for arrays)
- `T'default` – the default value of type `T` (usable in `comptime`)

### Type Inference and Capture
- `set a = 42;` infers `a` as `Int<32>` (default integer width).
- **Type capture** (unique to Posita):
  ```plaintext
  set auto[<T>..] = expression;
  ```
  Binds the type of `expression` to the compile-time name `T` for later use.

---

## Statements and Expressions

### Variable Declaration
```plaintext
set identifier : Type = expression;   // full form
set identifier = expression;          // type inference
set identifier : Type;                // uses type's default value (guaranteed initialized)
```
Variables are immutable by default. Use `mut` for mutability:
```plaintext
set mut x = 0;
x = x + 1;
```

### Assignment
`x = value` (requires `mut`). Compound assignments: `+=`, `-=`, `*=`, etc.

### Control Flow (All Structured, No Hidden Jumps)
- **Conditional**:
  ```plaintext
  if condition {
      // ...
  } else if condition {
      // ...
  } else {
      // ...
  }
  ```
- **if let** (pattern match shortcut):
  ```plaintext
  if let Some(val) = optional_value {
      // use val
  } else {
      // handle None
  }
  ```
  Can also chain multiple conditions: `if x > 0 and let Some(y) = opt { ... }`.
- **while let**:
  ```plaintext
  while let Some(item) = iter.next() {
      // process item
  }
  ```
- **Loops**:
  ```plaintext
  for item in iterable {
      // ...
  }
  while condition {
      // ...
  }
  loop {   // infinite loop
      // ...
  }
  ```
- **Leaving loops**:
  ```plaintext
  leave;          // exit innermost loop
  leave 'label;   // exit a labelled loop
  continue;       // next iteration
  ```
  Labels are defined with `'label:` before a loop.

### Functions
```plaintext
def function_name(param1: Type1, param2: Type2) -> ReturnType {
    // body
    return expression;
}
```
- Default parameter values: `def f(x: Int<32> = 0) { ... }`
- Return type can be omitted for auto-inference (not recommended for public APIs).

### Pattern Matching (full)
```plaintext
match value {
    pattern1 => expression1,
    pattern2 => { statements },
    _ => default,
}
```

### Finally Blocks for Cleanup
A `finally` block attached to a function body guarantees execution of cleanup logic after the function exits, regardless of whether it exits via `return`, `leave`, `leave with`, or `?` propagation. The block is placed after the function body, making all exit paths visible and structured.

**Restrictions**: The `finally` block must not contain `?`, `leave with`, or any operation that could change the function’s return value. It is intended for **infallible** cleanup only (e.g., memory release, resetting hardware). Fallible cleanup must be handled explicitly before the `finally` block or within it by silently logging the error (using `catch` without propagation).
```plaintext
def process() -> Result<(), Error> {
    set file = File.open("data.bin")?;
    // ... operations that may fail ...
    return Ok(());
} finally {
    // infallible cleanup only
    file.release_buffer();
    // If close may fail, handle before finally or log silently:
    // file.close() catch { |e| log(e) };
}
```
Multiple `finally` blocks are not allowed; put all cleanup in a single block.

### Expressions
- Arithmetic: `+`, `-`, `*`, `/`, `%` (with optional overflow suffixes `%`, `?`, `!`)
- Bitwise: `&`, `|`, `^`, `<<`, `>>`, `~`
- Logical: `and`, `or`, `not`
- Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
- Dereference: `*ptr`
- Address-of: `&var`
- Cast: `value as NewType` (safe), `value as! NewType` (bitcast)

---

## Error Handling

Posita treats error handling as a **first-class, statically enforced discipline** with zero hidden cost. It has no exceptions, no `null` pointers, and no type-erased error objects.

### The `Result` Type
All recoverable errors use `Result<T, E>`, which is a built-in enum:
```plaintext
type Result<T, E> = enum {
    Ok(T),
    Err(E),
}
```

### Propagating Errors with `?`
Use `?` to propagate the error to the caller. The error type must match the function's declared error type. If the error types differ but a `From` implementation exists, the compiler automatically inserts the conversion (see “Explicit Error Paths” below).
```plaintext
def read_file(path: &[Byte]) -> Result<String, FileError> {
    set file = File.open(path)?;   // propagates FileError
    // ...
}
```
The compiler enforces that every `?` is compatible with the function's error signature. There is no type-erased error representation; the specific error type is always known at compile time, enabling full monomorphization and zero overhead.

### Handling Errors Locally with `catch`
`catch` provides pattern-matching on the error value directly after a fallible expression:
```plaintext
def process() -> Result<(), ProcessError> {
    set data = fetch() catch {
        |NetworkError| {
            log("network error");
            leave with ProcessError::NetworkFail;
        }
        |ParseError| {
            leave with ProcessError::BadData;
        }
    };
    // use data
    return Ok(());
}
```
Inside a `catch` block, you can:
- Use `leave with <error>` to return an error from the enclosing function.
- Perform cleanup or logging and then propagate the same error via `?`.
- Convert the error into a different type and continue.

### Early Return with `leave with`
`leave with` is a structured, non-local exit that returns an error from the current function. It is only allowed inside `catch` blocks or where the compiler can statically verify that the enclosing function returns `Result`.
```plaintext
def example() -> Result<Int<32>, MyError> {
    set x = dangerous_op() catch {
        |err| leave with err;   // directly return Err(err)
    };
    // ...
}
```

### Explicit Error Paths and `From` Conversions
To manage error type combinatorics, Posita allows explicit `impl From<SpecificError> for MyError` declarations. The `?` operator will automatically apply such conversions when the error types differ. This is **not** implicit magic: all possible conversions are statically known, and tooling can display them (via `posita check --show-error-paths` or IDE hover). The conversion must be a pure function; its definition serves as documentation of the error path. There is no requirement to write `.map_err(Into::into)?` unless the programmer desires explicitness at the call site.

### Fatal Errors: `panic`
For unrecoverable situations (e.g., out of memory, arithmetic overflow in checked mode), Posita provides `panic`. It can only be used within explicitly marked `unsafe` contexts or in `comptime` blocks (where it causes a compilation error). Normal application code should avoid `panic` and use `Result` instead.

### No Null Pointers
All plain references (`&T`, `&mut T`) are non-nullable. Nullable semantics are expressed explicitly via `Option<&T>`.
```plaintext
def maybe_get() -> Option<&Int<32>> {
    // ...
}
```

---

## Contracts and Constraints

Posita supports design-by-contract through `requires` and `ensures` clauses, and general type-level constraints via `invariant` and `constraint`.

### Function Contracts
- `requires`: a precondition that must hold when the function is called. The compiler may prove it statically or insert a runtime check (depending on safety mode).
- `ensures`: a postcondition that must hold upon return. The compiler attempts to verify it; unprovable postconditions cause a compile error in strict mode.

**Important**: Contract expressions are evaluated in the **mathematical integer domain** (ideal precision), not under the underlying type’s overflow policy. This ensures that the contract describes logical truth irrespective of machine arithmetic. If the type uses `overflow = wrap`, the contract must still hold mathematically; otherwise the compiler will reject it (or issue a warning in non-strict mode). For wrap semantics, consider using range contracts instead of exact algebraic equations.
```plaintext
def divide(a: Int<32>, b: Int<32>) -> Int<32>
    requires b != 0
    ensures result * b == a   // uses mathematical integers, independent of overflow policy
{
    return a / b;
}
```

### Loop Invariants
Loops may specify an `invariant` to aid automated verification:
```plaintext
def sum(arr: &[Int<32>]) -> Int<32>
    ensures result == fold(arr, 0, +)
{
    set mut total = 0;
    for i in 0..arr'len
        invariant total == fold(arr[0..i], 0, +)
    {
        total += arr[i];
    }
    return total;
}
```
The invariant describes a property that holds before each iteration and after the loop, enabling the compiler to prove the `ensures` clause.

### Type Invariants (already described in Type System)
Type invariants are part of the type definition and are enforced at construction time.

### Generic Constraints with `where`
For complex generic requirements, use `where` clauses to express trait bounds and associated type constraints:
```plaintext
def serialize<T, S>(value: &T, stream: &mut S) -> Result<(), Error>
    where T: Serialize,
          S: Write,
          T::Format: Display,
          S::Error: ConvertibleTo<Error>
{
    // implementation
}
```

### Reusable Constraint Blocks (`constraint`)
To avoid repetition, group constraints into a named `constraint` block:
```plaintext
constraint SortableContainer<C> {
    C: Container,
    C::Item: Ord,
    C::Item: Default,
}

def sort<C>(container: &mut C)
    where C: SortableContainer
{
    // implementation
}
```
Constraint blocks can include any bound allowed in a `where` clause, including nested constraints.

---

## Compile-Time Execution

### comptime Blocks
```plaintext
comptime {
    // code executed during compilation
}
```

### comptime Functions
```plaintext
comptime def eval_polynomial(coeffs: &[Int<32>], x: Int<32>) -> Int<64> {
    // ...
}
```

### Type Factories
A `comptime` function may return a `type`, enabling code-driven type construction:
```plaintext
comptime def make_vector(Elem: type, N: usize) -> type {
    return [Elem; N];
}

type Vec4f = make_vector(Float<32>, 4);
```
Type factories are evaluated entirely at compile time and can use the full reflection capabilities to generate complex types.

**Safety and termination**: `comptime` execution is subject to a configurable step limit and memory budget to prevent infinite recursion. Directly self-recursive type factories that do not have a terminating base case are detected and reported as a compile-time error. For self-referential types, use the `Self` keyword inside type definitions rather than type factories.

### Compile-Time Reflection (`@typeInfo`)
Posita provides a built-in `@typeInfo` function, accessible only in `comptime` contexts, to introspect any type's structure:
```plaintext
comptime {
    set info = @typeInfo(MyStruct);
    for field in info.fields {
        // generate serialization code
    }
}
```
This returns a `TypeInfo` enum that describes the type completely (fields, variants, bit widths, etc.), enabling serialization, auto-formatting, and other compile-time code generation.

### Optimization Hooks (advanced)
```plaintext
comptime {
    set plan = optimize(target = my_function, strategy = q_learn(config));
    plan.apply();
}
```

---

## Modules and Imports
```plaintext
import module_name;
import module_name as alias;
from module import item1, item2;
```

---

## Standard Library Initialization Philosophy
Posita avoids implicit conversions between compound literals and dynamic containers. Instead, standard library types provide explicit constructors. For convenient container initialization, `comptime` functions with variable compile-time arguments are used:
```plaintext
comptime def vec<T>(...args: T) -> Vector<T> {
    set v = Vector::with_capacity(args'len);
    for a in args { v.push(a); }
    return v;
}

set v = vec(1, 2, 3);  // Creates Vector<Int<32>> via explicit function
```
This approach preserves explicitness (no hidden allocations) while offering ergonomics through compile-time code generation.

---

## Undefined Behavior Prevention
Posita’s design eliminates entire categories of undefined behavior (UB) that plague C, C++, and even Rust (in `unsafe` code). The following table summarizes key protections.

| UB Category | Prevention Mechanism |
|-------------|----------------------|
| **Uninitialized variables** | Type-level defaults guarantee every variable is initialized at declaration. |
| **Integer overflow (signed)** | Default `trap` policy; configurable per-type/operator; compiler range analysis can statically eliminate checks. |
| **Division by zero** | Contract `requires b != 0` enforced at compile time or via runtime panic. |
| **Null pointer dereference** | References are non-null; `Option<&T>` enforces checking before use. |
| **Dangling pointers** | Borrow checker ensures references never outlive referent; default copy semantics reduce accidental moves. |
| **Data races** | `&mut T` is exclusive, `&T` is shared; compile-time borrow rules prevent simultaneous mutable access. |
| **Buffer overflow** | Array access checked via contract or runtime bounds; pointer arithmetic constrained to explicit `Ptr` types. |
| **Invalid enum values** | Type invariants guarantee only valid variants exist; `as!` cast requires provable compatibility. |
| **Misaligned pointers** | `Ptr` type carries alignment; safe casts check alignment; `as!` demands proof of alignment. |
| **Type punning / transmute** | `as!` requires compile-time verification that source and target layouts are compatible; otherwise rejected. |
| **Infinite loops at compile time** | `comptime` execution bounded by step and memory limits; self-recursive type factories detected. |
| **Static initialization order** | Compile-time evaluation ensures constant initializers; module-level variables use zero-init or type defaults. |

The compiler can be configured in **strict mode** where any construct that cannot be statically proven safe is rejected. This guarantees that a successfully compiled Posita program is free of the listed UB by construction.

---

## Complete Example
```plaintext
type Byte = UInt<8> with default = 0x00;
type BufPtr = Ptr<size = UInt<16>, pointee = Byte>;

def concat(left: &[Byte], right: &[Byte]) -> Result<BufPtr, AllocError>
    requires left'len + right'len <= max_alloc_size
    ensures result'len == left'len + right'len
{
    set total_len = left'len + right'len;
    set result     = alloc(BufPtr, total_len)?;
    set mut pos    = 0;

    for i in 0..left'len {
        result[pos] = left[i];
        pos = pos + 1;
    }

    for j in 0..right'len {
        if right[j] == 0xFF {
            leave;
        }
        result[pos] = right[j];
        pos = pos + 1;
    }

    return Ok(result);
} finally {
    // infallible cleanup only
    // free_any_temp();
}
```
This example combines bit-width parameterization, type defaults, `?` error propagation, `leave`, contracts, and a `finally` block for structured cleanup.

---

## Relationship to Other Languages
- **From Ada**: explicit representation control, attribute syntax, strong typing, English keywords, contract-based verification, default initialization.
- **From Rust**: `Result`-based error handling (without type erasure), `if let`, `match`, block expressions, trait-like generics, borrow checker.
- **Unique to Posita**: bit-width parameterized integers with explicit overflow control, orthogonal pointer sizes, type-level defaults with invariants, `leave`/`leave with`, type capture `auto[<T>..]`, fully static error monomorphization, compile-time type factories, reflection, structured `finally` blocks, and systematic elimination of undefined behavior.

---

## Design Q&A: Frequently Asked Questions

**Q: Why “ultra-static typing”? How is it different from dependent types or refinement types?**
A: Ultra-static typing is a paradigm where all representation details and behavioral guarantees are resolved at compile time. Unlike dependent types, Posita only allows types to depend on compile-time constants, avoiding runtime evidence. Unlike refinement types, Posita enforces that all checks (including type invariants and contracts) must be provable at compile time in strict mode, leaving zero runtime validation overhead. It combines the precision of Ada’s representation clauses with Zig’s compile-time reflection, but integrates them into a single, coherent type system.

**Q: Why copy semantics by default? Doesn’t it harm performance?**
A: Copy semantics ensure variables remain valid after assignment, eliminating “moved-from” states—a crucial property for safety-critical reasoning. Small types (integers, small structs) copy at register speed. For large types, Posita expects references (`&T`) to be used, and standard library containers like `Vector` are non-`Copy` (they implement `Drop`). Explicit `move` is available as an optimization, not the default. Users may also manually implement `Copy` for large types if they accept the cost, preserving explicitness.

**Q: How does Posita’s `with default` compare to Rust’s `Default` trait?**
A: Rust’s `Default` is a trait that must be explicitly called (`T::default()`); variables are not automatically initialized. Posita’s `with default` is a type-level attribute that automatically initializes every variable of that type at declaration. Furthermore, the default value is checked against type invariants, guaranteeing that no variable ever holds an invalid value. This eliminates all “use of uninitialized variable” UB without any boilerplate.

**Q: Why so many error handling keywords (`?`, `catch`, `leave with`, `finally`)? Isn’t it complex?**
A: Each construct has a distinct, orthogonal role: `?` for propagation, `catch` for local handling with pattern matching, `leave with` for structured early exit from a function, and `finally` for unconditional cleanup. Unlike Rust, Posita avoids `Box<dyn Error>` and type erasure, keeping error types monomorphized. The apparent “complexity” is the price of making every error path explicit and auditable, which is essential in safety-critical code.

**Q: Can Posita’s error handling lead to combinatoric explosion of error types?**
A: Yes, but Posita provides three mitigations: (1) `From` trait implementations allow automatic conversion to a common application error type via `?`; (2) `catch` blocks can immediately convert diverse errors into a uniform type; (3) `comptime` can automatically derive error enums and their `From` implementations. These tools keep the explosion manageable without sacrificing static visibility. All conversions are statically visible to tooling.

**Q: How do you define overflow behavior? Will it slow down programs?**
A: Overflow is never undefined. The default is `trap` (compile-time error or runtime panic). Programmers can override with `overflow = wrap` or `overflow = saturate` at the type level, or use operator suffixes like `+%` for wrap. The compiler performs extensive range analysis based on type invariants and contracts, statically eliminating overflow checks when it can prove overflow impossible. This gives safety without unnecessary runtime cost.

**Q: Does Posita need procedural macros? Why not?**
A: No. Procedural macros introduce non-local, hard-to-debug code generation, undermining Posita’s explicit-source philosophy. Posita’s `comptime` combined with reflection (`@typeInfo`) and type factories already covers all typical macro use cases (serialization, DSLs, boilerplate generation) within a type-checked, debuggable, safe environment.

**Q: How do I initialize a dynamic container like `Vector` with elements? Is there list initialization?**
A: Posita rejects implicit conversion from array literals to dynamic containers to avoid hidden allocations. Instead, standard library provides explicit constructors like `Vector::from_array`. For ergonomics, `comptime` functions with variable compile-time arguments (e.g., `vec(1,2,3)`) expand into explicit push operations, maintaining transparency and zero-cost abstraction.

**Q: Why does Posita use `def` and `set` instead of `fn` and `let`?**
A: `def` (define) and `set` (posit) reinforce the language’s core metaphor: everything is an explicit “positing” of a definition or a setting of a value. These keywords are consistent with the English-readable style inherited from Ada and avoid overloading symbols.

**Q: How does Posita prevent infinite loops in type factories?**
A: Compile-time evaluation is bounded by step and memory limits, and the compiler detects direct self-recursive type factories. For self-referential types, the `Self` keyword provides a safe and explicit way to express recursion without risking non-termination.

**Q: Do contracts work with wrapping overflow types?**
A: Contracts are evaluated using mathematical integer arithmetic, independent of the underlying type’s overflow policy. If a contract cannot hold under the chosen policy (e.g., `ensures result * b == a` with `overflow = wrap`), the compiler will reject it. In such cases, use range constraints or switch to `overflow = trap`.

**Q: What happens if a `finally` block tries to return an error?**
A: `finally` blocks are infallible by design. Using `?` or `leave with` inside them is a compile-time error. Fallible cleanup must be performed outside the `finally` block or logged silently without altering the function’s return value.

**Q: Can I make a large struct `Copy`?**
A: Yes, by explicitly implementing the `Copy` trait. The compiler will not automatically derive `Copy` for types that implement `Drop`, but you can opt in manually. This makes the copy cost visible in the type signature, and you accept responsibility for the performance impact.

**Q: Doesn’t `?` automatically converting errors via `From` hide the error path?**
A: No, because all `From` implementations are statically known and visible to tooling. You can always inspect the complete error conversion chain via `posita check --show-error-paths` or IDE features. This provides the convenience of automatic conversion without sacrificing auditability.

**Q: How does Posita eliminate undefined behavior?**
A: By making every potentially dangerous operation either statically provable or explicitly checked. Uninitialized variables are impossible due to type defaults. Overflow, division by zero, and null dereferences are trapped or prevented by contracts. Data races are eliminated by borrow checking. The strict mode rejects any code that cannot be proven safe at compile time.
