# UB Prevention in Posita

Posita eliminates all language-level undefined behavior in safe code.
Every operation either succeeds with defined semantics or is rejected at compile time.

This document catalogs each category of undefined behavior common in systems
programming and explains how Posita prevents it.

---

## Uninitialized Variables

**The problem in C/C++**

Reading from uninitialized memory is undefined behavior. The value is
indeterminate, and the compiler may perform drastic optimizations assuming
the read never happens.

**Posita's solution**

Every type declares a default value (`with default = ...`). The compiler
automatically initializes any variable that is not explicitly given a value.
If a meaningful default does not exist, the type author uses `with no_default`,
which makes it a compile-time error to declare a variable without an initializer.

```posita
type Count = Int<32> with default = 0;
set c: Count;          // c == 0, guaranteed
```

```posita
type OwnedFd = Int<32>
    invariant n >= 0
    with no_default;   // must explicitly initialize

set fd: OwnedFd;       // compile error: OwnedFd has no default
```

---

## Integer Overflow (Signed)

**The problem in C/C++**

Signed integer overflow is undefined behavior. A simple addition can silently
corrupt program state or cause security vulnerabilities.

**Posita's solution**

Overflow is never undefined. The default policy is `trap` (compile-time error
in strict mode, runtime panic otherwise). Programmers can override the policy
per type or per operator.

```posita
type WrapCount = Int<32> with overflow = wrap;    // two's complement wrap
type SatCount  = Int<32> with overflow = saturate; // saturation
```

Operators can also carry suffixes: `+%` (wrap), `+?` (saturate), `+!` (trap).
The compiler uses range analysis and type invariants to eliminate checks
statically when possible.

---

## Division by Zero

**The problem**

Division or remainder with a zero divisor is undefined behavior in many
languages and a runtime crash in others.

**Posita's solution**

The compiler requires `requires b != 0` in contracts for division operations.
If the contract cannot be statically proven, a runtime check is inserted
(which panics, not UB).

```posita
def safe_div(a: Int<32>, b: Int<32>) -> Int<32>
    requires b != 0
{ a / b }
```

For signed division, `MIN / -1` always traps regardless of the overflow policy,
as this is a representability issue, not a standard overflow.

---

## Null Pointer Dereference

**The problem**

Dereferencing a null or invalid pointer is undefined behavior.

**Posita's solution**

References (`&T`, `&mut T`) are guaranteed non-null by the type system.
For nullable semantics, use `Option<&T>`, which forces explicit handling
of the `None` case before dereference.

```posita
def print_if_exists(maybe_ref: Option<&Data>) {
    match maybe_ref {
        Some(data) => { /* data is guaranteed valid */ }
        None => { /* handle absence */ }
    }
}
```

Raw pointers (`Ptr`) can only be dereferenced inside `unsafe` blocks,
confining the risk to `@trusted` functions with contracts.

---

## Dangling Pointers

**The problem**

A pointer to freed memory is a dangling pointer; using it is UB.

**Posita's solution**

The borrow checker ensures that references never outlive the data they point
to. Default copy semantics reduce accidental moves that could invalidate
pointers. Explicit `move` is available for ownership transfers, and the
compiler invalidates the source variable after a move.

```posita
set x = make_large_data();
set y = &x;          // borrows x
// x cannot be moved or dropped while y is live
```

---

## Data Races

**The problem**

Concurrent reads and writes to the same memory location without
synchronization cause data races, a notorious class of UB.

**Posita's solution**

`&mut T` is an exclusive, non-copyable borrow. While it is live, the original
variable is frozen—neither readable nor writable. `&T` permits shared reads.
The compile-time borrow rules prevent data races statically, without a runtime
cost.

---

## Buffer Overflow

**The problem**

Writing past the end of an array corrupts adjacent memory and is a leading
cause of security vulnerabilities.

**Posita's solution**

Array access (`arr[i]`) is checked against the array's bounds. The compiler
attempts to prove the index is in bounds via contracts or range analysis;
if it cannot, a runtime bounds check is inserted. Pointer arithmetic is
constrained to explicit `Ptr` types and is only permitted inside `unsafe`
blocks.

```posita
def safe_access(arr: &[Int<32>], i: usize) -> Int<32>
    requires i < arr'len
{ arr[i] }
```

---

## Invalid Enum Values

**The problem**

Setting an enum variable to a bit pattern not corresponding to any variant
is UB in many languages.

**Posita's solution**

Enum types have invariants that restrict their values to valid variants.
All construction paths are checked by the compiler. `as!` (bitcast) requires
compile-time proof that the cast does not violate the target type's invariant.

---

## Misaligned Pointers

**The problem**

Accessing a value through a pointer that is not aligned for its type is
undefined behavior on most architectures.

**Posita's solution**

The `Ptr` type carries alignment information. Safe casts check alignment
at compile time. `as!` demands proof of layout compatibility, including
alignment. The compiler emits a diagnostic if pointer arithmetic can be
statically proven to produce a misaligned pointer.

---

## Type Punning / Transmute

**The problem**

Reinterpreting one type's bit pattern as another type without care for layout
compatibility leads to UB.

**Posita's solution**

`as!` (bitcast) requires compile-time verification that the source and target
types have the same size and alignment (via `layout_of!`), or that a
truncation does not violate the target type's `invariant`. All uses of `as!`
are flagged for mandatory human review by `capsa audit`.

---

## Non-terminating Loops

**The problem**

An infinite loop without side effects is UB in C++ (allows compiler to
eliminate the loop or assume it does not happen).

**Posita's solution**

Loops may carry a `decreases` clause that proves strict decrease of a
non-negative measure. The compiler can use this to verify termination.
In strict mode, all loops must either have a `decreases` clause or be
provably finite through other analysis.

```posita
while i > 0
    decreases i
{ i -= 1; }
```

---

## Non-terminating Recursion

**The problem**

Recursive functions that do not terminate can cause stack overflow or
infinite execution.

**Posita's solution**

Recursive functions may declare a `terminates` clause specifying a
parameter that strictly decreases on each recursive call. The compiler
verifies the decrease, ensuring termination.

```posita
def factorial(n: Int<32>) -> Int<32>
    requires n >= 0
    terminates n
{
    if n == 0 { 1 } else { n * factorial(n - 1) }
}
```

---

## Infinite Loops at Compile Time

**The problem**

Compile-time evaluation (`comptime`) could hang the compiler indefinitely.

**Posita's solution**

`comptime` execution is bounded by step and memory limits. Exceeding the
limit produces a compile-time error, not a compiler hang.

---

## Static Initialization Order

**The problem**

The order of static initialization across translation units is unspecified in
C++, leading to subtle UB and crashes.

**Posita's solution**

Compile-time evaluation ensures constant initializers are fully resolved at
compile time. Module-level variables use zero-initialization or type defaults,
eliminating dynamic initialization ordering issues.

---

*Posita's strict mode forbids `unsafe` entirely; all contracts must be
statically proven. In strict mode, every UB category above is eliminated by
construction without any runtime checks.*
