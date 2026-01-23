# Creating Custom Architectures

CREATOR supports defining custom architectures through YAML configuration files. This allows adding new instruction sets or modifying existing ones.

## Creating an Architecture File
An architecture file is a YAML file that describes the architecture's properties, including its instruction set, registers, memory layout, and other relevant details. Instructions are defined with their binary encoding, assembly syntax, and semantics.

> [!NOTE]
> We provide a [JSON schema](https://json-schema.org/) for the architecure file at https://creatorsim.github.io/creator/schema/architecture.json.

The actual definition for an instruction is a simple JavaScript code block to manipulate the simulator state. Within this block, you have the `registers` variable to access the registers (e.g. `registers.PC`, or `registers[value]`), as well as [`CAPI`](CAPI.md), an API that allows you to interact with the simulator.


> [!IMPORTANT]
> The values stored in the registers are [BigInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt). Take that into account when reading or writing values:
> ```js
> const foo = registers.PC;  // 420n
> registers.PC = foo + 1n;   // 421n
> ```

---

Let's define a simple 8-bit architecture with a few instructions.

The first step is to create a YAML file, e.g., `simplearch.yml`, and fill out the `config`. We want an architecture where the word size and byte size are both 8 bits. We'll also make it little-endian, although it doesn't matter in this specific case because a word contains only one byte. `pc_offset` will be `0`. The entry point will be a function named `main`, or address `0x0` if it doesn't exist; we'll use the `;` character to write comments, and the names of the registers won't be sensitive (`PC` == `pc`). We'll also enable memory alignment and passing convention checks.

> [!IMPORTANT]
> The value of the program counter register (`program_counter`) inside the instruction definitions is affected by the `pc_offset`.
> 
> `pc_offset` is the offset that we'll add to the value of program counter the instruction "sees".
> 
> E.g. if `pc_offset` is `-4`, and we're executing an instruction at `0x0`, the "real" PC is `0x4` (because of the fetch performed at the start of the cycle), but the value of `registers.PC` (the "virtual" PC) will be `0x0`.

```yaml
version: 2.0.0
config:
  name: Simple8Bit
  description: A simple custom 8-bit architecture
  word_size: 8
  byte_size: 8
  endianness: little_endian
  pc_offset: 0
  main_function: main
  start_address: 0x0
  comment_prefix: ;
  sensitive_register_name: false
  memory_alignment: true
  passing_convention: true
```

<!-- Registers  -->

For the registers, we'll make a control register bank with a `PC` program counter register (we'll mark that with the `program_counter` property), and another integer register bank with a `A` and `B` register, as well as a `SP` stack pointer register (`stack_pointer` property).

> [!NOTE]
> A floating point bank would be defined as:
> ```yaml
>  # ...
>  - name: Floating point registers
>    type: fp_registers
>    double_precision: true  # or `false`, if sinlge-precision
> ```

All of these registers will be 8 bits, be initialized (`value`) and have a default value (`default_value`) of `0`, and will be both readable (`read` property) and writable (`write` property).

> [!NOTE]
> The `encoding` property will be used when decoding binary instructions, in this case we'll just make it sequential.
> 
> `name` is a list because a register can have multiple values, e.g. in RISC-V register `zero` can be also called `x0`, and so on. These names must all be unique.

```yaml
components:
  - name: Control registers
    type: ctrl_registers
    double_precision: false
    elements:
      - name:
          - PC
        nbits: 8
        encoding: 0
        value: 0
        default_value: 0
        properties:
          - read
          - write
          - program_counter
  - name: Integer registers
    type: int_registers
    double_precision: false
    elements:
      - name:
          - A
        encoding: 0
        nbits: 8
        value: 0
        default_value: 0
        properties:
          - read
          - write
      - name:
          - B
        encoding: 1
        nbits: 8
        value: 0
        default_value: 0
        properties:
          - read
          - write
      - name:
          - SP
        encoding: 2
        nbits: 8
        value: 0
        default_value: 0
        properties:
          - read
          - write
          - stack_pointer
```

