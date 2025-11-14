# Configuration

CREATOR supports extensive configuration through YAML files, allowing you to customize behavior, create command aliases, and define keyboard shortcuts.

## Configuration File Location

**Default location**: `~/.config/creator/config.yml`

**Custom location**: Use `--config` option:
```bash
creator6 --config ~/my-creator-config.yml program.s
```

If the configuration file doesn't exist, CREATOR creates it with default settings automatically.

## Configuration Structure

Source: `src/cli/creator6.mts` - `ConfigType` interface and `DEFAULT_CONFIG`

The configuration file has three main sections:

```yaml
settings:
  # General behavior settings
  
aliases:
  # Command aliases (shortcuts for long commands)
  
shortcuts:
  # Single-key keyboard shortcuts
```

## Settings Section

### Available Settings

**max_states** (number):
- Maximum number of states saved for `unstep`/`nur` (reverse execution)
- `0`: Reverse execution disabled
- `-1`: Unlimited history (uses more memory)
- Default: `100`

**accessible** (boolean):
- Enable accessibility mode for screen readers
- Provides verbose, structured output
- Disables visual formatting
- Default: `false`

**keyboard_shortcuts** (boolean):
- Enable single-key shortcuts in interactive mode
- Default: `true`

**auto_list_after_shortcuts** (boolean):
- Automatically run `list` command after keyboard shortcuts
- Helps see context after stepping
- Default: `true`

### Example Settings

```yaml
settings:
  max_states: 50          # Keep last 50 states
  accessible: false       # Standard visual mode
  keyboard_shortcuts: true # Enable shortcuts
  auto_list_after_shortcuts: false # Manual listing
```

## Aliases Section

Create command shortcuts to save typing.

### Syntax

```yaml
aliases:
  alias_name: full_command
```

### Built-in Examples

```yaml
aliases:
  # Short names for common commands
  s: step
  r: run
  b: break
  
  # Register groups
  regs: reg int_registers
  fregs: reg fp_registers
  
  # Common inspections
  pc: reg pc
  sp: reg sp
  ra: reg ra
  
  # Memory dumps
  text: hexview 0x00400000 64 16
  data: hexview 0x10010000 64 16
  stack: mem sp 16
```

### Custom Aliases

You can create aliases for any command with arguments:

```yaml
aliases:
  # Debugging helpers
  showargs: reg a0 a1 a2 a3
  showtemps: reg t0 t1 t2 t3 t4 t5 t6
  
  # Quick breakpoints
  bmain: break main
  bloop: break main_loop
  
  # Memory views with hints
  datahint: mem 0x10010000 8 --hints
  
  # Step multiple times
  step5: step 5
  step10: step 10
  
  # Combined operations
  runshow: run; reg; stack
```

### Using Aliases

Once defined, use them like regular commands:

```bash
creator> s          # Same as 'step'
creator> showargs   # Same as 'reg a0 a1 a2 a3'
creator> datahint   # Same as 'mem 0x10010000 8 --hints'
```

## Shortcuts Section

Define single-key shortcuts for interactive mode.

### Syntax

```yaml
shortcuts:
  command: "key"
```

The key must be a single character (in quotes).

### Default Shortcuts

```yaml
shortcuts:
  step: " "      # Spacebar
  run: "r"
  break: "b"
  snapshot: "s"
  unstep: "u"
  quit: "q"
  help: "h"
  list: "l"
```

### Custom Shortcuts

```yaml
shortcuts:
  # Execution control
  step: " "         # Spacebar
  run: "r"
  continue: "c"
  reset: "x"
  
  # State inspection  
  reg: "g"          # 'g' for reGisters
  mem: "m"
  stack: "k"
  
  # Debugging
  break: "b"
  list: "l"
  unstep: "u"
  snapshot: "s"
  restore: "o"      # 'o' for lOad
  
  # Utility
  help: "h"
  clear: "z"
  quit: "q"
```

### Shortcut Behavior

- Press the key once to execute the command
- If `auto_list_after_shortcuts` is enabled, instructions list automatically
- Shortcuts work only in interactive mode
- Disable with `keyboard_shortcuts: false`

## Complete Example Configuration

```yaml
# ~/.config/creator/config.yml

settings:
  max_states: 50
  accessible: false
  keyboard_shortcuts: true
  auto_list_after_shortcuts: true

aliases:
  # Basic command shortcuts
  s: step
  r: run
  b: break
  c: continue
  
  # Register shortcuts
  pc: reg pc
  sp: reg sp
  ra: reg ra
  args: reg a0 a1 a2 a3 a4 a5 a6 a7
  temps: reg t0 t1 t2 t3 t4 t5 t6
  saved: reg s0 s1 s2 s3 s4 s5 s6 s7 s8 s9 s10 s11
  
  # Memory shortcuts
  text: hexview 0x00400000 64 16
  data: hexview 0x10010000 64 16
  stackview: mem sp 16
  stackhex: hexview sp 64 16
  
  # Debugging shortcuts
  here: insn
  where: reg pc
  showstate: reg; stack
  
  # Quick stepping
  s5: step 5
  s10: step 10
  u5: unstep 5

shortcuts:
  step: " "
  run: "r"
  break: "b"
  snapshot: "s"
  unstep: "u"
  quit: "q"
  help: "h"
  list: "l"
  clear: "z"
```

## Configuration Loading

CREATOR loads configuration in this order:

1. **Default config**: Built-in settings
2. **User config**: `~/.config/creator/config.yml` (if exists)
3. **Custom config**: `--config` option (if specified)
4. **Command-line options**: Override config file settings

### Precedence Example

```yaml
# config.yml
settings:
  accessible: false
  max_states: 50
```

```bash
# Command line overrides accessible setting
creator6 --accessible program.s
# Result: accessible=true, max_states=50
```

## Validation and Errors

CREATOR validates configuration on load:

### Invalid YAML Syntax

```
Error loading configuration: bad indentation of a mapping entry
```
Fix YAML formatting (use 2-space indentation).

### Unknown Settings

Unknown settings are ignored with a warning:

```
Warning: Unknown setting 'unknow_option' ignored
```

### Invalid Values

```
Error: max_states must be a number
```
Check value types match expected format.
