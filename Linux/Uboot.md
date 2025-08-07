# Das U-Boot
-----

### U-Boot 的入口与加载流程

1.  **设备上电与 Boot ROM (BL0/Stage 0):**

      * **入口：** 当 SoC 上电后，CPU 的程序计数器（PC）会被强制设置为 SoC 内部的**Boot ROM**（或称作 Mask ROM）中预定义的一个地址。这个 Boot ROM 是芯片制造商在硬件中固化的代码，无法修改。
      * **加载 BL1：** Boot ROM 的主要任务是作为启动链的第一环。它会根据特定的启动介质（如 eMMC、SD 卡、NAND Flash、SPI Flash 等）和启动引脚配置，寻找并加载启动链的下一阶段代码，通常是你的 **BL1（First Stage Bootloader）**。Boot ROM 通常会将 BL1 加载到 SoC 内部一个很小的、速度很快的 **SRAM**（Static RAM）中。
      * **控制权移交：** BL1 被加载到 SRAM 后，Boot ROM 会将控制权（即将 PC 指向 SRAM 中 BL1 的起始地址）移交给 BL1。

2.  **BL1 (First Stage Bootloader):**

      * **主要任务：** BL1 通常非常小，因为它必须完全放入有限的 SRAM 中。它的主要职责是完成最低限度的硬件初始化，例如：
          * 初始化必要的时钟。
          * 配置 DRAM 控制器，使外部 DDR 内存可用。
          * 根据需要，初始化用于加载 BL2 的存储控制器（如 eMMC/SD/SPI 控制器）。
      * **加载 BL2：** 一旦 DRAM 初始化完成，BL1 就会将启动链的下一阶段代码，通常是 **BL2（Second Stage Bootloader）**，从启动介质加载到可用的 DRAM 中。
      * **控制权移交：** BL1 将控制权移交给 DRAM 中的 BL2。

3.  **BL2 (Second Stage Bootloader / 完整的 U-Boot):**

      * **入口：** 这是你编译的完整 U-Boot 映像的实际入口点。它在 DRAM 中运行，拥有更大的代码空间和可用的内存资源。
      * **主要任务：** BL2 会完成更复杂的硬件初始化，包括：
          * 更多的外设初始化（UART、USB、网络等）。
          * 文件系统支持（FAT、exFAT、Ext4 等）。
          * 提供完整的 U-Boot 命令行接口。
          * **加载 Linux Kernel 和 Device Tree：** BL2 会从存储介质中加载 Linux 内核映像（如 `zImage` 或 `Image`）和设备树文件（`.dtb`）到 DRAM 中。
          * **传递启动参数：** 它还会准备并传递启动参数给 Linux 内核（通常通过 ATAGs 或 FDT/DTB）。
          * **最终控制权移交：** 最后，BL2 会跳转到 Linux 内核的入口点，将控制权移交给内核。

-----

### 如何修改启动方式（检查升级文件）

要实现在加载 Linux 内核之前检查 MMC 分区是否存在升级文件并进行升级，你需要在 **BL2（完整 U-Boot）阶段**进行修改。这是因为只有在这个阶段，U-Boot 才具备：

  * 足够大的内存来处理升级文件。
  * 完整的文件系统支持（如 exFAT、FAT、Ext4），以便识别和读取升级文件。
  * MMC/SD 卡驱动支持，以便写入新的固件。
  * 用户交互能力（命令行），如果需要的话。

修改步骤如下：

1.  **识别存储升级文件的分区：**

      * 你需要预留一个 MMC/SD 卡上的特定分区来存放升级文件。例如，你可以约定为 MMC 设备的某个分区（如 `mmc 0:3`）。
      * 这个分区可以是任何 U-Boot 支持的文件系统格式（FAT、exFAT、Ext4 等）。

2.  **修改 U-Boot 的启动脚本或启动命令：**
    U-Boot 启动时会执行一系列命令。这些命令通常定义在：

      * **`bootcmd` 环境变量：** U-Boot 启动后会自动执行这个环境变量中定义的命令序列。你可以通过 `setenv bootcmd '...'` 来修改它，并用 `saveenv` 保存到存储介质。
      * **默认 `bootcmd` 在编译时定义：** 如果 `bootcmd` 环境变量不存在或无效，U-Boot 会使用编译时配置的默认 `bootcmd`。这个默认值通常在你的板级配置文件（`include/configs/<your_board>.h`）中定义，例如 `CONFIG_BOOTCOMMAND`。

    你需要修改这个 `bootcmd`，使其在加载内核之前插入你的升级逻辑。

