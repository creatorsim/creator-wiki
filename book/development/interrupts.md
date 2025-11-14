# Interrupts

CREATOR supports a comprehensive interrupt system for simulating asynchronous events and exception handling. This includes software interrupts, timer interrupts, external interrupts, and environment calls.

## Interrupt Architecture

### Overview

The interrupt system provides:
- **Multiple interrupt types**: Software, timer, external, environment calls
- **Privilege levels**: User mode and kernel mode execution
- **Handler management**: Configurable interrupt handlers
- **State preservation**: Automatic state saving on interrupt
- **Flexible configuration**: Architecture-defined behavior

Source: `src/core/executor/interrupts.mts`

## Interrupt Types

```typescript
export enum InterruptType {
    Software = "SOFTWARE",
    Timer = "TIMER",
    External = "EXTERNAL",
    EnvironmentCall = "ECALL",
}
```

### Software Interrupts
- Triggered by software explicitly
- Used for inter-process signaling
- Can be masked/unmasked

### Timer Interrupts
- Periodic interrupts from timer device
- Used for task switching, scheduling
- Configurable interval

### External Interrupts
- Triggered by external devices
- I/O completion, hardware events
- Asynchronous to program execution

### Environment Calls (ECALL)
- Synchronous system calls
- Transition from user to kernel mode
- Access OS services

## Execution Modes

```typescript
export enum ExecutionMode {
    User = 0,    // Unprivileged mode
    Kernel = 1,  // Privileged mode
}
```

**User Mode**:
- Normal program execution
- Restricted instruction set
- Cannot directly access devices
- Uses system calls for OS services

**Kernel Mode**:
- Privileged execution
- Full instruction access
- Direct device access
- Handles interrupts and exceptions

## Interrupt Handling Flow

### 1. Interrupt Occurs

An interrupt is triggered by:
- Device operation
- Software instruction
- Timer expiration
- Program exception
- System call (ecall)

### 2. Check if Interrupts Enabled

```typescript
export function checkInterrupt(): InterruptType | null {
    if (!status.interrupts_enabled) return null;
    return interruptCheck(InterruptType);
}
```

### 3. Handler Execution

If interrupt detected:

```typescript
export function handleInterrupt() {
    console_log("Interruption detected");
    
    // Switch to kernel mode
    status.execution_mode = ExecutionMode.Kernel;

    // Save current PC to EPC (Exception Program Counter)
    const epc_reg = crex_findReg_bytag("exception_program_counter");
    const pc_reg = crex_findReg_bytag("program_counter");
    
    if (epc_reg.match === 1) {
        const pc_value = readRegister(pc_reg.indexComp, pc_reg.indexElem);
        writeRegister(pc_value, epc_reg.indexComp, epc_reg.indexElem);
    }

    // Jump to interrupt handler
    const handler_address = interruptGetHandlerAddr(InterruptType);
    writeRegister(
        BigInt(handler_address),
        pc_reg.indexComp,
        pc_reg.indexElem
    );

    // Clear the interrupt
    interruptClear(InterruptType);
}
```

### 4. Handler Code Executes

Interrupt handler (in assembly):
- Saves additional context if needed
- Handles the interrupt
- Restores context
- Returns via special instruction (e.g., `mret`, `sret`)

### 5. Return to Normal Execution

- PC restored from EPC
- Mode switched back to User
- Execution continues where interrupted

## Architecture-Defined Functions

Interrupt behavior is defined in architecture YAML files. CREATOR compiles these into JavaScript functions:

```typescript
export function compileInterruptFunctions() {
    if (!architecture.interrupts) return;
    
    interruptEnable = new Function(
        "InterruptType",
        architecture.interrupts.enable
    );
    
    interruptDisable = new Function(
        "InterruptType",
        architecture.interrupts.disable
    );
    
    interruptCheck = new Function(
        "InterruptType",
        architecture.interrupts.check
    );
    
    interruptCreate = new Function(
        "InterruptType",
        "type",
        architecture.interrupts.create
    );
    
    interruptGetHandlerAddr = new Function(
        "InterruptType",
        architecture.interrupts.get_handler_addr
    );
    
    interruptClear = new Function(
        "InterruptType",
        architecture.interrupts.clear
    );
}
```

