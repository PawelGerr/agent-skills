# Bulk Insert (from a query)

Insert the result of a server-side LINQ query straight into a table, without a
round-trip to the app: generates `INSERT INTO target (cols) SELECT … FROM (subquery) AS s`.
Returns the number of affected rows.

> This page is for inserting the rows produced by an `IQueryable<TSource>`
> (another query, or a temp table). To insert a collection you already hold in
> memory (`List<T>`, array, …) via `SqlBulkCopy` / `COPY` / batched `INSERT`,
> see `Bulk-Insert.md`.

**Enable:** `AddBulkOperationSupport()` in `UseSqlServer/UseNpgsql/UseSqlite`.
**Namespace:** `using Thinktecture;`

## API

The method is on **`DbSet<TTarget>`** (not `DbContext`). The signature is
identical for every provider — only the `options` type differs:

```csharp
// SQL Server
Task<int> BulkInsertAsync<TTarget, TSource>(
   this DbSet<TTarget> target,
   IQueryable<TSource> sourceQuery,
   Func<InsertPropertyBuilder<TTarget, TSource>, InsertPropertyBuilder<TTarget, TSource>> mapPropertyCalls,
   SqlServerBulkInsertFromQueryOptions? options = null,
   CancellationToken cancellationToken = default)
   where TTarget : class
   where TSource : class;

// PostgreSQL — same shape, options: NpgsqlBulkInsertFromQueryOptions?
// SQLite     — same shape, options: SqliteBulkInsertFromQueryOptions?
```

The `InsertPropertyBuilder<TTarget, TSource>` maps each target column to a source
value expression:

```csharp
InsertPropertyBuilder<TTarget, TSource> Map<TProp>(
   Expression<Func<TTarget, TProp>> targetColumnSelector,
   Expression<Func<TSource, TProp>> sourceValueSelector);
```

Options (`SqlServerBulkInsertFromQueryOptions` / `NpgsqlBulkInsertFromQueryOptions` /
`SqliteBulkInsertFromQueryOptions`) expose only the target-table override:

| Property | Purpose |
|----------|---------|
| `IBulkOperationTableNameProvider TableNameProvider` | Override the target table (defaults to `IBulkOperationTableNameProvider.Default`, i.e. the model table). Called for the target entity type only. `ForRootEntity(tableName, schema)`; a `null` schema means the database default, not the model schema |
| `string? TableName` / `string? Schema` | `[Obsolete]`: a `null` part keeps the model value; setting them replaces `TableNameProvider`, setting `TableNameProvider` clears them; the last assignment wins |

## Examples

### Simplest — insert a projection of another query

Identical across SQL Server / PostgreSQL / SQLite (only identifier quoting in the
generated SQL differs):

```csharp
using Thinktecture;

// Source: any server-side IQueryable<TSource>.
var sourceQuery = ctx.Products
   .Where(p => p.Discontinued)
   .Select(p => new { p.Id, p.Name, p.Price });

int affected = await ctx.Set<Product>().BulkInsertAsync(
   sourceQuery,
   builder => builder
      .Map(target => target.Id,    source => source.Id)
      .Map(target => target.Name,  source => source.Name)
      .Map(target => target.Price, source => source.Price));
```

Generates `INSERT INTO [Products] (...) SELECT s.[Id], s.[Name], s.[Price] FROM (<sourceQuery>) AS s`.

### Source can be a temp table

`tempTable.Query` (an `IQueryable<T>`) is a valid source — register the row type
via `ConfigureTempTableEntity<T>()` if it is temp-table-only (see `Temp-Tables.md`):

```csharp
await using var tempTable = await ctx.BulkInsertIntoTempTableAsync(rows);

await ctx.Set<Product>().BulkInsertAsync(
   tempTable.Query,
   b => b.Map(t => t.Id, s => s.Id)
         .Map(t => t.Name, s => s.Name));
```

### Value selectors are full expressions

The source side of `Map` is not limited to a plain property — it accepts:

```csharp
await ctx.Set<Product>().BulkInsertAsync(
   sourceQuery,
   b => b
      .Map(t => t.Id,    s => s.Id)
      .Map(t => t.Name,  s => s.Name!.ToUpper())          // server function
      .Map(t => t.Count, s => s.Count * 2 + 1)            // arithmetic on source
      .Map(t => t.Note,  s => myCapturedVariable)         // captured variable → parameter
      .Map(t => t.Status, s => ProductStatus.Published)   // static member → parameter
      .Map(t => EF.Property<int>(t, "_privateField"),
           s => EF.Property<int>(s, "_privateField")));   // shadow / backing field
```

- Simple property access and `EF.Property<T>(...)` map to source columns.
- A value selector that is a single constant, captured variable or static member
  (e.g. a smart-enum value) is bound as a SQL parameter (`__insert_const_N`); a
  value converter of the target column is applied.