<!-- Memory -->
For the memory layout, we'll use a simple `.text`, `.data`, `.stack` layout:
```yaml
memory_layout:
  text:
    start: 0x0000
    end: 0x03FF
  data:
    start: 0x0400
    end: 0x7FFF
  stack:
    start: 0x8000
    end: 0xFFFF
```

<!-- Instructions  -->

Now it's time for the instructions. We want an architecture with two instructions: `NOP` and `ADD` in the `base` extension. The `NOP` instruction does nothing, while the `ADD` instruction adds the values of two registers and stores the result in a destination register.

As the instructions will have the same form, we can define an instruction template called `standard` defining that our instructions will be 1 word long (`nwords`), take 1 clock cycle (`clk_cycles`) and use the full word as an operation code (type `co`) field. We then override in each instruction the value of that field.

<!-- TODO: more complete example -->

```yaml
templates:
  - name: standard
    nwords: 1
    clk_cycles: 1
    fields:
      - name: opcode
        type: co
        startbit: 7
        stopbit: 0
        order: 0

instructions:
  base:
    - name: nop
      template: standard
      fields:
        - field: opcode
          value: "0x00"
      definition: ""

    - name: add
      template: standard
      fields:
        - field: opcode
          value: "0x80"
      definition: |
        const oldValueA = registers.A;
        registers.A = (oldValueA + registers.B) & 0xFFn;
        registers.F = CAPI.ARCH.calculateFlags_ADD(oldValueA, registers.B);
```

<!-- TODO: Directives  -->

```yaml
directives:
  - name: .data
    action: data_segment
    size: null
  - name: .text
    action: code_segment
    size: null
  - name: .bss
    action: global_symbol
    size: null
  - name: .zero
    action: space
    size: 1
  - name: .space
    action: space
    size: 1
  - name: .align
    action: align
    size: null
  - name: .balign
    action: balign
    size: null
  - name: .globl
    action: global_symbol
    size: null
  - name: .string
    action: ascii_null_end
    size: null
  - name: .asciz
    action: ascii_null_end
    size: null
  - name: .ascii
    action: ascii_not_null_end
    size: null
  - name: .byte
    action: byte
    size: 1
  - name: .half
    action: half_word
    size: 2
  - name: .word
    action: word
    size: 4
  - name: .dword
    action: double_word
    size: 8
  - name: .float
    action: float
    size: 4
  - name: .double
    action: double
    size: 8
```



## Plugins

<!-- TODO -->


## Interrupt Support
Now, let's take our architecture and add support for some simple maskable and nonmaskable interrupts.

### Custom handler

We'll define two new 1-bit integer registers `MIP` (_Maskable Interrupt Pending_) and `NIP` (_Nonmaskable Interrupt Pending_) that will be set to `1` when an interrupt of the type is pending. We'll also define another 1-bit integer register `IE` to enable (value of `1`) and disable (value of `0`) maskable interrupts.

We just need to add them to `simplearch.yml`:
```yml
components:
  # ...
  - name: Integer registers
    # ...
    elements:
      # ...
      - name:
          - MIP
        encoding: 2
        nbits: 1
        value: 0
        default_value: 0
        properties:
          - read
          - write
      - name:
          - NIP
        encoding: 3
        nbits: 1
        value: 0
        default_value: 0
        properties:
          - read
          - write
      - name:
          - IE
        encoding: 4
        nbits: 1
        value: 1
        default_value: 1
        properties:
          - read
          - write
```

Now we have to define some functions to determine how interrupts work in this architecture.

First, we need to define how to determine if an interrupt happened. CREATOR has some predefined types of interrupts, and here we'll use `InterruptType.Maskable` and `InterruptType.Nonmaskable`. We have to write a function that returns the type of interrupt (`InterruptType`), or `null` if there is no interrupt:

> [!NOTE]
> You don't have to check if interrupts are enabled here, we'll define that later.

```yml
interrupts:
  check: |
    if (registers.NIP) return InterruptType.Nonmaskable;
    if (registers.MIP) return InterruptType.Maskable;
    return null;
```


