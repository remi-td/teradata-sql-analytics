# Teradata Data Preparation & Feature Engineering

## Column Selection

### Antiselect — Return All Columns Except Specified
```sql
SELECT * FROM Antiselect(
    ON { db.table | db.view | (query) }
    USING
        Exclude('col1', 'col2')                       -- explicit columns or range e.g. '[1:5]'
) AS t;
```

---

## Normalization / Scaling

### Min-Max Scaling (0 to 1)
```sql
SELECT col,
       (col - MIN(col) OVER ()) / NULLIFZERO(MAX(col) OVER () - MIN(col) OVER ()) AS col_minmax
FROM db.table;
```

### Z-Score Standardization
```sql
SELECT col,
       (col - AVG(col) OVER ()) / NULLIFZERO(STDDEV_SAMP(col) OVER ()) AS col_zscore
FROM db.table;
```

### TD_ScaleFit / TD_ScaleTransform
See the Fit/Transform function section below.

---

## Binning

### Equal-Width Bins (Manual)
```sql
SELECT id, amount,
       CASE
           WHEN amount < 100  THEN 'low'
           WHEN amount < 500  THEN 'medium'
           WHEN amount < 1000 THEN 'high'
           ELSE 'very_high'
       END AS amount_bin
FROM db.table;
```

### Equal-Frequency Bins / Quantile-Based (Manual)
```sql
SELECT id, amount,
       NTILE(4)  OVER (ORDER BY amount) AS quartile,
       NTILE(10) OVER (ORDER BY amount) AS decile
FROM db.table;
```

### TD_BinCodeFit / TD_BinCodeTransform
See the Fit/Transform function section below.

---

## Encoding Categorical Variables

### One-Hot Encoding (Manual)
```sql
SELECT id,
       CASE WHEN category = 'A' THEN 1 ELSE 0 END AS cat_A,
       CASE WHEN category = 'B' THEN 1 ELSE 0 END AS cat_B,
       CASE WHEN category = 'C' THEN 1 ELSE 0 END AS cat_C
FROM db.table;
```

### Label / Ordinal Encoding (Manual)
```sql
SELECT id, category,
       DENSE_RANK() OVER (ORDER BY category) - 1 AS category_encoded
FROM db.table;
```

### TD_OneHotEncodingFit / TD_OneHotEncodingTransform
### TD_OrdinalEncodingFit / TD_OrdinalEncodingTransform
### TD_TargetEncodingFit / TD_TargetEncodingTransform
See the Fit/Transform function section below.

---

## Reshaping

### Unpivot (Wide → Long) — Manual
```sql
SELECT id, 'jan' AS month, jan_sales AS sales FROM db.t
UNION ALL
SELECT id, 'feb', feb_sales FROM db.t
UNION ALL
SELECT id, 'mar', mar_sales FROM db.t;
```

### TD_Unpivoting (Wide → Long)
```sql
-- Basic unpivot
SELECT * FROM TD_Unpivoting(
    ON { db.table | db.view | (query) } AS InputTable PARTITION BY ANY
    USING
        IDColumn('id_col')                            -- required: identifier column
        TargetColumns('jan_sales', 'feb_sales',
                      'mar_sales')                    -- required: columns to unpivot; or range e.g. '[2:4]'
        AttributeColName('month')                     -- optional: name for attribute column
                                                      -- default: 'AttributeName'
        ValueColName('sales')                         -- optional: name for value column
                                                      -- default: 'AttributeValue'
        AttributeAliasList('January', 'February',
                           'March')                   -- optional: friendly names for TargetColumns
                                                      -- must match count of TargetColumns
        Accumulate('region_col')                      -- optional: columns or range to pass through
        IncludeNulls('false')                         -- optional: include NULL values; default: 'false'
) AS t;

-- Advanced options
SELECT * FROM TD_Unpivoting(
    ON { db.table | db.view | (query) } AS InputTable PARTITION BY ANY
    USING
        IDColumn('id_col')
        TargetColumns('col1', 'col2', 'col3')         -- or range e.g. '[1:5]'
        InputTypes('true')                            -- optional: output multiple typed value columns
                                                      -- instead of single AttributeValue column
                                                      -- default: 'false'
        OutputVarchar('true')                         -- optional: cast AttributeValue to VARCHAR
                                                      -- default: 'false'; not used with InputTypes
        IndexedAttribute('false')                     -- optional: use column index instead of name
                                                      -- default: 'false'
        IncludeDataTypes('false')                     -- optional: add column with original data type
                                                      -- default: 'false'
) AS t;
```

