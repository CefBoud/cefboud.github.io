---
title: "A Practical Tour From Source to CPU"
author: cef
date: 2025-10-25
categories: [Technical Writing, Open Source]
tags: [Programming, C, Go, Python, Java]
render_with_liquid: false
description: "A guided tour through how code becomes something a computer can actually run. The post explores how languages like C, Python, Java, and Go build, package, and execute programs. How they link libraries, manage dependencies, and interact with native code."
---
## The Big Picture

The fundamental goal when writing software is to have our code translated into machine code (1s and 0s) that runs on the CPU. Before reaching machine code, we typically go through assembly.

Okay, let's back up. The CPU is the actual hardware, the "brain" of the computer. It has registers, a control unit, and handles data movement, among other tasks. Each CPU has its own instruction set (machine code): a collection of low-level instructions like "add this," "move that," and so on. Machine code is the lowest level; assembly is the more human-friendly representation.

```
MOV AX, 2    ; Move the value 2 into register AX
ADD AX, 1    ; Add 1 to the value in AX (AX now holds 3)
```

There are two major CPU architectures: ARM (developed by a UK company, commonly used in smaller devices but increasingly in computers mostly notably Apple's M chips) and x86 (developed by Intel). An x86_64 CPU can't run ARM64 code and vice versa.

The operating system (OS) is the software that manages the hardware, including the CPU, memory, file system, and I/O.

An OS can run on multiple architectures, and an architecture can be used by multiple OSes. For example: Linux on x86_64, macOS on ARM64, Windows on x86 (32-bit Intel). Pick your poison. Although Mac is now only on ARM64 (m series) and 32-bit x86 is now legacy. 

So when you're about to install a compiled program (like Git, Python, or SQLite), you're often faced with cryptic file names. You probably know your OS: "I'm on macOS, Windows, or Linux." Then you see something like *arm64* or *x64* and wonder, "What the heck should I pick?" Well, that refers to your machine's CPU architecture.
On Linux or macOS, you can run `uname -m` to check it. On my Mac with an M-series chip, it prints a short `arm64`.

When it comes to a program, the source code is the same, but the executable depends on the OS + architecture combination. Take SQLite, for example, using the `file` command:

```sh
➜ ~ file sqlite-tools-osx-arm64-3500400/sqlite3
sqlite-tools-osx-arm64-3500400/sqlite3: Mach-O 64-bit executable arm64

➜ ~ file sqlite-tools-osx-x64-3500400/sqlite3
sqlite-tools-osx-x64-3500400/sqlite3: Mach-O 64-bit executable x86_64

➜ ~ file sqlite-tools-win-arm64-3500400/sqlite3.exe
sqlite-tools-win-arm64-3500400/sqlite3.exe: PE32+ executable (console) Aarch64, for MS Windows

➜ ~ file sqlite-tools-linux-x64-3500400/sqlite3
sqlite-tools-linux-x64-3500400/sqlite3: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=26bd7240a96c1cadc9c7ffe878c7e0a4b554137e, for GNU/Linux 3.2.0, stripped
```


ELF (Linux), Mach-O (macOS), and PE (Windows, where `.exe` and `.dll` files belong) are executable file formats used by different operating systems to store programs (executables, libraries, etc.).

The file format tells the OS how to read the binary:

* where the code (`.text`) section is, where the data (`.data`) section is, and where the `bss` (Block Started by Symbol) section is for static or uninitialized variables. Unlike `.data`, which stores initialized variables, `.bss` holds zero-initialized or uninitialized data. These are concepts rooted in assembly and CPU memory organization.
* where to link libraries at runtime (dynamic libs like `.so`, `.dll`, etc.)
* where to start execution
* which architecture it's built for (ARM, x64, etc.)
* whether the binary is little-endian (least significant byte first) or big-endian (most significant byte first)

Basically, compilers turn high-level code into assembly, which is then converted into machine code. That machine code is stored using the OS's executable format (ELF, Mach-O, or PE), and the layout depends on that format. It's essentially the interface between the OS and the compiler.

| Feature              | ELF                 | Mach-O                 | PE                     |
| -------------------- | ------------------- | ---------------------- | ---------------------- |
| Primary OS       | Linux, Unix-like    | macOS, iOS             | Windows                |
| Architecture     | 32-bit, 64-bit      | 32-bit, 64-bit + Fat   | 32-bit, 64-bit (PE32+) |
| File extension   | Usually none, `.so` | Usually none, `.dylib` | `.exe`, `.dll`         |

---

The instruction set (machine code) is one part of a larger system called the ABI (Application Binary Interface).
The ABI also defines:

* **Data type layout and alignment** for example, an `int` is 4 bytes on 32-bit systems, but a `long` is 8 bytes on 64-bit.
  Data must be aligned: an `int` must start at an address that's a multiple of 4. CPUs fetch memory in word-sized chunks (4 or 8).

```
// int starts at 0x1001 (not aligned, CPU must fetch 2 words)
|---- word 1 ----|---- word 2 ----|
 0x1000          0x1004          0x1008
       [ int = 4 bytes starting at 0x1001 ]
```

Padding ensures proper alignment:

```c
struct Example {
    char a;   // 1 byte
    // 3 bytes of padding so 'b' starts on a 4-byte boundary
    int  b;   // 4 bytes
};
```
```
The order of fields in a struct affects its size.
Example:
`char, int, char` → `1 + 3 + 4 + 1 + 3 = 12 bytes`
`int, char, char` → `4 + 1 + 1 + 2 = 8 bytes`
A rule  of thumb is to place larger fields first to minimize padding.
```





* **Calling conventions** which registers hold function arguments, which register stores the return value, etc.

* **System call mechanisms** how user space code requests kernel services.

* Other things.

The ABI defines the low-level contract between compiled code and the OS at the stack/register level, while the API operates at a higher level (e.g., function calls in a library).
The ABI depends on both the OS and architecture, since registers and instruction sets are CPU-specific and defines how the OS interacts with these binaries.

So, the same C program compiled with the same options on the same CPU architecture will produce two different binaries: one ELF (Linux) and one PE (Windows). The machine instructions (`.text` section) are mostly the same, but system calls, libraries, and linking conventions differ, because the ABI differs.

```c
int main() { return 42; }
// On both Linux and Windows, this compiles to a few instructions (mov eax, 42; ret),
// but the binary format, linker, and libraries differ.
```


### Calling Assembly from C

```c
// main.c
#include <stdio.h>

// Declare the external assembly function
extern int add_two(int x, int y);

int main(void) {
    int result = add_two(5, 7);
    printf("Result: %d\n", result);
    return 0;
}
```

```
// macOS ARM64
    .text
    .globl _add_two
_add_two:
    add w0, w0, w1         // w0 = w0 + w1
    ret                    // return in w0
```

```sh
clang -arch arm64 -o main main.c add_two.s
./main
# Result: 12
```

Here, we're calling the `add_two` function written in assembly. If you look at the Linux kernel source code, you'll find plenty of examples where certain operations need to be done at a lower level, requiring assembly instead of C.

The same `add_two` function would look slightly different on Linux ARM64 (due to ABI differences, not instruction differences) and even more different on x86_64.

---

### Shared Libraries

Just like we use dependencies in Python, JavaScript, or Java, C programs rely on shared libraries: `.so` on Linux, `.dylib` on macOS, and `.dll` on Windows. These are precompiled code modules that can be used at runtime by multiple programs.
A single copy is shared in memory, making them efficient. Because they're loaded at runtime, we can update one shared library (say, OpenSSL) and automatically patch all programs that depend on it, no need to recompile each one.

```c
// example.c
void hello() { printf("Hello from the dynamically loaded library!\n"); }
// echo "compile shared lib" && gcc -c -fpic example.c -o example.o && gcc -shared -o libexample.so example.o
```

```c
// lib.c
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>

int main() {
    void* lib_handle = dlopen("./libexample.so", RTLD_LAZY);

    if (!lib_handle) return EXIT_FAILURE;
    void (*hello_func)() = dlsym(lib_handle, "hello");

    if (dlerror() != NULL) return EXIT_FAILURE;
    hello_func();

    dlclose(lib_handle);
    return EXIT_SUCCESS;
}
```

On Linux, the dynamic linker `ld.so` (or `ld-linux.so`) is responsible for loading `.so` libraries at runtime.
When a program requests a library, `ld.so` searches several locations: `/lib`, `/usr/lib`, directories listed in the `LD_LIBRARY_PATH` environment variable, and paths from `/etc/ld.so.conf` or `/etc/ld.so.conf.d/`.
To avoid expensive lookups each time, a cache (`/etc/ld.so.cache`) is maintained, listing all available libraries and their paths.

You can inspect which shared libraries a program uses with:

```sh
ldd /bin/ls
	linux-vdso.so.1 (0x0000fe2181b92000)
	libselinux.so.1 => /lib/aarch64-linux-gnu/libselinux.so.1 (0x0000fe2181af7000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000fe2181984000)
	/lib/ld-linux-aarch64.so.1 (0x0000fe2181b62000)
	libpcre2-8.so.0 => /lib/aarch64-linux-gnu/libpcre2-8.so.0 (0x0000fe21818f6000)
	libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2 (0x0000fe21818e2000)
	libpthread.so.0 => /lib/aarch64-linux-gnu/libpthread.so.0 (0x0000fe21818b1000)
```

On Windows, dynamic-link libraries (`.dll`) serve the same purpose (lots of memories for anyone who gamed on Windows).
On macOS, the equivalent is `.dylib`.

All of these library files conform to their respective executable formats ( **ELF** (Linux), **PE** (Windows), and **Mach-O** (macOS)) just different "flavors" of binary layouts.

## Python 
Python is the number one programming language in the TIOBE index.
For me personally, it feels so damn convenient, the idea-to-code transition feels effortless, and the language bends to your will. Performance concerns are for those who don't appreciate list comprehensions and the expressive beauty of Python.

Anyway, jokes aside, let's look at how Python is actually run and packaged.
First of all, unlike its parent C (Python is written in C), it's interpreted.
However, "interpreted" doesn't mean "no compilation."
The interpreter reads the source code and parses it into an Abstract Syntax Tree (AST).



```py
# hello.py
def say_hi(name):
    print(f"Hi, {name}!")
````

Running
`python3 -c "import ast; print(ast.dump(ast.parse(open('hello.py').read()), indent=2))"`
produces:

```py
Module(
  body=[
    FunctionDef(
      name='say_hi',
      args=arguments(
        posonlyargs=[],
        args=[arg(arg='name')],
        kwonlyargs=[],
        kw_defaults=[],
        defaults=[]),
      body=[
        Expr(
          value=Call(
            func=Name(id='print', ctx=Load()),
            args=[
              JoinedStr(
                values=[
                  Constant(value='Hi, '),
                  FormattedValue(
                    value=Name(id='name', ctx=Load()),
                    conversion=-1),
                  Constant(value='!')])],
            keywords=[]))],
      decorator_list=[]),
    Expr(
      value=Call(
        func=Name(id='say_hi', ctx=Load()),
        args=[Constant(value='Cef')],
        keywords=[]))],
  type_ignores=[])
```

This is the Abstract Syntax Tree (AST) representation of the code.
The tree is then compiled into bytecode (`.pyc` files), which is different from machine code.

```py
python3 -c "import dis; dis.dis(lambda a: a + 2)"
  1           0 LOAD_FAST                0 (a)
              2 LOAD_CONST               1 (2)
              4 BINARY_ADD
              6 RETURN_VALUE
```

`dis` is Python's disassembler for bytecode.

Python bytecode is stack-based:
It pushes values (variables, constants) onto a stack (`LOAD_FAST`, `LOAD_CONST`), performs an operation (`BINARY_ADD`), then returns a value (`RETURN_VALUE`).

Python caches compiled bytecode into `.pyc` files to skip recompilation next time.
Each `.pyc` contains a hash of the source file or a timestamp to check if recompilation is needed.

At its core, Python runs a loop that executes these bytecode instructions, like a virtual CPU running on top of C.
This is, of course, a simplification, but it's a useful mental model.

### Packages and Imports

Like most languages, Python relies on dependencies.
Interpreted code makes dependencies more flexible, you can inspect, modify, or even replace modules at runtime.

Python organizes code into modules and packages:

* A module is any `.py` file.
* A package is a directory containing an `__init__.py` file.
* A package is also a module (it can define objects), but a module isn't always a package.
* Code inside `__init__.py` runs when the package is imported, useful for setting up package-level constants, logging, or initialization logic.

When you `import` a module, e.g. `import requests`, Python begins module resolution, it searches for the module along directories listed in `sys.path`.

You can modify `sys.path` with the environment variable `PYTHONPATH`:

```sh
> python3 -c "import sys; print(sys.path)"
['', '/usr/lib/python3.13', '/usr/local/lib/python3.13/dist-packages', '/usr/lib/python3/dist-packages', '/usr/local/lib/python3.13/site-packages']

# add a custom directory
> export PYTHONPATH="/tmp/toto"
> python3 -c "import sys; print(sys.path)"
['', '/tmp/toto', '/usr/lib/python3.13', '/usr/local/lib/python3.13/dist-packages', '/usr/lib/python3/dist-packages', '/usr/local/lib/python3.13/site-packages']
```

Behind the scenes, `sys.path` is managed by a path finder.
Python uses the built-in `PathFinder` (in `sys.meta_path`) to locate modules, but you can extend this system.
By implementing a class with a `find_spec` method and adding it to `sys.meta_path`, you can load modules from anywhere, the network, GitHub gists, or even from an LLM (half joking).


Python can have non-Python dependencies, especially in performance-critical areas. Libraries like **NumPy**, **Pandas**, and **PyTorch** are mostly built in C or C++. This means that, unlike regular interpreted Python code (which runs the same on any OS or architecture), these native dependencies must be compiled for specific platforms.

Python **wheels** (`.whl` files) are the standard way Python packages are distributed. Like most things in life (almost not a joke), they're simply ZIP archives with a defined structure. You can have:

* **Pure Python wheels**: platform-independent, contain only `.py` files.
* **Platform-specific wheels**: include native compiled extensions (`.so`, `.pyd`, `.dylib`, etc.), tied to an OS and architecture.

For example, [NumPy](https://pypi.org/project/numpy/#files) currently lists over 70 wheels on PyPI, depending on Python version and platform:

```
numpy-2.3.4-cp313-cp313-macosx_14_0_x86_64.whl
numpy-2.3.4-cp313-cp313-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl
numpy-2.3.4-cp313-cp313-win_arm64.whl
```

---

### Dependency Resolution

`pip` (Python's package manager) resolves dependencies by building a dependency tree. For example:

```sh
$ pipdeptree
Warning!!! Possibly conflicting dependencies found:
* Jinja2==2.11.2
 - MarkupSafe [required: >=0.23, installed: 0.22]
------------------------------------------------------------------------
Flask==0.10.1
  - itsdangerous [required: >=0.21, installed: 0.24]
  - Jinja2 [required: >=2.4, installed: 2.11.2]
    - MarkupSafe [required: >=0.23, installed: 0.22]
  - Werkzeug [required: >=0.7, installed: 0.11.2]
Lookupy==0.1
pipdeptree==2.0.0b1
  - pip [required: >=6.0.0, installed: 20.1.1]
setuptools==47.1.1
wheel==0.34.2
```

In `requirements.txt` or modern configuration files like `pyproject.toml`, you specify dependencies with version constraints —
for example: `"aiohttp>=3.9.0"`, `"xdg-base-dirs==6.0.2"`.
`pip` will fetch those packages and their transitive dependencies.

An interesting feature of `pip` is [backtracking](https://pip.pypa.io/en/stable/topics/dependency-resolution/#backtracking).
If two packages require conflicting versions of the same dependency (e.g., package A needs `C <= 2.0` and package B needs `C == 2.1`), `pip` will automatically backtrack to earlier compatible versions when possible to resolve the conflict.

---

### Installation Paths and Environments

When you install a package with `pip`, it's placed in a **`site-packages`** directory:

* A **global** one (shared across the system),
* A **per-user** one (`--user` installs), and
* One **per virtual environment** used to isolate dependencies for individual projects.

In addition to `pip`, there are several other package managers, notably **`poetry`**, **`conda`**, and **`uv`**.
`uv` has been gaining a lot of momentum recently. You can install with just a
`curl <url> | sh` and I'm a just a sucker for that. It's faster, bundles the functionality of a lot of other tools and works. There is something magical about tools like it, that untangle a complex space and provide a clean solution that just works.

### Calling C from Python
```c
// say_hi.c
#include <Python.h>

static PyObject* say_hi(PyObject* self, PyObject* args) {
    const char* name;
    if (!PyArg_ParseTuple(args, "s", &name)) return NULL; // parse arg into name
    printf("Hi %s!\n", name); // print
    Py_RETURN_NONE; // return None
}

// define the method
static PyMethodDef SayHiMethods[] = {
    {"say_hi", say_hi, METH_VARARGS, "Say hi"},
    {NULL, NULL, 0, NULL}
};

// define the module
static struct PyModuleDef sayhimodule = {
    PyModuleDef_HEAD_INIT,
    "say_hi",
    NULL,
    -1,
    SayHiMethods
};

PyMODINIT_FUNC PyInit_say_hi(void) {
    return PyModule_Create(&sayhimodule);
}
````

```py
# setup.py
from setuptools import setup, Extension

module = Extension("say_hi", sources=["say_hi.c"])
setup(name="say_hi", ext_modules=[module])
```

```sh
# build the extension and call the C function from Python
python3 setup.py build_ext --inplace
python3 -c "import say_hi; say_hi.say_hi('Cef')"
# Hi Cef!
```

On Linux with Python 3.12, this generates a file like
`say_hi.cpython-312-x86_64-linux-gnu.so`, a **shared library object** containing your compiled C code.
Under the hood, **CPython** uses `dlopen` (as seen earlier) to load this shared object dynamically.

The C code (`PyMethodDef`, `PyArg_ParseTuple`, etc, part of the Python C API) defines the interface between C and Python, handling argument parsing, return values, and reference management.
In this example, we just print a string, but we could just as easily call powerful C libraries or run compute-intensive operations, then pass the result back to Python.



## Java


The JVM. The on-call pager alerts. The garbage collector. In other words, the glamorous life.
Java is the tried-and-true workhorse of the enterprise.

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hi Cef!");
    }
}
```

```sh
$ java Hello.java
Hi Cef!
```

Splendid! What happens with Java?
Much like Python, Java uses its own bytecode, which is platform-independent and executed by the Java Virtual Machine (JVM).
`.java` source files are first compiled into `.class` files containing this bytecode.

The JVM also uses JIT (Just-In-Time) compilation, which compiles frequently executed ("hot") sections of bytecode into native machine code for better performance.
In practice, the JVM runs a mix of interpreted bytecode and JIT-compiled native code. It continuously profiles execution and compiles the hottest methods.

There is also **GraalVM**, which can compile a Java application ahead of time into a native executable. This eliminates JVM startup overhead and can greatly improve performance, but the resulting binary becomes platform-dependent, unlike portable `.class` bytecode that runs anywhere a JVM exists.


### Why is Java Faster than Python?

If both Python and Java are interpreted, why is Java way faster than Python?

First of all, translation to bytecode is done ahead of time for Java, unlike Python. Although `.pyc` files are cached, they don't offer the same level of optimization.

Second, when the JVM "warms up" and the JIT compiler has done its thing (turning frequently accessed bytecode into machine code), execution becomes much faster.

Thirdly, there's dynamic vs. static typing. Python is dynamically typed, meaning types need to be checked at runtime, unlike Java where type checking is done once at compile time.

Finally, Python has the GIL (Global Interpreter Lock), which means only one thread can execute Python bytecode at a time, even if there are multiple processors. This has changed with the latest release of Python (3.14), which offers an optional GIL-free interpreter, bringing notable performance improvements for CPU heavy programs.



### Class Loader

Java loads dependencies using class loaders, which operate in a hierarchical structure:

1. **Bootstrap ClassLoader**  loads core Java classes from (`java.lang`, `java.util`, etc)
2. **Platform (or Extension) ClassLoader**  loads  platform modules (`java.sql`, `java.security.sasl`, etc )
3. **Application (System) ClassLoader**  loads from the **classpath**, where user-defined classes and third-party libraries live.

Java packages compiled bytecode (`.class` files) into JARs (Java Archives), which are ZIP files in disguise (is everything really a zip file?).
When you run a Java program, the JVM uses the application class loader to locate and load these classes from the classpath, a list of directories or JARs.

If a class can't be found, you'll see the infamous `ClassNotFoundException`.
You can specify the classpath manually via the `-cp` (or `-classpath`) flag or the `CLASSPATH` environment variable:

```sh
java -cp "lib/library1.jar:lib/library2.jar:." MyToto
```

You can even define your own custom class loader by extending the `ClassLoader` class and overriding methods like `findClass` and `loadClassData`.

Fun fact: back in the day, Java classes were loaded *inside web browsers*! This was done via an `AppletClassLoader` that fetched Java classes over HTTP. Wild times!


When you use a package manager like **Maven** or **Gradle**, you specify your dependencies (in files like `pom.xml`), and the required JARs are downloaded from a central repository, most famously, Maven Central.
Downloaded JARs are stored locally in `~/.m2/repository`, where Maven can reuse them across projects. These JARs are then added to the classpath during compilation and runtime. Maven also automatically handles transitive dependencies, dependencies of your dependencies.

Maven builds a dependency tree, and if a conflict occurs (e.g., two different versions of the same library), it applies the following resolution rules:

1. The dependency closest to the root (direct dependency) wins over transitive ones.
2. If they're at the same level, the first declared one in the `pom.xml` takes precedence.

---

### Thin vs. Uber JARs

You can have:

* Thin JARs, which include only your application code, while dependencies are loaded separately from the classpath.
* Uber JARs (also called "fat JARs"), which bundle your application and all its dependencies into a single JAR file.

The term 'Uber' JAR always sounded peculiar to me but with a large number of dependencies, potential version conflicts, and missing classes, "JAR hell" becomes a very real problem.

### ServiceLoader and Interfaces

Java is big on abstraction and interfaces.
One particularly elegant feature is the ServiceLoader mechanism, which allows you to define service interfaces and load implementations at runtime, without modifying code.

For example, you might define an interface:

```java
public interface TransportMethod {
    void travel();
}
```

Then, provide a configuration file at runtime:
`META-INF/services/com.cefboud.TransportMethod`

The file contains the name of the implementation class, e.g.:

```
com.cefboud.SpaceTravel
```

By changing the contents of that file, or by adding a new JAR with a different implementation, you can swap implementations dynamically. Pretty slick.

A real-world example is SLF4J (the Simple Logging Facade for Java).
It provides a unified logging API, while the actual logging backend (Logback, Log4j, etc.) can be swapped simply by changing which implementation JAR is present, all powered by the `ServiceLoader` mechanism.


### Running C from Java
This is done through JNI (Java Native Interface).
```c
// say_hi.c
#include <jni.h>
#include <stdio.h>
#include "SayHi.h"  // generated header from javac -h

