# Beyond Storage: State as a Runtime Control Problem in Parallel and Distributed Systems

This repository contains the ACM Computing Surveys manuscript "Beyond Storage: State as a Runtime Control Problem in Parallel and Distributed Systems" and its separate supplement. The revision builds to 19 pages for the manuscript and 20 for the supplement using the vendored ACM template. The revision history and links to the reviewer-response work are recorded in [the integration notes](docs/csur-revision-integration.md).

For collaboration and agent-specific writing rules, see `AGENTS.md`.

## Layout

- `main.tex`: main CSUR manuscript.
- `supplement.tex`: appendix-style supplement with its own cited references.
- `refs.bib`: BibTeX database for the current draft.
- `figures/`: TikZ figure sources for the propagation view and integrated runtime loop.
- `third_party/acmart-src/`: vendored ACM template files (`acmart` v2.19).

## Build

Use Tectonic's stable v2 compile entrypoint:

```bash
tectonic -X compile -Z search-path="$(pwd)/third_party/acmart-src" main.tex
tectonic -X compile -Z search-path="$(pwd)/third_party/acmart-src" supplement.tex
```

Or build both PDFs together with:

```bash
make pdf
```

### Authorless Version

`authorless-version/main_authorless.tex` is the authorless, combined manuscript
used for similarity checking at the original submission stage. Its main text is
a snapshot, and the current revision uses the root-level main manuscript and
supplement. The authorless build retains the appendix inputs and uses one
bibliography. Build it from its own directory so that its relative references
to the shared repository assets resolve correctly:

```bash
cd authorless-version
tectonic -X compile -Z search-path="../third_party/acmart-src" main_authorless.tex
```

In Windows PowerShell, the equivalent command is:

```powershell
Set-Location authorless-version
tectonic -X compile -Z search-path=../third_party/acmart-src main_authorless.tex
```

The generated PDF is `authorless-version/main_authorless.pdf`.

To verify that the actual compile path is using the vendored ACM template rather than Tectonic's cached copy, run:

```bash
make verify-template
```

Then inspect:

- `output/v219_stdout/main.stdout.log`
- `output/v219_stdout/supplement.stdout.log`

The first lines should show:

```text
Document Class: acmart 2026/06/27 v2.19
```

If a relative `search-path` still resolves to Tectonic's cached template on a local machine, prefer an absolute path to `third_party/acmart-src/`. The repository `Makefile` already does this to avoid accidentally compiling against the bundled older `acmart` copy.

The submission target is ACM `acmart` with `\documentclass[manuscript]{acmart}`. The main paper and supplement are packaged as separate files for CSUR editorial review.

## Status

The current package preserves the root-level manuscript structure, uses a vendored ACM template source, and keeps the main paper and supplement as separate submission artifacts for low-diff collaboration and review.

## Licensing

This is a manuscript collaboration repository. The authors have chosen to keep the manuscript sources and original figures under their existing copyright terms, with no additional repository-wide reuse license during peer review.

Third-party materials retain their own terms. The vendored [ACM template](third_party/acmart-src/LICENSE) includes the LaTeX Project Public License 1.3c, and the bundled [GitHub CLI](bin/LICENSE) includes its MIT license. Consult the original sources for the terms of any externally sourced figures or examples.
