# Bulk Insert-or-Update / Upsert (from in-memory entities)

Insert rows that don't exist yet and update the ones that do, in one operation.
The in-memory collection is pushed into a temp table and merged into the target:
**SQL Server** uses a `MERGE`, **PostgreSQL** uses `INSERT … ON CONFLICT … DO
UPDATE`, **SQLite** uses batched `INSERT`/`UPDATE` statements. Returns the number
of affected rows (inserted + updated). (Owned entity types: only inlined
ones, and only on SQLite — see the notes below.)

> This page is for upserting a collection you already hold in memory
> (`List<T>`, array, …). For a pure insert use `Bulk-Insert.md`; for a pure
> update use `Bulk-Update.md`. To update rows from a server-side LINQ query, see
> `Bulk-Update-from-Query.md`.

**Enable:** `AddBulkOperationSupport()` in `UseSqlServer/UseNpgsql/UseSqlite`.
**Namespace:** `using Thinktecture;`

## API

The method is on `DbContext` and is the **same for every provider**:

```csharp
Task<int> BulkInsertOrUpdateAsync<T>(
   this DbContext ctx,
   IEnumerable<T> entities,
   Expression<Func<T, object?>>? propertiesToInsert = null,
   Expression<Func<T, object?>>? propertiesToUpdate = null,
   Expression<Func<T, object?>>? propertiesToMatchOn = null,
   CancellationToken cancellationToken = default) where T : class;

Task<int> BulkInsertOrUpdateAsync<T>(
   this DbContext ctx,
   IEnumerable<T> entities,
   IBulkInsertOrUpdateOptions options,
   CancellationToken cancellationToken = default) where T : class;
```

- `propertiesToInsert` — columns written for **new** rows. `null` ⇒ all properties.
- `propertiesToUpdate` — columns written for **existing** rows. `null` ⇒ all
  properties. Pass `IEntityPropertiesProvider.Empty` (options only) to skip the
  UPDATE entirely (insert-only / "do nothing on conflict").
- `propertiesToMatchOn` — the **match key** that decides insert-vs-update. `null`
  ⇒ the entity's primary key. In the options object this is `KeyProperties`.

Only the **options type** differs per provider (all implement
`IBulkInsertOrUpdateOptions`, which exposes `PropertiesToInsert`,
`PropertiesToUpdate`, `KeyProperties`, `TableNameProvider`, obsolete `TableName`/`Schema`):

| Provider | Options type | Notable extra properties |
|----------|--------------|--------------------------|
| SQL Server | `SqlServerBulkInsertOrUpdateOptions` | `MergeTableHints`, `TempTableOptions` (`BatchSize`, `EnableStreaming`, `BulkCopyTimeout`, `SqlBulkCopyOptions`) |
| PostgreSQL | `NpgsqlBulkInsertOrUpdateOptions` | `ConflictDoNothing`, `TempTableOptions` |
| SQLite | `SqliteBulkInsertOrUpdateOptions` | `AutoIncrementBehavior` |

## Examples

### Simplest — upsert everything, match on the primary key

```csharp
using Thinktecture;

var products = new List<Product>
{
   new() { Id = 1, Name = "Widget", Price = 19.99m },   // exists -> updated
   new() { Id = 2, Name = "Gadget", Price = 39.99m },   // new    -> inserted
};

int affected = await ctx.BulkInsertOrUpdateAsync(products);
```

### Select insert / update / match columns

```csharp
await ctx.BulkInsertOrUpdateAsync(
   products,
   propertiesToInsert: p => new { p.Id, p.Name, p.Price },
   propertiesToUpdate: p => p.Price,   // only Price refreshed on existing rows
   propertiesToMatchOn: p => p.Id);
```

### With provider options

```csharp
// SQL Server — subset of columns + match key, MERGE hints, temp-table tuning
await ctx.BulkInsertOrUpdateAsync(products, new SqlServerBulkInsertOrUpdateOptions
{
   PropertiesToInsert = IEntityPropertiesProvider.Include<Product>(p => new { p.Id, p.Name, p.Price }),
   PropertiesToUpdate = IEntityPropertiesProvider.Include<Product>(p => p.Price),
   KeyProperties      = IEntityPropertiesProvider.Include<Product>(p => p.Id),
   MergeTableHints    = { SqlServerTableHintLimited.HoldLock },
   TempTableOptions   = { BatchSize = 5_000, EnableStreaming = true },
});

// PostgreSQL — INSERT ... ON CONFLICT DO UPDATE
await ctx.BulkInsertOrUpdateAsync(products, new NpgsqlBulkInsertOrUpdateOptions
{
   PropertiesToUpdate = IEntityPropertiesProvider.Include<Product>(p => p.Price),
   KeyProperties      = IEntityPropertiesProvider.Include<Product>(p => p.Id),
});

// SQLite
await ctx.BulkInsertOrUpdateAsync(products, new SqliteBulkInsertOrUpdateOptions
{
   PropertiesToUpdate    = IEntityPropertiesProvider.Include<Product>(p => p.Price),
   KeyProperties         = IEntityPropertiesProvider.Include<Product>(p => p.Id),
   AutoIncrementBehavior = SqliteAutoIncrementBehavior.KeepValueAsIs,
});
```

