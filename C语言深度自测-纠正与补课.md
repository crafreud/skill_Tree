# C 语言深度自测 —— 纠正与补课

> [!quote] 说明
> 以下是对你自测答案的逐题审查。
> ✅ = 正确　⚠️ = 理解有偏差　❌ = 错误/不会
> 每道题后面附上**正确答案 + 原理 + 你需要记住的关键点**。

---

## 一、指针与内存

### 题 1：指针偏移 ⚠️ 你的理解有偏差

```c
int a[] = {1, 2, 3, 4, 5};
int *p = (int *)(&a + 1);
printf("%d %d", *(a+1), *(p-1));
```

**输出：`2 5`**

你说 `*(a+1) = 2`，✅ 正确。

但 `*(p-1)` 你说"p-1 就是 a 的地址"，❌ **不是**。关键在于 `&a` 的类型：

```
a      → 类型 int*，指向 a[0]
&a     → 类型 int(*)[5]，指向整个数组（整个数组的指针）
&a + 1 → 跳过整个数组（5 × 4 = 20 字节），指向 a[5]（数组末尾之后）
(int *)(&a + 1) → 强转为 int*，p 指向 a[5]
p - 1  → 退回一个 int（4 字节），指向 a[4]
*(p-1) = a[4] = 5
```

> [!danger] 关键区别
> - `a + 1`：以 `int` 为单位偏移 1 步（4 字节）
> - `&a + 1`：以 `int[5]` 为单位偏移 1 步（20 字节）
>
> **这是面试高频题。记住：`&数组名` 和 `数组名` 的类型不同。**

---

### 题 2：malloc(0) ⚠️ 不完全对

> `malloc(0)` 返回什么？能用吗？

你说"返回空指针"。❌ **不一定。**

C 标准说这是**实现定义（implementation-defined）**的行为：

| 实现 | 返回值 |
|------|--------|
| Linux glibc | 返回一个**唯一的、可 `free` 的非 NULL 指针** |
| 某些嵌入式 libc | 可能返回 NULL |

```c
void *p = malloc(0);
if (p == NULL) {
    // 可能走到这里（取决于实现）
} else {
    // 也可能走到这里（Linux 下通常走这里）
    // 但 p 指向的内存大小为 0，你不能读写它
    free(p);  // 这是合法的
}
```

> [!tip] 记住
> `malloc(0)` 要么返回 NULL，要么返回一个**不能解引用但可以 free** 的指针。
> **永远不要假设它返回 NULL。** 正确做法：不要写 `malloc(0)`。

---

### 题 3：返回局部变量地址 ✅ 理解正确

```c
void get_str(char **out) {
    char buf[20] = "hello";
    *out = buf;  // 危险！
}
```

你说"函数结束后 buf 会被清除"，✅ 正确。这就是**悬空指针（dangling pointer）**。

**修正方案：**

```c
// 方案 1：调用者分配内存
void get_str(char *out, size_t len) {
    strncpy(out, "hello", len);
}

// 方案 2：返回堆内存（调用者负责 free）
void get_str(char **out) {
    *out = malloc(20);
    strcpy(*out, "hello");
}

// 方案 3：static（有线程安全问题，慎用）
void get_str(char **out) {
    static char buf[20] = "hello";
    *out = buf;
}
```

> [!warning] 更隐蔽的变体
> 面试官可能会问：如果改成 `static char buf[20]` 呢？
> 答：可以，但**不是线程安全的**，而且所有调用者共享同一个 buffer。

---

### 题 4：realloc 三种行为 ❌ 不会 → 补课

```c
void *realloc(void *ptr, size_t new_size);
```

| 情况       | 行为                          | 返回值                    |
| -------- | --------------------------- | ---------------------- |
| **情况 1** | 原地扩展成功（后面有足够空间）             | 返回**原地址**              |
| **情况 2** | 原地扩展失败，分配新空间，拷贝数据，**释放旧空间** | 返回**新地址**              |
| **情况 3** | 分配失败（内存不足）                  | 返回 **NULL**，**旧内存不释放** |

