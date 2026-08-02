---
title: "Copy-and-Patch Compilation: A Fast Compilation Algorithm for High-Level Languages and Bytecode"
source: https://dl.acm.org/doi/10.1145/3485513
author: Haoran Xu, Fredrik Kjølstad
company: Stanford University (adopted in CPython 3.13, .NET CoreCLR)
date_posted: 2021-10-01
date_digested: 2026-08-02
---

# Copy-and-Patch Compilation

## What's new to learn

1. **Stencil-based code generation**: Each opcode is pre-compiled at build time into a native machine code fragment (a *stencil*) with designated *holes* left for the values that vary at runtime (operands, stack offsets, branch targets). JIT compilation then becomes: copy the stencil bytes, fill in the holes.

2. **Holes as relocations**: The holes in stencils are not an ad-hoc mechanism — they are exactly the relocation records that the system linker uses in every `.o` file. Copy-and-patch exploits the existing abstraction: mark variable parts as external symbols, compile with LLVM, extract the resulting relocation entries as the hole table. JIT compilation is link-time relocation at runtime.

3. **Offline/online phase separation in compilers**: All expensive compilation work (instruction selection, register allocation, code layout) happens *offline* during the interpreter build. The runtime's only job is `memcpy` plus arithmetic fixups. Compilation cost per opcode execution is O(holes), not O(program × optimization_passes).

## Prerequisites

- **Bytecode interpreter basics**: an interpreter dispatches a loop over opcodes, calling a handler for each. Stack VMs (CPython, JVM) store operands on an eval stack; register VMs (Lua 5) use virtual registers.
- **What JIT compilation is**: generating native machine code from bytecode at runtime, as opposed to ahead-of-time (AOT) compilation.
- **Object file relocations** (helpful but not required): when a compiler emits a `.o` file with a call to an external function, it leaves a "hole" at the call-site and records a relocation entry. The linker later fills in the target address. This is the exact mechanism copy-and-patch re-uses.
- **W^X (write XOR execute)**: memory pages can be writable or executable but not both simultaneously. Any JIT must allocate write-mode memory, fill it, then flip it to execute-mode.

## The core idea

Modern language runtimes often have a three-tier execution stack:

| Tier | Speed | How it works |
|------|-------|--------------|
| Interpreter | 1× | Dispatch loop; one handler call per opcode |
| Template / copy-and-patch JIT | 2–4× | Copy pre-compiled stencils, patch holes |
| Optimizing JIT (e.g., LLVM) | 5–50× on hot loops | Full IR + optimization passes |

The fundamental question is: **what makes traditional JIT compilation expensive?**

Not the code-generation itself — emitting machine bytes is cheap. The cost is the *optimization pipeline*: instruction selection, register allocation, dead-code elimination, constant propagation, inlining. These passes grow super-linearly with program size.

But here is the observation that makes copy-and-patch possible: **for a dispatched bytecode interpreter, the structure of each opcode is fixed at write time**. The interpreter author already decided what operations each `LOAD_CONST`, `BINARY_ADD`, or `CALL` does. The only things that change at runtime are the operands — the literal value of the constant, the stack depth, the branch target.

So: do the expensive work once, at build time. Pre-compile each opcode handler to machine code with the variable parts marked as external symbols. LLVM produces object code with relocation records for each external symbol. Those relocation records define the "holes" in the stencil. At runtime, copy the stencil and patch the holes. Compilation is now just `memcpy` + a handful of pointer writes.

This is **Futamura's First Projection** made concrete:

> Specializing an interpreter on a specific program = compiling that program to native code.

Copy-and-patch pre-computes the offline part of that specialization (per opcode type), and defers only the trivial online part (per opcode instance) to runtime.

## Mechanics

### Build time: generating stencils

1. **Write opcode handlers as C functions**, using placeholder external symbols for each variable quantity:

   ```c
   // external symbols that will become holes
   extern int64_t _JIT_OPERAND_A;
   extern char*   _JIT_TARGET_LABEL;

   void stencil_BINARY_ADD() {
       int64_t a = stack_pop();
       int64_t b = stack_pop();
       stack_push(a + b + _JIT_OPERAND_A);  // example: add an immediate constant
       goto *_JIT_TARGET_LABEL;             // branch target
   }
   ```

2. **Compile with LLVM** (invoked at CPython build time, not at runtime). The compiler produces machine code for the function body, but leaves `_JIT_OPERAND_A` and `_JIT_TARGET_LABEL` as unresolved references — i.e., relocation entries in the object file.

3. **Extract stencil + hole table**:
   - Stencil bytes = the raw machine code bytes from the object file section
   - Hole list = `[(offset_in_stencil, relocation_type, symbol_name), ...]` from the ELF/Mach-O relocation table
   - Serialize both as C arrays baked into the interpreter binary

The result: a compiled interpreter binary that contains the stencil bytes for every opcode, plus the hole tables describing how to patch them. No LLVM dependency at runtime.

### Runtime: JIT compilation

When the Tier-1 interpreter detects a hot execution trace (in CPython: the "specializing adaptive interpreter" marks hot bytecodes):

1. **Allocate an executable code buffer** (mmap with PROT_WRITE, later flipped to PROT_EXEC).

2. **For each opcode in the trace**:
   ```
   a. Look up stencil[opcode]  → (bytes, holes)
   b. dst = buffer_cursor
   c. memcpy(dst, bytes, stencil_size)
   d. for each hole in holes:
        value = resolve(hole.symbol, current_instruction)
        write_value_at(dst + hole.offset, value, hole.reloc_type)
   e. buffer_cursor += stencil_size
   ```

