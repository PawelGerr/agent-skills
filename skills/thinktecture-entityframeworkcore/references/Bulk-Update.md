# Bulk Update (from in-memory entities)

Update a large in-memory collection in one fast operation. The rows are pushed
into a temp table and then applied to the target table: **SQL Server** uses a
`MERGE`, **PostgreSQL** uses `UPDATE … FROM <temp table>`, **SQLite** uses
batched `UPDATE` statements. Returns the number of affected rows. (Owned entity
types: SQLite only — see the notes below.)

> This page is for updating a collection you already hold in memory
> (`List<T>`, array, …) — every entity carries the new values. To update rows
> from the result of a server-side LINQ query (no round-trip to the app, set
> target columns from a related source set), see `Bulk-Update-from-Query.md`.

**Enable:** `AddBulkOperationSupport()` in `UseSqlServer/UseNpgsql/UseSqlite`.
**Namespace:** `using Thinktecture;`

## API

The method is on `DbContext` and is the **same for every provider**:

```csharp
Task<int> BulkUpdateAsync<T>(
   this DbContext ctx,
   IEnumerable<T> entities,
   Expression<Func<T, object?>>? propertiesToUpdate = null,
   Expression<Func<T, object?>>? propertiesToMatchOn = null,
   CancellationToken cancellationToken = default) where T : class;

Task<int> BulkUpdateAsync<T>(
   this DbContext ctx,
   IEnumerable<T> entities,
   IBulkUpdateOptions options,
   CancellationToken cancellationToken = default) where T : class;
```

- `propertiesToUpdate` — which columns to write. `null` ⇒ all properties.
- `propertiesToMatchOn` — the **match key** (the `JOIN`/`ON` columns). `null` ⇒
  the entity's primary key. In the options object this is the `KeyProperties`
  property.

Only the **options type** differs per provider (all implement `IBulkUpdateOptions`,
which exposes `PropertiesToUpdate`, `KeyProperties`, `TableNameProvider`, obsolete `TableName`/`Schema`):

| Provider | Options type | Notable extra properties |
|----------|--------------|--------------------------|
| SQL Server | `SqlServerBulkUpdateOptions` | `MergeTableHints`, `TempTableOptions` (`BatchSize`, `EnableStreaming`, `BulkCopyTimeout`, `SqlBulkCopyOptions`) |
| PostgreSQL | `NpgsqlBulkUpdateOptions` | `TempTableOptions` |
| SQLite | `SqliteBulkUpdateOptions` | `AutoIncrementBehavior` |

## Examples

### Simplest — update everything, match on the primary key

```csharp
using Thinktecture;

var products = new List<Product>
{
   new() { Id = 1, Name = "Widget", Price = 19.99m },
   new() { Id = 2, Name = "Gadget", Price = 39.99m },
};

int affected = await ctx.BulkUpdateAsync(products);
```

### Update only selected columns

The first projection selects the columns to **write**:

```csharp
// only Price is sent to the DB; match still uses the primary key
await ctx.BulkUpdateAsync(products, p => p.Price);

// equivalent with an anonymous type for multiple columns
await ctx.BulkUpdateAsync(products, p => new { p.Name, p.Price });
```

### Custom match key

```csharp
// update Price, but match rows on Sku instead of the primary key
await ctx.BulkUpdateAsync(
   products,
   propertiesToUpdate: p => p.Price,
   propertiesToMatchOn: p => p.Sku);
```

### With provider options

```csharp
// SQL Server — select columns + match key, tune the temp-table bulk copy
await ctx.BulkUpdateAsync(products, new SqlServerBulkUpdateOptions
{
   PropertiesToUpdate = IEntityPropertiesProvider.Include<Product>(p => p.Price),
   KeyProperties      = IEntityPropertiesProvider.Include<Product>(p => p.Id),
   MergeTableHints    = { SqlServerTableHintLimited.HoldLock, SqlServerTableHintLimited.RowLock },
   TempTableOptions   =
   {
      BatchSize = 5_000,
      EnableStreaming = true,
      BulkCopyTimeout = TimeSpan.FromSeconds(30),
   },
});

// PostgreSQL
await ctx.BulkUpdateAsync(products, new NpgsqlBulkUpdateOptions
{
   PropertiesToUpdate = IEntityPropertiesProvider.Include<Product>(p => p.Price),
   KeyProperties      = IEntityPropertiesProvider.Include<Product>(p => p.Id),
});

// SQLite
await ctx.BulkUpdateAsync(products, new SqliteBulkUpdateOptions
{
   PropertiesToUpdate    = IEntityPropertiesProvider.Include<Product>(p => p.Price),
   KeyProperties         = IEntityPropertiesProvider.Include<Product>(p => p.Id),
   AutoIncrementBehavior = SqliteAutoIncrementBehavior.KeepValueAsIs,
});
```

