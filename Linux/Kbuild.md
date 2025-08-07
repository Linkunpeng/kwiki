# Kbuild

## make config
make menuconfig 与 make oldconfig 的区别在于oldconfig 仅处理新增选项


当然可以。在Kbuild构建系统中，将 `.config` 文件中的配置变量（如 `CONFIG_HZ`）传递到C头文件（`.h`）以便在内核代码中使用，是一个自动化且严谨的过程。这个过程的核心是生成一个名为 `autoconf.h` 的头文件。

下面是这个过程的详细路径和步骤分解：

### 总览路径

`.config` -\> `make` -\> `scripts/kconfig/conf` -\> `include/generated/autoconf.h` -\> C代码

-----

### 详细步骤分解

#### 1\. 起点：`.config` 文件

这一切都始于内核源码根目录下的 `.config` 文件。当您运行 `make menuconfig`、`make xconfig` 或其他配置工具并保存时，您的所有选择都会被写入这个文件。

对于 `CONFIG_HZ`，您在配置菜单中可能会选择一个具体的频率值（如100, 250, 300, 1000 Hz）。在 `.config` 文件中，这通常会被保存为多个 `CONFIG_HZ_xxx` 变量中的一个，以及一个总的 `CONFIG_HZ` 变量。

例如，如果您选择了1000Hz，`.config` 文件中会包含类似下面这两行：

```text
# CONFIG_HZ_250 is not set
# CONFIG_HZ_300 is not set
CONFIG_HZ_1000=y
CONFIG_HZ=1000
```

  * `CONFIG_HZ_1000=y`：这是一个布尔值，表示选中了1000Hz的选项。
  * `CONFIG_HZ=1000`：这是一个整数值，直接定义了 `HZ` 的值。

#### 2\. 触发：`make` 命令

当您在内核源码树中运行 `make`（或 `make bzImage`, `make modules` 等）时，Kbuild构建系统开始工作。顶层 `Makefile` 会检查 `.config` 文件是否比生成的配置文件旧。如果需要更新，它会触发一个配置更新的目标。

这个过程会调用一个关键的配置工具。

#### 3\. 核心工具：`scripts/kconfig/conf`

`make` 命令会调用位于 `scripts/kconfig/` 目录下的 `conf` 这个宿主机程序。`conf` 程序的作用是读取 `Kconfig` 文件和 `.config` 文件，并根据这些输入生成多个输出文件，其中最重要的一个就是C头文件。

`conf` 程序被调用时，通常会带上 `--autoconf=` 或 `--autoheader=` 这样的参数，来指定生成的C头文件的路径和名称。

#### 4\. 中间产物：`include/generated/autoconf.h`

`conf` 程序执行后，会生成 `include/generated/autoconf.h` 文件。这个文件是 `.config` 和C代码之间的桥梁。它会将 `.config` 文件中的 `CONFIG_` 变量转换成C语言的宏定义。

继续以 `CONFIG_HZ` 为例，`autoconf.h` 中会生成如下内容：

```c
/*
 * Automatically generated file; DO NOT EDIT.
 * Linux/x86 6.x.x Kernel Configuration
 */
// ... 其他宏定义
#define CONFIG_HZ_1000 1
#define CONFIG_HZ 1000
// ... 其他宏定义
```

  * `CONFIG_HZ_1000=y` 被转换为了 `#define CONFIG_HZ_1000 1`。
  * `CONFIG_HZ=1000` 被转换为了 `#define CONFIG_HZ 1000`。

这样，任何包含了这个头文件的C代码都可以直接使用这些宏。

#### 5\. 终点：在C代码中使用

现在，内核的C代码就可以使用这些宏了。但是，几乎没有C文件会直接 `#include <generated/autoconf.h>`。这个过程是通过一个更加通用的包含链和编译器强制包含来实现的。

**路径1：通过 `kconfig.h` 间接包含**

1.  Kbuild在编译每个C文件时，会通过 `-include ./include/linux/kconfig.h` 编译器选项，强制将 `kconfig.h` 文件包含到每个编译单元中。

2.  打开 `include/linux/kconfig.h` 文件，您会看到它包含了我们生成的头文件：

    ```c
    #ifndef __LINUX_KCONFIG_H
    #define __LINUX_KCONFIG_H

    #include <generated/autoconf.h> // 在这里包含了 autoconf.h

    /* ... */

    #endif /* __LINUX_KCONFIG_H */
    ```

**路径2：通过其他头文件包含**

1.  一个典型的内核文件，比如 `kernel/time/tick-sched.c`，需要知道 `HZ` 的值。它会包含 `linux/tick.h`。

2.  `linux/tick.h` 可能会包含 `asm/param.h`。

3.  `asm/param.h` (位于 `arch/x86/include/asm/param.h`) 中定义了 `HZ` 宏，它的值直接来源于 `CONFIG_HZ`。

    ```c
    #ifndef _ASM_X86_PARAM_H
    #define _ASM_X86_PARAM_H

    #ifdef __KERNEL__
    # define HZ		CONFIG_HZ	/* Internal kernel timer frequency */
    # define USER_HZ	100		/* .. some user interfaces are in "ticks" */
    # define CLOCKS_PER_SEC	(USER_HZ)	/* like times() */
    #endif

    #endif /* _ASM_X86_PARAM_H */
    ```

由于 `autoconf.h` 已经被`kconfig.h`包含，并且`kconfig.h`被强制包含在每个文件中，所以当编译器处理到 `asm/param.h` 时，`CONFIG_HZ` 宏已经被定义为了 `.config` 中设定的值（例如1000）。因此 `HZ` 也就被成功定义为了1000。

### 总结

所以，`CONFIG_HZ` 的完整旅程是：

**用户配置 (`make menuconfig`) -\> `.config` 文件 (`CONFIG_HZ=1000`) -\> `make` 命令 -\> `scripts/kconfig/conf` 工具 -\> `include/generated/autoconf.h` (`#define CONFIG_HZ 1000`) -\> `include/linux/kconfig.h` (强制包含) -\> 内核C文件 (如 `asm/param.h` 中 `#define HZ CONFIG_HZ`)。**
