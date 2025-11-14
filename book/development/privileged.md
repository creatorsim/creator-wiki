# Privileged Instructions and Execution Modes

CREATOR supports privilege levels to simulate operating system and user space separation, similar to real processors.

## Execution Modes

**Source**: `src/core/executor/interrupts.mts` - `ExecutionMode` enum

CREATOR defines two execution modes:

```typescript
enum ExecutionMode {
  User = 0,    // Non-privileged mode
  Kernel = 1   // Privileged mode
}
```

### Mode Tracking

The current mode is tracked by `currentExecutionMode`:

```javascript
let currentExecutionMode = ExecutionMode.User;  // Initial mode

function setExecutionMode(mode: ExecutionMode): void {
  currentExecutionMode = mode;
}

function getExecutionMode(): ExecutionMode {
  return currentExecutionMode;
}

function isKernelMode(): boolean {
  return currentExecutionMode === ExecutionMode.Kernel;
}

function isUserMode(): boolean {
  return currentExecutionMode === ExecutionMode.User;
}
```

## Privileged Instructions

### Definition

An instruction is privileged if it has the `privileged` property set to `true` in its definition:

```yaml
# Architecture YAML
instructions:
  - name: "mret"
    format: "R"
    opcode: 0x73
    funct3: 0x0
    funct7: 0x18
    privileged: true      # This instruction requires kernel mode
    behavior: |
      pc = mepc;                              // Return to saved PC
      currentExecutionMode = ExecutionMode.User;  // Back to user mode
    description: "Return from machine mode trap"
```

### Privileged Instruction Check

Before executing any instruction, CREATOR checks privilege requirements:

```javascript
function execute_one_instruction() {
  // Fetch and decode
  const instruction = fetch_and_decode();
  
  // Check privilege level
  if (instruction.privileged && isUserMode()) {
    // Illegal instruction exception
    throw new PrivilegeViolationError(
      `Cannot execute privileged instruction '${instruction.name}' in user mode`
    );
  }
  
  // Execute if allowed
  instruction.handler(state, instruction.operands);
}
```

## Mode Transitions

### User to Kernel (Trap)

Transitions to kernel mode occur on:

1. **Exceptions**: Illegal instructions, page faults, etc.
2. **Interrupts**: Timer, external devices
3. **System Calls**: `ecall` instruction

```javascript
function handleTrap(cause: number): void {
  // Save current PC
  set_mepc(get_pc());
  
  // Save trap cause
  set_mcause(cause);
  
  // Switch to kernel mode
  setExecutionMode(ExecutionMode.Kernel);
  
  // Disable interrupts
  disableInterrupts();
  
  // Jump to trap handler
  const handler_addr = get_mtvec();
  set_pc(handler_addr);
}
```

### Kernel to User (Return)

Return to user mode with privileged instructions:

```javascript
// RISC-V: mret (Machine Return)
function execute_mret() {
  // Restore PC
  const return_addr = get_mepc();
  set_pc(return_addr);
  
  // Return to user mode
  setExecutionMode(ExecutionMode.User);
  
  // Re-enable interrupts (if they were enabled)
  const prev_mie = (get_mstatus() >> 7) & 0x1;
  if (prev_mie) {
    enableInterrupts();
  }
}

// MIPS: eret (Exception Return)
function execute_eret() {
  // Similar to mret
  set_pc(get_epc());
  setExecutionMode(ExecutionMode.User);
  
  // Clear exception flag
  const status = get_status();
  set_status(status & ~0x2);  // Clear EXL bit
}
```

## Common Privileged Instructions

### RISC-V Privileged Instructions

#### Machine Trap Return (mret)

```yaml
- name: "mret"
  privileged: true
  behavior: |
    // Restore PC from mepc
    pc = mepc;
    
    // Restore interrupt enable (MPIE -> MIE)
    const mpie = (mstatus >> 7) & 0x1;
    mstatus = (mstatus & ~0x8) | (mpie << 3);
    
    // Set MPIE to 1
    mstatus = mstatus | 0x80;
    
    // Return to user mode
    currentExecutionMode = ExecutionMode.User;
```

#### Supervisor Trap Return (sret)