```c
// 错误写法（内存泄漏！）
ptr = realloc(ptr, new_size);  // 如果返回 NULL，原内存泄漏

// 正确写法
void *tmp = realloc(ptr, new_size);
if (tmp == NULL) {
    // 分配失败，ptr 仍然有效，可以继续使用或 free
    handle_error();
} else {
    ptr = tmp;  // 分配成功，更新指针
}
```

> [!danger] 关键陷阱
> `realloc(ptr, 0)` 等价于 `free(ptr)`（C17 之前），这是未定义行为，**不要写**。

---

### 题 5：内存泄漏检测 ❌ 方法不对

你说"用魔数写入内存底部"。这个方法**不是检测内存泄漏的标准方法**，而且在嵌入式中也不实用。

**标准方法：**

| 工具 | 适用场景 | 原理 |
|------|----------|------|
| **Valgrind** | Linux 应用程序 | 拦截 malloc/free，记录所有分配和释放 |
| **AddressSanitizer (ASan)** | GCC/Clang 编译时开启 | 用 `-fsanitize=address` 编译，运行时检测 |
| **MSVC CRT** | Windows 应用 | `_CrtDumpMemoryLeaks()` |
| **嵌入式：自己封装 malloc/free** | MCU 项目 | 用链表记录每次分配，程序结束时检查未释放的 |

**嵌入式中的实用方法：**

```c
// 封装 malloc，记录分配信息
typedef struct {
    void *ptr;
    size_t size;
    const char *file;
    int line;
} alloc_record_t;

#define MALLOC(size) debug_malloc(size, __FILE__, __LINE__)
#define FREE(ptr)    debug_free(ptr, __FILE__, __LINE__)

// 程序运行一段时间后，打印所有未释放的分配
void dump_leaks(void);
```

> [!tip] 建议
> 在嵌入式中，最实用的做法是：**尽量避免动态分配**。用静态内存池、固定大小数组。
> 如果必须用 malloc，就封装一层，记录每次分配。

---

### 题 6：栈溢出 ❌ 不会 → 补课

**栈溢出的典型症状：**

| 症状        | 说明                 |
| --------- | ------------------ |
| 变量值莫名被改   | 局部变量被相邻函数的栈帧覆盖     |
| HardFault | 栈溢出到非法内存区域         |
| 函数返回地址被篡改 | 栈上的返回地址被覆盖，跳转到随机位置 |
| 程序行为"随机"  | 每次运行结果不同，因为栈布局可能不同 |

**如何用 `-fstack-protector` 检测：**

```bash
gcc -fstack-protector-strong -o app main.c
```

原理：编译器在函数的栈帧中插入一个 **canary 值**（哨兵值），函数返回前检查 canary 是否被篡改。如果被篡改，说明栈溢出，程序终止并报错。

```
┌──────────────────┐
│  局部变量         │
│  canary (哨兵值)  │ ← 编译器插入的
│  保存的寄存器     │
│  返回地址         │
└──────────────────┘
```

**嵌入式中的栈溢出检测：**

```c
// 方法 1：在任务栈底部填充特殊值（如 0xA5A5A5A5）
// 定期检查这个值是否被覆盖
#define STACK_FILL_MAGIC  0xA5A5A5A5

void check_stack_usage(uint32_t *stack_bottom, size_t stack_size) {
    for (size_t i = 0; i < stack_size / 4; i++) {
        if (stack_bottom[i] != STACK_FILL_MAGIC) {
            // 栈被使用到这里了
            printf("Stack used up to offset %d\n", i * 4);
            break;
        }
    }
}

// 方法 2：FreeRTOS 内置的 uxTaskGetStackHighWaterMark()
UBaseType_t watermark = uxTaskGetStackHighWaterMark(NULL);
// 返回值是栈空间从未被使用的最小值（单位：word）
```

---

