# Validating program execution
The [CREATOR CLI](../cli/README.md) allows to validate the execution of a program.

For this, you first need to define a YAML validator file. This file must contain the expected final state of the program, as well as any additional options you want to set for the validation. 


## Validator File

> [!NOTE]
> We provide a [JSON schema](https://json-schema.org/) for the validator file at https://creatorsim.github.io/creator/schema/validator-file.json.


The validator file is a YAML file that can contain the following fields:
- `maxCycles`: The maximum number of cycles to run the program for validation. If the program exceeds this number of cycles, it will be considered invalid.
- `floatThreshold`: The maximum allowed error for floating point register comparisons.
- `sentinel`: A boolean indicating whether to enable sentinel checking (calling convention errors).
- `state`: An object defining the expected final state of the program, which can contain:
  - `registers`: A mapping of integer register names to their expected values.
  - `floatRegisters`: A mapping of floating point register names to their expected values.
  - `memory`: A mapping of memory addresses (as strings) to their expected byte values.
  - `display`: The expected contents of the display buffer as a string.

The validator will return, following POSIX conventions, `0` if its valid and `1` if it's not, writting the error message to the standard error stream (stderr).



### Example
Quick example of a validator file:
```yaml
maxCycles: 100
floatThreshold: 10e-10
sentinel: true
state:
  floatRegisters:
    f0: 0x40866666
  registers:
    sp: 0x0FFFFFFC
    a1: 0x0000005A
  memory:
    "0x200000": 0x45
  display: "-144"
```

### Using the Validator
To use the validator file with CREATOR CLI, use the `--validate` option followed by the path to the validator YAML file. For example:
```bash
creator-cli -a RV32IMFD.yml -s assembly.s --validate config.yml
```