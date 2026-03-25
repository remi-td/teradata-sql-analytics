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

## Approximate Nearest Neighbor — HNSW

For large-scale similarity search, `TD_VectorDistance` with `TopK(-1)` computes exact distances but scales as O(N²). The HNSW family provides approximate nearest-neighbor search that scales to millions of vectors.

| Function | Role |
|----------|------|
| `TD_HNSW` | Build the HNSW graph index from input vectors; supports incremental Update and Delete |
| `TD_HNSWPredict` | Search the index — returns approximate top-K nearest neighbors for query vectors |
| `TD_HNSWSummary` | Inspect a built index — layer counts, node counts, graph statistics |

---

### TD_HNSW — Build Index

Constructs a Hierarchical Navigable Small World graph from input vectors. The graph is stored in the `OUT TABLE` as packed binary rows (`TD_HNSW_Nodes`). The primary output is a single status message row.

> **Always use `OUT TABLE`** — the primary output is only a status message. The graph itself is in the OUT TABLE.

```sql
-- Build (training operation)
SELECT * FROM TD_HNSW(
    ON { db.table | db.view | (query) } AS InputTable  -- no PARTITION BY, or PARTITION BY ANY [ORDER BY col]
    OUT [ PERMANENT | VOLATILE ] TABLE ModelTable(db.hnsw_model)  -- required; stores the graph
    USING
        IdColumn('id_col')                             -- required; unique row identifier
        VectorColumn('embedding_col')                  -- required; VECTOR, Vector32, VARBYTE, or BYTE column
        [ DistanceMeasure('euclidean'|'cosine'|'dotproduct') ]  -- default 'euclidean'; must match TD_HNSWPredict
        [ EfConstruction(32) ]                         -- neighbors searched during build; range [1, 1024]
        [ NumConnPerNode(32) ]                         -- connections added per new node; range [1, 1024]
        [ MaxNumConnPerNode(32) ]                      -- max connections per existing node; range [1, 1024]
        [ NumLayer(n) ]                                -- default computed from NumConnPerNode; range [1, 1024]
        [ EmbeddingSize(n) ]                           -- default computed from input data
        [ ApplyHeuristics('false') ]                   -- default false; true improves graph quality, increases build time
        [ Seed(seed_value) ]                           -- default random; set for reproducible graph
        [ UseSIMD('false') ]                           -- default false; enable for SIMD acceleration
) AS t;
```

**Output (primary):**

| Column | Type | Description |
|--------|------|-------------|
| `message` | VARCHAR(128) | Build status message |

**OUT TABLE schema:**

| Column | Type | Description |
|--------|------|-------------|
| `TD_HNSW_Nodes` | VARBYTE(64000) | Packed graph node data |

**Tuning guide:**

| Parameter | Default | Effect of increasing |
|-----------|---------|----------------------|
| `EfConstruction` | 32 | Better recall at search time; slower build |
| `NumConnPerNode` | 32 | More connections per node; better recall; larger index |
| `MaxNumConnPerNode` | 32 | Controls max fan-out; prevents over-connected nodes |
| `ApplyHeuristics` | false | Improves navigability; increases build time |

---

### TD_HNSW — Update and Delete

Insert new vectors into, or remove vectors from, an existing HNSW graph without rebuilding from scratch.

```sql
-- Update: insert new vectors into an existing graph
SELECT * FROM TD_HNSW(
    ON db.hnsw_model AS InputModelTable              -- existing graph (PARTITION BY ANY or no partition)
    ON { db.new_rows | (query) } AS InputTable DIMENSION  -- rows to insert
    OUT [ PERMANENT | VOLATILE ] TABLE ModelTable(db.hnsw_model_v2)
    USING
        IdColumn('id_col')
        VectorColumn('embedding_col')
        AlterOperation('update')
        [ UseSIMD('false') ]
) AS t;

-- Delete: remove vectors from an existing graph
SELECT * FROM TD_HNSW(
    ON db.hnsw_model AS InputModelTable
    ON { db.rows_to_delete | (query) } AS InputTable DIMENSION
    OUT [ PERMANENT | VOLATILE ] TABLE ModelTable(db.hnsw_model_v2)
    USING
        IdColumn('id_col')
        VectorColumn('embedding_col')
        AlterOperation('delete')
        [ DeleteMethod('reconstruction'|'deletenode') ]  -- default 'reconstruction'
        [ UseSIMD('false') ]
) AS t;
```

**`DeleteMethod` options:**

| Value | Behavior |
|-------|----------|
| `reconstruction` (default) | Graph is rebuilt after deletion — better graph quality |
| `deletenode` | Nodes removed in place, no rebuild — faster, but may degrade search quality |

---

### TD_HNSWPredict

*Documentation in progress — paste TD_HNSWPredict docs to continue.*

---

### TD_HNSWSummary

*Documentation in progress — paste TD_HNSWSummary docs to continue.*