## 二、编译与链接

### 题 7：static 的作用 ⚠️ 理解有偏差

你说"函数中 static 变量存储在堆中"。❌ **错误！**

| 作用域 | 声明位置 | 作用 | 存储位置 |
|--------|----------|------|----------|
| **函数内** | `void foo() { static int x; }` | 变量生命周期变为整个程序运行期间，但只在函数内可见 | **BSS/数据段**（不是堆！） |
| **文件内（全局）** | `static int g_count;` | 变量只在当前 `.c` 文件内可见，其他文件无法访问 | **BSS/数据段** |
| **头文件中的函数** | `static void helper();` | 函数只在包含该头文件的 `.c` 文件内可见（每个文件一份独立副本） | 代码段 |

> [!danger] 关键纠正
> **static 变量不在堆上！** 它们和全局变量一样，在 BSS 段（未初始化）或数据段（已初始化）。
> 堆是 `malloc`/`free` 管理的区域，需要手动分配和释放。
> static 变量是**自动分配、永不释放、地址固定**的。

**C 程序内存布局：**

```
高地址 ┌─────────────┐
       │   栈 (Stack)  │ ← 局部变量、函数调用帧
       │      ↓        │
       │      ...       │
       │      ↑        │
       │   堆 (Heap)   │ ← malloc/free 管理
       ├─────────────┤
       │   .bss        │ ← 未初始化的全局/static 变量（自动清零）
       │   .data       │ ← 已初始化的全局/static 变量
       │   .rodata     │ ← const 常量、字符串字面量
       │   .text       │ ← 代码（指令）
低地址 └─────────────┘
```

---

### 题 8：extern "C" ⚠️ 理解偏了

你说"调用 C 标准库时需要使用"。❌ **不是这个用途。**

**真正用途：**

`extern "C"` 是 **C++ 中**用来告诉编译器"用 C 的方式编译这个函数"。

```c
// 为什么需要？
// C++ 有函数重载，编译器会对函数名做 name mangling（名称修饰）
// void foo(int)  → 编译后可能变成 _Z3fooi
// void foo(double) → 编译后可能变成 _Z3food

// C 没有重载，不做 name mangling
// void foo(int) → 编译后就是 foo

// 如果你的 C++ 代码要调用 C 编译的库：
extern "C" {
    void my_c_function(int x);  // 告诉 C++ 编译器：不要 mangling，用 C 的名字
}
```

**什么时候必须用？**

```c
// 场景：你写了一个 C 库，想让 C++ 代码也能调用

// mylib.h
#ifdef __cplusplus
extern "C" {
#endif

void my_function(int x);  // C 和 C++ 都能用

#ifdef __cplusplus
}
#endif
```

> [!tip] 嵌入式中什么时候用？
> 如果你的项目是纯 C，**不需要**。
> 如果你用 C++ 写应用层但底层驱动是 C 的，**需要**。
> 如果你对接的第三方库是 C 写的（如 MQTT 库、JSON 库），**需要**。

---

### 题 9：.text / .data / .bss / .rodata ⚠️ 只记得一部分

| 段 | 存什么 | 特点 |
|----|--------|------|
| **.text** | 代码指令 | ✅ 你说对了。只读，存在 Flash 中 |
| **.data** | **已初始化**的全局变量和 static 变量 | 需要初值，存在 Flash 中，启动时拷贝到 RAM |
| **.bss** | **未初始化**的全局变量和 static 变量 | ❌ 你忘了。启动时自动清零，**不占 Flash 空间** |
| **.rodata** | 只读数据：`const` 变量、字符串字面量 | 存在 Flash 中 |

```c
int global_init = 10;      // .data（已初始化）
int global_uninit;          // .bss（未初始化，自动为 0）
static int static_var = 5;  // .data
static int static_zero;     // .bss
const int table[] = {1,2,3}; // .rodata
char *msg = "hello";        // msg 在 .data，"hello" 在 .rodata
```