## Control Functions

### Enable Interrupts

```typescript
export function enableInterrupts() {
    status.interrupts_enabled = true;
    return interruptEnable(InterruptType);
}
```

**Usage in Assembly** (RISC-V):
```assembly
csrsi mstatus, 0x8    # Set MIE bit
```

### Disable Interrupts

```typescript
export function disableInterrupts() {
    status.interrupts_enabled = false;
    return interruptDisable(InterruptType);
}
```

**Usage in Assembly** (RISC-V):
```assembly
csrci mstatus, 0x8    # Clear MIE bit
```

### Trigger Interrupt

```typescript
export function makeInterrupt(type: InterruptType): void {
    return interruptCreate(InterruptType, type);
}
```

**Usage** (from device or internal code):
```typescript
makeInterrupt(InterruptType.Timer);
```

### Check for Pending Interrupt

```typescript
export function checkInterrupt(): InterruptType | null {
    if (!status.interrupts_enabled) return null;
    return interruptCheck(InterruptType);
}
```

Called each instruction cycle.

## Example: RISC-V Interrupt Configuration

### Architecture YAML Definition

```yaml
interrupts:
  enable: |
    // Enable interrupts by setting MIE bit in mstatus
    const mstatus = readRegister(/* mstatus index */);
    writeRegister(mstatus | 0x8n, /* mstatus index */);
  
  disable: |
    // Disable interrupts by clearing MIE bit
    const mstatus = readRegister(/* mstatus index */);
    writeRegister(mstatus & ~0x8n, /* mstatus index */);
  
  check: |
    // Check if any interrupt is pending and enabled
    const mstatus = readRegister(/* mstatus index */);
    const mie = readRegister(/* mie index */);
    const mip = readRegister(/* mip index */);
    
    if ((mstatus & 0x8n) === 0n) return null;  // Interrupts disabled
    
    const pending = mip & mie;
    if (pending & 0x80n) return InterruptType.Timer;
    if (pending & 0x800n) return InterruptType.External;
    if (pending & 0x8n) return InterruptType.Software;
    return null;
  
  create: |
    // Set interrupt pending bit
    const mip = readRegister(/* mip index */);
    if (type === InterruptType.Timer) {
      writeRegister(mip | 0x80n, /* mip index */);
    } else if (type === InterruptType.External) {
      writeRegister(mip | 0x800n, /* mip index */);
    } else if (type === InterruptType.Software) {
      writeRegister(mip | 0x8n, /* mip index */);
    }
  
  get_handler_addr: |
    // Return address of interrupt handler from mtvec
    const mtvec = readRegister(/* mtvec index */);
    return Number(mtvec & ~0x3n);  // Clear mode bits
  
  clear: |
    // Clear interrupt pending bit
    const mip = readRegister(/* mip index */);
    writeRegister(mip & ~0x888n, /* mip index */);  // Clear all interrupt bits
```

### Interrupt Handler (Assembly)

```assembly
.text
.align 4
interrupt_handler:
    # Save context (not shown - save all registers to stack)
    
    # Determine interrupt cause
    csrr t0, mcause
    
    # Check if it's a timer interrupt (bit 31 set, code 7)
    li t1, 0x80000007
    beq t0, t1, timer_handler
    
    # Check for external interrupt
    li t1, 0x8000000B
    beq t0, t1, external_handler
    
    # Default handler
    j unknown_interrupt

timer_handler:
    # Handle timer interrupt
    # ...
    j restore_context

external_handler:
    # Handle external interrupt
    # ...
    j restore_context

unknown_interrupt:
    # Handle unknown interrupt
    # ...

restore_context:
    # Restore context (not shown)
    
    # Return from interrupt
    mret
```

## System Calls (ECALL)

### ECALL Flow

1. **Program executes ecall**:
```assembly
li a7, 1      # System call number
li a0, 42     # Argument
ecall         # Trigger environment call
```

2. **Interrupt handler invoked**:
- Mode switches to Kernel
- PC saved to EPC
- Jump to handler address

3. **Handler identifies syscall**:
```assembly
interrupt_handler:
    # Check if it's an ecall (mcause = 11 for U-mode ecall)
    csrr t0, mcause
    li t1, 11
    bne t0, t1, not_ecall
    
    # Read system call number from a7
    # Dispatch to appropriate handler
    beq a7, x0, sys_exit
    li t0, 1
    beq a7, t0, sys_print_int
    # ... more system calls
```

