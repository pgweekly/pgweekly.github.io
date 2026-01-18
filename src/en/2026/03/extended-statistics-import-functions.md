# Extended Statistics Import/Export Functions for PostgreSQL 19

## Introduction

PostgreSQL's extended statistics feature, introduced in version 10, allows the optimizer to understand correlations between columns that simple per-column statistics cannot capture. This includes n-distinct coefficients (how many distinct combinations exist), functional dependencies (when one column determines another), and Most Common Values (MCV) lists for multi-column combinations.

While PostgreSQL 17 introduced functions for importing and exporting relation and attribute statistics (`pg_restore_relation_stats`, `pg_restore_attribute_stats`), extended statistics were left out of this initial implementation. A recent thread on pgsql-hackers, initiated by Corey Huinker, addresses this gap with a comprehensive patch series that adds `pg_restore_extended_stats()`, `pg_clear_extended_stats()`, and related infrastructure.

This work is significant for several reasons:
- Enables complete statistics preservation across pg_dump/pg_restore and pg_upgrade
- Allows query planner experimentation with hypothetical statistics
- Supports schema-only dumps with statistics for testing query plans without actual data

## Technical Analysis

### The Problem with the Original Format

The original output format for `pg_ndistinct` and `pg_dependencies` types used a JSON structure where the keys themselves contained structured data:

```json
{"1, 2": 2323, "1, 3": 3232, "2, 3": 1500}
```

While technically valid JSON, this format posed several problems:
1. Keys containing comma-separated attribute numbers require additional parsing
2. Difficult to manipulate programmatically
3. No working input function existed—these types were effectively output-only

### The New JSON Format

The patch series introduces a cleaner, more structured JSON format. For `pg_ndistinct`:

```json
[
  {"attributes": [2, 3], "ndistinct": 4},
  {"attributes": [2, -1], "ndistinct": 4},
  {"attributes": [2, 3, -1], "ndistinct": 4}
]
```

For `pg_dependencies`:

```json
[
  {"attributes": [2], "dependency": 3, "degree": 1.000000},
  {"attributes": [2, 3], "dependency": -1, "degree": 0.850000}
]
```

Key improvements:
- **Proper JSON arrays** with named keys for each element
- **Clear separation** of attributes, values, and metadata
- **Machine-readable** without custom parsing logic
- **Negative attribute numbers** represent expressions in the statistics object (e.g., `-1` is the first expression)

### Input Function Implementation

The new input functions use PostgreSQL's JSON parser infrastructure with a custom semantic action handler. Here's a simplified view of the parsing state machine for `pg_ndistinct`:

```c
typedef enum
{
    NDIST_EXPECT_START = 0,
    NDIST_EXPECT_ITEM,
    NDIST_EXPECT_KEY,
    NDIST_EXPECT_ATTNUM_LIST,
    NDIST_EXPECT_ATTNUM,
    NDIST_EXPECT_NDISTINCT,
    NDIST_EXPECT_COMPLETE
} ndistinctSemanticState;
```

The parser validates:
- Proper JSON structure (array of objects)
- Required keys (`attributes` and `ndistinct` for ndistinct statistics)
- Attribute numbers within valid ranges (positive for columns, negative for expressions, but not beyond `STATS_MAX_DIMENSIONS`)
- No duplicate attributes within a single item

### Extended Statistics Functions

The patch introduces three main SQL functions:

**pg_restore_extended_stats()** — Imports extended statistics from a previously exported value:

```sql
SELECT pg_restore_extended_stats(
    'public',                    -- relation schema
    'my_table',                  -- relation name
    'public',                    -- statistics schema  
    'my_stats',                  -- statistics name
    false,                       -- inherited
    '{"version": ..., "ndistinct": [...], "dependencies": [...], "mcv": [...], "exprs": [...]}'::text
);
```

**pg_clear_extended_stats()** — Removes extended statistics data from `pg_statistic_ext_data`:

```sql
SELECT pg_clear_extended_stats(
    'public',        -- statistics schema
    'my_stats',      -- statistics name
    false            -- inherited
);
```

The functions follow the same patterns established for relation/attribute statistics:
- Return boolean indicating success
- Issue `WARNING` (not `ERROR`) on problems to avoid breaking pg_restore scripts
- Require `MAINTAIN` privilege on the target relation

### Validation and Safety

The implementation includes careful validation:

