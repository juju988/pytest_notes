# Pytest notes

## Useful arguments
Command | Result
------------ | -------------
`pytest -x` | exit after first failed test
`pytest -x --pdb` | auto-start debugger at first failure point. pdb is mainly for remote debugging. Under normal circumstances use PyCharm.
`pytest -k pattern` | run all tests matching pattern e.g. `pytest -k equality` would run all tests which have 'equality' in their name, `pytest -k 'equality and not fail'` would run tests that have 'equality' but not 'fail' in their name. `and`, `not` and `or` are allowed keywords, and parentheses are allowed for grouping. One option could be to have 'todo' in the name of the test and use `-k todo`.
`pytest -v` | instead of green dots or red F, output the name of the test and PASSED/FAILED. `-vv` gives even more info.
`pytest -ra` | output `@pytest.mark.skip` reasons at end of test report. (-r for reason, -a for all except passed)
`pytest --lf`| runs only last failed test
`pytest --ff` | runs from last failed test and continues
`pytest --nf` | tests new files first and then remaining files
`pytest --durations=2` | shows the 2 slowest tests
`pytest --durations=2 --vv` | shows the 2 slowest tests with actual durations
`pytest --sw` | stepwise fail/fix. Use --stepwise-skip if there's something that needs to be skipped until later
`pytest test_file_name.py::test_blah` | run function `test_blah()` in test_file_name.py. Alternately test_file_name.py::TestClass::test_blah. Note test classes should be in format `Test<Something>`.
`pytest --tb=no` | no traceback
`pytest --setup-show` | show progress of setting up fixtures. Note that you'll get messages like `SETUP F fixture name` where `F` means 'Function scope'. [More info on Fixture Scope here](#fixture-scope)
`pytest --fixtures -v` | show available fixtures, the ones in `conftest.py` are at the bottom. The optional `-v` flag gives the filename containing the fixtures.
`pytest --markers` | list the markers in use in the project, including custom ones.
`pytest -n=auto` | parallelise on as many cores as possible
`pytest -s` | show console output even for non-failing tests


## Using a config file
* PEP 518 allowed the use of a `pyproject.toml` file to specify what packages/tools a project needs.
* Many projects (e.g. Black, coverage.py, tox, pytest) allow you to specify their configurations in the `pyproject.toml` file.