// JNI function signature
JNIEXPORT void JNICALL Java_SayHi_sayHi(JNIEnv *env, jobject obj, jstring name) {
    const char *cname = (*env)->GetStringUTFChars(env, name, NULL);
    if (cname == NULL) return;
    printf("Hi %s!\n", cname);
    (*env)->ReleaseStringUTFChars(env, name, cname);
}
```
```java
// SayHi.java
public class SayHi {
    // Load the native library
    static {
        System.loadLibrary("say_hi"); // links to libsay_hi.so
    }

    // Declare native method
    public native void sayHi(String name);

    public static void main(String[] args) {
        new SayHi().sayHi("Cef"); // call C function
    }
}
```

```sh
# javac -h to generate header files for native methods
javac -h . SayHi.java
# generate ibsay_hi.so
clang -fPIC -shared -o libsay_hi.so say_hi.c
java -Djava.library.path=. SayHi
# Hi Cef!
```
Almost the exact same thing as Python. A C file to define how the Java code should interface with the C code (arg parsing, func definition, actual execution, return). Again, this is a simple print method, but we can wrap any powerful C code. One example is [Rockdb](https://github.com/facebook/rocksdb/tree/e687ca79b42ca8673de8ad50c97f3e8b9eefe414/java) which is written in C++ but exposes a full Java interface through JNI.

There is a gazillion more things about packaging and running Java apps, but let's stop here

## Go

Golang is a language I like quite a lot, it just oozes readability. I'm not a huge fan of the `err != nil` pattern, but you take the good with the bad in life.

Go is statically compiled. Unlike Java or Python, there's no interpreter shenanigans. Like Java, though, it has a Garbage Collector (GC). You don't manually `malloc` or `free` memory like in C; instead, the GC tracks object usage and frees memory for objects no longer in use.

Of course, garbage collection has a reputation for introducing overhead and hurting performance. The time spent scanning memory and reclaiming unused objects is time not spent running your code, for extremely latency-sensitive systems, this can be a deal-breaker.
Java has what's called a Stop-The-World pause, where the application briefly halts to perform GC or JIT compilation. Golang's GC and Java modern GC are much more sophisticated and better performant.

So when we say Go is garbage-collected, that means there's extra code bundled within your compiled binary that performs these memory management tasks, and that's exactly what happens.


```go
package main

