# Teaching Resources
CREATOR is a powerful tool for teaching computer architecture and assembly programming. This document provides resources and guidance for educators looking to integrate CREATOR into their curriculum.

<!--
## Grading Student Exercises
CREATOR supports saving and restoring the complete simulator state via snapshots. This feature is particularly useful for educators who want to create predefined states for students to explore or debug. Currently, it's only available in the [CLI](../cli/README.md). See [Snapshots](snapshots.md) for more information.
-->

CREATOR's [CLI](../cli/README.md) also allows you to validate a program against an expected final state. This includes specifying the values of memory, registers (including floating point registers, within an error threshold), and the display buffer, and whether to error on calling convention errors (sentinel).
See [Validating Program Execution](validator.md) for more information.


## Creating Custom Architectures
CREATOR allows educators to create custom architectures tailored to their curriculum. This can be done by defining new architecture [YAML](https://yaml.org/) files that specify instruction sets, registers, memory layout, and other architecture-specific details. See [Creating Custom Architectures](custom-architectures.md) for more information.
