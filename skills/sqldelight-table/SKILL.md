---
name: sqldelight-table
description: Checklist for adding a SQLDelight table, column or many-to-many relationship in a KMP app — the .sq file, the numbered .sqm migration, the logout wipe, junction tables, and the pitfalls (no DROP COLUMN on old SQLite, foreign keys not enforced). Use when introducing a cached entity, a new field that must survive across screens, or a relationship between entities.
---

# SQLDelight table

Every model a screen observes round-trips through the local DB (see `repository`). Ephemeral UI state (filters, an open panel) does not: that is `rememberSaveable` or the ViewModel's state (see `saved-state`).

## New table: all four, in one change

1. **`<Entity>.sq`** under `libraries/database/src/commonMain/sqldelight/…`: table, indexes, queries (`selectById`, `selectAll`, `upsert`, `deleteById`, `deleteAll`).
2. **`migrations/<N>.sqm`**, N = next integer. Prefer `CREATE TABLE IF NOT EXISTS`. Editing the `.sq` alone changes only fresh installs; existing users crash on upgrade.
3. **`deleteAll()` in the logout wipe** (`DatabaseProvider.clearAllTables()` or equivalent). Nothing catches a missed entry; the data survives sign-out.
4. **Local data source** with domain mappers (see `data-source`).

```sql
CREATE TABLE IF NOT EXISTS Note (
    id TEXT NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    updatedAt INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS note_updatedAt ON Note(updatedAt);

selectAll: SELECT * FROM Note ORDER BY updatedAt DESC;
upsert:    INSERT OR REPLACE INTO Note VALUES ?;
deleteAll: DELETE FROM Note;
```

## New column

- Add it to the table **and** to every insert/update query, plus `ALTER TABLE … ADD COLUMN` in a new `.sqm`.
- Read it in `toDomain`, write it in `toRow`, set it in the network mapper.
- **Dropping or renaming a column**: Android before API 34 ships SQLite < 3.35 with no `DROP COLUMN`. Recreate the table, copy, drop, rename.

## Many-to-many

- A **junction table** with a composite primary key and an index on each side. Never a JSON blob in the parent row.
- Set the relationship transactionally: delete-then-insert in one `transaction`.
- Foreign keys aren't enforced on every driver: cascade deletes in code.
- Cache from one network object: parent row, child rows, junction rows, in one write.

## Rules

- Don't persist what the project forbids (its CLAUDE.md lists sensitive fields).
- Store domain or backend types, never generated proto classes.
- Run the migration verify task (`verifySqlDelightMigration`) when it's configured.
