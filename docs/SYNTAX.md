# Posita Language Syntax (v0.1)

## Design Philosophy
Posita is a **ultra-static, systems programming language** where the programmer explicitly *posits* every representation detail: bit widths, pointer sizes, default values, and even optimization policies. It inherits Ada's precision, borrows Rust's modern ergonomics, and introduces its own constructs where they serve clarity.

- **Explicit over implicit**: No hidden ABI, no integer promotion surprises.
- **Compile-time over run-time**: Everything that can be done at compile time should be.
- **Readable as documentation**: Keywords are English words (`def`, `set`, `leave`), not cryptic symbols.

---

## Lexical Structure

### Keywords
`def`, `set`, `type`, `with`, `default`, `return`, `if`, `else`, `for`, `in`, `while`, `loop`, `leave`,
`comptime`, `import`, `as`, `true`, `false`, `null`, `auto`, `and`, `or`, `not`, `sizeof`, `alignof`

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
- References: `&T` (immutable), `&mut T` (mutable). References are checked at compile time and do not support pointer arithmetic.

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

### Pattern Matching (preview)
```plaintext
match value {
    pattern1 => expression1,
    pattern2 => { statements },
    _ => default,
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

### Optimization Hooks (advanced)
Users can interact with the optimizer through a comptime API:
```plaintext
comptime {
    set plan = optimize(target = my_function, strategy = q_learn(config));
    plan.apply();
}
```
This feature requires compiler support and is opt-in.

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

def concat(left: &[Byte], right: &[Byte]) -> BufPtr {
    set total_len = left'len + right'len;
    set result     = alloc(BufPtr, total_len);
    set mut pos    = 0;

    for i in 0..left'len {
        result[pos] = left[i];
        pos = pos + 1;
    }

    for j in 0..right'len {
        if right[j] == 0xFF {
            leave;   // stop copying on terminator
        }
        result[pos] = right[j];
        pos = pos + 1;
    }

    return result;
}
```
This example shows bit-width parameterization, type defaults, `leave`, and attribute syntax working together.

---

## Relationship to Ada and Rust
- From Ada: explicit representation control, attribute syntax, strong typing, English keywords.
- From Rust: `fn`-like `def` (adapted), block expressions, `match`, module system.
- Unique: `set`, `leave`, `with default`, bit-width `Int<N>`, type capture `auto[<T>..]`.
