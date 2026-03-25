# Teradata Vector Search

Functions for computing vector distances and building approximate nearest-neighbor indexes. Used in retrieval-augmented generation (RAG), semantic search, recommendation, and anomaly detection pipelines.

> **Pre-processing tip:** Normalize embeddings to unit length with `TD_VectorNormalize(Approach('UNITVECTOR'))` before computing cosine distances or building an HNSW index. Unit-normalized vectors make cosine distance equivalent to dot product distance and improve index quality. See `data-prep` topic.

---

## TD_VectorDistance

Computes pairwise distances between vectors in a target table and vectors in a reference table. Supports three distance measures, optional TopK filtering, and both individual float columns and `VECTOR`/`Vector32`/`VARBYTE` column types.

```sql
SELECT * FROM TD_VectorDistance(
    ON { db.table | db.view | (query) } AS TargetTable PARTITION BY ANY
    [ ON { db.table | db.view | (query) } AS ReferenceTable DIMENSION ]
    USING
        TargetIDColumn('target_id_col')                                    -- required
        TargetFeatureColumns({ 'col' | col_range }[,...])                  -- required; see column types below
        [ RefIDColumn('ref_id_col') ]                                      -- required when ReferenceTable provided
        [ RefFeatureColumns({ 'col' | col_range }[,...]) ]                 -- required when ReferenceTable provided
        [ DistanceMeasure({ 'Cosine' | 'Euclidean' | 'Manhattan' }[,...]) ]  -- default: all three
        [ TopK(integer_value) ]                                            -- default 1024; -1 = all pairs
        [ UseSIMD('false') ]                                               -- default false; enable for performance
) AS t;
```

**TargetFeatureColumns / RefFeatureColumns accepted types:**
- Individual numeric columns: `BYTEINT`, `SMALLINT`, `INTEGER`, `BIGINT`, `DECIMAL`, `FLOAT`, `REAL`, `DOUBLE PRECISION`
- Single `VECTOR` or `Vector32` UDT column
- Single `VARBYTE` column

> **Column order and type must match** between `TargetFeatureColumns` and `RefFeatureColumns`. Maximum 2018 feature columns when using individual numeric columns.

**Output columns:**

| Column | Type | Description |
|--------|------|-------------|
| `Target_ID` | BIGINT | Identifier from `TargetIDColumn` |
| `Reference_ID` | BIGINT | Identifier from `RefIDColumn` |
| `DistanceType` | VARCHAR | Distance measure: `cosine`, `euclidean`, or `manhattan` |
| `Distance` | DOUBLE PRECISION | Distance between the target and reference vectors |

**Distance measure guide:**

| Measure | Range | Interpretation | Best for |
|---------|-------|----------------|----------|
| `Cosine` | [0, 2] | 0 = identical direction, 1 = orthogonal, 2 = opposite | Embeddings (direction matters, not magnitude); normalize first |
| `Euclidean` | [0, ∞) | Straight-line distance in feature space | Spatial data, clusters |
| `Manhattan` | [0, ∞) | Sum of absolute differences per dimension | Sparse features |

---

### Usage patterns

**Semantic search — find top K most similar documents to a query vector:**

```sql
-- query_embedding is a single-row table with the query vector
SELECT * FROM TD_VectorDistance(
    ON db.document_embeddings AS TargetTable PARTITION BY ANY
    ON db.query_embedding AS ReferenceTable DIMENSION
    USING
        TargetIDColumn('doc_id')
        TargetFeatureColumns('embedding')    -- VECTOR column
        RefIDColumn('query_id')
        RefFeatureColumns('embedding')
        DistanceMeasure('Cosine')
        TopK(10)
) AS t
ORDER BY Distance ASC;
```

**All-pairs distance within a single table (no ReferenceTable):**

```sql
-- Omitting ReferenceTable computes distances between all pairs in TargetTable
SELECT * FROM TD_VectorDistance(
    ON db.item_embeddings AS TargetTable PARTITION BY ANY
    USING
        TargetIDColumn('item_id')
        TargetFeatureColumns('embedding')
        DistanceMeasure('Cosine')
        TopK(5)    -- return 5 nearest neighbors per item
) AS t;
```

**Multiple distance measures in one call:**

```sql
SELECT * FROM TD_VectorDistance(
    ON db.vectors AS TargetTable PARTITION BY ANY
    ON db.refs AS ReferenceTable DIMENSION
    USING
        TargetIDColumn('id')
        TargetFeatureColumns('feat1', 'feat2', 'feat3')   -- individual float columns
        RefIDColumn('ref_id')
        RefFeatureColumns('feat1', 'feat2', 'feat3')
        DistanceMeasure('Cosine', 'Euclidean')            -- both measures in one pass
        TopK(20)
        UseSIMD('true')
) AS t;
```

> **TopK(-1):** returns all N² pairs — use only on small tables. For large-scale nearest-neighbor search, use `TD_HNSW` instead (approximate, but orders of magnitude faster). See below.

---

## TD_HNSW / TD_HNSWPredict / TD_HNSWSummary

*Documentation in progress — paste TD_HNSW docs to continue.*
