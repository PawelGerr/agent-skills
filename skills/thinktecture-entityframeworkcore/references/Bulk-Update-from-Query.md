# Bulk Update (from a query)

Update a table by joining it to the result of a server-side LINQ query, without a
round-trip to the app. **SQL Server** generates
`UPDATE t SET … FROM target INNER JOIN (subquery) AS s ON …`; **PostgreSQL** and
**SQLite** generate `UPDATE target AS t SET … FROM (subquery) AS s WHERE …`.
Returns the number of affected rows.

> This page is for updating from another query/temp table. To update a
> collection you already hold in memory (`List<T>`, array, …), see
> `Bulk-Update.md`. For a *single-table* set-based update with a simple `Where`,
> prefer EF Core's native `ExecuteUpdateAsync` — reach for this API when the
> update must **join another query** (set target columns from a related source
> set) or needs **composite-key** joins.

**Enable:** `AddBulkOperationSupport()` in `UseSqlServer/UseNpgsql/UseSqlite`.
**Namespace:** `using Thinktecture;`

## API

The method is on **`DbSet<TTarget>`** (not `DbContext`). The signature is
identical for every provider — only the `options` type differs:

```csharp
// SQL Server
Task<int> BulkUpdateAsync<TTarget, TSource, TResult>(
   this DbSet<TTarget> target,
   IQueryable<TSource> sourceQuery,
   Expression<Func<TTarget, TResult?>> targetKeySelector,
   Expression<Func<TSource, TResult?>> sourceKeySelector,
   Func<SetPropertyBuilder<TTarget, TSource>, SetPropertyBuilder<TTarget, TSource>> setPropertyCalls,
   Expression<Func<TTarget, TSource, bool>>? filter = null,
   SqlServerBulkUpdateFromQueryOptions? options = null,
   CancellationToken cancellationToken = default)
   where TTarget : class
   where TSource : class;

// PostgreSQL — same shape, options: NpgsqlBulkUpdateFromQueryOptions?
// SQLite     — same shape, options: SqliteBulkUpdateFromQueryOptions?
```

The `SetPropertyBuilder<TTarget, TSource>` assigns each target property a value
expression (which may reference **both** target `e` and source `f`):

```csharp
SetPropertyBuilder<TTarget, TSource> Set<TProp>(
   Expression<Func<TTarget, TProp>> targetPropertySelector,
   Expression<Func<TTarget, TSource, TProp>> valueSelector);
```

Options (`SqlServerBulkUpdateFromQueryOptions` / `NpgsqlBulkUpdateFromQueryOptions` /
`SqliteBulkUpdateFromQueryOptions`) expose only the target-table override:

| Property | Purpose |
|----------|---------|
| `IBulkOperationTableNameProvider TableNameProvider` | Override the target table (defaults to `IBulkOperationTableNameProvider.Default`, i.e. the model table). Called for the target entity type only. `ForRootEntity(tableName, schema)`, `ForRootEntityWithDefaultSchema(tableName)` (model schema) or `ForRootEntityWithDefaultName(schema)` (model table name); a `null` schema means the database default, not the model schema |
| `string? TableName` / `string? Schema` | `[Obsolete]`: a `null` part keeps the model value; setting them replaces `TableNameProvider`, setting `TableNameProvider` clears them; the last assignment wins. Replace `TableName = "X"` by `ForRootEntityWithDefaultSchema("X")`, `Schema = "s"` by `ForRootEntityWithDefaultName("s")` and both by `ForRootEntity("X", "s")` |

## Examples

### Simplest — single-key join, copy source values

Identical across SQL Server / PostgreSQL / SQLite (only the generated SQL shape /
quoting differs):

```csharp
using Thinktecture;

var sourceQuery = ctx.Set<ProductUpdateDto>(); // any server-side IQueryable<TSource>

int affected = await ctx.Set<Product>().BulkUpdateAsync(
   sourceQuery,
   target => target.Id,     // join key on target
   source => source.Id,     // join key on source
   builder => builder
      .Set(e => e.Name,  (e, f) => f.Name)
      .Set(e => e.Count, (e, f) => f.Count));
```

### Composite key (anonymous type)

