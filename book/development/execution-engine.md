# Execution Engine

The execution engine is the heart of CREATOR, responsible for fetching, decoding, and executing instructions.

## Overview

CREATOR uses a classic instruction execution pipeline:

```
┌───────┐    ┌────────┐    ┌─────────┐    ┌────────┐    ┌───────────┐
│ Fetch │ -> │ Decode │ -> │ Execute │ -> │ Memory │ -> │ Writeback │
└───────┘    └────────┘    └─────────┘    └────────┘    └───────────┘
```

## Components

### 1. Instruction Fetch

**Location**: `src/core/executor/executor.mjs`

**Purpose**: Retrieve instruction from memory

```javascript
function fetch() {
  const pc = get_pc();
  
  // Validate PC
  if (!memory.isExecutable(pc)) {
    throw new SegmentationFault(`Invalid PC: 0x${pc.toString(16)}`);
  }
  
  // Read instruction (word-sized)
  const binary = memory.readWord(pc);
  
  return { pc, binary };
}
```

**Error Conditions**:
- PC out of bounds
- PC in non-executable memory (.data, stack)
- Unaligned PC (not 4-byte aligned for 32-bit architectures)

### 2. Instruction Decode

**Location**: `src/core/executor/decoder.mjs`

**Purpose**: Convert binary to instruction structure

#### RISC-V Decoding

```javascript
function decodeRISCV(binary) {
  const opcode = (binary >> 0) & 0x7F;
  
  // Extract fields based on format
  const rd = (binary >> 7) & 0x1F;
  const funct3 = (binary >> 12) & 0x7;
  const rs1 = (binary >> 15) & 0x1F;
  const rs2 = (binary >> 20) & 0x1F;
  const funct7 = (binary >> 25) & 0x7F;
  
  // Look up instruction
  const instr = instructionTable.lookup(opcode, funct3, funct7);
  
  if (!instr) {
    throw new InvalidInstructionError(`Unknown instruction: 0x${binary.toString(16)}`);
  }
  
  // Extract immediate based on format
  const immediate = extractImmediate(binary, instr.format);
  
  return {
    name: instr.name,
    format: instr.format,
    rd, rs1, rs2,
    immediate,
    binary,
    handler: instr.handler
  };
}
```

#### Instruction Formats

**R-Type** (Register):
```
31        25 24    20 19    15 14    12 11     7 6       0
┌───────────┬────────┬────────┬────────┬────────┬─────────┐
│  funct7   │   rs2  │   rs1  │ funct3 │   rd   │ opcode  │
└───────────┴────────┴────────┴────────┴────────┴─────────┘
```

**I-Type** (Immediate):
```
31                20 19    15 14    12 11     7 6       0
┌────────────────────┬────────┬────────┬────────┬─────────┐
│      imm[11:0]     │   rs1  │ funct3 │   rd   │ opcode  │
└────────────────────┴────────┴────────┴────────┴─────────┘
```

**S-Type** (Store):
```
31        25 24    20 19    15 14    12 11     7 6       0
┌───────────┬────────┬────────┬────────┬────────┬─────────┐
│ imm[11:5] │   rs2  │   rs1  │ funct3 │imm[4:0]│ opcode  │
└───────────┴────────┴────────┴────────┴────────┴─────────┘
```

**B-Type** (Branch):
```
31   30    25 24    20 19    15 14    12 11    8 7       6       0
┌────┬───────┬────────┬────────┬────────┬───────┬────┬──────────┐
│imm │imm    │   rs2  │   rs1  │ funct3 │ imm   │imm │  opcode  │
│[12]│[10:5] │        │        │        │ [4:1] │[11]│          │
└────┴───────┴────────┴────────┴────────┴───────┴────┴──────────┘
```

**U-Type** (Upper immediate):
```
31                                     12 11     7 6       0
┌──────────────────────────────────────┬────────┬─────────┐
│             imm[31:12]               │   rd   │ opcode  │
└──────────────────────────────────────┴────────┴─────────┘
```

**J-Type** (Jump):
```
31   30      21 20   19    12 11     7 6       0
┌────┬──────────┬────┬────────┬────────┬─────────┐
│imm │  imm     │imm │  imm   │   rd   │ opcode  │
│[20]│  [10:1]  │[11]│ [19:12]│        │         │
└────┴──────────┴────┴────────┴────────┴─────────┘
```

