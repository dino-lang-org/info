# Dino Language Specification

**Version:** 0.1 (Draft)
**Status:** Experimental

---

## Table of Contents

1. [Overview](#1-overview)
2. [Design Philosophy](#2-design-philosophy)
3. [Lexical Structure](#3-lexical-structure)
4. [Type System](#4-type-system)
5. [Access Modifiers](#5-access-modifiers)
6. [Compiler Directives](#6-compiler-directives)
7. [Variables and Declarations](#7-variables-and-declarations)
8. [Pointers and Memory Model](#8-pointers-and-memory-model)
9. [Structures](#9-structures)
10. [Functions](#10-functions)
11. [Control Flow](#11-control-flow)
12. [Expressions and `yield`](#12-expressions-and-yield)
13. [Templates](#13-templates)
14. [Name Mangling and FFI](#14-name-mangling-and-ffi)
15. [Attributes](#15-attributes)
16. [Planned Features](#16-planned-features)

---

## 1. Overview

Dino is an experimental, statically compiled, general-purpose programming language. It is primarily designed as a research and exploration vehicle, with its compiler implementation being largely AI-assisted.

Key characteristics:

- **Compilation model:** Static ahead-of-time (AOT) compilation. A JIT backend is planned to support scripting-like workflows.
- **Memory model:** Manual memory management. No garbage collector.
- **Paradigm:** Primarily functional, with object-oriented features (RAII via constructors/destructors on structs).
- **Syntax:** Inspired by C/C++, Java, Rust, and C#.
- **Semantics:** Inspired by C/C++ and Rust.
- **Toolchain:** Dino is distributed as a unified suite — compiler, standard library, and package manager — following Rust's model.

---

## 2. Design Philosophy

Dino attempts to combine what its author considers the best qualities of existing systems languages:

- The expressiveness and familiarity of C/C++ syntax
- The safety primitives of Rust (e.g., `nonull`, RAII)
- The readability touches of Java and C#
- A clean, minimal template system without the complexity of C++ NTTP
- A first-class, human-readable name mangling scheme to aid debugging and FFI

> **Note:** Because the compiler is significantly AI-generated, the language serves a secondary role as a proof-of-concept for AI-assisted language implementation.

---

## 3. Lexical Structure

### 3.1 Comments

```
// Single-line comment

/* Multi-line
   comment */
```

### 3.2 Identifiers

Identifiers follow the standard C-style rules: `[a-zA-Z_][a-zA-Z0-9_]*`.

### 3.3 Keywords

The following are reserved keywords in Dino:

```
auto        bool        break       case        char
const       default     else        enum        extern
fallthrough false       float32     float64     for
if          import      in          int8        int16
int32       int64       match       nonull      null
panic       private     public      return      struct
template    this        true        typename    uint8
uint16      uint32      uint64      while       yield
```

### 3.4 Literals

| Kind      | Examples                        |
|-----------|---------------------------------|
| Integer   | `0`, `42`, `0xFF`, `0b1010`     |
| Float     | `3.14`, `1.0e-5`                |
| Bool      | `true`, `false`                 |
| Char      | `'a'`, `'\n'`                   |
| String    | `"hello"`, `"line\n"`           |
| Null      | `null`                          |

---

## 4. Type System

Dino is **statically typed**. All types must be known at compile time.

### 4.1 Primitive Types

| Type      | Description                          |
|-----------|--------------------------------------|
| `int8`    | Signed 8-bit integer                 |
| `uint8`   | Unsigned 8-bit integer               |
| `int16`   | Signed 16-bit integer                |
| `uint16`  | Unsigned 16-bit integer              |
| `int32`   | Signed 32-bit integer                |
| `uint32`  | Unsigned 32-bit integer              |
| `int64`   | Signed 64-bit integer                |
| `uint64`  | Unsigned 64-bit integer              |
| `float32` | IEEE 754 single-precision float      |
| `float64` | IEEE 754 double-precision float      |
| `bool`    | Boolean (`true` / `false`)           |
| `char`    | Unicode scalar value, UTF-8 encoded (1–4 bytes) |

### 4.2 Pointer Types

See [§8 Pointers and Memory Model](#8-pointers-and-memory-model).

### 4.3 Struct Types

User-defined composite types. See [§9 Structures](#9-structures).

---

## 5. Access Modifiers

Dino has two access modifiers: `public` and `private`. If no modifier is specified, **the declaration is implicitly `private`**.

### 5.1 At File Scope

| Modifier  | Effect                                                       |
|-----------|--------------------------------------------------------------|
| `public`  | The declaration is visible to any file that imports this one |
| `private` | The declaration is visible only within the current file      |

### 5.2 At Struct Member Scope

| Modifier  | Effect                                                              |
|-----------|---------------------------------------------------------------------|
| `public`  | The field or method is accessible to any code using the struct      |
| `private` | The field or method is accessible only within the struct's own methods |

### 5.3 Examples

```dino
// Visible only inside this file
private int32 helperCount = 0;

// Visible to all importers
public int32 getCount() { return helperCount; }

public struct Wallet {
    private float64 balance;   // not accessible from outside
    public int32 ownerId;      // accessible from outside

    public float64 getBalance() { return this->balance; }
}
```

---

## 6. Compiler Directives

All compiler directives use the `@` prefix and are resolved entirely at compile time. They are not function calls — they have no runtime overhead.

### 6.1 `@import` — File Inclusion

`@import` is Dino's mechanism for including declarations from another file. It is **semantically** analogous to C's `#include` but is **not** a textual substitution — the compiler processes the target file and resolves its public declarations.

```
@import("path/to/file.dino");
```

File lookup is performed:
1. Relative to the directory of the current file.
2. Against any additional include paths supplied to the compiler.

### 6.2 Visibility of Imported Declarations

By default, imported symbols are available only within the importing file. To re-export them to downstream importers, prefix the directive with `public`:

```dino
public @import("file.dino");
```

### 6.3 Transitive Imports

When a file is imported with `public @import`, all of its own public declarations become available to further importers of the intermediate file.

```dino
// b.dino
public void doSomething() { /* ... */ }

// a.dino
public @import("b.dino");   // re-exports doSomething

// c.dino
@import("a.dino");

public int32 main() {
    doSomething();  // valid — transitively available from b.dino via a.dino
}
```

### 6.4 `@typeof` — Type of an Expression

`@typeof(expr)` evaluates to the **static type** of `expr` at compile time. The result can be used anywhere a type is expected.

```dino
int32 x = 10;
@typeof(x) y = 20;   // y is int32

// Composing with other directives:
@sizeof(@typeof('a'))   // -> size of char in bytes
```

`expr` is **not evaluated at runtime** — only its type is inspected. Side effects inside `@typeof` do not occur.

| Usage context        | Example                              |
|----------------------|--------------------------------------|
| Variable declaration | `@typeof(x) y = 0;`                  |
| Function parameter   | `void f(@typeof(x) val) {}`          |
| As argument          | `@sizeof(@typeof(someVar))`          |

### 6.5 `@sizeof` — Size of a Type

`@sizeof(type)` returns the size of `type` in bytes as a compile-time `uint64` constant.

```dino
@sizeof(int32)            // -> 4
@sizeof(float64)          // -> 8
@sizeof(Point)            // -> size of struct Point
@sizeof(@typeof('a'))     // -> size of char
```

`@sizeof` takes a **type**, not an expression. To get the size of a variable's type, compose with `@typeof`:

```dino
auto x = 3.14;
@sizeof(@typeof(x))       // -> 8  (float64)
```

### 6.6 `@decay` — Strip Type Modifiers

`@decay(type)` removes all qualifiers and indirection from a type, returning the bare underlying type.

```dino
@decay(const Point*)      // -> Point
@decay(const int32*)      // -> int32
@decay(int32*)            // -> int32
```

This is useful in templates when you need to work with the raw type regardless of how it was passed:

```dino
template <typename T>
public void process(T* ptr) {
    @decay(T) local = *ptr;   // copies the value into a local
}
```

| Input type       | `@decay` result |
|------------------|-----------------|
| `const T*`       | `T`             |
| `T*`             | `T`             |
| `T`              | `T`             |

> **Note:** `@decay` operates only on the outermost layer of indirection. `@decay(int32**)` yields `int32*`, not `int32`.

---

## 7. Variables and Declarations

### 7.1 Explicit Type Declarations

Variable declarations require an explicit type:

```dino
int32 x = 10;
float64 pi = 3.14159;
bool ready = false;
```

### 7.2 Type Inference with `auto`

When the type can be unambiguously inferred from the initializer, `auto` may be used instead of an explicit type. `auto` is **only** valid for type inference — it has no other purpose in the language (unlike C++11's later uses of `auto`).

```dino
auto x = 10;        // int32
auto pi = 3.14159;  // float64
auto p = &someStruct;  // inferred pointer type
```

`auto` is fully supported in `for-in` loop variables:

```dino
for (auto item in collection) { /* ... */ }
```

---

## 8. Pointers and Memory Model

Dino has **no reference types**. Data is either:
- **Copied** when passed by value, or
- **Not copied** when passed via a pointer.

### 8.1 Pointer Syntax

Pointer syntax follows C conventions:

```dino
int32* p = &x;
*p = 42;
```

### 8.2 `const` Pointers

The `const` qualifier prevents mutation of the pointed-to data:

```dino
const int32* p = &x;  // cannot write through p
```

### 8.3 `nonull` Qualifier

`nonull` is a **compile-time/runtime safety annotation** that asserts a pointer can never be `null`.

- At the call site, if a `nonull`-annotated pointer is `null` at runtime, the program calls `panic` and terminates.
- This is idiomatic Dino for "this should never be null; if it is, something is fundamentally wrong."

```dino
public Point mergePoints(nonull const Point* other) {
    // `other` is guaranteed non-null here; a null check was injected automatically
    return Point(this->x + other->x, this->y + other->y);
}
```

### 8.4 `panic`

`panic` is the language's mechanism for unrecoverable errors. It terminates the program with a diagnostic message. It is not an exception — there is no catching it.

### 8.5 Memory Management

Memory is managed manually. Structs support RAII through constructors and destructors (see [§9](#9-structures)). There is no garbage collector.

---

## 9. Structures

Structs are Dino's primary user-defined type. They support:
- Fields
- Constructors
- Destructors (RAII)
- Methods

### 9.1 Declaration Syntax

```dino
public struct Point {
    public float32 x, y;

    // Constructor
    public Point(float32 x, float32 y) {
        this->x = x;
        this->y = y;
    }

    // Destructor (optional; called when the struct goes out of scope)
    // ~Point() { /* cleanup */ }

    // Method
    public Point mergePoints(nonull const Point* other) {
        return Point(this->x + other->x, this->y + other->y);
    }
}
```

### 9.2 `this` Pointer

Inside methods and constructors, `this` is an implicit pointer to the current struct instance. Members are accessed via `this->field`.

### 9.3 Constructors

A constructor has the same name as the struct and no return type. Multiple constructors may be defined (overloading is supported, disambiguated by parameter types).

### 9.4 Destructors

A destructor is named `~StructName()` and takes no parameters. It is called automatically when a struct value goes out of scope (RAII). This enables deterministic resource cleanup (file handles, heap memory, etc.).

### 9.5 Iterators

A struct can be used in a `for-in` loop if it implements two methods:

```dino
T* begin();
T* end();
```

Where `T` is the element type. The loop iterates from `begin()` (inclusive) to `end()` (exclusive) by pointer increment.

---

## 10. Functions

### 10.1 Declaration Syntax

```dino
public int32 add(int32 a, int32 b) {
    return a + b;
}
```

A function body is **always required** unless the function is declared `extern` (see [§15](#15-attributes)).

### 10.2 Overloading

Functions may be overloaded by parameter types. Disambiguation occurs at compile time.

### 10.3 Variadic Functions (C-style)

C-style variadic functions are supported for FFI compatibility, using `...`:

```dino
#[extern] public int32 printf(const char* format, ...);
```

Their use in new Dino code is **discouraged**. Prefer variadic templates (see [§13](#13-templates)).

---

## 11. Control Flow

### 11.1 `if` / `else`

```dino
if (condition) {
    // ...
} else if (otherCondition) {
    // ...
} else {
    // ...
}
```

`if` can also be used as an expression. See [§12](#12-expressions-and-yield).

### 11.2 `while`

```dino
while (condition) {
    // ...
}
```

### 11.3 `for`

**C-style `for`:**

```dino
for (int32 i = 0; i < 10; i++) {
    // ...
}
```

**Range `for-in`:**

The loop variable must have an explicit type or use `auto`:

```dino
for (int32 number in arrayOfNums) {
    // ...
}

for (auto number in arrayOfNums) {   // type inferred from collection element type
    // ...
}
```

Requires `collection` to expose `begin()` and `end()` methods returning `T*`. See [§9.5](#95-iterators).

### 11.4 `match`

`match` compares a value against a list of patterns. No implicit fall-through — each `case` block terminates independently. To explicitly fall through to the next case, use `fallthrough` at the end of the block.

```dino
match (a - b) {
    case 1..=5:  yield "Between 1 and 5 (inclusive)";
    case 6..<10: yield "Between 6 and 10 (exclusive)";
    default:     yield "Out of range";
}
```

**Range syntax:**

| Syntax    | Meaning                   |
|-----------|---------------------------|
| `a..=b`   | Inclusive range `[a, b]`  |
| `a..<b`   | Exclusive range `[a, b)`  |

Ranges are permitted for any numeric type that supports comparison, including `float32` and `float64`:

```dino
match (ratio) {
    case 0.0..<0.5:  yield "low";
    case 0.5..=1.0:  yield "high";
    default:         yield "out of bounds";
}
```

> **Note:** Floating-point comparisons in `match` use the same semantics as `<` and `<=` in expressions. NaN does not match any range.

**Explicit fall-through:**

```dino
match (x) {
    case 1:
        doFirst();
        fallthrough;   // continues into case 2
    case 2:
        doSecond();
    default:
        doDefault();
}
```

---

## 12. Expressions and `yield`

`if` and `match` can function as **expressions**, producing a value. In this context, `yield` specifies the value produced by a branch.

```dino
int32 x = if (a > b) yield a else yield b;

int32 y = match (flag) {
    case true:  yield 1;
    case false: yield 0;
};

print(if (a > b) yield a else yield b);
```

### Why `yield` and not implicit last-expression?

Using `return` inside an expression-`if` or expression-`match` causes a **return from the enclosing function**, not from the expression. `yield` is explicit and unambiguous — it only yields a value from the current expression block, not from the function.

This is a deliberate divergence from Rust's implicit last-expression model, chosen for clarity.

---

## 13. Templates

Dino's template system is inspired by C++ templates but intentionally restricted. The goal is generic code reuse, not metaprogramming.

### 13.1 Syntax

```dino
template <typename T>
public T max(T a, T b) {
    return if (a > b) yield a else yield b;
}

template <typename... Ts>
public void doSomething(Ts... values) {
    print(values...);
}
```

### 13.2 Instantiation

Templates are instantiated at compile time for each unique combination of type arguments. This uses **duck typing** — no explicit constraints are required. If the instantiated code is invalid for the given types, a compile error is emitted.

**Type deduction** is the default: the compiler infers type arguments from the call's argument types in trivial cases.

**Explicit type arguments** are supported when deduction is ambiguous or when the caller wants to be explicit:

```dino
max(a, b);             // type deduced from a and b
max<float64>(a, b);    // explicit
```

### 13.3 Limitations

- No non-type template parameters (NTTP).
- No template specialization.
- Concepts / type constraints are **planned** but not yet implemented (see [§16](#16-planned-features)).

---

## 14. Name Mangling and FFI

Dino uses **human-readable name mangling**. Symbols in the object file are structured, readable strings — unlike C++ which uses encoded, opaque symbols.

### 14.1 Mangling Scheme

| Construct         | Symbol format                        | Example                              |
|-------------------|--------------------------------------|--------------------------------------|
| Free function     | `function_name(param_types)`         | `add(int32,int32)`                   |
| Struct constructor| `ctor StructName(param_types)`       | `ctor Point(float32,float32)`        |
| Struct destructor | `dtor StructName`                    | `dtor Point`                         |
| Struct method     | `method StructName.methodName(param_types)` | `method Point.mergePoints(const Point*)` |

Parameter names are omitted; only types are encoded. `const` and pointer qualifiers are preserved.

### 14.2 C Mangling (`#[no_mangle]`)

To emit a plain C-compatible symbol name (no prefix, no type encoding), use the `#[no_mangle]` attribute:

```dino
#[no_mangle] public void doSomething(int32 x, int32 y) { }
// Symbol in object file: "doSomething"
```

### 14.3 Extern Declarations (`#[extern]`)

To declare an external symbol to be resolved at link time:

```dino
#[extern] public int32 printf(const char* format, ...);
```

`#[extern]` is the **only** context where a function body may be omitted (replaced by `;`). In all other cases, a missing body is a compile error.

---

## 15. Attributes

Attributes are compiler directives annotating declarations. They are written with `#[...]` syntax.

| Attribute      | Applies to  | Effect                                                     |
|----------------|-------------|------------------------------------------------------------|
| `#[no_mangle]` | Functions   | Emits the bare function name as the symbol, no mangling    |
| `#[extern]`    | Functions   | Declares the function as externally resolved at link time  |

Additional attributes may be added in future versions.

---

## 16. Planned Features

The following features are on the roadmap but not yet implemented:

### 16.1 Enums and Tagged Unions

Enums are planned, inspired by Rust's enum model (tagged unions). This will allow expressing sum types:

```dino
// Illustrative syntax (not final)
enum Option<T> {
    Some(T),
    None,
}
```

### 16.2 Concepts / Type Constraints

Templates currently rely on duck typing with no constraints. A mechanism similar to C++20 Concepts is planned to provide meaningful compile errors and express type requirements explicitly.

### 16.3 JIT Backend

A JIT compilation mode is planned to allow Dino to be used in scripting contexts, improving interactivity and enabling REPL-style workflows.

### 16.4 Package Manager

A built-in package manager (similar to Cargo for Rust) will be distributed alongside the compiler and standard library, forming a unified Dino toolchain.

### 16.5 Standard Library

A standard library is planned, covering at minimum: collections, I/O, string handling, and memory utilities.

---

## Appendix A: Grammar Summary (Partial)

```
program         ::= (import_decl | decl)*

import_decl     ::= 'public'? '@import' '(' string_literal ')' ';'

decl            ::= access_modifier? (struct_decl | func_decl | var_decl)

access_modifier ::= 'public' | 'private'

struct_decl     ::= 'struct' IDENT '{' struct_member* '}'
struct_member   ::= access_modifier? (field_decl | ctor_decl | dtor_decl | method_decl)

func_decl       ::= attribute* type IDENT template_args? '(' param_list ')' (block | ';')
template_args   ::= '<' type (',' type)* '>'

param_list      ::= (param (',' param)*)? (',' '...')?
param           ::= 'nonull'? 'const'? type ('*')? IDENT

-- Compiler directives (compile-time only, no runtime cost)
directive_type  ::= '@typeof' '(' expr ')'
                  | '@decay'  '(' type ')'
sizeof_expr     ::= '@sizeof' '(' type ')'

type            ::= 'auto' | primitive_type | IDENT | directive_type
primitive_type  ::= 'int8' | 'uint8' | 'int16' | 'uint16' | 'int32' | 'uint32'
                  | 'int64' | 'uint64' | 'float32' | 'float64' | 'bool' | 'char'

var_decl        ::= type IDENT ('=' expr)? ';'   -- type may be 'auto'; initializer required with auto

stmt            ::= var_decl | expr_stmt | if_stmt | for_stmt | while_stmt
                  | match_stmt | return_stmt | block | 'panic' ';'

if_stmt         ::= 'if' '(' expr ')' block ('else' 'if' '(' expr ')' block)* ('else' block)?
while_stmt      ::= 'while' '(' expr ')' block
for_stmt        ::= 'for' '(' (var_decl | ';') expr? ';' expr? ')' block
                  | 'for' '(' type IDENT 'in' expr ')' block   -- type may be 'auto'
match_stmt      ::= 'match' '(' expr ')' '{' case_clause* default_clause? '}'
case_clause     ::= 'case' pattern ':' stmt* 'fallthrough'?
default_clause  ::= 'default' ':' stmt*
pattern         ::= expr (range_op expr)?         -- expr may be any numeric literal, incl. float
range_op        ::= '..=' | '..<'

yield_expr      ::= 'yield' expr
```

---

## Appendix B: Full Example

```dino
// point.dino

public struct Point {
    public float32 x, y;

    public Point(float32 x, float32 y) {
        this->x = x;
        this->y = y;
    }

    public ~Point() {
        // cleanup if needed
    }

    public Point mergeWith(nonull const Point* other) {
        return Point(this->x + other->x, this->y + other->y);
    }
}

// main.dino

@import("point.dino");

#[extern] public int32 printf(const char* format, ...);

public int32 main() {
    Point a = Point(1.0, 2.0);
    Point b = Point(3.0, 4.0);

    Point c = a.mergeWith(&b);

    int32 label = match (c.x) {
        case 0.0..<2.0:  yield 0;
        case 2.0..=10.0: yield 1;
        default:         yield -1;
    };

    printf("label: %d\n", label);
    return 0;
}
```

---

*This document is a living draft. As Dino evolves, sections will be expanded, revised, or added. Feedback and contributions are welcome.*
