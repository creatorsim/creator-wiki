# Creating Custom Architectures

CREATOR supports defining custom architectures through YAML configuration files. This allows adding new instruction sets or modifying existing ones.

## Architecture YAML Structure

### Complete Example

```yaml
name: "Custom RISC-V"
bits: 32
word_size: 4
endianness: little

# Register definitions
registers:
  int_registers:
    - { name: "zero", alias: ["x0"], default_value: 0, immutable: true }
    - { name: "ra", alias: ["x1"], default_value: 0 }
    - { name: "sp", alias: ["x2"], default_value: 0x7FFFEFFC }
    # ... more registers
  
  fp_registers:
    - { name: "ft0", alias: ["f0"], default_value: 0.0 }
    - { name: "ft1", alias: ["f1"], default_value: 0.0 }
    # ... more FP registers
  
  special_registers:
    - { name: "pc", default_value: 0x00400000 }
    - { name: "mstatus", default_value: 0 }
    - { name: "mtvec", default_value: 0x80000000 }
    # ... more CSRs

# Memory layout
memory:
  text:
    start: 0x00400000
    end: 0x0FFFFFFF
    permissions: "r-x"
  data:
    start: 0x10010000
    end: 0x7FFFFFFF
    permissions: "rw-"
  stack:
    start: 0x7FFFEFFC
    end: 0x70000000
    permissions: "rw-"
    grows: "down"

# Instruction definitions
instructions:
  - name: "add"
    format: "R"
    opcode: 0x33
    funct3: 0x0
    funct7: 0x00
    syntax: "add rd, rs1, rs2"
    description: "Add two registers"
    behavior: |
      rd = rs1 + rs2;
    cycles: 1
    
  - name: "addi"
    format: "I"
    opcode: 0x13
    funct3: 0x0
    syntax: "addi rd, rs1, imm"
    description: "Add immediate"
    behavior: |
      rd = rs1 + imm;
    cycles: 1
  
  # ... more instructions

# Interrupt configuration (optional)
interrupts:
  enable: |
    mstatus = mstatus | 0x8;  // MIE bit
  
  disable: |
    mstatus = mstatus & ~0x8;
  
  check: |
    return (mstatus & 0x8) && (mip & mie);
  
  create: |
    // type: InterruptType (0=software, 1=timer, 2=external, 3=ecall)
    switch(type) {
      case 0: mip = mip | 0x8; break;    // MSIP
      case 1: mip = mip | 0x80; break;   // MTIP
      case 2: mip = mip | 0x800; break;  // MEIP
      case 3: mcause = 11; break;        // ECALL
    }
  
  get_handler_addr: |
    return mtvec;
  
  clear: |
    mip = 0;

# Timer configuration (optional)
timer:
  tick_cycles: 1000
  is_enabled: |
    return (mstatus & 0x8) !== 0;
  advance: |
    if (mtime !== undefined) {
      mtime++;
      if (mtime >= mtimecmp) {
        mip = mip | 0x80;  // Set MTIP
      }
    }
  handler: |
    // Called when timer interrupt fires
    handleInterrupt(1);  // Timer interrupt
```

## Sections Explained

### 1. Basic Information

```yaml
name: "Architecture Name"
bits: 32              # Architecture bit width (8, 16, 32, 64)
word_size: 4          # Bytes per word (1, 2, 4, 8)
endianness: little    # "little" or "big"
```

### 2. Register Definitions

#### Integer Registers

```yaml
registers:
  int_registers:
    - name: "zero"                    # Primary name
      alias: ["x0", "r0"]            # Alternative names
      default_value: 0                # Initial value
      immutable: true                 # Cannot be modified (optional)
      
    - name: "ra"
      alias: ["x1", "return_addr"]
      default_value: 0
      
    - name: "sp"
      alias: ["x2", "stack_ptr"]
      default_value: 0x7FFFEFFC      # Initial stack pointer
```

**Properties**:
- `name`: Primary register name
- `alias`: List of alternative names
- `default_value`: Initial value on reset
- `immutable`: If true, writes are ignored (like RISC-V x0)

#### Floating-Point Registers

```yaml
fp_registers:
  - name: "ft0"
    alias: ["f0"]
    default_value: 0.0    # Float or double
    
  - name: "ft1"
    alias: ["f1"]
    default_value: 0.0
```

#### Special/Control Registers

```yaml
special_registers:
  - name: "pc"
    default_value: 0x00400000
    
  - name: "mstatus"      # Machine status
    default_value: 0
    
  - name: "mtvec"        # Trap vector
    default_value: 0x80000000
    
  - name: "mepc"         # Exception PC
    default_value: 0
```