1. **Attribute bounds checking**: Positive attnums must exist in `stxkeys`, negative attnums must not exceed the number of expressions
2. **Combination completeness**: For `pg_ndistinct`, all N-choose-K combinations must be present based on the longest attribute list
3. **Soft error handling**: Uses PostgreSQL's `ErrorSaveContext` for safe error reporting without crashing

Example validation for attribute numbers:

```c
if (attnum == 0 || attnum < (0 - STATS_MAX_DIMENSIONS))
{
    errsave(parse->escontext,
            errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
            errmsg("malformed pg_ndistinct: \"%s\"", parse->str),
            errdetail("Invalid \"%s\" element: %d.",
                      PG_NDISTINCT_KEY_ATTRIBUTES, attnum));
    return JSON_SEM_ACTION_FAILED;
}
```

## Community Insights

### Key Discussion Points

**Format Change Timing**: Tomas Vondra initially suggested a more structured JSON format. The community recognized this was the last opportunity to change the format before a working input function locked in backward compatibility requirements.

**Validation Scope**: There was significant discussion about how much validation to perform:
- Early patches had extensive checks for statistical consistency (e.g., MCV frequencies summing to 1.0)
- Reviewers pushed back, preferring minimal validation to avoid breaking legitimate but unusual imports
- Final consensus: validate structure and attribute references, but not statistical values

**pg_dependencies Special Case**: Unlike `pg_ndistinct` which stores all combinations, `pg_dependencies` may omit statistically insignificant combinations. This means the input function cannot enforce complete combination coverage for dependencies.

### Reviewer Feedback Integration

Michael Paquier provided extensive review and contributed significant improvements:
- Restructured the patch series for cleaner commits
- Split format changes from input function additions
- Added comprehensive regression tests achieving >90% code coverage
- Fixed compiler warnings on older GCC versions

Tom Lane caught style issues:
- Error detail messages converted to complete sentences
- Replaced `SOFT_ERROR_OCCURRED()` macro with direct state checks to avoid warnings

## Current Status

As of January 2026, the patch series has progressed significantly:

**Committed:**
- Output format changes for `pg_ndistinct` (new JSON array format)
- Output format changes for `pg_dependencies` (new JSON array format)  
- Input functions for both types with comprehensive validation
- `pg_clear_extended_stats()` function

**In Review (v27):**
- `pg_restore_extended_stats()` function
- pg_dump integration for extended statistics export/import

The pg_dump integration supports backward compatibility to PostgreSQL 10, with version-specific SQL generation to handle format differences.

## Technical Details

### Internal Storage Unchanged

Importantly, the internal binary storage format remains unchanged. The new input/output functions only affect the text representation. This means:
- No catalog changes required
- Existing data remains valid
- Binary COPY operations unaffected

### Expression Statistics Support

Extended statistics can include expressions (e.g., `CREATE STATISTICS s ON (a + b), c FROM t`). The implementation handles these via negative attribute numbers:
- `-1` = first expression
- `-2` = second expression
- etc.

The `exprs` element in the restore format contains per-expression statistics similar to `pg_statistic` entries, enabling complete round-trip preservation.

### MCV List Handling

MCV (Most Common Values) lists for extended statistics are particularly complex, containing:
- Value combinations across multiple columns
- Frequency and base frequency arrays
- Per-value null bitmaps

The implementation reuses infrastructure from attribute statistics import, with extensions for multi-column value arrays.

## Conclusion

This patch series represents a significant enhancement to PostgreSQL's statistics infrastructure. By enabling import/export of extended statistics, it:

1. **Completes the statistics story** started in PostgreSQL 17 for relation and attribute statistics
2. **Enables realistic testing** with production-like statistics on sanitized schemas
3. **Improves upgrade reliability** by preserving optimizer information across pg_upgrade

For DBAs and developers:
- Extended statistics created with `CREATE STATISTICS` will now survive pg_dump/pg_restore
- Query plan testing becomes more practical with `--no-data` dumps that include full statistics
- The new JSON format is human-readable for debugging and hypothetical scenario testing

The target release is PostgreSQL 19, with the remaining restore function and pg_dump integration expected to land soon.

## References

- [Original thread on pgsql-hackers](https://www.postgresql.org/message-id/CADkLM%3Ddpz3KFnqP-dgJ-zvRvtjsa8UZv8wDAQdqho%3DqN3kX0Zg%40mail.gmail.com)
- [PostgreSQL Extended Statistics Documentation](https://www.postgresql.org/docs/current/planner-stats.html#PLANNER-STATS-EXTENDED)
- [Commitfest Entry](https://commitfest.postgresql.org/patch/5517/)
