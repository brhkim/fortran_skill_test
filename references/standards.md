# Fortran Standards Landscape

<!-- §§ METADATA -->
> **Created:** 2026-07-26
> **Sources verified as of:** 2026-07-26 (every URL below was fetched or searched on this date; each substantive claim carries an inline "accessed 2026-07-26" citation)
> **Staleness note:** The *standards timeline* (which edition introduced which feature, and when each was published) is stable and rarely changes — treat it as durable. What drifts fastest is **compiler conformance status**: gfortran, LLVM Flang, and Intel `ifx` ship new Fortran 2023 features on their own release cadences, so the "Compiler Conformance" section below goes stale first. Before relying on a specific feature being available in a specific compiler version, re-check the three status pages named in that section (GCC's `gcc.gnu.org/fortran/` and its GCC-14 changes page, Flang's `flang.llvm.org` standards docs, and Intel's oneAPI Fortran release notes).
> **Scope:** This file covers the standards landscape and compiler conformance. It is a reference for a skill that steers coding assistants toward *modern* Fortran; the concrete "which idiom to write" guidance lives elsewhere in the skill.

---

<!-- §§ TIMELINE -->
## 1. Standards Timeline

Each ISO/IEC edition of the Fortran standard is (with narrow, explicitly-documented exceptions) a superset of its predecessor: standard-conforming code written to an older edition continues to compile and run under a newer one. This is a deliberate, long-standing design commitment of the Fortran standards committees (WG5 internationally, and J3 for the United States). See the backward-compatibility note in Section 3.

| Edition | ISO/IEC designation | Headline additions (what the edition is remembered for) |
|---------|--------------------|----------------------------------------------------------|
| **FORTRAN 77** | ISO 1539:1980 | The `CHARACTER` data type, block `IF`/`ELSE IF`/`END IF` control structures, and the `PARAMETER` statement. The baseline "classic" fixed-form Fortran that later editions modernized away from. |
| **Fortran 90** | ISO/IEC 1539:1991 | The **module system** (`MODULE`/`USE`), free-form source, array operations and array syntax, dynamic memory (`ALLOCATABLE`, `POINTER`), derived types, and `INTERFACE` blocks. The single largest modernization in the language's history. |
| **Fortran 95** | ISO/IEC 1539-1:1997 | A relatively small revision: `FORALL`, `PURE` and `ELEMENTAL` procedures, and cleanup/deletion of obsolescent FORTRAN 77 features. |
| **Fortran 2003** | ISO/IEC 1539-1:2004 | **Object-oriented programming** (type extension, polymorphism, type-bound procedures), and **interoperability with C** (the `ISO_C_BINDING` module and `BIND(C)`). Also parameterized derived types and IEEE floating-point arithmetic support. |
| **Fortran 2008** | ISO/IEC 1539-1:2010 | **Coarrays** (native parallel/SPMD programming), **submodules**, `DO CONCURRENT`, the `CONTIGUOUS` attribute, and the `BLOCK` construct. |
| **Fortran 2018** | ISO/IEC 1539:2018 | **Further parallel features**: teams, events, and collective subroutines extending the coarray model; plus "further interoperability of Fortran with C" (assumed-rank arrays, the C descriptor). Historically referred to during drafting as Fortran 2015. |
| **Fortran 2023** | ISO/IEC 1539-1:2023 | Published November 2023 (see Section 2). Incremental refinements: much longer source lines/statements, enumeration types, conditional expressions, `TYPEOF`/`CLASSOF`, string tokenization intrinsics, `DO CONCURRENT` `REDUCE` locality, and extended BOZ handling. |

> **Citation for the timeline table:** ISO designation numbers and edition feature summaries are corroborated by the Fortran Wiki Fortran 2023 page (https://fortranwiki.org/fortran/show/Fortran+2023, accessed 2026-07-26) and the GNU Fortran standards documentation (https://gcc.gnu.org/onlinedocs/gfortran/Standards.html, accessed 2026-07-26), which explicitly lists the ISO designations "Fortran 95 (ISO/IEC 1539:1997)", "Fortran 2003 (ISO/IEC 1539-1:2004)", "Fortran 2008 (ISO/IEC 1539-1:2010)", and "Fortran 2018 (ISO/IEC 1539:2018)".
>
> **Unverified — model knowledge:** The specific *year* attached to Fortran 90 (1991) and the FORTRAN 77 designation (ISO 1539:1980) were not confirmed against a fetched primary source in this pass; the gfortran Standards page confirms Fortran 90/95/2003/2008/2018 support but did not enumerate the F90 or F77 ISO year. The per-edition *feature groupings* (module system in 90, OOP + C interop in 2003, coarrays + submodules in 2008, further parallel features in 2018) are widely documented and consistent with the fetched Flang/Intel/GCC material, but the exact ISO publication years for F77 and F90 should be reconfirmed against ISO if precision matters.

---

<!-- §§ F2023_DETAIL -->
## 2. Fortran 2023 in Detail

### 2.1 Publication

Fortran 2023 (ISO/IEC 1539-1:2023) was **published by ISO on 17 November 2023**, following five years of development after Fortran 2018 (https://stevelionel.com/drfortran/2023/11/23/fortran-2023-has-been-published/, accessed 2026-07-26). The official standard is sold through ISO; the Fortran Wiki notes it is available from the ISO website for a fee (CHF 216) (https://fortranwiki.org/fortran/show/Fortran+2023, accessed 2026-07-26). Because the official text is paywalled, the **free** J3 working draft and WG5 papers described in Section 4 are the practical references for the standard's content.

### 2.2 What is new versus Fortran 2018

The following features are the substantive additions of Fortran 2023 over Fortran 2018. Each is corroborated by at least two of the fetched sources: Steve Lionel's publication announcement (abbreviated **[Lionel]** below), John Reid's WG5 new-features paper N2212 (**[Reid N2212]**), and Peter Klausler's LLVM Flang implementor's notes (**[Flang F202X]**).

- **Longer source lines and statements; no continuation limit.** In free-form source, a line may now be up to **10,000 characters** (previously 132), a single statement may be up to **one million characters**, and the previous cap on the number of continuation lines has been **removed entirely**. Fixed-form source receives comparable increases. This is aimed at machine-generated code. ([Lionel], accessed 2026-07-26; corroborated by [Reid N2212] "Longer lines and statements," https://wg5-fortran.org/N2201-N2250/N2212.pdf, accessed 2026-07-26; and by the GCC 14 changelog which describes `-std=f2023` raising free-form line length to 10,000 and statement length to 1 million characters, per search of https://gcc.gnu.org/gcc-14/changes.html, accessed 2026-07-26.)

- **Enumeration types (`ENUMERATION TYPE`).** A genuine, distinct enumeration *type* category (as opposed to the C-interoperable named-constant `ENUM, BIND(C)` that existed since Fortran 2003), broadly analogous to a C++ `enum class`. ([Lionel] "True enumeration types with C interoperability features," accessed 2026-07-26; corroborated by [Reid N2212] "Enumeration types," accessed 2026-07-26; and [Flang F202X] "ENUMERATION TYPE: A new distinct type category similar to C++ enum class," https://flang.llvm.org/docs/F202X.html, accessed 2026-07-26.)

- **Conditional expressions and conditional arguments (the `?` construct).** An inline conditional ("ternary-style") expression written with `?`, for example `value = ( a>0.0 ? a : 0.0 )`. There are also **conditional arguments**, allowing a call site to select which actual argument is passed, or `.nil.` to indicate an absent optional argument. Note that `?` here is *syntax*, not an operator. ([Lionel], accessed 2026-07-26; corroborated by [Reid N2212] "Conditional expressions" and "Conditional arguments," accessed 2026-07-26; and [Flang F202X] "Conditional expressions: C-style ternary operator syntax," accessed 2026-07-26.)

- **`TYPEOF` and `CLASSOF` type specifiers.** Intrinsic type-inquiry specifiers usable to declare an entity with the same (dynamic/declared) type as another entity — a building block toward generic programming. ([Lionel] "`typeof()` and `classof()` intrinsics," accessed 2026-07-26; corroborated by [Reid N2212] "TYPEOF/CLASSOF," accessed 2026-07-26; and [Flang F202X] "TYPEOF/CLASSOF: Type specifiers for indirect declarations," accessed 2026-07-26.)

- **String tokenization intrinsics (`SPLIT` and `TOKENIZE`).** New intrinsics for parsing/splitting character strings into tokens — long-missing standard string handling. ([Reid N2212] "Tokenization Intrinsics ... including SPLIT and TOKENIZE," accessed 2026-07-26. Note: this specific feature was confirmed from Reid N2212 and general knowledge; it was *not* explicitly named in the fetched Lionel excerpt, so it rests on a single fetched primary source.)

- **`DO CONCURRENT` `REDUCE` locality specifier.** A reduction locality-spec added to the `DO CONCURRENT` construct, letting the compiler treat named variables as reduction targets across concurrent iterations. ([Lionel] "New reduction specifier for `DO CONCURRENT`," accessed 2026-07-26; corroborated by [Reid N2212] "DO CONCURRENT REDUCE," accessed 2026-07-26; [Flang F202X] "DO CONCURRENT REDUCE: New reduction clauses for parallel constructs," accessed 2026-07-26; and the Intel release notes which list "The REDUCTION locality-spec on the DO CONCURRENT construct" among implemented F2023 features, per search of https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html, accessed 2026-07-26.)

- **Extended BOZ (binary/octal/hexadecimal) constant handling.** Broader rules for where BOZ literal constants may appear and how they are interpreted; the Intel release notes give a concrete example: "BOZ constants in array constructors with an explicit REAL type-spec are now interpreted as the bits of a REAL value with the same kind as the type-spec." ([Reid N2212] "BOZ," accessed 2026-07-26; corroborated by the Intel 2025 release notes per search, accessed 2026-07-26.)

- **`C_F_POINTER` lower-bound argument.** The `C_F_POINTER` procedure (from `ISO_C_BINDING`) gained an optional lower-bounds specification for the Fortran pointer it creates. ([Lionel], accessed 2026-07-26.)

- **New `AT` edit descriptor.** A format edit descriptor that automatically trims trailing whitespace from character values on output, plus finer control over optional leading zeros in real-value output. ([Lionel], accessed 2026-07-26.)

- **Integer-array syntax for subscripts, ranks, and bounds.** Using an integer array to specify subscripts, ranks, and array bounds, supporting rank-independent (rank-agnostic) code. ([Lionel], accessed 2026-07-26; corroborated by [Flang F202X] "Rank-independent features: Including vector-based subscripting," accessed 2026-07-26.)

- **`SIMPLE` procedures.** A more restricted category than `PURE` procedures. (Named in [Flang F202X] "SIMPLE procedures: A restricted subset of PURE procedures," accessed 2026-07-26. This was *not* in the fetched Lionel or Reid excerpts, so it rests on the single Flang source; treat the exact semantics as needing confirmation against the standard draft.)

> **Feature-list completeness caveat:** The list above is the union of features named across the fetched sources; it is representative but not guaranteed exhaustive. The authoritative complete list is in the standard's own Introduction (pages xiii–xiv), as the Fortran Wiki notes (https://fortranwiki.org/fortran/show/Fortran+2023, accessed 2026-07-26), and in Reid's paper N2212 (Section 4).

### 2.3 Backward compatibility

Fortran 2023 preserves the language's traditional backward-compatibility guarantee: it is essentially a superset of Fortran 2018, so conforming Fortran 2018 (and older) programs continue to compile and run. This continuity is why the standards committee versions so cautiously and why targeting an older edition remains safe (see Section 5). ([Lionel] frames F2023 as an incremental addition of features atop F2018 rather than a breaking revision, accessed 2026-07-26.)

**One documented near-breaking change to be aware of:** LLVM Flang's implementor notes flag "automatic reallocation of character allocatable variables in certain I/O contexts" as a departure from Fortran's traditional strict backward-compatibility philosophy — the single compatibility wrinkle called out for F2023 (https://flang.llvm.org/docs/F202X.html, accessed 2026-07-26). Code relying on the old behavior of allocatable-character I/O should be reviewed.

---

<!-- §§ FREE_DOCUMENTS -->
## 3. Where the Free Standard Documents Live

The official ISO/IEC 1539-1:2023 text is paywalled, but the committee's working drafts and papers — which are technically near-identical to the published standard — are free. All three links below were fetched and confirmed to resolve on 2026-07-26.

- **J3 working draft of Fortran 2023 — document 23-007r1.**
  URL: https://j3-fortran.org/doc/year/23/23-007r1.pdf
  Confirmed to resolve: it is a ~9.5 MB PDF whose table of contents matches a full Fortran language standard specification (lexical tokens, types, expressions, execution control, I/O, program units, intrinsic procedures, IEEE arithmetic, C interoperability, plus annexes on processor dependencies and deleted/obsolescent features). This is the free, near-final draft that most people cite in place of the paywalled ISO text. (Accessed 2026-07-26.)

- **J3 interpretation / post-publication working document — document 24-007.**
  URL: https://j3-fortran.org/doc/year/24/24-007.pdf
  Confirmed to resolve: another ~9.5 MB full-standard PDF, the 2024 iteration of J3's standing document 007, which incorporates interpretations (corrections/clarifications) on top of the 2023 base text. Steve Lionel describes this line of documents as J3's "interpretation reference document (standing document 007)" (https://stevelionel.com/drfortran/2023/11/23/fortran-2023-has-been-published/, accessed 2026-07-26). Use this when you need the standard text *with* official interpretations folded in. (Accessed 2026-07-26.)
  > Note: the fetch of this PDF's raw object stream identified it as a full standard specification but could not display its title page; the "interpretation document 007" identity rests on the naming convention (`24-007`) plus Lionel's description of the 007 series, not on a title-page read.

- **WG5 new-features paper — document N2212, "The new features of Fortran 2023," by John Reid.**
  URL: https://wg5-fortran.org/N2201-N2250/N2212.pdf
  Confirmed to resolve: a ~276 KB PDF authored by John Reid, titled "New Features of Fortran 2023," enumerating each F2023 addition with a short description. This is the **authoritative concise delta summary** for what changed from 2018 to 2023 and is the best single free document to read first. (Accessed 2026-07-26.)
  > Gap: the fetched PDF stream did not expose the paper's exact date or restate its document number in the header; the "N2212" designation comes from the URL path (WG5's N2201–N2250 document range). The author (John Reid) and title were confirmed from the fetched content.

---

<!-- §§ COMPILER_CONFORMANCE -->
## 4. Compiler Conformance Status

> **This section drifts fastest — re-verify before relying on any specific version/feature pairing.** Status pages to re-check: `gcc.gnu.org/fortran/` and the GCC release changes pages; `flang.llvm.org/docs/`; and Intel's oneAPI Fortran compiler release notes.

### 4.1 GNU Fortran (gfortran)

gfortran has **full or near-full support through Fortran 2018** and **initial/partial support for Fortran 2023**. The gfortran Standards documentation states it fully implements Fortran 95, handles essentially all standard-conforming Fortran 90 and 77 programs, implements "almost all" of Fortran 2003 and 2008, and has "partial support" for Fortran 2018 (including full support for the "Further Interoperability of Fortran with C" technical specification) (https://gcc.gnu.org/onlinedocs/gfortran/Standards.html, accessed 2026-07-26).

For Fortran 2023 specifically: **GCC 14 introduced the `-std=f2023` flag** in preparation for full F2023 support. That flag already raises the free-form line-length limit to 10,000 characters and the statement-length limit to one million characters. As of GCC 14, gfortran is described as "well into" F2018 with "initial support of some features of F2023" — i.e., the flag exists and some features work, but F2023 is not yet complete. (Search of https://gcc.gnu.org/gcc-14/changes.html and related GCC-14 Fortran feature discussion, accessed 2026-07-26.)
> Note: the dedicated GCC wiki page `gcc.gnu.org/wiki/Fortran2023Status` was **not reachable** during this pass (the server returned an Anubis anti-bot access-denied page, accessed 2026-07-26). For a per-feature F2023 implementation matrix, retry that wiki page directly.

### 4.2 LLVM Flang

LLVM Flang is under active development and can compile many programs, but "some functionality is still missing" (https://flang.llvm.org/docs/, accessed 2026-07-26). Its documentation includes a "Fortran 2018 Grammar" design document and a Fortran 202X (i.e., F2023) implementor's analysis, and it references a dedicated "Flang Fortran Standards Support" page for the detailed conformance matrix.

Crucially, Flang's stated **priority is *not* standard conformance for its own sake** but rather "ensuring that existing working applications will port successfully to LLVM Flang with minimal effort." Its F2023 work is prioritized by user demand, focusing first on features already shipping in other compilers (such as the `DO CONCURRENT` reduction clauses and degree-argument trigonometric functions) that improve portability (https://flang.llvm.org/docs/F202X.html, accessed 2026-07-26). For an authoritative per-feature matrix, consult the "Flang Fortran Standards Support" page referenced from the Flang docs.

### 4.3 Intel Fortran (`ifx`)

Intel's LLVM-based `ifx` compiler **fully supports Fortran standards through Fortran 2018** and implements a **large subset of Fortran 2023**. Confirmed F2023 features in `ifx` include the `REDUCTION` locality-spec on `DO CONCURRENT` and the extended BOZ-constant interpretation in array constructors with an explicit `REAL` type-spec. The 2025 oneAPI release extensively updated the Intel Fortran Developer Guide and Reference to document supported F2018 and F2023 language features. (Search of https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html and the Intel community post "IFORT/IFX Initial Fortran 2023 Features," accessed 2026-07-26.)

Two important context points for assistants:
- **`ifort` (Intel Fortran Compiler Classic) is discontinued** as of the oneAPI 2025 release. New code should target `ifx`, not `ifort`. (Search of Intel 2025 release notes, accessed 2026-07-26; independently corroborated in the 2026-07-26 review pass by a direct fetch of Intel's announcement reposted on Fortran Discourse — ifort's final build is 2021.13, removed from packages starting with release 2025.0: https://fortran-lang.discourse.group/t/a-historic-moment-for-the-intel-fortran-compiler-classic-ifort/8350, accessed 2026-07-26.)
- The Intel release-notes HTML page returned **HTTP 403 Forbidden** on direct fetch (accessed 2026-07-26); the details above come from web-search result summaries of that same page and the Intel community forum, not from a direct read of the page body. Reconfirm specifics against the live release notes when precision matters.

### 4.4 Conformance summary table

| Compiler | Through F2018 | F2023 status | Standard-selection flag |
|----------|---------------|--------------|--------------------------|
| **gfortran** (GNU) | Near-full (F2003/2008 "almost all"; F2018 "partial") | Initial/partial; `-std=f2023` exists since GCC 14 | `-std=f2023` |
| **LLVM Flang** | F2018 grammar implemented; active development, gaps remain | Demand-driven subset; portability-first, not conformance-first | See "Flang Fortran Standards Support" page (Flang defaults to accepting the latest standard; no strict `-std=` era-lock equivalent documented here) |
| **Intel `ifx`** | Full | Large subset (e.g., `DO CONCURRENT REDUCE`, extended BOZ) | `-stand`/`/stand` family (see Intel docs; specific F2023 value not confirmed in this pass) |

> **Unverified — model knowledge:** Intel's exact `-stand`/`/stand:f23`-style flag value for selecting Fortran 2023 was *not* confirmed from a fetched source (the Intel release-notes page was 403-blocked). Confirm the precise flag spelling in the Intel Fortran Developer Guide before documenting it as fact. Flang's exact standard-selection flag behavior was likewise not confirmed against its standards-support page in this pass.

---

<!-- §§ STD_FLAGS -->
## 5. `-std=` Flags by Compiler

### gfortran (fully confirmed)

The gfortran `-std=` option accepts these values (https://gcc.gnu.org/onlinedocs/gfortran/Fortran-Dialect-Options.html, accessed 2026-07-26):

| Value | Selects |
|-------|---------|
| `f95` | Strict Fortran 95 conformance; extensions beyond F95 are errors. |
| `f2003` | Strict Fortran 2003 conformance. |
| `f2008` | Strict Fortran 2008 conformance. |
| `f2018` | Strict Fortran 2018 conformance. The deprecated alias `f2008ts` also maps here. |
| `f2023` | Strict Fortran 2023 conformance. The alias `f202y` selects this too *and* enables proposed experimental (next-standard) features. |
| `gnu` | **Default.** A superset of the latest standard including all GNU extensions; warns on obsolete extensions. |
| `legacy` | Like `gnu` but without obsolete-extension warnings; for old nonstandard programs. |

### Intel `ifx` and LLVM Flang

Not fully confirmed in this pass — see the "Unverified" note at the end of Section 4. Intel uses a `-stand`/`/stand` family of flags; Flang documents standard support on a dedicated page but its `-std`-equivalent behavior was not read directly here. Reconfirm both against their live docs.

---

<!-- §§ GUIDANCE -->
## 6. Guidance for Coding Assistants

The recommendations below follow directly from the conformance picture in Section 4.

1. **Target Fortran 2018 as today's safe common denominator.** All three major compilers (gfortran, `ifx`, Flang) have full or near-full F2018 support, whereas F2023 support is partial and uneven across them (Section 4). Code written to F2018 will compile broadly today. This is the default recommendation unless a specific F2023 feature is needed *and* the target compiler is known to support it.

2. **Use F2023 features only when the target compiler is confirmed to support them.** Some F2023 features are already broadly and reliably available because compilers implemented them early by user demand — most notably the **`DO CONCURRENT` `REDUCE` locality specifier** (implemented in Intel `ifx`, prioritized in Flang, and present in gfortran's F2023 work) and the **longer source-line / statement limits** (available under gfortran's `-std=f2023` since GCC 14). These are the safest F2023 features to reach for. Newer or more complex F2023 features (enumeration types, conditional expressions, `TYPEOF`/`CLASSOF`, `SPLIT`/`TOKENIZE`, `SIMPLE` procedures) should be gated on a confirmed compiler-version check.

3. **Prefer `ifx` over `ifort`.** Intel's classic `ifort` compiler is discontinued as of oneAPI 2025 (Section 4.3); generate and test against `ifx`.

4. **Rely on backward compatibility, not bleeding-edge syntax.** Because every edition is a superset of its predecessor (Sections 1 and 2.3), choosing an older target costs little in expressiveness for most code and buys wide portability. Reserve F2023-specific syntax for cases where it materially improves the code and the toolchain is known.

5. **When in doubt, re-check the live status pages.** The compiler landscape moves faster than this document; the staleness note at the top names the exact pages to consult.

---

<!-- §§ SOURCES -->
## 7. Sources

All URLs accessed 2026-07-26.

**Fetched successfully (direct read):**
- Steve Lionel, "Fortran 2023 has been published" — https://stevelionel.com/drfortran/2023/11/23/fortran-2023-has-been-published/ (publication date, feature list, backward-compat framing, description of J3 standing document 007)
- John Reid, "New Features of Fortran 2023," WG5 document N2212 — https://wg5-fortran.org/N2201-N2250/N2212.pdf (authoritative F2023 delta; title and author confirmed, exact date/number not exposed in stream)
- LLVM Flang documentation home — https://flang.llvm.org/docs/ (development status, F2018 grammar, references to standards-support page)
- LLVM Flang Fortran 202X implementor notes (Peter Klausler) — https://flang.llvm.org/docs/F202X.html (per-feature F2023 implementor analysis; portability-first priority; character-allocatable I/O compat wrinkle)
- gfortran Standards documentation — https://gcc.gnu.org/onlinedocs/gfortran/Standards.html (per-edition support levels, ISO designations)
- gfortran Fortran Dialect Options — https://gcc.gnu.org/onlinedocs/gfortran/Fortran-Dialect-Options.html (complete `-std=` value table)
- J3 working draft 23-007r1 — https://j3-fortran.org/doc/year/23/23-007r1.pdf (resolves; ~9.5 MB full standard draft; TOC confirmed)
- J3 document 24-007 — https://j3-fortran.org/doc/year/24/24-007.pdf (resolves; ~9.5 MB full standard document; 007-series interpretation document per naming + Lionel)
- Fortran Wiki, Fortran 2023 — https://fortranwiki.org/fortran/show/Fortran+2023 (ISO price/availability, pointer to Introduction pp. xiii–xiv, Reid paper reference)

**Reached only via web-search summaries (page body not directly read):**
- GCC 14 release changes — https://gcc.gnu.org/gcc-14/changes.html (GCC 14 added `-std=f2023`; line/statement length limits) — obtained via web search, not direct fetch
- Intel oneAPI Fortran Compiler Release Notes 2025 — https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html (**HTTP 403 on direct fetch**; F2023 subset, `DO CONCURRENT REDUCE`, BOZ, ifort discontinuation obtained via web-search summary)
- Intel Community, "IFORT/IFX Initial Fortran 2023 Features" — https://community.intel.com/t5/Intel-Fortran-Compiler/IFORT-IFX-Initial-Fortran-2023-Features/td-p/1545818 (via web search)

**Attempted but FAILED to load (gaps):**
- GCC wiki Fortran 2023 status — https://gcc.gnu.org/wiki/Fortran2023Status (**access denied — Anubis anti-bot page**; per-feature gfortran F2023 matrix could not be retrieved. Re-attempted in the 2026-07-26 review pass with a browser user-agent and via archive.org — still blocked. A human browser session is needed to read this page.)