Then, we must define how different types of interrupts can be created and cleared. We'll receive the desired type (`InterruptType`) inside the `type` variable:
```yml
interrupts:
  # ...
  create: |
    switch (type) {
      case InterruptType.Maskable:
        registers.MIP = 1n;
        break;
      case InterruptType.Nonmaskable:
        registers.NIP = 1n;
        break;
    }

  clear: |
    switch (type) {
      case InterruptType.Maskable:
        registers.MIP = 0n;
        break;
      case InterruptType.Nonmaskable:
        registers.NIP = 0n;
        break;
    }

  global_clear: |
    registers.MIP = 0n;
    registers.NIP = 0n;
```

> [!NOTE]
> `clear` is optional, it gets overriten by `global_clear` if it's not defined

Next, how they can be enabled and disabled, per type (and globally), as well as how to check if they are enabled. For the sake of simplicity, we'll assume nonmaskable interrupts can't be disabled.
```yml
# ...
interrupts:
  # ...
  is_enabled: |
    switch (type) {
      case InterruptType.Maskable:
        return registers.MIE === 1n;
      case InterruptType.Nonmaskable:
        // nonmaskable are always enabled
        return true;
    }
    return false;

  is_global_enabled: |
    return true;

  enable: |
    switch (type) {
      case InterruptType.Maskable:
        return registers.MIE = 1n;
        break;
      // we don't need to do anything for nonmaskable
    }

  disable: |
    switch (type) {
      case InterruptType.Maskable:
        registers.IE = 0n;
        break;
      // can't disable nonmaskable
    }

  global_enable: |
    registers.IE = 1n;

  global_disable: |
    registers.IE = 0n;
```

> [!NOTE]
> `enable` and `disable` are optional, they gets overriten by `global_` counterparts if it's not defined.
> `is_global_enabled` is optional, and defaults to `return true`.


Finally, we define the custom interrupt handler. This handler will disable interrupts, store `PC` in the stack, and jump to `0x0`. To disable interrupts, we do it "manually" by clearing `IE`, but we can also use [CAPI](CAPI.md) to reuse the code we already defined.

> [!NOTE]
> Using these CAPI functions is recommended way of doing it, as it allows the application to (secretly) keep track of these interrupts.

```yml
interrupts:
  handlers:
    custom: |
      // disable interrupt
      CAPI.INTERRUPTS.disable(type);

      // store PC in stack
      registers.SP = (registers.SP - 1n) & 0xFFn;
      CAPI.MEM.write(registers.SP, 1, registers.PC);

      // jump to handler
      registers.PC = 0n;
```

<!-- RETI -->

Many architectures have a specific instruction to return from an interrupt, so let's make one, `reti`. This instruction will clear and enable interrupts and jump back to the address stored in the stack:
```yml
instructions:
  base:
    # ...
    - name: reti
      template: standard
      fields:
        - field: opcode
          value: "0x01"
      definition: |
        // enable interrupts
        CAPI.INTERRUPTS.globalEnable();

        // pop return address from stack
        registers.PC = CAPI.MEM.read(registers.SP, 1);
        registers.SP = (registers.SP + 1n) & 0xFFn;

        // notify UI that handler has finished
        CAPI.INTERRUPTS.clearHighlight();
```