### Insert-only (skip the update on conflict)

Provide **zero** properties to update — the UPDATE part is skipped entirely, so
existing rows are left untouched and only new rows are inserted:

```csharp
await ctx.BulkInsertOrUpdateAsync(products, new SqlServerBulkInsertOrUpdateOptions
{
   PropertiesToUpdate = IEntityPropertiesProvider.Empty,
});
```

On PostgreSQL the dedicated `ConflictDoNothing` flag does the same via
`ON CONFLICT … DO NOTHING`:

```csharp
await ctx.BulkInsertOrUpdateAsync(products, new NpgsqlBulkInsertOrUpdateOptions
{
   ConflictDoNothing = true,   // silently skip conflicting rows instead of updating
});
```

### Override the target table

```csharp
await ctx.BulkInsertOrUpdateAsync(products, new SqlServerBulkInsertOrUpdateOptions
{
   TableNameProvider = IBulkOperationTableNameProvider.ForRootEntity("ArchivedProducts", "archive"),
});
```

Bulk upsert writes the **root** table only (owned types in separate tables are not
supported, see the notes below), so `ForRootEntity` covers the table override.
`IBulkOperationTableNameProvider.Create(getTableName)` and a custom
`IBulkOperationTableNameProvider` work as well; they are called for the root entity type.

## Notes & pitfalls

- **Enable the feature** (`AddBulkOperationSupport()`), or the method throws at
  runtime.
- The **match key** decides insert vs. update: `propertiesToMatchOn` on the
  convenience overload, `KeyProperties` on the options object. `null`/unset ⇒ the
  entity's primary key.
- `PropertiesToInsert` and `PropertiesToUpdate` are independent — list the insert
  columns and the (typically smaller) set of columns to refresh on existing rows
  separately. Use `IEntityPropertiesProvider.Exclude<T>(…)` to keep all but a few.
- **Insert-only:** set `PropertiesToUpdate = IEntityPropertiesProvider.Empty`
  (all providers) or `ConflictDoNothing = true` (PostgreSQL only).
- The mechanism differs by provider: SQL Server `MERGE` (entities staged in a temp
  table), PostgreSQL `INSERT … ON CONFLICT … DO UPDATE`, SQLite batched
  `INSERT`/`UPDATE`. The return value is the total affected rows (inserted +
  updated).
- The table override only changes the **target** table; column names are still
  resolved from the EF Core model.
- Override tables via `TableNameProvider` (`IBulkOperationTableNameProvider.GetTableName(IEntityType)`
  returning `BulkOperationTableName(Name, Schema)`). Built-in: `IBulkOperationTableNameProvider.Default`
  (model tables, the default of `TableNameProvider`), `ForRootEntity(tableName, schema)`
  (root only; throws when the operation reaches an owned type in a separate table) and
  `Create(getTableName)` (calls the delegate for every entity type).
  A `null` schema means the database default, not the model schema.
- `TableName`/`Schema` are `[Obsolete]` (CS0618): root table only, a `null` part keeps the
  model value, throw when the operation reaches an owned type in a separate table. Setting
  `TableName`/`Schema` replaces `TableNameProvider`, setting `TableNameProvider` clears
  them, and the last assignment wins. Migrating `TableName = "X"` to `ForRootEntity("X")`
  removes the model schema; pass the schema explicitly. No transaction is added by the
  library; atomicity is the responsibility of the caller's transaction.
- **Owned entity types** (verified by tests per provider):
  - SQL Server / PostgreSQL: not supported for any owned kind (inlined, separate
    `OwnsOne`, `OwnsMany`). Throws `NotSupportedException` ("Temp tables don't support
    owned entities.") before the target table is written.
  - SQLite: inlined `OwnsOne` works. Separate `OwnsOne` and `OwnsMany` throw
    `NotSupportedException` ("The owned type property '…' is selected for bulk-insert-or-update but
    there are no properties for performing the update.") before the target table is
    written.
  Entities with shadow properties must be attached to the `DbContext` so the values
  can be read.
  **Workaround:** bulk **insert** supports owned types, so for an upsert split the
  work — `BulkInsertAsync` the new rows and update existing rows' owned columns via
  EF Core's native `ExecuteUpdateAsync` — or restructure the owned type into scalar
  properties.