```yaml
- name: "sret"
  privileged: true
  behavior: |
    pc = sepc;
    const spie = (sstatus >> 5) & 0x1;
    sstatus = (sstatus & ~0x2) | (spie << 1);
    sstatus = sstatus | 0x20;
    currentExecutionMode = ExecutionMode.User;
```

#### CSR Instructions

Control and Status Register (CSR) access is often privileged:

```yaml
- name: "csrrw"
  privileged: true  # Some CSRs are privileged
  behavior: |
    const csr_addr = imm & 0xFFF;
    
    // Check CSR access permissions
    if (isPrivilegedCSR(csr_addr) && isUserMode()) {
      makeInterrupt(InterruptType.IllegalInstruction);
      return;
    }
    
    // Read CSR
    const old_value = readCSR(csr_addr);
    
    // Write CSR
    writeCSR(csr_addr, rs1);
    
    // Write old value to rd
    rd = old_value;
```

### MIPS Privileged Instructions

#### Exception Return (eret)

```yaml
- name: "eret"
  privileged: true
  behavior: |
    pc = epc;
    const status = readCP0(12);  // Status register
    writeCP0(12, status & ~0x2); // Clear EXL bit
    currentExecutionMode = ExecutionMode.User;
```

#### Move to CP0 (mtc0)

```yaml
- name: "mtc0"
  privileged: true
  behavior: |
    const cp0_reg = rd;  // CP0 register number
    const value = rs1;
    writeCP0(cp0_reg, value);
```

#### Move from CP0 (mfc0)

```yaml
- name: "mfc0"
  privileged: true
  behavior: |
    const cp0_reg = rd;
    const value = readCP0(cp0_reg);
    rs1 = value;
```

## Control and Status Registers (CSRs)

### CSR Categories

CSRs are classified by privilege level:

```javascript
const CSR_CATEGORIES = {
  // Machine-mode CSRs (0x300-0x3FF)
  MACHINE: {
    min: 0x300,
    max: 0x3FF,
    required_mode: ExecutionMode.Kernel
  },
  
  // Supervisor-mode CSRs (0x100-0x1FF)
  SUPERVISOR: {
    min: 0x100,
    max: 0x1FF,
    required_mode: ExecutionMode.Kernel
  },
  
  // User-mode CSRs (0x000-0x0FF)
  USER: {
    min: 0x000,
    max: 0x0FF,
    required_mode: ExecutionMode.User
  }
};

function isPrivilegedCSR(csr_addr: number): boolean {
  // Check privilege level encoded in CSR address
  const privilege = (csr_addr >> 8) & 0x3;
  
  // 0=user, 1=supervisor, 2=hypervisor, 3=machine
  return privilege > 0;
}

function checkCSRAccess(csr_addr: number): void {
  const privilege = (csr_addr >> 8) & 0x3;
  
  if (privilege === 3 && !isKernelMode()) {
    throw new PrivilegeViolationError(
      `Cannot access machine CSR 0x${csr_addr.toString(16)} in user mode`
    );
  }
}
```

### Common CSRs

**Machine-Mode CSRs**:
```
0x300  mstatus   - Machine status
0x301  misa      - ISA and extensions
0x304  mie       - Machine interrupt enable
0x305  mtvec     - Machine trap-handler base address
0x340  mscratch  - Machine scratch register
0x341  mepc      - Machine exception program counter
0x342  mcause    - Machine trap cause
0x343  mtval     - Machine trap value
0x344  mip       - Machine interrupt pending
```

**User-Mode CSRs**:
```
0xC00  cycle     - Cycle counter (read-only)
0xC01  time      - Timer (read-only)
0xC02  instret   - Instructions retired (read-only)
```

## Implementing Privilege Checks

### Architecture Definition

In your architecture YAML:

```yaml
instructions:
  # Privileged instruction
  - name: "csrw"
    privileged: true
    behavior: |
      // Privilege check is automatic
      // This code only runs if check passes
      const csr_addr = imm & 0xFFF;
      
      // Additional CSR-specific check
      if (isPrivilegedCSR(csr_addr) && isUserMode()) {
        makeInterrupt(InterruptType.IllegalInstruction);
        return;
      }
      
      writeCSR(csr_addr, rs1);
  
  # Non-privileged instruction
  - name: "add"
    privileged: false  # or omit (default is false)
    behavior: |
      rd = rs1 + rs2;
```