> [!tip] 嵌入式中为什么要关注这个？
> 因为 MCU 的 RAM 有限。你需要知道：
> - `.data` + `.bss` 的大小 = 你的静态 RAM 占用
> - 栈的大小 = 你的局部变量和函数调用开销
> - 堆的大小 = 你的 malloc 使用量
>
> 用 `arm-none-eabi-size your_app.elf` 可以查看各段大小。

---

### 题 10：弱符号 ✅ 理解正确

```c
__attribute__((weak)) void my_callback(void) {
    // 默认实现（弱符号）
    // 用户可以在其他文件中定义同名函数（强符号）来覆盖
}
```

**嵌入式典型用途：**

```c
// 默认的系统初始化（弱符号）
__attribute__((weak)) void SystemInit(void) {
    // 默认什么都不做
}

// 用户可以在自己的代码中覆盖
void SystemInit(void) {
    // 自定义初始化：配置时钟、电源等
}
```

---

### 题 11：编译四个阶段 ❌ 顺序和内容都错了

| 阶段 | 命令 | 输入 | 输出 | 做什么 |
|------|------|------|------|--------|
| **1. 预处理** | `gcc -E` | `.c` | `.i` | 展开宏、处理 `#include`、删除注释 |
| **2. 编译** | `gcc -S` | `.i` | `.s` | 语法分析、语义分析、生成汇编代码 |
| **3. 汇编** | `gcc -c` | `.s` | `.o` | 将汇编翻译成机器码（目标文件） |
| **4. 链接** | `gcc` | `.o` + 库 | `.elf`/可执行文件 | 合并目标文件、解析符号引用、分配地址 |

```
hello.c → [预处理] → hello.i → [编译] → hello.s → [汇编] → hello.o → [链接] → hello.elf
```

> [!tip] 嵌入式中为什么要理解这个？
> - 编译报错"undefined reference" → 链接阶段的问题（函数声明了但没定义）
> - 编译报错"multiple definition" → 链接阶段的问题（头文件中定义了全局变量）
> - 想看预处理结果：`gcc -E main.c > main.i`（排查宏展开问题）

---

### 题 12：链接脚本 ❌ 不会 → 补课

**链接脚本（`.ld`）**告诉链接器：把代码和数据放在内存的什么位置。

嵌入式中**必须用链接脚本**，因为你需要精确控制：

```ld
/* STM32F103C8T6 的链接脚本（简化版） */
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 64K  /* Flash 起始地址和大小 */
    RAM (xrw)  : ORIGIN = 0x20000000, LENGTH = 20K  /* RAM 起始地址和大小 */
}

SECTIONS
{
    .text :           /* 代码段 */
    {
        *(.text)      /* 所有 .o 文件的 .text 段 */
    } > FLASH         /* 放在 Flash 中 */

    .data :           /* 已初始化数据 */
    {
        *(.data)
    } > RAM AT> FLASH /* 运行在 RAM，但存储在 Flash（启动时拷贝） */

    .bss :            /* 未初始化数据 */
    {
        *(.bss)
    } > RAM           /* 放在 RAM 中，启动时清零 */
}
```

> [!tip] 你需要理解的关键点
> - Flash 地址 `0x08000000` 是 STM32 的代码存储起始地址
> - RAM 地址 `0x20000000` 是 STM32 的 RAM 起始地址
> - `.data` 段的 `AT> FLASH` 表示"加载地址在 Flash，运行地址在 RAM"
> - 启动代码（`startup_stm32f103.s`）负责把 `.data` 从 Flash 拷贝到 RAM，把 `.bss` 清零

---

## 三、宏与预处理

### 题 13：MAX 宏的陷阱 ⚠️ 知道有坑但没说清楚

```c
#define MAX(a,b) ((a)>(b)?(a):(b))
```

**陷阱：参数被求值多次！**