### CREATOR handler
As we mentioned in [Interrupt Handling](../web/execution.md#interrupt-handling), CREATOR has two different interrupt handlers: the default "CREATOR" one, and a custom architecture-defined one. We also mentioned that the default handler is able to handle "architecture-defined system calls". Let's see a more concrete example, by implemening them in our architecture.

> [!TIP]
> Why would we want this? Because we want to have our cake and eat it too.
> 
> Before interrupts were added to CREATOR, the definition of RISC-V's `ecall` function didn't create an interrupt, it just executed the desired system call depending on the value of register `a7`. But we wanted to have "real" interrupts and a "real" `ecall`. The problem is that this required an interrupt handler, and we didn't want to force our users to use it, we didn't want to silently include a kernel, and we didn't want to have two architectures: one with interrupts and one without.
> 
> So the solution (compromise) we found was this one, a second (default) interrupt handler that can be programmed in JS.


These system calls will generate a new type of interrupt (`InterruptType.EnvironmentCall`), so let's quickly modify the architecture to take them into account. We'll also add a new `EIP` register to signal that that type of interrupt is pending.
```yaml
components:
  # ...
  - name: Integer registers
    # ...
    elements:
      # ...
      - name:
          - EIP
        encoding: 2
        nbits: 1
        value: 0
        default_value: 0
        properties:
          - read
          - write

# ...
interrupts:
  check: |
    if (registers.NIP) return InterruptType.Nonmaskable;
    if (registers.MIP) return InterruptType.Maskable;
    if (registers.EIP) return InterruptType.EnvironmentCall;
    return null;

  is_enabled: |
    switch (type) {
      case InterruptType.Maskable:
        return registers.MIE === 1n;
      case InterruptType.EnvironmentCall:
        return registers.EIE === 1n;
      case InterruptType.Nonmaskable:
        // nonmaskable are always enabled
        return true;
    }
    return false;

  enable: |
    switch (type) {
      case InterruptType.Maskable:
        // we don't need to do anything for nonmaskable
        break;
      default:
        registers.IE = 1n;
        break;
    }

  disable: |
    switch (type) {
      case InterruptType.Maskable:
        // can't disable nonmaskable
        break;
      default:
        registers.IE = 0n;
        break;
    }

  create: |
    switch (type) {
      case InterruptType.Maskable:
        registers.MIP = 1n;
        break;
      case InterruptType.Nonmaskable:
        registers.NIP = 1n;
        break;
      case InterruptType.EnvironmentCall:
        registers.EIP = 1n;
        break;
    }

  clear: |
    switch (type) {
      case InterruptType.Maskable:
        registers.MIP = 0n;
        break;
      case InterruptType.Nonmaskable:
        registers.NIP = 0n;
        break;
    }

  global_clear: |
    registers.MIP = 0n;
    registers.NIP = 0n;
    registers.EIP = 0n;

  # ...
```

Now we can define our `syscall` instruction:
```yaml
# ...
instructions:
  base:
    # ...
    - name: syscall
      template: standard
      fields:
        - field: opcode
          value: "0x02"
      definition: CAPI.INTERRUPTS.create(InterruptType.EnvironmentCall);
```

The convention will be that the type of system call we want to use will be stored in register `A`, while register `B` will hold extra information. For example, a system call to print a number will print whatever `B` is holding.

To allow CREATOR's handler to handle them, as system calls depend on each architecture, we must define that in the architecture definition:
```yaml
# ...
interrupts:
  handlers:
    # ...
    creator_syscall: |
      switch (registers.A) {
        case 1n:
          CAPI.SYSCALL.print(registers.B, 'int32');
          break;
      }

      // notify UI that handler has finished
      CAPI.INTERRUPTS.clearHighlight();
```


## Privileged instructions
CREATOR also supports having privileged instructions that can only be executed in kernel mode, by adding the `privileged` property. This is the reason for having system calls in the first place, we allow the user to ask doing things that require a higher privilege (e.g. accessing I/O) without giving them that privilege itself.

Let's say that in our architecture, an interrupt always triggers an execution mode change. To achieve that, we should modify the custom handler so that it sets kernel mode (`CAPI.INTERRUPTS.setKernelMode();`) and modify the `reti` instruction so that it goes back to user mode (`CAPI.INTERRUPTS.setUserMode();`). As `reti` should only be used while dealing with interrupts, we'll make it a privileged instruction.

```yaml
instructions:
  base:
    # ...
    - name: reti
      # ...
      properties:
        - privileged
      definition: |
        // enable interrupts
        CAPI.INTERRUPTS.globalEnable();

        // pop return address from stack
        registers.PC = CAPI.MEM.read(registers.SP, 1);
        registers.SP = (registers.SP + 1n) & 0xFFn;

        // notify UI that handler has finished
        CAPI.INTERRUPTS.clearHighlight();

        // set user mode
        CAPI.INTERRUPTS.setUserMode();

# ...

interrupts:
  handlers:
    custom: |
      // disable interrupt
      CAPI.INTERRUPTS.disable(type);

      // set kernel mode
      CAPI.INTERRUPTS.setKernelMode();

      // store PC in stack
      registers.SP = (registers.SP - 1n) & 0xFFn;
      CAPI.MEM.write(registers.SP, 1, registers.PC);

      // jump to handler
      registers.PC = 0n;
```


## Timers



## Devices
