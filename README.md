[![build status](https://github.com/asottile/cfgv/actions/workflows/main.yml/badge.svg)](https://github.com/asottile/cfgv/actions/workflows/main.yml)
[![pre-commit.ci status](https://results.pre-commit.ci/badge/github/asottile/cfgv/main.svg)](https://results.pre-commit.ci/latest/github/asottile/cfgv/main)

cfgv
====

Validate configuration and produce human readable error messages.

## Installation

```bash
pip install cfgv
```

## Sample error messages

These are easier to see by example.  Here's an example where I typo'd `true`
in a [pre-commit](https://pre-commit.com) configuration.

```
pre_commit.clientlib.InvalidConfigError:
==> File /home/asottile/workspace/pre-commit/.pre-commit-config.yaml
==> At Config()
==> At key: repos
==> At Repository(repo='https://github.com/pre-commit/pre-commit-hooks')
==> At key: hooks
==> At Hook(id='flake8')
==> At key: always_run
=====> Expected bool got str
```

## API

### `cfgv.validate(value, schema)`

Perform validation on the schema:
- raises `ValidationError` on failure
- returns the value on success (for convenience)

### `cfgv.apply_defaults(value, schema)`

Returns a new value which sets all missing optional values to their defaults.

### `cfgv.remove_defaults(value, schema)`

Returns a new value which removes all optional values that are set to their
defaults.

### `cfgv.load_from_filename(filename, schema, load_strategy, exc_tp=ValidationError)`

Load a file given the `load_strategy`.  Reraise any errors as `exc_tp`.  All
defaults will be populated in the resulting value.

Most useful when used with `functools.partial` as follows:

```python
load_my_cfg = functools.partial(
    cfgv.load_from_filename,
    schema=MY_SCHEMA,
    load_strategy=json.loads,
    exc_tp=MyError,
)
```

## Making a schema

A schema validates a container -- `cfgv` provides `Map` and `Array` for
most normal cases.

### writing your own schema container

If the built-in containers below don't quite satisfy your usecase, you can
always write your own.  Containers use the following interface:

```python
class Container(object):
    def check(self, v):
        """check the passed in value (do not modify `v`)"""

    def apply_defaults(self, v):
        """return a new value with defaults applied (do not modify `v`)"""

    def remove_defaults(self, v):
        """return a new value with defaults removed (do not modify `v`)"""
```

### `Map(object_name, id_key, *items)`

The most basic building block for creating a schema is a `Map`

- `object_name`: will be displayed in error messages
- `id_key`: will be used to identify the object in error messages.  Set to
  `None` if there is no identifying key for the object.
- `items`: validator objects such as `Required` or `Optional`

Consider the following schema:

```python
Map(
    'Repo', 'url',
    Required('url', check_any),
)
```

In an error message, the map may be displayed as:

- `Repo(url='https://github.com/pre-commit/pre-commit')`
- `Repo(url=MISSING)` (if the key is not present)

### `Array(of, allow_empty=True)`

Used to nest maps inside of arrays.  For arrays of scalars, see `check_array`.

- `of`: A `Map` / `Array` or other sub-schema.
- `allow_empty`: when `False`, `Array` will ensure at least one element.

When validated, this will check that each element adheres to the sub-schema.

## Validator objects

Validator objects are used to validate key-value-pairs of a `Map`.

### writing your own validator

If the built-in validators below don't quite satisfy your usecase, you can
always write your own.  Validators use the following interface:

```python
class Validator(object):
    def check(self, dct):
        """check that your specific key has the appropriate value in `dct`"""

    def apply_default(self, dct):
        """modify `dct` and set the default value if it is missing"""

    def remove_default(self, dct):
        """modify `dct` and remove the default value if it is present"""
```

It may make sense to _borrow_ functions from the built in validators.  They
additionally use the following interface(s):

- `self.key`: the key to check
- `self.check_fn`: the [check function](#check-functions)
- `self.default`: a default value to set.

### `Required(key, check_fn)`

Ensure that a key is present in a `Map` and adheres to the
[check function](#check-functions).

### `RequiredRecurse(key, schema)`

Similar to `Required`, but uses a [schema](#making-a-schema).

### `Optional(key, check_fn, default)`

If a key is present, check that it adheres to the
[check function](#check-functions).

- `apply_defaults` will set the `default` if it is not present.
- `remove_defaults` will remove the value if it is equal to `default`.

### `OptionalRecurse(key, schema, default)`

Similar to `Optional` but uses a [schema](#making-a-schema).

- `apply_defaults` will set the `default` if it is not present and then
  validate it with the schema.
- `remove_defaults` will remove defaults using the schema, and then remove the
  value it if it is equal to `default`.

### `OptionalNoDefault(key, check_fn)`

Like `Optional`, but does not `apply_defaults` or `remove_defaults`.

### `Conditional(key, check_fn, condition_key, condition_value, ensure_absent=False)`

- If `condition_key` is equal to the `condition_value`, the specific `key`
will be checked using the [check function](#check-functions).
- If `ensure_absent` is `True` and the condition check fails, the `key` will
be checked for absense.

Note that the `condition_value` is checked for equality, so any object
implementing `__eq__` may be used.  A few are provided out of the box
for this purpose, see [equality helpers](#equality-helpers).

### `ConditionalOptional(key, check_fn, default, condition_key, condition_value, ensure_absent=False)`

Similar to ``Conditional`` and ``Optional``.

### `ConditionalRecurse(key, schema, condition_key, condition_value, ensure_absent=True)`

Similar to `Conditional`, but uses a [schema](#making-a-schema).

### `NoAdditionalKeys(keys)`

Use in a mapping to ensure that only the `keys` specified are present.

## Equality helpers

Equality helpers at the very least implement `__eq__` for their behaviour.

They may also implement `def describe_opposite(self):` for use in the
`ensure_absent=True` error message (otherwise, the `__repr__` will be used).

### `Not(val)`

Returns `True` if the value is not equal to `val`.

### `In(*values)`

Returns `True` if the value is contained in `values`.

### `NotIn(*values)`

Returns `True` if the value is not contained in `values`.

## Check functions

A number of check functions are provided out of the box.

A check function takes a single parameter, the `value`, and either raises a
`ValidationError` or returns nothing.

### `check_any(_)`

A noop check function.

### `check_type(tp, typename=None)`

Returns a check function to check for a specific type.  Setting `typename`
will replace the type's name in the error message.

For example:

```python
Required('key', check_type(int))
# 'Expected bytes' in both python2 and python3.
Required('key', check_type(bytes, typename='bytes'))
```

Several type checking functions are provided out of the box:

- `check_bool`
- `check_bytes`
- `check_int`
- `check_string`
- `check_text`

### `check_one_of(possible)`

Returns a function that checks that the value is contained in `possible`.

For example:

```python
Required('language', check_one_of(('javascript', 'python', 'ruby')))
```

### `check_regex(v)`

Ensures that `v` is a valid python regular expression.

### `check_array(inner_check)`

Returns a function that checks that a value is a sequence and that each
value in that sequence adheres to the `inner_check`.

For example:

```python
Required('args', check_array(check_string))
```

### `check_and(*fns)`

Returns a function that performs multiple checks on a value.

For example:

```python
Required('language', check_and(check_string, my_check_language))
```

### `check_or(*fns)`

Returns a function that requires one of the multiple checks to pass.

For example:
```python
Required('language', check_and(check_string, check_int))
```