```c
int x = 5, y = 3;
int z = MAX(x++, y++);
// 展开为：((x++)>(y++)?(x++):(y++))
// x++ 求值了 2 次！x 从 5 变成 7，不是 6

z = MAX(get_sensor_value(), 0);
// 如果 get_sensor_value() 有副作用（如改变硬件状态），会被调用 2 次！
```

> [!danger] 这就是为什么需要 SAFE_MAX
> 用 `typeof` 和语句表达式确保每个参数只求值一次。

---

### 题 14：SAFE_MAX 的作用 ⚠️ 说了但不够精确

```c
#define SAFE_MAX(a,b) ({typeof(a) _a=(a); typeof(b) _b=(b); _a>_b?_a:_b;})
```

你说"保证内部函数不会被外面的内容影响"。❌ 不准确。

**正确解释：**

- `typeof(a) _a = (a)`：先将参数求值一次，存入临时变量
- 后续只使用临时变量 `_a` 和 `_b`
- `({...})` 是 GCC 的**语句表达式**（statement expression），整个块的值是最后一个表达式

**效果：每个参数只被求值一次。**

```c
int z = SAFE_MAX(x++, y++);
// 展开为：({int _a=(x++); int _b=(y++); _a>_b?_a:_b;})
// x++ 只执行 1 次，y++ 只执行 1 次 ✅
```

---

### 题 15：#pragma once vs #ifndef ✅ 基本正确

|       | `#pragma once`       | `#ifndef`      |
| ----- | -------------------- | -------------- |
| 原理    | 编译器保证只包含一次           | 通过宏名判断是否已包含    |
| 可移植性  | 大多数编译器支持，但非标准        | C/C++ 标准，完全可移植 |
| 缺点    | 同一文件有多个副本时可能失效       | 宏名冲突时可能失效      |
| 嵌入式建议 | ✅ 推荐用 `#pragma once` | 也行，但要注意宏名唯一性   |

你说"嵌入式模块复用多用 `#ifndef`"，✅ 合理。

---

### 题 16：packed 和 aligned ❌ 不清楚 → 补课

```c
// packed：取消内存对齐，按 1 字节对齐
struct __attribute__((packed)) sensor_data {
    uint8_t  id;       // 1 字节
    uint32_t value;    // 4 字节（不会填充 3 字节空隙）
    uint8_t  status;   // 1 字节
};
// sizeof = 6（没有 padding）

// 普通结构体（默认对齐）
struct sensor_data_normal {
    uint8_t  id;       // 1 字节 + 3 字节 padding
    uint32_t value;    // 4 字节
    uint8_t  status;   // 1 字节 + 3 字节 padding
};
// sizeof = 12（有 padding）
```

**什么时候用 packed？**

```c
// 场景 1：通信协议的数据包（必须严格对应字节流）
// 场景 2：硬件寄存器映射（必须和硬件地址对应）
// 场景 3：Flash 存储结构（节省空间）
```

**什么时候用 aligned？**

```c
// 强制对齐到 4 字节边界（提高访问速度）
struct __attribute__((aligned(4))) fast_data {
    uint8_t a;
    uint8_t b;
};
// sizeof = 4（不是 2，因为对齐到了 4 字节边界）
```

> [!warning] packed 的代价
> - 访问未对齐的变量可能**更慢**（有些架构甚至会触发异常）
> - 取 packed 结构体成员的地址时，编译器会生成**更复杂的代码**
> - **只在必须严格控制布局时使用**（协议解析、硬件寄存器）

---

## 四、你需要重点补的盲区总结

