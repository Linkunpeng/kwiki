# DKMS

When you update your kernel to a new version, apt or other package managers will trigger DKMS to update out-of-tree modules.


## When updating the Linux kernel, modules must be updated for several reasons

- The kernel ABI may change even if the API remains identical, as the kernel update can alter binary compatibility.
- Since the kernel is written in C, modifying data structures in kernel headers can change their memory layout.
- In such cases, passing function parameters with these structs may yield unexpected results due to layout mismatches.


## 内核头文件的变化可能导致ABI发生变化
- 比如在内核数据结构中增加或减少成员变量， 会导致内存排布发生变化。
- 外部模块对成员的索引偏移可能不一样。所以在内核更新之后需要及时重新编译外部驱动
