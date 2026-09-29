# Storage Providers

Storage providers abstract file I/O across cloud and on-premise backends. File connectors (CSV, JSON, Parquet, Excel) use storage providers so the same pipeline code works with local files, cloud storage, and SFTP.

## Available Providers

| Provider | NuGet Package | Backend | DI extension |
|---|---|---|---|
| File system | `NPipeline.StorageProviders` | Local files | (built-in) |
| AWS S3 | `NPipeline.StorageProviders.S3.Aws` | Amazon S3 | `AddAwsS3StorageProvider` |
| S3-Compatible | `NPipeline.StorageProviders.S3.Compatible` | MinIO, Ceph, DigitalOcean Spaces, R2 | `AddS3CompatibleStorageProvider` |
| Azure Blob | `NPipeline.StorageProviders.Azure` | Azure Blob Storage | `AddAzureBlobStorageProvider` |
| ADLS Gen2 | `NPipeline.StorageProviders.Adls` | Azure Data Lake Storage Gen2 | `AddAdlsGen2StorageProvider` |
| Google Cloud Storage | `NPipeline.StorageProviders.Gcp` | GCS | `AddGcsStorageProvider` |
| SFTP | `NPipeline.StorageProviders.Sftp` | SSH File Transfer | `AddSftpStorageProvider` |

Custom providers implement `IStorageProvider` (and optionally `IDeletableStorageProvider`, `IMoveableStorageProvider`, `IConfigurableStorageProvider`) from `NPipeline.StorageProviders`.

## Usage Pattern

Pass the provider to a file connector through its options, alongside a `StorageUri`:

```csharp
using NPipeline.Connectors.Json;
using NPipeline.StorageProviders.Models;
using NPipeline.StorageProviders.S3.Aws;

var storageProvider = new AwsS3StorageProvider(new AwsS3StorageProviderOptions
{
    DefaultRegion = RegionEndpoint.USEast1,
});

var source = JsonConnector.Source<Order>(
    StorageUri.Parse("s3://bucket/orders.ndjson"),
    o => o with { Provider = storageProvider });
```

Local files need neither a provider nor a resolver. When `Provider` is `null`, the connector resolves one from `Resolver`; when that is also `null`, a default resolver with the file system is used.

> [!IMPORTANT]
> Storage configuration is passed through the connector's options (`Provider`, `Resolver`) or through DI, not through `PipelineContext.Parameters`. Set `Provider` on the connector's options, or register providers with DI and let the resolver select by URI scheme.

## Resolver

A `StorageResolver` selects the provider by URI scheme when several are registered. With DI, registering providers through their extension methods lets the connector resolve the right one from the URI:

```csharp
services.AddAwsS3StorageProvider(o => { /* configure credentials/region */ });
services.AddAzureBlobStorageProvider(o => { /* configure */ });
```

## Custom Storage Providers

Implement `IStorageProvider` from `NPipeline.StorageProviders` to support a custom backend. The base package provides the common interface all file connectors use.
