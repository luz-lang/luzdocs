# Usage

## Interactive REPL

Run `main.py` with no arguments to open the interactive shell:

```bash
python main.py
```

```
Luz Interpreter v1.18.0 - Type 'exit' to terminate
Luz > x = 10
Luz > write(x * 2)
20
Luz > name: string = "world"
Luz > write($"Hello {name}!")
Hello world!
Luz > exit
```

The REPL evaluates one statement at a time. Multi-line constructs (functions, classes, loops) are not supported in the REPL — use a file for those.

## Run a file

```bash
python main.py program.luz
```

Any `.luz` file can be executed this way. The interpreter runs the file from top to bottom and exits when it reaches the end.

## Parse-only check

The `--check` flag runs the lexer and parser only, returning any syntax errors as JSON. This is used by the VS Code extension:

```bash
python main.py --check program.luz
```

Output: `[]` on success, or `[{"line": N, "message": "..."}]` on error.

## Run the test suite

```bash
python -m pytest tests/test_suite.py -v
```

Run a specific test class or case:

```bash
python -m pytest tests/test_suite.py::TestArithmetic -v
python -m pytest tests/test_suite.py::TestArithmetic::test_int_addition -v
```

## Run all examples

```bash
python run_examples.py
```

Runs every `.luz` file in `examples/`, prints a pass/fail/interactive status for each, and exits with code 1 if any failures are found.

## Using the standalone installer (Windows)

Download and run the installer from the [download page](https://elabsurdo984.github.io/luz-lang/download/). After installation `luz`, `luzc`, and `ray` are available from any terminal:

```bash
luz program.luz    # run a file (interpreter)
luz                # open the REPL
luzc program.luz   # compile to a native executable
ray install user/repo   # install a package
```

---

## Native compiler (`luzc`)

`luzc` compiles Luz source files to native Windows executables using the bundled TCC backend. No clang or LLVM installation is required.

### Basic usage

```bash
luzc program.luz              # compile → program.exe
luzc program.luz -o myapp     # compile with custom output name
luzc program.luz --run        # compile and run immediately
```

### Inspect intermediate output

```bash
luzc program.luz --emit-c          # print generated C source to stdout
luzc program.luz --emit-c -o out.c # save generated C source to a file
luzc program.luz --emit-llvm       # emit LLVM IR (legacy, deprecated)
```

### Debugging and diagnostics

```bash
luzc program.luz --tokens     # print the token stream
luzc program.luz --ast        # print the AST
luzc program.luz --hir        # print the HIR (high-level IR)
luzc program.luz --check      # run type checking only, print errors
luzc program.luz --check-json # type checking errors as JSON
```

### Information flags

```bash
luzc --version
luzc --help
```

### Environment variables

| Variable | Purpose |
|---|---|
| `LUZ_HOME` | Set by the installer; points to the install directory where TCC and the runtime live |
| `LUZ_TCC` | Override the path to the `tcc` binary |
| `LUZ_RT` | Override the path to the `luz_rt.c` runtime file |

> **Note:** The interpreter (`luz.exe`) and the compiler (`luzc.exe`) share the same language syntax for most constructs, but `luzc` requires typed function signatures and uses `alert("message")` as a function call rather than a keyword. See the language reference for details.
