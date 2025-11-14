# Memory System

CREATOR's memory system simulates computer memory with segmentation, access control, and debugging features.

## Overview

The memory system provides:
- **Segmented memory**: Separate text, data, and stack regions
- **Access validation**: Read/write/execute permissions
- **Endianness handling**: Little-endian by default
- **Memory hints**: Symbolic names for debugging
- **Stack tracking**: Monitor stack usage

## Memory Class

**Location**: `src/core/memory/Memory.mts`

### Class Structure

```typescript
class Memory {
  private segments: Map<string, Uint8Array>;
  private layout: MemoryLayout;
  private hints: Map<bigint, string>;
  
  constructor(layout: MemoryLayout);
  
  // Read operations
  readByte(address: bigint): number;
  readHalfWord(address: bigint): number;
  readWord(address: bigint): number;
  readDoubleWord(address: bigint): bigint;
  
  // Write operations
  writeByte(address: bigint, value: number): void;
  writeHalfWord(address: bigint, value: number): void;
  writeWord(address: bigint, value: number): void;
  writeDoubleWord(address: bigint, value: bigint): void;
  
  // Bulk operations
  loadROM(data: Uint8Array, address: bigint): void;
  readBlock(address: bigint, length: number): Uint8Array;
  writeBlock(address: bigint, data: Uint8Array): void;
  
  // Validation
  isReadable(address: bigint): boolean;
  isWritable(address: bigint): boolean;
  isExecutable(address: bigint): boolean;
  
  // Debugging
  getHint(address: bigint): string | null;
  setHint(address: bigint, hint: string): void;
  clear(): void;
  clone(): Memory;
}
```

## Memory Layout

### RISC-V / MIPS Layout

```
0x00000000 ┌──────────────────────────┐
           │      Reserved            │
0x00400000 ├──────────────────────────┤
           │   .text (Code)           │
           │   r-x (Read, Execute)    │
0x0FFFFFFF ├──────────────────────────┤
           │      Reserved            │
0x10010000 ├──────────────────────────┤
           │   .data (Global Data)    │
           │   rw- (Read, Write)      │
           │                          │
           │   Heap (grows up)        │
           │          ↓               │
0x7FFFFFFF ├──────────────────────────┤
           │                          │
           │          ↑               │
           │   Stack (grows down)     │
0x7FFFEFFC ├──────────────────────────┤ <- Initial SP
           │      Reserved            │
0xFFFFFFFF └──────────────────────────┘
```

### Z80 Layout

```
0x0000 ┌──────────────────────┐
       │   ROM/Program        │
       │   r-x                │
0x7FFF ├──────────────────────┤
0x8000 │   RAM                │
       │   rw-                │
0xFFFF └──────────────────────┘
```

### Layout Definition

```typescript
interface MemoryLayout {
  segments: {
    text: {
      start: bigint;
      end: bigint;
      permissions: "r-x";
    };
    data: {
      start: bigint;
      end: bigint;
      permissions: "rw-";
    };
    stack: {
      start: bigint;  // Initial SP
      end: bigint;
      permissions: "rw-";
      grows: "down";
    };
  };
  word_size: number;  // 1, 2, or 4 bytes
  endianness: "little" | "big";
}
```

## Memory Operations

### Read Operations

#### Read Byte (8-bit)

```typescript
readByte(address: bigint): number {
  // Validate address
  if (!this.isReadable(address)) {
    throw new SegmentationFault(
      `Cannot read from address 0x${address.toString(16)}`
    );
  }
  
  // Find segment
  const segment = this.getSegment(address);
  if (!segment) {
    throw new SegmentationFault(`Invalid address: 0x${address.toString(16)}`);
  }
  
  // Calculate offset
  const offset = Number(address - segment.start);
  
  // Read byte
  return segment.data[offset];
}
```

#### Read Half Word (16-bit)

