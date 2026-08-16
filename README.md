# Mocklis documentation

Source for the [Mocklis](https://github.com/mocklis) documentation, written in reStructuredText and built with [Sphinx](https://www.sphinx-doc.org/). The rendered result is published on Read the Docs.

Mocklis itself — the source generator and the step libraries — lives in a separate repository. This one contains prose and samples only.

## Building locally

Requires Python 3.11 or later.

```sh
python -m venv .venv
.venv\Scripts\activate          # Windows;  source .venv/bin/activate elsewhere
pip install -r requirements.txt
```

Then, from the repository root:

```sh
.\make.bat html                 # Windows;  make html elsewhere
```

The result ends up in `build/html`; open `build/html/index.html` to read it.

Other useful targets — `make.bat` and `Makefile` pass everything through to `sphinx-build -M`:

| Target      | Does                                        |
|-------------|---------------------------------------------|
| `html`      | Build the HTML site                         |
| `linkcheck` | Check that external links still resolve     |
| `clean`     | Remove the build directory                  |
| `help`      | List all available targets                  |

## Contents

Everything lives under `source/`, one directory per section, each with an `index.rst` and its images:

| Section           | Covers                                                              |
|-------------------|---------------------------------------------------------------------|
| `introduction`    | What Mocklis is and why it works the way it does                     |
| `installation`    | The NuGet packages and how they fit together                         |
| `getting-started` | A linear walk-through building a first set of mocks                  |
| `usage`           | Attribute parameters, type parameters, debugging and refactoring     |
| `reference`       | The full catalogue of steps and verifications, and what the generator emits for each kind of interface member |
| `extensions`      | Writing your own steps                                               |
| `faq`             | Frequently asked questions                                           |

New sections need both a `source/<name>/index.rst` and an entry in the `toctree` in `source/index.rst`.

## Contributing

Issues and pull requests are welcome. When editing:

- Use `.. sourcecode:: csharp` for code samples, indented four spaces.
- Within a page, the heading hierarchy is `=` over- and under-lined for the title, then `=`, `-`, and `'` for successive levels.
- Link between sections with `:doc:`, e.g. ``:doc:`../reference/index` ``.
- `source/.editorconfig` sets the formatting rules for `.rst` files: four-space indent, LF endings, UTF-8, no trailing whitespace, final newline.

Please make sure `make html` completes without new warnings before opening a pull request.

## Licence

Copyright © 2019-2026 Esbjörn Redmo and contributors. All rights reserved.
