# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **documentation** repository for Mocklis — a C#/.NET source-code generator that creates test doubles (mocks) from interfaces, plus a step-based API for giving them behaviour. There is **no C# source here**: every code sample in the `.rst` files describes the separate Mocklis product repository. When a sample looks wrong or outdated, it must be checked against that repository, not against anything in this tree.

Output is published on Read the Docs (`.readthedocs.yaml` points at `source/conf.py`).

## Build

Sphinx is not vendored; install it first:

```
pip install -r requirements.txt
```

Then build from the repository root (`make.bat` on Windows, `Makefile` elsewhere — both delegate every target to `sphinx-build -M`):

```
.\make.bat html          # -> build/html/index.html
.\make.bat linkcheck     # verify external links
.\make.bat clean
.\make.bat help          # list all sphinx-build -M targets
```

`build/` and `source/.venv/` are gitignored.

## Layout

`source/` holds one directory per top-level section, each with a single `index.rst` and its images alongside. The section order is fixed by the `toctree` in `source/index.rst`:

introduction → installation → getting-started → usage → reference → extensions → faq

Adding a section means creating `source/<name>/index.rst` **and** registering it in that toctree — Sphinx warns about orphaned documents otherwise.

Roughly, `introduction` sells the library, `getting-started` is a linear tutorial, `usage` covers day-to-day work (attribute parameters, type parameters, debugging, refactoring), `reference` is the exhaustive per-step catalogue plus a walkthrough of what the generator emits for each member kind, and `extensions` explains writing custom steps.

## reStructuredText conventions

Follow the existing files rather than generic RST habits:

- **Code samples** use `.. sourcecode:: csharp` (not `code-block`), body indented 4 spaces.
- **Heading hierarchy** within a page: document title over- *and* under-lined with `=`; sections underlined `=`; subsections `-`; sub-subsections `'`.
- **Cross-references** between sections use `:doc:` with a relative path including `index`, e.g. ``:doc:`../reference/index` ``.
- **Inline literals** (double backticks) are for identifiers that exist in the API — type names, step names, attribute parameters. Single backticks (rendered as italics) are used loosely for informal concepts like `mock property`, `Mocklis class`, `restricted type`.
- Long lines are fine; the VS Code linter is configured to ignore `D001`.

`source/.editorconfig` governs `.rst` files: 4-space indent, LF endings, UTF-8, trailing whitespace trimmed, final newline. `.gitattributes` additionally forces LF on `*.rst` and `Makefile`, and marks image formats binary.

## Versioning

`version` (short `X.Y`) and `release` (full, including pre-release tags) live in `source/conf.py` and are the only place the documented Mocklis version is recorded. Release work happens on a branch named for the release (e.g. `v1.4.0-beta.1`) and merges to `main`; bumping those two values plus the copyright year is the version-introduction commit.
