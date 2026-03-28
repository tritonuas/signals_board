# Signals Board and Hardware-In-The-Loop Test Harness. 

## Table of Contents

* [Signals Board and Hardware-In-The-Loop Test Harness](#signals-board-and-hardware-in-the-loop-test-harness)
* [Part 1: STM32 High-Speed ADC to UART Data Streamer](#part-1-stm32-high-speed-adc-to-uart-data-streamer)
  * [Components](#components)
  * [Overview](#overview)
  * [⚙️ Hardware Peripherals Used](#️-hardware-peripherals-used)
  * [🔄 System Architecture & Execution Flow](#-system-architecture--execution-flow)
    * [1. Initialization](#1-initialization)
    * [2. The 10ms Control Loop](#2-the-10ms-control-loop)
  * [📦 Data Packet Structure](#-data-packet-structure)
* [Part 2: STM32 UART-to-USB CDC Bridge & PWM Controller](#part-2-stm32-uart-to-usb-cdc-bridge--pwm-controller)
  * [📌 Overview](#-overview)
  * [⚙️ Hardware Peripherals Used](#️-hardware-peripherals-used-1)
  * [🔄 System Architecture & Execution Flow](#-system-architecture--execution-flow-1)
    * [1. Startup & Waveform Generation](#1-startup--waveform-generation)
    * [2. PWM & DMA Initialization](#2-pwm--dma-initialization)
    * [3. The "Ping-Pong" UART-to-USB Pipeline](#3-the-ping-pong-uart-to-usb-pipeline)
    * [4. The Main Loop](#4-the-main-loop)
  * [📦 Data Pipeline Structure](#-data-pipeline-structure)

# Part 1: STM32 High-Speed ADC to UART Data Streamer

## Components: 
1. STM32F401CCU6 Development Board (WeAct Studio Blackpill)
2. Jumper cables

## Overview
This project is an embedded C application built for an STM32 microcontroller using the Hardware Abstraction Layer (HAL). It is designed to continuously sample 9 channels of analog data, package the readings with a hardware-calculated checksum, and transmit the data stream reliably over UART every 10 milliseconds. 

To maximize CPU efficiency, the system heavily relies on **Direct Memory Access (DMA)** for both data acquisition (ADC) and transmission (UART), paired with a **Double Buffering** strategy to ensure data integrity during asynchronous transfers.

---

## ⚙️ Hardware Peripherals Used
* **ADC1 (Analog to Digital Converter):** Configured for 12-bit continuous conversion. It samples analog inputs and directly writes the results to memory via DMA.
* **DMA (Direct Memory Access):** * *ADC to Memory:* Offloads the CPU by automatically transferring ADC readings into the `raw_adc` array.
    * *Memory to UART:* Handles sending the structured data payload over UART without blocking the main program loop.
* **TIM3 (Timer 3):** Configured to generate an interrupt exactly every 10ms, acting as the system's heartbeat/scheduler.
* **USART1:** Configured at **115200 baud (8N1)** with Hardware Flow Control (RTS/CTS) enabled to stream the packaged data to an external receiver.
* **CRC (Cyclic Redundancy Check):** Hardware-accelerated CRC module used to calculate a 32-bit checksum for each packet, ensuring data integrity over the serial link.
* **GPIO (General Purpose I/O):** Several pins on Port B (PB3, PB4, PB12-PB15) are initialized as push-pull outputs, available for external control or debugging.

---

## 🔄 System Architecture & Execution Flow

### 1. Initialization
Upon startup, the system initializes the clock to use the High-Speed Internal (HSI) oscillator. It then initializes all GPIOs, the DMA controller, and the required peripherals (ADC, Timer, UART, and CRC). 

### 2. The 10ms Control Loop
The main program relies on a non-blocking architecture driven by a 10ms timer interrupt:
1.  **Timer Interrupt:** Every 10ms, `TIM3` triggers `HAL_TIM_PeriodElapsedCallback`, setting the `flag_10ms` variable to `1`.
2.  **State Check:** The main `while(1)` loop detects the flag, resets it to `0`, and checks if the UART line is free (`UART_ready`).
3.  **Data Packaging:** If the UART is ready, the system loops through the 9 latest ADC readings. It tags each reading with its channel ID to ensure the receiving end knows which data belongs to which channel.
4.  **Hardware Checksum:** The hardware CRC peripheral computes a checksum over the payload, which is appended to the end of the packet.
5.  **DMA Transmission:** The completed packet is handed off to the UART DMA controller via `HAL_UART_Transmit_DMA`.
6.  **Double Buffering:** The system flips the `active_buf` variable. Because DMA operates in the background, the next 10ms cycle will construct its data in a secondary buffer, preventing the program from overwriting the memory currently being transmitted. 
7.  **Tx Complete Callback:** Once the DMA transmission completes, `HAL_UART_TxCpltCallback` fires, setting `UART_ready` back to `1` so the next packet can be sent.

---

## 📦 Data Packet Structure
Each packet transmitted over UART is exactly **22 bytes** long and follows this specific structure:


| Byte Index | Size | Description |
| :--- | :--- | :--- |
| `0` | 1 Byte | **Sync Byte:** Hardcoded to `0xAA` to denote the start of a packet. |
| `1` | 1 Byte | **Payload Size/ID:** Hardcoded to `18`. |
| `2 - 19` | 18 Bytes | **ADC Payload:** 9 sequential 16-bit blocks. <br> • **Bits [0-11]:** 12-bit Raw ADC Value. <br> • **Bits [12-15]:** 4-bit Channel ID (0 to 8). |
| `20 - 21` | 4 Bytes | **Checksum:** 32-bit Hardware CRC calculated over the first 20 bytes of the buffer. *(Note: Overlaps previous word alignment, stored at the end of the 32-bit aligned array).* |

# Part 2: STM32 UART-to-USB CDC Bridge & PWM Controller

## 📌 Overview
This project is an embedded C application built for an STM32 microcontroller. It serves a dual purpose: 
1. **High-Speed Data Bridge:** It acts as a transparent, non-blocking bridge that receives 22-byte data payloads via UART and immediately forwards them to a computer over USB using the Communication Device Class (CDC / Virtual COM Port).
2. **Advanced Hardware PWM Generation:** It concurrently drives multiple PWM output channels, including a smooth, DMA-driven quadratic "breathing" LED effect, entirely offloaded from the main CPU loop.

---

## ⚙️ Hardware Peripherals Used
* **USART1 & USB (CDC):** * USART1 is configured at **115200 baud** to receive incoming data streams.
    * The USB peripheral is configured as a Virtual COM port to push that data to a connected host machine.
* **DMA (Direct Memory Access):** * *UART Rx:* Catches incoming serial data into memory without CPU intervention.
    * *TIM3 PWM:* Streams a pre-calculated wave table directly into the timer's compare registers to create smooth hardware animations.
* **Hardware Timers (TIM2, TIM3, TIM5):**
    * **TIM3:** Runs dynamically, utilizing 4 channels (PA6, PB0, PA7, etc.) linked to DMA to generate a 2000-step quadratic fade-in/fade-out "breathing" sequence.
    * **TIM2 & TIM5:** Configured for static PWM outputs at specific duty cycles (e.g., 5%, 75%, 120%) across various pins (PA5, A0-A3).

---

## 🔄 System Architecture & Execution Flow

### 1. Startup & Waveform Generation
Before entering the main loop, the MCU pre-calculates a 2000-step quadratic curve (`breath_table`). 
* It calculates the mathematical fade-in for the first 1000 steps and mirrors it for the fade-out.
* The math scales perfectly to the timer's maximum period (`15999`), ensuring maximum resolution for the PWM brightness.

### 2. PWM & DMA Initialization
Once the curve is calculated, `HAL_TIM_PWM_Start_DMA` is called for TIM3's four channels. From this point on, the DMA controller continuously cycles the pre-calculated array into the timer, animating the outputs infinitely with zero CPU overhead. TIM2 and TIM5 are also started with fixed duty cycle values.

### 3. The "Ping-Pong" UART-to-USB Pipeline
To prevent data loss while bridging UART to USB, the system uses a strict **Double Buffering (Ping-Pong)** architecture utilizing the `HAL_UARTEx_ReceiveToIdle_DMA` function and a custom Rx Event Callback:
1.  **DMA Reception:** The UART DMA listens for incoming bytes and stores them in row `0` of the `rx_payload` array.
2.  **Event Trigger:** When exactly 22 bytes are received (or the line goes idle), the `HAL_UARTEx_RxEventCallback` interrupts the system.
3.  **Instant Flip & Re-arm:** The code instantly locks the filled row, flips the `current_buf` tracker to row `1`, and re-arms the UART DMA. This ensures the hardware is ready to catch the very next byte in the background without missing a beat.
4.  **USB Transmission:** Finally, the CPU takes the safely locked row `0` and transmits its contents over USB using `CDC_Transmit_FS()`.

### 4. The Main Loop
Because the PWM generation and UART-to-USB forwarding are entirely interrupt and DMA-driven, the `while(1)` loop is completely empty, leaving 100% of the CPU cycles available for future feature expansion.

---

## 📦 Data Pipeline Structure
* **Input:** 22-byte fixed payloads arriving asynchronously via USART1.
* **Buffering:** 2D Array `rx_payload[2][22]` acting as a ping-pong buffer.
* **Output:** Raw bytes forwarded sequentially over USB CDC to the host PC.
