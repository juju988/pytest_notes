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
Override a function of an object such that it returns what you want it to. Example below of overriding a Stripe payment processor so that no actual call is made. `monkeypatch.setattr` arguments are object, method, return value (return value could also be a call to another method which returns something).

    def charge_customer(amount):
        response = Stripe(charge(amount))
        if response.get('status') == 'success':
            current_order_status = 'processing'
        else:
            display_error_message(respone.get('error'))

    def test_payment(monkeypatch):
        monkeypatch.setattr(Stripe, "charge", dict(status="success"))
        charge_customer(199)
        assert current_order_status == 'processing'


## pytest decorators
Register customer marks in `pyproject.toml`:

    [tool.pytest.ini_options]
    markers = [
        "slow: marks tests as slow (deselect with '-m \"not slow\"')",
        "serial",
    ]

Then you can mark particular tests with `@pytest.mark.slow` and run with:
`pytest -m slow` - only run slow tests, or
`pytest -m "not slow"` - only run not slow tests

### @pytest.fixture
Decorator to mark a fixture factory function.

    @pytest.fixture
    def fruit_bowl():
        return [Fruit("apple"), Fruit("banana")]

    def test_fruit_salad(fruit_bowl):
        # Act
        fruit_salad = FruitSalad(*fruit_bowl)

        # Assert
        assert all(fruit.cubed for fruit in fruit_salad.fruit)

In this example, `test_fruit_salad` “requests” `fruit_bowl`, and when pytest sees this, it will execute the `fruit_bowl()` fixture function and pass the object it returns into `test_fruit_salad` as the `fruit_bowl` argument.


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