### 3. Memory Layout

```yaml
memory:
  text:
    start: 0x00400000
    end: 0x0FFFFFFF
    permissions: "r-x"    # Read, execute only
    
  data:
    start: 0x10010000
    end: 0x7FFFFFFF
    permissions: "rw-"    # Read, write
    
  stack:
    start: 0x7FFFEFFC     # Initial SP
    end: 0x70000000
    permissions: "rw-"
    grows: "down"         # or "up"
```

**Permission strings**:
- `r`: Readable
- `w`: Writable
- `x`: Executable
- `-`: Not allowed

### 4. Instruction Definitions

#### Instruction Fields

```yaml
instructions:
  - name: "add"               # Instruction name
    format: "R"               # Format type (R, I, S, B, U, J, or custom)
    opcode: 0x33             # Opcode value
    funct3: 0x0              # Function code 3 (optional)
    funct7: 0x00             # Function code 7 (optional)
    syntax: "add rd, rs1, rs2"  # Assembly syntax
    description: "Add registers"  # Human description
    behavior: |              # JavaScript behavior
      rd = rs1 + rs2;
    cycles: 1                # Execution cycles
```

#### Instruction Formats

**R-Type (Register)**:
```yaml
- name: "add"
  format: "R"
  fields:
    opcode: { bits: [6, 0] }
    rd: { bits: [11, 7] }
    funct3: { bits: [14, 12] }
    rs1: { bits: [19, 15] }
    rs2: { bits: [24, 20] }
    funct7: { bits: [31, 25] }
  behavior: "rd = rs1 + rs2;"
```

**I-Type (Immediate)**:
```yaml
- name: "addi"
  format: "I"
  fields:
    opcode: { bits: [6, 0] }
    rd: { bits: [11, 7] }
    funct3: { bits: [14, 12] }
    rs1: { bits: [19, 15] }
    imm: { bits: [31, 20], signed: true }
  behavior: "rd = rs1 + imm;"
```

**S-Type (Store)**:
```yaml
- name: "sw"
  format: "S"
  fields:
    opcode: { bits: [6, 0] }
    imm_low: { bits: [11, 7] }
    funct3: { bits: [14, 12] }
    rs1: { bits: [19, 15] }
    rs2: { bits: [24, 20] }
    imm_high: { bits: [31, 25] }
  immediate: "imm = (imm_high << 5) | imm_low"
  behavior: |
    const addr = rs1 + imm;
    memory.writeWord(addr, rs2);
```

**Custom Format**:
```yaml
- name: "custom_instr"
  format: "Custom"
  encoding:
    opcode: 0x7F
    field1: { bits: [11, 7], type: "register" }
    field2: { bits: [19, 12], type: "immediate", signed: false }
    field3: { bits: [31, 20], type: "immediate", signed: true }
  behavior: |
    // Custom behavior
    const result = field1 + field2 - field3;
    rd = result;
```

#### Behavior Syntax

Behavior is JavaScript code with access to:

**Available Variables**:
- `rd`, `rs1`, `rs2`: Register indices
- `imm`: Immediate value
- `pc`: Program counter
- `memory`: Memory object
- Register names: `ra`, `sp`, `t0`, etc.

**Available Functions**:
```javascript
// Register access
readRegister(name)
writeRegister(name, value)

// Memory access
memory.readByte(address)
memory.readWord(address)
memory.writeByte(address, value)
memory.writeWord(address, value)

// Control flow
set_pc(address)
get_pc()

// Interrupts
makeInterrupt(type)
enableInterrupts()
disableInterrupts()

// Helpers
signExtend(value, bits)
zeroExtend(value, bits)
```

**Example Behaviors**:

```javascript
// ALU operation
"rd = rs1 + rs2;"

// Load instruction
"rd = memory.readWord(rs1 + imm);"

// Store instruction
"memory.writeWord(rs1 + imm, rs2);"

// Branch
`if (rs1 === rs2) {
  pc = pc + imm;
} else {
  pc = pc + 4;
}`

// Jump and link
`rd = pc + 4;
pc = pc + imm;`

// System call
`makeInterrupt(InterruptType.EnvironmentCall);
pc = pc + 4;`
```

### 5. Interrupt Configuration

