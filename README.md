# Compiling Itself: A C Language Subset Compiler

> Which came first — the compiler or the source code? This project might just answer that.

A **self-hosting C subset compiler** written in C, inspired by [c4](https://github.com/rswier/c4). Small enough to compile itself. Complete enough to teach you everything about how a compiler turns characters into execution.

---

## Features

- Supports `int`, `char`, and pointer types
- Function definitions, calls, and recursion
- Control flow: `if` / `else`, `while`, `return`, `enum`
- Full unary and binary operators (bitwise, logical, comparison, arithmetic)
- Array access and type casting
- Built-in virtual machine that directly interprets bytecode
- **Can compile itself (bootstrapping)**

---

## Architecture

```
Source code (.c file)
        │
        ▼
┌───────────────────┐
│   Lexer           │  next()
│                   │  Tokenizes the character stream
│  Handles:         │
│  identifiers      │
│  numbers          │
│  strings          │
│  keywords         │
│  operators        │
└─────────┬─────────┘
          │
          ▼
┌──────────────────────────────────┐
│   Parser                         │  Recursive descent
│                                  │
│  program()                       │  Entry — loops over global declarations
│  ├─ global_declaration()         │  Variables / functions / enums
│  │   ├─ enum_declaration()       │  Enum constants
│  │   └─ function_declaration()   │
│  │       ├─ function_parameter() │  Parameter list
│  │       └─ function_body()      │  Function body
│  │           └─ statement()      │  Statements
│  │               └─ expression() │  Expressions (precedence climbing)
│  └─ ...                          │
│                                  │
│  Emits bytecode into text segment as it parses
└─────────┬────────────────────────┘
          │
          ▼
┌───────────────────┐
│   Virtual Machine │  eval()
│                   │  Interprets bytecode from the text segment
│  Registers:       │
│  pc / bp / sp / ax│
│                   │
│  Memory segments: │
│  text / data / stack
└───────────────────┘
```

---

## Instruction Set

### 1. Load & Store
| Instruction | Description |
|-------------|-------------|
| `IMM`  | Load immediate value into `ax` |
| `LEA`  | Load local variable address into `ax` |
| `LC`   | Load `char` from address in `ax` |
| `LI`   | Load `int` from address in `ax` |
| `SC`   | Save `ax` as `char` to address on stack top |
| `SI`   | Save `ax` as `int` to address on stack top |
| `PUSH` | Push `ax` onto the stack |

### 2. Arithmetic & Bitwise
| Instruction | Description |
|-------------|-------------|
| `ADD` / `SUB` / `MUL` / `DIV` / `MOD` | Arithmetic |
| `OR` / `XOR` / `AND` | Bitwise operations |
| `SHL` / `SHR` | Bit shifts |
| `EQ` / `NE` / `LT` / `LE` / `GT` / `GE` | Comparisons |

### 3. Control Flow
| Instruction | Description |
|-------------|-------------|
| `JMP`  | Unconditional jump |
| `JZ`   | Jump if `ax == 0` |
| `JNZ`  | Jump if `ax != 0` |
| `CALL` | Call function (saves return address) |
| `ENT`  | Enter function (create stack frame) |
| `ADJ`  | Adjust stack pointer (clean up arguments) |
| `LEV`  | Leave function (restore frame and return) |

### 4. Native Calls
| Instruction | Maps to |
|-------------|---------|
| `OPEN` | `open()` |
| `CLOS` | `close()` |
| `READ` | `read()` |
| `PRTF` | `printf()` |
| `MALC` | `malloc()` |
| `FREE` | `free()` |
| `MSET` | `memset()` |
| `MCMP` | `memcmp()` |
| `EXIT` | `exit()` |

---

## Symbol Table

During lexing, every identifier is stored in a flat integer array acting as a symbol table:

```
Each identifier occupies IdSize slots:
[ Token | Hash | Name | Type | Class | Value | BType | BClass | BValue ]
```

`BType`, `BClass`, and `BValue` temporarily shadow global variable metadata when entering a function scope, and are restored on exit. This is how the compiler handles local variable scoping without heap allocation per scope.

---

## Build & Run

> ⚠️ This compiler runs in **32-bit mode** and requires the `-m32` flag. macOS users: Xcode 12+ dropped 32-bit support — use Linux or Docker instead.

```bash
# Compile the compiler (note the -m32 flag)
gcc -m32 cc.c -o cc

# Or simply use the Makefile
make compile

# Use it to compile a C source file
./cc hello.c

# Enable debug mode (prints each bytecode instruction as it executes)
./cc -d hello.c
```

### Running the tests

```bash
# Run the built-in test suite
make test
# Equivalent to: ./cc test.c
```

---

## Bootstrapping: How It Compiles Itself

The most interesting property of this compiler is that it can compile its own source code:

```bash
# One step — compile itself and run the tests
make bootstrap
# Equivalent to: ./cc cc.c test.c

# Or step by step:
# Step 1: use system gcc to produce the first binary
gcc -m32 cc.c -o cc

# Step 2: use cc to compile its own source
./cc cc.c
```

This is the engineering answer to the chicken-and-egg paradox: the very first compiler always has to be bootstrapped by an external tool. After that, it can sustain itself — compiling the next version of its own source, indefinitely.

It mirrors how production compilers like GCC and Clang work. The compiler you use today was compiled by yesterday's compiler.

---

## Known Issues

`test.c` includes three test cases marked as expected failures, all related to how **logical operators handle negative numbers**:

```c
test(1, c || c);    // failed — || gives wrong result when c is negative
test(TRUE, b && c); // failed — && mishandles negative operands
test(TRUE, c && c); // failed
```

The root cause: the VM's `JNZ` (jump if not zero) instruction does not treat negative integers as truthy, which is non-standard — C requires any non-zero value to be true.

---

## Supported Language Subset

```c
// Types
int x;
char c;
int *ptr;

// Control flow
if (x > 0) { ... } else { ... }
while (x > 0) { x = x - 1; }

// Functions
int add(int a, int b) {
    return a + b;
}

// Enums
enum { False, True };

// Pointers & arrays
int arr[10];
*(arr + 2) = 42;

// Sizeof
int size;
size = sizeof(int);
```



---

## References & Further Reading

**Source inspirations**
- [rswier / c4 — C in four functions](https://github.com/rswier/c4)
- [lotabout / write-a-C-interpreter](https://github.com/lotabout/write-a-C-interpreter)
- [archeryue / cpc](https://github.com/archeryue/cpc)
- [tch0 / JustAToyCCompiler](https://github.com/tch0/JustAToyCCompiler)

**Articles**
- [Which articles cover c4 - C in four functions? (Zhihu, Chinese)](https://www.zhihu.com/question/28249756/answer/84307453)

**Books**
- Niklaus Wirth — *Compiler Construction*
- [The Elements of Computing Systems — Build a Modern Computer from First Principles (nand2tetris)](https://github.com/woai3c/nand2tetris)