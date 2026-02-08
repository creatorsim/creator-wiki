# Devices

CREATOR's device system provides a flexible abstraction for I/O operations and OS services. Devices are memory-mapped and handle system calls through a standardized interface.

## Device Architecture

### Overview

Devices in CREATOR:
- Are memory-mapped (specific address ranges)
- Have control and status registers
- Provide data buffers
- Handle system calls (ecall/syscall)
- Can be enabled/disabled
- Maintain their own state

### Device Base Class

All devices extend the abstract `Device` class:

```typescript
abstract class Device {
    readonly ctrl_addr: number;        // Control register address
    readonly status_addr: number;      // Status register address
    readonly data: {                   // Data buffer range
        start: number;
        end: number;
    };
    enabled: boolean;                  // Device state
    memory: Memory;                    // Device memory
    
    abstract handler(): void;          // Device behavior
}
```

Source: `src/core/executor/devices.mts`

## Built-in Devices

CREATOR includes two primary devices by default:

### Console Device

Handles console I/O operations.

**Address Map** (typical):
```
0xF0000000: Control register
0xF0000004: Status register
0xF0000008-0xF000000F: Data buffer (8 bytes)
```

**System Calls Handled**:
- **1**: Print integer
- **2**: Print float
- **3**: Print double
- **4**: Print string
- **5**: Read integer
- **6**: Read float
- **7**: Read double
- **8**: Read string
- **11**: Print character
- **12**: Read character

**Implementation**:

```typescript
class ConsoleDevice extends Device {
    handler(): void {
        const ctrlValue = this.readValue(this.ctrl_addr).getUint32(0);
        
        switch (ctrlValue) {
            case 1:  // print_int
                const intValue = this.readValue(this.data.start).getInt32(0);
                display_print(intValue.toString());
                break;
                
            case 4:  // print_string
                const addr = this.readValue(this.data.start).getInt32(0);
                const str = readStringFromMemory(addr);
                display_print(str);
                break;
                
            case 5:  // read_int
                // Pause execution, wait for input
                status.run_program = 3;
                keyboard_read(callback);
                break;
                
            // ... other cases
        }
        
        this.clear();  // Clear control register after handling
    }
}
```

**Print Operation Flow**:
1. Program writes syscall code to a7 (RISC-V) or equivalent
2. Program writes arguments to appropriate registers
3. Program executes `ecall`
4. Executor checks devices for matching control code
5. ConsoleDevice.handler() is called
6. Device reads data from its buffer
7. Output is sent to display
8. Control register is cleared

**Read Operation Flow**:
1. Program initiates read syscall
2. Device pauses execution (`status.run_program = 3`)
3. UI prompts for input
4. User enters data
5. Callback writes data to device buffer
6. Execution resumes
7. Program reads result from register

### OS Driver Device

Handles OS-level operations.

**Address Map** (typical):
```
0xF0000010: Control register
0xF0000014: Status register
0xF0000018-0xF000001F: Data buffer (8 bytes)
```

**System Calls Handled**:
- **9**: sbrk (allocate heap memory)
- **10**: exit (terminate program)

**Implementation**:

```typescript
class OSDriver extends Device {
    handler(): void {
        const ctrlValue = this.readValue(this.ctrl_addr).getUint32(0);
        
        switch (ctrlValue) {
            case 9: {  // sbrk
                const size = this.readValue(this.data.start).getUint32(0);
                const addr = MEM.alloc(size);
                this.writeValue(this.data.start, addr);
                break;
            }
                
            case 10:  // exit
                SYSCALL.exit();
                break;
        }
        
        this.clear();
    }
}
```

## Device Memory

Each device has its own internal memory separate from main memory:

```typescript
constructor(config) {
    // Calculate memory size needed
    const minAddr = Math.min(ctrl_addr, status_addr, data.start);
    const maxAddr = Math.max(ctrl_addr, status_addr, data.end);
    
    // Create device memory
    this.memory = new Memory({
        sizeInBytes: maxAddr - minAddr + 1,
        baseAddress: BigInt(minAddr),
        bitsPerByte: 8,
        wordSize: 4,
        endianness: [0, 1, 2, 3]  // Big Endian
    });
}
```

## Device Lifecycle

### Initialization

Devices are registered in the global `devices` Map:

```typescript
export const devices = new Map<string, Device>([
    ["console", new ConsoleDevice({
        ctrl_addr: 0xF0000000,
        status_addr: 0xF0000004,
        data: { start: 0xF0000008, end: 0xF000000F },
        enabled: true
    })],
    ["os", new OSDriver({
        ctrl_addr: 0xF0000010,
        status_addr: 0xF0000014,
        data: { start: 0xF0000018, end: 0xF000001F },
        enabled: true
    })]
]);
```

### Address Checking

When memory is accessed, CREATOR checks if it's a device address:

```typescript
export function checkDeviceAddr(addr: number): string | null {
    for (const [id, device] of devices) {
        if (!device.enabled) continue;
        if (device.isDeviceAddr(addr)) return id;
    }
    return null;
}
```

### Handler Invocation

Devices are "woken up" each cycle to process pending requests:

```typescript
export function handleDevices(): void {
    for (const [_id, device] of devices) {
        if (device.enabled) {
            device.handler();
        }
    }
}
```

Called from the main execution loop after each instruction.

### Reset

Devices can be reset to initial state:

```typescript
export function resetDevices(): void {
    for (const [_id, device] of devices) {
        device.reset();
    }
}
```

## Creating Custom Devices

### Step 1: Define Device Class

