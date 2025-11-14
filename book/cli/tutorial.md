# Tutorial Mode

CREATOR includes an interactive tutorial mode for learning RISC-V assembly programming. This guided learning experience is perfect for beginners and those new to RISC-V.

## Starting the Tutorial

Launch tutorial mode with:
```bash
creator --tutorial --architecture architecture/riscv32.yml
```

Or using the `-T` shorthand:
```bash
creator -T -a architecture/riscv32.yml
```

## Overview

The tutorial provides:
- **Step-by-step lessons**: Progressive learning path
- **Hands-on exercises**: Practice with real code
- **Interactive prompts**: Guided command execution
- **Built-in test program**: Pre-loaded demonstration code
- **Progress tracking**: See how far you've come
- **Instant feedback**: Learn from mistakes immediately

## Tutorial Structure

### Lesson Flow

1. **Introduction**: Welcome and overview
2. **Architecture Basics**: Understanding RISC-V
3. **Program Walkthrough**: Examining sample code
4. **Instruction Execution**: Running your first instruction
5. **Register Inspection**: Viewing and understanding registers
6. **Memory Exploration**: Examining data in memory
7. **Breakpoints**: Setting and using breakpoints
8. **Program Flow**: Running to completion
9. **Advanced Features**: Additional commands
10. **Completion**: Summary and next steps

### Interactive Format

Each lesson includes:
- **Explanation**: Concept description with examples
- **Demonstration**: Showing the feature in action
- **Practice**: You try the command yourself
- **Feedback**: Confirmation or guidance
- **Progression**: Move to next lesson when ready

## Sample Tutorial Program

The tutorial uses this RISC-V program:

```assembly
.data
    .align 2
   myword:      .word   0x12345678
   stringz:     .string "This is a string"
   
.text
    main:
        addi t1, zero, 0x123
        
        # print word value 
        la a0, myword
        lw a1, 0(a0)
        li a7, 1
        ecall

        # print string value
        la a0, stringz
        li a7, 4
        ecall

        # return
        jr ra
```

This program demonstrates:
- Data declarations
- Immediate operations
- Label usage
- Memory access
- System calls
- Function return

## Tutorial Features

### Progressive Learning

Lessons build on each other:
1. Start with simple concepts
2. Gradually introduce complexity
3. Reinforce previous lessons
4. Practical application focus

### Guided Commands

The tutorial asks you to execute specific commands:

**Example Interaction**:
```
─────────────────────────────────────────────────────
Examining Registers

Registers are small, fast storage locations in the CPU.
RISC-V has 32 general-purpose registers (x0-x31).

To view the integer registers, use the 'reg' command.

Try it now by typing 'reg int_registers':

CREATOR> reg int_registers

int_registers:
zero(x0): 0x00000000  ra(x1): 0x00000000  ...

The first column shows the register name, and the second
column shows its value...

Press Enter to continue...
```

### Visual Presentation

Features (in normal mode):
- **Progress bars**: Show tutorial completion
- **Boxed text**: Important information highlighted
- **Colored output**: Key concepts emphasized
- **Dividers**: Clear section separation
- **Wrapped text**: Readable paragraph formatting

### Flexible Pacing

- **Self-paced**: Move at your own speed
- **Skip or repeat**: No penalty for exploration
- **Try commands freely**: Experiment beyond requirements
- **Exit anytime**: Save progress (implicit through your learning)

## Command Execution

### Expected Commands

Tutorial expects specific commands at certain points:

**Exact Match**:
```
Try typing 'list':
CREATOR> list      ✓ Correct - advances tutorial
CREATOR> l         ✗ Different - tutorial remains on this step
```

**Prefix Match** (for some commands):
```
Try 'mem 0x200000':
CREATOR> mem 0x200000      ✓ Correct
CREATOR> mem 0x200000 16   ✓ Also correct (additional args OK)
```

### Freedom to Explore

You can execute ANY command:
```
Expected command: step

CREATOR> reg t1          # Different command - executes normally
CREATOR> mem 0x10000000  # Explore freely
CREATOR> step            # When ready, enter expected command
```

Tutorial only advances when you execute the expected command, but you're free to explore.

### Help During Tutorial

**Get hints**:
```
CREATOR> help
# Shows all available commands

Expected command reminder shown in tutorial text
```

**Exit tutorial**:
```
CREATOR> quit
# Exits tutorial and simulator
```

## Lessons in Detail

### Lesson 1: Welcome
- Introduction to CREATOR
- Tutorial overview
- What you'll learn

### Lesson 2: RISC-V Architecture
- Register file (x0-x31)
- Register naming conventions
- Memory organization
- Instruction formats

### Lesson 3: The Program
- Source code walkthrough
- Data section (`.data`)
- Text section (`.text`)
- Labels and directives

### Lesson 4: Listing Instructions
Command: `list`
- View loaded instructions
- Understand addressing
- Identify labels
- Read machine code

