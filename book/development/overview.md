# Development Guide Overview

This guide covers extending and developing CREATOR, including architecture, core internals, and customization.

## Target Audience

- Developers adding new features to CREATOR
- Researchers implementing custom architectures
- Contributors to the CREATOR project
- Advanced users customizing behavior

## What You'll Learn

### Core Architecture
- How CREATOR is structured
- Execution engine internals
- Decoder and compiler pipeline
- State management system

### Memory System
- Memory implementation
- Address spaces and layouts
- Endianness handling
- Memory hints and annotations

### Devices
- Device abstraction
- Console and I/O devices
- OS driver implementation
- Creating custom devices

### Interrupts
- Interrupt system architecture
- Interrupt types and handlers
- Privilege levels
- Implementation patterns

### Assemblers
- Assembler integration framework
- Creating custom assemblers
- Web vs CLI targets
- Symbol and metadata handling

### Custom Architectures
- YAML architecture definition
- Instruction set specification
- Register bank configuration
- Memory layout design

## Source Code Structure

```
src/
├── cli/                    # Command-line interface
│   ├── creator6.mts       # Main CLI application
│   ├── runner.mts         # Binary runner
│   ├── tutorial.mts       # Tutorial mode
│   └── utils.mts          # CLI utilities
│
├── core/                   # Core simulator
│   ├── core.mjs           # Main core API
│   ├── events.mjs         # Event system
│   │
│   ├── assembler/         # Assembly compilation
│   │   ├── assembler.mjs  # Core assembler
│   │   ├── creatorAssembler/  # Native assembler
│   │   ├── sjasmplus/     # Z80 assembler
│   │   └── rasm/          # Alternative Z80
│   │
│   ├── capi/              # C API and system calls
│   │   ├── initCAPI.mts   # CAPI initialization
│   │   ├── syscall.mts    # System calls
│   │   ├── memory.mts     # Memory operations
│   │   ├── registers.mts  # Register operations
│   │   └── interrupts.mts # Interrupt handling
│   │
│   ├── executor/          # Execution engine
│   │   ├── executor.mjs   # Main executor
│   │   ├── decoder.mjs    # Instruction decoder
│   │   ├── devices.mts    # Device system
│   │   ├── interrupts.mts # Interrupt handling
│   │   ├── IO.mjs         # I/O operations
│   │   ├── stats.mts      # Statistics tracking
│   │   └── timers.mts     # Timer management
│   │
│   ├── memory/            # Memory subsystem
│   │   ├── Memory.mts     # Memory implementation
│   │   └── StackTracker.mts  # Call stack tracking
│   │
│   ├── register/          # Register subsystem
│   │   ├── registerLookup.mjs     # Register lookup
│   │   ├── registerOperations.mjs # Register ops
│   │   └── registerGlowState.mjs  # State tracking
│   │
│   ├── sentinel/          # Security/validation
│   │   └── sentinel.mjs   # Sentinel system
│   │
│   └── utils/             # Utilities
│       ├── architectureProcessor.mjs  # Architecture loading
│       ├── bigint.mjs     # BigInt utilities
│       ├── creator_logger.mjs  # Logging
│       ├── creator_ga.mjs # Analytics
│       ├── float_bigint.mjs  # Float conversion
│       └── schema.mts     # Type definitions
│
├── web/                   # Web interface
│   ├── main.ts            # Web entry point
│   ├── App.vue            # Main Vue component
│   ├── assemblers.ts      # Assembler selection
│   ├── components/        # Vue components
│   └── monaco/            # Editor integration
│
└── rpc/                   # RPC server
    ├── README.md
    └── server.mts         # RPC server implementation
```

## Development Workflow

### Setting Up Development Environment

1. **Clone Repository**:
```bash
git clone https://github.com/creatorsim/creator.git
cd creator
```

2. **Install Dependencies**:
```bash
npm install  # or yarn install
```

3. **Build**:
```bash
npm run build
```

4. **Run Tests**:
```bash
npm test
```

### Making Changes

1. **Branch**: Create feature branch
2. **Code**: Make your changes
3. **Test**: Verify functionality
4. **Document**: Update docs
5. **Commit**: Descriptive commit message
6. **PR**: Submit pull request

## Key Concepts

### Architecture Definition

Architectures are defined in YAML files specifying:
- Instruction sets
- Register banks
- Memory layout
- System calls
- Interrupts

See [Creating Custom Architectures](custom-architectures.md).

### Instruction Execution

The execution pipeline:
1. **Fetch**: Read instruction from memory
2. **Decode**: Parse instruction format
3. **Execute**: Perform operation
4. **Writeback**: Update registers/memory
5. **Update**: Advance PC

### State Management

CREATOR maintains complete simulator state:
- Register values
- Memory contents
- Program counter
- Execution mode
- Device states

State can be saved/restored via snapshots.

### Device System

Devices provide I/O and OS services:
- Memory-mapped interface
- Control/status registers
- Data buffers
- Callback handlers

See [Devices](devices.md) for details.

## API Overview

### Core API (`core.mjs`)

```javascript
// Load architecture
loadArchitecture(yamlString, isaExtensions)

// Compile assembly
assembly_compile(source, compiler)

// Execute instructions
step()  // Execute one instruction
reset() // Reset to initial state

// State management
snapshot(extraData)
restore(stateJson)

// Register access
dumpRegister(name, format)
getRegisterInfo(name)

// Memory access
main_memory.readWord(address)
main_memory.writeWord(address, bytes)
```

### Memory API

```javascript
// Create memory
new Memory({
    sizeInBytes,
    bitsPerByte,
    wordSize,
    baseAddress,
    endianness
})

// Access
memory.read(address)
memory.write(address, value)
memory.readWord(address)
memory.writeWord(address, bytes)

// Hints
memory.setHint(address, {tag, type, sizeInBits})
memory.getHint(address)
```

## Design Principles

### Modularity
- Clear separation of concerns
- Independent subsystems
- Well-defined interfaces

### Extensibility
- Plugin architecture for assemblers
- Custom device support
- Flexible architecture definitions

### Testability
- Unit testable components
- Integration test support
- Snapshot-based testing

### Performance
- Efficient execution
- Caching where appropriate
- Lazy evaluation

## Contributing

### Code Style
- TypeScript/JavaScript ES6+
- Consistent formatting
- Descriptive names
- Comments for complex logic

### Testing
- Unit tests for new features
- Integration tests for workflows
- Regression tests for bugs

### Documentation
- Update relevant docs
- Add examples
- Explain design decisions

## Resources

- **GitHub**: Source code and issues
- **Architecture Files**: `architecture/` directory
- **Examples**: `examples/` directory
- **Tests**: `tests/` directory

## Guide Structure

1. **[Core Architecture](core-architecture.md)**: Overall system design
2. **[Execution Engine](execution-engine.md)**: How instructions execute
3. **[Memory System](memory-system.md)**: Memory implementation
4. **[Devices](devices.md)**: Device abstraction and implementation
5. **[Interrupts](interrupts.md)**: Interrupt system
6. **[Assembler Integration](assemblers.md)**: Adding assemblers
7. **[Creating Custom Architectures](custom-architectures.md)**: Architecture design
8. **[Privileged Instructions](privileged.md)**: Privilege levels

## Next Steps

- Read [Core Architecture](core-architecture.md) for system overview
- Study [Devices](devices.md) to understand I/O
- Learn [Interrupts](interrupts.md) for interrupt handling
- Explore [Assembler Integration](assemblers.md) for compilation
- Design [Custom Architectures](custom-architectures.md)
