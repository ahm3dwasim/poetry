# OSS Report — python-poetry/poetry #2056

## FLAG: SMALL FIX
A one-line condition change (plus a regression test) makes the treatment of
`VIRTUAL_ENV` consistent; it is a targeted bug fix, not a feature or refactor.

## Root cause (3 lines)
`EnvManager.get()` only looked for an in-project `.venv` when either no external
env was active (`not in_venv`) or a managed env was explicitly activated in
`envs.toml` (`env is not None`). So when `VIRTUAL_ENV`/`CONDA_PREFIX` was set and
no managed env had been activated, an existing in-project `.venv` was skipped and
the external prefix was used — while a cache-dir managed env in the same situation
was preferred over `VIRTUAL_ENV`, giving inconsistent precedence.

## Diff summary
- `src/poetry/utils/env/env_manager.py`: add `or self.in_project_venv_exists()` to
  the branch condition in `get()` so an in-project `.venv` takes precedence over an
  externally activated environment, consistent with an explicitly activated
  managed virtualenv. Added an explanatory comment.
- `tests/utils/env/test_env_manager.py`: add
  `test_get_prefers_in_project_venv_over_env_var` (parametrized over `VIRTUAL_ENV`
  and `CONDA_PREFIX`) asserting the in-project `.venv` wins over the env var.

## Test commands and output
```
$ poetry run python -m pytest tests/utils/env/test_env_manager.py -q
55 passed, 1 skipped

$ poetry run python -m pytest tests/utils/env/ -q
141 passed, 7 skipped

$ poetry run ruff check src/poetry/utils/env/env_manager.py tests/utils/env/test_env_manager.py
All checks passed!

$ poetry run ruff format --check src/poetry/utils/env/env_manager.py tests/utils/env/test_env_manager.py
2 files already formatted

$ poetry run mypy src/poetry/utils/env/env_manager.py
Success: no issues found in 1 source file
```
Before the fix, a focused reproduction test showed `manager.get()` returning the
`VIRTUAL_ENV` prefix (`/environment/prefix`) instead of the project's `.venv`;
after the fix it returns the in-project `.venv`.

## Risks
- Behavior change for users who set `VIRTUAL_ENV` and also have an in-project
  `.venv` present without `virtualenvs.in-project = false`: Poetry now uses the
  in-project `.venv` rather than the externally activated environment. This matches
  the already-established precedence for explicitly activated managed envs
  (`test_get_prefers_explicitly_activated_virtualenvs_over_env_var`), so it makes
  behavior consistent rather than introducing a new inconsistency.
- Users who relied on `VIRTUAL_ENV` overriding a present in-project `.venv` can
  still opt out by setting `virtualenvs.in-project = false`, which makes
  `in_project_venv_exists()` return `False`.
- No change when there is no in-project `.venv`, or when `virtualenvs.in-project`
  is `false`.

## Draft PR description
> **Make treatment of `VIRTUAL_ENV` consistent for in-project `.venv` (fixes #2056)**
>
> When an environment is externally activated (`VIRTUAL_ENV` or `CONDA_PREFIX`),
> `EnvManager.get()` preferred an explicitly activated managed virtualenv (recorded
> in `envs.toml`) over the external prefix, but ignored an existing in-project
> `.venv` and used the external prefix instead. This made the precedence of
> `VIRTUAL_ENV` depend on whether the project's environment lived in the cache dir
> or in-project.
>
> This change gives an in-project `.venv` the same precedence as an explicitly
> activated managed virtualenv: a project-specific environment wins over an
> externally activated one. Users who prefer the old behavior can set
> `virtualenvs.in-project = false`.
>
> Added a parametrized regression test (`VIRTUAL_ENV` and `CONDA_PREFIX`).

## Estimated cost spent
~ $1.4 of the ~$5 session budget.
