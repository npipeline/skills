# Connectors Index

## File Format Connectors

| Connector | NuGet Package | Key Source Type | Key Sink Type |
|---|---|---|---|
| CSV | `NPipeline.Connectors.Csv` | `CsvSourceNode<T>` | `CsvSinkNode<T>` |
| JSON | `NPipeline.Connectors.Json` | `JsonSourceNode<T>` | `JsonSinkNode<T>` |
| Parquet | `NPipeline.Connectors.Parquet` | `ParquetSource<T>` | `ParquetSink<T>` |
| Excel | `NPipeline.Connectors.Excel` | `ExcelSource<T>` | `ExcelSink<T>` |

All file format connectors support streaming reads and writes. Configure file paths, delimiters, and formatting via `PipelineContext.Parameters` or constructor injection.

### Usage Example (CSV)

```csharp
dotnet add package NPipeline.Connectors.Csv
```

```csharp
var source = builder.AddSource<CsvSourceNode<Order>, Order>("read-csv");
var sink = builder.AddSink<CsvSinkNode<EnrichedOrder>, EnrichedOrder>("write-csv");
```

## Database Connectors

| Connector | NuGet Package | Type |
|---|---|---|
| PostgreSQL | `NPipeline.Connectors.Postgres` | Relational |
| SQL Server | `NPipeline.Connectors.SqlServer` | Relational |
| MySQL | `NPipeline.Connectors.MySQL` | Relational |
| DuckDB | `NPipeline.Connectors.DuckDB` | Embedded OLAP |
| Snowflake | `NPipeline.Connectors.Snowflake` | Cloud Data Warehouse |
| MongoDB | `NPipeline.Connectors.MongoDB` | Document |
| Azure Cosmos DB | `NPipeline.Connectors.Azure.CosmosDb` | Document |

Database connectors provide source nodes (query-based streaming) and sink nodes (insert/upsert). Connection strings are typically passed via `PipelineContext.Parameters`.

PostgreSQL and SQL Server connectors also include Roslyn analyzers (`*.Analyzers` packages) that validate queries at build time.

### Usage Example (PostgreSQL)

```csharp
dotnet add package NPipeline.Connectors.Postgres
```

```csharp
var context = new PipelineContext(new PipelineContextConfiguration(
    Parameters: new Dictionary<string, object>
    {
        ["ConnectionString"] = "Host=...;Database=orders;..."
    }
));

// Source: streaming query
var source = builder.AddSource<SqlServerSourceNode<Order>, Order>("read-orders");
// Sink: batch insert
var sink = builder.AddSink<SqlServerSinkNode<EnrichedOrder>, EnrichedOrder>("save-orders");
```

## Messaging Connectors

| Connector | NuGet Package | Type |
|---|---|---|
| Kafka | `NPipeline.Connectors.Kafka` | Event Streaming |
| RabbitMQ | `NPipeline.Connectors.RabbitMQ` | Message Broker |
| AWS SQS | `NPipeline.Connectors.Aws.Sqs` | Cloud Queue |
| Azure Service Bus | `NPipeline.Connectors.Azure.ServiceBus` | Cloud Messaging |

Messaging connectors provide streaming sources (long-lived consumers) and sinks (producers). They support at-least-once delivery semantics.

### Usage Example (Kafka)

```csharp
dotnet add package NPipeline.Connectors.Kafka
```

```csharp
var source = builder.AddSource<KafkaSource, Order>("consume");
var sink = builder.AddSink<KafkaSink, EnrichedOrder>("produce");
```

## Other Connectors

| Connector | NuGet Package | Purpose |
|---|---|---|
| HTTP | `NPipeline.Connectors.Http` | REST API source/sink |
| Azure | `NPipeline.Connectors.Azure` | Generic Azure services |
| Data Lake | `NPipeline.Connectors.DataLake` | Azure Data Lake Storage |

## Pattern: All Connectors

All connectors follow this pattern:

1. **Source** — Extend `SourceNode<T>`. Override `OpenStream` to produce items from the external system.
2. **Sink** — Extend `SinkNode<T>`. Override `ConsumeAsync` to write items to the external system.
3. **Configuration** — Connection details via `PipelineContext.Parameters` or constructor injection.
4. **Streaming** — Sources return `IDataStream<T>` for lazy streaming.