| 盲区                 | 优先级    | 建议学习方式                                           |
| ------------------ | ------ | ------------------------------------------------ |
| **realloc 三种行为**   | 🔥 高   | 读 man realloc，写测试代码验证                            |
| **栈溢出检测**          | 🔥 高   | 用 FreeRTOS 的 `uxTaskGetStackHighWaterMark` 实际测一下 |
| **.bss 段**         | 🔥 高   | 用 `arm-none-eabi-size` 查看你的固件各段大小                |
| **链接脚本**           | 🔥🔥 高 | 读你项目的 `.ld` 文件，理解每一行的含义                          |
| **packed/aligned** | 🔥🔥 高 | 写代码对比 packed 和非 packed 的 sizeof                  |
| **编译四阶段**          | 🔥 中   | 用 `gcc -E/-S/-c` 实际走一遍                           |
| **extern "C"**     | 🔥 中   | 写一个 C++ 调用 C 函数的例子                               |
| **malloc(0)**      | ⚡ 低    | 了解即可，实际项目不要写                                     |
| **static 存储位置**    | 🔥🔥 高 | 纠正"堆"的错误概念，记住在 BSS/数据段                           |
| **SAFE_MAX 原理**    | 🔥 中   | 理解语句表达式 `({...})` 的用法                            |

---

## 五、补充自测题（你没覆盖到的重要知识点）

> [!note] 这些知识点同样高频出现在面试和实际开发中

### 5.1 volatile 和 const

- [ ] `volatile` 是什么意思？为什么嵌入式中必须用？不用会怎样？这个提醒编译器在读取这个地址的时候不去优化 因为嵌入式硬件可能会操作这个指针指向的内容 不用的话可能会导致死循环
  ```c
  // 以下代码有什么问题？
  uint32_t *reg = (uint32_t *)0x40021000;
  while (*reg == 0) {}  // 编译器可能优化成死循环
  ```

- [ ] `const volatile` 是什么？什么时候用？硬件寄存器在硬件层面已经定死了 所以这个修饰的地址本身不会被修改
  ```c
  // 为什么硬件寄存器经常是 const volatile？
  const volatile uint32_t *reg = (uint32_t *)0x40021000;
  ```

### 5.2 结构体与联合体

- [ ] 以下代码输出什么？共用体word 输出 0x12 0x34 以此类推，这是大端储存方式 高地址数据在前 
  ```c
  union {
      uint32_t word;
      struct {
          uint8_t byte0;
          uint8_t byte1;
          uint8_t byte2;
          uint8_t byte3;
      } bytes;
  } data = {0x12345678};
  printf("%02X %02X %02X %02X",
      data.bytes.byte0, data.bytes.byte1,
      data.bytes.byte2, data.bytes.byte3);
  ```
  （答案取决于大小端）

- [ ] 什么是位域（bitfield）？以下结构体的布局你知道吗？ 在一个u8中 分别占用 1 2 5位可以省空间
  ```c
  struct flags {
      uint8_t enable  : 1;
      uint8_t mode    : 2;
      uint8_t level   : 5;
  };
  ```

### 5.3 函数指针进阶

- [ ] 看懂这个函数指针数组（回调表）： 使用现在的状态去触发不同的函数逻辑
  ```c
  typedef void (*handler_t)(void);
  handler_t handlers[] = {handle_idle, handle_run, handle_stop};
  handlers[current_state]();  // 状态机
  ```

### 5.4 中断相关

- [ ] 中断服务函数（ISR）中**不能做**什么？为什么？
  - 不能 `malloc` 这个我不是很清楚
  - 不能 `printf` printf可能会操作底层 进入到其他函数中好像是
  - 不能阻塞等待 防止程序卡死？
  - 不能用浮点（除非保存 FPU 上下文）浮点数占用栈空间很大？

- [ ] 什么是中断上下文和任务上下文？区别是什么？ 中断上下文指的是在中断发生时 将现在的程序状态和当前指针压入栈中 在中断结束后根据栈来还原程序场景 任务上下文指的是任务之间切换时根据程序块去保存任务状态和互斥量什么的 这个我不太清楚

---

> [!abstract] 行动建议
> 1. 先把上面标记为 🔥🔥 的盲区补完
> 2. 用实际代码验证每一道题（不要只看答案，要跑一遍）
> 3. 把这些知识点写进你的 skill_Tree，用自己的话复述一遍
> 4. 补完后再进入第二层（工程能力）