# Interactive Mode

The CREATOR CLI includes a powerful interactive REPL (Read-Eval-Print Loop) that allows exploring program execution step by step.

## Starting Interactive Mode

```bash
# Load and enter interactive mode
creator6 program.s

# Load with architecture
creator6 -a mips32 program.s
```

## REPL Interface

### Command Prompt

![CLI Main](./img/cli_main.png)
*Figure: CREATOR CLI interactive prompt.*

### Command History

- **Up/Down Arrows**: Navigate command history
- **Ctrl+R**: Search history (if terminal supports)

## Basic Workflow

### 1. Load Program

```bash
creator6 fibonacci.s
```

Program is assembled and ready to execute.

### 2. Step Through Code

```bash
creator> step
```

Executes one instruction, shows next instruction.

### 3. Inspect State

```bash
creator> reg t0
creator> mem 0x10010000
creator> stack
```

### 4. Continue Execution

```bash
creator> run
```

Runs until breakpoint or completion.

### 5. Debugging

```bash
creator> break 0x00400010
creator> run
creator> reg
```

## Advanced Features

### Reverse Debugging

Step backwards through execution:

```bash
creator> step
creator> step
creator> unstep     # Go back one step
creator> nur 5      # Run back 5 steps
```

The CLI maintains history of previous states (configurable maximum).

### State Snapshots

Save and restore execution states:

```bash
creator> snapshot state1
creator> run
creator> restore state1   # Back to saved point
```

### Memory Hints

View memory addresses symbolically:

```bash
creator> mem 0x10010000 --hints
Output: array[0] = 42
```

### Breakpoints

Set multiple breakpoints:

```bash
creator> break 0x00400010
creator> break main_loop
creator> break fibonacci+8
creator> list
creator> run
```

### Keyboard Shortcuts

- **Enter**: Repeat last command
- **Ctrl+C**: Stop running program
- **Ctrl+D**: Exit CREATOR
- **Arrows**: Navigate history
- **Ctrl+L**: Clear screen (use `clear` command)

## Common Workflows

### Debugging Loop

```bash
# Load program
creator6 program.s

# Set breakpoint at function
creator> break calculate

# Run to breakpoint
creator> run

# Inspect arguments
creator> reg a0 a1 a2

# Step through function
creator> step
creator> step
creator> reg t0    # Check intermediate results

# View memory if needed
creator> mem $sp

# Continue to next breakpoint
creator> continue
```

### Finding Bugs

```bash
# Run until error
creator> run

# Check error location
creator> reg pc

# Rewind a few steps
creator> unstep
creator> unstep
creator> unstep

# Inspect state before error
creator> reg
creator> stack
creator> mem $sp

# Step forward slowly
creator> step
creator> reg
```

### Performance Analysis

```bash
# Run program
creator> silent

# View statistics
creator> stats
Instructions executed: 1234
Cycles: 5678
CPI: 4.60
```

## Command Reference

Quick reference for interactive mode commands:

### Execution Control
- `step [n]` - Execute n instructions
- `unstep [n]` - Reverse n instructions
- `run` - Run until completion/breakpoint
- `nur [n]` - Run backwards n instructions
- `silent` - Run without output
- `continue` - Resume after breakpoint
- `pause` - Interrupt running program (Ctrl+C)
- `reset` - Restart program

### Breakpoints
- `break <addr>` - Set breakpoint
- `list` - Show breakpoints
- `delete <n>` - Remove breakpoint

### State Inspection
- `reg [name...]` - Show registers
- `mem <addr> [count]` - Show memory
- `hexview <addr> [count]` - Hex dump
- `stack [count]` - Show stack
- `insn` - Show next instruction

### State Management
- `snapshot <name>` - Save state
- `restore <name>` - Load state

### Utility
- `clear` - Clear screen
- `help [cmd]` - Show help
- `about` - Version info
- `quit` - Exit

## Tips and Tricks

### 1. Repeat Last Command

Press **Enter** without typing to repeat the last command:

```bash
creator> step
creator> <Enter>    # steps again
creator> <Enter>    # steps again
```

### 2. Multiple Arguments

Many commands accept multiple arguments:

```bash
creator> reg t0 t1 t2 t3
creator> step 5
creator> mem 0x10010000 16
```

### 3. Use Aliases

Define shortcuts in config:

```yaml
aliases:
  s: step
  r: run
  b: break
  rs: reg sp
```

Then use:
```bash
creator> s      # same as step
creator> b main # same as break main
```

### 4. Combine Commands

Use snapshots for experimentation:

```bash
creator> snapshot try1
creator> run
# Didn't work, try different approach
creator> restore try1
creator> break different_location
creator> run
```

### 5. Stack Navigation

View calling context:

```bash
creator> stack 10
# Shows return addresses and stack frames
# Navigate to specific frame
creator> mem $sp-32 8
```

## Error Messages

### Common Errors

**Invalid address**:
```
Error: Address 0x12345678 is not accessible
```
Check memory layout for your architecture.

**Unknown register**:
```
Error: Unknown register 'x99'
```
Use valid register names (see architecture guide).

**No breakpoints**:
```
No breakpoints set
```
Use `break <addr>` to set breakpoints first.

**Snapshot not found**:
```
Error: Snapshot 'xyz' not found
```
List snapshots with `snapshot` (no args).

## Configuration

Customize interactive mode in `~/.config/creator/config.yml`:

```yaml
# Command aliases
aliases:
  s: step
  r: run
  
# Keyboard shortcuts
shortcuts:
  step: "s"
  run: "r"
  
# Display settings
registers_per_line: 4
memory_bytes_per_line: 16
stack_entries: 8

# History
max_undo_states: 100
save_history: true
history_file: "~/.creator_history"
```

## Accessible Mode

For screen readers:

```bash
creator6 --accessible program.s
```

Features:
- Screen reader friendly output
- Structured state descriptions
- Verbose register/memory dumps
- No visual formatting

## Next Steps

- See [Commands Reference](commands-reference.md) for complete command details
- Read [Configuration](configuration.md) for customization options
- Check [Examples](examples.md) for practical workflows
- Review [Troubleshooting](troubleshooting.md) for common issues