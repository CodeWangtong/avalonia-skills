# Persistence — FreeSql / SQLite / MySQL / Mapster

> **Router**: `SKILL.md`. **~85 lines.** Keywords: `IFreeSql`, monthly shard, `IOrderReadRepository`, isolated config DB.
>
> Read path via repositories ( **`mvvm-toolkit.md`** ); write/handlers (**`messaging-logging-validation.md`**).

## Policy

| Topic | Choice |
|-------|--------|
| ORM | **FreeSql** |
| Default engine | **SQLite** |
| Alternate | **MySQL** |
| **Configuration data** | **Separate DB** (path or catalog name isolated from transactional data) |
| **Heavy monthly rollover** | **Monthly connection shard** (`yyyy-MM` file or schema) |

## Mapster (boundaries)

Call **`TypeAdapterConfig`** once at startup; DTO ↔ entity at **repository / façade** tier.

```csharp
TypeAdapterConfig<OrderDto, OrderEntity>.NewConfig()
    .Ignore(dest => dest.Id);
```

## FreeSql: shard example

Register **config** vs **operational/monthly** **`IFreeSql`** separately:

```csharp
public sealed class OperationalDatabaseRouter
{
    public static string ShardKeyUtc(DateTime utc) =>
        utc.ToUniversalTime().ToString("yyyy-MM", System.Globalization.CultureInfo.InvariantCulture);
}

public sealed class BusinessSqlFactory : IBusinessSqlFactory
{
    private readonly System.Collections.Concurrent.ConcurrentDictionary<string, IFreeSql> _cache = new();
    private readonly OperationalDatabaseRouter _router = new();

    public IFreeSql ForMonth(DateTime utc)
    {
        var key = OperationalDatabaseRouter.ShardKeyUtc(utc);
        return _cache.GetOrAdd(key, CreateShard);
    }

    private IFreeSql CreateShard(string yyyyMm)
    {
        var root = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            "MyApp", "biz");
        Directory.CreateDirectory(root);
        var conn = $"Data Source={Path.Combine(root, $"{yyyyMm}.db")};";
        return new FreeSqlBuilder()
            .UseConnectionString(DataType.Sqlite, conn)
            .UseAutoSyncStructure(true)
            .Build();
    }
}
```

Avoid **`UseAutoSyncStructure`** in production without review—prefer migrations.

## Repositories & façades

- **`IOrdersQuery`** → **`IOrderReadRepository.ListRowsAsync`**
- **Handlers** → **`IOrderWriteRepository`** / transactional wrapper

Never push **telemetry-scale** payloads into the **configuration** database.

## MySQL

```csharp
.UseConnectionString(DataType.MySql, configuration["ConnectionStrings:Orders"]!)
```

Monthly naming convention example: **`_2026_05`** database suffix.

## MediatR + repositories

Handlers receive **`IFreeSql` for current month** or **`factory.ForMonth(...)`**, using **UTC month keys** aligned with rollover rules.
