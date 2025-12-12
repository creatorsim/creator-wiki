# CREATOR API (CAPI)

CAPI allows instruction definitions to interact with custom CREATOR functions.

> [!TIP]
> `CAPI` is a global variable, so you can access it from your browser's developer console if you're using the web application.


## Memory
Interaction with CREATOR's memory.


### `CAPI.MEM.write`
```ts
CAPI.MEM.write(
  address: bigint,
  bytes: number,
  value: bigint,
  reg_name?: string,
  hint?: string,
  noSegFault: boolean = true,
): void { }
```

Writes a specific `value` to the specified `address` and number of `bytes`. You can provide extra information about which register used to hold the value (`reg_name`), or any hint about what that value might be (`hint`). To enable checking if the address is in a writable segment, set `noSegFault` to false.

E.g.:
```js
CAPI.MEM.write(registers.t0, 1, registers.t0, "t0");
```


### `CAPI.MEM.read`
```ts
CAPI.MEM.read(
  address: bigint,
  bytes: number,
  reg_name: string,
  noSegFault: boolean = true,
): bigint { }
```

Reads a specific number of `bytes` in the specified `address` and returns them. You can provide extra information about which register will hold the value (`reg_name`). To enable checking if the address is in a readable segment, set `noSegFault` to false.

E.g.:
```js
registers.t0 = CAPI.MEM.read(registers.t1, 1, "t0");
```


<!--
### `CAPI.MEM.alloc`
> [!WARNING]
> This function is not currently implemented.

```ts
CAPI.MEM.alloc(bytes: number): number { }
```

Allocates the specified `size` number of bytes in the heap.
-->



### `CAPI.MEM.addHint`

```ts
CAPI.MEM.addHint(address: bigint, hint: string, sizeInBits?: number): boolean { }
```

Adds a `hint` (description of what the address holds) for the specified memory `address`. If a hint already exists at the specified address, it replaces it. You can optionally specify the size of the stored type in bits (`sizeInBits`). Returns `true` if the hint was successfully added, else `false`.

E.g.:
```js
CAPI.MEM.addHint(registers.f0, "float64", 64);
```


## System calls
CREATOR's system calls.


### `CAPI.SYSCALL.exit`

```ts
CAPI.SYSCALL.exit(): void { }
```

Terminates the execution of the program.

E.g.:
```js
CAPI.SYSCALL.exit();
```



### `CAPI.SYSCALL.print`

```ts
CAPI.SYSCALL.print(
  value: number | bigint,
  type: "int32" | "float" | "double" | "char" | "string",
): void { }
```

Prints the specified `value` of the specified `type` to the console.