- A constant inside a larger expression is rendered as a SQL literal.
- Arithmetic / method-call expressions are translated to SQL by EF Core.

### Complex types (since 10.6.0)

The target selector can go through complex properties (`ComplexProperty`, class or
struct, nested allowed), or it can name a whole complex property:

```csharp
await ctx.Set<Message>().BulkInsertAsync(
   tempTable.Query,
   b => b
      .Map(t => t.Id, s => s.Id)
      .Map(t => t.MessageTimestamp, s => s.MessageTimestamp)
      .Map(t => t.EffectiveMessage, s => s.EffectiveMessage)); // whole complex property

// single member instead: .Map(t => t.EffectiveMessage.ReceivedAt, s => s.EffectiveMessage.ReceivedAt)
```

- A whole complex property expands into one column per member, recursively for
  nested complex types.
- The value of a whole complex property is the source complex property or a
  captured variable. A `new` expression is not evaluated on the client; assign it
  to a variable first.
- Members with a value converter work; a captured value is converted with the
  converter of the target column.
- A whole complex property skips members whose value the database generates
  (`GetBeforeSaveBehavior() == Ignore`, e.g. computed columns); map such a member
  explicitly to write it. When every writable member of a whole complex property
  is generated: `ArgumentException` "No column can be written, because every
  member of the selected complex properties is generated
  (PropertySaveBehavior.Ignore). Expression: <that selector>", also when other
  `Map` entries exist.
- `NotSupportedException`:
  - complex collection ("The complex collection '…' is not supported.");
  - nullable complex property as a whole ("The nullable complex property '…'
    cannot be set as a whole."); a member through a nullable complex property,
    class or struct (`t => t.OptionalBoundary!.Upper`,
    `t => t.Boundary.Value.Upper`), throws "The member '…' of '…' is a
    nullable complex property.";
  - target selector through a member that is not a complex property, e.g. a
    navigation ("The member '…' is not a complex property of '…'");
  - target selector that is not a member chain on the lambda parameter, e.g.
    `t => t.Count + 1` ("Only simple property access expressions (e => e.Property),
    member access through complex properties (e => e.ComplexProperty.Property) or
    EF.Property calls on the lambda parameter are supported.");
  - `EF.Property` not on the lambda parameter, e.g.
    `EF.Property<long>(t.EffectiveMessage, "ReceivedAt")` ("EF.Property is
    supported only on the lambda parameter, not through a complex property, a
    navigation or another expression."); use member access instead.
- `InvalidOperationException` "The property '…' is not a mapped property on '…'."
  for an unmapped member; for a complex member it names the complex type.
- `ArgumentException`:
  - captured `null` as the value of a whole complex property, or any member access
    on a captured `null`: "The member '…' of '…' cannot be read because the
    captured value is null."; pass a non-null value or map the members one by one.
    Captured values are evaluated member by member, so `captured.HasValue` on a
    `null` `int?` and `captured == null ? 0 : captured.Count` throw as well; check
    for `null` before the call.
  - same target column mapped twice, also a whole complex property plus one of its
    members ("The target column '…' is mapped more than once."), on all providers.
- Nested complex types need settable or `init` members, because EF Core cannot
  bind a nested complex type to a constructor parameter.

### Override the target table

```csharp
await ctx.Set<Product>().BulkInsertAsync(
   sourceQuery,
   b => b.Map(t => t.Id, s => s.Id).Map(t => t.Name, s => s.Name),
   new SqlServerBulkInsertFromQueryOptions { TableNameProvider = IBulkOperationTableNameProvider.ForRootEntity("ArchivedProducts", "archive") });
   // PostgreSQL: new NpgsqlBulkInsertFromQueryOptions { ... }
   // SQLite:     new SqliteBulkInsertFromQueryOptions { ... }
```

## Notes & pitfalls

- **Enable the feature** (`AddBulkOperationSupport()`), or the method throws at
  runtime.
- This is **`DbSet<TTarget>.BulkInsertAsync(IQueryable source, map)`** — a
  *different* API from `DbContext.BulkInsertAsync(collection)` (the in-memory
  variant, see `Bulk-Insert.md`). Calling `ctx.Set<T>()` first gives you the
  `DbSet`.
- You must configure **at least one** `Map` entry, otherwise it throws
  `ArgumentException`.
- The source must be a server-evaluable `IQueryable<TSource>`; its SQL and
  parameters are inlined as the `SELECT` subquery.
- Returns `Task<int>` — the number of inserted rows.
- For a temp-table source whose row type is registered only via
  `ConfigureTempTableEntity<T>()`, column names resolve correctly because the
  executor also looks the type up under its temp-table name.