#### Immediate Extraction

```javascript
function extractImmediate(binary, format) {
  switch (format) {
    case 'I':
      return signExtend(binary >> 20, 12);
      
    case 'S':
      const imm_11_5 = (binary >> 25) & 0x7F;
      const imm_4_0 = (binary >> 7) & 0x1F;
      return signExtend((imm_11_5 << 5) | imm_4_0, 12);
      
    case 'B':
      const imm_12 = (binary >> 31) & 0x1;
      const imm_10_5 = (binary >> 25) & 0x3F;
      const imm_4_1 = (binary >> 8) & 0xF;
      const imm_11 = (binary >> 7) & 0x1;
      const imm = (imm_12 << 12) | (imm_11 << 11) | 
                  (imm_10_5 << 5) | (imm_4_1 << 1);
      return signExtend(imm, 13);
      
    case 'U':
      return (binary >> 12) << 12;
      
    case 'J':
      const jimm_20 = (binary >> 31) & 0x1;
      const jimm_10_1 = (binary >> 21) & 0x3FF;
      const jimm_11 = (binary >> 20) & 0x1;
      const jimm_19_12 = (binary >> 12) & 0xFF;
      const jimm = (jimm_20 << 20) | (jimm_19_12 << 12) |
                   (jimm_11 << 11) | (jimm_10_1 << 1);
      return signExtend(jimm, 21);
      
    default:
      return 0;
  }
}

function signExtend(value, bits) {
  const sign = (value >> (bits - 1)) & 0x1;
  if (sign) {
    // Negative: extend with 1s
    return value | (~0 << bits);
  }
  return value;
}
```

### 3. Instruction Execution

**Location**: `src/core/executor/instructionCompiler.mts`

**Purpose**: Execute instruction behavior

#### Compiled Handlers

Instructions are compiled from YAML definitions to JavaScript:

```yaml
# Architecture YAML
instructions:
  - name: add
    format: R
    opcode: 0x33
    funct3: 0x0
    funct7: 0x00
    behavior: "rd = rs1 + rs2"
    cycles: 1
```

Compiled to:

```javascript
function execute_add(state, operands) {
  const { rd, rs1, rs2 } = operands;
  
  // Read source registers
  const val_rs1 = readRegister(rs1);
  const val_rs2 = readRegister(rs2);
  
  // Execute behavior
  const result = val_rs1 + val_rs2;
  
  // Write destination
  writeRegister(rd, result);
  
  // Advance PC
  state.pc += 4n;
  
  // Update stats
  state.stats.cycles += 1;
}
```

#### Example Instructions

**ADD (R-type)**:
```javascript
function execute_add(state, { rd, rs1, rs2 }) {
  const result = readRegister(rs1) + readRegister(rs2);
  writeRegister(rd, result & 0xFFFFFFFFn);  // 32-bit mask
  state.pc += 4n;
}
```

**ADDI (I-type)**:
```javascript
function execute_addi(state, { rd, rs1, immediate }) {
  const result = readRegister(rs1) + BigInt(immediate);
  writeRegister(rd, result & 0xFFFFFFFFn);
  state.pc += 4n;
}
```

**LW (I-type, Load Word)**:
```javascript
function execute_lw(state, { rd, rs1, immediate }) {
  const addr = readRegister(rs1) + BigInt(immediate);
  
  // Check alignment
  if (addr % 4n !== 0n) {
    throw new UnalignedAccessError(`Unaligned load at 0x${addr.toString(16)}`);
  }
  
  // Read memory
  const value = state.memory.readWord(addr);
  
  // Write to register
  writeRegister(rd, value);
  
  state.pc += 4n;
  state.stats.memory_reads++;
}
```

**SW (S-type, Store Word)**:
```javascript
function execute_sw(state, { rs1, rs2, immediate }) {
  const addr = readRegister(rs1) + BigInt(immediate);
  const value = readRegister(rs2);
  
  // Check alignment
  if (addr % 4n !== 0n) {
    throw new UnalignedAccessError(`Unaligned store at 0x${addr.toString(16)}`);
  }
  
  // Write memory
  state.memory.writeWord(addr, value);
  
  state.pc += 4n;
  state.stats.memory_writes++;
}
```

