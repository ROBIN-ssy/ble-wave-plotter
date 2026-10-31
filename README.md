# 蓝牙波形绘图仪（STM32F103 + HC-04）

把单片机上采集的 ADC 波形，通过蓝牙串口发到电脑/手机上的上位机，**实时画成曲线**。

用现成的上位机协议（`[p,value]` 格式），不需要自己写 PC 端软件。

---

## 链路

```
STM32F103C8T6  ──USART3 115200 8N1──►  HC-04 蓝牙透传  ──BLE──►  手机/电脑上位机
   PB10(TX)  PB11(RX)                                              （Vofa+ / 匿名上位机）
```

HC-04 上电即进入透传模式，**不需要手动开广播**，也不用发 AT 指令。

---

## 硬件连接

| 功能 | 引脚 |
|---|---|
| HC-04 TX/RX | `PB10` (USART3_TX) / `PB11` (USART3_RX) |
| LED 红 / 绿 / 蓝 | `PB0` / `PA7` / `PA6` |
| SWD 调试 | `PA13` (SWDIO) / `PA14` (SWCLK) |

---

## 数据格式

发送的是**纯文本**，每条一帧，上位机按逗号切分：

```
单通道：  [p,123]
双通道：  [p,123,456]
```

`wave_plotter.c` 里做了电压映射 —— ADC 原始值 `0~4095` 线性映射到 `0~330`，即 **0~3.30 V**：

```c
mapped_val = (data[i] * 330) / 4095;
sprintf(tx_buf, "[p,%d]", mapped_val);
HAL_UART_Transmit(huart, (uint8_t *)tx_buf, str_len, HAL_MAX_DELAY);
```

这样上位机收到的直接就是毫伏数，把纵轴单位设成 mV 就是真实电压。

---

## 文件

```
ble-wave-plotter/
├── ble_wave_hc04.ioc               CubeMX 配置（时钟树、引脚、外设都在这）
├── STM32F103C8TX_FLASH.ld          链接脚本
└── Core/
    ├── Inc/
    │   ├── main.h
    │   ├── wave_plotter.h          ★ 波形发送接口
    │   └── stm32f1xx_*.h
    ├── Src/
    │   ├── main.c                  ★ 主循环：ADC 采集 → 发送
    │   ├── wave_plotter.c          ★ 单/双通道打包与发送
    │   └── stm32f1xx_*.c
    └── Startup/
        └── startup_stm32f103c8tx.s
```

`Drivers/`（HAL 库）、`Debug/`（构建产物）、`.settings/` 都已剔除 ——
用 CubeMX 打开 `.ioc` 可以原样重新生成。

---

## 两个接口

```c
void Plotter_Send_1CH(UART_HandleTypeDef *huart, int *data, uint16_t length);
void Plotter_Send_2CH(UART_HandleTypeDef *huart, int *data1, int *data2, uint16_t length);
```

传 UART 句柄、数据数组、长度即可，内部负责映射、格式化、发送。

---

## 已知的限制（如实说明）

1. **每发一个点 `HAL_Delay(2)`**，也就是给上位机留处理时间。
   代价是采样率被压在约 **500 点/秒**，而且发送期间 CPU 是阻塞的。
   要提速应该改成 DMA + 环形缓冲，但目前这个版本没做。
2. **文本协议**比二进制协议多占带宽，115200 波特率下每帧约 8 字节，
   实际带宽利用率不高。好处是人眼可读、调试方便。
3. 只做了单/双通道，没做时间戳，横轴靠上位机按点序推算。

---

## 怎么烧

用 STM32CubeIDE 打开 `ble_wave_hc04.ioc` 生成工程，把 `Core/` 覆盖进去，
接 ST-Link 走 SWD 下载。

---

## 相关

同样的 `wave_plotter` 模块以精简形式收在
[stm32-driver-modules](https://github.com/ROBIN-ssy/stm32-driver-modules) 的 `F4/ble-wave/`，
那里只有驱动本身；**这里是完整可跑的工程**。

---

## 许可

MIT。
