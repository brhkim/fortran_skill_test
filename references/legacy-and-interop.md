# Legacy Fortran & C Interoperability

> **Reference file** for the `fortran` skill.
> **Created:** 2026-07-26
> **Sources verified as of:** 2026-07-26 (see the Sources section at the end).
> **Staleness note:** This file is the *most stable* reference in the skill because it
> documents settled language semantics (fixed-form source rules, implicit typing,
> the obsolescence timeline) that change only when a new Fortran standard is ratified
> — roughly every 5 years. The one part that *does* drift is the tools list
> (`findent`, `fprettify`, compiler flag spellings); re-verify those against the
> linked upstream repositories before relying on exact command syntax.

Citation convention used throughout: each substantive claim carries an inline
citation of the form `[URL, accessed 2026-07-26]`. Assertions that could not be
grounded in a fetched page are explicitly flagged `unverified — model knowledge`.
Full URLs are consolidated in **Sources** at the end.

---

## §§ CARDINAL_RULE

**When maintaining a legacy file, match its existing style. Do not half-modernize.**

A file that mixes fixed-form FORTRAN 77 with a few free-form lines, or that has
`implicit none` bolted onto three subroutines out of forty, is *harder* to read and
maintain than either a consistently old file or a consistently modern one. A
half-converted file also breaks compilation in subtle ways: fixed-form and free-form
cannot coexist in the same file, and a compiler invoked with `-ffixed-form` will
silently misread free-form continuations.

The Fortran Wiki's modernization guidance stresses that modernization should be
*pragmatic rather than pedantic*, observing that "code such as Lapack and many other
scientific codes were written to work on limited computers, and 'niceties of
standards was less of an issue.'" [https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]

Practical decision rule:
- **Bug fix or small change to a legacy file** → stay in the file's existing form and
  style. Change the minimum.
- **Deliberate modernization pass** → convert the *whole* file (or a whole,
  self-contained module) in one commit, then verify it still compiles and passes tests
  before moving on.

---

## §§ FIXED_FORM_RULES

Fixed-form (also called "fixed format") source is the layout inherited from punched
cards. It is the default the compiler assumes for `.f`, `.for`, and `.f77` file
extensions. The `.f` → fixed-form mapping is documented on the fortran-lang gotchas
page, which states that the `.f` extension indicates legacy fixed source form while
`.f90` denotes free form [https://fortran-lang.org/learn/quickstart/gotchas/, accessed
2026-07-26 — verified in the 2026-07-26 review pass]; the `.for`/`.f77` variants are
common compiler convention (`unverified — model knowledge` for those two extensions).

The West Virginia University HPC "Best Practices in Modern Fortran" course describes
the historical column layout precisely:

