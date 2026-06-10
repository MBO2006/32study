
# HC05.c 源码分析（带详细注释）

> 本文件对蓝牙模块驱动 `HC05.c` 进行逐段分析，解释每个函数和数据结构的用途、设计意图及注意事项。

## 1. 头文件及全局变量定义

```c
#include "HC05.h"
#include "LED.h"
#include "Servo.h"
#include "OLED.h"
#include <stdlib.h>
```

- `HC05.h`：包含模块自身宏定义、函数声明、命令字符串宏。
- `LED.h`、`Servo.h`、`OLED.h`：提供外设控制接口（LED、舵机、OLED显示屏）。
- `<stdlib.h>`：提供 `atoi`、`strtol` 等转换函数。

```c
static char rxBuffer[HC05_RX_BUF_SIZE];   // 接收缓冲区（大小定义在头文件，通常64字节）
static uint8_t rxIndex = 0;               // 缓冲区写指针
static uint16_t timeoutCounter = 0;       // 帧间超时计数器（每个字符到来时重置为10，10ms一次递减）

static uint8_t led1_state = 0;            // LED1当前状态（0-灭，1-亮）
static uint8_t led2_state = 0;            // LED2当前状态
static float current_angle = 0.0f;        // 当前舵机角度（0~180°）
```

- 这几个 `static` 全局变量保存整个模块的运行状态。
- `timeoutCounter` 用于检测一帧命令是否接收完成（10ms 定时调用 `HC05_Tick10ms`，每收到一个字符重置为 10，若 100ms 内无新字符则认为帧结束）。

---

## 2. HC05_Init —— 初始化串口及蓝牙模块

```c
void HC05_Init(uint32_t baudrate)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 开启 GPIOA 和 USART1 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    // PA9 → USART1_TX （复用推挽输出）
    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // PA10 → USART1_RX （浮空输入）
    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // 配置 USART 参数：波特率、8位数据、无校验、1停止位、无流控
    USART_InitStructure.USART_BaudRate            = baudrate;
    USART_InitStructure.USART_WordLength          = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits            = USART_StopBits_1;
    USART_InitStructure.USART_Parity              = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode                = USART_Mode_Rx | USART_Mode_Tx;
    USART_Init(USART1, &USART_InitStructure);

    // 使能接收中断（RXNE）
    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);

    // 配置中断优先级并开启
    NVIC_InitStructure.NVIC_IRQChannel                   = USART1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority        = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd                = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // 启动 USART1
    USART_Cmd(USART1, ENABLE);

    // 清空缓冲区和超时状态，向蓝牙发送固件版本信息
    rxIndex = 0;
    timeoutCounter = 0;
    HC05_SendString("VER: " FW_VERSION "\r\n");
}
```

- 该函数在 `main` 中调用，波特率由上位机设置决定（如 9600）。
- 发送版本号的目的是方便上位机确认连接成功。

---

## 3. 基本发送函数

```c
static void HC05_SendByte(uint8_t byte)
{
    uint32_t timeout = 0xFFFF;
    while ((USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET) && --timeout); // 等待发送寄存器空，带超时保护
    if (timeout) USART_SendData(USART1, byte);
}
```

- 每次发送一个字节，若硬件死锁（TXE 一直为 RESET），超时后放弃发送，避免死循环。

```c
void HC05_SendString(const char *str)
{
    if (str == NULL) return;
    while (*str) HC05_SendByte((uint8_t)*str++);
}
```

- 循环发送字符串，直到 `\0`。

```c
void HC05_Printf(const char *fmt, ...)
{
    char buffer[64];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buffer, sizeof(buffer), fmt, args); // 格式化字符串
    va_end(args);
    HC05_SendString(buffer);
}
```

- 提供类似 `printf` 的格式化输出功能，通过蓝牙串口发送。

---

## 4. LED 和舵机状态更新函数

