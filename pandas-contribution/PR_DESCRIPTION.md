# BUG: DataFrame.duplicated() loses index / raises ValueError when DataFrame has no columns

## Problem

Two related bugs affect `DataFrame.duplicated()` when there are no columns in scope.

### Bug 1 — DataFrame with rows but zero columns returns empty Series

```python
import pandas as pd

df = pd.DataFrame(index=[10, 20, 30])   # rows but no columns
df.duplicated()
```

**Actual output (wrong):**
```
Series([], dtype: bool)    # ← index lost, values missing
```

**Expected output:**
```
10    False
20     True
30     True
dtype: bool
```

### Bug 2 — `subset=[]` raises `ValueError`

```python
df = pd.DataFrame({'a': [1, 1, 2], 'b': [4, 4, 6]}, index=[100, 200, 300])
df.duplicated(subset=[])
```

**Actual output (wrong):**
```
ValueError: not enough values to unpack (expected 2, got 0)
```

**Expected output:**
```
100    False
200     True
300     True
dtype: bool
```

---

## Root Cause

Both bugs share the same origin. In `pandas/core/frame.py`, the early-return guard uses:

```python
if self.empty:
    return self._constructor_sliced(dtype=bool)
```

`DataFrame.empty` returns `True` whenever **any** axis has length 0 — including the column axis. A DataFrame like `pd.DataFrame(index=[10, 20, 30])` has three rows but zero columns, so `.empty` is `True`, causing `duplicated()` to return a vacuous empty Series before even inspecting the rows.

The second bug is triggered when `subset` resolves to an empty sequence (either because the DataFrame has no columns, or because the caller explicitly passes `subset=[]`). The code then reaches:

```python
labels, shape = map(list, zip(*map(f, vals), strict=True))
```

With an empty `vals`, `zip(strict=True)` of nothing produces an empty iterator that cannot be unpacked into two names → `ValueError`.

---

## Fix

**Change 1** — restrict early-exit to truly row-less DataFrames:

```python
# Before:
if self.empty:

# After:
if len(self) == 0:
```

**Change 2** — handle `len(subset) == 0` before the `zip(strict=True)` line:

```python
if len(subset) == 0:
    # No columns to distinguish rows; every row is identical to every other.
    nrows = len(self)
    if nrows <= 1:
        rv = np.zeros(nrows, dtype=bool)     # single row is never a duplicate
    elif keep == "first":
        rv = np.ones(nrows, dtype=bool); rv[0]  = False
    elif keep == "last":
        rv = np.ones(nrows, dtype=bool); rv[-1] = False
    else:   # keep is False
        rv = np.ones(nrows, dtype=bool)
    return self._constructor_sliced(rv, index=self.index).__finalize__(
        self, method="duplicated"
    )
```

---

## Tests

New tests added in `pandas/tests/frame/methods/test_duplicated_no_columns.py`:

| Test class | What it covers |
|---|---|
| `TestDuplicatedNoColumns` | DataFrame with rows but zero columns — all three `keep=` modes, int/str/float/MultiIndex, single-row edge case, fully-empty DataFrame |
| `TestDuplicatedEmptySubset` | `subset=[]` on a DataFrame that has columns — all three `keep=` modes, index preservation, no `ValueError` |
| `TestDuplicatedRegressions` | All pre-existing normal behaviour is unchanged |

---

## Checklist

- [x] Tests added for the new behaviour
- [x] Tests added for the regression (all existing behaviour passes)
- [x] No change in behaviour for DataFrames with columns and a non-empty subset
- [x] Consistent with `Index.duplicated()` semantics
- [x] Added to whatsnew (Bug Fixes section)

---

## Whatsnew Entry

Add to `doc/source/whatsnew/v3.1.0.rst` (or next patch release):

```rst
Bug Fixes
~~~~~~~~~
- Bug in :meth:`DataFrame.duplicated` where calling the method on a
  :class:`DataFrame` with rows but no columns (or with ``subset=[]``)
  returned an empty :class:`Series` or raised ``ValueError`` instead of
  correctly marking each row as a duplicate of every other row
  (:issue:`XXXXX`)
```