## Monkeypatching
Override a function of an object such that it returns what you want it to. `monkeypatch` is a built-in fixture. Example below of overriding a Stripe payment processor so that no actual call is made. `monkeypatch.setattr` arguments are object, method, return value (return value could also be a call to another method which returns something).

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
@pytest.mark.skip | skip a test - see [Markers](#markers) for this and the following two lines
@pytest.mark.skipif | skip a test conditionally
@pytest.mark.xfail | test expected to fail - add flags `(reason='a reason', strict=True)` to raise Error if passes

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
        if config.getoption("--use_custom_scope", None):    # supply a command-line arg
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
`tmp_path` and `monkeypath` are likely the most commonly used built in fixtures.

#### tmp_path (function scope)

    def test_temp_path(tmp_path):
        file = tmp_path / "a_file.txt"    # note the slash operator! I think it works because it's a pathlib.Path?

The base directory is called `pytest-XX` where XX is numeric. Pytest only keeps the last few temporary directories and clears out older ones. Specify a custom base directory with `pytest --basetemp=custom_dirname`.

#### tmp_path_factory (session scope)
This is an alternative to `tempfile.TemporaryDirectory`

    def test_temp_path_factory(tmp_path_factory):
        path = tmp_path_factory.mktemp("my_temp_dir")
        f = path / "a_file.txt"

#### capsys
Capture writes to stdout and stderr.

    def test_capsys(capsys):
        # do something that outputs to command line e.g. print statement
        output, err = capsys.readouterr()
        assert output.rstrip() == # expected output

#### caplog
Captures logger output

There are a bunch more including `cache`, `recwarn` (for testing warning messages), `request` (self-reporting on the running test function), `testdir` (temporary test dir).

#### monkeypatch
[See Monkeypatching](#monkeypatching)

## Useful pytest plugins
Name | Description
------------ | -------------
`pytest-xdist` | splits tests across cores. Use flag `--looponfail` to rerun test if any file changes.
`pytest-cov` | shows code coverage
`pytest-clarity` | for highlighting unexpected values in dictionaries
`pytest-instafail` | stop on first failure
`pytest-sugar` | errors/fails and progress bar
`pytest-benchmark` | find slowest tests
`pytest-socket` | disables network calls during testing
`pytest-randomly` | randomise test order
`pytest-selenium` | easy config of browser testing
`pytest-factoryboy` | database model generator
`pytest-bdd` | bdd-style tests
`pytest-freezegun` | freeze time readings (couldn't you do this with a mock?)

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
Alternately (on another system)
...\project_folder_path\venv\scripts\black.exe

## Checking for exceptions
If the following DOES NOT raise an error (TypeError in this case) then the test will fail:

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

## Parametrize
### Function parameters
Send in a bunch of different parameters. This example could be better, see p. 64 onwards of Okken (2021), for more.

    @pytest.mark.parametrize('user_name, user_availability', 
                                [(val1a, val2a), 
                                (val1b, val2b), 
                                (val1c, val2c)])
    def test_function_parametrization(user_name, user_availability):
        u = User(name=user_name, availability=user_availability)
        id = Users.add(u)
        assert # something or other

The first arg for parametrize are the parameters (comma sep, note the quotes around them), the second arg is a list of tuples (or lists) of values. You can also call other fixtures outside the quotes.

Note that it's better practise to only change the params you're interested in, as the output gives you each parameter that was tested. In the above, if you were only testing different values for `user.availability` it would make sense to set a default `user.name` and only pass in the changing arguments.

You can specify a particular parameter to run by listing it in square brackets e.g.:

    pytest "file_name.py::test_blah[actual parameter]"

I'm not sure how this works but see p. 70 Okken (2021). Tip to use quotes to make sure brackets/spaces don't mess with the parameter.

### Fixture parameters
You can apply params to fixtures too:

    @pytest.fixture(params=['val1', 'val2', 'val3'])
    def do_something(request):
        return request.param

    def test_fixture_parametrization(do_something):
        u = User(state=do_something)
        id = Users.add(u)
        assert # something or other

For every test that uses this fixture, the test will be called anew for each parameter. Note that this uses `return` not `yield` as you (or just me?) might expect.

'same test, different data' - use function param
'same test, different start state' - use fixture param
Okken (2021)

### Improving parameter descriptions
#### With a list of objects
If you pass in a push of different parameters, pytest just labels the output with `param[0]`, `param[1]`, `param[2]`, etc. To get a bit more descriptive you can use a function to provide the output label. You use `ids` to pass in descriptions:

    card_list = [
        Card("foo", state="todo"),
        Card("foo", state="in prog"),
        Card("foo", state="done"),
    ]

    def card_state(card):
        return card.state

    @pytest.mark.parametrize("starting_card", card_list, ids=card_state)
    def test_id_func(cards_db, starting_card):
        ...
        index = cards_db.add_card(starting_card)
        cards_db.finish(index)
        card = cards_db.get_card(index)
        assert card.state == "done"

Note `cards_list` is the abstract set of parameters, and `ids=card_state` pulls the state description from each card. Nice. You can do this with a lambda if you're a pro:

    @pytest.mark.parametrize('starting_card', card_list, ids=lambda c : c.state)

See [Lambdas](#lambdas) for more info. What I'm not sure about is how we're getting the `c` values to pass into the lambda. Is it implied as part of the `pytest.mark.parametrize`?

#### With a dictionary

    text_variants = {
        "Short": "x",
        "With Spaces": "x y z",
        "End In Spaces": "x    ",
        "Mixed Case": "SuMmArY wItH MiXeD cAsE",
        "Unicode": "¡¢£¤¥¦§¨©ª«¬®¯°±²³´µ¶·¸¹º»¼½¾",
        "Newlines": "a\nb\nc",
        "Tabs": "a\tb\tc",
    }


    @pytest.mark.parametrize("variant", text_variants.values(), ids=text_variants.keys())
    def test_summary_variants(cards_db, variant):
        i = cards_db.add_card(Card(summary=variant))
        c = cards_db.get_card(i)
        assert c.summary == variant

Note that as long as no changes are made to the dictionary between calls, Python guarantees that the elements of keys()
and values() will be aligned with each other.

### Hook parameters
See `pytest_generate_tests` - p. 68 Okken (2021). Use if you want to change params with a CLI flag.

## Markers
Any time you want to be able to select a special set of tests you can give them a marker.

### Built in markers

Marker | Description
------------ | -------------
`@pytest.mark.skip(reason=None)` | skip the test (maybe it's still due for completion)
`@pytest.mark.skipif(condition, ..., *, reason)` | condition can be anything that evaluates to True. Any True condition will make the test skip. An example if you want to test different things based on different environment variables.
`@pytest.mark.xfail(condition, ..., *, reason, run=True, raises=None, strict=True)` | test is expected to fail - useful for TDD
`@pytest.mark.filterwarnings(warning)` | adds warning filter to test
`@pytest.mark.parametrize(argnames, argvals)` | [see Function parameters](#function-parameters)
`@pytest.mark.usefixtures(fix1, fix2, ...)` | mark a test as needing this bunch of fixtures

### Custom markers

    @pytest.mark.smoke

Select using `pytest -m smoke`. You can add to the top of a test class as well (which effectively adds it to all methods) as just to a single method. You can apply multiple marks to a method just by stacking them up.

In order to stop pytest complaining about custom markers you have to register them in `pytest.ini`:

    [pytest]
    markers =
        smoke: just some smoke tests - this description can be handy for other devs to see what was intended

Use `pytestmark` to apply the mark to all the tests in a module:

    `pytestmark = pytest.mark.smoke`

or multiple markers

    `pytestmark = [pytest.mark.smoke, pytest.mark.initial]`

You can apply to fixtures:

    @pytest.fixture(params=[pytest.param(marks=pytest.mark.smoke), 'val1', 'val2', 'val3'])
    def do_something(request):
        return request.param

You can run for multiple markers using `and`, `or`, `not` keywords and brackets (remember the quotes):

    pytest -m "smoke and not initial"        

You can add --strict-markers to your pytest.ini files to give quick feedback on errors involving misspelled marks.

    [pytest]
    addopts =
        --strict-markers
        -ra        # recommended

    xfail_strict true    # recommended - report an xfail test that passes as an error

There is a way to pass arguments to fixtures to enable them to do different things - see p. 89 Okken (2021).

## Unittest
Typical set up:

    import unittest

    class TestStringMethods(unittest.TestCase):

        def setUp():
            # same as a fixture in pytest

        def test_upper(self):       # tests start 'test...'
            self.assertEqual('foo'.upper(), 'FOO')

        def tearDown():
            # teardown stuff         

    if __name__ == "__main__":
        unittest.main()             # runs all unittest test cases

There are class and module-level fixture methods (setUpClass, tearDownClass, setUpModule, tearDownModule)

To run:
python -m unittest -v tests/test_module1.TestClass.test_method
-v to increase verbosity

Decorators similar to pytest. Can be applied to classes as well as functions.

Decorator/Marker | Description
------------ | -------------
@unittest.skip | skip
@unittest.skipIf | skipif
@unittest.skipUnless | why not just use skipif?
@unittest.expectedFailure | xfail

Looks like the equivalent of parametrized testing is the `self.subTest`?

### Asserts
Method | Checks that
------------ | -------------
assertEqual(a, b) | a == b
assertNotEqual(a, b) | a != b
assertTrue(x) | bool(x) is True
assertFalse(x) | bool(x) is False
assertIs(a, b) | a is b
assertIsNot(a, b) | a is not b
assertIsNone(x) | x is None
assertIsNotNone(x) | x is not None
assertIn(a, b) | a in b
assertNotIn(a, b) | a not in b
assertIsInstance(a, b) | isinstance(a, b)
assertNotIsInstance(a, b) | not isinstance(a, b)
assertRaises(exc, fun, *args, **kwds) | fun(*args, **kwds) raises exc
assertLogs(logger, level) | The `with` block logs on *logger* with minimum *level*
assertNoLogs(logger, level) | The `with` block does not log on *logger* with minimum *level*
assertAlmostEqual(a, b) | round(a-b, 7) == 0
assertNotAlmostEqual(a, b) | round(a-b, 7) != 0
assertGreater(a, b) | a > b
assertGreaterEqual(a, b) | a >= b
assertLess(a, b) | a < b
assertLessEqual(a, b) | a <= b
assertRegex(s, r) | r.search(s)
assertNotRegex(s, r) | not r.search(s)
assertCountEqual(a, b) | *a* and *b* have the same elements in the same number, regardless of their order.
assertMultiLineEqual(a, b) | strings
assertSequenceEqual(a, b) | sequences
assertListEqual(a, b) | lists
assertTupleEqual(a, b) | tuples
assertSetEqual(a, b) | sets or frozensets
assertDictEqual(a, b) | dicts

### [Mock](https://docs.python.org/3/library/unittest.mock.html) and MagicMock

    from unittest.mock import MagicMock
    thing = ProductionClass()
    thing.method = MagicMock(return_value=3)
    thing.method(3, 4, 5, key='value')
    >> 3

`side_effect` allows raising an exception when a mock is called.

    mock = Mock(side_effect=KeyError('foo'))
    mock()
    >> KeyError: 'foo'

Notice there is a Mock class and a MagicMock class. MagicMock implements a lot of magic methods (the dunder methods) and seems to be the one to use by default, but some people say you should use Mock as you might not want those magic methods to be implemented (so that you know you're testing the right behaviour, not the behaviour of the methods that MagicMock already has implemented).

Here are the magic methods as implemented in MagicMock:

    __lt__: NotImplemented
    __gt__: NotImplemented
    __le__: NotImplemented
    __ge__: NotImplemented
    __int__: 1
    __contains__: False
    __len__: 0
    __iter__: iter([])
    __exit__: False
    __aexit__: False
    __complex__: 1j
    __float__: 1.0
    __bool__: True
    __index__: 1
    __hash__: default hash for the mock
    __str__: default str for the mock
    __sizeof__: default sizeof for the mock


MagicMock - I don't know if Mock does this, but as in the example above you can 'add' methods to MagicMock just by calling them. If you do `dir(thing)` it will now have a method called `method`. Hmm, the normal unittest.Mock does exactly the same thing there. The methods of a method added using Mock and MagicMock also appear to be the same. So for `dir(thing.method)` outputs the same for both Mock and MagicMock:

`['assert_any_call', 'assert_called', 'assert_called_once', 'assert_called_once_with', 'assert_called_with', 'assert_has_calls', 'assert_not_called', 'attach_mock', 'call_args', 'call_args_list', 'call_count', 'called', 'configure_mock', 'method_calls', 'mock_add_spec', 'mock_calls', 'reset_mock', 'return_value', 'side_effect']`

You can add a side_effect by doing:

    thing.method.side_effect = TypeError

So if you then call `thing.method()` an error will be raised. `side_effect` just defines something to be called any time the Mock is called - it can be a function, for raising exceptions, or dynamically changing return values. If you call `type(thing.method)` you get `<class 'unittest.mock.Mock'>`. A `side_effect` can be cleared by setting it to `None`.

Here's the complete signature:
`class unittest.mock.Mock(spec=None, side_effect=None, return_value=DEFAULT, wraps=None, name=None, spec_set=None, unsafe=False, **kwargs)`

`spec` is for an existing object but you can add new methods. `spec_set` is to use an existing object but not allow adding new methods. Setting `name` can be useful for debugging.


### Patch
`unittest.mock.patch(target, new=DEFAULT, spec=None, create=False, spec_set=None, autospec=None, new_callable=None, **kwargs)`

`patch()` can be a function or class decorator, or a context manager. `target` is patched with `new` object. The patch is undone when the function or with statement exits. Note that if `new` is omitted then `target` is replace with `MagicMock` (not `Mock`) (or `AsyncMock` if it's an `async` method). `target` in the format `package.module.ClassName`. If you set `spec=True` the mock will have the methods of the target but you can add new ones, or if `spec_set=True` then you can't add new ones. `autospec=True` **is better**, will copy over the attributes of the `target` too. Methods and functions with `autospec` will have their args checked and will raise `TypeError` if called with the wrong signature.

`patch()` takes arbitrary keyword arguments.

I think the equivalent in pytest is `monkeypatch` - here's an example:

    import pytest_mock

    def test_electric_guitars(guitar_data: List[...], mocker: pytest_mock.MockFixture):
        mocker.patch('get_guitars_from_db', autospec=True, return_value=guitar_data)

So, looks like the `get_guitars_from_db().return_value` attribute is overridden.

#### patch.object
`patch.object(target, attribute, new=DEFAULT, spec=None, create=False, spec_set=None, autospec=None, new_callable=None, **kwargs)`

    @patch.object(SomeClass, 'class_method')
    def test(mock_method):
        SomeClass.class_method(3)
        mock_method.assert_called_with(3)

I don't really understand the example above. What is `mock_method`? So, maybe `object` can be anything? Here's another example, this time patching a dictionary:

    @patch.dict('os.environ', {'newkey': 'newvalue'})
    class TestSample(unittest.TestCase):
        def test_sample(self):
            self.assertEqual(os.environ['newkey'], 'newvalue')

`patcher` objects have `start()` and `stop()` methods, so you can turn off the patch mid-function. Be aware that you have to patch an object where it is looked up, usually the place where it is instantiated.

## Notes on testing
Overall approach is to be very fine detailed. Test one thing at a time.

### Things to test
Think about:
    * non-trivial happy path
    * interesting inputs e.g. 0, 1, >1, <1, huge numbers, optional inputs, empty inputs, invalid inputs. Do you allow duplicate objects? May need to define what a duplicate is.
    * error states and error handling. What specific Exceptions will be raised? What are the logging levels?
    * interesting end states, such as an error, or an empty database

In delving deeper it might raise questions about desired behaviour which is a great time to get feedback from the team.

Write down a **Testing Strategy** before you get started on testing. Easy to get lost in the details. Have somewhere you can share that with the team (maybe on the story?).

### Config files
`pytest.ini`: primary pytest config file, also defines pytest root dir
`conftest.py`: fixtures and hook functions
`__init__.py`: including in a folder allows test to have same name as in another folder. Good habit to have __init__.py in each folder so collisions don't come up.
`tox.ini`: alternative to pytest.ini
`pyproject.toml`: alternative to pytest.ini - **used by Poetry**. To use Black you need to configure here.
`setup.cfg`: alternative to pytest.ini

If you launch pytest in a folder that doesn't contain a config file it will look up in the next directory recursively until it finds a config file. If it doesn't find one it won't run unless you've specified the start folder. When it finds a config file that's the root directory (`rootdir`). So best to have a pytest.ini at the root of your project even if you don't have any settings in it, just to prevent pytest from running up further and accidentally picking up the config file for another project. You can have a different conftest.py file in each folder if you want, for defining different fixtures/hooks. Lower conftest.py files inherit from ones further up the directory structure, may be better to stick to one though for simplicity.

#### pytest.ini

    [pytest]
    addopts =
        --strict-markers        # don't allow typos in markers
        --strict-config         # don't skip over difficulties loading confg
        -ra                     # display extra info on failures/errors at end of test run
        -v                      # make verbose

    testpaths = tests           # where to look for tests
    pythonpath = src            # where to look for code to be tested (if it's in /src)

    markers =
        smoke: subset of tests
        exception: check for expected exceptions

#### pyproject.toml

    [tool.pytest.ini_options]
    addopts = [
        "--strict-markers",
        "--strict-config",
        "-ra"
        ]

    testpaths = "tests"
    pythonpath = "src"

    markers = [
        "smoke: subset of tests",
        "exception: check for expected exceptions"
    ]

#### setup.cfg

    [tool:pytest]
    addopts =
        --strict-markers
        --strict-config
        -ra

    testpaths = tests
    pythonpath = src

    markers =
        smoke: subset of tests
        exception: check for expected exceptions

## Coverage
coverage.py measures code coverage. pytest-cov is a plugin that makes it slightly easier to use.

    pip install coverage
    pip install pytest-cov

    pytest --cov=name_of_folder name_of_subfolder       # space separated.
    pytest --cov=some_code some_code/test_some_code.py  # test code in 'some_code' with 'test_some_code.py'
    pytest --cov=some_code some_code                    # equiv to above line because 'test_some_code.py' is the only file in the directory starting with 'test_'.

For a single file (that also contained test_ functions within it) this worked:

    pytest --cov=single_file single_file.py

This correctly doesn't expect there to be tests for the test_ functions.

Coverage has a config file too, called `.coveragerc` (in the root folder of the whole project) which should point to the code under test (I think):

    [paths]
    source =
        cards_proj/src/cards
        */site-packages/cards

Here's the description in Okken (2021) which I don't understand:
"This file is the coverage.py configuration file, and the source setting tells coverage
to treat the cards_proj/src/cards directory as if it’s the same as the installed cards
within */site-packages/cards. The asterisk (*) is a wildcard there to save us a bit
of typing. You can type out the whole /path/to/venv/lib/python3.x/site-packages/cards
path if you wish."

Ah, OK this is very cool - find lines that are not covered by tests:

    coverage report --show-missing

    ...
    venv\lib\python3.9\site-packages\cards\api.py           72      3    96%   72, 76, 78
    venv\lib\python3.9\site-packages\cards\cli.py           86     53    38%   17, 24-26, 31
    -35, 45-59, 69-74, 80-84, 90-94, 100-101, 107-108, 116-117, 120-125, 130-133
    ...

To view that report in html format:

    coverage html

Ohhh - OK now that's super cool! You can click into the files and see exactly which lines are not called by the tests.

You can exclude certain lines from testing if absolutely not needed by marking the block like this:

    if __name__ == '__main__':  # pragma: no cover
        main()
    else:
        import pytest           # this allows parametrization and markers

Here's a breakdown of commands:
`pytest --cov=cards <test path>`: run with a simple report.
`pytest --cov=cards --report=term-missing <test path>`: which lines weren’t run.
`pytest --cov=cards --report=html <test path>`:  generate an HTML report.
`coverage run --source=cards -m pytest <test path>`: run the test suite with coverage.
`coverage report`: show a simple terminal report.
`coverage report --show-missing`: show which lines weren’t run.
`coverage html`: generate an HTML report.

## Context Manager
Just a nice way to get access to the flat file db:

    from contextlib import contextmanager
    @contextmanager
        def cards_db():
        db_path = get_path()
        db = cards.CardsDB(db_path)
        yield db
        db.close()

Within the code, whenever the db is required, just use:

    with cards_db() as db:
        # do whatever with db

Lovely!

## Mocking
See Chapter 10 of Okken (2010). `patch` is the workhorse of mocking. Here's an example:

    def test_mock_version():
        with mock.patch('cards.cli.cards') as mock_cards:
            mock_cards.__version__ = '1.2.3'
            result = runner.invoke(app, ["version"])
            assert result.stdout.rstrip() == '1.2.3'

Using the `with` context manager clears up the patch once done. I guess it would be cleared anyway though once function exited? Note in Okken (2021) that the weird part is you're patching `cards.cli.cards` rather than `cards`. You have to patch what the CLI will be seeing. `cards.cli` imports `cards` so that's where you have to do the patching.

Mocking stuff that returns something is quite intuitive, as per the above example. But how do you test a function that doesn't return anything? In this case you can make an assert that it was called correctly using `object.method.assert_called_with(args)`:

    @pytest.fixture()
    def mock_cardsdb():
        with mock.patch("cards.cli.cards.CardsDB", autospec=True) as CardsDB:
            yield CardsDB.return_value

    def test_add_with_owner(mock_cardsdb):
        cards_cli("add some task -o brian")
        expected = cards.Card("some task", owner="brian", state="todo")
        mock_cardsdb.add_card.assert_called_with(expected)

The above will fail if the method was called with the wrong arguments, even the wrong type. There are quite a few variants of [`assert_called()`](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.assert_called).

**BEWARE!** - when you are using mocks you are testing IMPLEMENTATION, not BEHAVIOUR. Therefore it is best to limit mocking to those cases where you want to raise an Exception (using `mock_object.side_effect = Exception`), or if it's difficult to simulate the tested behaviour any other way, for example testing a payment processor without an actual payment being involved, or where you want to check some interim class is passing along messages correctly.

There is such a thing as *Designing for Testability*, where you could for example emit some output (a `print` statement maybe) to confirm that an action was carried out, rather than returning nothing.

Here are a couple of exercises from Okken (2021) Chapter 10 which it took me a while to get my head around:

1. In test_my_info.py, write a test that uses mock and changes the return value of Path.home() to "/users/fake_user", and checks the return value of home_dir().
    
    import my_info

    def test_my_home_returns_correct_value():
        with mock.patch('my_info.home_dir', autospec=True) as mock_home_dir:
            mock_home_dir.return_value = "/users/fake_user"
            value = my_info.home_dir()
            assert value == "/users/fake_user"

Notes: first problem was that because `my_info` is imported I thought I could just refer to `home_dir` (e.g. `with mock.patch('home_dir')`) but you have to use the full dotted path, i.e. `my_info.home_dir`. Just add `as mock_...` to make the name clear. Then you specify what you want the mock method to return with `return_value`. Then call the actual method and the mock method will be called instead.

2. Write another test that also calls home_dir(), but instead of checking the value, just asserts that Path.home() is called by home_dir().

    def test_my_home_is_called():
        with mock.patch('my_info.Path.home', autospec=True) as mock_path:
            my_info.home_dir()
            mock_path.assert_called_once()
        # check to see if Path.home() was called

Notes: again had a mental block with what to patch - it turned out to be the full dotted path through to the `Path.home()` method. Again, call the actual method and the patched-in object will be called.

## Continuous Integration
[`tox`](https://tox.readthedocs.io/en/latest) is a CI tool that can be run locally and in conjunction with other CI tools on a server. You can use `tox` to test your app in multiple versions of Python. I don't really want to do that right now so skipping that. This is Okken (2021) chapter 11.

Well, actually, you don't HAVE to test in multiple Python versions, you can just stick to one and `tox` can do other stuff.

Here's an example tox.ini file:

    [tox]
    envlist = py308
    isolated_build = True
    
    [testenv]
    deps =
        pytest
        faker
        pytest-cov
    commands = pytest --cov=cards --cov=tests --cov-fail-under=100

Optional commands:

    --cov=tests: run all tests
    --cov-fail_under=X: fail if < X% coverage
    {posargs}: allow to send in pytest arguments (- )could be useful if you just want to test one file) separate from tox args with --, e.g.:
        $ tox -c tox_posargs.ini -e py310 -- -k test_version --no-cov

Here's another example, this time setting up a `[testenv]`:

    [tox]
    envlist = py39, py310
    skipsdist = true
    
    [testenv]
    deps = pytest
           pytest-srcpaths
           -rrequirements.txt       # installs the deps in requirements.txt
    commands = pytest
    
    [pytest]
    addopts = -ra
    testpaths = tests
    pythonpath = src

Recommended that you also have a .coveragerc file if using with tox to tell coverage "which source paths should be considered identical". Oh, OK this is because tox creates a virtual environment with a copy of the code at .tox/py3028..../site_packages...

    [paths]
    source =
        src,
        .tox/*site-packages

Run like `tox -c tox_coverage.ini -e py308`

If you run `tox` on the cmd line it will install the envlist, then the dependencies, and then will run pytest.

### Tox with GitHub Actions
You need a `.github/workflows/main.yml` file at the top of your project. See [Buiding and Testing Python](https://docs.github.com/en/actions/guides/building-and-testing-python):

    name: Python package                # can be anything
    on: [push, pull_request]            # when to run test
    jobs:                               # note the indentation is 2 spaces
      build:
      runs-on: ubuntu-latest
      strategy:
        matrix:
          python: ["3.8", "3.9"]
        
      steps:                            # steps to run
        - uses: actions/checkout@v2     # the GitHub Actions tool that checks out our code into a temp workspace
        - name: Setup Python
          uses: actions/setup-python@v2 # another GitHub Actions tool to set up Python
          with:
            python-version: ${{ matrix.python }}
        - name: Install Tox and any other packages
          run: pip install tox          # install tox
        - name: Run Tox
          run: tox -e py                # runs tox

Once the .yml file is pushed to GitHub it will be automatically run.

## Testing Scripts and Applications
A chapter for scripts that are not installable via pip.

### Subprocess
For grabbing output to console:

    import subprocess
    def test_hello():
        command = ["python", "hello.py"]
        pipe = subprocess.run(command, stdout=subprocess.PIPE)
        output = pipe.stdout.decode('utf8')
        assert output == "Hello, World!\n"

Auto-running tests with apps that don't need building (no setup.py file), in tox.ini:      

    [tox]
    envlist = py39, py310
    skipsdist = true        # means tox won't try to build it
    
    [testenv]
    deps = pytest
    commands = pytest

For testing from 'outside' you can always add the line:
  
    ... rest of code ...

    if __name__ == '__main__':
        main()

Adding that doesn't affect importing it into anything else, but it means you can launch it individually with tests, e.g. if the code above was in a file called `hello.py`, you could create a file `test_hello.py`:

    import hello
    def test_main(capsys):      # capsys is a built-in pytest fixture
        hello.main()
        output, _ = capsys.readouterr()
        assert output == "Hello, World!\n"

Or even just test other functions directly. I guess you only need `main()` if you want to test it running.

### Separating tests from code to be tested
See #pytest.ini - you can set the test and src directories. You can get the list of search paths (to check you're looking in the right place) with:

    import sys
    for p in sys.path:
        print(p)

## Python debugger (pdb)
Useful for remote debugging if terminal only. Launch with:

    pytest --pdb

If any assertions fail the code will open a terminal with `(Pdb)` - once in this state use the following commands:

Command | Long form + Description
------------ | -------------
l | 'list' - lists the 11 lines surrounding the failed assertion. Also `l first, last` lists code between `first` and `last` line numbers.
ll | list all lines in current function
q | quit
w | 'where' - prints stack trace
p expr | 'print' - evaluates expression
pp expr | pretty print evaluated expression
a | 'args' - prints the argument list of the expression
s | 'step' - executes current line and goes to next
n | 'next' - executes current line and steps to next in current function
r | 'return' - continue until current function returns
c | 'continue' - until next breakpoint
unt line | 'until' - continues until given line number

## Lambdas
Common applications of lambdas are sorting and filtering data. Some lambda examples follow, to try to get them into my wooden head:

    @pytest.mark.parametrize('starting_card', card_list, ids=lambda c : c.state)

... but where are we picking up the value for each `c`? OK I think it's from the implementation of parametrize where we have the following:

    ids: Optional[
        Union[
            Iterable[Union[None, str, float, int, bool]],
            Callable[[Any], Optional[object]],
        ]

The `typing.Callable` type is an ABC that just requires the `__call__` method, so a lambda or a function would work there. OK honestly I don't really know how it works or how you would derive the fact that you can pass in a lambda. Somehow the list `card_list` is pushed into the `ids` such that each item in the list is processed by whatever `ids` points to.

General syntax:

    lambda arg1, arg2, arg3: expression

Note there are no brackets around arguments. As long as you can do an inline expression you're golden.

### More lambda examples

    items = [1, 2, 3, 4]
    sum = reduce(lambda x, y: x + y, items)     # 10 - note x, y are taking the first 2 elements of `items` at a time
    (lambda x: 3*x+1)(3)                        # 10

The problem is that lambdas are anonymous functions so how do you get information to them? You can give them a name, e.g.:

    l = lambda x: 3*x+1
     [l(x) for x in items]  # [4, 7, 10, 13]

Note that you can supply as many arguments (x, y, z ...) as you like to a lambda, even 0 arguments!

    scifi_authors = ['Isaac Asimov', 'Frank Herbert', 'Leigh Brackett']
    sorted(scifi_authors, key=lambda x: x.split()[-1])  # ['Isaac Asimov', 'Leigh Brackett', 'Frank Herbert']

## Map, Filter, Reduce
All work on there being a function followed by a list of inputs.

    items = [1, 2, 3, 4]
    squared = list(map(lambda x: x*x))                  # [1, 4, 9, 16]
    evens = list(filter(lambda x: x%2 == 0, items))     # [2, 4]

    from functools import reduce 
    sum = reduce(lambda x, y: x * y, items)             # 24

Note: `reduce` always takes 2 arguments because it's doing (((1 * 2) * 3) * 4)

## List Comprehensions
Generally in the format of project, source, filter:

    usernames = [u.name for u in get_active_customers() if u.last_purchase == 'today']
                |       |                               |
                |       source ------------------------ filter -----------------------
                projection

## References
Okken, B. (2021) *Python Testing with pytest* Second Edition (pre-print), Raleigh, The Pragmatic Bookshelf
Python Software Foundation (2021) *unittest — Unit testing framework* https://docs.python.org/3/library/unittest.html
Python Software Foundation (2021) *Quick Guide* https://docs.python.org/3/library/unittest.mock.html
