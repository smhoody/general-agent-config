# Google Style Guide - Python
[Source](https://google.github.io/styleguide/pyguide.html)

Place this in a `docs/engineering/coding-standards.md` file in your repo.


## Exceptions
Decision: Allowed but must be used carefully
Follow conditions:
- Make use of built-in exception classes when it makes sense. For example, raise a `ValueError` to indicate a programming mistake like a violated precondition, such as may happen when validating function arguments.
- Do not use `assert` statements in place of conditionals or validating preconditions. They must not be critical to the application logic. A litmus test would be that the `assert` could be removed without breaking the code. `assert` conditionals are not guaranteed to be evaluated. For pytest based tests, `assert` is okay and expected to verify expectations.
**Good**:
```py
if minimum < 1024:
    # Note that this raising of ValueError is not mentioned in the doc
    # string's "Raises:" section because it is not appropriate to
    # guarantee this specific behavioral reaction to API misuse.
    raise ValueError(f'Min. port must be at least 1024, not {minimum}.')
port = self._find_next_open_port(minimum)
if port is None:
    raise ConnectionError(
        f'Could not connect to service on port {minimum} or higher.')
# The code does not depend on the result of this assert.
assert port >= minimum, (
    f'Unexpected port {port} when minimum was {minimum}.')
return port
```
**Bad**:
```py
assert minimum >= 1024, 'Minimum port must be at least 1024.'
# The following code depends on the previous assert.
port = self._find_next_open_port(minimum)
assert port is not None
# The type checking of the return statement relies on the assert.
return port
```
- Never use catch-all statements like `except:` or `except Exception:` unless you are reraising the exception.
- Minimize the amount of code in a `try`/`except` block. The larger the body of the `try`, the more likely that an ecxeption will be raised by a line of code that you didn't expect to raise an exception.

## Comprehensions & Generator Expressions
Decision: Okay to use for simple cases, but not allowed in the case of multiple `for` or `filter` clauses.
**Good**:
```py
result = [mapping_expr for value in iterable if filter_expr]

result = [
    is_valid(metric={'key': value})
    for value in interesting_iterable
    if a_longer_filter_expression(value)
]

descriptive_name = [
    transform({'key': key, 'value': value}, color='black')
    for key, value in generate_iterable(some_input)
    if complicated_condition_is_met(key, value)
]

result = []
for x in range(10):
    for y in range(5):
        if x * y > 10:
            result.append((x, y))

unique_names = {user.name for user in users if user is not None}
```
**Bad**:
```py
result = [(x, y) for x in range(10) for y in range(5) if x * y > 10]

return (
    (x, y, z)
    for x in range(5)
    for y in range(5)
    if x != y
    for z in range(5)
    if y != z
)
```

## Default Argument Values
Decision: Okay in most cases, but do not use mutable object as default values in the function.
**Good**:
```py
def foo(a, b=None):
    if b is None:
        b = []
def foo(a, b: Sequence | None = None):
    if b is None:
        b = []
def foo(a, b: Sequence = ()):  # Empty tuple OK since tuples are immutable.
    ...
```
**Bad**:
```py
from absl import flags
_FOO = flags.DEFINE_string(...)

def foo(a, b=[]): # The list is persisted through multiple function calls.
    ...
def foo(a, b=time.time()):  # Is `b` supposed to represent when this module was loaded?
    ...
def foo(a, b=_FOO.value):  # sys.argv has not yet been parsed...
    ...
def foo(a, b: Mapping = {}):  # Could still get passed to unchecked code.
    ...
```

## Function and Method Decorators
Decision: Use judiciously when there is a clear advantage. Avoid `staticmethod` and limit use of `classmethdo`.
- Should follow the same import and naming guidelines as functions.
- Docstring should clearly state that the function is a decorator.
- Write unit tests for decorators
- Avoid external dependencies in the decorator itself (i.e. don't rely on files, sockets, db connections, etc.) since they might not be available when the decorator runs.
- Should (with valid params, and as much as possible) be gauranteed to succeed in all cases.
- Use `classmethod` only when writing a named constructor, or a class-specific routine that modifies necessary global state such as a process-wide cache.

## Threading
Decision: Encouraged to use when there is a clear benefit.
- Do not rely on atomicity of built-in types. 
- Use the `queue` module's Queue data type as the preferred way to communicate data between threads. Otherwise, use the threaing module and its locking primitives. 
- Prefer condition variables and `threading.Condition` instead of using lower-level locks.

## Power Features
Decision: Avoid
- Python is an extremely flexible language and gives you many fancy features such as custom metaclasses, access to bytecode, on-the-fly compilation, dynamic inheritance, object reparenting, import hacks, reflection (e.g. some uses of `getattr()`), modification of system internals, `__del__` methods implementing customized cleanup, etc.
- Standard library modules and classes that internally use these features are okay (e.g. `abc.ABCMeta`, `dataclasses`, and `enum`).

## Type-annotated Code
Decision: Strongly encouraged
Examples:
```py
def func(a: int) -> list[int]: # Function type hints 
a: SomeType = some_func() # Variable type hints
```

## Style Rules
- Line length: max 80 chars
    - Exceptions:
        - Long import statements
        - URLs, pathnames, or long flags in comments
        - Lint disable comments (e.g.: `# pylint: disable:invalid-name`)
    - Do not use backslash for explicit line continuation, prefer using line joining through parentheses, brackets, and braces.

- Parentheses: Use sparingly
    - **Good**:
    ```py
    if foo:
        bar()
    while x:
        x = bar()
    ```
    - **Bad**:
    ```py
    if (x):
        bar()
    if not(x):
        bar()
    return (foo)
    ```

- Indentation: 4 spaces
    - Never use tabs. 
    - Implied line continuation should align wrapped elements vertically or use a hanging 4-space indent
    - Closing (round, square or curly) brackets can be placed at the end of the expression, or on separate lines, but then should be indented the same as the line with the corresponding opening bracket.
    - **Good**:
    ```py
    foo = long_function_name(var_one, var_two,
                            var_three, var_four)
    meal = (spam,
            beans)

    # Aligned with opening delimiter in a dictionary.
    foo = {
        'long_dictionary_key': value1 +
                                value2,
        ...
    }

    # 4-space hanging indent; nothing on first line.
    foo = long_function_name(
        var_one, var_two, var_three,
        var_four)
    meal = (
        spam,
        beans)

    # 4-space hanging indent; nothing on first line,
    # closing parenthesis on a new line.
    foo = long_function_name(
        var_one, var_two, var_three,
        var_four
    )
    meal = (
        spam,
        beans,
    )

    # 4-space hanging indent in a dictionary.
    foo = {
        'long_dictionary_key':
            long_dictionary_value,
        ...
    }
    ```
    - **Bad**:
    ```py
    # Stuff on first line forbidden.
    foo = long_function_name(var_one, var_two,
        var_three, var_four)
    meal = (spam,
        beans)

    # 2-space hanging indent forbidden.
    foo = long_function_name(
        var_one, var_two, var_three,
        var_four)

    # No hanging indent in a dictionary.
    foo = {
        'long_dictionary_key':
        long_dictionary_value,
        ...
    }
    ```

- Trailing commas: should have their own line, separate from the closing token `]`, `)`, or `}`.
    - **Good**:
    ```py
    golomb3 = [0, 1, 3]
    golomb4 = [
        0,
        1,
        4,
        6,
    ]
    ```
    - **Bad**:
    ```py
    golomb4 = [
        0,
        1,
        4,
        6,]
    ```

- Blank lines: two blank lines between top-level definitions (function or class definitions), one blank line between method definitions and between the docstring of a `class` or the first method. 
    - No blank line following a `def` line
    ` Use single blank lines as you judge appropriate within functions or methods.

- Whitespace
    - No whitespace insite parentheses or braces
        - Good: `spam(ham[1], {'eggs': 2}, [])`
        - Bad: `spam( ham[ 1 ], { 'eggs': 2 }, [ ] )`
    - No whitespace before a comma, semicolon, or colon. No whitespace after a commas, semicolon, or colon, except at the end of the line.
        - Good: `x, y = y, x`
        - Bad: `x , y = y , x`
    - Never use spaces around `=` when passing keyword arguments or defining a default parameter value, EXCEPT when a type annotation is present.
        - Good: `def complex(real, imag=0.0): return Magic(r=real, i=imag)` (no annotation, no space)
        - Good: `def complex(real, imag: float = 0.0): return Magic(r=real, i=imag)` (spacing type annotation present)
        - Bad: `def complex(real, imag = 0.0): return Magic(r = real, i = imag)` (spacing with no type anno.)
        - Bad: `def complex(real, imag: float=0.0): return Magic(r = real, i = imag)` (no spacing with type anno.)
    - Don't use spaces to vertically align tokens on consecutive lines, it becomes a maintenance burden (applies to `:` `#` `=`, etc.)
        - Good:
        ```py
        foo = 1000  # comment
        long_name = 2  # comment that should not be aligned
        ```
        - Bad:
        ```py
        foo       = 1000  # comment
        long_name = 2     # comment that should not be aligned
        ```


## Comments and Docstrings
Decision: encouraged

- Docstrings should be included when they provide additional information. E.g.: providing specifics on how a test should be run or explaining external dependency.
- Examples of a bad docstring (doesn't provide new info): 
    - `"""Tests for foo.bar."""`
    - 
    ```py
    def test_foo_is_barred(self):
        """Tests that bar is applied to foo."""
    ```
- Mandatory for a function (including method, generator, or property) that has one or more of the following properties:
    - being part of the public API
    - nontrivial size
    - non-obvious logic
- Should give enough information to write a call to the function without reading the function's code
    - This applies to non-trivially-sized, ambiguous functions; a function like `get_id_to_name(ids: Sequence[T]) -> Mapping` should be self-explanatory.
- Can be descriptive-style (`"""Fetches rows from a Bigtable."""`) or imperative-style (`"""Fetch rows from a Bigtable."""`), but the style should be consistent within a file.
- Parts of a docstring:
    - `Args`: List each parameter by name. A description should follow the name, and be separated by a colon followed by either a space or newline. If the description is too long to fit on a single 80-character line, use a hanging indent of 2 or 4 spaces more than the parameter name (be consistent with the rest of the docstrings in the file).
    - `Returns`: Describe the semantics of the return value, including any type information that the type annotation does not provide. If the function only returns None, this section is not required. It may also be omitted if the docstring starts with “Return”, “Returns”, “Yield”, or “Yields” (e.g. """Returns row from Bigtable as a tuple of strings.""") and the opening sentence is sufficient to describe the return value.
    - `Raises`: List all exceptions that are relevant to the interface followed by a description. Use a similar exception name + colon + space or newline and hanging indent style as described in Args.
        - Don't document exceptions that get raised if the API specified in the docstring is violated.
- Overridden methods: A method that overrides a method from a base class does not need a docstring if it is explicitly decorated with @override, unless the overriding method’s behavior materially refines the base method’s contract, or details need to be provided (e.g., documenting additional side effects), in which case a docstring with at least those differences is required on the overriding method.
- Classes: Classes should have a docstring below the class definition describing the class. Public attributes, excluding properties, should be documented here in an Attributes section and follow the same formatting as a function’s `Args` section.
    - Example:
    ```py
    class SampleClass:
        """Summary of class here.

        Longer class information...
        Longer class information...

        Attributes:
            likes_spam: A boolean indicating if we like SPAM or not.
            eggs: An integer count of the eggs we have laid.
        """

        def __init__(self, likes_spam: bool = False):
            """Initializes the instance based on spam preference.

            Args:
            likes_spam: Defines if instance exhibits this preference.
            """
            self.likes_spam = likes_spam
            self.eggs = 0

        @property
        def butter_sticks(self) -> int:
            """The number of butter sticks we have."""
    ```
    - All class docstrings should start with a one-line summary that describes what the class instance represents.
- Block and inline comments: should be placed in tricky parts of the code
    - Complicated operations get a few lines of comments before the operations commence. Non-obvious ones get comments at the end of the line.
    - **Good**:
    ```py
    # We use a weighted dictionary search to find out where i is in
    # the array.  We extrapolate position based on the largest num
    # in the array and the array size and then do binary search to
    # get the exact number.

    if i & (i-1) == 0:  # True if i is 0 or a power of 2.
    ```
    - **Bad**:
    ```py
    # Now go through the b array and make sure whenever i occurs
    # the next element is i+1
    ```
- String rules: 
    - Prefer using f-strings for formatting strings
    - A single join with `+` is okay, but do not format with `+`.
    - Avoid using the `+` and `+=` operators to accumulate a string within a loop.  (can lead to quadratic rather than linear running time).
    - Be consistent with your choice of single or double quote character within a file. 
    - Prefer `"""` for multi-line strings rather than `'''`.
    - Multi-line strings do not flow with the indentation of the rest of the program. If you need to avoid embedding extra space in the string, use concatenated single-line strings
        - Good:
        ```py
        long_string = """This is fine if your use case can accept
            extraneous leading spaces."""
        ```
        - Good:
        ```py
        long_string = ("And this is fine if you cannot accept\n" +
                        "extraneous leading spaces.")
        ```
        - Good:
        ```py
        long_string = ("And this too is fine if you cannot accept\n"
                        "extraneous leading spaces.")
        ```
        - Bad:
        ```py
        long_string = """This is pretty ugly.
        Don't do this.
        """
        ```
- Logging:
    - For logging functions that expect a pattern-string (with %-placeholders) as their first argument: Always call them with a string literal (not an f-string!) as their first argument with pattern-parameters as subsequent arguments. Some logging implementations collect the unexpanded pattern-string as a queryable field. It also prevents spending time rendering a message that no logger is configured to output
    - Good:
    ```py
    import os
    from absl import logging

    logging.info('Current $PAGER is: %s', os.getenv('PAGER', default=''))

    homedir = os.getenv('HOME')
    if homedir is None or not os.access(homedir, os.W_OK):
        logging.error('Cannot write to home directory, $HOME=%r', homedir)
    ```
    - Bad:
    ```py
    import os
    from absl import logging

    logging.info('Current $PAGER is:')
    logging.info(os.getenv('PAGER', default=''))

    homedir = os.getenv('HOME')
    if homedir is None or not os.access(homedir, os.W_OK):
        logging.error(f'Cannot write to home directory, $HOME={homedir!r}')
    ```
- Error messages: 
    - Follow these guidelines:
        1. The message needs to precisely match the actual error condition.
        2. Interpolated pieces need to always be clearly identifiable as such.
        3. They should allow simple automated processing (e.g. grepping).
    - Good:
    ```py
    if not 0 <= p <= 1:
        raise ValueError(f'Not a probability: {p=}')

    try:
        os.rmdir(workdir)
    except OSError as error:
        logging.warning('Could not remove directory (reason: %r): %r',
                        error, workdir)
    ```
    - Bad:
    ```py
    if p < 0 or p > 1:  # PROBLEM: also false for float('nan')!
        raise ValueError(f'Not a probability: {p=}')

    try:
        os.rmdir(workdir)
    except OSError:
        # PROBLEM: Message makes an assumption that might not be true:
        # Deletion might have failed for some other reason, misleading
        # whoever has to debug this.
        logging.warning('Directory already was deleted: %s', workdir)

    try:
        os.rmdir(workdir)
    except OSError:
        # PROBLEM: The message is harder to grep for than necessary, and
        # not universally non-confusing for all possible values of `workdir`.
        # Imagine someone calling a library function with such code
        # using a name such as workdir = 'deleted'. The warning would read:
        # "The deleted directory could not be deleted."
        logging.warning('The %s directory could not be deleted.', workdir)
    ```
- TODO Comments: for code that is temporary, a short-term solution, or good-enough but not perfect.
    - Begins with `TODO` in all caps, a parenthesized link to a resource that contains the context, ideally a bug reference. 
    - Examples:
        - Good: `# TODO(example.com/12345): Investigate cpufreq optimizations.`
        - Bad: `TODO(username): File an issue and use a '*' for repetition.`
            - Avoid adding TODOs that refer to an individual or team as context.


## Naming
- Names should be descriptive. This includes functions, classes, variables, attributes, files, and any other named entity.
- Avoid:
    - Abbreviations, especially ones that are ambiguous or unfamiliar to the readers outside the project.
    - Single character names
        - Exceptions: counters (e.g. `i`, `j`, `k`, etc.), `e` for exception identifier in `try`/`except` statements, `f` as file handle in `with` statements, private type-variables with no constraints (`_T = TypeVar("_T)`).
    - Dashes (`-`) in any package/module name
    - `__double_leading_and_trailing_underscore__` names (reserved by Python)
    - names that needlessly include the type of the variable (for example: `id_to_name_dict`)
- Conventions (generally follow PEP 8 naming conventions):
    - Prepending a single underscore (`_`) has some support for protecting module variables and functions (linters will flag protected member access). Note that it is okay for unit tests to access protected constants from the modules under test.
    - Prepending a double underscore (`__` aka “dunder”) to an instance variable or method effectively makes the variable or method private to its class (using name mangling); we discourage its use as it impacts readability and testability, and isn’t really private. Prefer a single underscore.
    - Place related classes and top-level functions together in a module. Unlike Java, there is no need to limit yourself to one class per module.
    - Use CapWords for class names, but lower_with_under.py for module names.
    - New unit test files follow PEP 8 compliant lower_with_under method names, for example, `test_<method_under_test>_<state>`
- File naming:
    - Avoid dashes (`-`), this allows them to be imported and unittested.


## Function Length
- Prefer small and focused functions.
- Long functions are sometimes appropriate, so there is no hard limit on length. 
- If a function exceeds about 40 lines, think about whether it can be broken up without harming the structure of the program.
    - Even if your long function works perfectly now, someone modifying it in a few months may add new behavior. This could result in bugs that are hard to find. Keeping your functions short and simple makes it easier for other people to read and modify your code.

## Type Annotations
General rules:
- Annotate functions and methods that are expected to be used in several areas of the codebase.
- Annotating `self` or `cls` is not necessary. 
- Annotate code that is prone to type-related errors (previous bugs or complexity).
- Annotate code that is hard to understand.

## Line Breaking
In general, follow the indentation rules/patterns of the project.
With type annotation, many function signatures become one param per line:
```py
def my_method(
    self,
    first_var: int,
    second_var: Foo,
    third_var: Bar | None,
) -> int:
  ...
```
Always prefer breaking between variables, and not, for example, between variable names and type annotations. However, if everything fits on the same line, go for it.
```py
def my_method(self, first_var: int) -> int:
  ...
```
Prefer not to break types. However, sometimes they are too long to be on a single line (try to keep sub-types unbroken).
```py
def my_method(
    self,
    first_var: tuple[list[MyLongType1],
                     list[MyLongType2]],
    second_var: list[dict[
        MyLongType3, MyLongType4]],
) -> None:
  ...
```

## NoneType
- NoneType is a “first class” type, and for typing purposes, None is an alias for NoneType. If an argument can be None, it has to be declared! You can use | union type expressions
- Use explicit X | None instead of implicit
- Good: 
```py
def modern_or_union(a: str | int | None, b: str | None = None) -> str:
  ...
def union_optional(a: Union[str, int, None], b: Optional[str] = None) -> str:
  ...
```
- Bad:
```py
def nullable_union(a: Union[None, str]) -> str:
  ...
def implicit_optional(a: str = None) -> str:
  ...
```

## Type Aliases
You can declare aliases of complex types.
- The name of an alias should be CapWorded. If the alias is used only in this module, it should be _Private.
- The alias should be type-annotated as typing.TypeAlias, to distinguish it from a non-type global variable.
- Do not use type aliases as a substitute for the from module import symbol idiom

Good:
```py
from typing import TypeAlias

_LossAndGradient: TypeAlias = tuple[tf.Tensor, tf.Tensor]
ComplexTFMap: TypeAlias = Mapping[str, _LossAndGradient]
```
Bad:
```py
import immutabledict

immutabledict = immutabledict.immutabledict
```

## Tuples vs Lists
Typed lists can only contain objects of a single type. Typed tuples can either have a single repeated type or a set number of elements with different types. The latter is commonly used as the return type from a function.
```py
a: list[int] = [1, 2, 3]
b: tuple[int, ...] = (1, 2, 3)
c: tuple[int, str, float] = (1, "2", 3.5)
```

## Type Variables
The Python type system has generics. A type variable, such as `TypeVar` and `ParamSpec`, is a common way to use them.
```py
from collections.abc import Callable
from typing import ParamSpec, TypeVar
_P = ParamSpec("_P")
_T = TypeVar("_T")
...
def next(l: list[_T]) -> _T:
  return l.pop()

def print_when_called(f: Callable[_P, _T]) -> Callable[_P, _T]:
  def inner(*args: _P.args, **kwargs: _P.kwargs) -> _T:
    print("Function was called")
    return f(*args, **kwargs)
  return inner
```

A `TypeVar` can be constrained.
```py
AddableType = TypeVar("AddableType", int, float, str)
def add(a: AddableType, b: AddableType) -> AddableType:
  return a + b
```

A type variable must have a descriptive name, unless it meets all of the following criteria:
- not externally visible
- not constrained
Good:
```py
_T = TypeVar("_T")
_P = ParamSpec("_P")
AddableType = TypeVar("AddableType", int, float, str)
AnyFunction = TypeVar("AnyFunction", bound=Callable)
```
Bad:
```py
T = TypeVar("T")
P = ParamSpec("P")
_T = TypeVar("_T", int, float, str)
_F = TypeVar("_F", bound=Callable)
```

## Annotating
When annotating function signatures, prefer abstract container types like `collections.abc.Sequence` over concrete types like `list`. If you need to use a concrete type (for example, a `tuple` of typed elements), prefer built-in types like `tuple` over the parametric type aliases from the `typing` module (e.g., `typing.Tuple`).
Bad:
```py
from typing import List, Tuple

def transform_coordinates(original: List[Tuple[float, float]]) ->
    List[Tuple[float, float]]:
  ...
```
Good:
```py
from collections.abc import Sequence

def transform_coordinates(original: Sequence[tuple[float, float]]) ->
    Sequence[tuple[float, float]]:
  ...
```

## Generics
When annotating, prefer to specify type parameters for generic types in a parameter list; otherwise, the generics’ parameters will be assumed to be `Any`.

Good:
```py
def get_names(employee_ids: Sequence[int]) -> Mapping[int, str]:
  ...
```
Bad:
```py
# This is interpreted as get_names(employee_ids: Sequence[Any]) -> Mapping[Any, Any]
def get_names(employee_ids: Sequence) -> Mapping:
  ...
```
If the best type parameter for a generic is `Any`, make it explicit, but remember that in many cases `TypeVar` might be more appropriate
Bad:
```py
def get_names(employee_ids: Sequence[Any]) -> Mapping[Any, str]:
  """Returns a mapping from employee ID to employee name for given IDs."""
```
Good:
```py
_T = TypeVar('_T')
def get_names(employee_ids: Sequence[_T]) -> Mapping[_T, str]:
  """Returns a mapping from employee ID to employee name for given IDs."""
```


## Conclusion
Be consistent!!

If you’re editing code, take a few minutes to look at the code around you and determine its style. If they use _idx suffixes in index variable names, you should too. If their comments have little boxes of hash marks around them, make your comments have little boxes of hash marks around them too.
