---
name: fortran
description: Write, review, and modernize Fortran code with modern (2018/2023) idiom, fpm-based tooling, and verified compiler flags. Use whenever a task involves Fortran — .f90/.f95/.f03/.f08 or fixed-form .f/.for files, fpm projects, numerical/HPC code, maintaining legacy FORTRAN 77, or C-Fortran interop via iso_c_binding. Assistants default to legacy F77 idiom and their Fortran often fails to compile; this skill corrects toward modern practice and mandates a compile-verify loop. Covers greenfield code, legacy maintenance (match existing style; don't half-modernize), toolchain setup (fpm, stdlib, gfortran/ifx/flang), and standards guidance (target F2018 as the safe baseline).
metadata:
  audience: coding assistants working on Fortran code
  domain: fortran, scientific computing, HPC
  created: "2026-07-26"
  sources-verified: "2026-07-26"
  maintainer: Brian Heseung Kim (@brhkim)
---

# Fortran

Guidance for writing, reviewing, and modernizing Fortran. The core problem this skill solves: model training data is dominated by legacy FORTRAN 77 style (fixed-form, implicit typing, COMMON blocks, GOTO), so generated Fortran drifts legacy and frequently fails to compile — a 2024 benchmark found Fortran among the worst languages for LLM compilation success (https://fortran-lang.discourse.group/t/benchmarking-large-language-models/7880, accessed 2026-07-26). Modern Fortran (2008/2018/2023) is a different-looking language with a real package manager (fpm) and standard library (stdlib). Default to modern idiom for new code; match existing style when maintaining legacy.

> **Maintainer note:** Created 2026-07-26; all sources verified as of that date. Each reference file opens with its own staleness note. Fastest-drifting facts: compiler versions, Fortran 2023 conformance, fpm/stdlib releases.

## Modern Defaults (new code — non-negotiable unless the codebase says otherwise)

| Rule | Instead of (legacy) | Why |
|------|--------------------|-----|
| Free-form source (`.f90` suffix regardless of standard) | Fixed-form columns 1–72 | Fixed-form silently truncates at column 72 |
| `implicit none` in every program unit and module | Implicit I–N integer typing | Typos become silent wrong-type variables |
| Modules + `use ..., only:` | `COMMON` blocks, `include` | Compile-time interface checking |
| `intent(in/out/inout)` on every dummy argument | Unannotated arguments | Compiler catches aliasing/mutation bugs |
| Allocatable arrays | Pointers, fixed max-size arrays | No leaks; auto-deallocation |
| Kind parameters via `iso_fortran_env` (`real64`) | `real*8`, `double precision` | Portable, standard-conforming |
| Structured `do` / `end do`, `exit`, `cycle` | Labeled `DO`/`CONTINUE`, `GOTO` | Readability; GOTO forms are obsolescent |
| Explicit `result` + pure/elemental where possible | Side-effecting functions | Optimizer- and reader-friendly |

Full ruleset with reasoning, examples, and gotchas: `references/modern-idiom.md`.

## Decision Tree

```
Fortran task?
├─ New code / new project
│   ├─ Set up project → fpm (`fpm new`), see references/toolchain.md
│   ├─ Write code → modern defaults above + references/modern-idiom.md
│   └─ Which standard → target F2018; F2023 cautiously (references/standards.md)
├─ Maintaining existing code
│   ├─ Fixed-form / F77 style → match existing style; do NOT half-modernize
│   │   → references/legacy-and-interop.md (fixed-form rules, COMMON, etc.)
│   └─ Deliberate modernization pass → 7-step order of operations,
│       references/legacy-and-interop.md
├─ C ↔ Fortran interop → iso_c_binding recipes,
│   references/legacy-and-interop.md
└─ Build/compile problems, flags, compiler choice → references/toolchain.md
```

## Compile-Verify Loop (always)

Generated Fortran must be compiled before being declared done. Do not skip this — compilation failure is the documented LLM failure mode in Fortran.

```bash
# Preferred verifier (most complete runtime checking):
gfortran -std=f2018 -Wall -Wextra -fcheck=all -fbacktrace -ffpe-trap=invalid,zero,overflow -o prog src.f90
# In an fpm project:
fpm build --flag "-std=f2018 -Wall -fcheck=all"
```

If only Flang is available: a clean Flang build is NOT sufficient verification — Flang has no documented runtime bounds-check/FPE-trap flags; cross-check with gfortran when possible (see `references/toolchain.md`). Legacy fixed-form code: compile with `-std=legacy -ffixed-form`, and establish a passing baseline *before* editing.

## Compiler Quick Table

| Compiler | Status (as of 2026-07-26) | Strict/debug flags |
|----------|---------------------------|--------------------|
| gfortran | Stable, default open-source choice | `-std=f2018 -Wall -Wextra -fcheck=all -fbacktrace -ffpe-trap=...` |
| Intel ifx | Current Intel compiler; classic ifort discontinued (oneAPI 2025) | `-stand f18 -warn all -check all -traceback -fpe-all=0` |
| LLVM Flang | Usable, still maturing; no runtime-check flags | `-std=f2018 -Werror` (limited) |
| LFortran | Alpha; interactive/REPL niche only | — |

Details, citations, and cross-compiler flag mapping: `references/toolchain.md`.

## Reference Files

| File | Load when |
|------|-----------|
| `references/modern-idiom.md` | Writing or reviewing modern Fortran: arrays, allocatables, floats, strings, I/O, derived types, gotchas |
| `references/toolchain.md` | fpm projects, stdlib, compiler selection, exact flags, build errors |
| `references/legacy-and-interop.md` | Fixed-form/F77 code, modernization passes, iso_c_binding / mixed-language builds |
| `references/standards.md` | Standard-version questions, F2023 features, conformance status, free standard PDFs |

For intrinsic function signatures and syntax details, prefer the community-maintained reference over model memory: https://fortran-lang.org/learn/intrinsics/ (man-page-style intrinsics index; accessed 2026-07-26).

All reference files cite every substantive claim with its exact source URL and access date; claims resting on model knowledge rather than a fetched page are flagged inline as "unverified — model knowledge." Trust the citations over this summary if they ever disagree.
