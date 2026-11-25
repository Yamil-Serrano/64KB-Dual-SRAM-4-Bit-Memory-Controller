# 64KB Dual-SRAM 4-Bit Memory Controller

A memory controller module that interfaces a 4-bit CPU with standard 8-bit SRAM chips, enabling individual nibble read and write operations without wasting the unused half of each memory byte. The design implements a transparent read-modify-write cycle within a single clock period using discrete 74HC-series logic, making it suitable for breadboard prototyping and custom PCB integration.

## Overview

This RAM module bridges the gap between 4-bit data paths and 8-bit memory organization. While the SRAM stores 8-bit bytes, the CPU operates on 4-bit nibbles. This controller allows the CPU to access either the upper or lower nibble of any byte independently, treating the memory space as individually addressable nibble locations when combined with the appropriate control signals.

The core challenge addressed by this design is the inability of standard SRAM to perform partial-byte writes. A 4-bit CPU attempting to modify only four bits of an eight-bit memory location would normally corrupt the adjacent nibble. This module solves this problem by performing an automatic read-modify-write operation entirely within a single clock cycle, completely transparent to the programmer.

To achieve the full 64 KB address space, the design uses **two 32K x 8 SRAM chips** connected in parallel. The MSB of the address bus (`ADDR[15]`) acts as a bank selector, using the `~CE` (Chip Enable) pins to ensure that only one SRAM drives the data bus at any given time.

## Architecture

The memory module consists of two primary data paths: a read path that delivers the selected nibble to the CPU, and a write path that reconstructs a complete byte from the old byte and the new nibble before writing it back to SRAM.

### Read Path

During read operations, the active SRAM outputs a full byte through the 74HC245 bidirectional transceiver. That byte is then fed into a 74HC157 2:1 multiplexer, which selects either the upper or lower nibble based on the SEL signal and presents the chosen nibble to the CPU's 4-bit data bus.

### Write Path

Write operations reconstruct a full byte before it ever reaches the SRAM. First, the existing byte is read out of SRAM through the 74HC245 and captured by a 74HC573 transparent latch. That latched byte is then combined with the new nibble coming from the CPU through two 74HC257 multiplexers, which merge the old byte and the new nibble into a complete byte. Finally, that reconstructed byte is driven back through the 74HC245, now switched to the opposite direction, and written into SRAM. The whole sequence is coordinated by the clock signal, which splits the operation into a rising-edge phase for reading the old byte and a falling-edge phase for writing the modified byte back.

### Memory Expansion (64 KB)

The base design uses a single 32 KB SRAM. To expand to 64 KB, a second SRAM is added in parallel, sharing the address bus, data bus, `~WE`, and `~OE` signals.

- **SRAM1 (Lower 32 KB):** `~CE` is connected directly to `ADDR[15]`. Selected when `ADDR[15] = 0`.
- **SRAM2 (Upper 32 KB):** `~CE` is connected to the output of a NAND gate used as an inverter, driven by `ADDR[15]`. Selected when `ADDR[15] = 1`.

The `~OE` pins of both SRAMs are tied to GND. The chip select logic guarantees that only one SRAM drives the data bus at a time, making the expansion completely plug-and-play with no changes to the control logic or CPU firmware.

## Schematic
<img width="1132" height="799" alt="image" src="https://github.com/user-attachments/assets/e87dd026-751e-4388-9d26-266c17a1e253" />

### PCB Design in Kicad
<img width="1919" height="1030" alt="image" src="https://github.com/user-attachments/assets/bded0256-840b-413b-bebf-f1684157f93a" />


## Components Used

| IC | Function | Quantity | Purpose |
|---|---|---|---|
| CY62256LL-70 | SRAM (32K x 8) | 2 | 64 KB total memory storage |
| 74HC245 | Octal Bus Transceiver | 1 | Bidirectional data bus control |
| 74HC573 | Octal Transparent Latch | 1 | Captures and holds old byte during write operations |
| 74HC257 | Quad 2:1 Multiplexer | 2 | Byte reconstruction from old byte and new nibble |
| 74HC157 | Quad 2:1 Multiplexer | 1 | Nibble selection during read operations |
| 74HC00 | Quad NAND Gate | 1 | Control signal generation and bank selection |

## Specifications

### Memory Characteristics

- **Memory Devices:** 2x CY62256LL-70 (32K x 8 SRAM)
- **Total Capacity:** 64 KB (65,536 bytes)
- **Address Lines:** 16 lines (A0-A15). A0-A14 are shared between both SRAMs, while A15 acts as the bank selector.
- **Organization:** 8-bit bytes, each containing two 4-bit nibbles.

### Interface Signals

| Signal | Direction | Description |
|---|---|---|
| SW | Input | Write enable (1 = write, 0 = read) |
| CLK | Input | System clock, controls phase timing |
| SEL | Input | Nibble select (0 = low nibble, 1 = high nibble) |
| ADDR | Input | Memory address (16 lines, A0-A15) |