> "Back to the first versions of Fortran, the first 5 columns of characters were
> dedicated as label fields. A C in column 1 means that the line was treated as a
> comment and any character in column 6 means that the line was a continuation of the
> previous statement."
> [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

### Column layout table

| Columns | Purpose |
|---------|---------|
| 1–5     | Statement label (a number, referenced by `GOTO`, `DO`, `FORMAT`, etc.), or a comment marker in column 1 |
| 1       | `C`, `c`, or `*` here marks the **entire line** as a comment |
| 6       | Any non-blank, non-zero character here marks the line as a **continuation** of the previous statement |
| 7–72    | The statement body |
| 73–onward | **Ignored** (historically the card sequence number) |

Grounding for the column-1 comment marker and column-6 continuation: the WVU HPC page
quoted above. The 72-column cutoff is the gfortran default: `-ffixed-line-length-n`
"Set column after which characters are ignored in typical fixed-form lines in the
source file," with popular values 72 (default), 80, and 132.
[https://gcc.gnu.org/onlinedocs/gfortran/Fortran-Dialect-Options.html, accessed 2026-07-26]

### Reading fixed-form safely — gotchas

- **Blanks are insignificant** in fixed form. `DO 10 I = 1,100` and `DO10I=1,100`
  are identical; the infamous `DO 10 I = 1.100` (a period instead of a comma) is a
  legal *assignment* to a variable named `DO10I`, not a loop. `unverified — model knowledge`
  (classic language semantics; not on a page fetched for this file — verify against a
  standard reference before relying on it in an edit).
- **A `D` or `d` in column 1** can mean a debug line if the code was compiled with
  `-fd-lines-as-comments`, in which case those lines "are treated as comment lines."
  [https://gcc.gnu.org/onlinedocs/gfortran/Fortran-Dialect-Options.html, accessed 2026-07-26]
  Do not assume `D`-lines are dead code; check the build flags.
- **Continuation lines** carry a marker in column 6. When editing, preserve the
  column alignment exactly — inserting or deleting leading characters shifts the body
  out of columns 7–72 and changes meaning.

### The modern contrast (free form)

The same WVU HPC page describes free form (introduced in Fortran 90):

> "Today those restrictions are no longer needed. Statements can start in the first
> column. An exclamation mark starts a comment and comments could be inserted any
> place in the line, except literal strings. Blanks help the readability of the code."
> [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

Free form uses a trailing `&` for line continuation (in place of the column-6 marker)
and `!` for comments. Free form is the default for `.f90`/`.f95`/later extensions —
the `.f90` = free-form mapping is confirmed by the fortran-lang gotchas page
[https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26]; the
later-extension variants (`.f95`, `.f03`, `.f08`) are common compiler convention
(`unverified — model knowledge` for those).

---

## §§ IMPLICIT_TYPING

By default (with no `implicit none`), Fortran assigns a type to any undeclared
variable based on its first letter. The WVU HPC page states the rule:

> "By default all variables starting with `i`, `j`, `k`, `l`, `m` and `n` are
> integers. All others are real. Despite of this being usually true for small codes,
> it is better to turn those defaults off with `implicit none`."
> [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

Mnemonic: variables beginning **I–N** are `INTEGER` by default (the range spells the
first two letters of *IN*teger); everything else is `REAL`.

### Why this matters when maintaining legacy code

- A typo in a variable name does **not** raise an error under implicit typing — it
  silently creates a new variable. This is the single most common source of latent
  bugs in old Fortran.
- Before you can trust any edit, determine whether the routine has `implicit none`.
  If it does not, every name is potentially load-bearing and mistyping is silent.
- Legacy code sometimes relied on the I–N convention deliberately (loop counters
  named `I, J, K, N`). Preserve that when matching style.

### Adding `implicit none` (a modernization step, not a maintenance step)

Adding `implicit none` to a routine forces every variable to be declared explicitly.
This is the highest-value modernization move because it surfaces typos and undeclared
variables at compile time — but it is a *whole-routine* change: you must then declare
every variable the routine uses, or it will not compile.

---

## §§ COMMON_BLOCKS

A `COMMON` block is FORTRAN's original mechanism for sharing storage (global
variables) across program units, predating modules. `COMMON` was introduced in
FORTRAN II and was declared **obsolescent in Fortran 2018**, alongside `EQUIVALENCE`,
`BLOCK DATA`, `FORALL`, and labeled `DO`.
[https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]

Typical form:

```fortran
      COMMON /GRID/ NX, NY, DX, DY
```

The same `COMMON /GRID/ ...` declaration must appear in every routine that touches
those variables, and the declarations must agree in type and order. `unverified —
model knowledge` (well-known semantics; not directly on a fetched page).

### Hazards when maintaining COMMON

- **Positional matching, not by name.** Storage is associated by *position* within the
  block, not by variable name. Two routines can name the same slot differently. If
  the declarations disagree in size or type across routines, you get silent memory
  corruption.
- **Blank COMMON vs named COMMON.** `COMMON // A, B` (blank) has different
  initialization and sizing rules than named `COMMON /NAME/`. `unverified — model
  knowledge`.
- **`SAVE` semantics.** Whether COMMON variables persist between calls has historically
  been compiler-dependent; legacy code may assume persistence that modern compilers do
  not guarantee without `SAVE`. `unverified — model knowledge`.

### Modernization path

Replace a `COMMON` block with a **module** containing the same variables, then `use`
the module wherever the block was declared. Modules give named (not positional)
association, type checking, and controlled visibility. The WVU HPC page notes that in
a module, "By default all variables, subroutines and functions inside a module are
visible, but restrictions can be made using the `private` statement."
[https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

---

## §§ EQUIVALENCE

`EQUIVALENCE` forces two or more variables (or arrays) to share the same storage,
aliasing them. It dates to the original IBM 704 FORTRAN and was declared
**obsolescent in Fortran 2018**.
[https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]

Historically it was used to save memory (reuse a scratch array under two names) or to
reinterpret bytes (view a `REAL` array as `INTEGER`). Both uses are hazardous:

- Memory reuse via `EQUIVALENCE` obscures data flow — a write through one name
  silently changes the other.
- Type-punning via `EQUIVALENCE` has undefined behavior in the type system.

**Modernization:** For scratch-memory reuse, use separate `allocatable` arrays. For
deliberate type reinterpretation, use the `transfer` intrinsic or `iso_c_binding`
pointer conversion instead. `unverified — model knowledge` (the *replacement*
recommendation is standard practice but not on a page fetched for this file).

---

## §§ ARITHMETIC_IF

The arithmetic `IF` is a three-way branch on the sign of an expression:

```fortran
      IF (EXPR) L1, L2, L3
```

Control transfers to label `L1` if `EXPR < 0`, `L2` if `EXPR == 0`, `L3` if
`EXPR > 0`. It was introduced in the original IBM 704 FORTRAN, and was **deleted in
Fortran 2018** (having been obsolescent before that).
[https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]

The Fortran Wiki modernization page recommends that "Arithmetic IF statements should
become `SELECT CASE` blocks or `IF/ELSEIF/ELSE/ENDIF` structures."
[https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]

Modern equivalent:

```fortran
      if (expr < 0) then
         ! ... was L1
      else if (expr == 0) then
         ! ... was L2
      else
         ! ... was L3
      end if
```

When *reading* old code, trace all three labels carefully — the same label may be
the target of more than one arm, which collapses cases in the modern rewrite.

---

## §§ LABELED_DO_CONTINUE

Old-style `DO` loops terminate on a numbered label rather than on `END DO`:

```fortran
      DO 100 I = 1, N
         X(I) = 0.0
  100 CONTINUE
```

The `CONTINUE` statement is a no-op that exists only to carry the terminating label.
Two legacy hazards:

- **Shared DO termination:** several nested `DO` statements terminating on the *same*
  label. This was declared obsolescent; the Flang feature history lists the preferred
  alternative as "an END DO or CONTINUE statement for each DO statement," and the WVU
  HPC page notes the modern `end do` form "is clearer for the user when the loops grow
  or became nested."
  [https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]
  [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]
- **Termination on an executable statement** (not `CONTINUE`) — e.g. the last line of
  the loop body carries the label. Editing that line risks moving the label.

Modern equivalent uses `end do` and drops the label entirely. Modern loop control uses
`exit` and `cycle` rather than `GOTO`.
[https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

`labeled DO` was itself declared obsolescent in Fortran 2018.
[https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]

---

## §§ ENTRY

An `ENTRY` statement defines an additional entry point inside a subroutine or
function, so one body of code can be called under several names with different dummy
argument lists. It was introduced in FORTRAN IV and declared **obsolescent in Fortran
2008**.
[https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]

`ENTRY` is confusing to maintain because local variables are shared across entry
points and the active dummy arguments differ per entry. **Modernization:** split into
separate procedures in a module, sharing state via module variables or an explicit
derived type rather than implicit shared locals. `unverified — model knowledge` (the
replacement approach is standard; not on a fetched page).

---

## §§ OBSOLESCENCE_TIMELINE

The Fortran standard distinguishes **obsolescent** features (still valid, but flagged
as redundant with a preferred alternative) from **deleted** features (no longer part
of the standard language, though most compilers still accept them via a legacy mode).

The table below is compiled from the Flang "Fortran feature history cheat sheet"
[https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]
supplemented by an Anthropic web search summary of authoritative sources for the five
features deleted in Fortran 90/95
[web search, accessed 2026-07-26 — see Sources; the underlying Intel "Deleted and
Obsolescent Language Features" page returned HTTP 403 and could not be fetched directly].

| Feature | Made obsolescent | Deleted | Preferred replacement |
|---------|------------------|---------|-----------------------|
| Fixed source form | Fortran 90 | (not deleted) | Free form |
| Arithmetic IF | Fortran 90 | **Fortran 2018** | `IF`/`ELSE IF` construct or `SELECT CASE` |
| Computed GO TO | Fortran 90 | (not deleted) | `SELECT CASE` |
| Alternate return | Fortran 90 | (not deleted) | Status argument + `SELECT CASE` |
| Statement functions | Fortran 90 | (not deleted) | Internal procedures |
| `CHARACTER*x` length form | Fortran 90 | (not deleted) | `character(len=x)` |
| Shared DO termination / non-block `DO` | Fortran 90 | **Fortran 2018** (non-block DO was deleted) | `END DO`/`CONTINUE` per `DO` |
| `PAUSE` statement | Fortran 90 | **Fortran 95** | Ordinary I/O + read to resume |
| `ASSIGN` / assigned GO TO / assigned FORMAT | Fortran 90 | **Fortran 95** | `SELECT CASE`; character-variable format |
| Floating-point DO index | Fortran 90 | **Fortran 95** | Integer index |
| `H` edit descriptor (Hollerith editing in FORMAT) | Fortran 90 | **Fortran 95** | Character constants |
| Branch to `END IF` from outside its block | Fortran 90 | **Fortran 95** | Restructure control flow |
| `ENTRY` statement | **Fortran 2008** | (not deleted) | Separate module procedures |
| `COMMON` | **Fortran 2018** | (not deleted) | Module variables |
| `EQUIVALENCE` | **Fortran 2018** | (not deleted) | `allocatable`, `transfer`, or pointers |
| `BLOCK DATA` | **Fortran 2018** | (not deleted) | Module with initialized variables |
| `FORALL` | **Fortran 2018** | (not deleted) | `DO CONCURRENT` |
| Labeled `DO` | **Fortran 2018** | (not deleted) | `DO ... END DO` |

> Audit note (2026-07-26 review pass): the Flang cheat sheet states that the features
> deleted in Fortran 95 "were obsolescent in F90," and that Fortran 2018 deleted
> "Arithmetic IF and non-block DO" while marking COMMON, EQUIVALENCE, BLOCK DATA,
> FORALL, and labeled DO obsolescent. The table above was corrected to match that page.
> [https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html, accessed 2026-07-26]

The five features deleted in Fortran 90→95 (per the web-search summary of
authoritative sources): "real and double precision DO loop index variables, branching
to END IF from an outer block, PAUSE statements, ASSIGN statements and assigned GO TO
statements and the use of an assigned integer as a FORMAT specification, and Hollerith
editing in FORMAT." [web search, accessed 2026-07-26]

> **Practical implication:** "Deleted" does not mean "won't compile." Most production
> compilers accept deleted features under a legacy dialect (see the compiler-flags
> section). But a strict-standard build (`-std=f2018`, etc.) will reject them, so
> knowing the timeline tells you which features force a legacy flag.

---

## §§ MODERNIZATION_STRATEGY

Recommended order of operations. Each step is independently verifiable (compile + test
after each), which is what makes incremental modernization safe. The overarching
Fortran Wiki guidance is to be pragmatic, not pedantic — replace obsolete or
non-standard constructs "while maintaining functionality."
[https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]

1. **Establish a test baseline first.** Capture the current program's outputs on
   representative inputs before touching anything. Every subsequent step is validated
   against this baseline. `unverified — model knowledge` (standard refactoring
   discipline).

2. **Add `implicit none` and explicit interfaces.** This is the highest-value step:
   it surfaces typos and type mismatches at compile time (see IMPLICIT_TYPING). Do it
   one routine at a time, declaring every variable as you go. The WVU HPC page
   endorses turning defaults "off with `implicit none`."
   [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

3. **Wrap procedures in modules.** Placing procedures in a `module` gives the caller an
   explicit interface (compile-time argument checking) automatically. Modules also
   provide the destination for the next step.
   [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

4. **Replace `COMMON` blocks with module variables.** Once a module exists, move each
   `COMMON` block's variables into module scope and replace the `COMMON` declarations
   with `use` statements (see COMMON_BLOCKS).

5. **Replace obsolescent control flow.** Arithmetic `IF` → `IF`/`SELECT CASE`; computed
   `GO TO` → `SELECT CASE`; labeled `DO`/`CONTINUE` → `DO ... END DO`. The Fortran
   Wiki page gives worked examples including "Computed GO TO → SELECT CASE."
   [https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]

6. **Modernize type declarations.** Replace non-standard `REAL*8` with a named kind:
   the Fortran Wiki page recommends
   ```fortran
   integer, parameter :: dp = selected_real_kind(15, 50)
   real(dp) :: variable
   ```
   [https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]
   Fortran 2008 also provides standard named kinds `real32`, `real64`, `real128`.
   [https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html, accessed 2026-07-26]

7. **Convert fixed form → free form (last).** Do this after the logical restructuring,
   because form conversion touches every line and would otherwise pollute the diffs of
   the earlier semantic steps. Fixed form is "Declared obsolescent in Fortran 95, but
   still part of Fortran 2008," so this step is optional.
   [https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]
   > **Source conflict (flagged, unresolved):** the Fortran Wiki quote above dates fixed
   > form's obsolescence to Fortran 95, but the Flang feature-history cheat sheet lists
   > "fixed form source" among the Fortran 90 obsolescent features (and the timeline
   > table above follows Flang). The two secondary sources genuinely disagree; consult
   > the standard's Annex B if the exact edition matters. Either way, fixed form is
   > obsolescent but not deleted.

Other transforms the Fortran Wiki page documents with examples: `ENCODE`/`DECODE` →
internal-file `READ`/`WRITE`; assigned `FORMAT` → character variable; DEC
`STRUCTURE`/`RECORD` → derived `TYPE` with `%` component access; hard-coded I/O units
(5, 6, 7) → `*` for standard input/output.
[https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26]

### Tools that help

| Tool | What it does | Source |
|------|--------------|--------|
| `findent` | "indent and convert Fortran sources" — indents/beautifies fixed and free form, and **converts fixed → free** form; ships vim/emacs/gedit plugins | [https://github.com/MFTabriz/findent, accessed 2026-07-26]; [https://sourceforge.net/projects/findent/, accessed 2026-07-26] |
| `fprettify` | "auto-formatter for modern fortran source code" — strict whitespace formatting, auto-indentation, continuation alignment (Python) | [https://github.com/fortran-lang/fprettify, accessed 2026-07-26] |
| `convert.f90` (Metcalf) | Fixed → free form converter | [https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26] |
| `to_f90.f90` (Alan Miller) | Fixed → free form converter | [https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran, accessed 2026-07-26] |

> `findent` and `fprettify` are actively developed; verify current invocation flags at
> the linked repositories before scripting them — this is the part of this file most
> likely to drift.

### Compiler flags for legacy code (gfortran)

All from [https://gcc.gnu.org/onlinedocs/gfortran/Fortran-Dialect-Options.html, accessed 2026-07-26]:

| Flag | Effect (quoted where possible) |
|------|--------------------------------|
| `-std=legacy` | "equivalent but without the warnings for obsolete extensions, and may be useful for old nonstandard programs" |
| `-ffixed-form` | Selects the fixed-form layout — "Fixed form was traditionally used in older Fortran programs" |
| `-ffree-form` | Selects free form — "The free form layout was introduced in Fortran 90" |
| `-ffixed-line-length-n` | "Set column after which characters are ignored in typical fixed-form lines" (72 default, 80, 132) |
| `-std=f95` / `f2003` / `f2008` / `f2018` / `f2023` | "strict conformance to the Fortran 95, Fortran 2003, Fortran 2008, Fortran 2018 and Fortran 2023 standards, respectively; errors are given for all extensions beyond the relevant language standard" |
| `-fdec` | "DEC compatibility mode ... These features are nonstandard and should be avoided at all costs" |
| `-fd-lines-as-comments` | lines beginning with 'd'/'D' "are treated as comment lines" |

Rule of thumb: to *compile an old program unchanged*, reach for `-std=legacy` plus the
correct form flag; to *drive modernization*, tighten to `-std=f2018` and fix what the
compiler rejects.

---

## §§ C_INTEROP_OVERVIEW

Standardized C interoperability arrived in **Fortran 2003**. The gfortran manual:

> "Since Fortran 2003 (ISO/IEC 1539-1:2004(E)) there is a standardized way to generate
> procedure and derived-type declarations and global variables that are interoperable
> with C."
> [https://gcc.gnu.org/onlinedocs/gfortran/Interoperability-with-C.html, accessed 2026-07-26]

Two intrinsic limits to remember:

> "Not all C features have a Fortran equivalent or vice versa. For instance, neither
> C's unsigned integers nor C's functions with variable number of arguments have an
> equivalent in Fortran."
> [https://gcc.gnu.org/onlinedocs/gfortran/Interoperability-with-C.html, accessed 2026-07-26]

And the array-layout mismatch, which is the most common source of interop bugs:

> "array dimensions are reversely ordered in C and that arrays in C always start with
> index 0 while in Fortran they start by default with 1" — so Fortran `A(i,j)`
> corresponds to C's `A[j-1][i-1]`.
> [https://gcc.gnu.org/onlinedocs/gfortran/Interoperability-with-C.html, accessed 2026-07-26]

---

## §§ C_INTEROP_KINDS

The `ISO_C_BINDING` intrinsic module supplies kind parameters whose storage matches
the corresponding C type. Use them (never hard-coded kind numbers) so the binding
stays correct across platforms.

| Fortran type | Named constant | C type |
|--------------|----------------|--------|
| `INTEGER` | `C_INT` | `int` |
| `INTEGER` | `C_SHORT` | `short int` |
| `INTEGER` | `C_LONG` | `long int` |
| `INTEGER` | `C_LONG_LONG` | `long long int` |
| `INTEGER` | `C_INT8_T` | `int8_t` |
| `INTEGER` | `C_INT64_T` | `int64_t` |
| `REAL` | `C_FLOAT` | `float` |
| `REAL` | `C_DOUBLE` | `double` |
| `REAL` | `C_LONG_DOUBLE` | `long double` |
| `COMPLEX` | `C_FLOAT_COMPLEX` | `float _Complex` |
| `LOGICAL` | `C_BOOL` | `_Bool` |
| `CHARACTER` | `C_CHAR` | `char` |

Table from [https://gcc.gnu.org/onlinedocs/gfortran/ISO_005fC_005fBINDING.html, accessed 2026-07-26].
The same page notes GNU Fortran *extensions* (128-bit integers, unsigned types) and
additional character named constants including `C_NULL_CHAR`, `C_NEW_LINE`, and
`C_CARRIAGE_RETURN`. `C_SIZE_T` (matching C `size_t`) is also provided and is used in
the string example below. `unverified — model knowledge` for the exact `C_SIZE_T`
entry not being in the fetched excerpt, though it appears in a fetched code example.

The module also defines two derived types and the corresponding null values:
`C_PTR` with `C_NULL_PTR`, and `C_FUNPTR` with `C_NULL_FUNPTR`.
[https://gcc.gnu.org/onlinedocs/gfortran/ISO_005fC_005fBINDING.html, accessed 2026-07-26]

The intrinsic procedures the module provides are: `C_ASSOCIATED`, `C_F_POINTER`,
`C_F_PROCPOINTER`, `C_FUNLOC`, `C_LOC`, and `C_SIZEOF`.
[https://gcc.gnu.org/onlinedocs/gfortran/ISO_005fC_005fBINDING.html, accessed 2026-07-26]

---

## §§ C_INTEROP_PROCEDURES

From the fortran-lang "Procedures for binding to C interfaces" page
[https://fortran-lang.org/learn/intrinsics/cfi/, accessed 2026-07-26]:

- **`c_loc(x)`** — "Obtain the C address of an object." The argument must have the
  `pointer` or `target` attribute and cannot be coindexed. Returns a `c_ptr`.
- **`c_associated(c_ptr_1 [, c_ptr_2])`** — tests whether a C pointer is null, or
  whether two point to the same target. "The return value is of type logical; it is
  .false. if either c_ptr_1 is a C NULL pointer or if c_ptr1 and c_ptr_2 point to
  different addresses."
- **`c_f_pointer(cptr, fptr [, shape])`** — "Assigns the target (the C pointer cptr)
  to the Fortran pointer fptr and specifies its shape if fptr points to an array."
- **`c_funloc(x)`** — "Determines the C address of the argument" for an interoperable
  procedure; returns a `c_funptr`.
- **`c_f_procpointer(cptr, fptr)`** — "Assigns the target of the C function pointer
  cptr to the Fortran procedure pointer fptr," enabling cross-language callbacks.
- **`c_sizeof(x)`** — "Returns the number of bytes occupied by the argument."

Association test example (verbatim):

```fortran
subroutine association_test(a,b)
use iso_c_binding, only: c_associated, c_loc, c_ptr
implicit none
real, pointer :: a
type(c_ptr) :: b
  if(c_associated(b, c_loc(a))) &
     stop 'b and a do not point to same target'
end subroutine association_test
```
[https://fortran-lang.org/learn/intrinsics/cfi/, accessed 2026-07-26]

> Caution (added in the 2026-07-26 review pass): this snippet is verbatim from the
> upstream documentation, but its logic is inverted — `c_associated(b, c_loc(a))`
> returns `.true.` when `b` and `a` *do* refer to the same target, so the `stop`
> message fires in exactly the opposite case its text describes. Treat it as a syntax
> illustration of `c_associated`/`c_loc`, not as correct program logic; negate the
> condition (`if (.not. c_associated(...))`) for a real association check.

### C_F_POINTER in detail

Arguments (from the gfortran page):
- **`CPTR`** — a scalar `C_PTR`, input.
- **`FPTR`** — a Fortran pointer interoperable with `CPTR`, output.
- **`SHAPE`** — optional rank-one integer array giving the dimensions; required when
  `FPTR` is an array.
- **`LOWER`** — optional rank-one integer array of lower bounds; valid only when
  `SHAPE` is present. Lower bounds "default to 1 unless specified otherwise."

Verbatim worked example — call a C function that hands back a pointer, then view it as
a 12-element Fortran array:

```fortran
program main
  use iso_c_binding
  implicit none
  interface
    subroutine my_routine(p) bind(c,name='myC_func')
      import :: c_ptr
      type(c_ptr), intent(out) :: p
    end subroutine
  end interface
  type(c_ptr) :: cptr
  real,pointer :: a(:)
  call my_routine(cptr)
  call c_f_pointer(cptr, a, [12])
end program main
```
[https://gcc.gnu.org/onlinedocs/gfortran/C_005fF_005fPOINTER.html, accessed 2026-07-26]

---

## §§ C_INTEROP_CALLING

To be callable across the language boundary, a procedure must carry the `BIND(C)`
attribute. The gfortran manual: "Subroutines and functions have to have the BIND(C)
attribute to be compatible with C."
[https://gcc.gnu.org/onlinedocs/gfortran/Interoperability-with-C.html, accessed 2026-07-26]

### Scalar arguments and `VALUE`

C passes scalars **by value**; Fortran passes **by reference** by default. Use the
`VALUE` attribute on a Fortran dummy to match a C by-value parameter; omit it (and the
argument is a pointer on the C side). Verbatim from the gfortran manual, for the C
prototype `int func(int i, int *j)`:

```fortran
integer(c_int) function func(i,j)
  use iso_c_binding, only: c_int
  integer(c_int), VALUE :: i
  integer(c_int) :: j
```
[https://gcc.gnu.org/onlinedocs/gfortran/Interoperable-Subroutines-and-Functions.html, accessed 2026-07-26]

Here `i` (C `int`, by value) gets `VALUE`; `j` (C `int *`, a pointer) does not.

### Strings: null termination is on you

C strings are null-terminated `char` arrays; Fortran `CHARACTER` variables are
fixed-length and space-padded, with no terminator. When calling a C function you must
append `C_NULL_CHAR` yourself. Verbatim example calling a C `print` function:

C side:
```c
#include <stdio.h>
void print_C(char *string)
{
   printf("%s\n", string);
}
```

Fortran side:
```fortran
use iso_c_binding, only: C_CHAR, C_NULL_CHAR
interface
  subroutine print_c(string) bind(C, name="print_C")
    use iso_c_binding, only: c_char
    character(kind=c_char) :: string(*)
  end subroutine print_c
end interface
call print_c(C_CHAR_"Hello World"//C_NULL_CHAR)
```
[https://gcc.gnu.org/onlinedocs/gfortran/Interoperable-Subroutines-and-Functions.html, accessed 2026-07-26]

Note the two devices: `bind(C, name="print_C")` maps the Fortran name to the exact C
symbol (case matters — see mixed-language build notes), and the argument is declared
as an assumed-size `character(kind=c_char)` array `string(*)`, not a Fortran scalar
string. The literal is built with the `C_CHAR_"..."` kind prefix and terminated with
`//C_NULL_CHAR`.

A second string example — calling C's `strncpy` from Fortran (verbatim), showing
`C_SIZE_T` and `VALUE` on the length:

```fortran
use iso_c_binding
implicit none
character(len=30) :: str,str2
interface
  subroutine strncpy(dest, src, n) bind(C)
    import
    character(kind=c_char),  intent(out) :: dest(*)
    character(kind=c_char),  intent(in)  :: src(*)
    integer(c_size_t), value, intent(in) :: n
  end subroutine strncpy
end interface
str = repeat('X',30)
call strncpy(str, c_char_"Hello World"//C_NULL_CHAR, &
             len(c_char_"Hello World",kind=c_size_t))
print '(a)', str
end
```
[https://gcc.gnu.org/onlinedocs/gfortran/Interoperable-Subroutines-and-Functions.html, accessed 2026-07-26]

### Passing arrays

Because C stores arrays in row-major order starting at index 0 while Fortran is
column-major starting at 1, a Fortran `A(i,j)` is the C `A[j-1][i-1]` (see
C_INTEROP_OVERVIEW). For a one-dimensional array there is no ordering issue, only the
0-vs-1 base. Pass the array normally (by reference) and let the C side index from 0.
The `C_F_POINTER` example above shows the reverse direction: receiving a C array
pointer and giving it a Fortran shape.
[https://gcc.gnu.org/onlinedocs/gfortran/Interoperability-with-C.html, accessed 2026-07-26]

### Calling Fortran from C

The mechanism is symmetric: give the Fortran procedure `BIND(C, name="...")`, then
declare a matching `extern` prototype on the C side using the interoperable types.
The scalar example above (`int func(int i, int *j)` implemented in Fortran) is exactly
this direction — C sees an ordinary `func` symbol. `unverified — model knowledge` for
the framing "symmetric"; the underlying `BIND(C)`/`VALUE` mechanics are grounded in the
gfortran examples cited above.

---

## §§ MIXED_LANGUAGE_BUILD

### Name mangling — the problem `bind(c)` solves

Before Fortran 2003, mixing Fortran and C meant guessing each compiler's *name
mangling*: Fortran compilers historically decorated external symbol names
inconsistently — lowercasing, and appending zero, one, or two underscores (e.g. a
Fortran `FOO` might emit `foo_`, `foo__`, `FOO`, or `foo` depending on the compiler
and flags). C, by contrast, uses the source name essentially unchanged. Matching them
required per-compiler macros and was a perennial portability headache.
`unverified — model knowledge` (the specific historical mangling variants are
well-known lore but were not on a page fetched for this file; verify against a
compiler manual if the exact decoration matters for a given build).

`BIND(C)` fixes this by making the Fortran compiler emit the symbol under a
**specified, unmangled C name**. With `bind(C, name="print_C")` the object file
contains exactly the symbol `print_C` — no lowercasing, no underscores — so the C and
Fortran object files link without any mangling guesswork.
[https://gcc.gnu.org/onlinedocs/gfortran/Interoperable-Subroutines-and-Functions.html, accessed 2026-07-26]
(The `name=` specifier is demonstrated in the string example above.)

### Practical linking notes

- Compile each language with its own compiler (e.g. `gcc -c foo.c`, `gfortran -c
  bar.f90`) and link with the compiler that pulls in the correct runtime — commonly
  the Fortran driver (`gfortran`), because it links the Fortran runtime libraries
  automatically. `unverified — model knowledge` (standard practice; not on a fetched
  page).
- Keep the interoperable kinds (`c_int`, `c_double`, …) on the Fortran side aligned
  with the actual C types; a mismatch compiles cleanly but corrupts data at the
  boundary.
- For arrays, account for the 0-vs-1 base and row- vs column-major ordering documented
  above.

---

## §§ SOURCES

All URLs accessed 2026-07-26.

**Fetched successfully and cited above:**
- gfortran — Interoperability with C: https://gcc.gnu.org/onlinedocs/gfortran/Interoperability-with-C.html
- gfortran — ISO_C_BINDING module: https://gcc.gnu.org/onlinedocs/gfortran/ISO_005fC_005fBINDING.html
- gfortran — Interoperable Subroutines and Functions: https://gcc.gnu.org/onlinedocs/gfortran/Interoperable-Subroutines-and-Functions.html
- gfortran — C_F_POINTER: https://gcc.gnu.org/onlinedocs/gfortran/C_005fF_005fPOINTER.html
- gfortran — Fortran Dialect Options: https://gcc.gnu.org/onlinedocs/gfortran/Fortran-Dialect-Options.html
- fortran-lang — Procedures for binding to C interfaces: https://fortran-lang.org/learn/intrinsics/cfi/
- fortran-lang — Best Practices (index) and Style Guide: https://fortran-lang.org/learn/best_practices/ ; https://fortran-lang.org/learn/best_practices/style_guide/
- Fortran Wiki — Modernizing Old Fortran: https://fortranwiki.org/fortran/show/Modernizing+Old+Fortran
- WVU HPC — Best Practices in Modern Fortran: https://wvuhpc.github.io/Modern-Fortran/11-Best-Practices/index.html
- Flang — A Fortran feature history cheat sheet: https://releases.llvm.org/18.1.0/tools/flang/docs/FortranFeatureHistory.html
- findent: https://github.com/MFTabriz/findent ; https://sourceforge.net/projects/findent/
- fprettify: https://github.com/fortran-lang/fprettify
- Web search summary (deleted-features list), accessed 2026-07-26, drawing on the Intel
  "Deleted and Obsolescent Language Features" doc and the Flang cheat sheet.

**Attempted but NOT fetched (gaps — see below):**
- Intel — "Deleted and Obsolescent Language Features"
  (https://www.intel.com/content/www/us/en/docs/fortran-compiler/developer-guide-reference/2023-1/deleted-and-obsolescent-language-features.html):
  returned **HTTP 403 Forbidden**. Its content is represented here only through the
  Anthropic web-search summary, not a direct read.
- fortran-lang best-practices index page and style-guide page were fetched but proved
  to be navigation/table-of-contents pages without the interop or modernization prose
  hoped for; substantive best-practices content in this file therefore comes from the
  WVU HPC page and the Fortran Wiki page instead.
- The F2023 draft (23-007r1.pdf, https://j3-fortran.org/doc/year/23/23-007r1.pdf) was
  deliberately **not** fetched (very large PDF); per task guidance, obsolescence claims
  are grounded in the Flang cheat sheet and the web-search summary of authoritative
  secondary sources instead.

**Unverified — model-knowledge assertions in this file (each flagged inline too):**
- Fixed/free source-form file-extension conventions: the core `.f` → fixed / `.f90` →
  free mapping was verified against the fortran-lang gotchas page in the 2026-07-26
  review pass; only the `.for`/`.f77`/`.f95`+ variants remain model knowledge.
- The `DO 10 I = 1.100` blanks-insignificant hazard and the general "blanks are
  insignificant in fixed form" rule.
- COMMON positional-matching / blank-vs-named / `SAVE` persistence details.
- EQUIVALENCE replacement recommendations (`transfer`, allocatables, pointers).
- ENTRY replacement approach (split into module procedures).
- Test-baseline-first modernization discipline (step 1).
- Historical Fortran name-mangling underscore/case variants.
- "Link with the Fortran driver" practical linking note; "symmetric" framing of
  calling Fortran from C.
