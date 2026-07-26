# Modern Fortran Idiom and Pitfalls

<!-- METADATA
created: 2026-07-26
sources verified as of: 2026-07-26
staleness note: The idiomatic guidance here is drawn from the fortran-lang.org "Learn"
materials (Best Practices + Quickstart), the legacy fortran90.org best-practices page,
fortranwiki.org, and Steve Lionel's "Doctor Fortran" blog. The most drift-prone content
is (a) standard-version attributions (e.g. what "Fortran 2018" or "Fortran 2023" added),
(b) intrinsic module names/members in iso_fortran_env, and (c) any URL under
fortran-lang.org/learn/ (the site is actively reorganized). Re-check the fortran-lang
Best Practices index and Quickstart index first if a link 404s.
-->

## How to use this file

This is an on-demand reference for writing **modern Fortran** (Fortran 90 and later, through
Fortran 2018/2023 features where noted) rather than legacy fixed-form FORTRAN 77. Coding
assistants tend to emit FORTRAN 77 idiom (fixed-form columns, implicit typing, `COMMON`
blocks, `GOTO`, `real*8`) and code that fails to compile. Every rule below is followed by the
reasoning and a citation. Adapt the code patterns; do not copy claims without the cited source.

Citation convention: each claim carries an inline source URL and "accessed 2026-07-26". A
consolidated Sources list appears at the end.

Code-snippet convention: many snippets below are *fragments* that assume a real-kind
parameter (`dp` or `wp`) is already in scope — e.g. via
`use, intrinsic :: iso_fortran_env, only : dp => real64` (Section 7). They will not
compile standalone without that declaration. Snippets were verified by inspection, not
compilation.

---

## 1. The modern-defaults ruleset

These are the non-negotiable defaults for new Fortran. Apply all of them unless a specific
constraint (C interop, legacy interface) forces otherwise.

| # | Rule | Instead of (legacy) | Why |
|---|------|---------------------|-----|
| 1 | Free-form source, `.f90` extension | Fixed-form, `.f` extension | `.f` triggers fixed-form column rules and causes compile failures; `.f90` denotes free-form. [gotchas] |
| 2 | `implicit none` in **every** program unit | Implicit typing (I–N rule) | Catches typos as compile errors instead of silently creating new variables. [gotchas][fortranwiki] |
| 3 | `module` for shared data/procedures | `COMMON` blocks, `include` files | Modules are "the preferred way [to] create modern Fortran libraries and applications" and generate explicit interfaces. [modules_programs] |
| 4 | `intent(in/out/inout)` on all dummy args | Untyped dummy args | Lets the compiler check misuse and self-documents. [organising_code] |
| 5 | `allocatable` arrays | `pointer` for owned memory | Automatic deallocation on scope exit; "it is not possible to create memory leaks." [fortran90-bp] |
| 6 | Kind parameters via `iso_fortran_env` (`real64`) or `selected_real_kind` | `real*8`, `double precision` | Portable, self-documenting precision. [floating_point] |
| 7 | Structured `do` / `end do`, `case`, `if`/`end if` | Labeled `GOTO`, arithmetic `IF` | Readable, analyzable control flow. [quickstart] |
| 8 | Explicit `print "(A)", ...` formatting | Bare `print *` | `print *` prepends a blank column (punch-card carriage-control legacy). [gotchas] |

> Attribution note: rules 2–8 are directly supported by fetched pages (cited). Rule 1's
> `.f`/`.f90` mapping is from the gotchas page; the broader "free-form is the modern default"
> framing is standard community consensus reflected across all fetched fortran-lang pages.

### Minimal correct program skeleton

```fortran
program demo
  use, intrinsic :: iso_fortran_env, only : dp => real64
  implicit none

  real(dp) :: x
  x = 1.0_dp / 3.0_dp
  print "(A, g0)", "x = ", x
end program demo
```

Structure supported by: `implicit none` in every unit [fortranwiki, accessed 2026-07-26];
`iso_fortran_env` kind aliasing [https://fortran-lang.org/learn/best_practices/floating_point/,
accessed 2026-07-26]; explicit format to avoid the leading blank
[https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26].

---

## 2. implicit none — the single most important rule

