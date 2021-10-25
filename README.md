# Pytest notes

## Useful arguments
Command | Result
------------ | -------------
`pytest -x` | exit after first failed test
`pytest -x --pdb` | auto-start debugger at first failure point
`pytest -k pattern` | run all tests matching pattern e.g. `pytest -k equality` would run all tests which have 'equality' in their name, `pytest -k 'equality and not fail'` would run tests that have 'equality' but not 'fail' in their name. `and`, `not` and `or` are allowed keywords, and parentheses are allowed for grouping.
`pytest -v` | instead of greed dots or red F, output the name of the test and PASSED/FAILED. `-vv` gives even more info.
`pytest --lf`	| runs only last failed test
`pytest --ff` | runs from last failed test and continues
`pytest --nf` | tests new files first and then remaining files
`pytest --durations=2` | shows the 2 slowest tests
`pytest --durations=2 --vv` | shows the 2 slowest tests with actual durations
`pytest --sw` | stepwise fail/fix. Use --stepwise-skip if there's something that needs to be skipped until later
`pytest test_file_name.py::test_blah` | run function `test_blah()` in test_file_name.py. Alternately test_file_name.py::TestClass::test_blah. Note test classes should be in format `Test<Something>`.
`pytest --tb=no` | no traceback
`pytest --setup-show` | show progress of setting up fixtures. Note that you'll get messages like `SETUP F fixture name` where `F` means 'Function scope'. [More info on Fixture Scope here](#fixture_scope)


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


## Decorators
### Custom marks
Register custom marks in `pyproject.toml`:

    [tool.pytest.ini_options]
    markers = [
        "slow: marks tests as slow (deselect with '-m \"not slow\"')",
        "serial",
    ]

Then you can mark particular tests with `@pytest.mark.slow` and run with:
`pytest -m slow` - only run slow tests, or
`pytest -m "not slow"` - only run not slow tests

### Other decorators
Decorator | Description
------------ | -------------
@pytest.fixture | [see Fixtures](#fixtures)
@pytest.mark.skip | skip a test
@pytest.mark.skipif | skip a test conditionally
@pytest.mark.xfail | test expected to fail

## Fixtures
### @pytest.fixture
Decorator to mark a fixture factory function.

    @pytest.fixture
    def user_group():
        # Arrange
        # you could set up some other thing here, like grabbing data from a db
        return [User("Bill"), User("Ted")]

    def test_fruit_salad(user_group):
        # Act
        group = Group(*user_group)

        # Assert
        assert all(user.cool for user in group.users)

In this example, `test_fruit_salad` “requests” `fruit_bowl`, and when pytest sees this, it will execute the `fruit_bowl()` fixture function and pass the object it returns into `test_fruit_salad` as the `fruit_bowl` argument.

    import pytest

    @pytest.fixture()
    def temp_jobs_database():
        with TemporaryDirectory() as database_directory:
            db = jobs.jobs_database(Path(database_directory))    #setup...
            yield db                # ... yield back to test_empty_db ...
            db.close()              # ... teardown - close db ...
                                    # ... and remove temporary directory

    def test_empty_db(temp_jobs_database):
        assert jobs_database.count() == 0

*NOTE:* you can do multiple actions on the db returned by the fixture, e.g. adding multiple items, but it's torn down at the end of each `test_...` method.

### Fixture Scope
The default scope for fixtures is the function - so, if you call up a fixture in your function, when the function returns the fixture will be destroyed. Sometimes you don't want to do that, for example if what you're setting up in the fixture takes a while and you want to access it for multiple tests. Easy to do, just use this:

    @pytest.fixture(scope='module')

The complete set of keywords are:
Keyword | Description
------------ | -------------
`function` | default
`class` | run once per test class
`module` | run once per module
`package` | run once per package
`session` | run once per test session


## Useful pytest plugins
Name | Description
------------ | -------------
`pytest-xdist` | splits tests across cores
`pytest-cov` | shows code coverage
`pytest-clarity` | for highlighting unexpected values in dictionaries
`pytest-instafail` | stop on first failure
`pytest-sugar` | errors/fails and progress bar
`pytest-benchmark` | find slowest tests
`pytest-socket` | disables network calls during testing

## Virtual Environment
Should be provided by Poetry, else in your project folder:

    python3 -m venv venv
    .\venv\bin\Activate.ps1

I couldn't get activate.bat to work on Windows.

## Black
Install per this page:
https://black.readthedocs.io/en/stable/integrations/editors.html
After a pip install the path was:
...\Documents\python_testing_with_pytest\venv\bin\black.exe


## Checking for exceptions
If the follow DOES NOT raise an error (TypeError in this case) then the test will fail:

    with pytest.raises(TypeError):
        '2' % 2

Can add more info, e.g. the expected the error message.

    match_regex = "not all arguments converted during string formatting"
    with pytest.raises(TypeError, match=match_regex) as excinfo:
        '2' % 2

## Arrange, Act, Assert
Remember to separate tests into stages, with asserts being at the end. It can be helpful to have a gap between the code blocks, but don't bother commenting each section as the structure will be obvious. Arrange-Act-Assert-Act-Assert ... is an anti-pattern.

## Useful imports
For paths:

    from pathlib import Path

For temp directories:

    from tempfile import TemporaryDirectory
    with TemporaryDirectory() as database_dir:
        database_path = Path(database_directory)

