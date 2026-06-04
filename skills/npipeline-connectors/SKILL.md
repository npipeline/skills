---
name: npipeline-connectors
description: Use when the user wants to connect NPipeline pipelines to external data sources or destinations. Covers all built-in connectors (CSV, JSON, Parquet, Excel, Postgres, SQL Server, MySQL, MongoDB, DuckDB, Snowflake, CosmosDB, Kafka, RabbitMQ, AWS SQS, Azure Service Bus, HTTP, Data Lake) and storage providers (S3, Azure Blob, ADLS Gen2, GCS, SFTP, S3-Compatible). Use when user mentions "read CSV", "connect to Postgres", "Kafka source", "write to S3", "storage provider", "connector", or any database/messaging/file format name.
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline Connectors

This skill covers the built-in connector packages that provide pre-built source and sink nodes for external systems.

## Workflow

When the user wants to read from or write to an external system, identify the connector type, confirm the NuGet package, then guide them through registration.

### Phase 1: Identify the Connector

NPipeline provides three categories of connectors — determine which one the user needs:

**File Format Connectors** — Read/write files in specific formats.
**Database Connectors** — Source/sink nodes for relational and NoSQL databases.
**Messaging Connectors** — Stream sources/sinks for message queues and event streams.
**Storage Providers** — Abstract file access across cloud and on-premise storage backends.

Consult `references/connectors-index.md` for the full catalog of available connectors with their NuGet packages and key types.

### Phase 2: Install the Package

Each connector is a separate NuGet package:

```
dotnet add package NPipeline.Connectors.Csv
dotnet add package NPipeline.Connectors.Postgres
dotnet add package NPipeline.Connectors.Kafka
```

### Phase 3: Use the Connector

All connectors follow the same pattern — use as source or sink nodes in `PipelineBuilder`:

```csharp
// File format source
var source = builder.AddSource<CsvSourceNode<Order>, Order>("read-csv");

// Database sink
var sink = builder.AddSink<PostgresSinkNode<EnrichedOrder>, EnrichedOrder>("save-to-postgres");

// Message queue source
var source = builder.AddSource<KafkaSourceNode<Order>, Order>("consume-orders");
```

Connector nodes support constructor injection and access `PipelineContext` for configuration (connection strings, file paths, etc.).

### Storage Providers

Storage providers abstract file I/O across cloud backends. Use them when connectors need to read/write files from cloud storage:

```
dotnet add package NPipeline.StorageProviders.S3
```

See `references/storage-providers.md` for the list of storage providers.