4. **System call executes**:
- Accesses kernel resources
- Performs operation
- Sets return value in a0

5. **Return to user mode**:
```assembly
    # Execution continues after ecall
    mret
```

## Implementing Custom Interrupts

### Step 1: Define Interrupt Type

Add to architecture interrupts configuration or use existing types.

### Step 2: Interrupt Source

Create interrupt source (e.g., device):

```typescript
class InterruptingDevice extends Device {
    handler(): void {
        // ... device logic ...
        
        // Trigger interrupt when condition met
        if (shouldInterrupt) {
            makeInterrupt(InterruptType.External);
        }
    }
}
```

### Step 3: Handler Code

Write interrupt handler in assembly:

```assembly
.align 4
custom_interrupt_handler:
    # Save registers
    addi sp, sp, -64
    sw ra, 0(sp)
    sw t0, 4(sp)
    # ... save more registers
    
    # Handle interrupt
    # ... custom logic ...
    
    # Restore registers
    lw ra, 0(sp)
    lw t0, 4(sp)
    # ... restore more registers
    addi sp, sp, 64
    
    # Return
    mret
```

### Step 4: Register Handler

Configure handler address in appropriate CSR:

```assembly
.text
main:
    # Set interrupt handler address
    la t0, custom_interrupt_handler
    csrw mtvec, t0
    
    # Enable interrupts
    li t0, 0x8
    csrs mstatus, t0
    
    # ... rest of program ...
```

## Debugging Interrupts

### CLI Commands

**Check interrupt state**:
```
CREATOR> reg mstatus
CREATOR> reg mie
CREATOR> reg mip
```

**View handler address**:
```
CREATOR> reg mtvec
```

**Inspect EPC after interrupt**:
```
CREATOR> reg mepc
```

### Common Issues

**Interrupts not firing**:
- Check `status.interrupts_enabled`
- Verify interrupt enable bits in CSRs
- Confirm interrupt pending bit set
- Ensure handler address is valid

**Handler not executing**:
- Check mtvec points to valid address
- Verify handler code is loaded in memory
- Ensure alignment requirements met

**Wrong interrupt type handled**:
- Check mcause value
- Verify interrupt priority logic
- Validate pending bit checks

## Best Practices

### Handler Design

1. **Save Context**: Preserve all registers used
2. **Minimal Work**: Keep handlers short and fast
3. **Re-entrancy**: Handle nested interrupts if supported
4. **Clear Source**: Clear interrupt source before returning
5. **Error Handling**: Validate interrupt source

### Interrupt Management

1. **Critical Sections**: Disable interrupts for atomic operations
2. **Priority**: Handle high-priority interrupts first
3. **Latency**: Minimize interrupt handling time
4. **Testing**: Test all interrupt paths thoroughly

### Performance

1. **Handler Location**: Place in fast memory
2. **Inline Critical Code**: Avoid function calls in handler
3. **Deferred Work**: Move heavy processing out of handler
4. **Interrupt Coalescing**: Batch similar interrupts

## Architecture-Specific Details

### RISC-V

**CSRs**:
- `mstatus`: Machine status (MIE bit)
- `mie`: Machine interrupt enable
- `mip`: Machine interrupt pending
- `mtvec`: Interrupt handler address
- `mepc`: Exception program counter
- `mcause`: Exception/interrupt cause

**Interrupt Codes** (mcause):
- Software: 3
- Timer: 7
- External: 11

**Return Instructions**:
- `mret`: Return from machine mode
- `sret`: Return from supervisor mode
- `uret`: Return from user mode (if N extension)

### MIPS

**Registers**:
- `Status`: Interrupt enable
- `Cause`: Interrupt cause
- `EPC`: Exception PC

**Instructions**:
- `syscall`: Trigger system call
- `eret`: Return from exception

## Next Steps

- Read [Devices](devices.md) for interrupt-driven I/O
- Study [Privileged Instructions](privileged.md) for mode switching
- See [Execution Engine](execution-engine.md) for integration
- Review [Core Architecture](core-architecture.md) for system overview
