---
name: npipeline-connectors
description: Use when the user wants to connect NPipeline pipelines to external data sources or destinations. Covers all built-in connectors (CSV, JSON, Parquet, Excel, Postgres, SQL Server, MySQL, MongoDB, DuckDB, Snowflake, CosmosDB, Kafka, RabbitMQ, AWS SQS, Azure Service Bus, HTTP, Data Lake) and storage providers (S3, Azure Blob, ADLS Gen2, GCS, SFTP, S3-Compatible). Use when user mentions "read CSV", "connect to Postgres", "Kafka source", "write to S3", "storage provider", "connector", or any database/messaging/file format name.
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Connectors

This skill covers the built-in connector packages that provide pre-built source and sink nodes for external systems.

## Workflow

When the user wants to read from or write to an external system, identify the connector type, confirm the NuGet package, then guide them through registration.

### Phase 1: Identify the Connector

NPipeline provides these categories. Each is a separate NuGet package.

| Category | Connectors |
|---|---|
| **File formats** | CSV (`NPipeline.Connectors.Csv`), JSON, Parquet, Excel |
| **Databases** | PostgreSQL, SQL Server, MySQL, Snowflake, MongoDB, Cosmos DB, DuckDB |
| **Messaging** | Kafka, RabbitMQ, AWS SQS, Azure Service Bus |
| **HTTP** | REST APIs (pagination, auth, rate limiting) |
| **Data Lake** | Hive-style partitioned Parquet tables |
| **Storage providers** | File system, S3, Azure Blob, ADLS Gen2, GCS, SFTP, S3-compatible |

Consult `references/connectors-index.md` for the catalog and per-connector usage.

### Phase 2: Install the Package

```
dotnet add package NPipeline.Connectors.Csv
dotnet add package NPipeline.Connectors.Postgres
dotnet add package NPipeline.Connectors.Kafka
```

### Phase 3: Use the Connector

> [!IMPORTANT]
> A connector node is constructed first (from a factory helper or connector-specific configuration), then added with `builder.AddSource(nodeInstance, "name")` or `builder.AddSink(nodeInstance, "name")`. The `builder.AddSource<CsvSourceNode<Order>, Order>(...)` form only works for node types with a parameterless constructor.

**File connectors** CSV, JSON, and Excel expose a `XConnector` factory and `*Options` records; Parquet uses `ParquetSourceNode<T>` / `ParquetSinkNode<T>` with `ParquetConfiguration`. The URI can be a file, a directory, or a glob.

```csharp
using NPipeline.Connectors.Csv;
using NPipeline.StorageProviders.Models;

var source = CsvConnector.Source<Order>(
    StorageUri.Parse("s3://bucket/orders/*.csv"),
    o => o with { Delimiter = ";" });

builder.AddSource(source, "read-csv");
```

**HTTP** uses `HttpConnector` with `HttpSourceOptions<T>` / `HttpSinkOptions<T>` and an `HttpClient` or `IHttpClientFactory`:

```csharp
var source = HttpConnector.Source<Order>(
    new Uri("https://api.example.com/orders"),
    httpClient,
    o => o with { ItemsJsonPath = "data" });

builder.AddSource(source, "read-orders");
```

**Database and messaging** connectors expose `XSourceNode<T>` / `XSinkNode<T>` classes configured with a `XConfiguration` object or via DI (`services.AddPostgresConnector(...)`, `services.AddMongoConnector(...)`, and so on):

```csharp
using NPipeline.Connectors.Postgres;

var config = new PostgresConfiguration { ConnectionString = "Host=...;Database=orders;..." };
var source = new PostgresSourceNode<Order>(
    "SELECT id, customer, amount FROM orders WHERE status = 'pending'",
    config,
    row => new Order(row.Get<int>("id"), row.Get<string>("customer")!, row.Get<decimal>("amount")));

builder.AddSource(source, "read-orders");
```

### Storage Providers

Storage providers abstract file I/O across cloud backends; file connectors take one through their options (`Provider`) or resolve it from the URI. See `references/storage-providers.md`.

### Configuration Notes

- File-format options are immutable records created with `with` expressions. Each connector's `ReadOptions`/`WriteOptions` extends shared file options: `Provider`, `Resolver`, `Compression`, `BufferSize`, `Recursive`, `RowErrorHandler`, `RawExcerptLength` (sources), and `AtomicWrite`, `NullItems`, `DeletePartialOnFailure` (sinks).
- Database connection strings and credentials are configured on the connector's configuration object or through DI.
- `PipelineContext` is available to connector nodes, but connector configuration is supplied through options/configuration objects rather than context dictionaries.

Consult `references/connectors-index.md` and `references/storage-providers.md` for details.
