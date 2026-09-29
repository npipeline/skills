# Connectors Index

## File Format Connectors

CSV, JSON, and Excel are built on one pair of base classes in `NPipeline.Connectors`, so they share options, storage handling, error handling, and metrics. Each exposes a `XConnector` factory with `Source<T>(uri, configure?)` and `Sink<T>(uri, configure?)`, plus `XSourceNode<T>` / `XSinkNode<T>` types that take the options record directly.

| Connector | NuGet Package | Source | Sink | Options |
|---|---|---|---|---|
| CSV | `NPipeline.Connectors.Csv` | `CsvSourceNode<T>` | `CsvSinkNode<T>` | `CsvReadOptions` / `CsvWriteOptions` |
| JSON | `NPipeline.Connectors.Json` | `JsonSourceNode<T>` | `JsonSinkNode<T>` | `JsonReadOptions` / `JsonWriteOptions` |
| Excel | `NPipeline.Connectors.Excel` | `ExcelSourceNode<T>` | `ExcelSinkNode<T>` | `ExcelReadOptions` / `ExcelWriteOptions` |
| Parquet | `NPipeline.Connectors.Parquet` | `ParquetSourceNode<T>` | `ParquetSinkNode<T>` | `ParquetConfiguration` |

### Usage Example (CSV)

```csharp
using NPipeline.Connectors.Csv;
using NPipeline.StorageProviders.Models;

var source = CsvConnector.Source<Order>(StorageUri.FromFilePath("orders.csv"));
var sink = CsvConnector.Sink<EnrichedOrder>(StorageUri.FromFilePath("processed.csv"));

builder.AddSource(source, "read-csv");
builder.AddSink(sink, "write-csv");
```

A URI can be one file (`orders.csv`), a directory (`orders/`), or a glob (`s3://bucket/2026/*.csv`). Sources decompress files ending in `.gz`, `.br` or `.zz`; sinks compress by the same suffixes. Sinks write atomically: on the file system a temporary file is moved into place, and a failed write deletes its partial output.

### Shared file options

| Option | Direction | Default | Description |
|---|---|---|---|
| `Uri` | Both | required | File, directory, or glob |
| `Provider` | Both | `null` | Storage provider; when null, resolved from `Resolver` |
| `Compression` | Both | `Auto` | `Auto` by suffix: `.gz`, `.br`, `.zz`/`.zlib`, `.deflate`; or an explicit value |
| `BufferSize` | Both | 64 KB | Reader/writer buffer |
| `Recursive` | Source | `false` | Include subdirectories |
| `RowErrorHandler` | Source | fails the read | `RowError` → `Fail`, `Skip`, or `DeadLetter` |
| `RawExcerptLength` | Source | 256 | Characters of raw record kept in errors; `0` omits raw data |
| `AtomicWrite` | Sink | `Auto` | `Auto`, `Always`, `Never` |
| `NullItems` | Sink | `Throw` | `Throw`, `Skip`, or `Write` for a null item |
| `DeletePartialOnFailure` | Sink | `true` | Delete partial output when a write fails |

Column binding uses member names case-insensitively (via `[Column]` / `[IgnoreColumn]` and a `ColumnNamingPolicy`); `MissingColumns` decides what a missing column means. The connectors emit `System.Diagnostics.Metrics` and `ActivitySource` instruments named `NPipeline.Connectors`.

## Database Connectors

| Connector | NuGet Package | Type | DI Extension |
|---|---|---|---|
| PostgreSQL | `NPipeline.Connectors.Postgres` | Relational | `AddPostgresConnector` |
| SQL Server | `NPipeline.Connectors.SqlServer` | Relational | `AddSqlServerConnector` |
| MySQL | `NPipeline.Connectors.MySQL` | Relational | `AddMySqlConnector` |
| DuckDB | `NPipeline.Connectors.DuckDB` | Embedded OLAP | `AddDuckDBConnector` |
| Snowflake | `NPipeline.Connectors.Snowflake` | Cloud warehouse | `AddSnowflakeConnector` |
| MongoDB | `NPipeline.Connectors.MongoDB` | Document | `AddMongoConnector` |
| Azure Cosmos DB | `NPipeline.Connectors.Azure.CosmosDb` | Document | `AddCosmosDbConnector` |