Key selectors support single **and** composite keys; build a composite key with
an anonymous type — the member names must match on both sides:

```csharp
await ctx.Set<Product>().BulkUpdateAsync(
   sourceQuery,
   e => new { e.Id, e.Name },
   f => new { f.Id, f.Name },
   builder => builder.Set(e => e.Count, (e, f) => f.Count));
```

The source key may map differently named columns, since you control the projection:

```csharp
var sourceQuery = ctx.Products.Select(p => new { Identifier = p.Id, Number = p.Count });

await ctx.Set<Product>().BulkUpdateAsync(
   sourceQuery,
   e => e.Id,
   f => f.Identifier,
   builder => builder.Set(e => e.Count, (e, f) => f.Number));
```

### Optional `filter` — restrict which rows update

The `filter` is `Expression<Func<TTarget, TSource, bool>>` and can reference
**both** target and source. Rows that match the join but fail the filter are left
untouched (it adds a `WHERE` / extra `ON` condition):

```csharp
await ctx.Set<Product>().BulkUpdateAsync(
   sourceQuery,
   e => e.Id,
   f => f.Id,
   builder => builder.Set(e => e.Name, (e, f) => f.Name),
   filter: (e, f) => e.Count < f.Count);        // target & source compared

// also valid: filter referencing only the target, a captured variable,
// or arithmetic across both, e.g. (e, f) => e.Count + f.Count > 15
```

### Value expressions are full expressions

The `Set` value selector accepts more than plain source properties:

```csharp
await ctx.Set<Product>().BulkUpdateAsync(
   sourceQuery,
   e => e.Id,
   f => f.Id,
   builder => builder
      .Set(e => e.Name,   (e, f) => f.Name)                  // source property
      .Set(e => e.Count,  (e, f) => e.Count + f.Count * 2)   // arithmetic over target & source
      .Set(e => e.Total,  (e, f) => 42)                      // constant → SQL literal
      .Set(e => e.Note,   (e, f) => myCapturedVariable)      // captured variable
      .Set(e => e.Status, (e, f) => ProductStatus.Published) // static member / smart-enum
      .Set(e => EF.Property<int>(e, "_privateField"),
           (e, f) => EF.Property<int>(f, "_privateField"))); // shadow / backing field
```

- Simple property access and `EF.Property<T>(...)` map to columns.
- Constants, captured variables and static members are evaluated once and
  rendered as SQL literals with the type mapping of the target column, so a value
  converter (e.g. smart enums) is applied.
- Arithmetic / method-call expressions are translated to SQL by EF Core.
- The only SQL parameters of the `UPDATE` come from the source query.

### Complex types (since 10.6.0)

The target selector can go through complex properties (`ComplexProperty`, class or
struct, nested allowed), or it can name a whole complex property:

```csharp
await ctx.Set<Message>().BulkUpdateAsync(
   tempTable.Query,
   e => new { e.Id, e.MessageTimestamp },
   f => new { f.Id, f.MessageTimestamp },
   b => b.Set(e => e.EffectiveMessage.ReceivedAt, (e, f) => f.EffectiveMessage.ReceivedAt)); // single member

// whole complex property, value from the source
b => b.Set(e => e.EffectiveMessage, (e, f) => f.EffectiveMessage)

// whole complex property, value from a captured variable
var effectiveMessage = new TimestampPair(7, 8);
b => b.Set(e => e.EffectiveMessage, (e, f) => effectiveMessage)
```

- A whole complex property expands into one `SET` assignment per member column,
  recursively for nested complex types.
- The value of a whole complex property is the source complex property or a
  captured variable. A `new` expression is not evaluated on the client; assign it
  to a variable first.
- Members with a value converter work; a captured value is converted with the
  converter of the target column.
- A whole complex property skips members whose value the database generates
  (`GetBeforeSaveBehavior() == Ignore`, e.g. computed columns); set such a member
  explicitly to write it. When every writable member of a whole complex property
  is generated: `ArgumentException` "No column can be written, because every
  member of the selected complex properties is generated
  (PropertySaveBehavior.Ignore). Expression: <that selector>", also when other
  `Set` entries exist.