Supported types are:
- `"int32"`: signed 32-bit integer
- `"float"` / `"double"`: [JS's Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)
- `"char"`: UTF-16 character value
- `"string"`: address of a null-terminated array of 1-byte chars

E.g.:
```js
CAPI.SYSCALL.print(registers.a0, "char");
```


### `CAPI.SYSCALL.read`

```ts
CAPI.SYSCALL.read(
  dest_reg_info: string,
  type: "int32" | "float" | "double" | "char" | "string",
  aux_info?: string,
): void { }
```

Reads the specified value `type` from the console and stores it in the specified register by name (`dest_reg_info`).

Supported types are:
- `"int32"`: signed 32-bit integer
- `"float"` / `"double"`: [JS's Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)
- `"char"`: UTF-16 character value
- `"string"`: `aux_info` is the name of the register that holds the address of a null-terminated array of 1-byte chars

E.g.:
```js
CAPI.SYSCALL.read("a0", "char");
CAPI.SYSCALL.read("a0", "string", "a1");
```



### `CAPI.SYSCALL.get_clk_cycles`

```ts
CAPI.SYSCALL.get_clk_cycles(): number { }
```

Returns the number of clock cycles that have passed since the program started.

E.g.:
```js
CAPI.SYSCALL.get_clk_cycles();
```


<!--
### `CAPI.SYSCALL.sbrk`

```ts
CAPI.SYSCALL.sbrk(value1: string, value2: string): void { }
```


E.g.:
```js
CAPI.SYSCALL.sbrk("a0", "v0");
```
-->



## Validation

### `CAPI.SYSCALL.raise`

```ts
CAPI.SYSCALL.raise(msg: string): never { }
```

Raises an error with a specific `msg`.

E.g.:
```js
CAPI.SYSCALL.raise("Help!");
```


### `CAPI.SYSCALL.isOverflow`

```ts
CAPI.SYSCALL.isOverflow(op1: bigint, op2: bigint, res_u: bigint): boolean { }
```

Checks if the result `res_u` of operating two operands `op1` and `op2` caused an overflow.

E.g.:
```js
CAPI.SYSCALL.isOverflow(registers.t0, registers.t1, registers.t0 + registers.t1);
```


## Stack
These functions are used for the stack tracker and sentinel modules, and are a way to tell CREATOR when a new function frame begins and ends. They should be included in the instructions that jump to, o return from, a routine, as is the case of RISC-V's `jal` and `jr` instructions.


### `CAPI.STACK.beginFrame`

```ts
CAPI.STACK.beginFrame(addr?: bigint): void { }
```

Marks the beginning of a function frame at `address`. If not specified, it takes the current (real) value of the program counter.

E.g.:
```js
registers.pc = ...
CAPI.SYSCALL.beginFrame();
```


### `CAPI.STACK.endFrame`

```ts
CAPI.STACK.endFrame(): void { }
```

Ends the current stack frame.

E.g.:
```js
registers.pc = ...
CAPI.SYSCALL.endFrame();
```



## Floating point


### `CAPI.FP.split_double`

```ts
CAPI.FP.split_double(reg: bigint, index: 0 | 1): string { }
```

Given a double precision IEEE 754 value `reg`, gets the 32-bits most significant (`index=1`) bits or the least significant bits (`index=0`). It returns it as a string of bits.

E.g.:
```js
const foo = CAPI.FP.split_double(registers.t0, 1);
```


### `CAPI.FP.uint2float32`

```ts
CAPI.FP.uint2float32(value: number): number { }
```

Transforms the unsigned integer `value` to a single precision IEEE754 [JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number).

E.g.:
```js
CAPI.FP.uint2float32(5);
```


### `CAPI.FP.float322uint`

```ts
CAPI.FP.float322uint(value: number): bigint { }
```

Transforms the single precision floating point ([JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)) `value` into an unsigned integer.

E.g.:
```js
CAPI.FP.uint2float32(5.0);
```


### `CAPI.FP.int2uint`

```ts
CAPI.FP.int2uint(
  value: number | bigint,
  bits: number = 64,
): bigint { }
```

Transforms a signed integer ([JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)) `value` into an unsigned integer of `bits` number of bits.

E.g.:
```js
CAPI.FP.int2uint(-5);
```


### `CAPI.FP.uint2int`

```ts
CAPI.FP.uint2int(value: number | bigint): bigint { }
```

Transforms an unsigned integer `value` into a signed integer ([JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)) of `bits` number of bits.

E.g.:
```js
CAPI.FP.uint2int(5);
```


### `CAPI.FP.uint2float64`

```ts
CAPI.FP.uint2float64(value0: number | bigint, value1?: number): number { }
```

Transforms an unsigned integer `value0` into a signed integer to a 64-bit float ([JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)). Supports two calling conventions:
1. Single `value0` argument: converts a 64-bit integer directly
2. Two 32-bit arguments: converts low (`value0`) and high (`value1`) 32-bit parts.

E.g.:
```js
CAPI.FP.uint2float64(5);
```


### `CAPI.FP.float642uint`

```ts
CAPI.FP.float642uint(value: number): [number, number] { }
```

Transforms the double precision floating point ([JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)) `value` into an unsigned integer.

E.g.:
```js
CAPI.FP.float642uint(5.0);
```


### `CAPI.FP.check_ieee`

```ts
CAPI.FP.check_ieee(sign: string, exponent: string, mantissa: string): number { }
```

Check the type of number is in IEEE 754 format. Returns a 10-bit mask where the position of the set bit indicates the type of the IEEE 754 number:
- 0 -> -inf
- 1 -> -normalized number
- 2 -> -non-normalized number
- 3 -> -0
- 4 -> +0
- 5 -> +non-normalized number
- 6 -> +normalized number
- 7 -> +inf
- 8 -> signaling NaN
- 9 -> quiet NaN

E.g.:
```js
CAPI.FP.check_iee(parseInt(a[0]), parseInt(a.slice(1,9), 2), parseInt(a.slice(10), 2));
```


### `CAPI.FP.float2bin`

```ts
CAPI.FP.float2bin(f: number): string { }
```

Transforms the single precision floating point [JS Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number) `f` into a binary string.

E.g.:
```js
CAPI.FP.float2bin(5.0);
```



## Registers


### `CAPI.REG.read`

```ts
CAPI.REG.read(name: string): bigint { }
```

Returns the value stored in register `name`.

E.g.:
```js
const foo = CAPI.REG.read("t0");
```


### `CAPI.REG.write`

```ts
CAPI.REG.write(value: bigint, name: string): void { }
```

Stores `value` in register `name`.

E.g.:
```js
CAPI.REG.write(0x69n, "t0");
```



## Architecture
`CAPI.ARCH` exposes the interface of the currently loaded [architecture plugin](custom-architectures.md#plugins).

The supported plugins are [`riscv`](#risc-v), [`mips`](#mips) and [`z80`](#z80).

### RISC-V

The RISC-V architecture plugin provides utilities for handling floating point operations and immediate value generation.

#### `generateLoadImmediate`

```ts
CAPI.ARCH.generateLoadImmediate(val: bigint | number, destReg: string): string { }
```

Generates a sequence of RISC-V instructions to load an immediate value into a register. Returns a semicolon-separated string of instructions that can load values of any size into the destination register.

E.g.:
```js
const instructions = CAPI.ARCH.generateLoadImmediate(0x12345678n, "t0");
// Returns: "lui t0, 0x12345;addiw t0, t0, 1656"
```


#### `toJSNumberD`

```ts
CAPI.ARCH.toJSNumberD(bigIntValue: bigint): [number, string] { }
```

Converts a 64-bit register value to a JavaScript number when the D (double-precision floating point) extension is enabled. Returns a tuple containing the numeric value and a type string indicating the representation.

The function handles:
- Canonical 64-bit NaN (`0x7ff8000000000000n`) → `[NaN, "NaN64"]`
- NaN-boxed 32-bit floats (upper 32 bits all 1's) → `[value, "NaNBfloat32_64"]`
- Valid 64-bit doubles → `[value, "float64"]`
- Zero → `[0, "float32"]`

E.g.:
```js
let value, type;
[value, type] = CAPI.ARCH.toJSNumberD(registers.ft0);
```


#### `toJSNumberS`

```ts
CAPI.ARCH.toJSNumberS(bigIntValue: bigint): [number, string] { }
```

Converts a 32-bit register value to a JavaScript number when only the S (single-precision floating point) extension is enabled. Returns a tuple containing the numeric value and a type string.

The function handles:
- Canonical 32-bit NaN (`0x7fc00000n`) → `[NaN, "NaN32"]`
- Valid 32-bit floats → `[value, "float32"]`
- Zero → `[0, "float32"]`

E.g.:
```js
let value, type;
[value, type] = CAPI.ARCH.toJSNumberS(registers.ft0);
```


#### `toBigInt`

```ts
CAPI.ARCH.toBigInt(number: number, type: string): bigint { }
```

Converts a JavaScript number to a bigint representation suitable for storing in a RISC-V register. The `type` parameter specifies the format.

Supported types:
- `"float32"`: Single-precision float
- `"float64"`: Double-precision float
- `"NaNBfloat32_64"`: NaN-boxed single-precision (upper 32 bits all 1's)
- `"NaN64"`: Canonical 64-bit NaN
- `"NaN32"`: Canonical 32-bit NaN

E.g.:
```js
registers.ft0 = CAPI.ARCH.toBigInt(3.14, "float32");
```


#### `NaNBox`

```ts
CAPI.ARCH.NaNBox(bigIntValue: bigint): bigint { }
```

NaN-boxes a 32-bit floating point value for use in 64-bit floating point registers. Sets the upper 32 bits to all 1's as per the RISC-V specification (section 21.2).

E.g.:
```js
const nanBoxed = CAPI.ARCH.NaNBox(registers.ft0);
```


### MIPS

The MIPS architecture plugin provides utilities for handling double-precision floating point operations across register pairs.

#### `validateEvenRegister`

```ts
CAPI.ARCH.validateEvenRegister(regName: string): void { }
```

Validates that a register number is even. Required for double-precision operations in MIPS, which use register pairs. Throws an error if the register is not even.

E.g.:
```js
CAPI.ARCH.validateEvenRegister("f0");  // OK
CAPI.ARCH.validateEvenRegister("f1");  // Throws error
```


#### `readDouble`

```ts
CAPI.ARCH.readDouble(regName: string): number { }
```

Reads a double-precision floating point value from a MIPS register pair. The `regName` must be even (e.g., `"f0"`, `"f2"`). The function reads from the specified register and the next sequential register to reconstruct a 64-bit double.

E.g.:
```js
const value = CAPI.ARCH.readDouble("f0");  // Reads f0 and f1
```


#### `writeDouble`

```ts
CAPI.ARCH.writeDouble(value: number, regName: string): void { }
```

Writes a double-precision floating point value to a MIPS register pair. The `regName` must be even. The function splits the value across the specified register and the next sequential register.

E.g.:
```js
CAPI.ARCH.writeDouble(3.14159, "f0");  // Writes to f0 and f1
```


#### `readDoublePair`

```ts
CAPI.ARCH.readDoublePair(reg1Name: string, reg2Name: string): number[] { }
```

Reads two double-precision values from register pairs and returns them as an array. Both register names must be even.

E.g.:
```js
const [val1, val2] = CAPI.ARCH.readDoublePair("f0", "f2");
```


#### `writeDoublePair`

```ts
CAPI.ARCH.writeDoublePair(value1: number, value2: number, reg1Name: string, reg2Name: string): void { }
```

Writes two double-precision values to register pairs. Both register names must be even.

E.g.:
```js
CAPI.ARCH.writeDoublePair(1.5, 2.5, "f0", "f2");
```


#### `binaryDoubleOperation`

```ts
CAPI.ARCH.binaryDoubleOperation(
  destReg: string,
  src1Reg: string,
  src2Reg: string,
  operation: (val1: number, val2: number) => number
): void { }
```

Performs a binary operation on two double-precision values. Reads from source register pairs, applies the operation function, and writes the result to the destination register pair. All register names must be even.

E.g.:
```js
CAPI.ARCH.binaryDoubleOperation("f0", "f2", "f4", (a, b) => a + b);
```


#### `unaryDoubleOperation`

```ts
CAPI.ARCH.unaryDoubleOperation(
  destReg: string,
  srcReg: string,
  operation: (val: number) => number
): void { }
```

Performs a unary operation on a double-precision value. Reads from the source register pair, applies the operation function, and writes the result to the destination register pair. Both register names must be even.

E.g.:
```js
CAPI.ARCH.unaryDoubleOperation("f0", "f2", (x) => Math.sqrt(x));
```


### Z80

The Z80 architecture plugin provides utilities for flag calculations, keyboard handling, and I/O operations.

#### Flag Constants

The following flag bit masks are available:

```js
CAPI.ARCH.S_FLAG   // 0x80 - Sign flag
CAPI.ARCH.Z_FLAG   // 0x40 - Zero flag
CAPI.ARCH.H_FLAG   // 0x10 - Half-carry flag
CAPI.ARCH.PV_FLAG  // 0x04 - Parity/Overflow flag
CAPI.ARCH.N_FLAG   // 0x02 - Add/Subtract flag
CAPI.ARCH.C_FLAG   // 0x01 - Carry flag
```


#### State Variables

The following variables track the Z80 system state:

```js
CAPI.ARCH.borderColor     // Border color (0-7) for ZX Spectrum
CAPI.ARCH.interruptMode   // Current interrupt mode (0, 1, or 2)
CAPI.ARCH.interruptPin    // Interrupt pin state (0 = low, 1 = high)
CAPI.ARCH.timerCounter    // Timer counter value (bigint)
CAPI.ARCH.keyState        // Object mapping event.code strings to pressed state (boolean)
CAPI.ARCH.keyMap          // Keyboard matrix mapping port high bytes to key codes
```

E.g.:
```js
// Set interrupt mode 2
CAPI.ARCH.interruptMode = 2;

// Check if a key is pressed
if (CAPI.ARCH.keyState["KeyA"]) {
  // Key A is pressed
}

// Set border color to red
CAPI.ARCH.borderColor = 2n;
```


#### `calculateFlags_INC`

```ts
CAPI.ARCH.calculateFlags_INC(oldValue: bigint, initialF: bigint): bigint { }
```

Calculates flags for an 8-bit INC (increment) operation. Returns the new F register value with appropriate flags set. The C flag is preserved from `initialF`.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_INC(registers.A, registers.F);
```


#### `calculateFlags_DEC`

```ts
CAPI.ARCH.calculateFlags_DEC(oldValue: bigint, initialF: bigint): bigint { }
```

Calculates flags for an 8-bit DEC (decrement) operation. Returns the new F register value. The C flag is preserved, and N flag is always set.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_DEC(registers.A, registers.F);
```


#### `calculateFlags_ADD`

```ts
CAPI.ARCH.calculateFlags_ADD(val1: bigint, val2: bigint): bigint { }
```

Calculates flags for 8-bit addition (ADD). Returns the new F register value with all flags computed based on the operation.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_ADD(registers.A, registers.B);
```


#### `calculateFlags_ADC`

```ts
CAPI.ARCH.calculateFlags_ADC(val1: bigint, val2: bigint, initialF: bigint): bigint { }
```

Calculates flags for 8-bit addition with carry (ADC). The carry bit is extracted from `initialF`.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_ADC(registers.A, registers.B, registers.F);
```


#### `calculateFlags_SUB`

```ts
CAPI.ARCH.calculateFlags_SUB(val1: bigint, val2: bigint): bigint { }
```

Calculates flags for 8-bit subtraction (SUB). The N flag is always set for subtraction operations.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_SUB(registers.A, registers.B);
```


#### `calculateFlags_SBC`

```ts
CAPI.ARCH.calculateFlags_SBC(val1: bigint, val2: bigint, initialF: bigint): bigint { }
```

Calculates flags for 8-bit subtraction with carry (SBC). The carry bit is extracted from `initialF`.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_SBC(registers.A, registers.B, registers.F);
```


#### `calculateFlags_LOGICAL`

```ts
CAPI.ARCH.calculateFlags_LOGICAL(result: bigint, setH: number): bigint { }
```

Calculates flags for logical operations (AND, OR, XOR). Set `setH` to `1` for AND operations (sets H flag), or `0` for OR/XOR operations (resets H flag). N and C flags are always reset.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_LOGICAL(registers.A & registers.B, 1);
```


#### `calculateFlags_CP`

```ts
CAPI.ARCH.calculateFlags_CP(val1: bigint, val2: bigint): bigint { }
```

Calculates flags for the compare (CP) operation. Performs a subtraction for flag calculation purposes without storing the result.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_CP(registers.A, registers.B);
```


#### `calculateFlags_ADD16`

```ts
CAPI.ARCH.calculateFlags_ADD16(val1: bigint, val2: bigint, initialF: bigint): bigint { }
```

Calculates flags for 16-bit addition (ADD HL, rr). The S, Z, and P/V flags are preserved from `initialF`. N flag is reset.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_ADD16(registers.HL, registers.BC, registers.F);
```


#### `calculateFlags_SBC16`

```ts
CAPI.ARCH.calculateFlags_SBC16(val1: bigint, val2: bigint, initialF: bigint): bigint { }
```

Calculates flags for 16-bit subtraction with carry (SBC HL, rr). The carry bit is extracted from `initialF`.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_SBC16(registers.HL, registers.BC, registers.F);
```


#### `calculateFlags_BIT`

```ts
CAPI.ARCH.calculateFlags_BIT(value: bigint, bit: number, initialF: bigint): bigint { }
```

Calculates flags for the BIT instruction, which tests a specific bit (0-7) in a value. The C flag is preserved from `initialF`, and H flag is always set. N flag is always reset.

E.g.:
```js
registers.F = CAPI.ARCH.calculateFlags_BIT(registers.A, 7, registers.F);
```


#### `calculateFlags_ROTATE`

```ts
CAPI.ARCH.calculateFlags_ROTATE(result: bigint, carry: bigint): bigint { }
```

Calculates flags for rotate and shift instructions (RLC, RRC, SLA, etc.). The `carry` parameter should be `0n` or `1n` representing the bit that was shifted out. H and N flags are always reset.

E.g.:
```js
const carry = registers.A >> 7n;
const result = ((registers.A << 1n) | carry) & 0xffn;
registers.F = CAPI.ARCH.calculateFlags_ROTATE(result, carry);
```


#### `calculateFlags_SRL`

```ts
CAPI.ARCH.calculateFlags_SRL(result: bigint, carry: bigint): bigint { }
```

Calculates flags for the SRL (Shift Right Logical) instruction. The `carry` parameter is the original bit 0 that was shifted out. S, H, and N flags are always reset.

E.g.:
```js
const carry = registers.A & 1n;
const result = registers.A >> 1n;
registers.F = CAPI.ARCH.calculateFlags_SRL(result, carry);
```


#### `pressKey`

```ts
CAPI.ARCH.pressKey(code: string): void { }
```

Registers a keyboard key as being pressed. The `code` parameter should be a JavaScript `event.code` string (e.g., `"KeyA"`, `"Digit1"`).

E.g.:
```js
CAPI.ARCH.pressKey("KeyA");
```


#### `releaseKey`

```ts
CAPI.ARCH.releaseKey(code: string): void { }
```

Registers a keyboard key as being released.

E.g.:
```js
CAPI.ARCH.releaseKey("KeyA");
```


#### `readULAKeyboard`

```ts
CAPI.ARCH.readULAKeyboard(port: bigint): bigint { }
```

Emulates a read from the ZX Spectrum ULA keyboard port. The high byte of the port address determines which half-row of keys to poll. Returns an 8-bit value where bits 0-4 represent key states (0 = pressed, 1 = not pressed), and bits 5-7 are typically high.

E.g.:
```js
const keyState = CAPI.ARCH.readULAKeyboard(0xf7fen);  // Read row with keys 1-5
```


#### `read`

```ts
CAPI.ARCH.read(port: bigint): bigint { }
```

Reads a byte from an I/O port. Handles:
- Port `0x01`: Keyboard buffer (non-blocking)
- ULA ports (keyboard matrix): ZX Spectrum keyboard
- Unhandled ports: Returns `0xFF`

E.g.:
```js
const value = CAPI.ARCH.read(0x01n);  // Read from keyboard buffer
```


#### `write`

```ts
CAPI.ARCH.write(port: bigint, value: bigint): void { }
```

Writes a byte to an I/O port. Handles:
- Port `0x02`: Screen output
- Port `0xFE`: ZX Spectrum ULA (sets border color from bits 0-2)
- Port `0x17`: CREATOR-specific port for ecall (exits on value `10n`)

E.g.:
```js
CAPI.ARCH.write(0x02n, 65n);  // Write 'A' to screen
CAPI.ARCH.write(0xfen, 0x02n);  // Set border color to red
```



## Interrupts
Functions to manage interrupts and privilege modes.

For more information, see [Interrupt Support](custom-architectures.md#interrupt-support).


### `CAPI.INTERRUPTS.setUserMode`

```ts
CAPI.INTERRUPTS.setUserMode(): void { }
```

Sets the privilege level to User.

E.g.:
```js
CAPI.INTERRUPTS.setUserMode();
```


### `CAPI.INTERRUPTS.setKernelMode`

```ts
CAPI.INTERRUPTS.setKernelMode(): void { }
```

Sets the privilege level to Kernel.

E.g.:
```js
CAPI.INTERRUPTS.setKernelMode();
```


### `CAPI.INTERRUPTS.create`

```ts
CAPI.INTERRUPTS.create(type: InterruptType): void { }
```

Creates an interrupt of the specified `type`.

E.g.:
```js
CAPI.INTERRUPTS.create(InterruptType.Software);
```


### `CAPI.INTERRUPTS.enable`

```ts
CAPI.INTERRUPTS.enable(type: InterruptType): void { }
```

Enables an interrupt `type`.

E.g.:
```js
CAPI.INTERRUPTS.enable(InterruptType.Software);
```


### `CAPI.INTERRUPTS.globalEnable`

```ts
CAPI.INTERRUPTS.globalEnable(): void { }
```

Globally enables interrupts.

E.g.:
```js
CAPI.INTERRUPTS.globalEnable();
```


### `CAPI.INTERRUPTS.disable`

```ts
CAPI.INTERRUPTS.disable(type: InterruptType): void { }
```

Disables an interrupt `type`.

E.g.:
```js
CAPI.INTERRUPTS.disable(InterruptType.Software);
```


### `CAPI.INTERRUPTS.globalDisable`

```ts
CAPI.INTERRUPTS.globalDisable(): void { }
```

Globally disables interrupts.

E.g.:
```js
CAPI.INTERRUPTS.globalDisable();
```


### `CAPI.INTERRUPTS.isEnabled`

```ts
CAPI.INTERRUPTS.isEnabled(type: InterruptType): boolean { }
```

Checks if an interrupt `type` is enabled.

E.g.:
```js
CAPI.INTERRUPTS.isEnabled(InterruptType.Software);
```


### `CAPI.INTERRUPTS.isGlobalEnabled`

```ts
CAPI.INTERRUPTS.isGlobalEnabled(): boolean { }
```

Checks if interrupts are globally enabled.

E.g.:
```js
CAPI.INTERRUPTS.isGlobalEnabled();
```


### `CAPI.INTERRUPTS.clear`

```ts
CAPI.INTERRUPTS.clear(type: InterruptType): void { }
```

Clears interrupts of the specified `type`.

E.g.:
```js
CAPI.INTERRUPTS.clear(InterruptType.Software);
```


### `CAPI.INTERRUPTS.globalClear`

```ts
CAPI.INTERRUPTS.globalClear(): void { }
```

Clears all interrupts.

E.g.:
```js
CAPI.INTERRUPTS.globalClear();
```


### `CAPI.INTERRUPTS.setCustomHandler`

```ts
CAPI.INTERRUPTS.setCustomHandler(): void { }
```

Sets the interrupt handler to the custom handler.

E.g.:
```js
CAPI.INTERRUPTS.setCustomHandler();
```


### `CAPI.INTERRUPTS.setCREATORHandler`

```ts
CAPI.INTERRUPTS.setCREATORHandler(): void { }
```

Sets the interrupt handler to the default CREATOR handler.

E.g.:
```js
CAPI.INTERRUPTS.setCREATORHandler();
```


### `CAPI.INTERRUPTS.setHighlight`

```ts
CAPI.INTERRUPTS.setHighlight(): void { }
```

Highlights the current instruction as "interrupted" in the UI.

E.g.:
```js
CAPI.INTERRUPTS.setHighlight();
```


### `CAPI.INTERRUPTS.clearHighlight`

```ts
CAPI.INTERRUPTS.setHighlight(): void { }
```

Removes the "interrupted" highlight in the UI. Typically used in instructions that return from the interrupt handler, such as RISC-V's `mret`.

E.g.:
```js
CAPI.INTERRUPTS.clearHighlight();
```