import "fmt"

func main() {
    fmt.Println("Hi Cef!")
}
```
```sh
go build -o say_hi  main.go 
./say_hi 
# Hi Cef!
file say_hi 
# say_hi: Mach-O 64-bit executable arm64
ls -lh say_hi | awk '{print $5}' 
# 2.2M
```
It's a plain executable file. On mac, a Mach-O binary, as we discussed earlier.
But in addition to the code we wrote, the binary also includes the Go runtime: the garbage collector, the Go scheduler, and other runtime niceties.

That's why even such a trivial program ends up being around 2.2 MB in size, it contains not just our code, but the whole runtime needed for memory management, goroutine scheduling, etc.

`nm` is unix command to list symbols in binaries. We can use it to inspect runtime and GC symbols in the Go binary.
```sh
nm say_hi | grep 'runtime\.' | head -n7
00000001000ec3c0 d _go:itab.runtime.errorString,error
00000001000ec3e0 d _go:itab.runtime.plainError,error
000000010016c5d0 d _go:runtime.inittasks
0000000100175f60 b _runtime._cgo_setenv
0000000100175f68 b _runtime._cgo_unsetenv
00000001000432c0 t _runtime._ExternalCode
0000000100043340 t _runtime._GC
```

Now, let's compare that to a simple C program:

```c
#include <stdio.h>

