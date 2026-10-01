# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

`zuspin` is an R package whose reason to exist is to **vendor Spin** (the Promela model checker,
nimble-code/Spin) and expose it to R as a library rather than as a command-line tool. The R layer
is meant to stay thin; the real work is getting a globals-heavy C program that assumes it owns
the process to behave as a re-entrant library under `R CMD INSTALL` on Linux, macOS and Windows
(see the CI matrix in [.github/workflows/R-CMD-check.yaml](.github/workflows/R-CMD-check.yaml)).

It is one of a family of sibling packages in `../` that vendor a solver into R, and it should
follow their conventions: `zusat` (CaDiCaL, C++), `zusmt` (OpenSMT, C++) and `zucbor` (TinyCBOR,
C). `zusmt/CLAUDE.md`, `zusmt/roadmap.md` and `zusmt/tools/vendor.sh` are the most complete
worked examples. Read them before designing build glue here.

## Current state

The package is a `usethis` skeleton: `DESCRIPTION` still has placeholder Title/Description/Authors,
there is no `src/`, no exported function and no test file beyond `tests/testthat.R`. Nothing
about Spin has been imported or written yet. Work is paced by [roadmap.md](roadmap.md); its
stages name the libspin stage each one depends on, and `../libspin/roadmap.md` is the C-side
plan. Stage 0 (package identity, API list) and Stage 1 (toolchain spike) can start now;
Stage 2 waits for libspin `v0.1.0`.

## Architecture decision: where the CLI-to-library conversion lives

Spin upstream is a CLI. Turning it into a library is a large, Spin-specific refactor (see the
checklist below) that has nothing to do with R, so the plan is:

1. **`libspin`, a separate repository** (github.com/pedrobtz/libspin, checked out at
   `../libspin`), holds the conversion. Its `main` was seeded from nimble-code/Spin `master`
   (6.5.2) with the `upstream` remote pointing at Spin, so upstream history stays mergeable. It gets its own plain-C test suite (repeated calls in one
   process, failure then success, independent instances, sanitizers) and can be reused outside R.
2. **`zuspin` vendors a pinned `libspin` tag** under `src/`, via a maintainer-side
   `tools/vendor.sh` that is idempotent and records version, commit and checksums, as `zusmt`
   does. The vendored tree is never hand-edited; anything R-specific that must change in it is a
   reapplicable patch rule, and R-specific glue lives in our own `src/*.c`.

If this decision changes, update this section first; every later stage depends on it.

Upstream facts that shape the work (Spin 6.5.2, `Src/`, ~26k lines of C): the grammar is
`spin.y` and `y.tab.c` is **not** committed, so the vendoring step must run yacc/bison and commit
the output; `exit()` is called directly only a handful of times but `alldone()` is spread across
ten files; there are about a dozen `system()` calls (the `cpp` preprocessor, launching the C
compiler for `pan`); and output is written with `printf`/`stdout` on nearly 3,000 lines, so I/O
redirection is the bulk of the mechanical work.

What the library conversion has to do to Spin specifically (do not assume upstream has any of
this):

- **Termination.** `main.c`'s `alldone()`/`exit()` and the fatal-error helpers must return error
  codes. `R CMD check` rejects `exit`, `abort` and `_exit` in package code outright.
- **Globals.** Spin keeps the parser (yacc `spin.y`, `spinlex.c`), symbol table, line/file
  tracking and option flags in file-scope and `static` variables. Mutable state goes into a
  context struct; read-only tables may stay global. A fresh context per call is what a CLI got
  for free from a fresh process.
- **I/O.** Output goes to `stdout`/`stderr` and the input is read through an external `cpp`
  spawned with `system()`. Both must become buffers/streams/callbacks; CRAN flags `printf` to
  the console and the package must not `system()` out of C.
- **Cleanup.** Spin leaks freely and relies on process exit. Per-call arenas or explicit
  teardown are needed so repeated calls do not accumulate.
- **Verifier generation is out of process by design.** Spin's `-a` mode emits `pan.c`/`pan.h`
  which the user compiles and runs; the library produces that source as a result. Compiling and
  running `pan` from R, if offered, is an R-side `callr`/`system2` concern, not a library one.

For the R binding: the C core returns error codes, the shim translates them into R errors only
after cleanup, and nothing may unwind through Spin code by `longjmp` (`Rf_error` included) while
it holds resources.

## Commands

```r
devtools::document()                      # roxygen -> man/, NAMESPACE
devtools::load_all()
devtools::test()                          # all tests
testthat::test_file("tests/testthat/test-<name>.R")
devtools::check()                         # what CI runs
pkgdown::build_site()                     # docs; GitHub Pages workflow builds on push to main
```

From a shell: `R CMD build . && R CMD check --as-cran zuspin_*.tar.gz`. Once `src/` exists,
`R CMD INSTALL --preclean .` is the honest rebuild; stale `.o` files are easy to be fooled by.

## Conventions carried over from the sibling packages

- Portable `make` only in `src/Makevars`: list every object by hand, no `$(wildcard)`, no `-W`
  overrides. Prefer a single `Makevars` over a drifting `Makevars.win`.
- Generated parser output (yacc/lex) is produced at vendoring time and committed, so installing
  the package never needs bison/flex.
- Upstream licence files stay in the vendored tree; add a `LICENSE.note` and list Spin's authors
  as `ctb`/`cph` in `Authors@R`.
- Strings crossing the R boundary carry a declared encoding (`Rf_translateCharUTF8`,
  `Rf_mkCharCE(..., CE_UTF8)`), not the session's.