Database connectors provide `XSourceNode<T>` (query-based streaming) and `XSinkNode<T>` (insert/upsert), configured with an `XConfiguration` object or through DI. PostgreSQL and SQL Server include Roslyn analyzers that validate queries at build time.

### Usage Example (PostgreSQL)

```csharp
using NPipeline.Connectors.Postgres;

var config = new PostgresConfiguration
{
    ConnectionString = "Host=localhost;Database=orders;Username=app;Password=secret"
};

var source = new PostgresSourceNode<Order>(
    "SELECT id, customer, amount FROM orders WHERE status = 'pending'",
    config,
    row => new Order(row.Get<int>("id"), row.Get<string>("customer")!, row.Get<decimal>("amount")));

var sink = new PostgresSinkNode<EnrichedOrder>(config);
// ... connect and run
```

## Messaging Connectors

| Connector | NuGet Package | Source | Sink |
|---|---|---|---|
| Kafka | `NPipeline.Connectors.Kafka` | `KafkaSourceNode<T>` (emits `KafkaMessage<T>`) | `KafkaSinkNode<T>` |
| RabbitMQ | `NPipeline.Connectors.RabbitMQ` | `RabbitMqSourceNode<T>` | `RabbitMqSinkNode<T>` |
| AWS SQS | `NPipeline.Connectors.Aws.Sqs` | `SqsSourceNode<T>` | `SqsSinkNode<T>` |
| Azure Service Bus | `NPipeline.Connectors.Azure.ServiceBus` | `ServiceBusQueueSourceNode<T>` / `ServiceBusSubscriptionSourceNode<T>` | `ServiceBusQueueSinkNode<T>` / `ServiceBusTopicSinkNode<T>` |

Messaging connectors provide long-lived consumers and producers with at-least-once delivery. Configure them with a `XConfiguration` object or a `services.AddXConnector(...)` extension.

> [!NOTE]
> `KafkaSourceNode<T>` is a `SourceNode<KafkaMessage<T>>`: downstream nodes receive `KafkaMessage<T>` (key, value, headers, offset, and acknowledgment), not bare `T`.

## HTTP Connector

`NPipeline.Connectors.Http` (package `NPipeline.Connectors.Http`) reads from and writes to REST APIs. It supports page-number, offset, cursor, `Link`-header, next-URL, and custom pagination, auth providers, and a `System.Threading.RateLimiting.RateLimiter`.

```csharp
using NPipeline.Connectors.Http;

var source = HttpConnector.Source<Order>(
    new Uri("https://api.example.com/orders"),
    httpClient,
    o => o with
    {
        ItemsJsonPath = "data",
        Pagination = HttpPagination.Cursor(new CursorPaginationOptions { CursorJsonPath = "meta.next_cursor" }),
        Auth = new BearerTokenAuthProvider(token),
    });

builder.AddSource(source, "orders");
```

## Data Lake Connector

`NPipeline.Connectors.DataLake` provides `DataLakeTableSourceNode<T>` and `DataLakePartitionedSinkNode<T>` for Hive-style partitioned Parquet tables, plus a `DataLakeCompactor`.

## Pattern

All connector nodes are source or sink node instances:

1. **Source** — produces an `IDataStream<T>` (Kafka produces `KafkaMessage<T>`).
2. **Sink** — consumes an `IDataStream<T>`.
3. **Configuration** — a connector-specific configuration/options object, or DI.
4. **Adding to the graph** — `builder.AddSource(node, "name")` / `builder.AddSink(node, "name")`, or `builder.AddSource<CsvSourceNode<T>, T>(...)` only when the node has a parameterless constructor (file and HTTP connectors require options, so they use an instance).
