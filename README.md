# Serilog.Sinks.AzureBlobStorage

![Build status](https://dev.azure.com/cloudscope/Open%20Source/_apis/build/status/SeriLog-AzureBlobSink%20release "Build status")
[![NuGet Badge](https://buildstats.info/nuget/Serilog.Sinks.AzureBlobStorage)](https://www.nuget.org/packages/Serilog.Sinks.AzureBlobStorage/)

Writes to a file in [Windows Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/).

Azure Blob Storage offers [appending blobs](https://docs.microsoft.com/en-us/rest/api/storageservices/understanding-block-blobs--append-blobs--and-page-blobs/), which allow you to add content quickly to a single blob without locking it for updates. For this reason, appending blobs are ideal for logging applications.

The AzureBlobStorage sink appends data to the blob in text format. Here's a sample line:

```
[2018-10-17 23:03:56 INF] Hello World!
```

**Package** - [Serilog.Sinks.AzureBlobStorage](http://nuget.org/packages/serilog.sinks.azureblobstorage) | **Platforms** - .Net Standard 2.0

**Usage**

```csharp
var cloudAccount = CloudStorageAccount.Parse("ConnectionString");

var log = new LoggerConfiguration()
    .WriteTo.AzureBlobStorage(cloudAccount)
    .CreateLogger();
```

You can also specify a named connection string, using the connectionStringName property in the constructor.

You must also provide an IConfiguration instance, either manually in the constructor or through dependency injection.

This example uses a named connection string called "myConnection".

```csharp

var builder = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddEnvironmentVariables();
IConfigurationRoot config = builder.Build();

var log = new LoggerConfiguration()
    .WriteTo.AzureBlobStorage(connectionStringName: "myConnection", config)
    .CreateLogger();
```

You can avoid manually creating an IConfiguration if you use [Two-stage Initialization](https://github.com/serilog/serilog-aspnetcore#two-stage-initialization) or you have established dependency injection for IConfiguration.

```csharp

public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseSerilog((context, services, configuration) => configuration
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.AzureBlobStorage(connectionStringName: "MyConnectionString", context.Configuration)
        )
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

**_Other options_**

In addition to the storage connection, you can also specify:

- Message line format (default: [{Timestamp:yyyy-MM-dd HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception})
- Blob container (default: logs)
- Blob filename (default: log.txt)

### Configuration examples

#### Rolling file example

By default, the log file name is logs.txt, but you can add date substitutions to create a rolling file implementation. These are more fully shown in the
[Unit Test](https://github.com/chriswill/serilog-sinks-azureblobstorage/blob/master/tests/Serilog.Sinks.AzureBlobStorage.UnitTest/BlobNameFactoryUT.cs)
project. But as an example, you can create a log file name like this: {yyyy}/{MM}/{dd}/log.txt

```csharp
  .WriteTo.AzureBlobStorage(cloudAccount, Serilog.Events.LogEventLevel.Information, storageFileName: "{yyyy}/{MM}/{dd}/log.txt")
```

On December 15, 2018 (when this was written), log files would appear to be in a folder structure as shown below:

```

\2018
-----\12
      ----\15
            log.txt

```

As of version 2.0.0, the values are not required to appear in descending order, e.g.: yy MM dd hh mm. In addition, the values can appear more than once. For example, this is a valid format string which will create the following file name:

```
{yyyy}/{MM}/{dd}/{yyyy}-{MM}-{dd}_{HH}:{mm}.txt
2019/06/20/2019-06-20_14:40.txt
```

#### Maximum file size

You can limit the size of each file created as of version 2.0.0. There is a constructor parameter called `blobSizeLimitBytes`. By
default, this is null, meaning that files can grow without limitation. By providing a value, you can specify the maximum size of a file. Logging more than this amount will cause a new file to be created.

#### Maximum number of files per container

You can limit the number of files created as of version 2.1.0. There is a constructor parameter called `retainedBlobCountLimit` to control this behavior. Once the limit is reached, a file will
be deleted every time a new file is created in order to stay within this limit.

#### Batch posting example

By default, whenever there is a new event to post, the Azure Blob Storage sink will send it to Azure storage. For cost-management or performance reasons, you can
choose to "batch" the posting of new log events.

You should create the sink by calling the [AzureBatchingBlobStorageSink](https://github.com/chriswill/serilog-sinks-azureblobstorage/blob/master/src/Serilog.Sinks.AzureBlobStorage/Sinks/AzureBlobStorage/AzureBatchingBlobStorageSink.cs) class, which inherits from PeriodicBatchingSink.

An example configuration is:

```csharp
  .WriteTo.AzureBlobStorage(blobServiceClient, Serilog.Events.LogEventLevel.Information, writeInBatches:true, period:TimeSpan.FromSeconds(15), batchPostingLimit:10)
```

This configuration would post a new batch of events every 15 seconds, unless there were 10 or more events to post, in which case they would post before the time limit.

To specify batch posting using configuration, configure use need to set these mandatory values:

```json
"WriteTo": [
    {
        "Name": "AzureBlobStorage",
        "Args": {
            "connectionString": "",
            "storageContainerName": "",
            "storageFileName": "",
            "writeInBatches": "true", // mandatory
            "period": "0.00:00:30", // mandatory sets the period to 30 secs
            "batchPostingLimit": "50", // optional
        }
    }
  ]
```

### Development

Do not use the Azure Storage Emulator as a development tool, because it does not support Append Blobs. Instead, use [Azurite](https://github.com/Azure/Azurite), which is Microsoft's new tool for local storage emulation.

### JSON configuration

It is possible to configure the sink using [Serilog.Settings.Configuration](https://github.com/serilog/serilog-settings-configuration) by specifying the folder and file name and connection string in `appsettings.json`:

```json
"Serilog": {
   "Using": [
      "Serilog.Sinks.AzureBlobStorage"
   ],
  "WriteTo": [
    {"Name": "AzureBlobStorage", "Args": {"connectionString": "", "storageContainerName": "", "storageFileName": ""}}
  ]
}
```

Example of authentication using Managed identity:

```json
"Serilog": {
   "Using": [
      "Serilog.Sinks.AzureBlobStorage"
   ],
  "WriteTo": [
    {"Name": "AzureBlobStorage", "Args": { "formatter": "Serilog.Formatting.Json.JsonFormatter", "storageAccountUri": "", "storageContainerName": "", "storageFileName": ""}}
  ]
}
```

JSON configuration must be enabled using `ReadFrom.Configuration()`; see the [documentation of the JSON configuration package](https://github.com/serilog/serilog-settings-configuration) for details.

### XML `<appSettings>` configuration

To use the file sink with the [Serilog.Settings.AppSettings](https://github.com/serilog/serilog-settings-appsettings) package, first install that package if you haven't already done so:

```powershell
Install-Package Serilog.Settings.AppSettings
```

Instead of configuring the logger in code, call 'ReadFrom.AppSettings()':

```csharp
var log = new LoggerConfiguration()
    .ReadFrom.AppSettings()
    .CreateLogger();
```

In your application's 'App.config' or 'Web.config' file, specify the file sink assembly and required path format under the '<appSettings>' node:

```xml
<configuration>
  <appSettings>
    <add key="serilog:using:AzureBlobStorage" value="Serilog.Sinks.AzureBlobStorage" />
    <add key="serilog:write-to:AzureBlobStorage.connectionString" value="DefaultEndpointsProtocol=https;AccountName=ACCOUNT_NAME;AccountKey=KEY;EndpointSuffix=core.windows.net" />
    <add key="serilog:write-to:AzureBlobStorage.formatter" value="Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact" />
```

### Using Managed Identity

Managed identity allows you to access blob storage using either a system-assigned or user-assigned managed identity, rather than a connection string.

If you are using a system-assigned managed identity, provide the storageAccountUri argument as shown in the example above. To use a user-assigned managed identity, retrieve the
identity value from your AppService or Virtual Machine configuration and provide it using the managedIdentityClientId parameter.

For more information on Managed Identity, please visit [Managed identities for Azure resources](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview).

### Acknowledgements

This is a fork of the Serilog [Azure Table storage sink](https://github.com/serilog/serilog-sinks-azuretablestorage). Thanks
and acknowledgements to the original authors of that work.
