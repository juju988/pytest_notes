# Pytest notes

## Useful arguments
Command | Result
------------ | -------------
pytest -x | exit after first failed test
pytest -x --pdb | auto-start debugger at first failure point
pytest --lf	| runs only last failed test
pytest --ff | runs from last failed test and continues
pytest --nf | tests new files first and then remaining files
pytest --durations=2 | shows the 2 slowest tests
pytest --durations=2 --vv | shows the 2 slowest tests with actual durations
pytest --sw | stepwise fail/fix. Use --stepwise-skip if there's something that needs to be skipped until later


## Using a config file
* PEP 518 allowed the use of a `pyproject.toml` file to specify what packages/tools a project needs.
* Many projects (e.g. Black, coverage.py, tox, pytest) allow you to specify their configurations in the `pyproject.toml` file.

## Monkeypatching

## pytest decorators

## Useful pytest plugins
Name | Description
------------ | -------------
pytest-xdist | splits tests across cores
pytest-cov | shows code coverage
pytest-clarity | for highlighting unexpected values in dictionaries
pytest-instafail | stop on first failure
pytest-sugar | errors/fails and progress bar
pytest-benchmark | find slowest tests
pytest-socket | disables network calls during testing