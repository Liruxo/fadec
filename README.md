# Fadec — Fast Decoder for x86-32 and x86-64

Fadec is a fast and lightweight decoder for x86-32 and x86-64. To meet the goal of speed, lookup tables are used to map the opcode the (internal) description of the instruction encoding. This table currently has a size of roughly 19.5 kiB (for 32/64-bit combined).

*Note: This is not a disassembler, it does not intend to produce valid assembly.*

## Key features

> **Q: Why not just just use any other decoder available out there?**
>
> A: Because I needed to embed a small and fast decoder in a project which didn't link against a libc.

- **Small size:** the compiled library uses only 40 kiB and the main decode routine is only a few hundreds lines of code.
- **Performance:** Fadec is significantly faster than libopcodes or Capstone due to the absence of high-level abstractions and the small lookup table.
- **Almost no dependencies:** the formatter only uses the function `snprintf`, the decoder itself has no dependencies, making it suitable for environments without a full libc or `malloc`-style memory allocation.
- **Correctness:** even corner cases should be handled correctly (if not, that's a bug), e.g., the order of prefixes, the presence of the `lock` prefix, or properly handling VEX.W in 32-bit mode.

## Basic Usage
```c
FdInstr instr;
// Decode from buffer in 64-bit mode and virtual address 0x401000
int ret = fd_decode(buffer, sizeof(buffer), 64, 0x401000, &instr);
// ret<0 indicates an error, ret>0 the number of decoded bytes
// Relevant properties of instructions can now be queries using the FD_* macros.
```

## API

The API consists of two functions to decode and format instructions, as well as several accessor macros (see [fadec.h](fadec.h)). Direct access of the `FdInstr` structure is not recommended.

- `int fd_decode(const uint8_t* buf, size_t len, int mode, uintptr_t address, FdInstr* out_instr)`
    - Decode a single instruction
    - Return value: number of bytes used, or a negative value in case of an error.
    - `buf`/`len`: buffer containing instruction bytes. At most 15 bytes will be read. If the instruction is longer than `len`, an error value is returned.
    - `mode`: architecture mode, either `32` or `64`.
    - `address`: virtual address of the decoded instruction. This is used for computing jump targets and segment-offset-relative memory operations (MOV with moffs* encoding) and stored in the instruction.
    - `out_instr`: Pointer to the instruction buffer, might get written partially in case of an error.
- `void fd_format(const FdInstr* instr, char* buf, size_t len)`
    - Format a single instruction to a human-readable format.
    - `instr`: decoded instruction.
    - `buf`/`len`: buffer for formatted instruction string

## Intended differences to other decoders
To achieve higher performance, minor differences to other decoders exist, requiring special handling.

- The decoded operand sizes are not always exact. However, the exact size can be reconstructed in all cases.
    - For instructions with rare memory access sizes (e.g. `lgdt`), the provided size is zero. These are: `cmpxchg16b`, `cmpxchg8b`, `fbld` (for 80-bit), `fbstp` (for 80-bit), `fldenv`, `frstor`, `fsave`, `fstenv`, `fstp` (for 80-bit), `fxrstor`, `fxsave`, `lds`, `lds`, `lgdt`, `lidt`, `lldt`, `ltr`, `sgdt`, `sidt`, `sldt`, `str`
    - For some SSE/AVX instructions, the operand size is an over-approximation of the real size, e.g. for permutations or extensions.
    - The operand size of segment and FPU registers is always zero.
- An implicit `fwait` in FPU instructions is decoded as a separate instruction (matching the opcode layout in machine code). For example:
    - `finit` is decoded as `FD_FWAIT` + `FD_FINIT`
    - `fninit` is decoded as plain `FD_FINIT`
- For `scas` and `cmps`, the `repz` prefix can be queried using `FD_HAS_REP` (matching prefix byte in machine code).

## Known issues
- The EVEX prefix (AVX-512) is not supported (yet).
- The layout of entries in the tables can be improved to improve usage of caches. (Help needed.)
- No Python API.
- Low test coverage. (Help needed.)
- No benchmarking has been performed yet. (Help needed.)
- Prefixes for indirect jumps and calls are not properly decoded, e.g. `notrack`, `bnd`. This requires additional information on the prefix ordering, which is currently not decoded. (Analysis of performance impact and help needed.)

If you find any other issues, please report a bug. Or, even better, send a patch fixing the issue.