3.  **编写升级逻辑：**

    ```bash
    # 假设升级文件名为 "firmware.bin"，位于 mmc 0:3 分区
    # 假设固件烧写工具为 "update_tool.bin"

    # 1. 检查升级文件是否存在
    if fatls mmc 0:3 /firmware.bin; then
        echo "Found upgrade file. Starting upgrade process..."

        # 2. 如果需要，加载升级工具（如果你的升级逻辑比较复杂，需要一个工具）
        # fatload mmc 0:3 ${loadaddr} update_tool.bin
        # go ${loadaddr}
        # 或者，直接在 U-Boot 命令中实现升级逻辑

        # 3. 执行升级操作
        # 这部分是核心，具体取决于你要升级什么以及如何升级。
        # 示例：假设要烧写到 eMMC 的另一个分区（例如，启动分区或用户数据区）
        # 假设 firmware.bin 是一个裸二进制文件，要烧写到 mmc 0 的某个固定地址
        # 请根据你的实际烧写方式调整
        echo "Loading firmware.bin..."
        fatload mmc 0:3 ${loadaddr} firmware.bin
        echo "Erasing target area (e.g., mmc 0:0)..."
        # 假设目标是mmc的某个裸分区，需要先擦除
        # mmc erase 0 0x0 0x1000 # 示例：擦除mmc 0的0x0地址开始的0x1000个扇区
        echo "Writing firmware.bin to target..."
        # 假设烧写到mmc 0的某个固定偏移（这里需要谨慎，错误操作可能砖化设备）
        # mmc write ${loadaddr} 0x0 0x1000 # 示例：从loadaddr写入0x1000个扇区到mmc 0的0x0地址

        # 更安全的升级方式是，先加载到内存，然后验证（CRC/哈希），再烧写
        # crc32 ${loadaddr} ${filesize} && echo "CRC OK" || echo "CRC ERROR! Aborting."

        # 4. 删除升级文件以避免重复升级
        echo "Deleting firmware.bin..."
        fatrm mmc 0:3 firmware.bin

        echo "Upgrade complete. Rebooting..."
        reset # 升级完成后重启
    else
        echo "No upgrade file found. Proceeding with normal boot."
    fi

    # 正常的内核加载和启动流程
    fatload mmc 0:1 ${loadaddr} zImage
    fatload mmc 0:1 ${fdt_addr} ${board_name}.dtb
    bootz ${loadaddr} - ${fdt_addr}
    ```

    **注意：**

      * **安全性和错误处理：** 上述示例非常简化，实际的升级逻辑需要考虑错误处理、断电保护、固件验证（CRC、哈希、数字签名）以及回滚机制，以防止设备变砖。
      * **烧写命令：** `mmc write` 或其他烧写命令非常危险，请务必仔细查阅 U-Boot 文档并了解你的 SoC 和板卡的存储布局。
      * **工具：** 如果升级过程复杂，你可能需要编写一个小的 C 程序并将其编译成 U-Boot 命令（通过 U-Boot 的 `U_BOOT_CMD` 宏注册）或一个裸二进制文件，通过 `go` 命令执行。

4.  **修改 U-Boot 启动项 (`bootcmd`)：**
    有两种主要方式：

    **a. 修改板级配置文件 (推荐，编译时生效):**
    这是最稳健的方式。在你的板级配置文件中（通常是 `include/configs/<your_board>.h`），修改或添加 `CONFIG_BOOTCOMMAND` 宏。

    ```c
    #define CONFIG_BOOTCOMMAND \
        "if fatls mmc 0:3 /firmware.bin; then " \
            "echo \"Found upgrade file. Starting upgrade process...\"; " \
            /* Add your upgrade commands here, e.g., */ \
            "fatload mmc 0:3 ${loadaddr} firmware.bin; " \
            "mmc write ${loadaddr} 0x0 0x1000; " /* 示例：非常危险，请根据实际情况修改 */ \
            "fatrm mmc 0:3 firmware.bin; " \
            "echo \"Upgrade complete. Rebooting...\"; " \
            "reset; " \
        "else " \
            "echo \"No upgrade file found. Proceeding with normal boot.\"; " \
            "fatload mmc 0:1 ${loadaddr} zImage; " \
            "fatload mmc 0:1 ${fdt_addr} ${board_name}.dtb; " \
            "bootz ${loadaddr} - ${fdt_addr}; " \
        "fi"
    ```

    修改后，需要**重新编译 U-Boot 并烧写**到开发板。

    **b. 在 U-Boot 命令行中设置环境变量 (运行时生效，需保存):**
    如果你已经有了一个可运行的 U-Boot，可以在命令行中临时修改 `bootcmd`，并用 `saveenv` 保存到非易失性存储（如 eMMC/NAND/SPI Flash 的环境变量分区）。

    ```bash
    setenv bootcmd 'if fatls mmc 0:3 /firmware.bin; then fatload mmc 0:3 ${loadaddr} firmware.bin; mmc write ${loadaddr} 0x0 0x1000; fatrm mmc 0:3 firmware.bin; reset; else fatload mmc 0:1 ${loadaddr} zImage; fatload mmc 0:1 ${fdt_addr} ${board_name}.dtb; bootz ${loadaddr} - ${fdt_addr}; fi'
    saveenv
    reset
    ```

    这种方式方便调试，但如果环境变量区被擦除或损坏，U-Boot 会回退到编译时定义的 `CONFIG_BOOTCOMMAND`。

-----

### 总结

修改 U-Boot 的启动流程以实现升级功能，是一个涉及 U-Boot 内部机制和嵌入式系统固件管理的复杂任务。你需要：

1.  **深入理解 U-Boot 启动流程和命令行语法。**
2.  **了解你的 SoC 的存储布局和烧写机制。**
3.  **设计健壮的升级策略，包括错误处理和回滚。**
4.  **在 U-Boot 源代码中或通过环境变量修改 `bootcmd`。**

强烈建议在进行此类修改之前，备份你的设备，并在测试环境中进行充分验证。