```typescript
readHalfWord(address: bigint): number {
  // Check alignment
  if (address % 2n !== 0n) {
    throw new UnalignedAccessError(
      `Unaligned halfword read at 0x${address.toString(16)}`
    );
  }
  
  // Read two bytes
  const byte0 = this.readByte(address);
  const byte1 = this.readByte(address + 1n);
  
  // Combine based on endianness
  if (this.layout.endianness === "little") {
    return byte0 | (byte1 << 8);
  } else {
    return (byte0 << 8) | byte1;
  }
}
```

#### Read Word (32-bit)

```typescript
readWord(address: bigint): number {
  // Check alignment
  if (address % 4n !== 0n) {
    throw new UnalignedAccessError(
      `Unaligned word read at 0x${address.toString(16)}`
    );
  }
  
  // Read four bytes
  const byte0 = this.readByte(address);
  const byte1 = this.readByte(address + 1n);
  const byte2 = this.readByte(address + 2n);
  const byte3 = this.readByte(address + 3n);
  
  // Combine based on endianness
  if (this.layout.endianness === "little") {
    return byte0 | (byte1 << 8) | (byte2 << 16) | (byte3 << 24);
  } else {
    return (byte0 << 24) | (byte1 << 16) | (byte2 << 8) | byte3;
  }
}
```

### Write Operations

#### Write Byte

```typescript
writeByte(address: bigint, value: number): void {
  // Validate address
  if (!this.isWritable(address)) {
    throw new SegmentationFault(
      `Cannot write to address 0x${address.toString(16)}`
    );
  }
  
  // Find segment
  const segment = this.getSegment(address);
  const offset = Number(address - segment.start);
  
  // Write byte (mask to 8 bits)
  segment.data[offset] = value & 0xFF;
}
```

#### Write Word

```typescript
writeWord(address: bigint, value: number): void {
  // Check alignment
  if (address % 4n !== 0n) {
    throw new UnalignedAccessError(
      `Unaligned word write at 0x${address.toString(16)}`
    );
  }
  
  // Split into bytes
  if (this.layout.endianness === "little") {
    this.writeByte(address, value & 0xFF);
    this.writeByte(address + 1n, (value >> 8) & 0xFF);
    this.writeByte(address + 2n, (value >> 16) & 0xFF);
    this.writeByte(address + 3n, (value >> 24) & 0xFF);
  } else {
    this.writeByte(address, (value >> 24) & 0xFF);
    this.writeByte(address + 1n, (value >> 16) & 0xFF);
    this.writeByte(address + 2n, (value >> 8) & 0xFF);
    this.writeByte(address + 3n, value & 0xFF);
  }
}
```

### Bulk Operations

#### Load ROM

Used after assembly to load program:

```typescript
loadROM(data: Uint8Array, address: bigint): void {
  // Validate can write to this region
  if (!this.isWritable(address)) {
    throw new Error("Cannot load ROM to read-only segment");
  }
  
  // Copy data
  for (let i = 0; i < data.length; i++) {
    this.writeByte(address + BigInt(i), data[i]);
  }
}
```

#### Read Block

```typescript
readBlock(address: bigint, length: number): Uint8Array {
  const result = new Uint8Array(length);
  
  for (let i = 0; i < length; i++) {
    result[i] = this.readByte(address + BigInt(i));
  }
  
  return result;
}
```

## Access Validation

### Permission Checking

```typescript
isReadable(address: bigint): boolean {
  const segment = this.getSegmentInfo(address);
  return segment && (
    segment.permissions.includes('r') ||
    segment.permissions.includes('x')
  );
}

isWritable(address: bigint): boolean {
  const segment = this.getSegmentInfo(address);
  return segment && segment.permissions.includes('w');
}

isExecutable(address: bigint): boolean {
  const segment = this.getSegmentInfo(address);
  return segment && segment.permissions.includes('x');
}
```

### Segment Lookup

```typescript
private getSegmentInfo(address: bigint): SegmentInfo | null {
  for (const [name, segment] of Object.entries(this.layout.segments)) {
    if (address >= segment.start && address <= segment.end) {
      return { name, ...segment };
    }
  }
  return null;
}
```

## Memory Hints

Memory hints provide symbolic names for debugging:

