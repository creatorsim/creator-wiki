# Core Architecture

This document describes the internal architecture of CREATOR's core execution engine.

## System Overview

CREATOR follows a modular architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────┐
│             User Interface Layer            │
│  (CLI: creator6.mts / Web: App.vue)        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│              Core API Layer                 │
│         (src/core/core.mjs)                 │
└──────────────────┬──────────────────────────┘
                   │
       ┌───────────┴───────────┐
       │                       │
┌──────▼──────┐        ┌──────▼──────┐
│  Assembler  │        │  Executor   │
│   System    │        │   System    │
└─────────────┘        └──────┬──────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
       ┌──────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
       │   Memory    │ │ Registers │ │   Devices   │
       │   System    │ │  System   │ │   System    │
       └─────────────┘ └───────────┘ └─────────────┘
```

## Core Components

### 1. Core API (`core.mjs`)

**Purpose**: Primary interface for creating and controlling simulator instances

**Key Functions**:
```javascript
// Create simulator instance
creator_init(architecture_obj)

// Assembly
assemble(source_code, options)

// Execution control
execute_one_instruction()
execute_program()
reset()

// State access
get_registers()
get_memory()
get_pc()
set_register(name, value)
```

**Responsibilities**:
- Initialize architecture
- Coordinate assembler and executor
- Provide unified API
- Manage simulator lifecycle

### 2. Architecture Processor

**Location**: `src/core/utils/architectureProcessor.mjs`

**Purpose**: Load and validate architecture definitions

**Process**:
1. Load YAML architecture file
2. Parse instruction formats
3. Compile instruction handlers
4. Generate register maps
5. Configure memory layout

**Architecture Structure**:
```javascript
{
  name: "RISC-V RV32I",
  bits: 32,
  registers: {
    int_registers: [...],
    fp_registers: [...],
    special_registers: [...]
  },
  memory: {
    text_start: 0x00400000,
    data_start: 0x10010000,
    stack_start: 0x7FFFEFFC
  },
  instructions: [
    {
      name: "add",
      format: "R",
      opcode: 0x33,
      funct3: 0x0,
      funct7: 0x00,
      behavior: "rd = rs1 + rs2"
    },
    // ...
  ]
}
```

### 3. Assembler System

**Location**: `src/core/assembler/`

**Purpose**: Convert assembly source to machine code

**Architecture**:
```
┌──────────────────────┐
│  Assembler Manager   │
└──────────┬───────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼───┐   ┌────▼────┐
│Native │   │External │
│Asm    │   │  (Z80)  │
└───────┘   └─────────┘
```

**Assembly Process**:
1. **Tokenization**: Break source into tokens
2. **First Pass**: Collect labels and directives
3. **Second Pass**: Generate machine code
4. **Memory Loading**: Load binary into memory
5. **Symbol Resolution**: Create symbol table

**Key Data Structures**:
```javascript
// Instruction list (for display/debugging)
instructions = [
  {
    Address: "0x00400000",
    Label: "main",
    loaded: "li t0, 10",  // Assembled instruction
    user: "li t0, 10",    // Original source
    Break: false          // Breakpoint flag
  },
  // ...
]

// Symbol table
tag_instructions = {
  "main": 0x00400000,
  "loop": 0x00400010,
  "exit": 0x00400020
}

// Main memory (loaded binary)
main_memory = Memory instance
```

### 4. Executor System

**Location**: `src/core/executor/`

**Purpose**: Execute instructions and manage state

**Components**:

#### Decoder (`decoder.mjs`)
- Decode binary to instruction
- Extract operands
- Identify instruction type

```javascript
function decode(binary) {
  const opcode = (binary >> 0) & 0x7F;
  const rd = (binary >> 7) & 0x1F;
  const funct3 = (binary >> 12) & 0x7;
  const rs1 = (binary >> 15) & 0x1F;
  const rs2 = (binary >> 20) & 0x1F;
  const funct7 = (binary >> 25) & 0x7F;
  
  return {
    name: lookupInstruction(opcode, funct3, funct7),
    rd, rs1, rs2,
    immediate: extractImmediate(binary, format)
  };
}
```

#### Instruction Compiler (`instructionCompiler.mts`)
- Compile instruction behaviors from YAML
- Generate JavaScript functions
- Optimize for performance

```javascript
function compileInstruction(instr) {
  const behavior = instr.behavior;
  
  // Parse behavior string and generate JS function
  return new Function('state', 'operands', `
    const { rd, rs1, rs2, imm } = operands;
    const regs = state.registers;
    
    // Execute behavior
    ${compileBehavior(behavior)}
    
    // Update PC
    state.pc += 4;
  `);
}
```

#### Executor (`executor.mjs`)
- Main execution loop
- PC management
- Instruction fetch/execute cycle

```javascript
function execute_one_instruction() {
  // Fetch
  const pc = get_pc();
  const binary = memory.readWord(pc);
  
  // Decode
  const instruction = decode(binary);
  
  // Execute
  const handler = instruction_handlers[instruction.name];
  handler(state, instruction.operands);
  
  // Update stats
  stats.instructions_executed++;
  stats.cycles += instruction.cycles;
  
  // Check interrupts
  if (interrupts_enabled) {
    checkInterrupt();
  }
  
  // Check devices
  handleDevices();
}
```

### 5. Memory System

**Location**: `src/core/memory/Memory.mts`

**Purpose**: Simulate memory hierarchy

**Features**:
- Segmented memory (text, data, stack)
- Access validation
- Endianness handling
- Memory hints (symbolic names)

**Memory Class**:
```typescript
class Memory {
  private segments: Map<string, Uint8Array>;
  private layout: MemoryLayout;
  