```yaml
interrupts:
  # Enable interrupts
  enable: |
    mstatus = mstatus | 0x8;  // Set MIE bit
  
  # Disable interrupts
  disable: |
    mstatus = mstatus & ~0x8;  // Clear MIE bit
  
  # Check if interrupt should fire
  check: |
    // Return true if interrupt pending and enabled
    return (mstatus & 0x8) && (mip & mie);
  
  # Create interrupt
  create: |
    // type: 0=software, 1=timer, 2=external, 3=ecall
    switch(type) {
      case 0: mip |= 0x8; break;    // Software
      case 1: mip |= 0x80; break;   // Timer
      case 2: mip |= 0x800; break;  // External
      case 3: 
        mcause = 11;                 // ECALL from M-mode
        break;
    }
  
  # Get interrupt handler address
  get_handler_addr: |
    return mtvec;
  
  # Clear interrupt flags
  clear: |
    mip = 0;
```

### 6. Timer Configuration

```yaml
timer:
  tick_cycles: 1000        # Ticks every N cycles
  
  is_enabled: |
    return (mstatus & 0x8) !== 0;  // Check if timer enabled
  
  advance: |
    // Increment timer
    if (mtime !== undefined) {
      mtime++;
      
      // Check for timer interrupt
      if (mtime >= mtimecmp) {
        mip |= 0x80;  // Set MTIP bit
      }
    }
  
  handler: |
    // Called when timer fires
    handleInterrupt(1);  // Fire timer interrupt
```

## Complete Architecture Example

### Minimal 8-bit Architecture

```yaml
name: "Simple 8-bit"
bits: 8
word_size: 1
endianness: little

registers:
  int_registers:
    - { name: "a", alias: ["r0"], default_value: 0 }
    - { name: "b", alias: ["r1"], default_value: 0 }
    - { name: "c", alias: ["r2"], default_value: 0 }
  
  special_registers:
    - { name: "pc", default_value: 0 }
    - { name: "flags", default_value: 0 }

memory:
  text:
    start: 0x0000
    end: 0x7FFF
    permissions: "r-x"
  data:
    start: 0x8000
    end: 0xFFFF
    permissions: "rw-"

instructions:
  - name: "add"
    format: "R"
    opcode: 0x01
    syntax: "add a, b"
    description: "A = A + B"
    behavior: "a = (a + b) & 0xFF;"
    cycles: 1
  
  - name: "mov"
    format: "I"
    opcode: 0x02
    syntax: "mov a, #imm"
    description: "A = immediate"
    behavior: "a = imm & 0xFF;"
    cycles: 1
  
  - name: "jmp"
    format: "J"
    opcode: 0x03
    syntax: "jmp addr"
    description: "Jump to address"
    behavior: "pc = imm;"
    cycles: 2
```

## Loading Custom Architectures

### CLI

```bash
creator6 -a path/to/custom.yml program.s
```

### Web

Upload architecture YAML in the architecture selector.

### Programmatic

```javascript
import { creator_init } from 'creator-core';
import yaml from 'js-yaml';
import fs from 'fs';

const archYAML = fs.readFileSync('custom.yml', 'utf8');
const architecture = yaml.load(archYAML);

const simulator = creator_init(architecture);
```

## Validation

CREATOR validates architecture definitions:

### Required Fields

- `name`: Architecture name
- `bits`: Bit width
- `word_size`: Bytes per word
- `registers`: At least int_registers
- `memory`: At least one segment
- `instructions`: At least one instruction

### Common Errors

**Missing opcode**:
```
Error: Instruction 'add' missing opcode
```

**Invalid register reference**:
```
Error: Unknown register 'x99' in instruction behavior
```

**Invalid memory layout**:
```
Error: Memory segments overlap: text and data
```

**Syntax error in behavior**:
```
Error: Invalid JavaScript in behavior for 'add':
  Unexpected token '='
```

## Testing Custom Architectures

```javascript
// Test basic functionality
function testArchitecture(arch) {
  const sim = creator_init(arch);
  
  // Test register access
  sim.set_register('a', 42);
  assert(sim.get_register('a') === 42);
  
  // Test memory access
  sim.memory.writeByte(0x8000, 0x55);
  assert(sim.memory.readByte(0x8000) === 0x55);
  
  // Test instruction execution
  sim.assemble('add a, b');
  sim.execute_one_instruction();
  // Verify result
  
  console.log('Architecture tests passed!');
}
```

## Best Practices

1. **Start Simple**: Begin with minimal instruction set
2. **Test Incrementally**: Add instructions one at a time
3. **Document Behavior**: Clear comments in behavior code
4. **Validate Thoroughly**: Test edge cases
5. **Follow Conventions**: Use standard register/memory layouts where possible

## Next Steps

- Review [Core Architecture](core-architecture.md) for integration details
- See [Execution Engine](execution-engine.md) for instruction handling
- Read [Interrupts](interrupts.md) for interrupt configuration
- Check [API Reference](../api-reference/core-api.md) for programmatic access
