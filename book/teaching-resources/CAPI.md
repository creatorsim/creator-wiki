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
CAPI.MEM.write(0x12n, 1, 0x200000n, "t0");
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
CAPI.MEM.read(0x200000n, 1, "t0");
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
CAPI.MEM.addHint(0x200000n, "float64", 64);
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
CAPI.SYSCALL.print(0x45n, "char");
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


<!--
### Exceptions

* capi_raise (msg) &rarr; Show an error message.
  ```javascript
  capi_raise('Problem detected :-(') ;
  ```

* capi_arithmetic_overflow ( op1, op2, res_u ) &rarr; Checks if there is some arithmetic overflow.
  ```javascript
  var isover = capi_arithmetic_overflow(rs1, inm, rs1+inm);
  if (isover)
       { capi_raise('Integer Overflow'); }
  else { rd = rs1 + inm; }
  ```

* capi_bad_align ( addr, type ) &rarr; Checks if address is aligned for this architecture.
  ```javascript
  var isnotalign = capi_bad_align(rs1+inm, 'd');
  if (isnotalign) { capi_raise('The memory must be align'); }
  ```


### Memory access

* capi_mem_write ( destination_address, value2store, byte_or_half_or_word, reg_name ) &rarr; Store a value into an address.
  ```javascript
  capi_mem_write(base+off+4, val, 'w', rt_name);
  ```

* capi_mem_read ( source_address, byte_or_half_or_word, reg_name ) &rarr; Read a value from an address.
  ```javascript
  capi_mem_read(0x12345, 'b', rt_name) ;
  ```


### Syscalls

* capi_exit () &rarr; System call to stop the execution.

* capi_print_int ( register_id ) &rarr; System call for printing an integer.

* capi_print_float ( register_id ) &rarr; System call for printing a float.

* capi_print_double ( register_id ) &rarr; System call for printing a double.

* capi_print_char ( register_id ) &rarr; System call for printing a char.

* capi_print_string ( register_id ) &rarr; System call for printing a string.

* capi_read_int ( register_id ) &rarr;  System call for reading an integer.

* capi_read_float ( register_id ) &rarr; System call for reading a float.

* capi_read_double ( register_id ) &rarr; System call for reading a double.

* capi_read_char ( register_id ) &rarr; System call for reading a char.

* capi_read_string ( register_id, register_id_2 ) &rarr; System call for reading a string.

* capi_sbrk ( value1, value2 ) &rarr; System call for allocating memory.

* capi_get_clk_cycles ( ) &rarr; Get CLK Cylces.
  
#### Syscall examples (ecall RISC-V)  
  
  ```javascript
  switch(a7) {
    case 1:  capi_print_int('a0');
             break;
    case 2:  capi_print_float('fa0');
             break;
    case 3:  capi_print_double('fa0');
             break;
    case 4:  capi_print_string('a0');
             break;
    case 5:  capi_read_int('a0');
             break;
    case 6:  capi_read_float('fa0');
             break;
    case 7:  capi_read_double('fa0');
             break;
    case 8:  capi_read_string('a0', 'a1');
             break;
    case 9:  capi_sbrk('a0', 'a0');
             break;
    case 10: capi_exit();
             break;
    case 11: capi_print_char('a0');
             break;
    case 12: capi_read_char('a0');
             break;
  }
  ```


### Check stack

* capi_callconv_begin ( addr ) &rarr; Save current state at the CPU that must be preserved by calling convention.
  ```javascript
  capi_callconv_begin(inm)
  ```

* capi_callconv_end () &rarr; Checks if the current state at the CPU is the same as when capi_callconv_begin was called.
  ```javascript
  capi_callconv_end();
  ```


### Draw stack

* capi_drawstack_begin ( addr ) &rarr; It updates in the User Interface the current stack trace.
  ```javascript
  capi_drawstack_begin(inm);
  ```

* capi_drawstack_end () &rarr; It updates in the User Interface the current stack trace.
  ```javascript
  capi_drawstack_end() ;
  ```


### Representation

* capi_int2uint ( value ) &rarr; Signed integer to unsigned integer
  ```javascript
  capi_int2uint(5) ;
  ```

* capi_uint2int ( value ) &rarr; Unsigned integer to signed integer.
  ```javascript
  capi_uint2int(5) ;
  ```
  
* capi_uint2float32 ( value ) &rarr; Unsigned integer to simple precision IEEE754.
  ```javascript
  capi_uint2float32(5) ;
  ```
  
* capi_float322uint ( value ) &rarr; Simple precision IEEE754 to unsigned integer.
  ```javascript
  capi_float322uint(5) ;
  ```
  
* capi_uint2float64 ( value0, value1 ) &rarr; Unsigned integer to double precision IEEE754.
  ```javascript
  capi_uint2float64 ( value0, value1 )
  ```

* capi_float642uint ( value ) &rarr; Double precision IEEE754 to unsigned integer.
  ```javascript
  capi_float642uint(5) ;
  ```
  
* capi_float2bin ( f ) &rarr; Simple precision IEEE754 to binary.
  ```javascript
  capi_float2bin(5) ;
  let a = capi_float2bin(rs1);
  ```
  
* capi_split_double ( reg, index ) &rarr; Given a double precision IEEE 754 value, get the 32-bits most significant (index=1) bits or the least significant bits (index=0).
  ```javascript
  var val = capi_split_double(ft, 0);
  ```

* capi_check_ieee ( sign, exponent, mantissa ) &rarr; Indicates the type of an IEEE value:
  * 0 &rarr; -infinite
  * 1 &rarr; -normalized number
  * 2 &rarr; -non-normalized number
  * 3 &rarr; -0
  * 4 &rarr; +0
  * 5 &rarr; +normalized number
  * 6 &rarr; +non-normalized number
  * 7 &rarr; +inf
  * 8 &rarr; -NaN
  * 9 &rarr; +NaN
  
  <br />
  
  ```javascript
  rd = capi_check_ieee(parseInt(a[0]), parseInt(a.slice(1,9), 2), parseInt(a.slice(10), 2));
  ```
 -->