  readByte(address: bigint): number;
  readHalfWord(address: bigint): number;
  readWord(address: bigint): number;
  
  writeByte(address: bigint, value: number): void;
  writeHalfWord(address: bigint, value: number): void;
  writeWord(address: bigint, value: number): void;
  
  loadROM(data: Uint8Array, address: bigint): void;
  clear(): void;
  
  // Memory hints for debugging
  getHint(address: bigint): string | null;
  setHint(address: bigint, hint: string): void;
}
```

**Memory Layout**:
```javascript
{
  text: {
    start: 0x00400000n,
    end: 0x0FFFFFFFn,
    permissions: "r-x"  // Read, execute only
  },
  data: {
    start: 0x10010000n,
    end: 0x7FFFFFFFn,
    permissions: "rw-"  // Read, write
  },
  stack: {
    start: 0x7FFFEFFCn,
    grows: "down",
    permissions: "rw-"
  }
}
```

### 6. Register System

**Location**: `src/core/register/`

**Purpose**: Manage register state

**Components**:

#### Register Lookup (`registerLookup.mjs`)
```javascript
const registerMap = {
  // Integer registers
  "x0": 0, "zero": 0,
  "x1": 1, "ra": 1,
  "x2": 2, "sp": 2,
  "x3": 3, "gp": 3,
  // ... all 32 registers
  
  // Floating-point registers
  "f0": 0, "ft0": 0,
  "f1": 1, "ft1": 1,
  // ... all 32 FP registers
  
  // Special registers
  "pc": -1,
  "mstatus": -2,
  "mtvec": -3
};
```

#### Register Operations (`registerOperations.mjs`)
```javascript
function readRegister(name) {
  const index = registerMap[name];
  if (index === 0) return 0n;  // x0 always zero
  return registers.int[index];
}

function writeRegister(name, value) {
  const index = registerMap[name];
  if (index === 0) return;  // x0 immutable
  registers.int[index] = BigInt(value);
  
  // Mark register as modified (for glow effect)
  registerGlow.set(name, true);
}
```

#### Register Glow State (`registerGlowState.mjs`)
Tracks register modifications for UI highlighting:
```javascript
const glowState = new Map();

function markModified(regName) {
  glowState.set(regName, Date.now());
}

function clearOldGlow() {
  const now = Date.now();
  for (const [reg, time] of glowState) {
    if (now - time > 500) {  // 500ms glow duration
      glowState.delete(reg);
    }
  }
}
```

### 7. Device System

**Location**: `src/core/executor/devices.mts`

**Purpose**: I/O and OS services

**Architecture**:
```javascript
abstract class Device {
  ctrl_addr: bigint;    // Control register
  status_addr: bigint;  // Status register
  data_addr_start: bigint;
  data_addr_end: bigint;
  
  abstract handler(addr: bigint, value: bigint): void;
  
  writeValue(value: bigint): void;
  readValue(): bigint;
  clear(): void;
}
```

**Device Manager**:
```javascript
const devices = new Map<string, Device>();

function checkDeviceAddr(addr: bigint): Device | null {
  for (const device of devices.values()) {
    if (addr >= device.data_addr_start && 
        addr <= device.data_addr_end) {
      return device;
    }
  }
  return null;
}

function handleDevices(): void {
  for (const device of devices.values()) {
    if (isDeviceReady(device)) {
      device.handler(device.ctrl_addr, 0n);
    }
  }
}
```

See [Devices](devices.md) for details.

### 8. Interrupt System

**Location**: `src/core/executor/interrupts.mts`

**Purpose**: Handle interrupts and exceptions

**Types**:
```typescript
enum InterruptType {
  Software = 0,
  Timer = 1,
  External = 2,
  EnvironmentCall = 3
}

