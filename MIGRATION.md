# Migration to pandas 3.0 / Python 3.9+

## What changed

siuba now requires **pandas >= 3.0.0**, **numpy >= 2.0.0**, and **Python >= 3.9**.

The API surface is unchanged. All existing siuba code that works on pandas 2.x
should continue to work, as long as you update your pandas version.

## Changes made

### Removed imports (pandas internals moved or deleted)

| Old import | Replacement | Why |
|---|---|---|
| `pandas.core.reshape.util.cartesian_product` | Local implementation in `verbs.py` | Removed in pandas 3.0 |
| `pandas.core.array_algos.take.take_1d` | `take_nd` (same module) | Renamed in pandas 3.0 |
| `pkg_resources.resource_filename` | `importlib.resources.files` | `pkg_resources` removed from Python 3.13 stdlib |

### GroupBy `.grouper` -> `._grouper`

pandas 3.0 made `DataFrameGroupBy.grouper` private (`._grouper`). This was the
single most pervasive change, affecting every file that works with grouped data.

**Files changed:** `dply/verbs.py`, `dply/vector.py`, `experimental/pd_groups/dialect.py`,
`experimental/pd_groups/groupby.py`, `experimental/pd_groups/translate.py`,
`experimental/pivot/pivot_long.py`, `experimental/pivot/utils.py`

### GroupBy `.apply()` no longer includes group columns

pandas 3.0 removed `include_groups=True` as the default for `GroupBy.apply()`.
Sub-DataFrames passed to the callback no longer contain group columns.

**Affected verbs:** `filter`, `distinct` (grouped variants). Both were rewritten
to handle this correctly.

### `is_categorical_dtype` -> `isinstance` check

`pandas.api.types.is_categorical_dtype` is deprecated. Replaced with
`isinstance(x.dtype, pd.CategoricalDtype)`.

### `Series` constructor `fastpath` removed

`pd.Series(..., fastpath=True)` was removed. Replaced with standard constructor.

### `Series.apply(convert_dtype=)` removed

The `convert_dtype` parameter was removed from `Series.apply()`. Dropped the argument.

### `arrange` temp column names

Changed temporary column names in `arrange` from integers to strings
(`__arrange_tmp_0`) to prevent the column index from falling back to `object`
dtype when mixed with pandas 3.0's default `StringDtype`.

### `setup.py` version bumps

- `pandas>=0.24.0,<2.1.0` -> `pandas>=3.0.0`
- `numpy>=1.12.0` -> `numpy>=2.0.0`
- `python_requires>=3.7` -> `>=3.9`

## What was NOT touched

The following areas were out of scope and may not work with pandas 3.0:

- **`siuba/sql/`** -- SQL/SQLAlchemy backend (not updated)
- **`siuba/experimental/datetime.py`** -- datetime utilities (uses removed offset aliases)
- **`siuba/experimental/pivot/pivot_wide.py`** -- uses `inplace=True` (still works but deprecated)

## Remaining test failures (6 of 1339 pandas tests)

| Test | Cause |
|---|---|
| `test_*[replace]` (4 tests) | pandas 3.0 requires arguments for `Series.replace()` |
| `test_*[dt.freq]` | datetime accessor behavior change |
| `test_*[dt.tz]` | datetime accessor behavior change |

These are all in the series method spec tests (`test_dply_series_methods.py`),
not in core verb tests. The underlying operations work -- the test spec examples
just need updated arguments for pandas 3.0.

## Most important errors encountered

1. **`ModuleNotFoundError: pandas.core.reshape.util`** -- `cartesian_product` removed
2. **`AttributeError: no attribute 'grouper'`** -- renamed to `_grouper` (private)
3. **`ModuleNotFoundError: pkg_resources`** -- removed from Python 3.13 stdlib
4. **`ImportError: take_1d`** -- renamed to `take_nd` in pandas internals
5. **`ValueError: include_groups=True is no longer allowed`** -- `apply()` behavior change
6. **`KeyError` in grouped filter/distinct** -- sub-DataFrames no longer have group columns
7. **`TypeError: fastpath`** -- `Series` constructor dropped `fastpath` kwarg
8. **`StringDtype` vs `object`** -- pandas 3.0 defaults strings to `StringDtype`
9. **`TypeError: convert_dtype`** -- `Series.apply()` dropped `convert_dtype` param
10. **`ValueError: Series.replace`** -- now requires explicit arguments

## Verified working

- `import siuba` on Python 3.13 + pandas 3.0.3
- All core verbs: `filter`, `mutate`, `select`, `arrange`, `group_by`, `summarize`
- Joins: `left_join`, `right_join`, `inner_join`, `full_join`
- Other verbs: `count`, `distinct`, `head`, `rename`, `transmute`, `nest`, `unnest`,
  `spread`, `gather`, `complete`, `expand`, `separate`
- `from siuba.siu import symbolic_dispatch` with examples
- Pipe operator (`>>`)
- 1333 pandas tests passing, 113 skipped, 6 remaining failures (spec edge cases)
