# STM32 启动流程与内存分布

[返回目录](../README.md)

**先由硬件确定从哪里启动、装入 MSP 和复位入口，再由启动代码准备系统及 C 运行环境，最后进入 `main()`。**把硬件动作和软件动作分开，启动流程就容易理解了。

本文主要以 **STM32F103C8T6 从内部主 Flash 启动**为例。

## 1. 从上电到 main

```text
上电，内部复位电路使芯片保持复位
    ↓ 电源达到规定条件，并经过复位延时
释放复位，硬件采样 BOOT 引脚，确定启动存储器
    ↓
建立启动映射，读取初始 MSP 和 Reset_Handler 入口
    ↓
执行 Reset_Handler，调用 SystemInit
    ↓
准备 C 运行环境：复制 .data、清零 .bss、进行运行库初始化
    ↓
调用用户编写的 main()
```

BOOT 选择发生在读取初始 MSP 之前；主 Flash 启动时，向量表及启动代码通常都保存在 Flash 中。[启动配置][rm0008]、[ST F1 启动文件][startup]

<details>
<summary>SystemInit 和 C 运行环境初始化分别做什么？</summary>

`SystemInit()` 执行底层系统初始化，具体内容由芯片和库版本决定。常见 F1 标准库工程会在这里配置时钟源、PLL、总线分频和 Flash 等待周期；也可能配置向量表位置。

STM32Cube HAL 工程的主时钟配置通常放在 `main()` 中的 `SystemClock_Config()`。不能把“PLL 已配置完成”视为进入 `main()` 前的通用前提，应检查自己工程的源码。[系统初始化模板][system-init]

C 运行环境初始化包括把 `.data` 的初值复制到 RAM、将 `.bss` 清零，以及工具链需要的其他初始化。C++ 工程还可能运行静态对象构造函数。

不同工具链的实现位置有所不同。例如，Keil 启动文件常进入运行库的 `__main`，再调用用户的 `main()`；两者不是同一个入口。[Keil 启动文件][startup-keil]

</details>

## 2. 零地址和 Flash 地址是什么关系

F103 主 Flash 的固定基地址是 **`0x08000000`**。从主 Flash 启动时，硬件还让零地址成为这片 Flash 的别名：

```text
CPU 访问 0x00000000 ──启动映射──┐
                              ├──→ 同一片 Flash 的开头
CPU 访问 0x08000000 ──固定访问──┘
```

| 启动别名地址 | Flash 固定地址 | 保存的内容及用途 |
| --- | --- | --- |
| `0x00000000` | `0x08000000` | 初始栈顶地址，硬件装入 MSP |
| `0x00000004` | `0x08000004` | 复位处理函数入口，硬件据此开始执行 |
| `0x00000008` | `0x08000008` | NMI 处理函数入口 |

每项占 4 字节。向量表保存的是数值和入口地址，函数指令在入口地址指向的位置。**映射没有复制数据，两组地址访问的是同一份内容。**[向量表说明][pm0056]

<details>
<summary>两个地址之间是什么？BOOT 又能选择哪些启动方式？</summary>

地址空间是 CPU 可以发出的地址编号，不等于芯片实际安装的存储容量。零地址附近提供启动别名访问，之后大量区域是保留或未实现地址，不能当成额外 RAM 或 Flash。

F103C8T6 的标称主 Flash 为 64 KB，固定地址范围是 `0x08000000～0x0800FFFF`；SRAM 为 20 KB，范围是 `0x20000000～0x20004FFF`。[数据手册][f103c8]

F103 的启动选择如下：

| BOOT0 | BOOT1 | 启动存储器 |
| --- | --- | --- |
| 0 | 任意 | 主 Flash |
| 1 | 0 | 系统存储器 |
| 1 | 1 | SRAM |

主 Flash 和系统存储器启动时，零地址映射到对应存储器。SRAM 启动使用 F103 的专门启动机制，SRAM 仍通过 `0x20000000` 访问，需要按参考手册设置运行时向量表，不能直接照搬上面的 Flash 别名图。[RM0008：Boot configuration][rm0008]

ST 出厂 Bootloader 位于系统存储器；F103C8T6 对应区域从 `0x1FFFF000` 开始。自己编写的 Bootloader 通常放在用户 Flash 中。直接从主 Flash 启动普通应用时，不必先执行 ST 出厂 Bootloader。

</details>

## 3. MSP 初始值与栈空间

**Flash 保存初始栈顶地址；MSP 是 CPU 寄存器；栈里的数据放在 RAM。**

假设链接配置把初始栈顶设为 `0x20005000`：

```text
Flash 首个 4 字节保存 0x20005000
    ↓ 硬件读取
MSP ← 0x20005000
    ↓ 程序使用栈
数据写入其下方的 RAM，栈向低地址增长
```

这里的 `0x20005000` 是 20 KB SRAM 末端之后的栈顶边界，实际栈数据在有效 RAM 内。分配 1 KB 栈占用的是 RAM；向量表中保存初始 MSP 的那一项仍只占 4 字节。

栈可用于自动局部变量、保存的寄存器、返回地址及部分参数。普通整型函数返回值通常通过 `R0` 传递；返回地址通常先进入 `LR`，必要时保存到栈。[Arm 调用约定][aapcs32]