By default, Fortran types variables whose names start with the letters **I through N** as
`INTEGER`, and everything else as `REAL`, unless declared. This is "a legacy of the past" and
developers are "strongly advised to put an `implicit none` statement in your program and in each
module." [https://fortranwiki.org/fortran/show/implicit+none, accessed 2026-07-26]

The concrete failure mode: a typo silently creates a *new* implicitly-typed variable rather
than raising an error. The gotchas page gives the example of writing `nbofchildrem` instead of
`nbofchildren` — with implicit typing this compiles and silently misbehaves.
[https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26]

Placement caveat: `implicit none` at module level covers module procedures, but it does **not**
propagate into an `interface` block — add it separately inside the interface if needed.
[https://fortranwiki.org/fortran/show/implicit+none, accessed 2026-07-26]

Historical/standards context: implicit typing dates to the original 1956 Fortran standard.
Steve Lionel has argued that default implicit typing should be made **obsolescent** (flagged by
standards-checking modes) rather than deleted, to discourage new use without breaking legacy
code; the feature "caused programmers more trouble" than almost any other. His memorable framing
of the hazard: "GOD IS REAL, unless declared INTEGER."
[https://stevelionel.com/drfortran/2021/09/18/doctor-fortran-in-implicit-dissent/, accessed 2026-07-26]

Modern extension (Fortran 2018): `implicit none (type, external)` additionally requires that
all external procedures be declared, catching another class of silent errors. This is discussed
in the Doctor Fortran "implicit dissent" thread as the direction of travel.
[https://stevelionel.com/drfortran/2021/09/18/doctor-fortran-in-implicit-dissent/, accessed 2026-07-26]

---

## 3. Modules over COMMON and includes

Guidance from [https://fortran-lang.org/learn/best_practices/modules_programs/, accessed 2026-07-26]:

- One module per source file, filename matching the module name for navigation.
- Prefix module names with the library name to avoid clashes across dependencies.
- Begin each module with a comment documenting its purpose; document each procedure and its
  argument intents. "Documentation is [...] one of the most important parts of creating
  long-living software, regardless of language."
- Set default visibility with a `private` statement, then `public` only the exported symbols.
- Import with `use ..., only :` to make dependencies explicit.
- Module procedures live after a `contains` statement.
- **Module variables are static (implicitly saved).** Restrict them to constants/parameters;
  export non-constant state as `protected` rather than `public`.

```fortran
module fpm_toml
  implicit none
  private

  public :: read_package_file
  public :: toml_table, toml_array
end module fpm_toml
```

```fortran
use fpm_error,   only : error_t, fatal_error
use fpm_strings, only : string_t
```

**Submodules** (Fortran 2008) break recompilation cascades: declare the interface in the parent
module, put the implementation in a `submodule`. Editing the implementation does not force
recompilation of everything that `use`s the parent.
[https://fortran-lang.org/learn/best_practices/modules_programs/, accessed 2026-07-26]

```fortran
module stdlib_quadrature
  implicit none
  private
  public :: trapz

  interface trapz
    pure module function trapz_dx_dp(y, dx) result(integral)
      real(dp), intent(in) :: y(:)
      real(dp), intent(in) :: dx
      real(dp) :: integral
    end function trapz_dx_dp
  end interface trapz
end module stdlib_quadrature

submodule (stdlib_quadrature) stdlib_quadrature_trapz
contains
  pure module function trapz_dx_dp(y, dx) result(integral)
    ! implementation
  end function trapz_dx_dp
end submodule stdlib_quadrature_trapz
```

Program bodies should stay thin: "reusing implementations from modules allows you to write
reusable code and focus the program unit on conveying user input to the respective library
functions." [https://fortran-lang.org/learn/best_practices/modules_programs/, accessed 2026-07-26]

Note: it is recommended to always place functions and subroutines within modules; the module
generates the required **explicit interface**, which enables assumed-shape arguments and
compile-time argument checking. [https://fortran-lang.org/learn/quickstart/organising_code/,
accessed 2026-07-26]

---

## 4. Procedures: intent, functions vs subroutines, optional args

Source: [https://fortran-lang.org/learn/quickstart/organising_code/, accessed 2026-07-26].

`intent` values: `in` (read-only), `out` (write-only), `inout` (read-write). "It is good
practice to always specify the `intent` attribute for dummy arguments; this allows the compiler
to check for unintentional errors and provides self-documentation."

Functions should not modify their arguments — all function arguments should be `intent(in)`;
such functions are `pure`.

```fortran
function vector_norm(vec) result(norm)
  implicit none
  real, intent(in) :: vec(:)
  real :: norm
  norm = sqrt(sum(vec**2))
end function vector_norm
```

Optional arguments with `present`:

```fortran
function vector_norm(vec, p) result(norm)
  real, intent(in) :: vec(:)
  integer, intent(in), optional :: p
  real :: norm
  if (present(p)) then
    norm = sum(abs(vec)**p) ** (1.0/p)
  else
    norm = sqrt(sum(vec**2))
  end if
end function vector_norm
```

Use the `result` clause to name the return value (avoids naming the return via the function name
itself). Prefer defining procedures inside modules so callers get an explicit interface and can
pass assumed-shape arrays. [https://fortran-lang.org/learn/quickstart/organising_code/, accessed 2026-07-26]

---

## 5. Arrays

### 5.1 Passing arrays: assumed-shape (preferred) vs explicit-shape

Source: [https://fortran-lang.org/learn/best_practices/arrays/, accessed 2026-07-26].

Four ways to pass arrays: **assumed-shape** (preferred), **assumed-rank**, **explicit-shape**,
**assumed-size** (avoid). Assumed-shape uses colon notation and requires an explicit interface
(i.e. the procedure lives in a module).

```fortran
subroutine f(r)
  real(dp), intent(out) :: r(:)     ! assumed-shape
  integer :: n, i
  n = size(r)
  do i = 1, n
    r(i) = 1.0_dp / i**2
  end do
end subroutine f
```

"No array copy is done, [...] the shape and size information is automatically passed along and
checked at compile and optionally at runtime." Strides can be passed without copying:
`call f(r(1:10:2))`. Avoid obscuring intent with `call f(r(:))` — just write `call f(r)`.

Explicit-shape passes the dimensions manually and is mainly for C interop / legacy:

```fortran
subroutine f(n, r)
  integer, intent(in) :: n
  real(dp), intent(out) :: r(n)     ! explicit-shape
  ...
end subroutine
```

Critical limitation: with explicit-shape "the shape is not checked" — wrong dimension args give
"potentially incorrect results" silently, and non-contiguous strides force temporary-array
copies. Assumed-**size** arrays (asterisk final dimension) disable `size`/`shape` and "should be
avoided in favour of assumed-shape or assumed-rank arrays."

Assumed-rank (Fortran 2018) writes `r(..)` and dispatches with `select rank`:

```fortran
subroutine h(r)
  real(dp), intent(in) :: r(..)
  select rank(r)
  rank(1)
    ! ...
  rank(2)
    ! ...
  end select
end subroutine h
```

### 5.2 Declaration, bounds, storage order

Source: [https://fortran-lang.org/learn/quickstart/arrays_strings/, accessed 2026-07-26].

Two equivalent declaration notations:

```fortran
integer, dimension(10) :: array1
integer :: array2(10)
real, dimension(10, 10) :: array3
real :: array4(0:9)      ! custom lower bound
real :: array5(-5:5)
```

- Arrays are **one-based by default**: the first element along any dimension is index 1.
- Arrays are stored in **column-major** order: the first index varies fastest. (Loop with the
  first index innermost for cache efficiency — the inverse of C's row-major convention.)

Custom bounds propagate into procedures; query the upper bound with `ubound`:

```fortran
subroutine print_eigenvalues(kappa_min, lam)
  integer, intent(in) :: kappa_min
  real(dp), intent(in) :: lam(kappa_min:)
  integer :: kappa
  do kappa = kappa_min, ubound(lam, 1)
    print *, kappa, lam(kappa)
  end do
end subroutine print_eigenvalues
```

Validate shapes defensively:

```fortran
if (size(r) /= 4) error stop "Incorrect size of 'r'"
if (any(shape(r) /= [2, 2])) error stop "Incorrect shape of 'r'"
```

Note: `size(r)` returns the *total* number of elements; pass a dimension as the second argument
(`size(r, 1)`) for one axis. [https://fortran-lang.org/learn/best_practices/arrays/, accessed 2026-07-26]

### 5.3 Slicing and constructors

Slice syntax is `start:end:stride`
[https://fortran-lang.org/learn/quickstart/arrays_strings/, accessed 2026-07-26]:

```fortran
array1(1:10:2)     ! odd indices
array1(10:1:-1)    ! reversed
array1(:) = 0      ! whole-array assignment
array2(:,1)        ! first column
```

Array constructors [https://fortran-lang.org/learn/best_practices/arrays/, accessed 2026-07-26]:

```fortran
integer :: r(5)
r = [1, 2, 3, 4, 5]

real(dp) :: s(5)
s = [real(dp) :: 1, 2, 3, 4, 5]              ! typed constructor

integer :: i
real(dp) :: t(5)
t = [(real(i**2, dp), i = 1, size(t))]        ! implied-do
```

### 5.4 Elementwise operations

Source: [https://fortran-lang.org/learn/best_practices/element_operations/, accessed 2026-07-26].

Prefer whole-array (elemental) expressions over explicit loops. Mark scalar-logic procedures
`elemental` so they auto-apply across arrays of any conforming shape. **`elemental` implies
`pure`** (no side effects).

```fortran
real(dp) elemental function nroot(n, x) result(y)
  integer, intent(in) :: n
  real(dp), intent(in) :: x
  y = x**(1._dp / n)
end function
```

```fortran
print *, nroot(2, 9._dp)
print *, nroot(2, [1._dp, 4._dp, 9._dp, 10._dp])
print *, nroot(2, reshape([1._dp, 4._dp, 9._dp, 10._dp], [2, 2]))
print *, nroot([2, 3, 4, 5], [1._dp, 4._dp, 9._dp, 10._dp])
```

---

## 6. Allocatable arrays and move_alloc

Source: [https://fortran-lang.org/learn/best_practices/allocatable_arrays/, accessed 2026-07-26]
unless otherwise noted.

The `allocatable` attribute "provides a safe way for memory handling." Unlike pointers,
allocatables are deallocated automatically on scope exit — no leaks. fortran90.org states it
directly: "When using allocatable arrays (as opposed to pointers), Fortran manages the memory
automatically and it is not possible to create memory leaks."
[https://www.fortran90.org/src/best-practices.html, accessed 2026-07-26]

```fortran
real(dp), allocatable :: temp(:)
allocate(temp(10))
```

Guard before use / re-allocation:

```fortran
if (allocated(arr)) print *, arr
if (allocated(lam)) deallocate(lam)
allocate(lam(10))
```

`intent(out)` on an allocatable dummy deallocates any prior allocation on entry:

```fortran
subroutine foo(lam)
  real(dp), allocatable, intent(out) :: lam(:)
  allocate(lam(5))
end subroutine foo
```

Allocation does **not** initialize; use `source=` to fill:

```fortran
real(dp), allocatable :: arr(:)
allocate(arr(10), source=0.0_dp)
```

Allocation-on-assignment: assigning a whole array to an allocatable auto-(re)allocates it to the
right-hand-side shape (a modern convenience; enabled by default in standard-conforming
compilers). Combined with `move_alloc` this enables leak-free dynamic resizing:

```fortran
subroutine resize(var, n)
  real(wp), allocatable, intent(inout) :: var(:)
  integer, intent(in), optional :: n
  real(wp), allocatable :: tmp(:)
  integer :: this_size, new_size
  integer, parameter :: initial_size = 16

  if (allocated(var)) then
    this_size = size(var, 1)
    call move_alloc(var, tmp)      ! moves allocation; var becomes unallocated
  else
    this_size = initial_size
  end if

  if (present(n)) then
    new_size = n
  else
    new_size = this_size + this_size/2 + 1
  end if

  allocate(var(new_size))

  if (allocated(tmp)) then
    this_size = min(size(tmp, 1), size(var, 1))
    var(:this_size) = tmp(:this_size)
  end if
end subroutine resize
```

`move_alloc` transfers an allocation from one allocatable to another without copying the data;
the source becomes unallocated. Preferred over manual pointer juggling.
[https://fortran-lang.org/learn/best_practices/allocatable_arrays/, accessed 2026-07-26]

> Note: the fetched allocatable-arrays example did not declare `tmp` and had a minor
> `inital_size`/`initial_size` typo on the source page; both are corrected above so the snippet
> compiles. The `move_alloc` semantics and structure are as published.

Rule of thumb: reach for `pointer` only when you genuinely need aliasing / linked structures.
For owned buffers, use `allocatable`.

---

## 7. Floating point

### 7.1 Kinds and precision

Source: [https://fortran-lang.org/learn/best_practices/floating_point/, accessed 2026-07-26].

Never use `real*8` (non-standard) or bare `real`/`double precision` for portable code. Define a
kind parameter once and use it everywhere. Three idiomatic ways:

```fortran
integer, parameter :: dp = selected_real_kind(15)          ! 15 sig. digits
integer, parameter :: dp = kind(0.0d0)                      ! infer from a double literal
use, intrinsic :: iso_fortran_env, only : dp => real64     ! preferred: named, portable
```

A central kind module is recommended:

```fortran
module kind_parameter
   integer, parameter :: sp = selected_real_kind(6, 37)    ! 32 bits
   integer, parameter :: dp = selected_real_kind(15, 307)  ! 64 bits
   integer, parameter :: qp = selected_real_kind(33, 4931) ! 128 bits

   integer, parameter :: i1 = selected_int_kind(2)
   integer, parameter :: i2 = selected_int_kind(4)
   integer, parameter :: i4 = selected_int_kind(9)
   integer, parameter :: i8 = selected_int_kind(18)
end module kind_parameter
```

### 7.2 Literal-constant traps

**Always suffix floating literals with the kind** — this is the single most common precision bug.
Writing `real(dp), parameter :: x = 9.3` first rounds `9.3` to *single* precision, then widens;
`9.3_dp` is correct. [https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26]

```fortran
real(dp) :: a, b, c
a = 1.0_dp
b = 3.5_dp
c = 1.34e8_dp
```

fortran90.org states the rule flatly: "Always write all floating point constants with the _dp
suffix: 1.0_dp, 3.5_dp, 1.34e8_dp."
[https://www.fortran90.org/src/best-practices.html, accessed 2026-07-26]

**Second literal trap:** integer-suffixed division. `1_dp / 3_dp` is *integer* division (the
values are integers with kind `dp`) and yields 0, not 0.3333. Use decimal points: `1.0_dp /
3.0_dp`. [https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26]

To force real division from integer operands:

```fortran
a = real(3, dp) / 4     ! 0.75_dp
a = 3 * 1.0_dp / 4      ! 0.75_dp
```

### 7.3 Comparison pitfall

Never compare floats with `==`. Compare within a tolerance (model knowledge — the fetched pages
document the *cause*, i.e. rounding of literals, but do not give an explicit `abs(a-b) < tol`
recipe). Flagged: **unverified — model knowledge** for the exact idiom below; the underlying
rounding hazard is documented at
[https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26].

```fortran
if (abs(a - b) <= tol * max(abs(a), abs(b))) then   ! relative tolerance
```

### 7.4 Printing without precision loss

Use `(g0)` (unlimited) or an exponential format like `(es24.16e3)`.
[https://fortran-lang.org/learn/best_practices/floating_point/, accessed 2026-07-26]
fortran90.org recommends `(es23.16)` for double precision and notes doubles need **17**
significant digits (not 16) to round-trip.
[https://www.fortran90.org/src/best-practices.html, accessed 2026-07-26]

---

## 8. Integer division gotcha

Source: [https://fortran-lang.org/learn/best_practices/integer_division/, accessed 2026-07-26].

Integer `/` truncates toward zero, and equal-precedence operators evaluate **left to right** —
so operand order changes results:

```fortran
integer :: n
n = 3
print *, n / 2           ! 1   (truncated)
print *, n*(n + 1)/2     ! 6   (multiply first)
print *, n/2*(n + 1)     ! 4   (n/2 = 1 first, then *4)  <-- left-to-right surprise
n = -3
print *, n / 2           ! -1  (truncation, not floor)
```

Fix by casting or multiplying by a real:

```fortran
print *, real(n, dp) / 2    ! 1.5
print *, n * 1.0_dp / 2     ! 1.5
```

Practical rule: in mixed integer/real formulas, put a real factor *before* any division, or cast
explicitly.

---

## 9. Strings

Source: [https://fortran-lang.org/learn/quickstart/arrays_strings/, accessed 2026-07-26].

Fixed-length declaration:

```fortran
character(len=4) :: first_name
first_name = 'John'
```

**Deferred-length allocatable character** (the modern default for variable-length text) — two
forms:

```fortran
character(:), allocatable :: first_name
allocate(character(4) :: first_name)     ! explicit allocation
first_name = 'John'

character(:), allocatable :: last_name
last_name = 'Smith'                      ! allocation on assignment (preferred)
```

Concatenation with `//`, and `trim` to drop trailing blanks:

```fortran
full_name = first_name // ' ' // last_name
```

Arrays of strings require a fixed element length:

```fortran
character(len=10), dimension(2) :: keys, vals
keys = [character(len=10) :: "user", "dbname"]
vals = [character(len=10) :: "ben", "motivation"]
! use trim(vals(i)) when printing
```

> Content gap: the fortran90.org best-practices page was fetched but its **string-handling
> section did not surface** in the retrieval (the fetch reported allocatable/deferred-length
> character handling as "absent from this best practices guide"). The deferred-length guidance
> above is therefore sourced from the fortran-lang Quickstart, which is authoritative and
> current. If deeper string idioms are needed later, re-fetch fortran90.org directly.

---

## 10. File I/O (modern patterns)

Source: [https://fortran-lang.org/learn/best_practices/file_io/, accessed 2026-07-26].

Files are managed by **unit identifiers**. Use `newunit=` to let the runtime pick a free unit
(never hard-code unit numbers — a legacy habit that collides).

```fortran
integer :: io
open(newunit=io, file="log.txt")   ! default: create if absent, read+write
! ...
close(io)
```

Read-only / write-only with explicit status and action:

```fortran
open(newunit=io, file="log.txt", status="old", action="read")    ! runtime error if absent
read(io, *) a, b
close(io)

open(newunit=io, file="log.txt", status="new", action="write")   ! or status="replace"
write(io, *) a, b
close(io)
```

Check existence with `inquire`; handle errors with `iostat`/`iomsg` instead of letting the
program abort:

```fortran
logical :: exists
inquire(file="log.txt", exist=exists)

integer :: io, stat
character(len=512) :: msg          ! iomsg needs a fixed-length buffer with room
open(newunit=io, file="log.txt", status="old", action="read", &
     iostat=stat, iomsg=msg)
if (stat /= 0) then
  print *, trim(msg)
end if
```

Appending, navigation, and scratch files:

```fortran
open(newunit=io, file="log.txt", position="append", status="old", action="write")
```

- `rewind` returns to the first record; `backspace` steps back one record.
- `status="scratch"` opens a temporary file auto-deleted on close.
- Delete a file by `close(io, status="delete")`.

---

## 11. Derived types and OOP

Source: [https://fortran-lang.org/learn/quickstart/derived_types/, accessed 2026-07-26].

Basic type (a struct); access members with `%`:

```fortran
type :: t_pair
  integer :: i
  real :: x
end type

type(t_pair) :: pair
pair%i = 1
pair%x = 0.5
```

Construct positionally or by keyword; give components default initializers:

```fortran
type :: t_pair
  integer :: i = 1
  real :: x = 0.5
end type

pair = t_pair(1, 0.5)
pair = t_pair(i=1, x=0.5)
pair = t_pair()            ! all defaults
pair = t_pair(i=2)         ! partial
```

Attributes on the type: `public`/`private`, `bind(c)` (C interop), `extends(parent)`
(inheritance), `sequence`, `abstract`. Components may carry `allocatable`, `pointer`,
`contiguous`, etc.

**Type-bound procedures** (the OOP mechanism) go after a `contains` inside the type. Use `class`
(not `type`) for the passed-object dummy to enable polymorphism:

```fortran
module m_shapes
  implicit none
  private
  public :: t_square

  type :: t_square
    real :: side
  contains
    procedure :: area
  end type

contains

  real function area(self) result(res)
    class(t_square), intent(in) :: self
    res = self%side**2
  end function

end module m_shapes

program main
  use m_shapes
  implicit none
  type(t_square) :: sq
  sq%side = 0.5
  print *, sq%area()      ! function form; subroutines use `call sq%area(x)`
end program main
```

**When to use OOP:** derived types are always worth it for grouping related data (structs).
Reach for type-bound procedures, `class` polymorphism, `abstract` types and `deferred`
procedures when you need runtime dispatch — e.g. a callback interface where callers supply
context. The fortran90.org page shows the abstract-type callback pattern:
[https://www.fortran90.org/src/best-practices.html, accessed 2026-07-26]

```fortran
type, abstract :: integrand
contains
  procedure(func), deferred :: eval
end type
```

Users `extends` this type and provide a concrete `eval`, carrying context as components — type
safe and flexible. For simple numeric kernels, prefer plain module procedures and elemental
functions over OOP ceremony.

---

## 12. Gotchas quick-reference

Condensed from [https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26]:

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| Implicit typing | Typo silently creates a new variable | `implicit none` in every unit |
| Implied `save` | `integer :: c = 0` persists across calls (one-shot compile-time init, unlike C) | Assign separately if you need reset each call |
| Float literal default kind | `x = 9.3` rounds to single before widening to `dp` | Suffix: `9.3_dp` |
| Integer-suffixed division | `1_dp / 3_dp` = 0 (integer division) | Use `1.0_dp / 3.0_dp` |
| Leading space in `print *` | Extra blank column in output | `print "(A)", "text"` |
| File extension | `.f` = fixed-form column rules; compile failure | Use `.f90` for free-form |

The "implied save" trap in full: initializing a variable in its declaration
(`integer :: c = 0`) is "a one-shot compile time initialization" that gives the variable the
`save` attribute — it survives across procedure invocations, unlike the C idiom where such
syntax resets on each entry. [https://fortran-lang.org/learn/quickstart/gotchas/, accessed 2026-07-26]

---

## Sources

All accessed 2026-07-26.

Successfully fetched:

- https://fortran-lang.org/learn/best_practices/ (Best Practices index)
- https://fortran-lang.org/learn/best_practices/style_guide/
- https://fortran-lang.org/learn/best_practices/floating_point/
- https://fortran-lang.org/learn/best_practices/integer_division/
- https://fortran-lang.org/learn/best_practices/modules_programs/
- https://fortran-lang.org/learn/best_practices/arrays/
- https://fortran-lang.org/learn/best_practices/allocatable_arrays/
- https://fortran-lang.org/learn/best_practices/element_operations/
- https://fortran-lang.org/learn/best_practices/file_io/
- https://fortran-lang.org/learn/quickstart/ (Quickstart index)
- https://fortran-lang.org/learn/quickstart/gotchas/
- https://fortran-lang.org/learn/quickstart/arrays_strings/
- https://fortran-lang.org/learn/quickstart/derived_types/
- https://fortran-lang.org/learn/quickstart/organising_code/
- https://www.fortran90.org/src/best-practices.html
- https://fortranwiki.org/fortran/show/implicit+none
- https://stevelionel.com/drfortran/2021/09/18/doctor-fortran-in-implicit-dissent/

Content gaps / caveats:

- The fortran90.org best-practices page was fetched, but its string-handling material did not
  surface in retrieval; deferred-length string guidance (Section 9) is sourced from the
  fortran-lang Quickstart instead. Re-fetch fortran90.org directly if deeper string idioms are
  needed.
- The float-comparison tolerance idiom in Section 7.3 is flagged **unverified — model
  knowledge**; the fetched pages document the rounding cause but not an explicit
  `abs(a-b) < tol` recipe.
- Not fetched (out of scope for this file): the fortran-lang Best Practices "callbacks",
  "type_casting", and "multidim_arrays" subpages, and several Quickstart subpages
  (hello_world, variables, operators_control_flow). Their URLs appear under
  https://fortran-lang.org/learn/best_practices/ and https://fortran-lang.org/learn/quickstart/
  if a future revision needs them.
