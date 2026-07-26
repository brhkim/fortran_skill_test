<!-- §§ METADATA -->
# Fortran Toolchain Reference: Build Tooling & Compilers

> **File created:** 2026-07-26
> **Sources verified as of:** 2026-07-26 (all URLs fetched directly on this date; see the
> **Sources** section at the end for the full list, and the inline `accessed 2026-07-26`
> annotations on each claim).
> **Staleness note:** The fastest-drifting facts in this file are (1) **fpm release version**,
> (2) **Intel ifx / oneAPI release version and Fortran 2023 coverage**, (3) **LLVM Flang major
> version and standards-conformance status**, and (4) **LFortran alpha/beta milestone counter**.
> Re-check the fpm releases page, the Intel Fortran Compiler release notes, the Flang standards
> page, and the LFortran site before relying on any version number older than a few months.
> Version numbers and conformance status age much faster than the flag spellings and the
> conceptual guidance, which are comparatively stable.

This reference covers the modern Fortran build toolchain: the Fortran Package Manager (fpm),
the Fortran Standard Library (stdlib), the four compilers a coding assistant is likely to
encounter, per-compiler strictness/debugging flags, a well-known cross-compiler I/O gotcha,
and a recommended "compile-verify loop" that assistants should run before declaring a task done.

Scope boundary: this file is about **build tooling and compilers only**. Language-level modern
Fortran guidance (features, idioms, style) lives in the sibling reference files of this skill.

---

<!-- §§ FPM_OVERVIEW -->
## 1. fpm — the Fortran Package Manager

### 1.1 What fpm is and current status

