# 深入浅出 Zephyr 设备树（Device Tree）：解密 chosen 与 aliases 的设计哲学与应用

在嵌入式操作系统的开发中，如何实现“软件与硬件解耦”一直是架构设计核心痛点。作为目前备受瞩目的物联网 RTOS，**Zephyr** 深度依赖设备树（Device Tree, DTS）来描述底层硬件拓扑。

除了描述真实的物理外设之外，Zephyr 的设备树中还有两个非常特殊且高频使用的根级节点：**/chosen** 和 **/aliases**。它们虽然不直接对应某一个具体的物理寄存器，但在系统架构和应用开发中起到了至关重要的桥梁作用。

本文将带你彻底厘清这两个节点的本质区别、设计初衷以及在代码中的实际用法。

---

## 一、 `/chosen` 节点：给“系统底层/内核”看的配置开关

### 1. 什么是 `/chosen`？

`/chosen` 是设备树规范中的一个特殊根级节点。它**不代表真正的物理硬件设备**，而是专门用来向内核、Bootloader 或系统子系统传递**全局配置信息和默认软硬件绑定关系**。

### 2. 核心特点与典型属性

在 Zephyr 中，内核和子系统（如串口日志、内存管理、引导分区）在编译时需要一套标准统一的契约。不管你更换了什么芯片或板卡，内核只认这些标准的属性名：

* **`zephyr,console`**：指定系统默认的控制台输出设备（通常指向一个串口，如 `&uart0`）。`printk()` 或 `LOG` 宏打印的信息默认会通过它输出。
* **`zephyr,shell-uart`**：指定 Zephyr Shell 交互终端所使用的串口。
* **`zephyr,sram` / `zephyr,flash**`：指定系统运行时主要的内存和闪存区域。
* **`zephyr,code-partition`**：在支持 Flash 分区或 MCUboot 的系统中，指定当前应用程序实际运行的 Flash 分区。

### 3. 代码示例与 DTS 片段

```dts
/ {
    chosen {
        zephyr,console = &uart0;
        zephyr,shell-uart = &uart0;
        zephyr,flash = &flash0;
        zephyr,code-partition = &slot0_partition;
    };
};

```

### 4. 在 C 代码中如何访问？

Zephyr 提供了简洁的宏 API 来读取 `chosen` 节点中的配置，开发者无需手动解析字符串：

```c
#include <zephyr/devicetree.h>

/* 获取 zephyr,console 指代的节点标识符 */
#define CONSOLE_NODE DT_CHOSEN(zephyr_console)

```

---

## 二、 `/aliases` 节点：给“应用层/通用例程”看的硬件快捷方式

### 1. 什么是 `/aliases`？

`/aliases`（别名节点）是一个纯粹的**快捷方式映射表**。它的主要目的是为具体的物理外设（如某个按键、LED、传感器、特定串口）提供统一的简短命名，从而实现**应用代码的跨板卡复用**。

### 2. 为什么需要 aliases？

不同的开发板，板载 LED 连接的 GPIO 引脚可能完全不同（有的在 `GPIOA` 第 5 脚，有的在 `GPIOB` 第 12 脚）。
为了让同一个“点灯”（Blinky）例程能够在成百上千种不同的开发板上直接编译运行，Zephyr 规定标准板级设备树必须提供一些通用别名：

```dts
aliases {
    led0 = &red_led;
    sw0 = &button_0;
};

```

### 3. 在 C 代码中如何访问？

在编写应用程序时，你不需要关心这个 LED 到底挂在哪个 GPIO 控制器上，只需要使用别名：

```c
#include <zephyr/devicetree.h>

#define LED0_NODE DT_ALIAS(led0)

static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);

```

当更换开发板时，应用代码一行都不用修改，只需在对应新板子的 `.dts` 中把 `led0` 指向新的物理引脚即可。

---

## 三、 核心区别对比总结

| 维度 | `/chosen` (被选中的节点) | `/aliases` (别名) |
| --- | --- | --- |
| **主要服务对象** | **系统内核与子系统**（Console、Memory、Bootloader） | **应用层代码与通用例程**（Blinky、Sensor Demo） |
| **指向目标** | 系统级核心配置或服务（如 `zephyr,console = &uart0;`） | 具体的物理外设硬件（如 `led0 = &red_led;`） |
| **命名规范** | 部分属性名由 Zephyr 规范硬性规定（如 `zephyr,console`） | 名称完全由开发者、板级支持包（BSP）自定义 |
| **C 语言宏接口** | `DT_CHOSEN(zephyr_console)` | `DT_ALIAS(led0)` |
| **核心设计目的** | 确立系统运行时的全局资源绑定契约 | 实现“同一套代码，适配百种硬件”的解耦能力 |

---

## 四、 结语

在 Zephyr 的开发世界里，`chosen` 和 `aliases` 就像是设备树上的两个“导航仪”：

* **`chosen`** 告诉操作系统内核：“系统从哪里启动，日志往哪里打印”；
* **`aliases`** 告诉应用程序：“业务代码去哪里找你的按键和灯光”。

理解并灵活运用这两个节点，能够让你的 Zephyr 项目在多板卡移植、系统裁剪和架构设计时游刃有余。