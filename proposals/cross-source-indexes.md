# Proposal for materializing indexes across sources

**Status — Draft for future consideration**

An index with an explicit `source` can include stored attributes whose
effective sources differ from the index target. The existing
[`indexes[].source` field](../specification.md#explicit-indexes) identifies
where to create the index and the physical fields it needs.

## Example

Assume `maindb` and `statsdb` are two SQL sources.

```yaml
models:
  - name: player
    set: players
    source: maindb
    attrs:
      - name: region
        type: string(16)

      - name: score
        type: int
        source: statsdb

    indexes:
      - name: region_score
        source: statsdb
        attrs: [region, score]
```

The schema adds a `region` column to `statsdb.players`, uses the existing
`score` column there, and creates the index on `(region, score)`. The runtime
maintains a copy of `region` in `statsdb` under the same player `id`. Ordinary
attribute reads still resolve `region` to `maindb` and `score` to `statsdb`.

## Target schema

For each indexed attribute, use its existing field if its effective source
matches the index target. Otherwise, derive a field from the attribute's type
and add it to the target's storage for the model. Each generated field is
shared by all indexes that use the attribute in that model and target source.
The model's `id` identifies the row or document. The metadata keeps one
attribute declaration and one authoritative source for each value.

SQL targets add table columns. A search provider may materialize the same
values as document fields. The provider validates whether it can store the
required fields and build the requested index. An index without an explicit
`source` continues to use the common effective source of its attributes.

## Synchronization

- When creating a record, write the fields required by the target indexes
  under the same `id`, including defaults used by the authoritative sources.
- When updating a record, send changed values to their authoritative sources
  and to each index target that holds a copy. Combine changes to native and
  copied fields in a single write to the target record so a compound index
  sees the resulting tuple. Preserve unchanged fields.
- When deleting a record, remove its indexed data from each target.
- When adding an index for existing records, populate its fields from the
  authoritative sources before making the index available to queries.

For transactional targets, write the copies during the same flush as the
authoritative changes, before the first source commits. If the target checks
`unique` during `INSERT` or `UPDATE`, a duplicate aborts the flush and the
runtime rolls back the open transactions. Sources commit individually, so a
failure during a later commit can leave earlier commits in effect.