### Runtime Checks

```javascript
// Custom privilege validation
function validatePrivilege(instruction) {
  if (!instruction.privileged) {
    return true;  // Always allowed
  }
  
  if (isKernelMode()) {
    return true;  // Kernel can execute anything
  }
  
  // User mode cannot execute privileged instruction
  throw new PrivilegeViolationError(
    `Privilege violation: ${instruction.name} requires kernel mode`
  );
}
```

## Exception Handling

### Privilege Violations

```javascript
class PrivilegeViolationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "PrivilegeViolationError";
  }
}

function handlePrivilegeViolation(error: PrivilegeViolationError): void {
  console.error(`Privilege violation at PC=0x${get_pc().toString(16)}`);
  console.error(`Current mode: ${isKernelMode() ? 'Kernel' : 'User'}`);
  console.error(error.message);
  
  // Trigger illegal instruction trap
  makeInterrupt(InterruptType.IllegalInstruction);
}
```

## System Call Mechanism

System calls provide controlled transition from user to kernel mode:

```javascript
// ECALL instruction (RISC-V)
function execute_ecall() {
  // Save context
  set_mepc(get_pc() + 4n);  // Save return address
  
  // Set trap cause
  const cause = isKernelMode() ? 11 : 8;  // M-mode or U-mode ecall
  set_mcause(cause);
  
  // Switch to kernel mode
  setExecutionMode(ExecutionMode.Kernel);
  
  // Disable interrupts
  disableInterrupts();
  
  // Jump to trap handler
  const handler = get_mtvec();
  set_pc(handler);
}

// Trap handler (in kernel mode)
function trapHandler() {
  const cause = get_mcause();
  
  if (cause === 8) {
    // User-mode ecall - handle syscall
    const syscall_num = readRegister('a7');
    handleSyscall(syscall_num);
  }
  
  // Return to user mode
  execute_mret();
}
```

## Debugging Privileged Code

### Mode Tracking

```bash
# CLI commands
creator> reg execution_mode
execution_mode = User

creator> reg mstatus
mstatus = 0x00000088  # MIE=1, MPIE=1
```

### Privilege Violations

```bash
creator> step
Error: Privilege violation
Instruction: mret
Current mode: User
PC: 0x00400010
```

## Best Practices

1. **Minimal Privileged Instructions**: Only mark truly privileged operations
2. **Clear Mode Transitions**: Document when and why mode changes
3. **Validate CSR Access**: Check both instruction and CSR privilege levels
4. **Proper Context Saving**: Always save PC and status before mode switch
5. **Interrupt Handling**: Disable interrupts during mode transitions
6. **Testing**: Test both user and kernel mode paths

## Example: Complete Privilege System

```yaml
# Architecture with privilege support
name: "Privileged RISC-V"

# ... registers, memory, etc. ...

instructions:
  # User-mode instruction
  - name: "add"
    format: "R"
    privileged: false
    behavior: "rd = rs1 + rs2;"
  
  # Trap to kernel
  - name: "ecall"
    format: "I"
    privileged: false  # Users can call ecall
    behavior: |
      mepc = pc + 4;
      mcause = isKernelMode() ? 11 : 8;
      currentExecutionMode = ExecutionMode.Kernel;
      disableInterrupts();
      pc = mtvec;
  
  # Return to user
  - name: "mret"
    format: "R"
    privileged: true  # Only kernel can return
    behavior: |
      pc = mepc;
      const mpie = (mstatus >> 7) & 0x1;
      mstatus = (mstatus & ~0x8) | (mpie << 3);
      currentExecutionMode = ExecutionMode.User;
```

## Next Steps

- Review [Interrupts](interrupts.md) for trap handling details
- See [Devices](devices.md) for system call implementation
- Read [Core Architecture](core-architecture.md) for system integration
- Check [Custom Architectures](custom-architectures.md) for privilege configuration