```typescript
class MemoryHints {
  private hints: Map<bigint, string>;
  
  setHint(address: bigint, hint: string): void {
    this.hints.set(address, hint);
  }
  
  getHint(address: bigint): string | null {
    return this.hints.get(address) || null;
  }
  
  // Automatically generate hints from symbols
  generateFromSymbols(symbols: Map<string, bigint>): void {
    for (const [name, addr] of symbols) {
      this.setHint(addr, name);
    }
  }
  
  // Generate array element hints
  generateArrayHints(baseAddr: bigint, name: string, size: number, elementSize: number): void {
    for (let i = 0; i < size; i++) {
      const addr = baseAddr + BigInt(i * elementSize);
      this.setHint(addr, `${name}[${i}]`);
    }
  }
}
```

### Using Hints

```typescript
// When displaying memory
function displayMemory(address: bigint, count: number): void {
  for (let i = 0; i < count; i++) {
    const addr = address + BigInt(i * 4);
    const value = memory.readWord(addr);
    const hint = memory.getHint(addr);
    
    if (hint) {
      console.log(`0x${addr.toString(16)}: ${hint} = ${value}`);
    } else {
      console.log(`0x${addr.toString(16)}: ${value}`);
    }
  }
}

// Output:
// 0x10010000: array[0] = 5
// 0x10010004: array[1] = 12
// 0x10010008: array[2] = 3
```

## Stack Tracking

**Location**: `src/core/memory/StackTracker.mts`

### Stack Tracker Class

```typescript
class StackTracker {
  private initialSP: bigint;
  private maxDepth: bigint;
  private frames: StackFrame[];
  
  push(returnAddr: bigint, frameSize: number): void {
    this.frames.push({
      returnAddr,
      frameSize,
      sp: this.getCurrentSP()
    });
  }
  
  pop(): StackFrame | null {
    return this.frames.pop() || null;
  }
  
  getDepth(): number {
    return this.frames.length;
  }
  
  getCurrentFrame(): StackFrame | null {
    return this.frames[this.frames.length - 1] || null;
  }
  
  getMaxDepth(): bigint {
    return this.maxDepth;
  }
  
  // Check for stack overflow
  checkOverflow(): boolean {
    const current = this.getCurrentSP();
    const used = this.initialSP - current;
    return used > this.maxDepth;
  }
}

interface StackFrame {
  returnAddr: bigint;
  frameSize: number;
  sp: bigint;
  locals?: Map<string, any>;
}
```

### Stack Visualization

```typescript
function displayStack(count: number = 8): void {
  const sp = readRegister('sp');
  
  console.log("Stack (top entries):");
  console.log("Address        Value         Description");
  console.log("─".repeat(50));
  
  for (let i = 0; i < count; i++) {
    const addr = sp + BigInt(i * 4);
    
    try {
      const value = memory.readWord(addr);
      const hint = memory.getHint(addr) || "";
      
      console.log(
        `0x${addr.toString(16).padStart(8, '0')}  ` +
        `0x${value.toString(16).padStart(8, '0')}  ` +
        hint
      );
    } catch (error) {
      break;  // Stack end reached
    }
  }
}

// Output:
// Stack (top entries):
// Address        Value         Description
// ──────────────────────────────────────────────────
// 0x7fffeff0  0x00400010  Return address
// 0x7fffeff4  0x00000005  Local variable 'n'
// 0x7fffeff8  0x00000000  Saved register s0
// 0x7fffeffc  0x00000000  (initial SP)
```

## Memory Cloning

For snapshots and unstep:

```typescript
clone(): Memory {
  const cloned = new Memory(this.layout);
  
  // Deep copy all segments
  for (const [name, segment] of this.segments) {
    cloned.segments.set(name, new Uint8Array(segment));
  }
  
  // Copy hints
  cloned.hints = new Map(this.hints);
  
  return cloned;
}
```

## Performance Optimization

### Memory Caching

