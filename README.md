# Pytest notes

## Useful arguments
Command | Result
pytest --x | exit after first failed test
pytest --pdb | auto-start debugger at first failure point
pytest --lf	| runs only last failed test
pytest --ff | runs from last failed test and continues
pytest --nf | tests new files first and then remaining files
pytest --durations=2 | shows the 2 slowest tests
pytest --durations=2 --vv | shows the 2 slowest tests with actual durations

