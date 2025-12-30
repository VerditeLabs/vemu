# CLAUDE.md

This file provides guidance for Claude Code when working with this repository.

## Project Overview

**vemu** is a vectorized emulation framework for high-performance fuzzing. It uses AVX-512 SIMD instructions to run up to 16 virtual machines simultaneously per thread, achieving billions of instructions per second for snapshot-based fuzzing.

### Key Concepts

- **Vectorized Emulation**: JIT-compiles target code to AVX-512 equivalents, running 16 VMs per thread using 512-bit registers (zmm0-zmm31)
- **Snapshot Fuzzing**: Starts from partially-executed system state (memory + registers) for deterministic, high-performance fuzzing
- **Soft MMU**: Custom memory management with byte-level permissions for stronger-than-ASAN memory protections
- **Differential Coverage**: Tracks code, register, and memory state across VMs to detect divergence

### Architecture

- Targets run on Xeon Phi or AVX-512 capable processors
- Memory is interleaved at dword (32-bit) level across all 16 VMs
- Uses kmask registers (k0-k7) for conditional lane execution when VMs diverge
- Guest address translation: host_addr = guest_addr * 16 (simplified)

## Build System

This is a C/C++ project. Common file types:
- `.c` / `.cpp` - Source files
- `.h` / `.hpp` - Header files
- Object files, libraries, and executables are gitignored

## Development Guidelines

### Performance Considerations

- Instruction decoder is typically the bottleneck on Xeon Phi
- Prefer memory loads over immediate values in generated code
- Use `{1to16}` broadcasting for constant operands
- Aligned 64-byte (512-bit) memory operations are preferred

### Code Generation

When working with JIT code:
- Map guest registers to zmm registers
- Use `vpaddd`, `vpsubd`, etc. instead of scalar operations
- Handle divergence with kmask conditional execution
- Memory operations use `vmovdqa32` for aligned access

### Memory Model

- VMs share interleaved memory layout
- Byte operations require read-modify-write with shifting/masking
- Unaligned operations need multiple memory accesses
