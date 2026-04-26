# Architecture

Luz has two execution paths: an **interpreter** (`luz.exe`) for development and a **native compiler** (`luzc`) that produces standalone executables via TCC (Tiny C Compiler).

## Interpreter pipeline

```
Source code (text)
      |
   [Lexer]           luz/lexer.py
      |
 Token stream
      |
   [Parser]          luz/parser.py
      |
  AST (tree)
      |
 [TypeChecker]       luz/typechecker.py
      |
  Error list
      |
 [Interpreter]       luz/interpreter.py
      |
   Result
```

## Compiler pipeline (2.0beta)

```
Source code (text)
      |
   [Lexer]           compiler/src/
      |
 Token stream
      |
   [Parser]          compiler/src/
      |
  AST (tree)
      |
  [HIR Lowering]     compiler/src/
      |
  HIR nodes
      |
 [C Code Generator]  compiler/src/ccodegen.cpp
      |
   C source code
      |
   [TCC]             bundled (~100 KB), no external install
      linked with compiler/runtime/luz_rt.c
      |
 Native executable (.exe)
```

TCC (Tiny C Compiler) is bundled with the installer. Users need no external compiler toolchain — no clang, no LLVM, no MSVC. The `LUZ_HOME` environment variable (set by the installer) tells `luzc` where to find TCC and the runtime.

> **Legacy:** The previous LLVM-based pipeline (`--emit-llvm`) is still accessible but deprecated. Use `--emit-c` for the current backend.

## Lexer (`luz/lexer.py`)

Converts raw source text into a flat list of `Token` objects.

- Handles integers, floats (including `.5` notation), strings, format strings, identifiers, keywords, and all operators.
- Emits a `QUESTION` token for `?` to support nullable type annotations (`T?`).
- Tracks line numbers on every token so errors can report the correct source line.
- Resolves string escape sequences (`\n`, `\t`, `\\`, `\"`) at this stage.
- Format strings store the raw template (e.g. `"Hello {name}"`) - expression parsing happens later in the parser.
- Optionally delegates to a C implementation (`luz/c_lexer/luz_lexer.dll` / `.so`) for faster tokenisation.

## Parser (`luz/parser.py`)

Consumes the token stream and builds an **Abstract Syntax Tree (AST)** using a **recursive descent parser**.

Operator precedence is enforced through a chain of parsing functions, each calling the next higher-precedence level:

```
logical_or -> logical_and -> logical_not -> comp_expr
           -> arith_expr -> term -> power -> factor
```

Each node type (`BinOpNode`, `IfNode`, `CallNode`, `ClassDefNode`, `StructDefNode`, etc.) is a plain Python class defined at the top of the file. Nodes hold references to their child nodes, forming a tree.

Format string parsing: the lexer stores raw template text; the parser splits it on `{...}` (tracking brace depth for nesting), then sub-parses each embedded expression using a fresh `Lexer` + `Parser`.

Type expressions (used in variable annotations and function signatures) are parsed by `parse_type_expr()`, which produces strings like `"int"`, `"string?"`, `"list[int]"`, or `"dict[string, float]"`.

## Type Checker (`luz/typechecker.py`)

Walks the AST before execution and collects type errors without running any code. The main entry point is `TypeChecker().check(ast)`, which returns a list of `TypeCheckError` objects.

Key responsibilities:

- **Typed variable declarations** - checks that the assigned value matches the declared type annotation.
- **Function call arity** - checks that the number of arguments (positional + keyword) falls within the allowed range, accounting for default parameters.
- **Function return type** - checks that `return` expressions match the declared `-> type` annotation.
- **Class attribute inference** - scans the `init` body to build a map of `self.<attr>` types, enabling attribute access checks on typed instances.
- **Unused variable / import / parameter detection** - reports identifiers that are declared but never read (Go-style).
- **Type inference** - propagates types through arithmetic (`int + float -> float`), function calls, and binary operations.

The `T` class defines type constants (`T.INT`, `T.STRING`, etc.) and a `T.compatible(declared, actual)` predicate that handles nullable types, generic collections, and fixed-size numeric widening.

## HIR Lowering (`luz/hir.py`)

The **High-level Intermediate Representation** sits between the AST and LLVM IR. It has two jobs:

1. Make every type explicit - every node carries a `type` annotation.
2. Desugar complex constructs into a minimal set of primitives that map directly to LLVM IR patterns.

Desugaring performed by `Lowering`:

| Source construct | HIR equivalent |
|---|---|
| `for i = s to e step k` | `HirWhile` with explicit counter |
| `for x in list` | index-based `HirWhile` |
| `switch/case` | `HirIf`/`elif`/`else` chain |
| `match expr { }` | chain of equality checks |
| `x ?? y` | `if x != null { x } else { y }` |
| `a if cond else b` | `if cond { a } else { b }` |
| `$"hello {x}"` | series of `+` concatenations via `to_str()` |
| `obj.method(args)` | top-level runtime function call |
| list comprehension | `HirWhile` building a list |

## C Code Generator (`compiler/src/ccodegen.cpp`)

`CCodeGen` lowers HIR nodes to C source code, which is then compiled by the bundled TCC.

**Value representation.** All Luz values are represented as a tagged union in C:

```c
luz_value_t  =  { i32 tag, i32 pad, i64 data }   (16 bytes)
```

The `tag` field identifies the runtime type (`0` = null, `1` = bool, `2` = int, `3` = float, `4` = string, `5` = list, `6` = dict, `7` = object). The `data` field holds an integer, a float (bitcast), or a pointer.

**Code generation targets:**

