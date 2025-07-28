# 内核数据结构

## offsetof && container_of 

>在C语言中，`container_of`是一个非常实用的宏，它允许你通过结构体成员的指针来获取整个结构体的指针。这个宏在Linux内核中被广泛使用，特别是在实现链表和其他数据结构时。
> 通常用于从list_head 中获取到完整的数据结构， 由于是宏操作并且是在编译器进行计算，所以非常高效。
> offsetof 获取成员在结构体中的偏移量, 辅助实现container_of

### 宏的定义
`container_of`宏的基本定义如下：

```c
#define container_of(ptr, type, member) ({                      \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})

// offsetof宏的定义
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

### 工作原理
这个宏的工作原理基于以下几点：
1. **类型检查**：通过`typeof`获取成员的类型，确保类型安全。
2. **零指针技巧**：将`0`转换为结构体指针，然后获取成员的地址，这样得到的地址就是成员在结构体中的偏移量。
3. **指针运算**：用成员的实际地址减去偏移量，得到结构体的起始地址。

### 使用示例
下面是一个简单的例子，展示了如何使用`container_of`：

```c
#include <stdio.h>
#include <stddef.h>

// 定义一个结构体
struct person {
    char name[20];
    int age;
    int height;
};

int main() {
    // 创建一个person实例
    struct person p = {"Alice", 30, 170};
    
    // 获取age成员的指针
    int *age_ptr = &p.age;
    
    // 使用container_of获取结构体指针
    struct person *p_ptr = container_of(age_ptr, struct person, age);
    
    // 通过结构体指针访问其他成员
    printf("Name: %s\n", p_ptr->name);
    printf("Height: %d\n", p_ptr->height);
    
    return 0;
}
```

### 注意事项
1. **确保指针有效**：传入的指针必须是有效的，否则结果将不可预测。
2. **类型匹配**：`type`和`member`必须匹配，否则会导致类型错误。
3. **结构体对齐**：由于编译器可能会对结构体成员进行对齐，因此偏移量可能不是连续的，但`container_of`会正确处理这种情况。

通过使用`container_of`，你可以在内核编程或其他需要高效数据结构的场景中，方便地从成员指针获取整个结构体的指针，这在实现链表、树等数据结构时特别有用。


## list_head

在设计上，内核数据结构 `list`（特指 Linux 内核中的 `struct list_head`）与普通的链表（例如你在用户空间编写的、包含数据字段的 `Node` 结构体）有显著不同。这两种方式在将 `list` 写入 `node` 的 `struct` 和将 `node` 写入 `list` `struct` 上也有不同的应用场景。

-----

### 内核 `list` 与普通 `list` 的设计差异

#### 普通 `list` 的设计

在 C/C++ 等语言中，我们通常会这样设计一个普通的双向链表节点：

```c
struct Node {
    int data; // 存储实际数据
    struct Node *next; // 指向下一个节点
    struct Node *prev; // 指向前一个节点
};

