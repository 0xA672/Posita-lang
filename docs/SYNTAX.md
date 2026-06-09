# Posita Language Syntax
**Document revision: 2026-06-09** (working draft, not a frozen specification)

> [!NOTE]
> This document version tracks its own edits. It does **not** correspond to a language specification release.
> Posita itself is in pre-alpha; the syntax is under active design and may change without notice.

## Design Philosophy
Posita is a **ultra-static, systems programming language** where the programmer explicitly *posits* every representation detail: bit widths, pointer sizes, default values, and even error paths. All decisions are made visible in source and enforced at compile time with zero runtime overhead.

- **Explicit over implicit**: No hidden ABI, no type-erased errors, no null pointers.
- **Compile-time over run-time**: Error handling, defaults, and optimizations are resolved statically.
- **Readable as documentation**: English keywords (`def`, `set`, `leave`, `catch`), not cryptic symbols.

---

## Lexical Structure

### Keywords
`def`, `set`, `type`, `with`, `default`, `return`, `if`, `else`, `for`, `in`, `while`, `loop`, `leave`,
`comptime`, `import`, `as`, `true`, `false`, `null`, `auto`, `and`, `or`, `not`, `sizeof`, `alignof`,
`Result`, `Option`, `catch`, `panic`, `unsafe`, `if`, `let`, `while`, `let`, `defer`

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

### Bit-width Parameterized Integers
- Signed: `Int<bits>`, `bits` must be compile-time constant, 1..64.
- Unsigned: `UInt<bits>`.
- Example: `Int<13>`, `UInt<8>`.

### Pointers and References
- Raw pointer type: `Ptr<size = SizeType, pointee = PointeeType>`
  - `size`: type that determines the pointer's own width (e.g., `UInt<16>`).
  - `pointee`: the type it points to.
- Syntactic sugar: `*T` is a platform-word-sized pointer to `T`.
- References: `&T` (immutable), `&mut T` (mutable). References are checked at compile time and do not support pointer arithmetic. No null references are allowed; use `Option<&T>` for nullable semantics.

### Type-level Default Values
```plaintext
type MyInt = Int<8> with default = 1;
```
Any variable declared as `MyInt` without an explicit initializer will be automatically initialized to `1`.

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
set identifier : Type;                // uses type's default value
```
Variables are immutable by default. Use `mut` for mutability:
```plaintext
set mut x = 0;
x = x + 1;
```

### Assignment
`x = value` (requires `mut`). Compound assignments: `+=`, `-=`, `*=`, etc.

### Control Flow
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

### Deferred Execution (`defer`)
A `defer` block is executed at the end of the enclosing scope, regardless of how the scope is exited (normal return, `leave`, or error propagation). Multiple `defer` blocks execute in LIFO order. It is useful for resource cleanup.
```plaintext
def process() -> Result<(), Error> {
    set file = File.open("data.bin")?;
    defer file.close();
    // ... operations that may fail ...
    return Ok(());
}
```

### Expressions
- Arithmetic: `+`, `-`, `*`, `/`, `%`
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
Use `?` to propagate the error to the caller. The error type must match the function's declared error type.
```plaintext
def read_file(path: &[Byte]) -> Result<String, FileError> {
    set file = File.open(path)?;   // propagates FileError
    // ...
}
```
The compiler enforces that every `?` is compatible with the function's error signature. There is no `Box<dyn Error>` or other type-erased errors; the specific error type is always known at compile time, allowing complete monomorphization and zero overhead.

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
    return [Elem; N];   // or Array<Elem, N>
}

type Vec4f = make_vector(Float<32>, 4);
```
Type factories are evaluated entirely at compile time and can use the full reflection capabilities to generate complex types.

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

## Complete Example
```plaintext
type Byte = UInt<8> with default = 0x00;
type BufPtr = Ptr<size = UInt<16>, pointee = Byte>;

comptime def make_buffer(len: usize) -> type {
    return [Byte; len];
}

def concat(left: &[Byte], right: &[Byte]) -> Result<BufPtr, AllocError> {
    set total_len = left'len + right'len;
    set result     = alloc(BufPtr, total_len)?;
    defer if result != null { free(result); }  // optional cleanup
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
}
```
This example now includes `defer`, `?`, `leave`, type factory usage, and bit-width parameterization.

---

## Relationship to Other Languages
- **From Ada**: explicit representation control, attribute syntax, strong typing, English keywords.
- **From Rust**: `Result`-based error handling (but without type erasure), `if let`, `match`, block expressions.
- **Unique to Posita**: bit-width parameterized integers, orthogonal pointer sizes, type-level defaults, `leave`/`leave with`, type capture `auto[<T>..]`, fully static error monomorphization, compile-time type factories and reflection.
