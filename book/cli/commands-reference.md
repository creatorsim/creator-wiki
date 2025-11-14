# Commands Reference

Complete reference for all CREATOR CLI commands. Commands are case-insensitive and support [aliases](#command-aliases) defined in your configuration.

## Execution Control

### `step`

Execute one instruction and display the result.

**Syntax**: `step` or just press Enter  
**Alias**: Configured in config file (commonly `s`)

**Output**:
```
0x00400000 (0x00000293) addi t0,zero,0
```

Shows: `<address> (<machine_code>) <assembly>`

**Behavior**:
- Executes the instruction at current PC
- PC advances to next instruction
- Registers and memory update accordingly
- State is saved for `unstep`

**When to Use**:
- Step-by-step debugging
- Observe instruction-level behavior
- Verify each operation

---

### `unstep`

Undo the last executed instruction (reverse step).

**Syntax**: `unstep`  
**Alias**: Configured in config file

**Requirements**:
- `max_states` must be set in config (>0)
- At least one instruction must have been executed

**Behavior**:
- Restores simulator to state before last instruction
- PC moves backward
- Register and memory changes reverted
- Can unstep multiple times up to `max_states` limit

**Example**:
```
CREATOR> step
0x00400000 (0x00000293) addi t0,zero,0

CREATOR> unstep
# PC back at 0x00400000, t0 unchanged
```

**When to Use**:
- Went too far in debugging
- Want to re-examine instruction
- Study before/after states

---

### `run [count]`

Execute multiple instructions continuously.

**Syntax**: 
- `run` - Run until completion or breakpoint
- `run <count>` - Run exactly N instructions

**Alias**: Configured in config file (commonly `r`)

**Examples**:
```
CREATOR> run           # Run to completion
CREATOR> run 100       # Run 100 instructions
CREATOR> run 1000000   # Run 1 million instructions
```

**Stops When**:
- Program completes (execution_index = -2)
- Breakpoint is hit
- Error occurs
- User interrupts (Ctrl+C or `pause` command)

**Output**:
- Each instruction displayed as it executes
- "Program execution completed." when done
- "Breakpoint hit at <address>" when breakpoint reached

**Performance**:
- Executes in chunks for responsive UI
- Can pause mid-execution

---

### `silent [count]`

Run instructions without displaying output.

**Syntax**: 
- `silent` - Run silently until completion
- `silent <count>` - Run N instructions silently

**Use Cases**:
- Fast execution without visual clutter
- Performance testing
- Running initialization code
- Getting to a specific point quickly

**Example**:
```
CREATOR> silent 1000   # Skip first 1000 instructions
CREATOR> list          # Now see where we are
```

---

### `continue`

Resume execution from paused or stepped state.

**Syntax**: `continue`  
**Alias**: Configured in config file (commonly `c`)

**Behavior**:
- If paused mid-run: Resumes from current point
- If at breakpoint: Steps once then continues running
- Otherwise: Steps once then runs

**Example Flow**:
```
CREATOR> run
# ... executes many instructions ...
CREATOR> pause
Execution paused.

CREATOR> continue
# Resumes execution
```

---

### `pause`

Toggle pause state during execution.

**Syntax**: `pause`

**Behavior**:
- If running: Pauses execution at next instruction
- If paused: Resumes execution

**Use Cases**:
- Stop runaway execution
- Examine state mid-run
- Interactive control during long runs

---

### `nur`

uN-Run (reverse run) - Step backwards until breakpoint or program start.

**Syntax**: `nur`

**Requirements**:
- `max_states` must be set in config
- Sufficient state history available

**Behavior**:
- Repeatedly unsteps
- Stops at breakpoints (going backwards)
- Stops when no more history available

**Example**:
```
CREATOR> run          # Execute past several breakpoints
CREATOR> nur          # Go back to last breakpoint
```

**When to Use**:
- Overshot target during debugging
- Want to re-examine earlier state
- Find when something changed

---

### `until <address|label>`

Run until specified address is reached.

**Syntax**: `until <address>` or `until <label>`

**Examples**:
```
CREATOR> until 0x400010    # Run until address
CREATOR> until loop_end    # Run until label
```

**Behavior**:
1. Sets temporary breakpoint at address/label
2. Runs program
3. When breakpoint hit, removes it

**Equivalent To**:
```
break <address>
run
break <address>  # Toggle off
```

---

### `reset`

Reset simulator to initial state.

**Syntax**: `reset`

**What Gets Reset**:
- Program counter to entry point
- All registers to initial values
- Memory to loaded state (ROM + initialized data)
- Execution history cleared
- All temporary state cleared

**What Persists**:
- Loaded program
- Architecture definition
- Breakpoints (optional: can clear manually)

**Example**:
```
CREATOR> run
Program execution completed.

CREATOR> reset
Program reset.

CREATOR> list
# Back at start
```

---

## Breakpoint Management

### `break [address|label]`

Set, remove, or list breakpoints.

**Syntax**:
- `break` - List all breakpoints
- `break <address>` - Toggle breakpoint at hex address
- `break <label>` - Toggle breakpoint at label
- `break <hex_number>` - Toggle at hex (without 0x)

**Alias**: Commonly `b`

**Examples**:
```
CREATOR> break                    # List breakpoints
CREATOR> break 0x400008           # Toggle at address
CREATOR> break main               # Toggle at label
CREATOR> break 10                 # Toggle at address 0x10
```

**Output**:
```
Breakpoint set at 0x00400008 (loop): addi t0,t0,1
# or
Breakpoint removed at 0x00400008 (loop): addi t0,t0,1
```

**Label Resolution**:
- If input matches a label, uses label's address
- Otherwise, treats as hex address (with or without 0x prefix)

**Visual Indicators**:
- Red `●` in `list` output
- Red highlighting of instruction line
- Mentioned in breakpoint list

---

## Inspection Commands

### `list [limit]`

Display loaded instructions with addresses, labels, and code.

**Syntax**:
- `list` - Show all instructions
- `list <n>` - Show first N instructions
- `list --limit <n>` - Show first N instructions (explicit)

**Output Format (Assembly Mode)**:
```
    B | Address | Label      | Loaded Instruction      | User Instruction
   ---|---------|------------|-------------------------|------------------------
 ➤  ● | 0x400000| main       | addi t0,zero,0x123      | li t0, 0x123
      | 0x400004|            | lui a0,0x10000          | la a0, myword
```

**Output Format (Binary Mode)**:
```
    B | Address | Label      | Decoded Instruction     | Machine Code (hex)
   ---|---------|------------|-------------------------|------------------------
 ➤    | 0x400000| main       | addi t0,zero,0x123      | 0x12300293
```

**Columns**:
- **B**: Breakpoint indicator (`●` if set)
- **➤**: Current PC indicator (green)
- **Address**: Instruction memory address
- **Label**: Label at this address (if any)
- **Loaded/Decoded**: The actual instruction in memory
- **User/Machine**: Original source or machine code hex

**Colors** (in normal mode):
- Green: Current instruction (at PC)
- Yellow: Previously executed instruction
- Red: Instructions with breakpoints

---

### `insn`

Display the current instruction at PC.

**Syntax**: `insn`

**Output**:
```
0x00400000 (0x00000293) addi t0,zero,0x123
```

Shows the instruction about to be executed.

---

### `reg <type|name> [format]`

Display register values.

**Syntax**:
- `reg list` - List available register types
- `reg <type>` - Show all registers of type
- `reg <type> <format>` - Show registers in specific format
- `reg <name>` - Show specific register
- `reg <name> <format>` - Show specific register in format

**Register Types** (architecture-dependent):
- `int_registers` - Integer/general-purpose registers
- `fp_registers` - Floating-point registers
- `csr_registers` - Control and Status Registers (RISC-V)
- Other architecture-specific types

**Formats**:
- `raw` - Hexadecimal (default)
- `decimal` or `dec` - Decimal (signed)
- `number` - Same as decimal

**Examples**:
```
CREATOR> reg list
Register types:
  int_registers
  fp_registers

CREATOR> reg int_registers
int_registers:
zero(x0): 0x00000000  ra(x1): 0x00000000  sp(x2): 0x7FFFFFFC  gp(x3): 0x10008000
t0(x5):   0x00000123  t1(x6): 0x00000000  t2(x7): 0x00000000  fp(x8): 0x00000000
...

CREATOR> reg int_registers dec
int_registers:
zero(x0): 0           ra(x1): 0           sp(x2): 2147483644  gp(x3): 268468224
t0(x5):   291         t1(x6): 0           t2(x7): 0           fp(x8): 0
...

CREATOR> reg t0
t0: 0x00000123 | 291

CREATOR> reg pc
PC: 0x00400000 | 4194304
```

**Register Aliases**:
- RISC-V: `x0`/`zero`, `x1`/`ra`, `x5`/`t0`, `x10`/`a0`, etc.
- Shows all aliases in parentheses

---

### `mem <address> [count]`

Display memory contents with hints and annotations.

**Syntax**:
- `mem <address>` - Show one word at address
- `mem <address> <count>` - Show count bytes starting at address

**Address Format**:
- Hexadecimal with 0x prefix: `0x10000000`
- Hexadecimal without prefix: `10000000`

**Examples**:
```
CREATOR> mem 0x10000000
0x10000000: 0x12345678

CREATOR> mem 0x10000000 16
0x10000000: 0x12345678  // myvar:int (32b)
0x10000004: 0x00000064  // count:int (32b)
0x10000008: 0x00000001
0x1000000C: 0x00000002
```

**Features**:
- **Hints**: Shows variable names and types if available
- **Colors** (normal mode): Different colors for different hints
- **Word-aligned**: Displays in architecture word sizes
- **Annotations**: Comments show semantic information

**Hints Format**:
```
0x10000000: 0x12345678  // tag:type (size in bits) @+offset
```

**Multiple Hints Per Word**:
```
0x10000000: 0x12FF00AB  // a:char (8b) @+0, b:char (8b) @+1, c:short (16b) @+2
```
Each hint is color-coded differently.

---

### `hexview <address> [count] [bytes_per_line]`

Display memory in hexdump format with ASCII representation.

**Syntax**:
- `hexview <address>` - Default: 16 bytes, 16 per line
- `hexview <address> <count>` - Specified bytes, 16 per line
- `hexview <address> <count> <bytes_per_line>` - Custom layout

**Examples**:
```
CREATOR> hexview 0x10000000
0x10000000: 78 56 34 12 48 65 6C 6C  6F 20 57 6F 72 6C 64 00  |xV4.Hello World.|

CREATOR> hexview 0x10000000 32 8
0x10000000: 78 56 34 12 48 65 6C 6C  |xV4.Hell|
0x10000008: 6F 20 57 6F 72 6C 64 00  |o World.|
0x10000010: 00 00 00 00 00 00 00 00  |........|
0x10000018: 00 00 00 00 00 00 00 00  |........|
```

**Format**:
```
<address>: <hex bytes...>  |<ASCII>|
```

**ASCII Column**:
- Printable characters shown as-is
- Non-printable shown as `.`
- Useful for finding strings in memory

---

### `stack [max_bytes]`

Display call stack information and stack memory contents.

**Syntax**:
- `stack` - Show stack with default limit (256 bytes)
- `stack <max_bytes>` - Show up to specified bytes

**Output Sections**:

1. **Call Stack Hierarchy**:
```
Call Stack:
  ► main (0x7FFFFFFC - 0x7FFFFFF0, 12 bytes)
    • calculate (0x7FFFFFF0 - 0x7FFFFFE4, 12 bytes)
      • helper (0x7FFFFFE4 - 0x7FFFFFDC, 8 bytes)
```

2. **Current Frame Details**:
```
Current Frame Details:
Function: helper
Frame: 0x7FFFFFE4 - 0x7FFFFFDC
Size: 8 bytes
Caller: calculate
Caller frame: 0x7FFFFFF0 - 0x7FFFFFE4
```

3. **Stack Memory Contents**:
```
Stack Memory Contents:
0x7FFFFFDC: 0x00000000  ← SP, helper frame start
0x7FFFFFE0: 0x00400020  // return address
0x7FFFFFE4: 0x00000005  ← helper frame end, calculate frame start
0x7FFFFFE8: 0x00400010  // saved ra
0x7FFFFFEC: 0x00000003  // local_var
0x7FFFFFF0: 0x00400000  ← calculate frame end, main frame start
```

**Frame Colors** (normal mode):
- Each stack frame shown in different color
- Current (top) frame in green
- Older frames in cyan, magenta, blue, yellow

**Annotations**:
- Frame boundaries marked
- Stack pointer (SP) indicated
- Variable names from hints
- Return addresses identified

---

## State Management

### `snapshot [filename]`

Save complete simulator state to a JSON file.

**Syntax**:
- `snapshot` - Auto-generate timestamped filename
- `snapshot <filename>` - Save to specific file

**Auto-Generated Format**:
```
snapshot-2025-01-15T10-30-45-123Z.json
```

**Saved State Includes**:
- All register values
- All memory contents
- Program counter
- Execution state
- Previous PC (for display)
- Stack tracker state
- Custom extra data

**Example**:
```
CREATOR> run 100
CREATOR> snapshot debug-state.json
Snapshot saved to debug-state.json

CREATOR> run
Program execution completed.

CREATOR> restore debug-state.json
State restored from debug-state.json

CREATOR> list
# Back at instruction 100
```

**Use Cases**:
- Save progress before risky operations
- Create checkpoints during debugging
- Share reproducer for bugs
- Test different execution paths from same state

---

### `restore <filename>`

Restore simulator state from a snapshot file.

**Syntax**: `restore <filename>`

**Requirements**:
- File must be valid snapshot JSON
- Architecture must match

**Behavior**:
1. Resets current state
2. Loads snapshot from file
3. Restores all registers, memory, PC
4. Restores previous PC tracking
5. Ready to continue from restored point

**Example**:
```
CREATOR> restore checkpoint1.json
State restored from checkpoint1.json

CREATOR> reg pc
PC: 0x00400064 | 4194404

CREATOR> step
# Continue from restored state
```

---

## Utility Commands

### `clear`

Clear the terminal screen.

**Syntax**: `clear`

**Effect**:
- Clears all terminal output
- Prompt reappears at top
- State unchanged

**When to Use**:
- Declutter after lots of output
- Fresh view for new debugging session
- Before taking screenshot

---

### `help`

Display all available commands with brief descriptions.

**Syntax**: `help`

**Output**:
- List of all commands
- Brief description of each
- Keyboard shortcuts (if enabled)
- Aliases (from configuration)

See the actual output in [utils.mts](../../src/cli/utils.mts).

---

### `about`

Display information about CREATOR.

**Syntax**: `about`

**Shows**:
- CREATOR version
- CLI version
- Runtime version (Deno/Node)
- Platform information
- Copyright and credits
- ASCII art logo (unless in accessible mode)

**Example Output** (normal mode):
```

 ██████╗██████╗ ███████╗ █████╗ ████████╗ ██████╗ ██████╗ 
██╔════╝██╔══██╗██╔════╝██╔══██╗╚══██╔══╝██╔═══██╗██╔══██╗
██║     ██████╔╝█████╗  ███████║   ██║   ██║   ██║██████╔╝
██║     ██╔══██╗██╔══╝  ██╔══██║   ██║   ██║   ██║██╔══██╗
╚██████╗██║  ██║███████╗██║  ██║   ██║   ╚██████╔╝██║  ██║
 ╚═════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝
    didaCtic and geneRic assEmbly progrAmming simulaTOR


╔════════════════════════════════════════════════════════════╗
║ CREATOR Information                                        ║
╠════════════════════════════════════════════════════════════╣
║ ⚙️  CREATOR CLI Version: 0.1.0                             ║
║ 🚀 CREATOR Core Version: 6.0.0                             ║
║ 🔧 Deno Version: 2.5.2                                     ║
╠════════════════════════════════════════════════════════════╣
║ System Information                                         ║
╠════════════════════════════════════════════════════════════╣
║ 💻 Platform: darwin                                        ║
║ 🧠 Architecture: arm64                                     ║
╠════════════════════════════════════════════════════════════╣
║ About                                                      ║
╠════════════════════════════════════════════════════════════╣
║ CREATOR is a didactic and generic assembly                 ║
║ simulator built by the ARCOS group at the                  ║
║ Carlos III de Madrid University (UC3M)                     ║
║ © Copyright (C) 2025 CREATOR Team                          ║
╚════════════════════════════════════════════════════════════╝
```

---

### `alias`

List defined command aliases.

**Syntax**: `alias`

**Output**:
```
Current command aliases:
  s   → step
  b   → break
  r   → run
  c   → continue
  reg → reg int_registers

Aliases can be defined in your config file at:
~/.config/creator/config.yml
```

**Note**: Aliases are defined in [Configuration](configuration.md) file, not via command.

---

### `quit`

Exit the simulator.

**Syntax**: `quit` or `exit` or `q`

**Behavior**:
- Exits interactive mode
- Closes the simulator
- No confirmation prompt

**Data Loss Warning**:
- Unsaved state is lost
- Use `snapshot` before quitting to save progress

---

## Command Aliases

Commands support aliasing via configuration file. Common aliases:
- `s` → `step`
- `b` → `break`
- `r` → `run`
- `c` → `continue`
- `u` → `unstep`

See [Configuration](configuration.md) for setting up aliases.

---

## Keyboard Shortcuts

When `keyboard_shortcuts` is enabled in config, single keypresses can execute commands:
- `Ctrl+S` → `step` (example)
- `Ctrl+B` → `break` (example)
- `Ctrl+L` → `clear` (example)

See [Configuration](configuration.md) for setting up shortcuts.

---

## Command-Line Editing

The interactive mode supports standard line editing:
- **Arrow Keys**: Navigate command history and cursor
- **Ctrl+C**: Cancel current input
- **Ctrl+D**: Exit simulator
- **Ctrl+L**: Clear screen (if not bound to command)
- **Tab**: Auto-completion (future feature)

---

## Tips and Tricks

### Efficient Debugging
```
# Quick breakpoint workflow
CREATOR> b main       # Set breakpoint
CREATOR> r            # Run to breakpoint
CREATOR> s            # Step a few times
CREATOR> reg t0       # Check specific register
CREATOR> mem 0x10000000  # Check memory
```

### Reverse Debugging Workflow
```
# Set max_states in config first
CREATOR> r 1000       # Run 1000 instructions
CREATOR> unstep       # Oops, went too far
CREATOR> unstep       # Back one more
CREATOR> s            # Re-execute carefully
```

### State Comparison
```
CREATOR> snapshot before.json
CREATOR> r 100
CREATOR> snapshot after.json
# Compare files externally or restore before.json to retry
```

### Finding Bugs
```
CREATOR> until suspect_function
CREATOR> list
CREATOR> break suspicious_line
CREATOR> r
CREATOR> reg int_registers
CREATOR> mem <address_from_register>
CREATOR> stack
```

---

## Next Steps

- Understand [Interactive Mode](interactive-mode.md) workflow
- Customize with [Configuration](configuration.md)
- Try [Tutorial Mode](tutorial.md) for guided practice
- See [Examples](examples.md) for real usage patterns