## 4. VTOR 配置与启动映射有什么区别

`VTOR` 是 Cortex-M3 系统控制块 SCB 中的寄存器，指定运行时异常和中断的向量表基地址。

| 配置 | 解决的问题 |
| --- | --- |
| BOOT 启动选择 | 复位启动时使用哪片存储器 |
| `SCB->VTOR` | 后续发生异常、中断时，到哪里查表 |

从主 Flash 启动且保持默认映射时，F103 的默认 `VTOR = 0` 可以访问 Flash 开头的向量表。写成下面这样，是显式使用同一张表的 Flash 固定地址：

```c
SCB->VTOR = 0x08000000;
```

若写成“Flash 基址 + 偏移”的形式，此时基址是 `0x08000000`，**偏移量是 0**，不是 `0x08000000`。[VTOR 说明][pm0056]

<details>
<summary>什么时候需要修改 VTOR？它能不能决定程序从哪里执行？</summary>

当要使用的向量表不在当前查表位置时，需要切换。例如：

- Bootloader 在 Flash 开头，应用向量表在 `0x08004000`：将 VTOR 指向应用的表。
- 改用 RAM 中的向量表：先把有效向量表放到 RAM，再设置 VTOR。

向量表地址要满足对齐要求；F1 模板要求基址和偏移按 `0x200` 对齐，并与链接配置一致。[F1 系统模板][system-init]

**设置 VTOR 不会搬运向量表，不会重新装入 MSP，也不会跳转执行程序。**Bootloader 切换应用还需正确处理运行状态、设置应用的 MSP，并跳到应用的复位入口；单写 VTOR 不能完成启动切换。

</details>

## 5. Flash 与 RAM 各放什么

这些段的地址和大小由链接配置决定，属于软件布局。

| 内容 | 常见运行位置 | 启动时怎样准备 |
| --- | --- | --- |
| 向量表、程序指令、只读数据 | Flash | 通常直接访问或执行 |
| `.data`：带非零初值的可写全局／静态变量 | RAM | 从 Flash 中的初始镜像复制 |
| `.bss`：未显式初始化或初始化为零的全局／静态变量 | RAM | 清零 |
| 堆 | RAM | 供动态分配使用 |
| 栈 | RAM | 初始化栈指针，随后按调用需要使用 |

```c
int a = 10;        /* 通常在 .data，Flash 中另有初值 10 */
int b;             /* 通常在 .bss，启动时清零 */
static int c = 0;  /* 通常也在 .bss */
```

函数的机器指令通常在 Flash。**函数入口地址这个数值**还可以保存在向量表、函数指针变量或寄存器中；它与“函数本体放在哪里”是两个概念。[继续看指针笔记](c-pointers.md)

## 6. 易错点与口述版本

| 容易混淆 | 正确理解 |
| --- | --- |
| 电压到阈值才开始复位 | 低于规定条件时保持复位，满足条件和延时后释放 |
| 在 SystemInit 中才选择 BOOT | 启动选择由硬件在执行复位入口前完成 |
| 读取零地址会复制整片 Flash | 别名映射只改变访问入口 |
| 向量表第一项就是栈空间 | 第一项仅保存初始栈顶地址 |
| 配置 VTOR 就能跳转应用 | VTOR 只指定查表位置 |
| 初始化 main 函数 | 应说准备 C 运行环境，再调用 main |

口述时可以这样说：

STM32 上电后先保持复位，电源满足规定条件后释放。硬件根据 BOOT 配置确定启动存储器，从启动向量表读取初始 MSP 和复位入口，再执行启动代码。

启动代码进行系统初始化，准备 .data、.bss 等 C 运行环境，最后调用 main。主 Flash 启动时，零地址是 Flash 开头的别名；VTOR 用于指定后续异常、中断使用的向量表。Flash 保存初始栈顶地址，真正的栈空间在 RAM。

## 参考资料

- [ST RM0008][rm0008]：启动配置和存储器映射。
- [ST PM0056][pm0056]：MSP、向量表和 SCB_VTOR。
- [STM32F103x8／xB 数据手册][f103c8]：容量、复位和地址分布。
- [ST F1 GCC 启动文件][startup]、[Keil 启动文件][startup-keil]、[系统初始化模板][system-init]：核对实际调用顺序和库版本差异。
- [Arm AAPCS32][aapcs32]：函数返回值、寄存器和调用约定。

[rm0008]: https://www.st.com/resource/zh/reference_manual/cd00171190-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf
[pm0056]: https://www.st.com/resource/en/programming_manual/pm0056-stm32f10xxx20xxx21xxxl1xxxx-cortex-m3-programming-manual-stmicroelectronics.pdf
[f103c8]: https://www.st.com.cn/resource/en/datasheet/stm32f103c8.pdf
[startup]: https://github.com/STMicroelectronics/cmsis-device-f1/blob/master/Source/Templates/gcc/startup_stm32f103xb.s
[system-init]: https://github.com/STMicroelectronics/cmsis-device-f1/blob/master/Source/Templates/system_stm32f1xx.c
[aapcs32]: https://github.com/ARM-software/abi-aa/blob/main/aapcs32/aapcs32.rst
[startup-keil]: https://github.com/STMicroelectronics/cmsis-device-f1/blob/master/Source/Templates/arm/startup_stm32f103xb.s