fpm is the official "Package manager and build system for Fortran"
([fpm.fortran-lang.org](https://fpm.fortran-lang.org/), accessed 2026-07-26). It manages
dependencies and builds projects with an integrated package-management workflow, analogous
to Rust's `cargo` or Node's `npm` but for Fortran.

- **Current release: v0.13.0, released 2026-02-17**
  ([github.com/fortran-lang/fpm/releases](https://github.com/fortran-lang/fpm/releases),
  accessed 2026-07-26). This release introduced `features` and `profiles` (a build-customization
  system for conditional compilation, compiler-specific settings, and platform-dependent
  behavior), improved macro parsing, support for multiple simultaneous library targets,
  gcc-15 fixes, and added `amdflang` to the supported-compiler list.
- **Recent release history** (same source, accessed 2026-07-26; dates corrected in the
  2026-07-26 review pass): v0.13.0 (2026-02-17), v0.12.0 (2025-05-18),
  v0.11.0 (2025-03-10), v0.10.1 (2024-03-24), v0.10.0 (2024-01-08).
- fpm is **pre-1.0** — the `0.x` version line means the manifest format and CLI can still
  change between minor releases. The docs themselves carry an "under construction" note
  ([fpm.fortran-lang.org](https://fpm.fortran-lang.org/), accessed 2026-07-26). Treat the
  manifest schema below as current-as-of-v0.13.0, not permanently frozen.

### 1.2 Installing fpm

All commands below are quoted from the fpm install page
([fpm.fortran-lang.org/install/index.html](https://fpm.fortran-lang.org/install/index.html),
accessed 2026-07-26).

| Method | Command(s) |
|--------|-----------|
| Conda (conda-forge) | `conda create -n fpm fpm` or `conda install fpm` |
| pipx | `pipx install fpm` |
| Homebrew (macOS) | `brew tap fortran-lang/homebrew-fortran` then `brew install fpm` |
| MacPorts (macOS) | `sudo port install fpm` |
| MSYS2 (Windows) | `pacman -S git mingw-w64-x86_64-gcc-fortran mingw-w64-x86_64-fpm` |
| Spack | `spack install fpm` |
| Arch Linux (AUR) | `git clone https://aur.archlinux.org/fortran-fpm.git` then `makepkg -si` |
| OpenBSD | `cd /usr/ports/devel/fpm` then `make install clean` |
| WinGet (Windows) | `winget install FortranLang.fpm` |

**Precompiled binaries** for macOS/Linux/Windows (x86-64) are on the
[releases page](https://github.com/fortran-lang/fpm/releases); on Linux/macOS you must
`chmod +x` the downloaded binary (accessed 2026-07-26).

**Build from source**: run the `./install.sh` script (installs to `~/.local/bin/fpm`), or do a
manual three-step bootstrap — download `fpm.F90`, compile it with gfortran, then use that
bootstrap binary to build the feature-complete fpm (accessed 2026-07-26).

### 1.3 `fpm new` project layout

`fpm new <name>` scaffolds a standard project. The conventional directory layout fpm
uses is:

```
myproject/
├── fpm.toml         # the package manifest (metadata, deps, build settings)
├── README.md
├── src/             # library source (modules) — auto-discovered
│   └── myproject.f90
├── app/             # main program(s) — auto-discovered as executables
│   └── main.f90
├── test/            # test program(s) — auto-discovered as tests
│   └── check.f90
└── example/         # example program(s) — auto-discovered
```

The source-directory conventions (`src/`, `app/`, `test/`, `example/`) are the defaults that
fpm's auto-discovery keys on; they map to `[library]`, `[[executable]]`, `[[test]]`, and
`[[example]]` targets respectively, and each can be overridden in `fpm.toml` via a
`source-dir` field (manifest reference,
[fpm.fortran-lang.org/spec/manifest.html](https://fpm.fortran-lang.org/spec/manifest.html),
accessed 2026-07-26).

> Note: the exact set of subfolders `fpm new` emits (and whether `example/` is created by
> default) is a scaffolding detail that has varied across fpm versions. The `src/ app/ test/`
> trio is stable; verify `example/` against your installed fpm version if it matters.
> (Flagged as needing local verification — model knowledge of the scaffolder's exact output.)

### 1.4 Core commands

| Command | Purpose |
|---------|---------|
| `fpm build` | Compile the library, executables, and (per flags) tests/examples. |
| `fpm run` | Build (if needed) and run an executable target; `fpm run -- <args>` passes args. |
| `fpm test` | Build and run the test targets under `test/`. |
| `fpm install` | Install built artifacts (uses the `[install]` manifest section). |
| `fpm publish` | Publish a package to the registry; supports `--dry-run`. |

`fpm publish` and its `--dry-run` option are explicitly documented
([fpm.fortran-lang.org](https://fpm.fortran-lang.org/), accessed 2026-07-26). The
`build`/`run`/`test`/`install`/`new` verbs are the standard package-manager workflow the site
describes; their exact behavior and flags are documented in the fpm tutorials (accessed
2026-07-26).

To pass compiler flags to a build, fpm supports profiles and the `--flag` option, and reads
the `FFLAGS`/compiler-selection environment (see §5 for the recommended strict-flag
invocation). Compiler selection is via `--compiler <name>` (e.g. `gfortran`, `ifx`, `flang`)
and `--profile <name>` (e.g. `debug`, `release`).

<!-- §§ FPM_MANIFEST -->
### 1.5 Worked `fpm.toml` example (verbatim from official docs)

The following is a consolidated example manifest reproduced verbatim from the fpm manifest
reference ([fpm.fortran-lang.org/spec/manifest.html](https://fpm.fortran-lang.org/spec/manifest.html),
accessed 2026-07-26). It demonstrates every major section: package metadata, `[fortran]` and
`[build]` settings, `[library]`, `[[executable]]`, `[[example]]`, `[[test]]`, root
`[dependencies]` (including all git-dependency variants), `[dev-dependencies]`, `[install]`,
`[preprocess]`, and `[extra]`.

```toml
# Package Metadata
name = "hello_world"
version = "1.0.0"
license = "Apache-2.0 OR MIT"
author = "Jane Doe"
maintainer = "jane.doe@example.com"
copyright = "Copyright 2020 Jane Doe"
description = "A short summary on this project"
homepage = "https://stdlib.fortran-lang.org"
categories = ["io"]
keywords = ["hdf5", "mpi"]

# Build Settings
[fortran]
implicit-typing = false
implicit-external = false
source-form = "free"

[build]
auto-executables = true
auto-examples = true
auto-tests = true
link = ["lapack", "blas"]
external-modules = ["netcdf", "h5lt"]

# Library Configuration
[library]
source-dir = "lib"
include-dir = "inc"
type = ["shared", "static"]

# Executable Targets
[[executable]]
name = "app-name"
source-dir = "prog"
main = "program.f90"

[[executable]]
name = "app-tool"
link = "z"
[executable.dependencies]
helloff = { git = "https://gitlab.com/everythingfunctional/helloff.git" }

# Example Targets
[[example]]
name = "demo-app"
source-dir = "demo"
main = "program.f90"

# Test Targets
[[test]]
name = "test-name"
source-dir = "testing"
main = "tester.F90"

[[test]]
name = "tester"
link = ["lapack", "blas"]
[test.dependencies]
helloff = { git = "https://gitlab.com/everythingfunctional/helloff.git" }

# Dependencies (Root Level)
[dependencies]
toml-f = { git = "https://github.com/toml-f/toml-f" }
toml-f-branch = { git = "https://github.com/toml-f/toml-f", branch = "main" }
toml-f-tag = { git = "https://github.com/toml-f/toml-f", tag = "v0.2.1" }
toml-f-rev = { git = "https://github.com/toml-f/toml-f", rev = "2f5eaba" }
my-utils = { path = "utils" }
my-package.namespace = "my-namespace"
example-package.namespace = "example-namespace"
example-package.v = "1.0.0"

# Development Dependencies
[dev-dependencies]
fftpack = { git = "https://github.com/fortran-lang/fftpack.git", preprocess.cpp.macros = ["REAL32"] }

# Installation Configuration
[install]
library = true
module-dir = "fortran/modules"

# Preprocessor Configuration
[preprocess]
[preprocess.cpp]
suffixes = ["F90", "f90"]
directories = ["src/feature1", "src/models"]
macros = ["FOO=2", "BAR=4", "VERSION={version}"]

[preprocess.fypp]

# Extra Data for Third-Party Tools
[extra.fpm.my-plugin]
setting = "value"
```

### 1.6 Dependencies (git and otherwise)

The `[dependencies]` section supports four ways to reference a package, all shown verbatim
above (manifest reference, accessed 2026-07-26):

- **Git, default branch:** `toml-f = { git = "https://github.com/toml-f/toml-f" }`
- **Git, pinned to a branch:** `{ git = "…", branch = "main" }`
- **Git, pinned to a tag:** `{ git = "…", tag = "v0.2.1" }`
- **Git, pinned to a commit:** `{ git = "…", rev = "2f5eaba" }`
- **Local path:** `my-utils = { path = "utils" }`
- **Registry / namespace:** `my-package.namespace = "my-namespace"` with an optional
  `example-package.v = "1.0.0"` version constraint.

For reproducible builds, prefer `tag =` or `rev =` over a bare `git =` or `branch =` — the
latter two float and can silently change what you build.

---

<!-- §§ STDLIB -->
## 2. Fortran stdlib — the Standard Library

### 2.1 What it is and current version

The Fortran Standard Library (stdlib) is "a community driven and agreed upon _de facto_
'standard' library for Fortran" that fills the gap left by the ISO Fortran Standard, which
defines no official standard library
([stdlib.fortran-lang.org](https://stdlib.fortran-lang.org/), accessed 2026-07-26).

Its scope spans (same source, accessed 2026-07-26):
- **Utilities:** containers, strings, files, OS integration, testing, logging.
- **Algorithms:** searching, sorting, merging.
- **Mathematics:** linear algebra, sparse matrices, special functions, FFT, random numbers,
  statistics, ODEs, numerical integration, optimization.

**Current release: v0.8.1, released 2025-01-26** (v0.8.0 on 2025-01-23 was the major
feature drop: 80-bit extended precision in `linalg`, matrix inverse/Cholesky/QR, sparse
algebra with an object-oriented API, and iterative solvers)
([github.com/fortran-lang/stdlib/releases](https://github.com/fortran-lang/stdlib/releases),
accessed 2026-07-26). Several modules carry an **experimental** status flag (noted per-module
in §2.2) — their APIs may change.

Documentation home: <https://stdlib.fortran-lang.org/> (accessed 2026-07-26).

### 2.2 Full module list

Reproduced from the stdlib module index
([stdlib.fortran-lang.org/lists/modules.html](https://stdlib.fortran-lang.org/lists/modules.html),
accessed 2026-07-26). This list includes both top-level modules and their submodules; the
top-level modules are the ones a user typically `use`s. Descriptions in quotes are verbatim
from the index; unquoted descriptions summarize the index entry.

**Terminal / text output**
- `stdlib_ansi` — "Terminal color and style escape sequences"

**Arrays, containers, and data structures**
- `stdlib_array` — "Module for index manipulation and general array handling"
- `stdlib_bitsets` — bitsets up to `huge(0_int32)` using 64-bit integers, two's complement
  (with submodules `stdlib_bitsets_64`, `stdlib_bitsets_large`)
- `stdlib_hashmaps` — "Hash map implementations with empirical SLOT expansion code"
  (with `stdlib_hashmap_chaining`, `stdlib_hashmap_open`, `stdlib_hashmap_wrappers`)
- `stdlib_hash_32bit` / `stdlib_hash_64bit` — 32-/64-bit hash functions (FNV, waterhouse,
  pengyhash, SpookyHash v2 submodules)

**Strings and character handling**
- `stdlib_ascii` — "Procedures for handling and manipulating intrinsic character variables and constants"
- `stdlib_string_type` — "String type for arbitrary character sequences"
- `stdlib_stringlist_type` — string list type
- `stdlib_strings` — "Basic string handling routines"
- `stdlib_str2num` — "Character to numerical type conversion"

**I/O and files**
- `stdlib_io` — "Support for file handling"
- `stdlib_io_mm` — "Matrix Market format for sparse and dense matrices"
- `stdlib_io_npy` — "NumPy npy format implementation" (load/save submodules)

**Kinds, constants, math utilities**
- `stdlib_kinds` — kind parameter specifications
- `stdlib_constants` — physical and mathematical constants
- `stdlib_codata` — "Codata Constants - Autogenerated" (with `stdlib_codata_type`)
- `stdlib_math` — mathematical utility functions (submodules include `all_close`, `arange`,
  `diff`, `is_close`, `linspace`, `logspace`, `meshgrid`)
- `stdlib_intrinsics` — "Alternative implementations offering faster and/or more accurate
  evaluation" of intrinsics (dot_product, matmul, sum submodules)
- `stdlib_optval` — "Generic function for fallback values on optional arguments"

**Linear algebra** (the largest area; wraps BLAS/LAPACK and adds high-level drivers)
- `stdlib_linalg` — "Support for various linear algebra procedures"; high-level submodules
  include: `cross_product`, `diag`, `kronecker`, `outer_product`, `cholesky`,
  `determinant` ("Determinant of a rectangular matrix"),
  `eigenvalues` ("Compute eigenvalues and eigenvectors"),
  `inverse` ("Compute inverse of a square matrix"),
  `least_squares` ("Least-squares solution to Ax=b"),
  `matrix_functions`, `norms`,
  `pseudoinverse` ("Compute the (Moore-Penrose) pseudo-inverse of a matrix"),
  `qr`, `schur`, `solve` ("Solve linear system Ax=b"), `svd` ("Singular-Value Decomposition")
- `stdlib_linalg_iterative_solvers` — "Iterative solvers with experimental status"
  (BiCGSTAB, CG, GMRES, PCG submodules)
- `stdlib_linalg_state` — "State/error handling derived type with pure procedures" (experimental)
- `stdlib_blas` — BLAS wrapper (levels 1/2/3 submodules)
- `stdlib_lapack_base` and the extensive `stdlib_lapack_*` submodule family — LAPACK routines
  (eigenvalue/SVD/least-squares drivers, factorizations, solvers, etc.)

**Sparse matrices**
- `stdlib_sparse` — sparse matrix operations (constants, conversion, kinds, operators, and
  SpMV submodules for COO/CSC/CSR/ELL/SELL-C formats; several marked experimental)
- `stdlib_specialmatrices` — "Structured matrices for PDE discretization and signal
  processing" (symmetric/general tridiagonal submodules)

**Special functions and quadrature**
- `stdlib_specialfunctions` — special mathematical functions (legendre, activations, gamma submodules)
- `stdlib_quadrature` — numerical integration (gauss, simps, trapz submodules)

**Statistics and random numbers**
- `stdlib_stats` — "Statistical methods including descriptive statistics" (mean, median,
  moment, var, cov, corr, pca submodules)
- `stdlib_stats_distribution_*` — beta, exponential, gamma, normal, uniform distributions
- `stdlib_random` — random number generation utilities

**Sorting, selection, searching**
- `stdlib_sorting` — "Overloaded sorting subroutines for integer, real, character, and string
  arrays" (ord_sort, radix_sort, sort, sort_adjoint submodules)
- `stdlib_selection` — "Find k-th smallest value or its index"

**Date/time, logging, errors, versioning**
- `stdlib_datetime` — "Date, time, and time interval handling for Fortran"
- `stdlib_logger` — "Logging information and error reporting derived type and procedures"
- `stdlib_error` — "Support for catching and handling errors" (experimental)
- `stdlib_version` — "Version information on stdlib"

> The index also lists many `*_comp`, `*_aux`, and numbered submodules (e.g.
> `stdlib_lapack_eigv_gen2`). These are implementation submodules of the top-level modules
> above and are generally not `use`d directly.

### 2.3 Using stdlib with fpm

Quoted from stdlib usage docs
([github.com/fortran-lang/stdlib](https://github.com/fortran-lang/stdlib), accessed 2026-07-26):

- **fpm 0.9.0 and later (metapackage — recommended):** add to `fpm.toml`:
  ```toml
  [dependencies]
  stdlib = "*"
  ```
  "Fpm 0.9.0 and later implements stdlib as a _metapackage_." The metapackage form lets fpm
  fetch and build a stdlib configured for your project automatically. (Since the current fpm
  is v0.13.0 — see §1.1 — the metapackage form is the default choice today.)

- **fpm 0.8.x and earlier (git branch):** add to `fpm.toml`:
  ```toml
  [dependencies]
  stdlib = { git="https://github.com/fortran-lang/stdlib", branch="stdlib-fpm" }
  ```
  The `stdlib-fpm` branch is a flattened, fpm-consumable mirror of the stdlib sources
  (the main branch uses a Python/fypp preprocessing build step that plain fpm cannot run
  directly).

### 2.4 Using stdlib with CMake

Quoted from stdlib usage docs
([github.com/fortran-lang/stdlib](https://github.com/fortran-lang/stdlib), accessed 2026-07-26):

Basic configure + build:
```bash
cmake -B build
cmake --build build
```

Test and install:
```bash
cmake --build build --target test
cmake --install build
```

Key configuration options: `-G Ninja` (build backend), `-DCMAKE_INSTALL_PREFIX` (install
location), `-DCMAKE_MAXIMUM_RANK` (maximum array rank; default 4, maximum 15). Full example
with custom settings:
```bash
export FFLAGS="-O3"
cmake -B build -G Ninja -DCMAKE_MAXIMUM_RANK:String=7 \
  -DCMAKE_INSTALL_PREFIX=$HOME/.local \
  -DCMAKE_VERBOSE_MAKEFILE=On -DCMAKE_BUILD_TYPE=NoConfig
```

---

<!-- §§ COMPILER_LANDSCAPE -->
## 3. Compiler landscape

Four compilers matter for most modern-Fortran work. Status and standards claims below are
from the fortran-lang compilers page ([fortran-lang.org/compilers](https://fortran-lang.org/compilers/),
accessed 2026-07-26) and the vendor sources cited inline.

### 3.1 GNU Fortran (`gfortran`) — the stable default

- **Status:** "mature free and open source compiler, part of the GNU Compiler Collection"
  ([fortran-lang.org/compilers](https://fortran-lang.org/compilers/), accessed 2026-07-26).
- **Standards:** Fortran 2018 support; full coarray (multi-image) support requires the
  OpenCoarrays wrapper (same source, accessed 2026-07-26). gfortran also accepts
  `-std=f2023` (see §4.1), reflecting ongoing Fortran 2023 work.
- **Why it's the default:** ubiquitous, free, packaged everywhere, and the most forgiving to
  install. Unless a task specifically needs another compiler, an assistant should assume
  gfortran.

### 3.2 Intel `ifx` (current) — `ifort` is discontinued

- **`ifx` status:** the current, actively developed LLVM-based Intel Fortran compiler with
  "full Fortran 2018 support" plus "a large subset of Fortran 2023 features" in the 2025
  release ([fortran-lang.org/compilers](https://fortran-lang.org/compilers/) and Intel 2025
  release-notes summary, accessed 2026-07-26). The 2025.1 toolkit ships `ifx` as the default
  Fortran compiler (Intel documentation summary, accessed 2026-07-26).
- **`ifort` (Intel Fortran Compiler Classic) is DISCONTINUED.** Intel's deprecation notice
  states ifort was deprecated and discontinued, and that **"The removal of ifort from the
  packages is planned for the version 2025.0 oneAPI packages … Intel® Fortran Compiler
  Classic (ifort) is now discontinued in oneAPI 2025 release."** Intel recommends transitioning
  to `ifx` "for continued Windows* and Linux* support, new language support, new language
  features, and optimizations." The stated plan: version 2025.0 and all future releases no
  longer ship ifort, and in late 2025 with the version 2026 release all ifort downloads are
  removed from Intel's registration center and repositories
  ([Intel Community deprecation notice](https://community.intel.com/t5/Intel-Fortran-Compiler/DEPRECATION-NOTICE-Intel-Fortran-Compiler-Classic-ifort/td-p/1545789)
  and [Intel Fortran Compiler oneAPI Release Notes 2025](https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html),
  accessed 2026-07-26).
  - **Implication for assistants:** do not emit `ifort` invocations for new work. Use `ifx`.
    The `-assume byterecl` and most `-check`/`-warn`/`-stand` flags carry over from ifort to
    ifx, but there is an official
    [Porting Guide for ifort Users to ifx](https://www.intel.com/content/www/us/en/developer/articles/guide/porting-guide-for-ifort-to-ifx.html)
    (accessed 2026-07-26) for behavioral differences.

  > Note: The primary Intel 2025 release-notes URL
  > (`https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html`)
  > returned **HTTP 403 Forbidden** to direct fetch on 2026-07-26 (Intel blocks the fetch
  > user-agent). The ifort-discontinuation wording above is quoted from Intel's own community
  > deprecation notice, which mirrors the release-notes text; the release-notes URL is retained
  > as a citation because it is the canonical source and is human-reachable in a browser.
  > A 2026-07-26 review pass re-attempted the Intel pages (browser user-agent, archive.org)
  > and they remained unreachable (HTTP 403); however, the discontinuation was independently
  > corroborated by a direct fetch of Intel's announcement as reposted on Fortran Discourse:
  > ifort's final build is version 2021.13 (shipped in HPC Toolkit 2024.2.0), and "Towards
  > the end of 2024 Intel will no longer provide ifort in our packages, starting with the
  > release version 2025.0"
  > ([fortran-lang.discourse.group/t/a-historic-moment-for-the-intel-fortran-compiler-classic-ifort/8350](https://fortran-lang.discourse.group/t/a-historic-moment-for-the-intel-fortran-compiler-classic-ifort/8350),
  > accessed 2026-07-26).

### 3.3 LLVM Flang (`flang` / `flang-new`) — newly usable

- **Status:** "LLVM's Fortran frontend," under active development; "is capable of generating
  executables for a number of examples, some functionality is still missing"
  ([flang.llvm.org/docs](https://flang.llvm.org/docs/), accessed 2026-07-26). This is the
  modern C++/MLIR-based Flang, distinct from the older "Classic Flang."
- **Standards:** Fortran 2018 is nearly complete — "All features except those listed …
  are supported. Almost all of the unsupported features are related to the multi-image
  execution" (i.e. coarrays). Fortran 2023 support is partial; "The two major missing features
  in Flang at present are coarrays and parameterized derived types (PDTs) with length type
  parameters"
  ([flang.llvm.org/docs/FortranStandardsSupport.html](https://flang.llvm.org/docs/FortranStandardsSupport.html),
  accessed 2026-07-26).
- **Version:** the Flang documentation is labeled "Flang 24 (In-Progress)"
  ([flang.llvm.org/docs](https://flang.llvm.org/docs/), accessed 2026-07-26); Flang's major
  version tracks the LLVM release train. The task framing of Flang becoming "newly usable
  around LLVM 20" is consistent with the docs but the precise "usable from LLVM 20" cutoff is
  **unverified — model knowledge**; treat Flang as usable-but-incomplete and verify against
  the LLVM release you actually have installed.
- **Binary name:** historically `flang-new`; the project has been renaming the driver to
  `flang`. Which name your install exposes depends on the LLVM version — check both.

### 3.4 LFortran — alpha, interactive/REPL niche

- **Status:** "modern, interactive, LLVM-based" compiler that can "execute user's code
  interactively … as well as compile to binaries"
  ([fortran-lang.org/compilers](https://fortran-lang.org/compilers/) and
  [lfortran.org](https://lfortran.org/), accessed 2026-07-26). It is explicitly **alpha**:
  the project states it is "expected to not work on third-party codes." The latest stable
  release noted was v0.49.0 (alpha), 2025-03-15; the beta bar is "reliably compile 10
  production-grade third-party packages," and as of early 2026 the counter stood at 9/10,
  with LFortran having compiled fpm itself (issue closed 2026-02-07)
  ([LFortran blog](https://lfortran.org/blog/2026/02/lfortran-compiles-fpm/), accessed 2026-07-26).
- **When to reach for it:** interactive exploration (Jupyter-style / REPL) and browser-based
  (WASM) demos. **Not** for building production code today.

### 3.5 Others (brief)

The fortran-lang compilers page (accessed 2026-07-26) also lists NAG (7.0, strong
standards/diagnostics, commercial), NVIDIA HPC SDK `nvfortran` (former PGI; GPU via
OpenACC/CUDA; free), HPE/Cray CCE, IBM XL Fortran, AMD AOCC (`amdflang`), Arm Compiler for
Linux, Oracle Developer Studio, and Silverfrost FTN95 (Fortran 95 + some 2003/2008, free
personal edition). These are out of scope for day-to-day assistant work but worth knowing
exist for HPC/vendor-specific tasks.

---

<!-- §§ FLAGS_TABLE -->
## 4. Per-compiler strictness & debugging flags

Use these when compiling for correctness (development, CI, the compile-verify loop in §5) —
**not** for optimized release builds. Every flag spelling below was verified against the
cited official documentation on 2026-07-26; do not substitute guessed spellings across
compilers, because they differ (e.g. gfortran's `-fcheck=all` is `-check all` under ifx and
has no direct single-flag equivalent under Flang).

### 4.1 gfortran

All from the GNU Fortran manual (accessed 2026-07-26):
[Code-Gen (check) options](https://gcc.gnu.org/onlinedocs/gfortran/Code-Gen-Options.html),
[Error-and-Warning options](https://gcc.gnu.org/onlinedocs/gfortran/Error-and-Warning-Options.html),
[Debugging options](https://gcc.gnu.org/onlinedocs/gfortran/Debugging-Options.html).

| Flag | Effect (verbatim / summarized from the manual) |
|------|-----------------------------------------------|
| `-std=f2018` | Conform-check against Fortran 2018. Accepted `-std=` keywords: `f95`, `f2003`, `f2008`, `f2018`, `f2023`, plus `gnu` and `legacy`. |
| `-Wall` | "commonly used warning options … that we recommend avoiding and that we believe are easy to avoid" (includes `-Waliasing`, `-Wconversion`, `-Wsurprising`, `-Wtabs`, `-Wunused`, and more). |
| `-Wextra` | "some warning options for usages of language features that may be problematic" (includes `-Wcompare-reals`, `-Wunused-parameter`, `-Wdo-subscript`, `-Wunused-intent-out`). |
| `-Wpedantic` (`-pedantic`) | "Issue warnings for uses of extensions to Fortran." |
| `-fcheck=all` | "Enable all run-time test of -fcheck." Keyword list: `all`, `array-temps` (warns when a temp array had to be made for an actual argument), `bits` (bit-intrinsic argument checks), `bounds` (array subscript + declared min/max checks), `do` (invalid loop-variable modification), `mem` (memory-allocation checks), `pointer` (pointer/allocatable checks), `recursion` (unmarked recursive procedure checks). |
| `-fbounds-check` | "Deprecated alias for -fcheck=bounds." Prefer `-fcheck=bounds` (or `-fcheck=all`). |
| `-fbacktrace` | Emit a runtime backtrace on a serious error / deadly signal. (`-fno-backtrace` disables it; backtrace-on-error is the runtime library's default behavior the manual describes.) |
| `-ffpe-trap=<list>` | Trap floating-point exceptions. Supported exceptions: `invalid` (e.g. `SQRT(-1.0)`), `zero` (division by zero), `overflow`, `underflow`, `inexact` (precision loss), `denormal`. A common development choice is `-ffpe-trap=invalid,zero,overflow`. |
| `-ffpe-summary=<list>` | Print the status of listed FP exceptions to `ERROR_UNIT` at `STOP`/`ERROR STOP`; accepts `none`, `all`, or a comma list. |
| `-g` | Generate debug info for a debugger (`-ggdb3` for richer GDB info). |

**Recommended gfortran strict set (development):**
`gfortran -std=f2018 -Wall -Wextra -fcheck=all -fbacktrace -ffpe-trap=invalid,zero,overflow -g`

> Caution on `-ffpe-trap=underflow,inexact`: `inexact` and `underflow` fire on almost all
> normal floating-point code, so trapping them is usually counterproductive. Trap
> `invalid,zero,overflow` for real-world debugging.

### 4.2 Intel `ifx`

Flag spellings from the Intel Fortran Compiler Developer Guide and Reference and
community/Intel documentation (accessed 2026-07-26). Note these are the classic Intel
short-form flags (Linux/macOS syntax; on Windows they take a leading `/` instead of `-`,
e.g. `/check:all`). ifx inherits the ifort flag vocabulary.

| Flag | Effect | gfortran analogue |
|------|--------|-------------------|
| `-stand f18` | Standards-conformance diagnostics against Fortran 2018 (`-stand` also accepts `f03`, `f08`, `f18`, and newer keywords such as `f23` on recent releases; the exact accepted set is version-dependent — verify against your installed Developer Guide). | `-std=f2018` |
| `-warn all` | "Check for compile-time warnings." | `-Wall -Wextra` |
| `-check all` | Enable all run-time checks (bounds, uninitialized, pointer, etc.). | `-fcheck=all` |
| `-traceback` | "produce a stack dump that is much more informative than what is ordinarily produced by the Fortran runtime" on a crash. | `-fbacktrace` |
| `-fpe-all=0` | Trap all floating-point exceptions and abort when one occurs (level `0` = strictest). Applies to all routines. | `-ffpe-trap=…` |
| `-fpe0` | Like `-fpe-all=0` but applies to the main program only; the common shorthand for "trap FP exceptions." | `-ffpe-trap=invalid,zero,overflow` |
| `-g` | Generate full debug information. | `-g` |
| `-O0` | Disable optimization (pair with the above for clean debugging). | `-O0` |

**Recommended ifx strict set (development):**
`ifx -stand f18 -warn all -check all -traceback -fpe0 -g -O0`

> The `-stand f18`, `-warn all`, `-check all`, `-traceback`, and `-fpe-all=0` spellings are
> confirmed by the Intel Developer Guide and Intel community documentation (accessed
> 2026-07-26). The precise set of `-stand` keywords accepted by the newest ifx (whether
> `f23` is spelled `f23` or otherwise) is **version-dependent — verify against your installed
> `ifx --help` or the matching Developer Guide** rather than assuming.

### 4.3 LLVM Flang (`flang` / `flang-new`)

From the Flang command-line argument reference and driver docs (accessed 2026-07-26):
[FlangCommandLineReference](https://flang.llvm.org/docs/FlangCommandLineReference.html),
[FlangDriver](https://flang.llvm.org/docs/FlangDriver.html).

| Flag | Effect (verbatim / summarized) | gfortran analogue |
|------|-------------------------------|-------------------|
| `-std=<arg>` (`--std=<arg>`) | "Language standard to compile for" (e.g. `-std=f2018`). Fortran 2023 (`-std=f2023`) support is in progress per LLVM RFC discussion. | `-std=f2018` |
| `-pedantic` (`--pedantic`) | "Warn on language extensions." | `-Wpedantic` |
| `-Werror` | Treat warnings as errors — "The only GCC/GFortran warning option currently supported is -Werror." | `-Werror` |
| `-g` | "Generate source-level debug information." | `-g` |

**Gap — no runtime bounds/FPE checking flag documented for Flang.** The Flang command-line
reference "does not include any explicit bounds checking flags like `-fcheck=bounds` or
similar runtime checking options" (accessed 2026-07-26). As of this file's date, LLVM Flang
has **no `-fcheck=all`-equivalent single flag** and no documented `-ffpe-trap`/`-fbacktrace`
equivalent in the reference. This is a real maturity gap: Flang catches fewer runtime errors
than gfortran or ifx, so **do not rely on Flang alone for correctness verification** — cross-
compile the same code with gfortran's `-fcheck=all` when possible.

**Recommended Flang strict set (development, given the gap):**
`flang -std=f2018 -pedantic -Werror -g`

> `-Wall`/`-Wextra` are **not** documented as supported by Flang (only `-Werror` among the
> GCC-style warning flags), so do not assume they behave as they do under gfortran.

### 4.4 Cross-compiler flag cheat sheet

| Intent | gfortran | ifx | Flang |
|--------|----------|-----|-------|
| Standards conformance (F2018) | `-std=f2018` | `-stand f18` | `-std=f2018` |
| Common warnings | `-Wall -Wextra` | `-warn all` | *(only `-Werror`; no `-Wall`)* |
| Warn on extensions | `-Wpedantic` | *(via `-stand`/`-warn`)* | `-pedantic` |
| Warnings → errors | `-Werror` | `-warn errors` | `-Werror` |
| All runtime checks | `-fcheck=all` | `-check all` | *(none documented)* |
| Array bounds only | `-fcheck=bounds` | `-check bounds` | *(none documented)* |
| Backtrace on crash | `-fbacktrace` | `-traceback` | *(none documented)* |
| Trap FP exceptions | `-ffpe-trap=invalid,zero,overflow` | `-fpe0` / `-fpe-all=0` | *(none documented)* |
| Debug info | `-g` | `-g` | `-g` |

> Empty cells for Flang reflect documented gaps as of 2026-07-26, not omissions in this
> table. The gfortran and ifx spellings are verified from official docs (§4.1, §4.2);
> `-warn errors` for ifx is the standard ifort/ifx spelling for warnings-as-errors and is
> **model knowledge — verify against `ifx --help`** if load-bearing.

---

<!-- §§ INTEROP_GOTCHA -->
## 5. Interop gotcha: unformatted `RECL` units differ between compilers

**The problem.** For `access='direct'` (and, by default, unformatted) files, the `RECL=`
value in an `OPEN` statement means **different things under gfortran vs. Intel Fortran**:

- **gfortran** measures `RECL` in **bytes**. "gfortran does not offer a way to have direct
  access with RECL in different units than bytes. The file storage unit is always 8 bits for
  (un)formatted files."
- **Intel Fortran (ifort/ifx)** by default measures `RECL` for unformatted data in **4-byte
  words (longwords)**, not bytes. "By default, ifort measures RECL in four-byte words, while
  gfortran uses byte units." (Formatted files use bytes on both.)

(Both statements from the comp.lang.fortran discussion and Intel community threads on RECL
units, accessed 2026-07-26 — see Sources.)

**Consequence.** The same source compiled with gfortran and with Intel will read/write
unformatted direct-access records of different physical sizes for the same `RECL` literal.
A file written by one is misread by the other; record-length mismatches and I/O errors follow.

**The fix.** Compile the Intel build with **`-assume byterecl`** (Windows:
`/assume:byterecl`), which makes ifx/ifort interpret `RECL` in bytes — matching gfortran and
the Fortran 2003 recommendation. "You can change ifort to use bytes units by using the
/assume:byterecl compiler option," and "Most compilers seem to be gravitating toward byte unit
measurement, and Fortran 2003 makes an explicit recommendation for byte units"
(Intel community + comp.lang.fortran, accessed 2026-07-26).

**Assistant rule.** Whenever you generate or debug Intel-compiled code that opens unformatted
or direct-access files — especially code meant to interoperate with gfortran-produced files or
to be portable — **add `-assume byterecl`** to the Intel compile line, or compute `RECL` with
the `INQUIRE(IOLENGTH=…)` intrinsic (which is portable and unit-agnostic) instead of hard-coding
a byte count.

---

<!-- §§ COMPILE_VERIFY_LOOP -->
## 6. Recommended compile-verify loop (for assistants)

Fortran is compiled and statically typed, and its compilers catch a large class of errors
that no amount of code-reading will. **An assistant must never declare Fortran work "done"
without actually compiling it under strict flags and, where a runnable target exists, running
it.** The loop:

1. **Build with strict flags.** Prefer gfortran for the correctness pass because it has the
   most complete runtime-checking flags (§4.1):
   ```bash
   gfortran -std=f2018 -Wall -Wextra -fcheck=all -fbacktrace \
            -ffpe-trap=invalid,zero,overflow -g -O0 -c *.f90
   ```
   Under fpm, wire the same flags into a debug profile / `--flag` and run `fpm build` and
   `fpm test` (§1.4). Under ifx use `-stand f18 -warn all -check all -traceback -fpe0`
   (§4.2). Flang's checking is weaker (§4.3), so do not treat a clean Flang build as
   sufficient verification.

2. **Treat warnings as errors during development.** Add `-Werror` (all three compilers
   support it) so `-Wall`/`-Wextra`/`-pedantic` findings block the "done" claim rather than
   scrolling past.

3. **Fix every diagnostic, then rebuild.** Re-run step 1 until the strict build is clean.
   Do not silence warnings by weakening flags.

4. **Run the program / tests.** A clean compile is necessary but not sufficient — run the
   executable (`fpm run`) and the test suite (`fpm test`) so that the runtime checks from
   `-fcheck=all` and `-ffpe-trap=…` actually fire on real data paths.

5. **Cross-compile when portability matters.** If the code targets more than one compiler,
   build it under at least gfortran **and** ifx; a second front-end catches nonconformant
   code the first accepts, and it surfaces the `RECL` interop gotcha (§5) before users do.

6. **Only then report done** — and state which compiler(s) and flags you verified against,
   so the reader can reproduce the result.

> Rationale: gfortran's `-fcheck=all` plus `-ffpe-trap` turns silent undefined behavior
> (out-of-bounds access, use of uninitialized memory, division by zero) into loud, located
> runtime aborts. Skipping this loop is the single most common way Fortran changes ship
> broken.

---

<!-- §§ SOURCES -->
## Sources

All URLs fetched or searched directly on **2026-07-26**.

**fpm**
- fpm home (overview, `fpm publish`, current version note): <https://fpm.fortran-lang.org/> — fetched OK.
- fpm manifest reference (verbatim `fpm.toml`, git-dependency syntax, all sections): <https://fpm.fortran-lang.org/spec/manifest.html> — fetched OK.
- fpm install page (all install methods, verbatim commands): <https://fpm.fortran-lang.org/install/index.html> — fetched OK.
- fpm releases (current v0.13.0 / 2026-02-17 and history): <https://github.com/fortran-lang/fpm/releases> — fetched OK.

**stdlib**
- stdlib home (purpose, scope): <https://stdlib.fortran-lang.org/> — fetched OK.
- stdlib module index (full module list): <https://stdlib.fortran-lang.org/lists/modules.html> — fetched OK.
- stdlib repo (fpm metapackage `stdlib = "*"`, `stdlib-fpm` branch, CMake commands): <https://github.com/fortran-lang/stdlib> — fetched OK.
- stdlib releases (current v0.8.1 / 2025-01-26 and history): <https://github.com/fortran-lang/stdlib/releases> — fetched OK.

**Compilers — landscape**
- fortran-lang compilers page (gfortran/ifx/ifort/Flang/LFortran + others, status/standards): <https://fortran-lang.org/compilers/> — fetched OK.

**gfortran flags**
- GNU Fortran check/code-gen options (`-fcheck=` keywords, `-fbounds-check`): <https://gcc.gnu.org/onlinedocs/gfortran/Code-Gen-Options.html> — fetched OK (after one 502 retry).
- GNU Fortran error/warning options (`-Wall`, `-Wextra`, `-Wpedantic`, `-std=` keywords): <https://gcc.gnu.org/onlinedocs/gfortran/Error-and-Warning-Options.html> — fetched OK.
- GNU Fortran debugging options (`-ffpe-trap`, `-ffpe-summary`, `-fno-backtrace`, `-g`): <https://gcc.gnu.org/onlinedocs/gfortran/Debugging-Options.html> — fetched OK.

**Intel ifx / ifort**
- Intel Fortran Developer Guide — Debugging (2025-0): <https://www.intel.com/content/www/us/en/docs/fortran-compiler/developer-guide-reference/2025-0/debugging.html> — referenced via search (Intel blocks direct fetch UA).
- Intel Fortran Developer Guide — Debugging and Optimizations (2024-2): <https://www.intel.com/content/www/us/en/docs/fortran-compiler/developer-guide-reference/2024-2/debugging-and-optimizations.html> — referenced via search.
- Intel Fortran Compiler oneAPI Release Notes 2025 (ifort discontinued in oneAPI 2025): <https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html> — **direct fetch returned HTTP 403**; content corroborated via Intel Community deprecation notice below (human-reachable in a browser).
- Intel Community — ifort deprecation/discontinuation notice (verbatim wording): <https://community.intel.com/t5/Intel-Fortran-Compiler/DEPRECATION-NOTICE-Intel-Fortran-Compiler-Classic-ifort/td-p/1545789> — via search.
- Intel — Porting Guide for ifort Users to ifx: <https://www.intel.com/content/www/us/en/developer/articles/guide/porting-guide-for-ifort-to-ifx.html> — via search.
- Fortran Discourse — "A Historic Moment for The Intel Fortran Compiler Classic (ifort)" (Intel announcement repost; ifort final build 2021.13, removal from 2025.0 packages): <https://fortran-lang.discourse.group/t/a-historic-moment-for-the-intel-fortran-compiler-classic-ifort/8350> — fetched OK (added in 2026-07-26 review pass).
- ifx runtime-check flags (`-check all`, `-traceback`, `-fpe-all=0`, `-warn all`): gjbex defensive-programming notes <https://gjbex.github.io/Defensive_programming_and_debugging/BugsAtRuntime/Verification/Compilers/ifort_flags/> and Intel community threads — via search.
- Intel Fortran Developer Guide — Standards (`-stand`): <https://www.intel.com/content/www/us/en/docs/fortran-compiler/developer-guide-reference/2025-0/standards.html> — **direct fetch returned HTTP 403**; `-stand f18` spelling corroborated via search results.

**LLVM Flang**
- Flang docs index (status, "Flang 24 In-Progress"): <https://flang.llvm.org/docs/> — fetched OK.
- Flang Fortran Standards Support (F2018 near-complete, F2023 partial, coarray/PDT gaps): <https://flang.llvm.org/docs/FortranStandardsSupport.html> — fetched OK.
- Flang command-line argument reference (`-std`, `-pedantic`, `-Werror`, `-g`; no bounds-check flag): <https://flang.llvm.org/docs/FlangCommandLineReference.html> — fetched OK.
- Flang driver (`-Werror` as the only supported GCC warning flag): <https://flang.llvm.org/docs/FlangDriver.html> — via search.

**LFortran**
- LFortran home (interactive/LLVM, alpha): <https://lfortran.org/> — via search.
- LFortran blog — "LFortran compiles fpm" (beta counter 9/10, fpm compiled 2026-02-07): <https://lfortran.org/blog/2026/02/lfortran-compiles-fpm/> — via search.

**Unformatted RECL interop gotcha**
- comp.lang.fortran — "-assume byterecl in gfortran": <https://groups.google.com/g/comp.lang.fortran/c/sZ1Hgo6M8pw> — via search.
- Intel community — unformatted-file portability (RECL units, `-assume byterecl`): <https://community.intel.com/t5/Intel-Fortran-Compiler/Problems-writing-reading-unformatted-files-with-Intel-fortran/td-p/1160770> — via search.
- Intel OPEN: RECL Specifier reference: <http://ahamodel.uib.no/intel/GUID-C6A40AAC-81D8-4DD8-A792-62792B3AC213.html> — via search.

---

## Known gaps and unverified claims (maintainer log)

- **Intel release-notes and Standards pages return HTTP 403 to direct fetch** —
  re-confirmed in the 2026-07-26 review pass (browser user-agent and archive.org routes
  also failed; even Intel's community-forum pages now 403 to automated fetch). The ifort
  discontinuation is nonetheless corroborated by a directly-fetched Fortran Discourse
  repost of Intel's announcement (see §3.2 and Sources). All Intel
  release-notes / developer-guide claims (ifort discontinuation, `-stand f18`, ifx flags) are
  corroborated from Intel's community forum, the porting guide, and third-party flag
  references rather than a direct fetch of the canonical pages. The canonical URLs are cited
  and are reachable in a browser; re-verify there if a claim is load-bearing.
- **`ifx -stand` newest keyword (`f23` vs other spelling)** — version-dependent; verify against
  the installed `ifx --help`. Flagged inline in §4.2.
- **`ifx -warn errors`** (warnings-as-errors) — standard ifort/ifx spelling, marked *model
  knowledge* in §4.4; verify against `ifx --help`.
- **"Flang usable from ~LLVM 20"** — marked *unverified — model knowledge* in §3.3; the Flang
  docs confirm active-but-incomplete status but do not pin a specific "usable-from" LLVM
  version. The docs page self-labels "Flang 24 (In-Progress)."
- **`fpm new` exact emitted subfolders** (whether `example/` is scaffolded by default) —
  flagged in §1.3 as needing local verification against the installed fpm version.
- **stdlib per-module experimental status** — the module index marks several modules
  experimental; that set changes release to release, so treat the experimental annotations in
  §2.2 as a snapshot.
