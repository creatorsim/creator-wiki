# Assemblers

CREATOR has support for a modular assembler system, allowing users to choose from multiple assemblers for different architectures. This chapter provides an overview of the available assemblers, their features, and how to use them within CREATOR.

## CREATOR Assembler

The default assembler built into CREATOR. This assembler is designed to work seamlessly with the defined architectures. It is currently used for RISC-V, and MIPS. Any custom architectures defined by users will, by default, also use this assembler.

### Limitations
Architectures with complex instruction sets may not be fully supported. One example is the Z80 architecture, which requires a more specialized assembler due to having multiple opcodes for the same instruction based on context.
Instruction sets with multiple optional fields are not supported.

## RASM

[RASM](https://github.com/EdouardBERGE/rasm) is a Free and open-source Z80 assembler. When choosing the Z80 architecture in CREATOR, RASM is used as the default assembler.

## Choosing an Assembler

By default, the assembler is chosen based on the architecture selected. From a user perspective, there is no need to choose. The appropriate assembler will always be used automatically.

Developers creating custom architectures can specify which assembler to use in the architecture definition file. For more information, see the [Assemblers](../development/assemblers.md) chapter.
