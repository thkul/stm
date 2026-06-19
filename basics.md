
# key knowledge


# tooling
** editor **
    ↓ write code
.c files  +  Makefile / CMakeLists.txt  +  linker script (.ld)
    ↓ `make` / `cmake --build`  (calls `gcc-arm-none-eabi` or `clang --target=arm-none-eabi`)
** binary (.elf or any executable) **
    ↓ `arm-none-eabi-size firmware.elf`  (check Flash / RAM usage)
    ↓
    ├── probe-rs way ──────────────────
    │   ↓ `probe-rs run --chip STM32F103C8 firmware.elf`
    │     (flash + debug + RTT logs via ST-Link SWD)
    │
    OR
    │
    └── legacy way ───────────────────
        ↓ `arm-none-eabi-objcopy -O binary firmware.elf firmware.bin`
          (only for STM32CubeProgrammer / USB DFU)
        ↓ `openocd -f stlink.cfg -f stm32f1x.cfg`
        ↓ server on TCP :3333
        ↓ `arm-none-eabi-gdb firmware.elf`  (+ SVD file for named registers)
        ↓ flash + debug

** executing firmware on MCU **


# stm interfaces pipeline
USB — не послідовний порт, а складний пакетний протокол з хостом (ініціатор) і девайсом (відповідач).
UART — це простий байтовий потік (~ 1 Mbs).
Щоб мати можливість ініціалізувати потік з обох сторін, macOS і ST-Link створює емуляцію послідовного COM‑порту.

***

MCU UART
    ↓
ST‑Link (робить USB‑CDC (Virtual COM Port))
    ↓
USB кабель
    ↓
macOS kernel driver (AppleUSBCDCACM)
    ↓
COM Port:
/dev/tty.usbmodemXXXX (for dial‑in)
/dev/cu.usbmodemXXXX (for call-up)

***

Комп’ютер (Virtual COM Port over USB)
        ↓
ST‑Link (Virtual COM Port over USB USB)
        ↓
  ┌───────────────┬───────────────┐
  │ SWD Debug     │ UART Bridge   │
  │ (прошивка)    │ (логи MCU)    │
  └───────────────┴───────────────┘
        ↓
       MCU

# ARM Cortex‑M (STM32) memory

```
0xFFFF_FFFF  ← Кінець адресного простору
      ↓
══════════════════════════════════════════════
0xE000_0000  ← System Control Space (SCS)
               NVIC, SCB, SysTick, ITM, DWT,
               MPU, Debug registers
══════════════════════════════════════════════
0xD000_0000  ← Device-specific region
               (TCM RAM, AXI SRAM, CCM RAM,
                кеші, internal buses — залежить від моделі)
      ↓
0xA000_0000
══════════════════════════════════════════════
0x6000_0000  ← External memory region
               FMC, QSPI, SDRAM, NOR/NAND,
               зовнішні Flash/RAM
══════════════════════════════════════════════
0x4000_0000  ← Peripherals
               GPIO, UART, SPI, I2C, TIM,
               ADC, DAC, DMA, RCC, PWR,
               USB, CAN, ETH, SDMMC…
══════════════════════════════════════════════
0x2000_0000  ← Internal SRAM region
               .data
               .bss
               heap ↑
               stack ↓
               (усі типи внутрішньої RAM,
                розбиті на банки залежно від MCU)
══════════════════════════════════════════════
0x0000_0000  ← Code region
               Flash (основна)
               Boot ROM
               System Bootloader
               ITCM Flash (якщо є)
══════════════════════════════════════════════
```

# Startup Code
`ENTRY(_reset);`
Точка входу — перша функція після увімкнення. Не main(), а `_reset()` — вона ініціалізує пам'ять і потім викликає main().

Послідовність від увімкнення до main():
1. MCU читає перше слово за адресою 0x08000000 → початкове значення Stack Pointer
2. Читає друге слово (0x08000004) → адреса функції reset()
3. Переходить на reset()
4. reset() обнуляє .bss
5. reset() копіює .data з Flash у RAM
6. reset() викликає main()