- `NotSupportedException`:
  - complex collection ("The complex collection '…' is not supported.");
  - nullable complex property as a whole ("The nullable complex property '…'
    cannot be set as a whole."); a member through a nullable complex property,
    class or struct (`e => e.OptionalBoundary!.Upper`,
    `e => e.Boundary.Value.Upper`), throws "The member '…' of '…' is a
    nullable complex property.";
  - value of a nullable type (e.g. a captured `TimestampPair?`) for a whole
    complex property ("The value of the whole complex property must not be of
    the nullable type '…'.");
  - target selector through a member that is not a complex property, e.g. a
    navigation ("The member '…' is not a complex property of '…'");
  - target selector that is not a member chain on the lambda parameter, e.g.
    `e => e.Count + 1` ("Only simple property access expressions (e => e.Property),
    member access through complex properties (e => e.ComplexProperty.Property) or
    EF.Property calls on the lambda parameter are supported.");
  - `EF.Property` not on the lambda parameter, e.g.
    `EF.Property<long>(e.EffectiveMessage, "ReceivedAt")` ("EF.Property is
    supported only on the lambda parameter, not through a complex property, a
    navigation or another expression."); use member access instead.
- `InvalidOperationException` "The property '…' is not a mapped property on '…'."
  for an unmapped member; for a complex member it names the complex type.
- `ArgumentException`:
  - captured `null` as the value of a whole complex property, or any member access
    on a captured `null` (also in `filter`): "The member '…' of '…' cannot be read
    because the captured value is null."; pass a non-null value or set the members
    one by one. Captured values are evaluated member by member, so
    `captured.HasValue` on a `null` `int?` and `captured == null ? 0 : captured.Count`
    throw as well; check for `null` before the call.
  - same target column set twice, also a whole complex property plus one of its
    members ("The target column '…' is set more than once."), on all providers.
- Nested complex types need settable or `init` members, because EF Core cannot
  bind a nested complex type to a constructor parameter.

### Source can be a temp table

`tempTable.Query` is a valid source. For a temp-table-only row type, register it
via `ConfigureTempTableEntity<T>()` (see `Temp-Tables.md`) so its column names
resolve correctly:

```csharp
await using var tempTable = await ctx.BulkInsertIntoTempTableAsync(rows);

await ctx.Set<Product>().BulkUpdateAsync(
   tempTable.Query,
   e => e.Id,
   f => f.Id,
   b => b.Set(e => e.Count, (e, f) => e.Count + f.Count));
```

### Override the target table

```csharp
await ctx.Set<Product>().BulkUpdateAsync(
   sourceQuery, e => e.Id, f => f.Id,
   b => b.Set(e => e.Name, (e, f) => f.Name),
   options: new SqlServerBulkUpdateFromQueryOptions { TableNameProvider = IBulkOperationTableNameProvider.ForRootEntity("Products_Staging", "dbo") });
   // PostgreSQL: new NpgsqlBulkUpdateFromQueryOptions { ... }
   // SQLite:     new SqliteBulkUpdateFromQueryOptions { ... }
```

## Notes & pitfalls

- **Enable the feature** (`AddBulkOperationSupport()`), or the method throws at
  runtime.
- This is **`DbSet<TTarget>.BulkUpdateAsync(IQueryable source, …)`** — a
  *different* API from `DbContext.BulkUpdateAsync(collection)` (the in-memory
  variant, see `Bulk-Update.md`). Call `ctx.Set<T>()` first to get the `DbSet`.
- You must configure **at least one** `Set` assignment, otherwise it throws
  `ArgumentException`.
- **When to use this vs alternatives:** join another query / composite-key join →
  this API; single-table simple `Where` → EF Core native `ExecuteUpdateAsync`;
  a collection in memory → `Bulk-Update.md`.
- The `filter` and the `Set` value expressions share the same translator.
- SQL Server emits `UPDATE t SET … FROM target INNER JOIN (subquery) AS s ON …`;
  PostgreSQL/SQLite emit `UPDATE target AS t SET … FROM (subquery) AS s WHERE …`.
- Returns `Task<int>` — the number of updated rows (`0` if the filter excludes
  everything).
