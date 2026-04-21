# clox — A Bytecode Virtual Machine for the Lox Language

A complete implementation of the **clox** bytecode interpreter for the [Lox programming language](https://craftinginterpreters.com/the-lox-language.html), based on Robert Nystrom's [*Crafting Interpreters*](https://craftinginterpreters.com/). clox compiles Lox source code into bytecode and executes it on a stack-based virtual machine written in C.

---

## What Is Lox?

Lox is a dynamically typed, high-level scripting language. It supports:

- **Dynamic typing** — Variables can hold values of any type.
- **First-class functions** — Functions are values that can be passed around, returned, and stored.
- **Closures** — Functions capture variables from their enclosing scope.
- **Classes and inheritance** — Single-inheritance OOP with methods, initializers, and `this`/`super`.
- **Automatic memory management** — A mark-and-sweep garbage collector handles all allocation.

### Example

```lox
class Animal {
  init(name) {
    this.name = name;
  }
  speak() {
    print this.name + " makes a sound.";
  }
}

class Dog < Animal {
  speak() {
    print this.name + " barks!";
  }
}

var dog = Dog("Rex");
dog.speak(); // Rex barks!

fun fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}

print fib(20); // 6765
print clock();  // seconds since program start (native function)
```

---

## How It Works

The interpreter operates as a **three-stage pipeline**: scanning, compiling, and executing.

### 1. Scanner (`scanner.c` / `scanner.h`)

The scanner (lexer) takes raw Lox source code as a string and produces a flat stream of **tokens**. It is a single-pass, on-demand scanner — the compiler asks for the next token only when it needs one.

Recognized token types include:
- **Single-character tokens** — `(`, `)`, `{`, `}`, `,`, `.`, `-`, `+`, `;`, `/`, `*`
- **One- or two-character tokens** — `!`, `!=`, `=`, `==`, `>`, `>=`, `<`, `<=`
- **Literals** — number literals (`3.14`), string literals (`"hello"`)
- **Identifiers and keywords** — `and`, `class`, `else`, `false`, `for`, `fun`, `if`, `nil`, `or`, `print`, `return`, `super`, `this`, `true`, `var`, `while`

Whitespace and `//` line comments are silently skipped.

### 2. Compiler (`compiler.c` / `compiler.h`)

The compiler is a **single-pass Pratt parser** that reads the token stream and directly emits bytecode — there is no intermediate AST. It uses a table of parsing rules indexed by token type, where each rule specifies a prefix function, an infix function, and a precedence level.

Key responsibilities:
- **Expression compilation** — Handles unary, binary, grouping, literals, variables, assignments, logical operators, and function calls using precedence climbing.
- **Statement compilation** — `print`, `if`/`else`, `while`, `for`, `return`, expression statements, blocks, variable declarations, function declarations, and class declarations.
- **Local variables** — Tracks locals on a compile-time stack with lexical scoping. Locals are resolved by index at compile time and accessed by slot at runtime.
- **Closures and upvalues** — When a function references a variable from an enclosing scope, the compiler emits `OP_CLOSURE` instructions with upvalue metadata so the VM can capture those variables at runtime.
- **Classes and methods** — Compiles class bodies, method definitions, `this` bindings, `super` calls, and inheritance chains.

Each function body (including the top-level script) gets its own `ObjFunction` with its own `Chunk` of bytecode.

### 3. Virtual Machine (`vm.c` / `vm.h`)

The VM is a **stack-based bytecode interpreter**. It maintains:

- **A value stack** — Operands are pushed and popped as instructions execute.
- **A call frame stack** — Each function invocation creates a `CallFrame` storing the closure, instruction pointer, and stack window for that call. The maximum call depth is 64 frames.
- **A globals table** — A hash table mapping variable names to values.
- **A strings table** — All strings are interned for fast equality comparison.

The core execution loop (`run()`) fetches, decodes, and dispatches one bytecode instruction at a time. The full instruction set is:

| Opcode | Description |
|---|---|
| `OP_CONSTANT` | Push a constant value onto the stack |
| `OP_NIL`, `OP_TRUE`, `OP_FALSE` | Push literal `nil`, `true`, or `false` |
| `OP_POP` | Discard the top stack value |
| `OP_GET_LOCAL` / `OP_SET_LOCAL` | Read/write a local variable by stack slot |
| `OP_GET_GLOBAL` / `OP_SET_GLOBAL` / `OP_DEFINE_GLOBAL` | Read/write/define a global variable by name |
| `OP_GET_UPVALUE` / `OP_SET_UPVALUE` | Read/write a captured closure variable |
| `OP_GET_PROPERTY` / `OP_SET_PROPERTY` | Read/write an instance field |
| `OP_GET_SUPER` | Access a superclass method |
| `OP_EQUAL`, `OP_GREATER`, `OP_LESS` | Comparison operators |
| `OP_ADD`, `OP_SUBTRACT`, `OP_MULTIPLY`, `OP_DIVIDE` | Arithmetic (+ also concatenates strings) |
| `OP_NOT`, `OP_NEGATE` | Unary operators |
| `OP_PRINT` | Print the top stack value |
| `OP_JUMP` / `OP_JUMP_IF_FALSE` / `OP_LOOP` | Control flow |
| `OP_CALL` | Call a function or method |
| `OP_INVOKE` / `OP_SUPER_INVOKE` | Optimized method invocation |
| `OP_CLOSURE` | Create a closure, capturing upvalues |
| `OP_CLOSE_UPVALUE` | Move an upvalue from the stack to the heap |
| `OP_RETURN` | Return from the current function |
| `OP_CLASS` | Create a new class object |
| `OP_INHERIT` | Wire up superclass inheritance |
| `OP_METHOD` | Bind a method to a class |

**Native functions:** The VM defines one built-in native function, `clock()`, which returns the elapsed CPU time in seconds.

---

## Supporting Subsystems

### Chunks (`chunk.c` / `chunk.h`)

A `Chunk` is a dynamic array of bytecode (`uint8_t` codes), a parallel array of source line numbers (for error reporting), and a `ValueArray` of constants. Each compiled function owns one chunk.

### Values (`value.c` / `value.h`)

The `Value` type represents every runtime value in Lox. Two representations are supported, selected at compile time:

- **Tagged union** (default) — A struct containing a `ValueType` enum and a union of `bool`, `double`, or `Obj*`.
- **NaN boxing** (enabled by `#define NAN_BOXING`) — Packs all value types into a single 64-bit `uint64_t` by exploiting the unused bits in IEEE 754 NaN values. This improves cache performance and reduces memory usage.

### Objects (`object.c` / `object.h`)

Heap-allocated Lox values (strings, functions, classes, instances, closures, upvalues, bound methods, native functions) are represented as `Obj` structs linked into an intrusive list rooted at `vm.objects`. Each object has a type tag and an `isMarked` flag used by the garbage collector.

Object types:
- **`ObjString`** — An immutable string with a cached hash for O(1) hash table lookups. All strings are interned.
- **`ObjFunction`** — A compiled function: name, arity, upvalue count, and its bytecode `Chunk`.
- **`ObjClosure`** — Wraps an `ObjFunction` with an array of captured `ObjUpvalue` pointers.
- **`ObjUpvalue`** — Points to a captured variable. While the variable is still on the stack, it points there; once the variable goes out of scope, its value is "closed over" into the upvalue's own `closed` field.
- **`ObjNative`** — A C function pointer callable from Lox.
- **`ObjClass`** — A class with a name and a method table.
- **`ObjInstance`** — An instance with a reference to its class and a field table.
- **`ObjBoundMethod`** — Binds a receiver instance to a closure for method dispatch.

### Hash Table (`table.c` / `table.h`)

An open-addressing hash table with linear probing, used for global variables, string interning, instance fields, and class methods. Key features:
- Load factor capped at 75%; the table grows by doubling when exceeded.
- Uses **tombstones** (sentinel entries) for deletion so probe sequences remain valid.
- Capacity is always a power of two, allowing bitwise AND instead of modulo for index computation.

### Memory and Garbage Collection (`memory.c` / `memory.h`)

All dynamic memory flows through a single `reallocate()` function, which tracks total bytes allocated. The garbage collector is a **mark-and-sweep** collector:

1. **Mark** — Starting from roots (the value stack, call frames, open upvalues, global variables, the compiler's own data, and the `init` string), recursively mark all reachable objects using a gray worklist.
2. **Sweep** — Walk the full object list and free any unmarked objects. Interned strings with dead keys are also removed from the string table.

Collection is triggered when `bytesAllocated` exceeds `nextGC`, which starts at 1 MB and grows by a factor of 2 after each collection.

### Debug Disassembler (`debug.c` / `debug.h`)

Provides human-readable disassembly of bytecode chunks. Enable diagnostic output by uncommenting the defines in `common.h`:
- `DEBUG_PRINT_CODE` — Print the disassembled bytecode after compilation.
- `DEBUG_TRACE_EXECUTION` — Print each instruction and the stack contents as the VM executes.
- `DEBUG_STRESS_GC` — Run the garbage collector on every allocation (for testing).
- `DEBUG_LOG_GC` — Log allocation, marking, and sweep events.

---

## Building

### Prerequisites

- A **C compiler** (GCC, Clang, or MSVC)
- **Make** (optional, for using the provided Makefile)

### Using Make

```sh
make clox
```

This compiles an optimized release build and copies the binary to the project root.

For a debug build:

```sh
make debug
```

### Manual Compilation

You can compile all `.c` files in `src/` directly:

```sh
gcc -O2 -o clox src/*.c
```

Or on Windows with MSVC:

```sh
cl /O2 /Fe:clox.exe src/*.c
```

---

## Usage

### Interactive REPL

Run `clox` with no arguments to start an interactive prompt:

```sh
./clox
```

```
> print 1 + 2;
3
> var greeting = "hello";
> print greeting + " world";
hello world
>
```

Type expressions and statements one line at a time. Press Ctrl+C or Ctrl+D to exit.

### Running a Script File

Pass a `.lox` file path as an argument:

```sh
./clox script.lox
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 64 | Incorrect usage (wrong number of arguments) |
| 65 | Compile error (syntax or static analysis error) |
| 70 | Runtime error |
| 74 | I/O error (could not open or read the source file) |

---

## Testing

Test cases live in `test/`. A Dart-based test runner is provided:

```sh
make test_clox
```

Or run the test runner directly:

```sh
dart tool/bin/test.dart clox
```

---

## Source File Reference

| File | Purpose |
|------|---------|
| `main.c` | Entry point: REPL, file reader, and CLI argument handling |
| `scanner.c/h` | Lexer — converts source text into tokens |
| `compiler.c/h` | Single-pass Pratt parser and bytecode emitter |
| `vm.c/h` | Stack-based bytecode virtual machine |
| `chunk.c/h` | Bytecode storage (instruction arrays + constant pools) |
| `value.c/h` | Runtime value representation (tagged union or NaN-boxed) |
| `object.c/h` | Heap-allocated object types (strings, functions, classes, etc.) |
| `table.c/h` | Hash table (open addressing, linear probing, tombstones) |
| `memory.c/h` | Memory allocator and mark-and-sweep garbage collector |
| `debug.c/h` | Bytecode disassembler and execution tracing |
| `common.h` | Shared includes, feature flags, and constants |

---

## Acknowledgments

Based on the clox interpreter from [*Crafting Interpreters*](https://craftinginterpreters.com/) by Robert Nystrom. The source code is licensed under the MIT License.
