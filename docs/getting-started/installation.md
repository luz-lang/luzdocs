# Installation

## Windows (standalone installer)

Download the installer from the [download page](https://elabsurdo984.github.io/luz-lang/download/). It bundles `luz.exe`, `luzc.exe`, `ray.exe`, the standard libraries, TCC (the bundled C compiler used by `luzc`), and optionally example programs. No Python required, no clang, no LLVM needed.

After installation, `luz`, `luzc`, and `ray` are available from any terminal:

```bash
luz program.luz        # run a file (interpreter)
luz                    # open the REPL
luzc program.luz       # compile to a native executable
luzc program.luz --run # compile and run immediately
```

The installer also sets the `LUZ_HOME` environment variable, which `luzc` uses to locate the bundled TCC compiler and the Luz C runtime automatically.

## From source

Requires **Python 3.8 or higher**. No external dependencies.

```bash
git clone https://github.com/Elabsurdo984/luz-lang.git
cd luz-lang
python main.py          # open the REPL
python main.py file.luz # run a file
```

No build step, no package manager, no virtual environment needed.

## VS Code extension

A full-featured language extension for VS Code is included in the `vscode-luz/` folder:

1. Copy `vscode-luz/` to `~/.vscode/extensions/`
2. Restart VS Code

Features: syntax highlighting, autocompletion, inline error detection, hover documentation, and snippets.

## Optional: build the C lexer

For faster tokenisation in the interpreter, compile the C lexer (requires MSYS2 on Windows):

```bash
cd luz/c_lexer
make
```

The interpreter automatically uses `luz_lexer.dll` (Windows) or `luz_lexer.so` (Linux/macOS) if it is present. This is not needed for `luzc`, which has its own built-in lexer.
