# RISC-V architecture


## Overview

[Reference Guide](https://creatorsim.github.io/creator/guides/riscv.pdf).


## System calls

| Service      | Trap Code | Input                       | Output                           | Notes |
|---	       |---	   |---	                         |---	                            |---    |
| print_int    | a7 = 1    | a0  = int    to be printed  | Print a0  to display             |       |
| print_float  | a7 = 2    | fa0 = float  to be printed  | Print fa0 to display             |       |
| print_double | a7 = 3    | fa0 = double to be printed  | Print fa0 to display             |       |
| print_string | a7 = 4    | a0  = 1st char's address    | Print string in the display      |       |
| read_int     | a7 = 5    |                             | Read integer in a0               |       |
| read_float   | a7 = 6    |                             | Read float   to fa0              |       |
| read_double  | a7 = 7    |                             | Read double  to fa0              |       |
| read_string  | a7 = 8    | a0 = buffer address, a1= buffer length | Read string           |       |
| sbrk         | a7 = 9    | a0 = number of bytes        | a0 points to the allocated memory | Allocation from heap |
| exit         | a7 = 10   |                             |                                   | End of execution |
| print_char   | a7 = 11   | a0  = ASCII code            | Print a0 to display               |       |
| read_char    | a7 = 12   |                             | Read char to a0                   |       |


## Interrupts

In RISC-V, when an interrupt happens, a bit is set in the `MIP` (_Machine Interrupt Pending_) control register.
Depending on the type of interrupt, it sets a different bit. 
For example:
- Bit `3` (`MSIP`) is set to indicate a _software_ interrupt
- Bit `11` (`MEIP`) is set to indicate an _external_ interrupt

Then, the value of the current instruction is stored in the `MEPC` control register. 
The value for the interrupt handler is stored in the `MTVEC` control register, where bits `1` and `0` (MODE) determine the vector mode, and the rest of the register encodes the base address (BASE).  
The different modes are:
- `0` (direct): All traps set `pc` to the base address
- `1` (vectored): Asynchronous interrupts set `pc` to $BASE+4\times cause$

Here we implemented the _direct_ mode, meaning that `MTVEC` holds `0x00000000`, the address of the handler.

> [!NOTE]
> As we'll see in [Interrupt handling](#interrupt-handling), this requires the
> handling routine to be at the start of the text (`.text`) segment.

Also, the cause of the interrupt is stored in the `MCAUSE` (_Machine Cause_).
This control register is divided into bit `31`, which holds the interrupt type, and the rest of the bits, each bit corresponding to a specific exception code.
Some of the most used are:
- `0`-`3` (`0x00000008`): Machine software interrupt
- `0`-`8` (`0x00000100`): Machine external interrupt - `1`-`11` (`0x80000800`): Environment call from U-mode

Therefore, in the case of the `ecall` instruction, bit `3` of `MIP` and bit 8 of `MCAUSE` are set.

### Interrupt enabling
The `MIE` control register is in charge, together with `MSTATUS`, of enabling/disabling interrupt types. The types use the same bits as in the `MIP` register.


### Interrupt handling
First, we need to talk about some new privileged instructions:

- `mret`: This instruction is used to return from an interrupt, which saves the `MEPC` to the `PC`, clears the interrupt by clearing bits `3` and `11` in `MIP`, and resetting `MCAUSE` to `0`. It also changes the execution mode back to `ExecutionMode.User` (U-mode)
- `csrrw`: This instruction switches the values of a control register and a user register. It's mainly used to store the values of user registers while handling the interrupt, as we can't operate with control registers. 
The `MSCRATCH` control register is provided in order to add an extra register.

Reference: [The RISC-V Instruction Set Manual Volume II: Privileged Architecture](https://github.com/riscv/riscv-isa-manual/), chapters 3.1, 3.3.1 and 3.3.2.


> [!NOTE]
> More details in the [Master Thesis "Implementing Interrupts, Timers, and Memory-Mapped I/O in CREATOR", by Luis Daniel Casais Mezquida](../../docs/interrupts-thesis.pdf), and [RISC-V's Specification](https://riscv.atlassian.net/wiki/spaces/HOME/pages/16154769/RISC-V+Technical+Specifications#ISA-Specifications).

### Implemented features
Here is the table of implemented RISC-V features:

| Chapter                                   | Feature                                                                   | Status             | Notes                                                                                                                        |
| ----------------------------------------- | ------------------------------------------------------------------------- | :----------------: | ---------------------------------------------------------------------------------------------------------------------------- |
| I.7.1                                     | CSR Instructions                                                          | ✅ | Only `csrrw`, and without checking for register `x0`                                                                         |
| II.3.1.1 - II.3.1.5                       | Processor and ISA information (`misa`, `mvendorid`, etc.)                 | ❌                |                                                                                                                              |
| II.3.1.6                                  | `mstatus`/`mstatush`                                                      | ✅ | Only _Privilege and Global Interrupt-Enable_ (chapter II.3.1.6.1). Only `mstatus`, as only the 32-bit version is implemented |
| II.3.1.7, II.3.1.9, II.3.1.13 - II.3.1.16 | Interrupts (`mtvec`, `mip`, `mie`, `mscratch`, `mepc`, `mcause`)          | ✅ | No `mtval`                                                                                                                   |
| II.3.1.8                                  | Trap Delegation                                                           | ❌                |                                                                                                                              |
| II.3.1.10                                 | Hardware performance Monitor                                              | ❌                |                                                                                                                              |
| II.3.1.11 - II.3.1.12                     | Counters                                                                  | ❌                |                                                                                                                              |
| II.3.2.1 - II.3.3.2                       | Environmen Calls and Trap-return                                          | ✅ | Not breakpoints                                                                                                              |
| II.3.1.17 - II.3.2, II.3.6 - II.3.7       | Environment, Security and Memory                                          | ❌                |                                                                                                                              |
| II.10                                     | Supervisor-Level ISA                                                      | ❌                |                                                                                                                              |
| II.4 - II.9, II.11 - II.18                | Volume II Extensions                                                      | ❌                |                                                                                                                              |



## Devices
There are two memory-mapped devices defined.

### `console`
Handles console I/O operations.

**Address Map**:
```
0xF0000000: Control register
0xF0000004: Status register
0xF0000008-0xF000000F: Data buffer (8 bytes)
```

Information about how the device works in the [Devices](../development/devices.md#consoledevice) section.


### `os`
Handles OS-level operations.

**Address Map** (typical):
```
0xF0000010: Control register
0xF0000014: Status register
0xF0000018-0xF000001F: Data buffer (8 bytes)
```

Information about how the device works in the [Devices](../development/devices.md#osdriver) section.