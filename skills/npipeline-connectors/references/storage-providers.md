# Storage Providers

Storage providers abstract file I/O across cloud and on-premise backends. Connectors use storage providers to read/write files regardless of the underlying storage system.

## Available Providers

| Provider | NuGet Package | Backend |
|---|---|---|
| AWS S3 | `NPipeline.StorageProviders.S3.Aws` | Amazon S3 |
| S3-Compatible | `NPipeline.StorageProviders.S3.Compatible` | MinIO, Ceph, DigitalOcean Spaces, etc. |
| Azure Blob | `NPipeline.StorageProviders.Azure` | Azure Blob Storage |
| ADLS Gen2 | `NPipeline.StorageProviders.Adls` | Azure Data Lake Storage Gen2 |
| Google Cloud Storage | `NPipeline.StorageProviders.Gcp` | GCS |
| SFTP | `NPipeline.StorageProviders.Sftp` | SSH File Transfer |

Base package: `NPipeline.StorageProviders` (common abstractions).

## Usage Pattern

Storage providers are used by file format connectors (CSV, JSON, Parquet) to locate and access files:

```csharp
// Configure S3 storage for CSV connector
var context = new PipelineContext(new PipelineContextConfiguration(
    Parameters: new Dictionary<string, object>
    {
        ["StorageProvider"] = "S3",
        ["BucketName"] = "my-bucket",
        ["FilePath"] = "data/orders.csv",
        ["Region"] = "us-east-1"
    }
));

var source = builder.AddSource<CsvSourceNode<Order>, Order>("read-from-s3");
```

## Custom Storage Providers

Implement the storage provider abstraction from `NPipeline.StorageProviders` to support custom backends. The base package provides the common interface that all file format connectors use.