int main() {
    printf("Hi Cef!\n");
    return 0;
}
// clang -o say_hi main.c
//ls -lh say_hi | awk '{print $5}'
// 33K
```

With C, we get a binary that's only 33 KB!
That's because C relies on the system's C standard library, famously libc on Linux, which is dynamically linked at runtime.
The C standard library is the interface between C programs and the operating system kernel.
On mac, the equivalent is `libSystem.dylib`.

```sh
otool -L say_hi 
say_hi:
        /usr/lib/libSystem.B.dylib 
```
It's possible to statically link in C as well, bundling the shared libraries into the final executable. This produces a larger binary and higher memory usage (since each binary carries its own copy of the libraries) but avoids dependency issues, useful for highly portable or embedded builds.

Go, however, goes all-in on static linking. You get a single, self-contained binary that includes everything it needs: runtime, garbage collector, scheduler, and standard library.
It "just works" which is exactly why Go binaries are so well-suited for containers and cloud.

### Go Modules and Minimal Version Selection

Go is famous for its integrated and rich native capabilities: the powerful standard library, built-in testing tools, formatter, builder, linter, and package manager all come with the language itself.

A Go module (a directory with a `go.mod` file at its root) is the basic unit of dependency management in Go. One particularly elegant feature is that a Go module lists both its direct and indirect (transitive) dependencies in the same `go.mod` file.
This means that when you add a new dependency, simply retrieving the listed dependencies (both direct and indirect) is enough, there's no need to traverse multiple layers of nested dependency trees. This greatly simplifies reproducibility and builds.

Go uses a strategy called [Minimal Version Selection (MVS)](https://go.dev/ref/mod#minimal-version-selection) when resolving dependencies.
Instead of always upgrading to the latest version, Go picks the oldest version that satisfies all dependency requirements.

For example:

* Module **A** depends on **D v1.2.0**
* Module **B** depends on **D v1.1.0**
* Versions of **D** are available up to **v1.5.0**

Go will select **v1.2.0**, the minimal version that satisfies all constraints, even though newer versions exist.

This approach ensures deterministic and reproducible builds.
If Go were to always fetch the newest available version, the build results could change unpredictably over time. With MVS, you'll always get the *same dependency graph* no matter when or where you build your project.


### Calling C from Go

```go
// main.go
package main

/*
#include <stdio.h>

void say_hi(const char* name) {
    printf("Hi from C, %s!\n", name);
}
*/
import "C"

func main() {
    C.say_hi(C.CString("Cef"))
}

```

```sh
go run main.go                              
#Hi from C, Cef!
```

Golang relies on CGO to interact with C code and libraries. Compared to the previous examples (Java and Python), it feels much simpler and more direct.
A fun thing to notice: once you use CGO, the generated Go binary becomes linked to the C standard library, so it's no longer purely statically linked like normal Go programs.

You can also specify C compiler flags via environment variables such as `CFLAGS` and `LDFLAGS` to control compilation and linking behavior, or interact directly with shared libraries (`.so`, `.dylib`, etc.).

CGO even allows you to call Go functions from C, enabling powerful and risky two-way integration between Go and native code.


## Conclusion
It really is fascinating that all this stuff works. And how well it works! I hope you learned a thing or two and had as much fun reading as I did exploring and writing!

If you want to get in touch, you can find me on Twitter/X at [@moncef_abboud](https://x.com/moncef_abboud).  
