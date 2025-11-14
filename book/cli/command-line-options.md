# Command-Line Options

The CREATOR CLI accepts various command-line options to configure architecture, input files, and behavior.

## Basic Usage

```bash
creator [options]
```

## Required Options

### `--architecture`, `-a`

**Required**: Yes  
**Type**: String (path to YAML file)  
**Description**: Path to the architecture definition file

Specifies which processor architecture to simulate.

**Examples**:
```bash
# RISC-V architecture
creator --architecture architecture/riscv32.yml --assembly program.s

# MIPS architecture
creator -a architecture/mips32.yml -s program.s

# Z80 architecture
creator -a architecture/z80.yml -s program.s
```

**Architecture Files**:
- `riscv32.yml` - RISC-V 32-bit (RV32I + extensions)
- `mips32.yml` - MIPS 32-bit
- `z80.yml` - Z80 8-bit processor
- Custom YAML files for your own architectures

## Input Options

### `--assembly`, `-s`

**Type**: String (path to assembly file)  
**Description**: Assembly source file to assemble and load

Assembles the specified file and loads it into memory.

**Example**:
```bash
creator -a riscv32.yml --assembly hello.s
```

**Behavior**:
1. Reads the assembly source file
2. Assembles using selected compiler
3. Loads into simulator memory
4. Enters interactive mode

### `--bin`, `-b`

**Type**: String (path to binary file)  
**Description**: Binary file to load directly into memory

Loads a pre-assembled binary file without compilation.

**Example**:
```bash
creator -a riscv32.yml --bin program.bin
```

**Use Cases**:
- Load binaries from external assemblers
- Test pre-compiled code
- Load ROM images
- Skip assembly step

**Note**: Binary and assembly options are mutually exclusive. If both are provided, binary takes precedence.

### `--library`, `-l`

**Type**: String (path to library file)  
**Description**: Library file to load before assembly

Loads a library of pre-assembled code that your program can reference.

**Example**:
```bash
creator -a riscv32.yml -l stdlib.lib -s program.s
```

**Use Cases**:
- Standard library functions
- Shared code modules
- Pre-compiled utilities

## Compilation Options

### `--compiler`, `-C`

**Type**: String  
**Default**: `default`  
**Description**: Assembler backend to use

Selects which assembler to use for compilation.

**Available Options**:
- `default` - CREATOR native assembler
- `sjasmplus` - Z80 assembler with macro support
- `rasm` - Alternative Z80 assembler

**Examples**:
```bash
# Use default CREATOR assembler
creator -a riscv32.yml -s program.s

# Use sjasmplus for Z80
creator -a z80.yml -s program.s --compiler sjasmplus

# Use rasm for Z80
creator -a z80.yml -s program.s -C rasm
```

**Compatibility**:
- `default`: Works with all architectures
- `sjasmplus`, `rasm`: Z80 only

### `--isa`, `-i`

**Type**: Array of strings  
**Default**: `[]`  
**Description**: ISA extensions to load

Loads additional instruction set extensions for architectures that support them.

**RISC-V Extensions**:
- `M` - Integer multiplication and division
- `F` - Single-precision floating-point
- `D` - Double-precision floating-point
- `A` - Atomic instructions
- `C` - Compressed instructions

**Examples**:
```bash
# RISC-V with M extension (multiply/divide)
creator -a riscv32.yml --isa M -s program.s

# RISC-V with multiple extensions
creator -a riscv32.yml --isa M F D -s program.s

# RISC-V base only (no extensions)
creator -a riscv32.yml -s program.s
```

## Mode Options

### `--tutorial`, `-T`

**Type**: Boolean  
**Default**: `false`  
**Description**: Start interactive tutorial mode

Launches the built-in interactive tutorial for learning RISC-V assembly.

**Example**:
```bash
creator --tutorial -a riscv32.yml
```

**Features**:
- Step-by-step lessons
- Hands-on exercises
- Built-in test program
- Interactive prompts
- Progress tracking

**Note**: When tutorial mode is active, assembly/binary options are ignored as the tutorial provides its own program.

See [Tutorial Mode](tutorial.md) for details.

### `--accessible`, `-A`

**Type**: Boolean  
**Default**: `false`  
**Description**: Enable accessible mode for screen readers