### Pivot (Long → Wide) — Manual
```sql
SELECT id,
       SUM(CASE WHEN month = 'jan' THEN sales END) AS jan,
       SUM(CASE WHEN month = 'feb' THEN sales END) AS feb,
       SUM(CASE WHEN month = 'mar' THEN sales END) AS mar
FROM db.long_table
GROUP BY id;
```

### TD_Pivoting (Long → Wide)
```sql
-- Mode 1: PivotColumn — a column contains the pivot keys (most common)
SELECT * FROM TD_Pivoting(
    ON { db.table | db.view | (query) } AS InputTable
        PARTITION BY id_col
        ORDER BY category_col
    USING
        PartitionColumns('id_col')                    -- must match PARTITION BY; list or range
        TargetColumns('value_col')                    -- columns to pivot; list or range
        PivotColumn('category_col')                   -- column whose values become new column headers
        PivotKeys('A', 'B', 'C')                      -- which values to pivot (others ignored)
        PivotKeysAlias('cat_a', 'cat_b', 'cat_c')     -- optional: rename pivot key output columns
        DefaultPivotValues('0', '0', '0')             -- optional: fill value when pivot key is absent
        Accumulate('name_col')                        -- optional: pass-through cols; list or range
        OutputColumnNames('id', 'name', 'cat_a',
                          'cat_b', 'cat_c')           -- optional: rename all output columns
) AS t;

-- Mode 2: RowsPerPartition — no pivot key column; spread N rows into N columns
-- ORDER BY is required to ensure deterministic output
SELECT * FROM TD_Pivoting(
    ON { db.table | db.view | (query) } AS InputTable
        PARTITION BY id_col
        ORDER BY seq_col
    USING
        PartitionColumns('id_col')                    -- list or range
        TargetColumns('value_col')                    -- list or range
        RowsPerPartition(4)                           -- max rows to pivot (1-2047 without aggregation)
                                                      -- NULLs added if partition has fewer rows
                                                      -- extra rows omitted if partition has more
) AS t;

-- Mode 3: Aggregation only — no RowsPerPartition or PivotColumn
SELECT * FROM TD_Pivoting(
    ON { db.table | db.view | (query) } AS InputTable
        PARTITION BY id_col
    USING
        PartitionColumns('id_col')                    -- list or range
        TargetColumns('amount', 'quantity')           -- list or range
        Aggregation('SUM')                            -- single aggregation for all target columns
        -- or per-column: Aggregation('amount:SUM', 'quantity:MAX')
        -- options: CONCAT, UNIQUE_CONCAT, SUM, MIN, MAX, AVG
) AS t;

-- CONCAT/UNIQUE_CONCAT additional options
SELECT * FROM TD_Pivoting(
    ON { db.table | db.view | (query) } AS InputTable
        PARTITION BY id_col
    USING
        PartitionColumns('id_col')                    -- list or range
        TargetColumns('tag_col')                      -- list or range
        Aggregation('CONCAT')
        Delimiters('|')                               -- optional: default ','; or per-column 'tag_col:|'
        CombinedColumnSizes(64000)                    -- optional: default 64000 (VARCHAR)
                                                      -- use >64000 for CLOB output
        TruncateColumns('tag_col')                    -- optional: list or range; truncate if over size
) AS t;
```

---

## Dimensionality Reduction

### TD_RandomProjectionMinComponents
Calculates the minimum number of components required before running `TD_RandomProjectionFit`.
Uses the Johnson-Lindenstrauss Lemma — run this first to estimate the `NumComponents` argument.

```sql
SELECT * FROM TD_RandomProjectionMinComponents(
    ON { db.table | db.view | (query) } AS InputTable
    USING
        TargetColumns('col1', 'col2', 'col3')         -- required: columns for projection; or range
        Epsilon(0.1)                                  -- optional: distortion tolerance, range (0, 1)
                                                      -- higher value = more distortion, fewer components
                                                      -- default: 0.1
) AS t;
-- Pass the output value as NumComponents in TD_RandomProjectionFit
```

### TD_RandomProjectionFit / TD_RandomProjectionTransform
See the Fit/Transform function section below.

---

## Pipeline — Apply Multiple Transforms in One Pass