```typescript
class CachedMemory extends Memory {
  private cache: Map<bigint, number>;
  private cacheSize: number = 1024;
  
  readWord(address: bigint): number {
    // Check cache
    if (this.cache.has(address)) {
      return this.cache.get(address)!;
    }
    
    // Read from memory
    const value = super.readWord(address);
    
    // Cache result
    if (this.cache.size >= this.cacheSize) {
      // Evict oldest entry (FIFO)
      const first = this.cache.keys().next().value;
      this.cache.delete(first);
    }
    this.cache.set(address, value);
    
    return value;
  }
  
  writeWord(address: bigint, value: number): void {
    // Invalidate cache
    this.cache.delete(address);
    
    // Write to memory
    super.writeWord(address, value);
  }
}
```

### Sparse Memory

For architectures with large address spaces:

```typescript
class SparseMemory extends Memory {
  private pages: Map<bigint, Uint8Array>;
  private pageSize: number = 4096;  // 4KB pages
  
  private getPage(address: bigint): Uint8Array {
    const pageAddr = (address / BigInt(this.pageSize)) * BigInt(this.pageSize);
    
    if (!this.pages.has(pageAddr)) {
      this.pages.set(pageAddr, new Uint8Array(this.pageSize));
    }
    
    return this.pages.get(pageAddr)!;
  }
  
  readByte(address: bigint): number {
    const page = this.getPage(address);
    const offset = Number(address % BigInt(this.pageSize));
    return page[offset];
  }
  
  writeByte(address: bigint, value: number): void {
    const page = this.getPage(address);
    const offset = Number(address % BigInt(this.pageSize));
    page[offset] = value & 0xFF;
  }
}
```

## Error Handling

### Memory Exceptions

```typescript
class MemoryError extends Error {
  constructor(message: string, public address: bigint) {
    super(message);
    this.name = "MemoryError";
  }
}

class SegmentationFault extends MemoryError {
  constructor(message: string, address: bigint) {
    super(message, address);
    this.name = "SegmentationFault";
  }
}

class UnalignedAccessError extends MemoryError {
  constructor(message: string, address: bigint) {
    super(message, address);
    this.name = "UnalignedAccessError";
  }
}

class StackOverflowError extends MemoryError {
  constructor(message: string, address: bigint) {
    super(message, address);
    this.name = "StackOverflowError";
  }
}
```

## Memory Dump

### Hexadecimal View

```typescript
function hexdump(address: bigint, length: number, width: number = 16): void {
  console.log("Address   " + " ".repeat(width * 3) + "ASCII");
  console.log("─".repeat(10 + width * 3 + width + 2));
  
  for (let i = 0; i < length; i += width) {
    const addr = address + BigInt(i);
    const hexPart: string[] = [];
    const asciiPart: string[] = [];
    
    for (let j = 0; j < width && i + j < length; j++) {
      try {
        const byte = memory.readByte(addr + BigInt(j));
        hexPart.push(byte.toString(16).padStart(2, '0'));
        asciiPart.push(byte >= 32 && byte < 127 ? String.fromCharCode(byte) : '.');
      } catch {
        break;
      }
    }
    
    console.log(
      `${addr.toString(16).padStart(8, '0')}: ` +
      hexPart.join(' ').padEnd(width * 3) +
      ' ' +
      asciiPart.join('')
    );
  }
}

// Output:
// Address    00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f  ASCII
// ───────────────────────────────────────────────────────────────────
// 10010000: 48 65 6c 6c 6f 2c 20 57 6f 72 6c 64 21 0a 00 00  Hello, World!...
// 10010010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

## Memory Statistics

```typescript
interface MemoryStats {
  total_reads: number;
  total_writes: number;
  bytes_read: number;
  bytes_written: number;
  cache_hits: number;
  cache_misses: number;
  
  // Per-segment stats
  text_accesses: number;
  data_accesses: number;
  stack_accesses: number;
  
  // Errors
  segmentation_faults: number;
  unaligned_accesses: number;
}
```

## Next Steps

- Review [Core Architecture](core-architecture.md) for system overview
- See [Execution Engine](execution-engine.md) for instruction processing
- Read [Stack Tracker](stack-tracker.md) for stack management details
- Check [Devices](devices.md) for memory-mapped I/O
