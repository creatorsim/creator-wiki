# CREATOR - didaCtic and geneRic assEmbly progrAmming simulaTOR

Welcome to the CREATOR documentation! CREATOR is a comprehensive, open-source assembly programming simulator designed for education and development across multiple computer architectures.

### 👨‍🎓 **Students, Start Here!**
New to CREATOR? Get up and running quickly:
1. **[Try it now in your browser](https://creatorsim.github.io/creator/)** - No installation needed!
2. Follow the **[Quick Start Guide](getting-started/quick-start.md)** to write your first program
3. Learn about the **[Web Interface](web/user-interface.md)** to navigate the simulator
4. Try the **[Interactive Tutorial Mode](cli/tutorial.md)** for step-by-step learning

### 👨‍🏫 **Teachers, Start Here!**
Setting up CREATOR for your course:
1. **[Installation Guide](getting-started/installation.md)** - Deploy for your classroom
2. **[CLI User Guide](cli/README.md)** - Command-line tools for assignments and grading
3. **[Architecture Guide](architecture/overview.md)** - Understand supported architectures (RISC-V, MIPS, Z80)
4. **[Custom Architectures](development/custom-architectures.md)** - Create your own instruction sets
5. **[Examples](cli/examples.md)** - Sample programs for teaching


---

## What is CREATOR?

CREATOR is a didactic and generic assembly programming simulator that provides a user-friendly environment for writing, assembling, and simulating assembly code. It supports various architectures, including RISC-V, MIPS, and Z80, and allows users to define custom architectures. It is available as both a web-based application and a command-line interface (CLI) tool.

## Key Features
- **Device System**: Console I/O, OS drivers, and custom device support
- **Interrupt Handling**: Software, timer, and external interrupt simulation
- **Stack Tracking**: Automatic call stack tracking and visualization
- **Memory Hints**: Tag memory locations with type information for easier debugging
- **Privileged Instructions**: Kernel/user mode separation with privilege levels