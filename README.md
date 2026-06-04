# NPipeline Skills

Agent skills for working with [NPipeline](https://github.com/npipeline/npipeline) — High-performance, streaming data pipelines for .NET.

## Install

Install skills into your coding agent using [skills.sh](https://skills.sh):

```bash
npx skills@latest add npipeline/skills
```

Then pick the skills you want and the agents you want to install them on.

### Available Skills

| Skill | Description |
|-------|-------------|
| **npipeline-pipeline-authoring** | Full lifecycle of creating pipelines: `IPipelineDefinition`, `PipelineBuilder`, connecting nodes, lambda nodes, running with `PipelineRunner`, `PipelineContext`, and dependency injection |
| **npipeline-node-development** | Writing custom node classes: `SourceNode<T>`, `TransformNode<TIn,TOut>`, `IStreamTransformNode<TIn,TOut>`, `SinkNode<T>`, and key conventions |
| **npipeline-connectors** | All built-in connectors (CSV, JSON, Parquet, Excel, Postgres, SQL Server, MySQL, MongoDB, DuckDB, Snowflake, CosmosDB, Kafka, RabbitMQ, AWS SQS, Azure Service Bus, HTTP, Data Lake) and storage providers |
| **npipeline-data-flow** | Advanced routing patterns: branching, fan-out, taps, `RouteNode`, joins, aggregation (tumbling/sliding windows), batching/unbatching, and pipeline composition |
| **npipeline-ai** | AI/LLM integration via `Microsoft.Extensions.AI.IChatClient`: transform, enrich, batched/stream variants, routing, and provider setup |
| **npipeline-resilience** | Three-tier failure model (item/node/pipeline), retry, circuit breakers, custom `IResiliencePolicy`, and dead-letter queues |
| **npipeline-observability** | Metrics, structured logging, and OpenTelemetry distributed tracing (Jaeger, Zipkin, Azure Monitor, AWS X-Ray, OTLP) |
| **npipeline-lineage** | Data provenance tracking: `FastLineage` (sampled) and `CompleteLineage` (full detail), `LineagePacket<T>`, correlation IDs |
| **npipeline-performance** | Optimization profiles (`Default` vs `HighThroughput`), ValueTask fast paths, execution strategies, parallelism, and common pitfalls |
| **npipeline-testing** | Unit testing nodes, integration testing with `PipelineTestHarness`, error path testing, and assertion helpers |
| **npipeline-utility-nodes** | Validation, cleansing, filtering, type conversion, and enrichment nodes with fluent builder APIs |

## Install a Single Skill

```bash
npx skills@latest add npipeline/skills@npipeline-ai
```

## Supported Agents

Works with any agent supported by [skills.sh](https://skills.sh): Claude Code, Cursor, Codex, Windsurf, Cline, OpenCode, and many more.
