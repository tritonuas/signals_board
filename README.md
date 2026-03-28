# Signals Board and Hardware-In-The-Loop Test Harness. 

# Part 1: STM32 High-Speed ADC to UART Data Streamer

## Components: 
1. STM32F401CCU6 Development Board (WeAct Studio Blackpill)
2. Jumper cables

## 📌 Overview
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

# Part 2: 
| Byte Index | Size | Description |
| :--- | :--- | :--- |
| `0` | 1 Byte | **Sync Byte:** Hardcoded to `0xAA` to denote the start of a packet. |
| `1` | 1 Byte | **Payload Size/ID:** Hardcoded to `18`. |
| `2 - 19` | 18 Bytes | **ADC Payload:** 9 sequential 16-bit blocks. <br> • **Bits [0-11]:** 12-bit Raw ADC Value. <br> • **Bits [12-15]:** 4-bit Channel ID (0 to 8). |
| `20 - 21` | 4 Bytes | **Checksum:** 32-bit Hardware CRC calculated over the first 20 bytes of the buffer. *(Note: Overlaps previous word alignment, stored at the end of the 32-bit aligned array).* |
