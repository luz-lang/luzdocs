# Modules

## Importing a file

Use `import` to run another `.luz` file. Everything defined in that file (variables, functions, classes) becomes available in the global scope:

```
import "utils.luz"

result = my_function(42)
write(result)
```

## Path resolution

The path in `import` is relative to the file that contains the `import` statement.

```
import "helpers/math.luz"
import "lib/strings.luz"
```

## Execution model

When a file is imported:

1. The file is read and executed from top to bottom.
2. All definitions land in the **global** scope of the importing file.
3. The import happens at the point where the `import` statement appears.

## Circular imports

Luz automatically tracks which files have been imported. If a file tries to import a file that is already being executed (directly or transitively), the second import is silently skipped.

```
# a.luz
import "b.luz"

# b.luz
import "a.luz"   # skipped — a.luz is already being executed
```

## Selective import

Use `from "file" import name1, name2` to import only specific names into the current scope:

```
from "utils.luz" import format_date, parse_date

write(format_date(today))
```

If any of the requested names does not exist in the module, an `ImportFault` is raised and **nothing** is imported — it's all-or-nothing.

## Module alias

Use `import "file" as alias` to wrap the module's exports in a namespace object. This avoids polluting the global scope:

```
import "utils.luz" as utils

write(utils.format_date(today))
write(utils.parse_date("2026-01-01"))
```

Access any exported name with dot syntax: `alias.name`.

## Example

`greetings.luz`:
```
function hello(name) {
    return $"Hello, {name}!"
}

function goodbye(name) {
    return $"Goodbye, {name}!"
}
```

`main.luz`:
```
import "greetings.luz"

write(hello("Alice"))    # Hello, Alice!
write(goodbye("Alice"))  # Goodbye, Alice!
```