### Lesson 5: Examining Registers
Command: `reg int_registers`
- Register banks
- Initial values
- Special registers (zero, ra, sp)

### Lesson 6: First Instruction
Command: `step`
- Execute single instruction
- Observe output format
- Understand address/code/assembly display

### Lesson 7: Register Changes
Command: `reg t1`
- View specific register
- See updated value
- Understand immediate operations

### Lesson 8: Memory Inspection
Command: `mem 0x200000`
- Data segment address
- Memory contents
- Data declaration correlation

### Lesson 9: Multiple Steps
Command: `run 5`
- Execute several instructions
- Batch execution
- Observe program progression

### Lesson 10: Breakpoints
Command: `break 24`
- Set breakpoint by address
- Breakpoint indicators
- Debugging strategy

### Lesson 11: Viewing Breakpoints
Command: `list`
- See breakpoint markers
- Understand breakpoint display
- Plan debugging approach

### Lesson 12: Reset
Command: `reset`
- Return to start
- Reset state
- Prepare for run

### Lesson 13: Run to Breakpoint
Command: `run`
- Execute until breakpoint
- Breakpoint hit message
- Examine state at breakpoint

### Lesson 14: Continue
Command: `continue`
- Resume execution
- Run to completion
- Program output

### Lesson 15: Help System
Command: `help`
- Available commands
- Command descriptions
- Further learning resources

### Lesson 16: Completion
- Congratulations message
- Summary of learned skills
- Transition to normal mode

## Post-Tutorial

After completing the tutorial:

**You know how to**:
- Load and run assembly programs
- Step through code instruction-by-instruction
- Examine registers and memory
- Set and use breakpoints
- Navigate the interface
- Use help resources

**Normal Mode**:
- Tutorial ends
- Full simulator access
- All commands available
- No more guidance
- Explore independently

**Next Steps**:
- Write your own programs
- Try advanced commands (`unstep`, `stack`, `snapshot`)
- Read full [Commands Reference](commands-reference.md)
- Explore [Examples](examples.md)
- Study [Configuration](configuration.md)

## Tips for Tutorial

### Getting the Most Out of It

1. **Read Carefully**: Tutorial explanations provide context
2. **Experiment**: Try related commands beyond what's asked
3. **Take Notes**: Write down important concepts
4. **Repeat if Needed**: Re-run tutorial to reinforce learning
5. **Ask Questions**: Use GitHub Discussions for help

### If You Get Stuck

**Can't remember command**:
- Tutorial text shows expected command
- Type `help` to see all commands
- Re-read the lesson explanation

**Wrong command**:
- Tutorial accepts only expected command to advance
- Other commands execute normally
- Try the expected command when ready

**Want to skip ahead**:
- Tutorial is linear (for pedagogical reasons)
- Can exit and start normal mode anytime
- Quick learners: Complete tutorial quickly

### After the Tutorial

**Practice**:
- Write simple programs
- Test your understanding
- Build complexity gradually

**Explore**:
- Try all commands
- Read documentation
- Experiment with features

**Build**:
- Implement algorithms
- Create utilities
- Solve problems

## Accessibility in Tutorial

### Accessible Mode

Run tutorial in accessible mode:
```bash
creator --tutorial --accessible -a riscv32.yml
```

**Changes**:
- Plain text (no colors)
- No ASCII art or boxes
- Clear, descriptive text
- Screen reader friendly

### Keyboard-Only

Tutorial is fully keyboard accessible:
- No mouse required
- Text-based interface
- Standard readline editing
- Navigation via Enter key

## Technical Details

### Tutorial Implementation

The tutorial system is implemented in `src/cli/tutorial.mts`:
- Self-contained lesson data
- Step management
- Command validation
- Visual formatting utilities

### State Management

- Tutorial tracks current step
- Remembers whether waiting for command
- Pre-loads demonstration program
- Resets simulator between relevant lessons

### Command Processing

Special handling during tutorial:
- All commands execute normally
- Only expected command advances tutorial
- Flexible matching (exact or prefix)
- Clear feedback on progress

## Customizing Tutorial Experience

### Configuration

Tutorial respects some config settings:
- `accessible` mode setting
- Terminal size detection
- Command history

**Does NOT use**:
- Aliases (to avoid confusion)
- Keyboard shortcuts (for clarity)
- Custom state limits

### Future Enhancements

Potential improvements:
- Multiple tutorial tracks (beginner/advanced)
- Other architectures (MIPS, Z80)
- Custom tutorial creation
- Progress saving
- Achievement system

## Next Steps

- Try the tutorial: `creator --tutorial -a riscv32.yml`
- Read [Command-Line Options](command-line-options.md) for startup options
- Study [Commands Reference](commands-reference.md) for all commands
- Explore [Interactive Mode](interactive-mode.md) for daily usage
- See [Examples](examples.md) for practical programs
