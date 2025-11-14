# Assembler Integration

CREATOR supports multiple assemblers through a standardized integration framework. This allows adding new assemblers for different architectures and syntaxes.

## Integration Overview

Source: `src/core/assembler/README.md` and actual implementations

### Compilation Targets

CREATOR assemblers support two targets:

1. **Web**: Compiled to WebAssembly for browser use
2. **Deno/Node**: Native executable called from CLI

### Folder Structure

```
src/core/assembler/
├── <assembler_name>/
│   ├── deno/
│   │   └── <assembler_name>.mjs
│   ├── web/
│   │   ├── wasm/
│   │   │   ├── <assembler_name>.js
│   │   │   └── <assembler_name>.wasm
│   │   └── <assembler_name>.mjs
│   └── utils.mjs  (optional)
```

## Assembler Requirements

### Core Responsibilities

1. **Parse source code**: Tokenize and analyze assembly
2. **Resolve symbols**: Calculate label addresses
3. **Generate machine code**: Produce binary output
4. **Load memory**: Populate simulator memory
5. **Create instruction list**: For debugging/display
6. **Generate symbol table**: Map labels to addresses

### Required Outputs

After assembly completes:
- `main_memory`: Loaded with binary code
- `instructions`: Array of decoded instruction objects
- `tag_instructions`: Map of label → address

## Built-in Assemblers

### CREATOR Native Assembler

**Features**:
- Full CREATOR integration
- Memory hints support
- Pseudo-instruction expansion
- All architectures

**Location**: `src/core/assembler/assembler.mjs`

### sjasmplus (Z80)

**Features**:
- Macro support
- Conditional assembly
- Include files
- Z80 specific

**Location**: `src/core/assembler/sjasmplus/`

### rasm (Z80)

**Features**:
- Fast compilation
- Modern syntax
- Z80 optimized

**Location**: `src/core/assembler/rasm/`

## Implementation Guide

### Web Version

**File**: `web/<assembler>.mjs`

```javascript
export async function assembleWeb(sourceCode) {
    // 1. Load WebAssembly module
    const module = await loadWasmModule();
    
    // 2. Create virtual file system
    module.FS.writeFile('/input.s', sourceCode);
    
    // 3. Run assembler
    module.callMain(['-o', '/output.bin', '/input.s']);
    
    // 4. Read outputs
    const binary = module.FS.readFile('/output.bin');
    const symbols = module.FS.readFile('/output.sym');
    
    // 5. Load into CREATOR
    main_memory.loadROM(binary, 0n);
    
    // 6. Decode instructions
    precomputeInstructions(sourceCode, symbols);
    
    // 7. Build symbol table
    set_tag_instructions(parseSymbols(symbols));
    
    return { status: 'ok' };
}
```

### Deno/Node Version

**File**: `deno/<assembler>.mjs`

```javascript
import { spawnSync } from 'child_process';
import fs from 'fs';
import os from 'os';
import path from 'path';

export function assembleDeno(sourceCode) {
    // 1. Create temp file
    const tmpDir = os.tmpdir();
    const inputFile = path.join(tmpDir, 'input.s');
    const outputFile = path.join(tmpDir, 'output.bin');
    
    fs.writeFileSync(inputFile, sourceCode);
    
    // 2. Execute assembler
    const result = spawnSync('assembler-executable', [
        '-o', outputFile,
        inputFile
    ]);
    
    if (result.status !== 0) {
        return {
            status: 'error',
            msg: result.stderr.toString()
        };
    }
    
    // 3. Read outputs
    const binary = fs.readFileSync(outputFile);
    const symbols = fs.readFileSync(outputFile + '.sym', 'utf8');
    
    // 4. Load into CREATOR
    main_memory.loadROM(new Uint8Array(binary), 0n);
    
    // 5. Decode and populate
    precomputeInstructions(sourceCode, symbols);
    set_tag_instructions(parseSymbols(symbols));
    
    // 6. Cleanup
    fs.unlinkSync(inputFile);
    fs.unlinkSync(outputFile);
    fs.unlinkSync(outputFile + '.sym');
    
    return { status: 'ok' };
}
```

## Instruction Decoding

After loading binary into memory, decode for display:

```javascript
function precomputeInstructions(source, symbolMap, symbols) {
    const instructions = [];
    let address = TEXT_START;
    
    while (address < TEXT_END) {
        const binary = main_memory.readWord(address);
        const decoded = decode(binary);
        
        instructions.push({
            Address: '0x' + address.toString(16),
            Label: symbolMap.get(address) || null,
            loaded: decoded.asm,
            user: source[lineMap.get(address)],
            Break: false
        });
        
        address += WORD_SIZE;
    }
    
    setInstructions(instructions);
}
```

## Symbol Table

Map labels to addresses:

```javascript
function parseSymbols(symbolFile) {
    const symbols = {};
    const lines = symbolFile.split('\n');
    
    for (const line of lines) {
        // Parse format: "label = 0x00400000"
        const match = line.match(/(\w+)\s*=\s*0x([0-9a-fA-F]+)/);
        if (match) {
            symbols[match[1]] = parseInt(match[2], 16);
        }
    }
    
    return symbols;
}
```

## Registering Assemblers

### CLI (`src/cli/creator6.mts`)

```typescript
const assembler_map = {
    default: assembleCreator,
    sjasmplus: sjasmplusAssemble,
    rasm: rasmAssemble,
    // Add new assembler here
    custom: customAssemble
};
```

### Web (`src/web/assemblers.ts`)

```typescript
export const assemblers = {
    'default': { name: 'CREATOR', fn: assembleCreator },
    'sjasmplus': { name: 'sjasmplus', fn: sjasmplusAssemble },
    'rasm': { name: 'rasm', fn: rasmAssemble },
    // Add new assembler here
    'custom': { name: 'Custom', fn: customAssemble }
};
```

## Error Handling

Return consistent error format:

```javascript
return {
    status: 'error',
    msg: 'Assembly error at line 42: Unknown instruction',
    line: 42,
    token: 'bad_instruction'
};
```

## Best Practices

1. **Validation**: Check source code before processing
2. **Error Messages**: Provide line numbers and context
3. **Cleanup**: Remove temporary files
4. **Performance**: Cache when possible
5. **Compatibility**: Test on all target platforms

## Testing

Test your assembler integration:

```javascript
// Test basic assembly
const result = await assemble(`
    .text
    main:
        li t0, 10
        add t1, t0, t0
`);
assert(result.status === 'ok');

// Test error handling
const errorResult = await assemble('invalid code');
assert(errorResult.status === 'error');

// Test symbol resolution
assert(tag_instructions['main'] === 0x00400000);
```

## Next Steps

- Study [Core Architecture](core-architecture.md) for integration points
- See [Execution Engine](execution-engine.md) for instruction handling
- Review existing assemblers for examples
- Read [Creating Custom Architectures](custom-architectures.md) for architecture integration