// 链表头（可选，也可以直接用 Node* head 表示）
struct List {
    struct Node *head;
    // 其他链表相关的元数据，比如大小等
};
```

这种设计中，`Node` 结构体直接包含了实际的数据和链表指针。

#### Linux 内核 `list` 的设计

Linux 内核的 `list` (即 `struct list_head`) 的设计哲学是 **侵入式 (Intrusive List)**。它不包含任何数据字段，只包含指向下一个和前一个元素的指针：

```c
struct list_head {
    struct list_head *next;
    struct list_head *prev;
};
```

那么数据在哪里呢？`struct list_head` 会被**嵌入**到实际数据结构中。例如，如果你有一个表示进程的 `task_struct`：

```c
struct task_struct {
    // ... 很多其他进程相关的数据 ...
    struct list_head tasks_list; // 嵌入的链表节点
    // ...
};
```

**主要差异总结：**

1.  **数据分离与否：**

      * **普通 `list`:** `Node` 结构体通常直接包含数据。
      * **内核 `list`:** `struct list_head` **不包含**任何数据，它只是一个通用的链表节点，实际数据存在于包含 `list_head` 的结构体中。

2.  **通用性与复用：**

      * **普通 `list`:** 如果你有多种类型的数据需要链表管理，你可能需要为每种数据类型定义一个 `Node` 结构体，或者使用 `void*` 来存储数据，这会引入类型转换和潜在的类型安全问题。
      * **内核 `list`:** `struct list_head` 是完全通用的。任何结构体，只要它嵌入了 `struct list_head`，就可以被放入内核的 `list` 中。通过 `container_of` 宏，可以从 `list_head` 指针反向推导出包含它的数据结构体的地址，从而访问实际数据。这大大提高了代码的复用性和泛用性。

3.  **内存布局与缓存：**

      * **普通 `list`:** `Node` 结构体通常是动态分配的，每个节点除了数据外还有额外的指针开销。如果数据本身较小，指针的开销可能相对较大。
      * **内核 `list`:** 链表节点直接内嵌在数据结构中，这使得数据和链表指针在内存中通常是连续的，有利于**缓存局部性**。访问一个链表节点通常也意味着访问了它所在的整个数据结构，减少了额外的内存查找。

4.  **多链表归属：**

      * **普通 `list`:** 一个数据项通常只能属于一个链表，因为它的 `next`/`prev` 指针是固定的。
      * **内核 `list`:** 一个数据结构可以嵌入多个 `struct list_head` 成员，从而同时属于多个不同的链表。例如，一个 `task_struct` 可以同时在一个“运行队列”链表中，又在一个“等待事件”链表中。

-----

### 两种方式的应用场景

#### 1\. 将 `list` 写入 `node` 的 `struct` (即 `struct list_head` 嵌入到数据结构中)

这种方式是 Linux 内核链表的设计方式。

  * **设计示例:**

    ```c
    struct my_data {
        int id;
        char name[32];
        struct list_head list_node; // 链表节点嵌入到数据结构中
    };

    // 链表头
    LIST_HEAD(my_global_list); // 声明并初始化一个 struct list_head 作为链表头
    ```

    当你创建一个 `my_data` 实例时，它的 `list_node` 成员就是这个数据结构在链表中的代表。

  * **应用场景:**

    1.  **系统级数据管理：** Linux 内核大量使用这种方式来管理各种系统资源，例如：
          * **进程列表 (tasks):** 所有 `task_struct` 通过 `tasks_list` 成员连接成一个双向链表。
          * **文件描述符 (file descriptors):** 每个打开的文件可能都有一个 `f_list` 成员，将其加入到进程的文件列表中。
          * **内存管理 (pages, free lists):** 内存页、空闲页等都可以通过嵌入 `list_head` 进行管理。
          * **设备驱动 (devices, drivers):** 设备和驱动程序的管理也常用链表。
    2.  **需要数据多重归属的场景：** 当一个数据对象可能同时存在于多个逻辑分组中时，这种方式非常高效。例如，一个任务可能既在CPU的运行队列中，又在等待某个锁的队列中。
    3.  **内存效率和缓存局部性要求高：** 在性能敏感的底层系统（如操作系统内核）中，减少内存碎片和提高缓存命中率至关重要。将链表节点与数据放在一起，有助于实现这一点。
    4.  **避免动态内存分配开销：** 如果数据结构是预先分配好的或者在栈上分配，那么链表节点就无需额外分配内存，减少了 `malloc`/`free` 的开销。

#### 2\. 将 `node` 写入 `list` `struct` (普通链表设计，链表节点包含数据)

这种方式是更常见的、我们在用户空间应用程序中通常使用的链表实现。

  * **设计示例:**

    ```c
    struct my_data_node {
        int data_id;
        char data_name[32];
        struct my_data_node *next;
        struct my_data_node *prev;
    };

    struct my_list_container {
        struct my_data_node *head;
        struct my_data_node *tail; // 或者其他链表头信息
        int size;
    };
    ```

  * **应用场景:**

    1.  **用户空间应用程序：** 在大多数用户空间的应用程序中，这种设计更直观、易于理解和实现。数据和其在链表中的“位置”被封装在同一个节点中。
    2.  **数据类型固定且不需多重归属：** 如果你的链表只存储一种特定类型的数据，并且一个数据项不需要同时出现在多个不同的链表中，这种方式简单明了。
    3.  **封装性要求较高：** 这种设计将链表的操作和节点本身封装得更好，用户可以直接操作 `my_data_node` 来获取数据和遍历。
    4.  **抽象和接口设计：** 在面向对象编程中，通常会把 `Node` 作为内部私有结构，而对外暴露一个 `List` 类，提供 `add`, `remove` 等方法，这种设计更符合这种封装要求。
    5.  **不追求极致的性能优化：** 相对于内核链表，其在缓存局部性上可能稍逊一筹，但对于大多数用户空间应用而言，这种差异通常可以忽略。

-----

### 总结

Linux 内核的 `struct list_head` 设计是一种高度优化、通用且侵入式的链表实现，旨在满足内核级别对性能、内存效率和多重归属的严格要求。它通过将链表节点嵌入到数据结构中，并提供 `container_of` 宏来实现数据的访问。

而普通链表的设计则更为直观和封装，适用于大多数用户空间应用程序，当对性能要求不是极致，且数据类型相对固定或不需多重归属时，是更简洁的选择。

理解这两种设计思想以及它们各自的应用场景，对于选择合适的链表实现方式至关重要。
