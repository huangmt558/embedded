# C 指针在嵌入式中的应用

[返回目录](../README.md)

**指针变量保存对象或函数的地址。**通过解引用可以间接访问对象，通过函数指针可以调用函数。对象包括普通变量、数组和结构体等。[C 标准草案：§6.2.5、§6.3.2.3][c11]

```c
int value = 10;
int *p = &value;  /* p 保存 value 的地址 */
/* 在函数中执行 *p = 20，会修改原来的 value。 */
```

下面按五种常见应用整理：寄存器访问、数组与缓冲区、结构体传参、动态内存、函数指针与回调。

## 1. 访问硬件寄存器

MCU 把外设寄存器映射到地址空间中。程序可以把寄存器的整数地址转换成相应指针，再解引用访问。

以 F103 的 GPIOA 输入数据寄存器为例，在外设时钟等条件已经准备好时：

```c
#include <stdint.h>

uint32_t read_gpioa_input(void)
{
    const volatile uint32_t *idr =
        (const volatile uint32_t *)0x40010808UL;
    return *idr;
}
```

| 写法 | 含义 |
| --- | --- |
| `uint32_t *` | 通过指向 32 位无符号对象的指针访问 |
| `volatile` | 要求保留相应访问，不能把外部可能改变的对象当作普通稳定变量处理 |
| `const` | 不允许通过这个指针修改所指对象，硬件仍可改变其值 |
| `*idr` | 解引用，访问对应对象 |

这里 `const` 和 `volatile` 修饰的是**所指对象的访问类型**，不是把地址本身“变成 volatile”。整数到指针的转换依赖平台；地址、位宽和权限应匹配实际硬件。[ST 寄存器定义][stm32-header]、[CMSIS 外设访问约定][cmsis]

<details>
<summary>volatile 能不能保证原子性或线程安全？</summary>

不能。`volatile` 处理的是编译器对相应访问的保留要求，不会自动给“读取、修改、写回”等多步操作加锁，也不自动解决任务、中断或 DMA 之间的同步。

是否需要临界区、原子操作或其他同步手段，要根据具体共享访问方式判断。

</details>

## 2. 操作数组和缓冲区

指针可以配合长度访问连续数据，用于遍历、读写和解析缓冲区；读写指针或索引还可以记录数据处理到了哪里。

```c
#include <stddef.h>
#include <stdint.h>

uint32_t sum_bytes(const uint8_t *buffer, size_t length)
{
    uint32_t sum = 0;
    for (size_t i = 0; i < length; ++i) {
        sum += buffer[i];
    }
    return sum;
}
```

调用者提供有效缓冲区和长度，函数通过地址访问原数据，无需复制整个数组。字符串传参也经常使用指针，接口需要约定长度或以空字符结尾。

**数组本身不是指针。**数组在大多数表达式中会转换为指向首元素的指针；`p++` 前进一个元素，字节步长取决于所指类型。[C 标准草案：§6.3.2.1、§6.5.6][c11]

<details>
<summary>为什么传入函数后，不能用 sizeof 求原数组长度？</summary>

函数形参写成 `uint8_t buffer[]` 时，会调整为指针类型。因此，函数内的 `sizeof(buffer)` 得到的是指针大小，不是调用者数组的容量。

在实际数组定义所在位置，`sizeof(array)` 才能得到该数组所占的总字节数。常见接口同时传入地址和长度。[C 标准草案：§6.7.6.3][c11]

</details>

## 3. 通过结构体指针传参

传递结构体地址，可以避免复制整个结构体，并允许函数修改原对象；只读接口可以使用指向 const 对象的指针。

```c
typedef struct {
    int value;
} Device;

void update_device(Device *dev)
{
    dev->value = 1;
}
```

调用时传入有效的 Device 对象地址。`dev->value` 等价于 `(*dev).value`。

对于大型结构体或其中的数组，这种方式可以减少参数复制及相关的栈空间开销。**C 仍然是按值传递：复制的是指针值，不是整个结构体。**

## 4. 动态分配内存

`malloc()` 等函数返回指向已分配空间的指针，程序通过它使用这片内存，使用完后调用 `free()`。[C 标准草案：§7.22.3][c11]

下面片段放在函数中使用，并包含 `<stdlib.h>`、`<stdint.h>`：

```c
uint8_t *buffer = malloc(128);
if (buffer != NULL) {
    /* 使用这 128 字节空间 */
    free(buffer);
    buffer = NULL;
}
```

分配失败时返回空指针。释放后不能继续访问原空间；设置一个指针为 NULL，也不会自动清除其他指向同一空间的指针。

## 5. 函数指针与回调

```c
int (*fun)(int);
```

这表示：**fun 是一个指针变量，可以指向“接收一个 int 参数、返回 int”的函数。**声明本身不会执行函数，调用前要让它指向类型匹配的有效函数。

```c
int add_one(int x)
{
    return x + 1;
}

int apply(int value, int (*callback)(int))
{
    return callback(value);
}
```

在调用处执行 `apply(3, add_one)`，结果是 4。这里把函数地址交给 `apply`，由它通过 `callback` 调用，这就是回调的一种用法。

也可以先令 `int (*fun)(int) = add_one;`，再用 `fun(3)` 调用。**函数指针是工具，回调是使用方式**；嵌入式中常用于事件通知、驱动接口和异步完成处理。

## 6. 易错点

| 容易混淆 | 正确理解 |
| --- | --- |
| 指针变量就是所指对象 | 它保存地址，目标对象有自己的存储空间 |
| volatile 修饰“地址” | 这里修饰的是通过指针访问的对象类型 |
| 数组与指针完全相同 | 类型、存储和 sizeof 行为不同 |
| 指针加 1 总是移动 1 字节 | 移动一个所指类型元素的大小 |
| 传指针就是 C 的引用传递 | C 按值传递，传递的是地址的副本 |
| `int (*fun)(int)` 是函数本体 | 它声明函数指针；`int *fun(int)` 才是返回 int 指针的函数声明 |
| 函数指针和函数返回地址一样 | 前者指定要调用谁，后者指定调用完成后回到哪里 |

函数的机器指令在普通 MCU 工程中通常位于 Flash；函数地址这个数值，可以存放在 Flash 向量表、RAM 中的函数指针变量或 CPU 寄存器中。[内存分布笔记](stm32-startup-memory.md)

## 7. 口述版本

指针变量保存对象或函数的地址。在嵌入式中，常用指针访问映射到地址空间的硬件寄存器，并通过 volatile 限定相应访问。

指针还用于数组和缓冲区处理、结构体传参、动态内存，以及函数指针和回调。通过传递地址，可以访问原数据并减少大型对象的复制；但 C 仍然按值传递，指针类型和使用范围也必须与目标对象匹配。

## 参考资料

- [C11 委员会草案 N1570][c11]：§6.2.5 指针类型、§6.3.2 转换、§6.7.3 限定符、§6.7.6.3 形参、§7.22.3 内存分配。
- [ST STM32F103xB 头文件][stm32-header]：GPIO 寄存器结构、基地址与访问宏。
- [Arm CMSIS：Peripheral Access][cmsis]：外设寄存器的类型与访问限定符。

[c11]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf
[stm32-header]: https://github.com/STMicroelectronics/cmsis-device-f1/blob/master/Include/stm32f103xb.h
[cmsis]: https://arm-software.github.io/CMSIS_6/v6.3.0/Core/group__peripheral__gr.html
