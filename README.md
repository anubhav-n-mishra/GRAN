# GRAN

A small statically-typed language that JIT-compiles to native code through LLVM 19.
Source text in, executed program out — lexer, recursive-descent parser, AST, LLVM
IR generation, and MCJIT execution against a C runtime loaded at startup.

Written to understand what actually happens between a text file and running machine
code, without a parser generator doing the interesting part.

```
.gran source
     │
     ▼
  Lexer          src/lexer.cpp         characters → tokens
     │
     ▼
  Parser         src/parser.cpp        tokens → AST   (recursive descent, include/ast.h)
     │
     ▼
  IR Generator   src/ir_generator.cpp  AST → LLVM IR  (llvm::IRBuilder)
     │
     ▼
  MCJIT          src/main.cpp          IR → native code, executed in-process
     │
     └── runtime symbols (screenit, screenit_int, screenit_double) are dlopen'd
         from libruntime.so and registered with the JIT before the module runs
```

There is a second backend: `src/transpiler.cpp` lowers the same AST to C. It exists
because it was the fastest way to check the parser was building the tree correctly
before IR generation worked — if the generated C ran and the IR did not, the bug was
in lowering, not in parsing.

## The language

```gran
// Factorial, iteratively.
func factorial(n) {
    var result = 1;
    for (var i = 1; i <= n; i = i + 1) {
        result = result * i;
    }
    return result;
}

var num = 5;
screenit "Factorial of:";
screenit num;
screenit factorial(num);

if (factorial(num) > 100) {
    screenit "That's a big number!";
} else {
    screenit "That's manageable.";
}
```

`screenit` is the print statement. Full grammar in
[LANGUAGE_SYNTAX.md](LANGUAGE_SYNTAX.md).

**Supported:** `var` declarations, integer / float / string / boolean literals,
arithmetic (`+ - * /`), comparison (`== != < <= > >=`), assignment, blocks,
`if`/`else`, `while`, C-style `for`, `func` declarations, `return`, `break`, and
`//` comments.

**Not implemented, and worth saying plainly:** logical operators (`&&`, `||`, `!`),
arrays, structs, and a real type checker — a variable is an integer unless it is
assigned a string literal. Code generation for user-defined functions is partial:
they parse and lower, but not every call shape emits correct IR. The JIT runs at
`CodeGenOptLevel::None`, so nothing here is optimised.

## Building

Needs LLVM 19 development headers, a C++17 compiler, and `llvm-config` on `PATH`.

```bash
# Debian / Ubuntu
sudo apt-get install llvm-19-dev clang-19 g++ make
```

```bash
git clone https://github.com/anubhav-n-mishra/GRAN.git
cd GRAN
make
```

`make` produces two artifacts: `libruntime.so` (built from `runtime.c`) and `gran`
(the compiler driver). The Makefile links with `-Wl,-rpath,'$ORIGIN'` so the binary
finds the runtime beside it.

## Running

```bash
./gran factorial.gran
```

`gran` takes exactly one argument, the source file. It resolves `libruntime.so`
from the working directory, so run it from the repository root. Compilation
progress goes to stderr; program output goes to stdout.

## Layout

| Path                   | What it is                                        |
| ---------------------- | ------------------------------------------------- |
| `src/lexer.cpp`        | Tokeniser                                          |
| `src/parser.cpp`       | Recursive-descent parser producing the AST         |
| `src/ir_generator.cpp` | AST walk emitting LLVM IR via `IRBuilder`          |
| `src/transpiler.cpp`   | Alternate backend: lowers the same AST to C        |
| `include/ast.h`        | Node definitions for the whole tree                |
| `runtime.c`            | Runtime support registered with the JIT            |
| `web/`                 | Small browser playground over an Express backend   |

## Status

A learning compiler, not a production one. It is complete enough to run the programs
in `examples/` and `factorial.gran`, and honest about where it stops — see the
"not implemented" list above.

## Licence

MIT — see [LICENSE](LICENSE).
