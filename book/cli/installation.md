# Installation

## Installing CREATOR CLI
### Installation via precompiled Binary
The easiest way to get started with the CLI version is to use the precompiled binaries. They include everything you need to run CREATOR without additional dependencies.
Download the latest precompiled binary for your OS from the [releases page](https://github.com/creatorsim/creator/releases)
and place it in a directory included in your system's PATH as `creator-cli`.

You're all set! You can verify the installation by running:

```bash
creator-cli --help
```

To access the CLI from any terminal, ensure the binary is in your PATH or create a symbolic link to it in a directory that is. See [below](#creating-an-alias-recommended) for instructions on creating an alias.


### Building from Source
Alternatively, you can build the CREATOR CLI locally.

#### Prerequisites
- [Git](https://git-scm.com) is required to download the project.
- [Deno](https://deno.land/) must be installed on your system. Follow the instructions on the Deno website to install it.
- [Bun](https://bun.sh/) is the recommended package manager to install dependencies[^1].

#### Steps
1. Clone the CREATOR repository:
    ```bash
    git clone https://github.com/creatorsim-development/creator-beta.git --recurse-submodules --depth=1 && cd creator-beta
    ```

2. Install dependencies:
    ```bash
    bun install
    ```

3. Build the CLI:
    ```bash
    bun build:cli
    ```

    > [!IMPORTANT]
    > This build script **requires** having Bun installed, but if you don't want to, you can use another package manager (e.g., [NPM](https://npmjs.org)) and do:
    > ```
    > npm wasm:cli && npm build:cli:native
    > ```

4. Test your installation:
    ```bash
    creator-cli --help
    ```
    You should see the CREATOR help message with available options.

> [!NOTE]
> You can also just run the CLI without building with Deno:
> ```
> deno run --allow-all src/cli/creator6.mts --help
> ```


## Getting Architecture Files

CREATOR requires architecture definition files to simulate different processors. These files are in YAML format and define the instruction set, registers, memory layout, and other architecture-specific details.

### Default Architectures

The repository includes several architecture files in the `architecture/` directory. The [releases](https://github.com/creatorsim/creator/releases) also include an `architectures.zip` file with the same contents.

Currently, the following architectures are provided:
- `RV32IMFD.yml` - RISC-V 32-bit (with interrupts)
- `RV64IMFD.yml` - RISC-V 64-bit
- `MIPS32.yml` - MIPS 32-bit
- `Z80.yml` - Z80 8-bit processor (with interrupts)


## Configuration
The CLI version supports user configuration via a YAML file located at:

- **macOS/Linux**: `~/.config/creator/config.yml`
- **Windows**: `%USERPROFILE%\.config\creator\config.yml`

This file is automatically created with defaults on first run. See the [Configuration](../cli/configuration.md) section for details.


## Updating
If you installed via precompiled binaries, download the latest version from the [releases page](https://github.com/creatorsim/creator/releases).

To update the CLI version, pull the latest changes from the repository and reinstall dependencies:
```bash
git pull origin main && bun install
```


## Next Steps
- Proceed to the [Command-Line Options](command-line-options.md) to learn how to run programs with different architectures and settings.


[^1:] Other Node package managers like [NPM](https://npmjs.org) or [yarn](https://yarnpkg.com/) _should_ work, but your mileage might vary.