3. **Flip memory to PROT_EXEC** (icache flush on AArch64; implicit on x86-64).

4. **Jump to buffer start**.

Patching is arithmetic: for a PC-relative branch relocation, `value = target_address - (stencil_start + reloc_offset)`. For an absolute address, `value = target_address`. A few lines of C per relocation type.

### Stencil variants handle instruction-level specialization

Not every opcode has one stencil. The optimizer may generate multiple variants. For example, CPython's Tier-2 compiler has a variant for `BINARY_OP_ADD_INT` (two boxed Python integers) and a separate variant for unboxed native integers. Each variant is a separate stencil. The runtime selects the cheapest correct variant for each instruction, based on type feedback from the Tier-1 specializing interpreter.

### Size and complexity

- **Build-time code**: ~1,000 lines of Python that orchestrates LLVM to extract stencils
- **Runtime JIT code**: ~100–400 lines of C (stencil lookup, memcpy, hole patching)
- **Stencil library**: compiled into the interpreter binary; for CPython ~600 KB of stencil bytes for ~500 micro-opcode variants

## Where it breaks

**No cross-opcode optimization.** Each stencil is compiled independently. A traditional optimizing JIT can inline across opcode boundaries, hoist loop-invariant loads, eliminate redundant type checks, and apply global register allocation over an entire loop body. Copy-and-patch cannot: it stitches together independently compiled fragments. Stencils must independently save/restore registers, preventing the register-assignment optimizations that a whole-function JIT performs.

**Stencil overhead per instruction.** Because each stencil includes the full prologue/epilogue for register management, consecutive stencils duplicate that work. A tight inner loop compiled with LLVM might use 5–10 instructions; the same loop as a sequence of stencils might take 50–100.

**Numerical workloads lose.** For compute-intensive Python code (NumPy-style loops), Numba (LLVM-backed JIT) still outperforms copy-and-patch JIT by 5–50×. Copy-and-patch is a clear win for general-purpose interpreted code but does not replace domain-specific JIT compilers for heavy numerical computation.

**Platform matrix at build time.** Stencils must be regenerated for each target ISA (x86-64, AArch64, riscv64, s390x, ...). This multiplies the build matrix. CPython 3.15 added RISC-V support; each new ISA requires new stencils and new relocation-type handlers.

**Startup cost shifts to build time.** Interpreter builds now require LLVM (even if only during the build process). This adds build-time complexity and a dependency on LLVM's object-file formats, which evolve across versions.

**CPython 3.13–3.14 measured ≤5% speedup on pyperformance** (the standard benchmark suite), sometimes negative, because most Python programs spend more time in C extension calls and I/O than in bytecode dispatch. The JIT only accelerates the bytecode evaluation layer, not CPython's massive C extension ecosystem.

## Why it works

The deep pattern is **offline/online phase separation** — one of the most powerful organizing principles in systems engineering:

| Problem | Offline (expensive, amortized) | Online (cheap, per-request) |
|---------|-------------------------------|------------------------------|
| Copy-and-patch JIT | Compile stencils via LLVM | memcpy + patch holes |
| SQL prepared statements | Parse + plan the query | Bind parameters + execute |
| Dynamic linking | Compile .o files, emit relocations | Linker resolves addresses at load time |
| Maglev consistent hash | Precompute lookup table | O(1) index lookup |
| C++ templates | Compile one instantiation per type | Zero per-call overhead |

In all cases, the key invariant is: **the expensive work depends only on the structure (opcode type, query shape, function signature), not on the specific runtime values**. Factor out what is structurally constant; defer only what genuinely varies.

For copy-and-patch specifically, the structural constant is "how do you implement BINARY_ADD on this machine?" — that is the stencil. The runtime variable is "what are the specific stack depth and operand value for this particular BINARY_ADD in this particular program?" — that is the hole-fill.

This is Futamura's First Projection restated: any interpreter can be turned into a compiler by separating opcode semantics (static) from operand binding (dynamic). Copy-and-patch is the most literal engineering realization of that separation: the stencil IS the compiled semantics, and patching IS the operand binding.

There is also a connection to **object file linking**: copy-and-patch at runtime is doing exactly what `ld` does at link time — taking relocatable code and fixing up the undefined references. JIT compilation becomes link-time resolution, just deferred from build time to run time. This explains why copy-and-patch required so little new infrastructure: the hard problem (expressing "this code has holes") was already solved by the object file format.

## Going deeper

1. **Original paper**: Haoran Xu and Fredrik Kjølstad, "Copy-and-Patch Compilation," OOPSLA 2021. The arXiv preprint is freely available at https://arxiv.org/abs/2011.13127 — includes the formal semantics of stencil generation, the template liveness analysis for minimizing stencil size, and TPC-H benchmarks showing 100–1000× faster code generation than LLVM.

2. **Deegen**: A more recent system (arXiv 2411.11469) that extends copy-and-patch with a tracing JIT layer — stencils are the base tier and the tracer fuses them into longer regions that CAN be optimized across opcode boundaries. Demonstrates that the two tiers (copy-and-patch for fast warm-up, tracing for peak throughput) compose cleanly.

3. **PEP 744** and Brandt Bucher's CPython implementation: https://peps.python.org/pep-0744/ — the design document explaining how copy-and-patch integrates with CPython's existing Tier-1 specializing adaptive interpreter. The relationship between the two tiers (Tier-1 type-specializes; Tier-2 copy-and-patch compiles hot traces) shows how copy-and-patch slots into a multi-tier execution architecture.