| HIR node | C output |
|---|---|
| `HirLiteral` | C literal or `luz_rt_str_literal` call |
| `HirLet` / `HirAssign` | local variable declaration / assignment |
| `HirLoad` | variable reference |
| `HirBinOp` | call to `luz_rt_<op>` dispatch helper |
| `HirUnaryOp` | call to `luz_rt_neg` / `luz_rt_not` |
| `HirIf` | C `if` / `else` blocks |
| `HirWhile` | C `while` loop |
| `HirReturn` | C `return` |
| `HirFuncDef` | C function |
| `HirCall` | C function call to runtime builtin or user function |
| `HirList` / `HirDict` | `luz_rt_make_list` / `luz_rt_make_dict` + appends |
| `HirFieldLoad/Store` | `luz_rt_getfield` / `luz_rt_setfield` |
| `HirIndex/IndexStore` | `luz_rt_getindex` / `luz_rt_setindex` |
| `HirClassDef` | one C function per method, named `ClassName__method` |

## Legacy: LLVM Code Generator

The previous LLVM-based backend (`--emit-llvm`) is still present but deprecated. It required `llvmlite` and an external linker (clang/gcc). The C backend is the default as of 2.0beta.

## Interpreter (`luz/interpreter.py`)

Walks the AST using the **Visitor pattern**: `visit(node)` dispatches to `visit_IfNode`, `visit_BinOpNode`, etc. based on the node's class name.

**Scope** is managed through a chain of `Environment` objects. Each block or function call creates a new environment linked to its parent, enabling proper variable scoping and closures. A single `_find_scope()` traversal is used for assignment to avoid the previous double-walk.

**OOP** is implemented through:

| Object | Role |
|---|---|
| `LuzClass` | Stores the method dictionary and a reference to the parent class |
| `LuzInstance` | Stores the instance's attribute dictionary and a reference to its class |
| `LuzSuperProxy` | Wraps an instance + a parent class; resolves `super.method()` calls |
| `LuzFunction` | A named function with a closure |
| `LuzLambda` | An anonymous function (short `fn(x) => expr` or long `fn(x) { body }`) |
| `LuzStruct` / `LuzStructInstance` | Value-type data container with typed fields |
| `LuzModule` | Wraps a module's exported namespace for `import "x" as alias` |

Method calls automatically inject `self` and `super` into the method's local scope.

**Control flow signals** (`return`, `break`, `continue`) are implemented as Python exceptions that propagate up the call stack and are caught at the appropriate node handler.

**Type enforcement** at runtime: typed variable assignments call `_check_type()`, which handles generic collections, nullable types, fixed-size numeric bounds (`_enforce_type()`), and class hierarchy walking. A `_TYPE_PARSE_CACHE` class-level dict avoids re-parsing generic type strings on repeated assignments.

## C Runtime (`compiler/runtime/luz_rt.c`)

The native runtime provides the heap-allocated data structures used by compiled programs. It is a single C file linked by TCC during compilation.

| File | Contents |
|---|---|
| `luz_rt.c` | `luz_value_t`, strings (with SSO), lists, dicts, objects, ARC, exceptions, I/O builtins |

Memory is managed with **ARC (Automatic Reference Counting)**. Every heap object starts with `refcount = 1`; `luz_*_retain` / `luz_*_release` increment / decrement it.

Strings use a **Small String Optimisation (SSO)**: strings up to 23 bytes are stored inline in the struct with no heap allocation.

## Error system (`luz/exceptions.py`)

All errors inherit from `LuzError`:

```
LuzError
├── SyntaxFault           (lexer / parser errors)
├── SemanticFault         (type errors, undefined variables, wrong argument count...)
│   ├── TypeViolationFault
│   ├── AttributeNotFoundFault
│   └── ArityFault
├── RuntimeFault          (division by zero, index out of bounds...)
├── CastFault             (failed type conversion)
└── UserFault             (raised by the alert keyword)
```

Every error carries a `line` attribute that is attached when the error propagates through `visit()`.

## File overview

```
luz-lang/
├── main.py               # Interpreter entry point: REPL, file execution, --check mode
├── luz/
│   ├── tokens.py         # TokenType enum and Token class
│   ├── lexer.py          # Lexer: text -> tokens
│   ├── parser.py         # Parser: tokens -> AST + all AST node classes
│   ├── typechecker.py    # Static type checker: AST -> list[TypeCheckError]
│   ├── hir.py            # HIR nodes + Lowering pass (AST -> HIR)
│   ├── codegen.py        # Legacy LLVM IR code generator (deprecated)
│   ├── interpreter.py    # Interpreter: executes the AST
│   ├── exceptions.py     # Full error class hierarchy
│   └── c_lexer/          # Optional C lexer (faster tokenisation for interpreter)
│       ├── luz_lexer.c
│       ├── bridge.py
│       └── Makefile
├── compiler/             # Native compiler (luzc) — C++ source
│   ├── src/
│   │   ├── main.cpp      # luzc entry point, CLI flag handling
│   │   └── ccodegen.cpp  # C code generator (HIR -> C source)
│   ├── include/luz/
│   │   └── ccodegen.hpp
│   ├── runtime/
│   │   └── luz_rt.c      # C runtime linked into every compiled program
│   ├── tools/            # Bundled TCC and support tools
│   └── CMakeLists.txt
├── libs/                 # Standard library (luz-math, luz-random, luz-io, ...)
├── tests/
│   ├── test_suite.py     # pytest test suite (interpreter)
│   └── fuzzer.py         # Random-input fuzzer
├── installer/            # Windows Inno Setup installer script
└── examples/             # Example programs
```
