# Custom Bootloader for STM32F4xx

A custom UART-based bootloader for STM32F4 series microcontrollers (tested on STM32F407 Discovery), supporting In-Application Programming (IAP) via serial communication. Includes a Python host-side programmer script.

---


## 🧠 How It Works

### Boot Flow

```
Reset
  └─> HAL Init, Clock, GPIO, UART, CRC
        └─> Is User Button pressed?
              ├─ YES → bootloader_uart_read_data()   [Bootloader Mode]
              └─ NO  → bootloader_jump_to_user_app() [User App @ Sector 2]
```

### Memory Layout

| Region        | Address Range               | Size     | Usage                      |
|---------------|-----------------------------|----------|----------------------------|
| Sector 0      | 0x0800 0000 – 0x0800 3FFF   | 16 KB    | **Our Custom Bootloader**  |
| Sector 1      | 0x0800 4000 – 0x0800 7FFF   | 16 KB    | (reserved)                 |
| Sector 2–7    | 0x0800 8000 – 0x0807 FFFF   | ~448 KB  | **User Application**       |
| System Memory | 0x1FFF 0000 – 0x1FFF 77FF   | 30 KB    | ST Factory Bootloader      |
| SRAM1         | 0x2000 0000 – 0x2001 BFFF   | 112 KB   | Stack / Heap / Data        |
| SRAM2         | 0x2001 C000 – 0x2001 FFFF   | 16 KB    | Extra RAM                  |

The user application is expected to start at **`0x08008000`** (Flash Sector 2).

---

## 🔌 Hardware Setup

**Board:** STM32F407 Discovery (or compatible STM32F4xx board)

| Signal       | MCU Pin  | Peripheral |
|--------------|----------|------------|
| USART2 TX    | PA2      | Communication UART (to PC) |
| USART2 RX    | PA3      | Communication UART (from PC) |
| USART3 TX    | PD8      | Debug UART |
| USART3 RX    | PB11     | Debug UART |
| User Button  | PA0 (B1) | Boot mode select |
| Status LED   | PD14 (LD5) | Write/Erase indicator |

**Baud Rate:** 115200, 8N1, no flow control

**To enter bootloader mode:** Hold the **User Button (B1)** during reset.

---

## 📡 Supported Commands

| Command                  | Code   | Response            | Description                            |
|--------------------------|--------|---------------------|----------------------------------------|
| `BL_GET_VER`             | `0x51` | 1 byte              | Read bootloader version                |
| `BL_GET_HELP`            | `0x52` | N bytes             | List all supported command codes       |
| `BL_GET_CID`             | `0x53` | 2 bytes             | Read MCU chip ID                       |
| `BL_GET_RDP_STATUS`      | `0x54` | 1 byte              | Read Flash read protection level       |
| `BL_GO_TO_ADDR`          | `0x55` | 1 byte (status)     | Jump to a specified address            |
| `BL_FLASH_ERASE`         | `0x56` | 1 byte (status)     | Sector or mass erase of user flash     |
| `BL_MEM_WRITE`           | `0x57` | 1 byte (status)     | Write binary data to flash/SRAM        |
| `BL_EN_RW_PROTECT`       | `0x58` | 1 byte (status)     | Enable write/read-write protection     |
| `BL_READ_SECTOR_STATUS`  | `0x5A` | 2 bytes             | Read all sector protection status      |
| `BL_DIS_RW_PROTECT`      | `0x5C` | 1 byte (status)     | Disable all sector protection          |

### Command Packet Format

Every command follows this structure:

```
[ Length (1B) | Command Code (1B) | Parameters (nB) | CRC32 (4B) ]
```

- **Length:** number of bytes that follow (excluding length byte itself)
- **CRC32:** computed over all bytes except the CRC field itself
- **ACK response:** `[0xA5, len_to_follow]` → then reply data
- **NACK response:** `[0x7F]`

---

## 🐍 Python Programmer Script

### Requirements

```bash
pip install pyserial
```

### Usage

```bash
python STM32_Programmer_V1.py
```

You will be prompted to enter your COM port (e.g., `COM3` on Windows, `/dev/ttyUSB0` on Linux), then an interactive menu appears:

```
+==========================================+
|               Menu                       |
|         STM32F4 BootLoader v1            |
+==========================================+

   BL_GET_VER                            --> 1
   BL_GET_HLP                            --> 2
   BL_GET_CID                            --> 3
   BL_GET_RDP_STATUS                     --> 4
   BL_GO_TO_ADDR                         --> 5
   BL_FLASH_MASS_ERASE                   --> 6
   BL_FLASH_ERASE                        --> 7
   BL_MEM_WRITE                          --> 8
   BL_EN_R_W_PROTECT                     --> 9
   BL_READ_SECTOR_P_STATUS               --> 11
   BL_DIS_R_W_PROTECT                    --> 13
   MENU_EXIT                             --> 0
```

### Flashing a User Application

1. Build your user app and export a `.bin` file named `user_app.bin`
2. Place `user_app.bin` in the same directory as `STM32_Programmer_V1.py`
3. Run the script and select **option 7** to erase Sector 2 first
4. Select **option 8**, then enter the base address `08008000`

---

## ⚙️ Build & Flash the Bootloader

### Prerequisites

- STM32CubeIDE (or any GCC ARM toolchain)
- STM32CubeProgrammer or ST-LINK utility (to flash the bootloader itself initially)

### Steps

1. Open the project in **STM32CubeIDE**
2. Build → produces `.elf` / `.bin`
3. Flash to **Sector 0** (address `0x08000000`) using ST-LINK
4. After that, all future user app updates can be done via the Python script over UART

---

## 🔐 CRC Verification

Every command is protected by a **CRC32** checksum computed using the STM32 hardware CRC peripheral (`HAL_CRC_Accumulate`). The bootloader verifies the CRC before executing any command, responding with NACK on failure.

---

## 📝 Notes

- Debug messages are printed over **USART3** (enable by uncommenting `#define BL_DEBUG_MSG_EN` in `main.c`)
- `BL_MEM_READ` and `BL_OTP_READ` are stubbed — left as exercises
- User application **must** have its vector table relocated to `0x08008000`
- The bootloader sets `SCB->VTOR` before jumping to the user app

---

## 📄 License

This software is licensed under terms found in the STMicroelectronics LICENSE file. See source file headers for details.
