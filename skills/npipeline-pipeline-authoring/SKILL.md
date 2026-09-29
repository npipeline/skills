---
name: npipeline-pipeline-authoring
description: Use when the user wants to create, define, or build an NPipeline data processing pipeline. Covers IPipelineDefinition, PipelineBuilder, connecting nodes, lambda nodes, running pipelines, PipelineContext, PipelineRunner, and dependency injection integration. Also use when the user mentions "create a pipeline", "define a pipeline", "build a pipeline", "add nodes", "IPipelineDefinition", "PipelineBuilder", "run a pipeline", or "AddNPipeline".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Pipeline Authoring

This skill covers the full lifecycle of authoring an NPipeline pipeline: defining the DAG structure, configuring the builder, integrating with dependency injection, and running the pipeline.

## Workflow

When a user wants to author a pipeline, follow this phased approach:

### Phase 1: Gather Requirements

Ask the user to describe their data processing needs. Determine:

1. **Data sources** — Where does the data come from? (file, database, API, message queue, in-memory collection)
2. **Transformations** — What operations need to happen? (validation, cleansing, enrichment, filtering, type conversion, aggregation)
3. **Data flow shape** — Is this a linear pipeline, or does it branch/merge/route?
4. **Output sinks** — Where does the data go? (database, file, API, message queue)
5. **Error handling requirements** — Should the pipeline retry failures? Skip bad items? Use dead-letter queues?

### Phase 2: Design the Graph

Map the requirements to NPipeline concepts:

- Each data source = a **Source Node** (`ISourceNode<TOut>`)
- Each transformation step = a **Transform Node** (`ITransformNode<TIn, TOut>`) or **Stream Transform** (`IStreamTransformNode<TIn, TOut>`)
- Each output destination = a **Sink Node** (`ISinkNode<TIn>`)
- Branches/merges = multiple `Connect()` calls from a single source
- Conditional routing = **Route Node** (see `npipeline-data-flow` skill)
- Sub-pipelines = **Composite Nodes** (see `npipeline-data-flow` skill)

For resilience, error handling, and optimization, see the `npipeline-resilience` and `npipeline-performance` skills.

### Phase 3: Implement the Pipeline Definition

Create a class implementing `IPipelineDefinition`:

```csharp
using NPipeline.Pipeline;

public class MyPipeline : IPipelineDefinition
{
    public void Define(PipelineBuilder builder, PipelineContext context)
    {
        // See reference files for full builder API
    }
}
```

The `Define` method is called once per execution. New node instances are created for each run.

Consult `references/builder-api.md` for the complete `PipelineBuilder` API reference.
Consult `references/node-types.md` for all available node types and their registration methods.
Consult `references/di-integration.md` for dependency injection setup.

### Phase 4: Configure and Run

**Simplest usage (no DI):**

```csharp
var runner = PipelineRunner.Create();
await runner.RunAsync<MyPipeline>();
```

**With context and parameters:**

```csharp
var config = new PipelineContextConfiguration(
    CancellationToken: cts.Token,
    Parameters: new Dictionary<string, object> { ["path"] = "/data/input.csv" }
);
var context = new PipelineContext(config);
await runner.RunAsync<MyPipeline>(context, cts.Token);
```

**With dependency injection:**

```csharp
// Registration
services.AddNPipeline(builder => builder
    .AddNode<MyTransform>()
    .AddPipeline<MyPipeline>());

// Or assembly scanning
services.AddNPipeline(typeof(MyPipeline).Assembly);

// Running
var provider = services.BuildServiceProvider();
await provider.RunPipelineAsync<MyPipeline>();
```

Consult `references/configuration.md` for context configuration options.

## Design Principles

NPipeline follows six non-negotiable principles. Keep these in mind when authoring pipelines:

1. **Streaming-first** — Data flows item-by-item via `IAsyncEnumerable<T>`. Nothing is buffered unless explicitly opted in.
2. **Fail-fast defaults** — A failure that is not retried follows `OnItemFailure`, which defaults to `Fail`. The `Default` optimization profile retries transient item failures three times; users opt into skip or dead-letter.
3. **Zero-allocation hot paths** — Per-item processing avoids heap allocations.
4. **Type safety at the graph level** — Typed handles (`SourceNodeHandle<TOut>`, `TransformNodeHandle<TIn,TOut>`, `SinkNodeHandle<TIn>`) prevent connecting incompatible nodes at compile time.
5. **Immutable configuration** — All config records are `sealed record` with `init`-only properties.
6. **Extension points over modification** — Add behavior through interfaces, not by modifying core classes.

## Next Steps

- For writing custom nodes, use the `npipeline-node-development` skill.
- For branching, routing, joins, aggregation, and composition, use the `npipeline-data-flow` skill.
- For error handling and resilience, use the `npipeline-resilience` skill.
- For connecting to external systems, use the `npipeline-connectors` skill.
- For validation, cleansing, filtering, and enrichment built-in nodes, use the `npipeline-utility-nodes` skill.
- For performance optimization and parallel execution, use the `npipeline-performance` skill.