**BEQ (B-type, Branch if Equal)**:
```javascript
function execute_beq(state, { rs1, rs2, immediate }) {
  const val1 = readRegister(rs1);
  const val2 = readRegister(rs2);
  
  if (val1 === val2) {
    // Branch taken
    state.pc += BigInt(immediate);
    state.stats.branches_taken++;
  } else {
    // Branch not taken
    state.pc += 4n;
    state.stats.branches_not_taken++;
  }
}
```

**JAL (J-type, Jump and Link)**:
```javascript
function execute_jal(state, { rd, immediate }) {
  // Save return address
  writeRegister(rd, state.pc + 4n);
  
  // Jump
  state.pc += BigInt(immediate);
  
  state.stats.jumps++;
}
```

**ECALL (Environment Call)**:
```javascript
function execute_ecall(state) {
  const syscall_num = readRegister('a7');
  
  // Trigger system call interrupt
  makeInterrupt(InterruptType.EnvironmentCall);
  
  // Handler will process syscall based on a7
  // and use a0-a6 for arguments
  
  state.pc += 4n;
}
```

### 4. Memory Operations

Memory instructions interact with the memory system:

```javascript
function loadMemory(address, size) {
  // Validate address
  if (!memory.isReadable(address)) {
    throw new SegmentationFault(`Cannot read from 0x${address.toString(16)}`);
  }
  
  // Check alignment
  if (size > 1 && address % BigInt(size) !== 0n) {
    throw new UnalignedAccessError(`Unaligned access at 0x${address.toString(16)}`);
  }
  
  // Check device addresses
  const device = checkDeviceAddr(address);
  if (device) {
    return device.readValue();
  }
  
  // Read from memory
  switch (size) {
    case 1: return memory.readByte(address);
    case 2: return memory.readHalfWord(address);
    case 4: return memory.readWord(address);
    default: throw new Error(`Invalid size: ${size}`);
  }
}

function storeMemory(address, value, size) {
  // Similar validation...
  
  // Check device addresses
  const device = checkDeviceAddr(address);
  if (device) {
    device.writeValue(value);
    device.handler(address, value);
    return;
  }
  
  // Write to memory
  switch (size) {
    case 1: memory.writeByte(address, value); break;
    case 2: memory.writeHalfWord(address, value); break;
    case 4: memory.writeWord(address, value); break;
  }
}
```

### 5. Control Flow

#### Branch Prediction

CREATOR uses static branch prediction (assume not taken) by default:

```javascript
function executeBranch(condition, target) {
  if (condition) {
    // Branch taken (misprediction penalty)
    state.pc = target;
    state.stats.branch_mispredictions++;
    state.stats.cycles += BRANCH_PENALTY;
  } else {
    // Branch not taken (correct prediction)
    state.pc += 4n;
  }
}
```

#### Jump Instructions

```javascript
// Direct jump
function executeJump(target) {
  state.pc = target;
}

// Indirect jump (JALR)
function executeJumpRegister(base, offset) {
  state.pc = (readRegister(base) + BigInt(offset)) & ~1n;
  // Clear LSB for alignment
}
```

### 6. Pipeline Simulation

CREATOR can simulate a simple pipeline (optional):

```javascript
const pipeline = {
  fetch: null,
  decode: null,
  execute: null,
  memory: null,
  writeback: null
};

function clockCycle() {
  // Writeback stage
  if (pipeline.writeback) {
    writeRegister(pipeline.writeback.rd, pipeline.writeback.result);
  }
  
  // Memory stage
  if (pipeline.memory) {
    if (pipeline.memory.isLoad) {
      pipeline.writeback = {
        rd: pipeline.memory.rd,
        result: loadMemory(pipeline.memory.addr, pipeline.memory.size)
      };
    } else if (pipeline.memory.isStore) {
      storeMemory(pipeline.memory.addr, pipeline.memory.value, pipeline.memory.size);
    } else {
      pipeline.writeback = pipeline.memory;
    }
  }
  
  // Execute stage
  if (pipeline.execute) {
    const result = executeALU(pipeline.execute);
    pipeline.memory = { ...pipeline.execute, result };
  }
  
  // Decode stage
  if (pipeline.decode) {
    pipeline.execute = decode(pipeline.decode.binary);
  }
  
  // Fetch stage
  const pc = get_pc();
  pipeline.decode = { binary: memory.readWord(pc) };
  set_pc(pc + 4n);
}
```

