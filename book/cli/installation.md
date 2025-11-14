# Installation

## Installing CREATOR CLI
### Installation via precompiled Binary
The easiest way to get started with the CLI version is to use the precompiled binaries. They include everything you need to run CREATOR without additional dependencies.
Download the latest precompiled binary for your OS from the [releases page](https://github.com/creatorsim/creator/releases)
and place it in a directory included in your system's PATH.



### Installation from Source
Alternatively, you can run the CREATOR CLI directly using Deno. Node.js is NOT supported due to webassembly limitations.

```bash
# Install Deno if you haven't already (macOS/Linux)
curl -fsSL https://deno.land/x/install/install.sh | sh

# Or using Homebrew (macOS)
brew install deno

# Clone the CREATOR repository
git clone https://github.com/creatorsim/creator.git
cd creator

# The CLI is ready to use directly with Deno
deno run --allow-all src/cli/creator6.mts --help
```

### Creating an Alias (Recommended)

For easier access, create an alias or script:

#### macOS/Linux (bash/zsh)

Add to your `~/.bashrc`, `~/.zshrc`, or `~/.bash_profile`:

```bash
# For Deno
alias creator='deno run --allow-all /path/to/creator/src/cli/creator6.mts'
```

Then reload your shell:
```bash
source ~/.bashrc  # or ~/.zshrc
```

### Verifying Installation

Test your installation:

```bash
creator --help
```

You should see the CREATOR help message with available options.

## Getting Architecture Files

CREATOR requires architecture definition files to simulate different processors. These are YAML files that define:
- Instruction sets
- Register banks
- Memory layout
- Assembler directives

### Default Architectures

The repository includes several architecture files in the `architecture/` directory:

- `RV32IMFD.yml` - RISC-V 32-bit
- `RV64IMFD.yml` - RISC-V 64-bit
- `RV32IMFD-Interrupts.yml` - RISC-V 32-bit with interrupt support
- `MIPS32.yml` - MIPS 32-bit
- `Z80.yml` - Z80 8-bit processor (with interrupts)

### Location

You can reference them in commands using absolute or relative paths:

```bash
creator --architecture architecture/riscv32.yml --assembly myprogram.s
```

## Configuration

The CLI version supports user configuration via a YAML file located at:

- **macOS/Linux**: `~/.config/creator/config.yml`
- **Windows**: `%USERPROFILE%\.config\creator\config.yml`

This file is automatically created with defaults on first run. See the [Configuration](../cli/configuration.md) section for details.

## Updating

If you installed via precompiled binaries, download the latest version from the [releases page](https://github.com/creatorsim/creator/releases).

To update the CLI version, pull the latest changes from the repository and reinstall dependencies:

```bash
git pull origin main
bun install
```

## Next Steps

- Continue to [Quick Start](quick-start.md) to write your first program
- Explore the [CLI User Guide](../cli/README.md) for command-line usage
- Check out the [Web Version](../web/overview.md) documentation for browser features
