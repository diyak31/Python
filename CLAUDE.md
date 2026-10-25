# CLAUDE.md

TheAlgorithms/Python: ~1,600 standalone, educational algorithm implementations.
Each `.py` file is one self-contained algorithm whose doctests are its tests.
Readability beats speed. There is no app, CLI, or package API to run.

## Environment (read first)

- Target is **free-threaded CPython 3.15t** (`.python-version`). The code uses
  3.14+/3.15-only syntax (`except A, B:` without parens, PEP 798 `[*x for x in xs]`),
  so system `python3` fails with `SyntaxError`. Never call bare `python`/`python3`/`pip`.
- Setup once: `uv sync --group=test` (uv downloads 3.15t itself). Then prefix
  everything with `uv run`.
- Do **not** use `uvx pytest`: it runs in an isolated env without numpy/scipy and
  fails with `ModuleNotFoundError`.
- New third-party dependency: add it to `[project].dependencies` in
  `pyproject.toml`, then `uv lock`. Never hand-edit `uv.lock`.

## Commands

```bash
uv run pytest sorts/bubble_sort.py           # one file; doctests are on via addopts (~1s)
uv run pytest graphs/                        # one category
uv run python -m doctest -v sorts/bubble_sort.py

# Full suite exactly as CI runs it (~40s, ~3.3k tests):
uv run --with=pytest-run-parallel pytest --iterations=8 --parallel-threads=auto \
  --ignore-gil-enabled --ignore=computer_vision/flip_augmentation.py \
  --ignore=computer_vision/harris_corner.py --ignore=computer_vision/mosaic_augmentation.py \
  --ignore=data_compression/peak_signal_to_noise_ratio.py --ignore=digital_image_processing/ \
  --ignore=docs/conf.py --ignore=project_euler/ --ignore=scripts/validate_solutions.py \
  --ignore=web_programming/current_stock_price.py \
  --ignore=web_programming/fetch_anime_and_play.py .

# Project Euler is a separate CI job:
uv sync --group=euler-validate --group=test
uv run pytest project_euler/                 # doctests (~25s)
uv run pytest scripts/validate_solutions.py  # checks every solution() answer (~75s)

# Lint (all must pass in CI):
uvx ruff check . && uvx ruff format --check .
uv run scripts/validate_filenames.py
uvx pre-commit run --files <changed files>   # --all-files works but takes ~1 min
```

`ty` runs in CI as advisory only: `uv run --python 3.15 --with ty ty check <file>`.

## Layout

- Top-level dirs are topic categories (`sorts/`, `maths/`, `graphs/`, ...) and many
  have sub-categories: `maths/special_numbers/` (harshad, kaprekar, perfect...),
  `maths/series/`, `maths/numerical_analysis/`, `data_structures/<kind>/`,
  `neural_network/activation_functions/`. Before adding a file, locate its closest
  sibling with `find . -name '*<related term>*' -not -path './.venv/*'` (not
  `ls <top-dir>`) and put the new file in the same directory. Never create a new
  top-level dir. Every dir with `.py` files needs an `__init__.py` (ruff `INP001`).
- Files end with `if __name__ == "__main__":` plus `import doctest; doctest.testmod()`.
- Separate pytest files are rare (24 total), in a `tests/` subdir or as `test_*.py`
  beside the code. `tests/test_sorts.py` covers `sorts/`. Default to doctests.
- `project_euler/problem_NNN/solN.py`: the dir is zero-padded to 3 digits and has an
  `__init__.py`. Each file has a `solution()` whose default args are the problem's
  input. Never doctest the final answer; it is checked against SHA-256 hashes in
  `scripts/project_euler_answers.json`.
- `scripts/` is repo tooling, not algorithms.

## Conventions

- Each algorithm has a module docstring with a Wikipedia or other source URL, type
  hints on every parameter and return, and doctests covering valid **and** invalid
  input. Bad input raises `ValueError` (or similar) and the doctest shows the
  `Traceback`. Return results instead of printing them.
- Use descriptive names: `greatest_common_divisor`, not `gcd`. Filenames are
  snake_case with no uppercase, spaces, or hyphens (`validate_filenames.py` fails CI).
- `pyproject.toml` per-file-ignores relax `ANN*` for legacy dirs (`sorts/`,
  `graphs/`, `maths/`...). That is debt, so still annotate new code. When a ruff
  rule fires, fix the code instead of adding `noqa` or new per-file-ignores.
- pre-commit **rewrites code**: `matchify` turns if/elif chains into `match`,
  `auto-walrus` adds `:=`, and `ruff format` reformats. Re-run the file's tests
  after it.
- CI runs test functions in parallel threads, 8 iterations each, under 3.15t.
  Don't let tests mutate module-level state. Doctests run once.

## Guardrails

- Never edit `DIRECTORY.md` or `README.md`. CI regenerates `DIRECTORY.md` from
  `scripts/build_directory_md.py` on every push.
- Never edit `scripts/project_euler_answers.json`.
- `web_programming/` doctests hit live websites, so offline or proxy failures there
  are environmental, not regressions.
- `digital_image_processing/` and the cv2 files ignored above need
  `uv sync --group=cv` (no 3.15t opencv wheel yet). A bare `uv run pytest .` errors
  during collection there and also runs `project_euler/`. Use the CI command instead.
- Keep each change to one algorithm file. Per the PR template, don't change an
  algorithm and its existing doctests in the same change.
