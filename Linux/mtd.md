# MTD subsystem


---

### NAND Flash 与分区表

对于 **NAND Flash** 设备，情况确实与传统的硬盘或 eMMC/SD 卡（它们通常使用 **MBR** 或 **GPT** 分区表）不同。

**NAND Flash 通常不包含标准的 MBR 或 GPT 分区表。**

这是因为 NAND Flash 的物理特性与块设备（如硬盘、eMMC、SD 卡）有很大差异：

* **擦除块 (Erase Block)：** NAND Flash 的读写是以**页**为单位，但擦除操作必须以更大的**擦除块**为单位进行。
* **坏块管理 (Bad Block Management)：** NAND Flash 在出厂时就可能存在坏块，并且在使用过程中也会产生新的坏块。NAND 控制器或软件层需要复杂的机制来管理这些坏块，将数据重定向到好的块上。标准的 MBR/GPT 分区表无法直接适应这种动态的坏块特性。
* **磨损均衡 (Wear Leveling)：** NAND Flash 的每个擦除块都有有限的擦写寿命。为了延长寿命，需要通过复杂的算法确保所有块的擦写次数相对均衡。

---

### MTD (Memory Technology Device) 与 `mtdparts`

正因为 NAND Flash 的这些特性，Linux 内核引入了 **MTD (Memory Technology Device)** 子系统来抽象和管理这类非传统的存储设备。MTD 层提供了一套统一的接口，让上层的文件系统（如 JFFS2, UBIFS）和应用程序无需关心底层 NAND Flash 的具体实现细节和坏块管理。

而 **`mtdparts` 命令行参数**正是为了告诉 Linux 内核的 MTD 子系统，它应该如何**逻辑上划分**这块 NAND Flash 存储区域。

* **U-Boot 传递 `mtdparts`：** 在基于 NAND Flash 的嵌入式系统中，U-Boot (或其他第一阶段引导加载程序) 在引导 Linux 内核时，会通过 **`bootargs`** 环境变量将预定义的 `mtdparts` 字符串传递给内核。
* **内核解析 `mtdparts`：** Linux 内核启动后，其 MTD 子系统会解析这个 `mtdparts` 字符串。根据字符串中定义的名称、大小和偏移量，内核会在内部**创建逻辑分区结构**，并在 `/dev/mtdX` 和 `/dev/mtd/名称` 路径下生成相应的设备节点。
* **这是一种“逻辑分区”的定义：** 它不是像 MBR/GPT 那样物理地写在 NAND Flash 上的元数据表，而是一个由引导加载程序（U-Boot）提供给内核的**配置指令**。内核根据这些指令来理解和使用 NAND Flash 上的不同区域。

---

**总结来说：**

对于 **NAND Flash** 设备：

* **没有**标准的 **MBR 或 GPT 分区表**。
* **依赖 U-Boot (或引导加载程序) 传递的 `mtdparts` 参数**来告诉 Linux 内核如何逻辑上划分存储区域。
* Linux 内核通过 **MTD 子系统**来管理 NAND Flash，并根据 `mtdparts` 信息创建逻辑分区设备节点。

这使得 NAND Flash 能够被有效地利用，同时应对其独特的硬件特性。
