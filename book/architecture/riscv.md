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

  The RISC-V-32 ISA defines different privilege modes that determine the level of access and control a process has over the system's resources levels, in particular, access to _Control Status Registers_ (CSRs), the privileged instruction set, and privileged ISA extensions. The instruction subset to control CSRs is provided in the _Zicsr_ extension @RISCVUnprivileged, chapter 7.
  The ISA defines three levels: User/Application (U), Supervisor (S), and Machine (M), allowing implementations to provide one, two, or three of them (There are only three supported combinations: M-level, M-level and U-level, or all three). M-level is the highest privilege level, while U-level is intended for conventional applications and S-level for operating systems @RISCVPrivileged, chapers 1 and 2, @Bulić2024 section 2.5.1.

  RISC-V handles exceptions and interrupts by generating traps.
As a trap involves elevating the privilege level, e.g. requesting an OS system call from a user program, both M and S privilege levels include a set of CSRs for handling them#footnote[As they are virtually the same, only the M-level set will be described.].
  The `mstatus` (_Machine Status_) register keeps track of and controls the CPU's current operating state.
In this register, there are several interrupt-related fields: field `MIE` (_Machine Interrupt Enable_) controls whether interrupts are globally enabled or disabled for that privilege mode, field `MPP` (_Machine Previous Privilege Mode_) holds the previous privilege mode, and, to allow nesting of interrupts, field `MPIE` stores the previous value of `MIE` when an interrupt occurs.
  The `mie` (_Machine Interrupt Enable_) register allows for a finer control of interrupts, allowing the programmer to set which types of interrupts are enabled: field `MSIE` (_Machine Software Interrupt Enable_), `MTIE` (_Machine Timer Interrupt Enable_), and `MEIE` (_Machine External Interrupt Enable_) control software, timer, and external interrupts, respectively.
  The `mip` (_Machine Interrupt Pending_) register indicates the interrrupts that are currently pending. Similarly to register `mie`, it specifies the type of interrupt on specific fields: `MSIP` (_Machine Software Interrupt Pending_), `MTIP` (_Machine Timer Interrupt Pending_), and `MEIP` (_Machine External Interrupt Pending_).
  Each of these fields may be writable or may be read-only. Register `mcause` (_Machine Cause_) provides information about the event that caused the trap. This register contains a field `I` specifying if the cause was an interrupt or an exception, while the rest of the bits are reserved for the exception code. RISC-V defines some of these exception codes, while others are left for the implementation to use.
  Register `mtvec` (_Machine Trap-Vector Base-Address_) holds the trap vector configuration.
Depending on the value of the `MODE` field, RISC-V allows for polled interrupts (_direct mode_, with a value of `0`) or vectored interrupts (_vectored mode_, with a value of `1`).
In direct mode, all traps cause the PC to be set to the address in the `BASE` field, while on vectored mode, traps set the PC to address $mono("BASE") + 4 times italic("cause")$, _cause_ being the exception code found in `mcause`.
  Finally, register `mepc` (_Machine Exception Program Counter_) holds the address of the instruction that generated the exception while it is handled @RISCVUnprivileged section 3.1, @Bulić2024 section 2.5.2.


When a trap is taken to M-mode, the corresponding flag in `mip` is set, and the `MIE` field in `mstatus` and the corresponding flag in `mie` are checked to determine if that interrupt is enabled.
  If it's enabled, the `MIE` field is copied into the `MPIE` field of `mstatus` and `MIE` is cleared, disabling further interrupts.
  Then, the previous privilege mode is stored in the `MPP` field of `mstatus`, the cause of the exception is encoded into `mcause`, the current PC is stored into the `mepc` register, and the PC is set to the address specified by `mtvec`.
  When the interrupt handler finishes, it calls `mret`, which resets the privilege mode (reading the `MPP` field in `mstatus`), re-enables interrupts (by copying back field `MPIE` to `MIE` in `mstatus`), and sets the PC to the value of `mepc`.
  With regard to the `mip` register, the ISA states that if the field is writable, a pending interrupt can be cleared by clearing the field, but if the field is read-only, the implementation must provide some other mechanism for clearing the pending interrupt @RISCVUnprivileged section 3.3, @Bulić2024 section 2.5.4.

More details in the Master Thesis "Implementing Interrupts, Timers, and Memory-Mapped I/O in CREATOR" By Luis Daniel Casais Mezquida
.