## Execution Modes

### Sequential Execution

Execute one instruction at a time (step mode):

```javascript
function execute_one_instruction() {
  // Save state for unstep
  if (MAX_STATES_TO_KEEP > 0) {
    saveState();
  }
  
  // Fetch
  const { pc, binary } = fetch();
  
  // Decode
  const instruction = decode(binary);
  
  // Execute
  instruction.handler(state, instruction);
  
  // Post-execution
  checkInterrupts();
  handleDevices();
  checkBreakpoints();
  
  // Update stats
  state.stats.instructions_executed++;
  
  return instruction;
}
```

### Continuous Execution

Run until completion or breakpoint:

```javascript
function execute_program() {
  let running = true;
  
  while (running) {
    try {
      execute_one_instruction();
      
      // Check for termination
      if (program_exited) {
        running = false;
      }
      
      // Check breakpoints
      if (hitBreakpoint()) {
        running = false;
      }
      
      // Yield to prevent blocking
      if (state.stats.instructions_executed % 1000 === 0) {
        await sleep(0);
      }
    } catch (error) {
      handleExecutionError(error);
      running = false;
    }
  }
}
```

### Silent Execution

Run without output (fast):

```javascript
function execute_silent() {
  // Disable output
  const originalConsole = global.console;
  global.console = { log: () => {} };
  
  try {
    execute_program();
  } finally {
    global.console = originalConsole;
  }
}
```

## Performance Optimization

### Instruction Caching

```javascript
const instructionCache = new Map();

function getCachedInstruction(binary) {
  if (instructionCache.has(binary)) {
    return instructionCache.get(binary);
  }
  
  const instruction = decode(binary);
  instructionCache.set(binary, instruction);
  return instruction;
}
```

### Register Aliasing

```javascript
// Keep frequently used registers in local variables
let pc = state.pc;
let sp = state.registers.int[2];  // Stack pointer
let ra = state.registers.int[1];  // Return address

// ... execute instructions ...

// Sync back
state.pc = pc;
state.registers.int[2] = sp;
state.registers.int[1] = ra;
```

### Fast Path for Common Instructions

```javascript
function fastExecute(binary) {
  const opcode = binary & 0x7F;
  
  // Fast path for common ALU instructions
  if (opcode === 0x33) {  // R-type
    return executeRTypeALU(binary);
  }
  
  // Fast path for loads/stores
  if (opcode === 0x03 || opcode === 0x23) {
    return executeMemoryOp(binary);
  }
  
  // Fall back to full decode
  return execute_one_instruction();
}
```

## Error Handling

### Execution Errors

```javascript
class ExecutionError extends Error {
  constructor(message, pc, instruction) {
    super(message);
    this.pc = pc;
    this.instruction = instruction;
  }
}

function handleExecutionError(error) {
  console.error(`Execution error at PC=0x${error.pc.toString(16)}`);
  console.error(`Instruction: ${error.instruction}`);
  console.error(`Error: ${error.message}`);
  
  // Save error state for debugging
  saveErrorState(error);
}
```

## Debugging Support

### Breakpoints

```javascript
const breakpoints = new Set();

function checkBreakpoints() {
  if (breakpoints.has(state.pc)) {
    throw new BreakpointHit(state.pc);
  }
}

function setBreakpoint(address) {
  breakpoints.add(address);
}

function deleteBreakpoint(address) {
  breakpoints.delete(address);
}
```

### Execution Tracing

```javascript
const trace = [];

function traceInstruction(instruction) {
  trace.push({
    pc: state.pc,
    instruction: instruction.name,
    registers: cloneRegisters(state.registers),
    timestamp: Date.now()
  });
  
  // Keep last 1000 instructions
  if (trace.length > 1000) {
    trace.shift();
  }
}
```

## Next Steps

- Review [Core Architecture](core-architecture.md) for system overview
- See [Memory System](memory-system.md) for memory details
- Read [Devices](devices.md) for I/O handling
- Check [Interrupts](interrupts.md) for interrupt processing
