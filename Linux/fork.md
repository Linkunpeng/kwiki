# fork()

A `fork()` system call creates a new process (the child process) that is an almost exact copy of the calling process (the parent process). However, the child process doesn't *execute* an ELF file directly *after* the `fork()`. Instead, the typical flow is `fork()` followed by `execve()` (or one of its variants) to load and execute a new ELF file.

Let's break down the process from `fork()` to `main()` when a new ELF file is executed:

---

### 1. The `fork()` System Call

When `fork()` is called:

* **Process Duplication:** The kernel creates a new process descriptor for the child. It copies most of the parent's process attributes, including its memory space (virtually, using copy-on-write), open file descriptors, signal handlers, and current working directory.
* **Copy-on-Write (CoW):** Crucially, the memory pages of the parent are not physically copied immediately. Instead, both parent and child share the same physical memory pages, marked as read-only. Only when either process attempts to *write* to a shared page is a private copy of that page created for the modifying process. This makes `fork()` very efficient.
* **Return Values:** `fork()` returns 0 in the child process and the process ID (PID) of the child in the parent process. A negative value indicates an error.

At this point, both the parent and child processes are executing the *same code* right after the `fork()` call. The child is a duplicate of the parent.

---

### 2. The Role of `execve()`

If the goal is to run a different program (an ELF executable), the child process (or sometimes the parent, but usually the child) will then call one of the `exec()` family of functions, most commonly `execve()`.

When `execve()` is called:

* **Memory Replacement:** This is the critical step for ELF file loading. `execve()` does *not* create a new process. Instead, it **replaces the entire memory space** of the *current* process with the new program. This means:
    * The existing code, data, heap, and stack segments of the calling process are discarded.
    * The program image of the new ELF file is loaded into memory.
* **Process ID (PID) Preservation:** The PID of the process remains the same. It's the same process, just running a new program.
* **Argument and Environment Passing:** The `execve()` function takes the path to the new executable, an array of command-line arguments, and an array of environment variables. These are set up for the new program.
* **File Descriptors:** Open file descriptors from the previous program are typically preserved unless marked with `O_CLOEXEC` (Close-on-exec).
* **No Return:** If `execve()` is successful, it *never returns* to the calling code. The new program begins execution from its entry point. If it fails, it returns -1.

---

### 3. ELF File Loading by the Kernel

When `execve()` is called and the kernel identifies the specified file as an ELF executable, the following steps occur:

* **ELF Header Parsing:** The kernel reads the ELF header from the specified file to verify its validity, architecture, and entry point.
* **Program Header Table (PHT) Parsing:** The kernel then reads the Program Header Table. The PHT describes how the different segments (e.g., code, data, read-only data) of the ELF file should be mapped into the process's virtual memory space.
* **Memory Mapping:** For each loadable segment described in the PHT (typically `PT_LOAD` segments):
    * The kernel uses `mmap()` internally to map regions of the ELF file into the process's virtual address space. These mappings are done with appropriate permissions (read, write, execute) as specified in the ELF program headers.
    * The code (`.text`) segment is mapped as read-only and executable.
    * The initialized data (`.data`) segment is mapped as read-write.
    * The uninitialized data (`.bss`) segment is typically zero-filled and mapped as read-write.
* **Stack and Heap Setup:** A new process stack is created and initialized, including pushing the command-line arguments and environment variables. The program break (for the heap) is typically set up as well.
* **Dynamic Linker (if applicable):** If the ELF executable is a dynamically linked executable (which most are), the kernel maps the **dynamic linker/loader** (e.g., `/lib64/ld-linux-x86-64.so.2` on Linux) into the process's memory space.
* **Entry Point Transfer:**
    * For statically linked executables, the kernel directly transfers control to the ELF file's `_start` entry point.
    * For dynamically linked executables, the kernel transfers control to the entry point of the **dynamic linker**.

---

### 4. Dynamic Linker's Role (for dynamically linked executables)

If the dynamic linker is involved, it takes over before `main()` is called:

* **Shared Library Loading:** The dynamic linker reads the ELF file's `DT_NEEDED` (shared library dependencies) entries and recursively loads all required shared libraries into the process's address space.
* **Relocations and Symbol Resolution:** It performs necessary relocations (e.g., adjusting addresses within the loaded libraries) and resolves symbols (e.g., finding the addresses of functions like `printf` from `libc`) to link the executable with its shared libraries.
* **Initialization Functions:** It executes any necessary initialization functions (`.init` sections, or constructors) from the main executable and the loaded shared libraries.
* **Transfer to `_start`:** Finally, the dynamic linker transfers control to the executable's own `_start` symbol.

---

### 5. From `_start` to `main()`

* **`_start` Function:** This is the actual entry point of your compiled C/C++ program (even though you write `main()`). It's typically provided by the C runtime library (e.g., `glibc`).
* **C Runtime Initialization:** The `_start` function performs further essential setup before `main()` is called, including:
    * Initializing the C runtime environment.
    * Setting up the stack.
    * Processing command-line arguments (`argc`, `argv`) and environment variables (`envp`) so they can be passed to `main()`.
    * Calling any global constructors (for C++ programs).
* **Calling `main()`:** Once all the necessary runtime setup is complete, `_start` makes a `call` instruction to your program's `main()` function, passing `argc`, `argv`, and `envp`.

This detailed sequence illustrates that `fork()` is about process duplication, and `execve()` is about program replacement and ELF loading, working together to launch new applications.