enum ExecutionMode {
  User = 0,
  Kernel = 1
}
```

**Interrupt Flow**:
```javascript
function handleInterrupt(type: InterruptType): void {
  // 1. Save current PC to EPC
  set_epc(get_pc());
  
  // 2. Set cause
  set_mcause(type);
  
  // 3. Switch to kernel mode
  set_execution_mode(ExecutionMode.Kernel);
  
  // 4. Disable interrupts
  disableInterrupts();
  
  // 5. Jump to handler
  const handler_addr = get_mtvec();
  set_pc(handler_addr);
}
```

See [Interrupts](interrupts.md) for details.

### 9. Statistics System

**Location**: `src/core/executor/stats.mts`

**Purpose**: Track execution metrics

```typescript
interface ExecutionStats {
  instructions_executed: number;
  cycles: number;
  cpi: number;  // Cycles per instruction
  
  // Instruction breakdown
  by_type: Map<string, number>;
  by_category: Map<string, number>;
  
  // Memory stats
  memory_reads: number;
  memory_writes: number;
  
  // Branch stats
  branches_taken: number;
  branches_not_taken: number;
  branch_prediction_accuracy: number;
}
```

## Execution Flow

### Complete Instruction Cycle

```
1. Fetch
   ├─ Read PC
   ├─ Read instruction from memory[PC]
   └─ Increment PC

2. Decode
   ├─ Extract opcode
   ├─ Identify instruction type
   ├─ Extract operands
   └─ Look up behavior handler

3. Execute
   ├─ Read source registers
   ├─ Perform operation
   ├─ Write destination register
   └─ Update PC (for branches)

4. Memory (if needed)
   ├─ Calculate address
   ├─ Read/write memory
   └─ Check alignment

5. Writeback
   ├─ Write result to register
   └─ Update glow state

6. Post-execution
   ├─ Update statistics
   ├─ Check interrupts
   ├─ Handle devices
   └─ Check breakpoints
```

### State Management

**Program State**:
```javascript
const state = {
  // Registers
  registers: {
    int: new BigUint64Array(32),
    fp: new Float64Array(32),
    pc: 0x00400000n,
    special: new Map()
  },
  
  // Memory
  memory: new Memory(),
  
  // Execution context
  execution_mode: ExecutionMode.User,
  interrupts_enabled: true,
  
  // Statistics
  stats: new ExecutionStats(),
  
  // Devices
  devices: new Map()
};
```

**State Snapshot** (for unstep):
```javascript
function captureState() {
  return {
    registers: cloneRegisters(state.registers),
    memory: state.memory.clone(),
    pc: state.pc,
    stats: { ...state.stats }
  };
}
```

## Data Flow

### Assembly to Execution

```
Source Code (.s)
    │
    ▼
┌─────────────┐
│  Tokenizer  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Parser    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Code Gen   │
└──────┬──────┘
       │
       ▼
Machine Code (binary)
    │
    ▼
┌─────────────┐
│   Memory    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Executor   │
└─────────────┘
```

## Thread Safety

CREATOR is single-threaded by design. State modifications are synchronous and atomic at the instruction level.

## Performance Considerations

### Optimization Strategies

1. **Instruction Compilation**: Pre-compile behaviors to JavaScript
2. **Register Caching**: Keep frequently accessed registers in local variables
3. **Memory Segmentation**: Avoid checking entire address space
4. **Lazy Evaluation**: Defer expensive operations until needed
5. **State Sharing**: Share immutable data between snapshots

### Bottlenecks

- **Memory Operations**: Most expensive operations
- **State Snapshots**: Deep cloning for unstep
- **Interrupt Checking**: Every instruction cycle
- **Device Polling**: Checked frequently

## Error Handling

### Error Hierarchy

```
SimulatorError
├─ AssemblyError
│  ├─ SyntaxError
│  ├─ UndefinedLabelError
│  └─ InvalidDirectiveError
├─ ExecutionError
│  ├─ SegmentationFault
│  ├─ InvalidInstructionError
│  └─ StackOverflowError
└─ ArchitectureError
   ├─ InvalidRegisterError
   └─ UnalignedAccessError
```

### Error Recovery

Most errors are fatal and halt execution. State remains valid for inspection.

## Extension Points

### Adding Features

1. **New Instructions**: Add to architecture YAML
2. **New Devices**: Extend Device class
3. **New Interrupts**: Add to InterruptType enum
4. **New Assemblers**: Implement assembler interface

## Next Steps

- Review [Execution Engine](execution-engine.md) for instruction execution details
- See [Memory System](memory-system.md) for memory management
- Read [Devices](devices.md) for I/O system
- Check [Interrupts](interrupts.md) for interrupt handling