Disables colors, ASCII art, and fancy formatting for compatibility with screen readers.

**Example**:
```bash
creator --accessible -a riscv32.yml -s program.s
```

**Changes in Accessible Mode**:
- No ANSI color codes
- Plain text output only
- No ASCII art
- Descriptive text for all information
- Structured table layouts

**Note**: Can also be set in configuration file. Command-line flag overrides config.

## Configuration Options

### `--config`, `-c`

**Type**: String (path to YAML file)  
**Default**: `~/.config/creator/config.yml`  
**Description**: Path to configuration file

Specifies a custom configuration file instead of the default location.

**Example**:
```bash
creator -a riscv32.yml -s program.s --config custom-config.yml
```

**Use Cases**:
- Different configurations for different projects
- Shared team configurations
- Testing configuration changes

See [Configuration](configuration.md) for config file format.

## Debugging Options

### `--debug`, `-v`

**Type**: Boolean  
**Default**: `false`  
**Description**: Enable debug mode with verbose logging

Enables detailed internal logging for debugging CREATOR itself.

**Example**:
```bash
creator --debug -a riscv32.yml -s program.s
```

**Output Includes**:
- Architecture loading details
- Assembly compilation steps
- Instruction decoding
- Memory operations
- Register updates

**Note**: Very verbose, useful for troubleshooting CREATOR bugs or understanding internal behavior.

## State Management Options

### `--state`

**Type**: String (path to JSON file)  
**Default**: None  
**Description**: File to save execution state on exit

Automatically saves simulator state to specified file when program exits.

**Example**:
```bash
creator -a riscv32.yml -s program.s --state mystate.json
```

**Use Cases**:
- Preserve debugging session
- Save checkpoint before exit
- Automatic state backup

**Note**: Use `restore` command in interactive mode to load saved states.

### `--reference`, `-r`

**Type**: String (path to file)  
**Default**: None  
**Description**: Reference file for validation (internal use)

Used internally for testing and validation. Not typically used by end users.

## Complete Examples

### Basic Assembly Execution
```bash
creator -a architecture/riscv32.yml -s hello.s
```

### Binary Loading
```bash
creator -a architecture/mips32.yml -b program.bin
```

### With Library and Extensions
```bash
creator -a architecture/riscv32.yml \
  --isa M F D \
  -l stdlib.lib \
  -s program.s
```

### Z80 with Custom Assembler
```bash
creator -a architecture/z80.yml \
  --compiler sjasmplus \
  -s game.asm
```

### Tutorial Mode
```bash
creator --tutorial -a architecture/riscv32.yml
```

### Accessible Mode
```bash
creator --accessible \
  -a architecture/riscv32.yml \
  -s program.s
```

### Debug Mode with State Save
```bash
creator --debug \
  -a architecture/riscv32.yml \
  -s program.s \
  --state debug-session.json
```

### Full-Featured Command
```bash
creator \
  --architecture architecture/riscv32.yml \
  --isa M F D \
  --library stdlib.lib \
  --assembly program.s \
  --compiler default \
  --config custom-config.yml \
  --accessible
```

## Option Priority

When multiple options conflict:

1. **Command-line flags** override config file settings
2. **Binary loading** (--bin) takes precedence over assembly (--assembly)
3. **Tutorial mode** (--tutorial) ignores assembly/binary options
4. **Custom config** (--config) overrides default config location

## Getting Help

Show all available options:
```bash
creator --help
```

## Common Patterns

### Development Workflow
```bash
# Edit program.s
# Test with:
creator -a riscv32.yml -s program.s
# Debug interactively
# Fix program.s
# Repeat
```

### Testing Different Architectures
```bash
# Test on RISC-V
creator -a riscv32.yml -s program.s

# Test on MIPS
creator -a mips32.yml -s program.s

# Test on Z80
creator -a z80.yml -s program.s
```

### Library Development
```bash
# Test library functions
creator -a riscv32.yml -l mylib.lib -s test.s
```

## Next Steps

- Learn [Interactive Mode](interactive-mode.md) commands
- Explore [Configuration](configuration.md) options
- Try [Tutorial Mode](tutorial.md)
- See [Examples](examples.md) for practical uses