### Exclude a few columns instead of listing all of them

```csharp
await ctx.BulkUpdateAsync(products, new SqlServerBulkUpdateOptions
{
   PropertiesToUpdate = IEntityPropertiesProvider.Exclude<Product>(p => p.CreatedAt),
});
```

### Override the target table

```csharp
await ctx.BulkUpdateAsync(products, new SqlServerBulkUpdateOptions
{
   TableNameProvider = IBulkOperationTableNameProvider.ForRootEntity("ArchivedProducts", "archive"),
});
```

`ForRootEntity` redirects the **root** table only and throws when the operation reaches
an owned type mapped to a separate table. Rows of the root table may already be written
at that point. For owned types in separate tables, use `IBulkOperationTableNameProvider.Create`
or implement `IBulkOperationTableNameProvider`. The provider is called for the root entity
type and for every separately mapped owned entity type (nested included).
SQL Server and PostgreSQL reject owned types, so the provider is called for the root
entity type only; SQLite also writes separately mapped owned types.

```csharp
// Appends "_Archive" to every model table name; no schema is emitted (pass t.GetSchema() to keep it).
var provider = IBulkOperationTableNameProvider.Create(t => new BulkOperationTableName(IBulkOperationTableNameProvider.Default.GetTableName(t).Name + "_Archive"));
await ctx.BulkUpdateAsync(products, new SqliteBulkUpdateOptions { TableNameProvider = provider });

// A class for rules that need more than one expression.
sealed class ArchiveTables : IBulkOperationTableNameProvider
{
   public BulkOperationTableName GetTableName(IEntityType entityType)
   {
      // Root: FindOwnership() is null. Owned types: distinguish by navigation, never by ClrType.
      var navigation = entityType.FindOwnership()?.PrincipalToDependent?.Name;
      return new BulkOperationTableName(navigation is null ? "ArchivedProducts" : $"ArchivedProducts_{navigation}");
   }
}

await ctx.BulkUpdateAsync(products, new SqliteBulkUpdateOptions { TableNameProvider = new ArchiveTables() }); // owned types: SQLite only
```

## Notes & pitfalls

- **Enable the feature** (`AddBulkOperationSupport()`), or the method throws at
  runtime.
- The **match key** is `propertiesToMatchOn` on the convenience overload, and
  `KeyProperties` on the options object — same thing. `null`/unset ⇒ the entity's
  primary key.
- `propertiesToUpdate` / `PropertiesToUpdate` lists the columns to **write**; use
  `IEntityPropertiesProvider.Exclude<T>(…)` to write everything except a few.
- The mechanism differs by provider: SQL Server runs a `MERGE` (entities go into a
  temp table first), PostgreSQL runs `UPDATE … FROM <temp table>`, SQLite issues
  batched `UPDATE` statements. The return value is the number of affected rows.
- The table override only changes the **target** table; column names are still
  resolved from the EF Core model, so the override target must have compatible
  columns.
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
  - SQLite: inlined `OwnsOne`, separate `OwnsOne` and `OwnsMany` are updated (owned rows
    by their primary key; owned entities must be tracked, e.g. loaded with the owner).
    The root table needs at least one column besides the key, otherwise the generated
    `UPDATE` has an empty `SET` and fails with a syntax error.
  Entities with shadow properties must be attached to the `DbContext` so the values
  can be read.
  **Workaround:** update the owned/mapped columns separately with EF Core's native
  `ExecuteUpdateAsync` (a single-table set-based update), or restructure the model
  so those columns are regular scalar properties rather than an owned type.
- To update from a server-side query instead of in-memory data (e.g. set target
  columns from a joined source set), use **`DbSet<T>.BulkUpdateAsync(IQueryable source, …)`**
  — a different API — see `Bulk-Update-from-Query.md`.
