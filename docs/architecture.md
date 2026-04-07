# Architecture

Luz has two execution paths: an **interpreter** for development and a **compiler** that produces native executables via LLVM.

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

## Compiler pipeline

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
  [HIR Lowering]     luz/hir.py
      |
  HIR nodes
      |
 [LLVMCodeGen]       luz/codegen.py
      |
   LLVM IR
      |
  [llvmlite opt]     (mem2reg, inlining, ...)
      |
 Object file (.o)
      |
   [gcc/clang]       linked with luz/runtime/
      |
 Native executable
```

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

## LLVM Code Generator (`luz/codegen.py`)

`LLVMCodeGen` lowers HIR nodes to LLVM IR via **llvmlite**.

**Value representation.** All Luz values are represented as a single LLVM struct type:

```
luz_value_t  =  { i32 tag, i32 pad, i64 data }   (16 bytes)
```

The `tag` field identifies the runtime type (`0` = null, `1` = bool, `2` = int, `3` = float, `4` = string, `5` = list, `6` = dict, `7` = object). The `data` field holds an integer, a float (bitcast), or a pointer (via `ptrtoint`).

**Code generation targets:**

| HIR node | LLVM output |
|---|---|
| `HirLiteral` | constant struct or `luz_rt_str_literal` call |
| `HirLet` / `HirAssign` | `alloca` + `store` |
| `HirLoad` | `load` |
| `HirBinOp` | call to `luz_rt_<op>` dispatch helper |
| `HirUnaryOp` | call to `luz_rt_neg` / `luz_rt_not` |
| `HirIf` | `cbranch` + `if.then` / `if.else` / `if.merge` basic blocks |
| `HirWhile` | `while.cond` / `while.body` / `while.exit` basic blocks |
| `HirReturn` | `ret` |
| `HirFuncDef` | top-level LLVM function |
| `HirCall` | `call` to runtime builtin or user function |
| `HirList` / `HirDict` | `luz_rt_make_list` / `luz_rt_make_dict` + appends |
| `HirFieldLoad/Store` | `luz_rt_getfield` / `luz_rt_setfield` |
| `HirIndex/IndexStore` | `luz_rt_getindex` / `luz_rt_setindex` |
| `HirClassDef` | one LLVM function per method, named `ClassName__method` |

**Optimisation.** After IR generation, `compile_to_object()` runs the llvmlite pass manager at `opt=2` (equivalent to `gcc -O2`).

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

## C Runtime (`luz/runtime/`)

The native runtime provides the heap-allocated data structures used by compiled programs.

| File | Contents |
|---|---|
| `luz_runtime.h/.c` | `luz_value_t`, strings (with SSO), lists, dicts, objects, ARC, exceptions, I/O builtins |
| `luz_rt_ops.h/.c` | Dynamic dispatch helpers: arithmetic, comparison, logical, membership, collection access, slicing |

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
├── main.py               # Entry point: REPL, file execution, --check mode
├── luz/
│   ├── tokens.py         # TokenType enum and Token class
│   ├── lexer.py          # Lexer: text -> tokens
│   ├── parser.py         # Parser: tokens -> AST + all AST node classes
│   ├── typechecker.py    # Static type checker: AST -> list[TypeCheckError]
│   ├── hir.py            # HIR nodes + Lowering pass (AST -> HIR)
│   ├── codegen.py        # LLVM IR code generator (HIR -> native code)
│   ├── interpreter.py    # Interpreter: executes the AST
│   ├── exceptions.py     # Full error class hierarchy
│   ├── c_lexer/          # Optional C lexer (faster tokenisation)
│   │   ├── luz_lexer.c
│   │   ├── bridge.py
│   │   └── Makefile
│   └── runtime/          # Native C runtime for compiled programs
│       ├── luz_runtime.h/.c
│       ├── luz_rt_ops.h/.c
│       └── Makefile
├── libs/                 # Standard library (luz-math, luz-random, luz-io, ...)
├── tests/
│   ├── test_suite.py     # pytest test suite
│   └── fuzzer.py         # Random-input fuzzer
├── installer/            # Windows Inno Setup installer script
└── examples/             # Example programs
```