## Control Logic

The 74HC00 generates all required control signals from the SW and CLK inputs. The core logic equations are:

```
NAND1 = NOT(CLK)
NAND2 = NOT(SW AND NAND1)
CTRL = NAND2
DIR = NOT(CTRL)
```

The CTRL signal controls SRAM write enable, latch enable, and multiplexer output enable. The DIR signal controls the direction of the bidirectional transceiver. On the clock's rising edge, CTRL is high and DIR is low: the SRAM stays in read mode, the 74HC245 points from SRAM to the circuit, and the 74HC573 latch is transparent, capturing the old byte. On the falling edge, CTRL goes low and DIR goes high: the SRAM switches to write mode, the 74HC245 reverses direction toward the SRAM, the latch holds the captured byte, and the 74HC257 multiplexers become active to merge in the new nibble.

## Operation Details

### Write Operation (SW = 1)

When the CPU initiates a write operation, the module executes the following sequence within a single clock cycle:

**Phase 1: Rising Edge - Read Old Byte**
- Active SRAM remains in read mode (WE = 1)
- 74HC245 configured for SRAM -> Circuit direction
- 74HC573 latch is transparent and captures the full byte from SRAM

**Phase 2: Falling Edge - Write Modified Byte**
- Active SRAM transitions to write mode (WE = 0)
- 74HC245 configured for Circuit -> SRAM direction
- 74HC573 holds the old byte value
- 74HC257 multiplexers reconstruct the new byte

### Read Operation (SW = 0)

Read operations are straightforward and do not require the read-modify-write process:

1. Active SRAM outputs the full byte to the 74HC245
2. 74HC245 passes the byte to the 74HC157 multiplexer
3. The multiplexer selects either the high or low nibble based on SEL
4. The selected nibble appears on the CPU data bus

## Timing Analysis and Maximum Frequency

Although the module is designed for robust operation in a modular breadboard/PCB environment, the theoretical maximum operating frequency is bounded by the critical path of the Read-Modify-Write (RMW) cycle. Since the clock has a 50% duty cycle, the total period (T_CLK) is limited by the slowest of the two phases multiplied by two.

### 1. Control Logic Delay (Common to both phases)
The CLK signal propagates through three cascaded NAND gates inside the 74HC00 to generate the CTRL and DIR signals.
*   Control time = 3 * 10ns = 30ns (Worst-case typical at 5V)

### 2. Phase 1: Read Old Byte (Rising Edge)
Data flows from the SRAM -> 74HC245 -> 74HC573. The SRAM must output valid data, it must pass through the transceiver, and the latch must capture it before the clock falls.
*   Control time: 30ns
*   SRAM access time (CY62256LL-70): 70ns
*   74HC245 propagation delay: 12ns
*   74HC573 setup time: 20ns
*   **Total Phase 1:** 30 + 70 + 12 + 20 = **132ns**

### 3. Phase 2: Write Reconstructed Byte (Falling Edge)
Data flows from the 74HC573 -> 74HC257 -> 74HC245 -> SRAM. The latch must output the data, the multiplexers must merge it, the transceiver must drive it, and the SRAM must capture it before ~WE goes high.
*   Control time: 30ns
*   74HC573 propagation delay: 15ns
*   74HC257 propagation delay: 12ns
*   74HC245 propagation delay: 12ns
*   SRAM setup time (Data valid before ~WE rises): 20ns
*   **Total Phase 2:** 30 + 15 + 12 + 12 + 20 = **89ns**

### 4. Theoretical Maximum Frequency
The critical path is Phase 1 (132ns). Therefore, the minimum clock period is:
Minimum Period (T_min) = 2 * 132ns = **264ns**

Maximum Frequency (F_max) = 1 / 264ns = 0.003787 GHz = **3.787 MHz**

> **Note:** Adding ~10-20ns for PCB parasitics and clock skew across modular jumpers brings the practical, stable maximum frequency to approximately **3.0 - 3.5 MHz**.**


## Educational Value

This project demonstrates several important concepts in digital design:

- **Memory Interfacing:** Connecting a CPU to memory with different data widths
- **Read-Modify-Write:** Performing atomic operations on memory
- **Transparent Hardware:** Implementing complex operations inside a single clock cycle
- **Bidirectional Data Buses:** Managing data flow direction on shared buses
- **Bank Switching:** Expanding memory using chip select signals
- **Timing Analysis:** Understanding propagation delays and critical paths
- **Signal Generation:** Deriving multiple control signals from basic logic gates

## Known Limitations

- **Maximum Frequency:** Ripple-through propagation limits top speed to approximately 3.78 MHz
- **No Wait States:** The module assumes the CPU can complete the operation in one cycle

## License

Licensed under the Solderpad Hardware License v2.1.
See the LICENSE file for full terms.

## Contact

If you have any questions or suggestions, feel free to reach out:

- GitHub: [Neowizen](https://github.com/Neowizen)


