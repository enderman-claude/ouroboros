# Self-Hosted Compiler Project: Notes

A compiler written in its own language, bootstrapped from a minimal Rust seed. The repo name ends in `-bootstrap` for the seed.

## Overview

There are two compilers and one language, with two feature tiers.

- **Rust seed (`-bootstrap`)**: a minimal compiler written in Rust. It has no optimisations, no JIT, no AOT, limited syscall access, and limited builtin packages. It can run scripts and programs, but it is not meant to be used for them. Its only job is to compile the minimal language well enough to produce stage 1.
- **Minimal language**: a restricted subset of the language. It is enough to write a compiler, and it can build itself.
- **Full compiler**: the complete compiler, with optimisations, JIT, and AOT. Its source must be expressible in the minimal language, or the minimal language must be extended first.

Build chain:

```
Rust seed  ->  stage 1 (minimal compiler binary)
stage 1    ->  stage 2 (compiler built by stage 1)
stage 2    ->  stage 3 (compiler built by stage 2)
stage 2 == stage 3 byte-for-byte  =>  fixed point reached
```

The full compiler is written in the minimal subset (or an extended version of it) and is built through this chain.

## Design Principles

### Keep the language small

The compiler only needs the subset of the language it is written in. Features added before the compiler needs them are extra work. The first self-hosted version should use only a handful of constructs:

- functions
- integers
- strings
- structs or tagged unions
- simple control flow

### Use the fixed-point test

Compile the self-hosted compiler with the Rust seed (stage 1), use that output to compile itself (stage 2), then compile again (stage 3). If stage 2 and stage 3 produce byte-identical binaries, the compiler is very likely correct. GCC and Go use the same approach.

### Keep the Rust seed as a frozen recovery tool

- Do not delete it once self-hosting works.
- Do not add convenience features to it, even when they would make the full compiler's source easier to write.
- Change it only to recover from a broken state or to support a minimal-language extension the full compiler needs.
- Tag its stable points.

### Keep the bootstrap's job narrow

The seed compiles the minimal subset and nothing more. JIT, AOT, and optimisation logic never go into it.

### Test against a shared corpus

Source programs with expected outputs form a contract that both the seed and the full compiler must satisfy. Tests for user-facing behaviour, optimisations, JIT, and AOT belong to the full compiler, not the seed.

### Automate the fixed-point loop

Stage 1 -> stage 2 -> stage 3 should be a single command. If the loop is painful, it gets skipped, and bugs accumulate.

## Repository Layout

Two options were discussed.

**Option A: separate repos by role**

- `lang-bootstrap`: the Rust seed, frozen and tagged.
- `lang`: the self-hosted compiler, including the full version with JIT and AOT.
- `lang-tests`: the shared corpus of source programs and expected outputs.

**Option B: monorepo**

```
lang/
  bootstrap/   Rust seed, tagged at stable points
  compiler/    self-hosted compiler (minimal subset, then full)
  tests/       shared corpus
  scripts/     fixed-point loop and test runner
```

Option B reduces cross-repo friction and keeps history in one place. Option A keeps the seed's history separate from active development.

## Compiler Architecture

### Shared front end and IR

Both AOT and JIT need the same lexer, parser, type checker, IR, and most optimisation passes. The only difference is the output stage:

- **AOT** emits an object file or executable once.
- **JIT** emits machine code into executable memory and calls it.

Design the backend around a clean IR, and most of the work is shared.

### IR

The Rust seed has no IR, so the first real decision is the IR's shape. An SSA-style IR is the usual choice, because constant folding, dead code elimination, inlining, and register allocation are easier to express on it. Keep it simple at first.

### Optimisations

Optimisation passes will likely need features the minimal language lacks, such as generics, closures, or richer types. Two options:

1. Write the passes in the minimal subset in an awkward style.
2. Extend the minimal language first, then rerun the fixed-point test with each extension.

Extending the language is usually the better choice, but the Rust seed must learn each extension too.

### JIT

- Needs executable memory: `mmap` with `PROT_EXEC` on Linux, `VirtualAlloc` on Windows.
- Needs a calling convention callable from the host.
- Needs a strategy for code invalidation if code is recompiled.
- Decide early whether the JIT is a separate runtime library or part of the compiler binary.

### AOT

- Start by emitting an object file and calling the system linker.
- A custom linker can come later.

### Backend target

Not yet decided. Candidates discussed: x86-64, ARM, or a VM or bytecode target. A VM target is much easier to start with. Native code generation adds work around calling conventions and register allocation and can wait.

## Build Order

1. Implement the IR in the Rust seed, with a simple lowering from the AST.
2. Get AOT working end to end for the minimal language with a simple backend, even if unoptimised.
3. Add optimisation passes one at a time, checking each against the fixed-point build.
4. Add the JIT as a second consumer of the same backend, reusing the code generator and swapping only the output sink and loader.
5. Port new passes to the minimal language once the seed handles them.

## Open Questions

- Backend target: x86-64, ARM, or a VM?
- Should the JIT run inside the compiled language's runtime, or as a separate tool?
- Does the full compiler need features the minimal language lacks (generics, richer types)? This is the decision that shapes the most work.
- Does the Rust seed ever need to change after the full compiler works, or is it purely a recovery tool?
- Monorepo (Option B) or separate repos (Option A)?

## Status

- [ ] Rust seed compiles the minimal subset
- [ ] Stage 1 -> stage 2 -> stage 3 fixed point reached
- [ ] Shared test corpus in place
- [ ] IR defined
- [ ] AOT end to end
- [ ] Optimisation passes
- [ ] JIT
