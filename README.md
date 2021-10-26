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
`pytest --setup-show` | show progress of setting up fixtures. Note that you'll get messages like `SETUP F fixture name` where `F` means 'Function scope'. [More info on Fixture Scope here](#fixture-scope)
`pytest --fixtures -v` | show available fixtures, the ones in `conftest.py` are at the bottom. The optional `-v` flag gives the filename containing the fixtures.


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

Other monkeypatch options:
`setattr`, `delattr`, `setitem` (for dicts), `delitem`, `setenv` (for environment vars), `delenv`, `syspath_prepend` (for Python import locations), `chdir` (change pwd)

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
        """ The docstring is output if you use pytest --fixtures """
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

*NOTE 1:* you can do multiple actions on the db returned by the fixture, e.g. adding multiple items, but it's torn down at the end of each `test_...` method (if it's the default `function` scope. Code before `yield` is the setup code. Code after `yield` is the teardown code.

*NOTE 2:* fixtures can call other fixtures. For example a db setup fixture with `session` scope could be called by a db reset fixture with `function` scope to keep test independence. Fixtures can only call other fixtures with equal or wider scope. Remember the scope is called out in the pytest output (and helpfully indented) e.g.:

    SETUP    S   outer_fixture
        SETUP    F    inner_fixture
        # do some stuff
        TEARDOWN    F    innerfixture
    TEARDOWN    S    outer_fixture 

*NOTE 3:* functions or fixtures can call multiple other fixtures (looks like multiple inheritance):

    @pytest.fixture(scope="function")
    def do_something_with_multiple_fixtures(fixture1, fixture2):
        # e.g. combine fixture1 and fixture2
        return fixture1

### Fixture Scope
The default scope for fixtures is the function - so, if you call up a fixture in your function, when the function returns the fixture will be destroyed. Sometimes you don't want to do that, for example if what you're setting up in the fixture takes a while and you want to access it for multiple tests. Easy to do, just use this:

    @pytest.fixture(scope='module')

The complete set of scope keywords are:
Keyword | Description
------------ | -------------
`function` | default
`class` | run once per test class
`module` | run once per module
`package` | run once per package - [see conftest.py](#conftest.py)
`session` | run once per test session - [see conftest.py](#conftest.py)

*Note:* tests should not rely on run-order.

### conftest.py
conftest.py is a 'local plugin' and can contain hook functions and fixtures. You need to put your fixture in here if you want it to be shared between multiple modules, as for when fixture scope is at `package` or `session` level. You still have to label the scope in the method signature and you'll have to provide the necessary imports, otherwise it's just a regular file. As an example you could add in the `temp_jobs_database()` method as used above. You don't have to import `conftest.py` into modules that use it - it's automatically imported by pytest when running. If you're using at `package` level you would want to have the file in the package directory.

### Dynamic fixture scoping
Instead of choosing from the default scopes you can set a custom scope by using a flag in the command line e.g.:

   @pytest.fixture(scope=my_custom_scope)
   def blah():
       # blah

`my_custom_scope` is then defined in `conftest.py`:

    def my_custom_scope(fixture_name, config):
        if config.getoption("--use_custom_scope", None):	# supply a command-line arg
            return "function"
        return "session"

To allow us to use the above method we need to provide a hook function:

    def pytest_addoption(parser):
        parser.addoption("--use_custom_scope",
                         action="store_true",
                         default=False,
                         help="reset for each test")

### autouse
You can set `autouse=True` for fixtures that you always want to run (autouse fixtures don't have to be explicitly called by a test):

    @pytest.fixture(autouse=True, scope='session')
    def test_timer():
        """Report elapsed test time"""
        start = time.time()
        yield
        duration = time.time() - start
        print(f'Elapsed time: {duration})

Run pytest with the `-s` flag (shortcut for `--capture=no`) to output print statements even if the test did not fail.

### Built in fixtures
#### tmp_path (function scope)

    def test_temp_path(tmp_path):
        file = tmp_path / "a_file.txt"    # note the slash operator! I think it works because it's a pathlib.Path?

The base directory is called `pytest-XX` where XX is numeric. Pytest only keeps the last few temporary directories and clears out older ones. Specify a custom base directory with `pytest --basetemp=custom_dirname`.

#### tmp_path_factory (session scope)
This is an alternative to `tempfile.TemporaryDirectory`

    def test_temp_path_factory(tmp_path_factory):
        path = tmp_path_factory.**mktemp**("my_temp_dir")
        f = path / "a_file.txt"

#### capsys
Capture writes to stdout and stderr.

    def test_capsys(capsys):
        # do something that outputs to command line e.g. print statement
        output, err = capsys.readouterr()
        assert output.rstrip() == # expected output

#### caplog
Captures logger output

#### monkeypatch
[See Monkeypatching](#monkeypatchin)

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
...\project_folder_path\venv\bin\black.exe


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


## Testing CLI applications
subprocess.run

    def test_cli_output():
        ps = subprocess.run(['app_name', 'command'])

Or, a library called typer has a module typer.testing.CliRunner
see p. 56 onwards of Okken (2021)

## References
Okken, B. (2021) *Python Testing with pytest* Second Edition (pre-print), Raleigh, The Pragmatic Bookshelf