```c
static void UpdateLEDState(uint8_t led1, uint8_t led2)
{
    if (led1) { LED1_On(); led1_state = 1; OLED_ShowString(2, 1, "LED1 ON "); }
    else      { LED1_Off(); led1_state = 0; OLED_ShowString(2, 1, "LED1 OFF"); }

    if (led2) { LED2_On(); led2_state = 1; OLED_ShowString(3, 1, "LED2 ON "); }
    else      { LED2_Off(); led2_state = 0; OLED_ShowString(3, 1, "LED2 OFF"); }
}
```

- 同时更新 GPIO 电平、全局状态变量和 OLED 显示。

```c
static void UpdateAngle(float angle)
{
    Servo_SetAngle(angle);            // 调用舵机驱动设置角度（PWM 0.5~2.5ms 占空比）
    current_angle = angle;
    OLED_ShowString(1, 1, "Angle:     ");
    OLED_ShowNum(1, 8, (uint32_t)angle, 3);
}
```

- 注意：`Servo_SetAngle` 内部会将角度映射为 PWM 比较值并输出到 PA0。

```c
static void SendFullStatus(void)
{
    HC05_Printf("STATUS: LED1=%s LED2=%s Angle=%d\r\n",
                led1_state ? "ON" : "OFF",
                led2_state ? "ON" : "OFF",
                (int)current_angle);
}
```

- 格式化发送当前完整状态，上位机通过解析该字符串更新界面 LED 指示灯和角度值。

---

## 5. HC05_ExecuteCommand —— 单一命令执行

```c
static void HC05_ExecuteCommand(const char *cmd)
{
    if (strcmp(cmd, CMD_LED1_ON) == 0)                     // "LED1_ON"
    {
        UpdateLEDState(1, led2_state);
        HC05_SendString("OK: LED1 ON\r\n");
        SendFullStatus();
    }
    else if (strcmp(cmd, CMD_LED1_OFF) == 0)               // "LED1_OFF"
    {
        UpdateLEDState(0, led2_state);
        HC05_SendString("OK: LED1 OFF\r\n");
        SendFullStatus();
    }
    // ... LED2 同理
    else if (strcmp(cmd, CMD_STATUS) == 0)                  // "STATUS"
    {
        SendFullStatus();
    }
    else if (strncmp(cmd, CMD_ANGLE_PREFIX, 3) == 0)       // 以 "ANG" 开头
    {
        const char *num_str = cmd + 3;
        int angle = atoi(num_str);                         // 转换为整数角度
        if (angle >= 0 && angle <= 180)
        {
            UpdateAngle((float)angle);
            HC05_SendString("OK\r\n");
            SendFullStatus();
        }
        else
        {
            HC05_SendString("ERR: Angle must 0~180\r\n");
        }
    }
    else
    {
        HC05_SendString("ERR: Unknown command\r\n");
    }
}
```

- 功能：接收一条**完整的**命令字符串，解析后执行动作，并回复 OK/ERR。
- 这是原本的命令处理入口，但只能处理单个命令。

---

## 6. HC05_ExtractAndExecute —— 从缓冲区中提取并执行一条命令

> **核心改进**：该函数解决了命令粘连的问题，使得缓冲区中可以连续多个命令（如 `ANG90LED1_ON`）被依次识别和执行。