```typescript
class MyCustomDevice extends Device {
    handler(): void {
        // Read control register
        const ctrlValue = this.readValue(this.ctrl_addr).getUint32(0);
        
        switch (ctrlValue) {
            case 1:
                // Handle operation 1
                const data = this.readValue(this.data.start);
                // Process data...
                break;
                
            case 2:
                // Handle operation 2
                // ...
                break;
        }
        
        // Clear control register when done
        this.clear();
    }
}
```

### Step 2: Register Device

```typescript
// In devices.mts, add to devices Map:
devices.set("mycustom", new MyCustomDevice({
    ctrl_addr: 0xF0000020,
    status_addr: 0xF0000024,
    data: { start: 0xF0000028, end: 0xF000002F },
    enabled: true
}));
```

### Step 3: Architecture Integration

Update architecture YAML to map system calls to device addresses:

```yaml
system_calls:
  - name: "my_custom_operation"
    code: 100
    device: "mycustom"
    control_code: 1
```

## Device Helper Methods

The `Device` base class provides utilities:

### writeValue

Write multi-byte value to device memory:

```typescript
protected writeValue(address: number, value: number, words: number = 1) {
    const bytes = this.memory.splitToBytes(BigInt(value));
    // Pad with zeros to fill word count
    bytes.unshift(...new Array(4 * words).fill(0));
    
    for (let i = 0; i < words; i++) {
        this.memory.writeWord(BigInt(address + i * 4), bytes);
    }
}
```

**Usage**:
```typescript
// Write 32-bit value
this.writeValue(this.data.start, 42);

// Write 64-bit value (2 words)
this.writeValue(this.data.start, bigValue, 2);
```

### readValue

Read multi-byte value as DataView:

```typescript
protected readValue(address: number, words: number = 1): DataView {
    let bytes: number[] = [];
    for (let i = 0; i < words; i++) {
        bytes = bytes.concat(
            this.memory.readWord(BigInt(address + i * 4))
        );
    }
    return new DataView(Uint8Array.from(bytes).buffer);
}
```

**Usage**:
```typescript
// Read 32-bit integer
const value = this.readValue(this.data.start).getInt32(0);

// Read 64-bit double
const double = this.readValue(this.data.start, 2).getFloat64(0);

// Read unsigned int
const unsigned = this.readValue(this.data.start).getUint32(0);
```

### clear

Clear control register:

```typescript
clear(): void {
    this.memory.writeWord(BigInt(this.ctrl_addr), [0, 0, 0, 0]);
}
```

Should be called after handling each request.

### isDeviceAddr

Check if address belongs to device:

```typescript
isDeviceAddr(addr: number): boolean {
    return (
        addr === this.ctrl_addr ||
        addr === this.status_addr ||
        (this.data.start <= addr && addr < this.data.end)
    );
}
```

## Example: Timer Device

Here's a complete example of a custom timer device:

```typescript
class TimerDevice extends Device {
    private startTime: number = 0;
    private interval: number = 0;
    
    handler(): void {
        const ctrl = this.readValue(this.ctrl_addr).getUint32(0);
        
        switch (ctrl) {
            case 1: {  // Start timer
                this.interval = this.readValue(this.data.start).getUint32(0);
                this.startTime = Date.now();
                break;
            }
            
            case 2: {  // Check timer
                const elapsed = Date.now() - this.startTime;
                const expired = elapsed >= this.interval;
                this.writeValue(this.data.start, expired ? 1 : 0);
                break;
            }
            
            case 3: {  // Stop timer
                this.startTime = 0;
                this.interval = 0;
                break;
            }
        }
        
        this.clear();
    }
}

// Register
devices.set("timer", new TimerDevice({
    ctrl_addr: 0xF0000030,
    status_addr: 0xF0000034,
    data: { start: 0xF0000038, end: 0xF000003F },
    enabled: true
}));
```

**Usage in Assembly**:
```
# Start 1000ms timer
li a0, 1000
li a7, 101    # Start timer syscall
ecall

# ... do work ...

# Check if timer expired
li a7, 102    # Check timer syscall
ecall
# a0 now contains 1 if expired, 0 if not
```

## Device Best Practices

### Design Principles

1. **Clear Interface**: Well-defined control codes
2. **Stateless When Possible**: Minimize internal state
3. **Atomic Operations**: Complete each operation fully
4. **Error Handling**: Validate inputs
5. **Documentation**: Explain behavior clearly

### Performance

1. **Lazy Evaluation**: Only process when control register set
2. **Efficient Handlers**: Keep handler() fast
3. **Minimal State**: Reduce memory footprint
4. **Clear Promptly**: Call clear() after handling

### Testing

1. **Unit Tests**: Test handler logic independently
2. **Integration Tests**: Test with actual programs
3. **Edge Cases**: Invalid inputs, boundary conditions
4. **Reset Behavior**: Verify reset() works correctly

## Device Debugging

### Enable Debug Logging

```typescript
class MyDevice extends Device {
    handler(): void {
        const ctrl = this.readValue(this.ctrl_addr).getUint32(0);
        console_log(`MyDevice: ctrl=${ctrl}`);
        
        // ... handle operations
    }
}
```

### Inspect Device State

In CLI:
```
CREATOR> mem 0xF0000000    # Control register
CREATOR> mem 0xF0000008 8  # Data buffer
```

### Common Issues

**Device not responding**:
- Check device is enabled
- Verify address range is correct
- Ensure handleDevices() is called

**Wrong data**:
- Verify endianness
- Check word alignment
- Validate read/write sizes

**Memory conflicts**:
- Ensure addresses don't overlap
- Check device address ranges
- Verify main memory doesn't extend into device space

## Next Steps

- Study [Interrupts](interrupts.md) for interrupt-driven devices
- Read [Core Architecture](core-architecture.md) for system integration
- See [Execution Engine](execution-engine.md) for execution flow
- Review [Memory System](memory-system.md) for memory operations