### TD_ColumnTransformer
Applies multiple Fit/Transform operations to an input table in a single step.
Each FitTable DIMENSION input is optional — include only the transforms needed.
Note: multiple `NonLinearCombineFitTable` instances are allowed; all other FitTable types allow only one.

```sql
SELECT * FROM TD_ColumnTransformer(
    ON { db.table | db.view | (query) } AS InputTable
    ON { db.table | db.view | (query) } AS BincodeFitTable DIMENSION
    ON { db.table | db.view | (query) } AS FunctionFitTable DIMENSION
    ON { db.table | db.view | (query) } AS NonLinearCombineFitTable DIMENSION
    ON { db.table | db.view | (query) } AS OneHotEncodingFitTable DIMENSION
    ON { db.table | db.view | (query) } AS OrdinalEncodingFitTable DIMENSION
    ON { db.table | db.view | (query) } AS OutlierFilterFitTable DIMENSION
    ON { db.table | db.view | (query) } AS PolynomialFeaturesFitTable DIMENSION
    ON { db.table | db.view | (query) } AS RowNormalizeFitTable DIMENSION
    ON { db.table | db.view | (query) } AS ScaleFitTable DIMENSION
    ON { db.table | db.view | (query) } AS SimpleImputeFitTable DIMENSION
    USING
        FillRowIDColumnName('row_id')                 -- optional: adds unique row ID column to output
) AS t;

-- Practical example: impute nulls, scale, and one-hot encode in one operation
SELECT * FROM TD_ColumnTransformer(
    ON db.new_data AS InputTable
    ON db.impute_fit  AS SimpleImputeFitTable DIMENSION
    ON db.scale_fit   AS ScaleFitTable DIMENSION
    ON db.onehot_fit  AS OneHotEncodingFitTable DIMENSION
) AS t;
```

---

## Feature Engineering Patterns

### Lag / Lead Features (Time Series)
```sql
SELECT id, dt, value,
       LAG(value, 1)  OVER (PARTITION BY id ORDER BY dt) AS value_lag1,
       LAG(value, 7)  OVER (PARTITION BY id ORDER BY dt) AS value_lag7,
       LEAD(value, 1) OVER (PARTITION BY id ORDER BY dt) AS value_next
FROM db.timeseries;
```

### Rolling Statistics
```sql
SELECT id, dt, value,
       AVG(value)         OVER (PARTITION BY id ORDER BY dt ROWS BETWEEN 6  PRECEDING AND CURRENT ROW) AS rolling_7d_avg,
       STDDEV_SAMP(value) OVER (PARTITION BY id ORDER BY dt ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS rolling_30d_std
FROM db.timeseries;
```

### Ratio / Interaction Features
```sql
SELECT id,
       revenue / NULLIFZERO(visits)        AS revenue_per_visit,
       clicks  / NULLIFZERO(impressions)   AS ctr,
       (revenue - cost) / NULLIFZERO(cost) AS roi
FROM db.campaign;
```

---

## Fit/Transform Functions

The following functions follow the two-phase Fit/Transform pattern.
See the `fit-transform-pattern` topic for full details on storing and reusing Fit outputs.

| Function Pair | Purpose |
|---|---|
| `TD_BinCodeFit` / `TD_BinCodeTransform` | Bin numeric data into categorical intervals |
| `TD_FunctionFit` / `TD_FunctionTransform` | Apply numeric transformations (log, sqrt, etc.) |
| `TD_NonLinearCombineFit` / `TD_NonLinearCombineTransform` | Generate features from non-linear formulas |
| `TD_OneHotEncodingFit` / `TD_OneHotEncodingTransform` | One-hot encode categorical columns |
| `TD_OrdinalEncodingFit` / `TD_OrdinalEncodingTransform` | Map categories to ordinal values |
| `TD_PolynomialFeaturesFit` / `TD_PolynomialFeaturesTransform` | Generate polynomial feature combinations |
| `TD_RandomProjectionFit` / `TD_RandomProjectionTransform` | Reduce dimensionality via random projection |
| `TD_RowNormalizeFit` / `TD_RowNormalizeTransform` | Normalize input columns row-wise |
| `TD_ScaleFit` / `TD_ScaleTransform` | Scale numeric columns (ZSCORE, RANGE, etc.) |
| `TD_TargetEncodingFit` / `TD_TargetEncodingTransform` | Encode categoricals using target statistics |