```c
static int HC05_ExtractAndExecute(char *buf)
{
    if (buf == NULL || *buf == '\0')
        return 0;

    // 跳过前导空白
    while (*buf == ' ' || *buf == '\r' || *buf == '\n')
        buf++;

    if (*buf == '\0')
        return 0;

    const char *start = buf;       // 记录命令起始位置

    // 依次尝试匹配每种已知命令
    if (strncmp(start, CMD_LED1_ON, strlen(CMD_LED1_ON)) == 0)
    {
        HC05_ExecuteCommand(CMD_LED1_ON);
        return (int)(strlen(CMD_LED1_ON) + (buf - start));
    }
    // ... 其他固定字符串命令同理

    else if (strncmp(start, CMD_ANGLE_PREFIX, 3) == 0)  // ANG开头
    {
        const char *num = start + 3;
        while (*num == ' ') num++;                    // 跳过可能的空格
        char *endptr;
        int angle = (int)strtol(num, &endptr, 10);   // 解析数字
        if (endptr != num)                           // 至少解析到一个数字？
        {
            char angleCmd[16];
            snprintf(angleCmd, sizeof(angleCmd), "ANG%d", angle);
            HC05_ExecuteCommand(angleCmd);
            return (int)((num - start) + (endptr - num) + 3); // 消耗长度
        }
        else
        {
            return 3;   // 无效数字，跳过"ANG"三个字符防止死循环
        }
    }
    else
    {
        // 未知命令，跳过到下一个分隔符或结尾
        const char *ptr = start;
        while (*ptr && *ptr != ' ' && *ptr != '\r' && *ptr != '\n')
            ptr++;
        int skipped = ptr - start;
        if (skipped == 0) skipped = 1;    // 至少跳过一个字符
        return skipped;
    }
}
```

- 返回值：**实际消耗的字符数**，调用者利用该长度移动缓冲区指针。
- 对 `ANG` 命令使用了更健壮的 `strtol`，能正确处理末尾换行。

---

## 7. 中断处理与定时轮询

### 中断服务函数

```c
void HC05_IRQHandler(void)
{
    if (USART_GetITStatus(USART1, USART_IT_RXNE) != RESET)
    {
        uint8_t data = USART_ReceiveData(USART1);
        if (rxIndex < HC05_RX_BUF_SIZE - 1)   // 防止溢出
            rxBuffer[rxIndex++] = data;
        timeoutCounter = 10;                  // 每收一个字符重置帧超时
        USART_ClearITPendingBit(USART1, USART_IT_RXNE);
    }
}
```

- 该函数被放置在 `USART1_IRQHandler` 中调用（见 main.c）。
- 每收到一个字节就存入 `rxBuffer` 并重置超时倒计时。

### 10ms 定时调用函数

```c
void HC05_Tick10ms(void)
{
    if (timeoutCounter > 0)
    {
        timeoutCounter--;
        if (timeoutCounter == 0 && rxIndex > 0)    // 超时且缓冲区非空
        {
            rxBuffer[rxIndex] = '\0';              // 安全终止字符串

            int processed = 0;
            while (processed < rxIndex)
            {
                char *p = rxBuffer + processed;
                int consumed = HC05_ExtractAndExecute(p); // 尝试解析并执行
                if (consumed <= 0) break;                // 解析失败，退出
                processed += consumed;

                // 跳过命令后的空白字符
                while (processed < rxIndex &&
                       (rxBuffer[processed] == ' ' ||
                        rxBuffer[processed] == '\r' ||
                        rxBuffer[processed] == '\n'))
                    processed++;
            }

            // 处理完毕，清空缓冲区
            rxIndex = 0;
            memset(rxBuffer, 0, HC05_RX_BUF_SIZE);
        }
    }
}
```

- 由 `main` 循环每 10ms 调用一次。
- 超时机制：若 10×10ms=100ms 内未收到新字符，则认为一帧结束，开始解析。
- **循环解析**的设计使得 `"ANG90ANG80ANG70"` 这样的粘连命令能被全部执行，最终舵机稳定在最后一次设定的角度（70°）。

---

## 8. 总结与使用注意事项

- **命令格式**：`LED1_ON / LED1_OFF / LED2_ON / LED2_OFF / STATUS / ANGx`（x 为 0~180 整数）。
- **帧分离依赖**：100ms 字符间隔超时，因此不要连续快速发送单个字符（上位机 Python 脚本一次发送完整命令，不会有问题）。
- **PWM 输出**：舵机使用 PA0（TIM2_CH1），需确保 `Servo_Init` 在 `LED_Init` 之后执行，避免 IO 功能被覆盖。
- **供电**：舵机必须外接 5V 且与 STM32 共地，否则可能无法转动或抖动。
- **改进点**：当前代码已通过 `HC05_ExtractAndExecute` 修复了命令粘连问题，使蓝牙控制更加可靠。

