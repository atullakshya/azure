# Azure .NET Developer Q&A: From Basic to Advanced

This document provides a comprehensive set of questions and answers for .NET developers working with Azure services. It covers key Azure applications used for development, explained in a basic manner, with details on when to use them, real-time use cases, and code snippets.

## Basic Level

### 1. What is Azure App Service?
Azure App Service is like a hosting service where you can run your web applications, APIs, and mobile apps without managing the physical servers. It's a platform that handles the infrastructure so you can focus on coding.

**When to use:** Use it when you need to deploy web apps quickly and don't want to worry about server maintenance, scaling, or security patches.

**Real-time use case:** Hosting a simple company website built with ASP.NET Core that employees can access to view company news.

**Code snippet:** Here's a basic ASP.NET Core web app that displays "Hello World":

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Hosting;

public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.Configure(app =>
                {
                    app.Run(async context =>
                    {
                        await context.Response.WriteAsync("Hello World from Azure App Service!");
                    });
                });
            });
}
```

### 2. What is Azure SQL Database?
Azure SQL Database is a cloud-based version of SQL Server. It's a database service where you store and manage your data using SQL queries, just like a regular database but hosted in the cloud.

**When to use:** Use it for applications that need a relational database with structured data, like storing user information, orders, or inventory.

**Real-time use case:** Storing customer data for an e-commerce website where users can register and place orders.

**Code snippet:** Connecting to Azure SQL Database from a .NET app:

```csharp
using System.Data.SqlClient;

string connectionString = "Server=tcp:yourserver.database.windows.net,1433;Initial Catalog=yourdatabase;Persist Security Info=False;User ID=yourusername;Password=yourpassword;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;";

using (SqlConnection connection = new SqlConnection(connectionString))
{
    connection.Open();
    string query = "SELECT * FROM Customers";
    using (SqlCommand command = new SqlCommand(query, connection))
    {
        using (SqlDataReader reader = command.ExecuteReader())
        {
            while (reader.Read())
            {
                Console.WriteLine(reader["CustomerName"]);
            }
        }
    }
}
```

### 3. What is Azure Blob Storage?
Azure Blob Storage is a service for storing large amounts of unstructured data, like files, images, videos, or backups. It's like a big file cabinet in the cloud.

**When to use:** Use it for storing files that don't fit well in a database, such as user-uploaded images, documents, or log files.

**Real-time use case:** Storing profile pictures for a social media app where users upload their photos.

**Code snippet:** Uploading a file to Azure Blob Storage:

```csharp
using Azure.Storage.Blobs;

string connectionString = "DefaultEndpointsProtocol=https;AccountName=youraccount;AccountKey=yourkey;EndpointSuffix=core.windows.net";
string containerName = "images";
string blobName = "profile.jpg";

BlobServiceClient blobServiceClient = new BlobServiceClient(connectionString);
BlobContainerClient containerClient = blobServiceClient.GetBlobContainerClient(containerName);
BlobClient blobClient = containerClient.GetBlobClient(blobName);

using (FileStream uploadFileStream = File.OpenRead("localfile.jpg"))
{
    await blobClient.UploadAsync(uploadFileStream, true);
}
```

## Intermediate Level

### 4. What is Azure Functions?
Azure Functions is a serverless computing service that lets you run small pieces of code (functions) without managing servers. It's like having code that runs only when needed, and you pay only for the time it runs.

**When to use:** Use it for event-driven tasks, like processing data when a file is uploaded or responding to HTTP requests without a full web app.

**Real-time use case:** Automatically resizing images when users upload them to your app.

**Code snippet:** A simple HTTP-triggered Azure Function:

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Logging;

public static class HelloWorldFunction
{
    [FunctionName("HelloWorld")]
    public static IActionResult Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        string name = req.Query["name"];

        string responseMessage = string.IsNullOrEmpty(name)
            ? "Hello, World!"
            : $"Hello, {name}!";

        return new OkObjectResult(responseMessage);
    }
}
```

### 5. What is Azure Key Vault?
Azure Key Vault is a secure storage service for secrets, keys, and certificates. It's like a safe where you keep sensitive information like passwords or encryption keys.

**When to use:** Use it to store and manage secrets securely, especially in production environments where you don't want sensitive data in your code.

**Real-time use case:** Storing database connection strings or API keys for a web application.

**Code snippet:** Retrieving a secret from Azure Key Vault:

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

string keyVaultUrl = "https://yourkeyvault.vault.azure.net/";
string secretName = "mySecret";

var client = new SecretClient(new Uri(keyVaultUrl), new DefaultAzureCredential());
KeyVaultSecret secret = await client.GetSecretAsync(secretName);
string secretValue = secret.Value;
```

### 6. What is Azure Service Bus?
Azure Service Bus is a messaging service that helps different parts of your application communicate asynchronously. It's like a post office where messages are sent and received between services.

**When to use:** Use it for decoupling applications, handling high-volume messaging, or implementing publish-subscribe patterns.

**Real-time use case:** Sending order confirmation emails after a purchase without slowing down the main checkout process.

**Code snippet:** Sending a message to Azure Service Bus queue:

```csharp
using Azure.Messaging.ServiceBus;

string connectionString = "Endpoint=sb://yournamespace.servicebus.windows.net/;SharedAccessKeyName=yourkeyname;SharedAccessKey=yourkey";
string queueName = "orders";

await using var client = new ServiceBusClient(connectionString);
await using ServiceBusSender sender = client.CreateSender(queueName);

ServiceBusMessage message = new ServiceBusMessage("Order placed: OrderId=123");
await sender.SendMessageAsync(message);
```

### 7. What is Azure DevOps?
Azure DevOps is a set of tools for software development, including version control, CI/CD pipelines, and project management. It's like a complete toolkit for building and deploying software.

**When to use:** Use it for managing the entire software development lifecycle, from coding to deployment.

**Real-time use case:** Setting up automated testing and deployment for a .NET web application.

**Code snippet:** A basic Azure DevOps pipeline YAML (not C# code, but configuration):

```yaml
trigger:
- main

pool:
  vmImage: 'windows-latest'

steps:
- task: UseDotNet@2
  inputs:
    packageType: 'sdk'
    version: '6.x'

- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    projects: '**/*.csproj'

- task: DotNetCoreCLI@2
  inputs:
    command: 'test'
    projects: '**/*Tests.csproj'
```

## Advanced Level

### 8. What is Azure Kubernetes Service (AKS)?
Azure Kubernetes Service (AKS) is a managed Kubernetes service for running containerized applications. It's like a powerful container orchestration platform where you can deploy and manage many containers automatically.

**When to use:** Use it for complex applications that need scaling, high availability, and microservices architecture.

**Real-time use case:** Running a microservices-based e-commerce platform with multiple services like user management, inventory, and payment processing.

**Code snippet:** A basic Kubernetes deployment YAML for a .NET app (not C# code):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dotnet-app
  template:
    metadata:
      labels:
        app: dotnet-app
    spec:
      containers:
      - name: dotnet-app
        image: yourregistry/dotnetapp:latest
        ports:
        - containerPort: 80
```

### 9. What is Azure API Management?
Azure API Management is a service for creating, publishing, and managing APIs. It's like a gateway that controls access to your APIs and provides features like rate limiting and analytics.

**When to use:** Use it when you have multiple APIs that need to be exposed to external users or partners, with security and monitoring.

**Real-time use case:** Managing APIs for a mobile app that connects to various backend services.

**Code snippet:** Creating an API in Azure API Management (using REST API, not direct C#):

```csharp
// This would typically be done through the Azure portal or ARM templates
// But here's how you might call an API through APIM in C#:

using System.Net.Http;

HttpClient client = new HttpClient();
string apiUrl = "https://yourapim.azure-api.net/yourapi/endpoint";
client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", "yoursubscriptionkey");

HttpResponseMessage response = await client.GetAsync(apiUrl);
string result = await response.Content.ReadAsStringAsync();
```

### 10. What is Azure Monitor?
Azure Monitor is a service for collecting and analyzing telemetry data from your applications and infrastructure. It's like a dashboard that shows you how your app is performing and alerts you to issues.

**When to use:** Use it to monitor application performance, detect issues, and gain insights into user behavior.

**Real-time use case:** Monitoring a web application for performance issues and setting up alerts for high error rates.

**Code snippet:** Logging to Azure Application Insights:

```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.Extensibility;

TelemetryConfiguration configuration = TelemetryConfiguration.CreateDefault();
configuration.InstrumentationKey = "your-instrumentation-key";
TelemetryClient telemetryClient = new TelemetryClient(configuration);

telemetryClient.TrackEvent("User logged in");
telemetryClient.TrackMetric("Processing time", 100.0);
```

### 11. What is Azure Active Directory?
Azure Active Directory (Azure AD) is a cloud-based identity and access management service. It's like a directory service that manages user accounts and permissions for applications.

**When to use:** Use it for authentication and authorization in applications, especially when integrating with Microsoft services.

**Real-time use case:** Implementing single sign-on for a corporate application used by employees.

**Code snippet:** Authenticating with Azure AD in a .NET app:

```csharp
using Microsoft.Identity.Client;

string clientId = "your-client-id";
string tenantId = "your-tenant-id";
string[] scopes = new string[] { "https://graph.microsoft.com/.default" };

IPublicClientApplication app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

AuthenticationResult result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();
string accessToken = result.AccessToken;
```

### 12. What is Azure Event Hubs?
Azure Event Hubs is a big data streaming platform that can receive and process millions of events per second. It's like a high-speed conveyor belt for data.

**When to use:** Use it for applications that need to handle large volumes of data in real-time, like IoT devices or social media feeds.

**Real-time use case:** Processing sensor data from thousands of IoT devices in a smart city application.

**Code snippet:** Sending events to Azure Event Hubs:

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;

string connectionString = "Endpoint=sb://yournamespace.servicebus.windows.net/;SharedAccessKeyName=yourkeyname;SharedAccessKey=yourkey";
string eventHubName = "sensordata";

await using var producerClient = new EventHubProducerClient(connectionString, eventHubName);

using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();

EventData eventData = new EventData("Sensor reading: 25.5°C");
eventBatch.TryAdd(eventData);

await producerClient.SendAsync(eventBatch);
```

### 13. What is Azure Logic Apps?
Azure Logic Apps is a service for creating workflows that automate business processes. It's like a visual programming tool for connecting different services and actions.

**When to use:** Use it for automating workflows that involve multiple services, like sending emails based on database changes.

**Real-time use case:** Automatically sending a welcome email when a new user registers in your app.

**Code snippet:** Logic Apps are typically created visually, but here's a conceptual representation (not executable code):

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "Send_email": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connection": {
              "name": "@parameters('$connections')['office365']['connectionId']"
            }
          },
          "method": "post",
          "path": "/Mail"
        }
      }
    },
    "triggers": {
      "When_a_new_user_registers": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connection": {
              "name": "@parameters('$connections')['sql']['connectionId']"
            }
          }
        }
      }
    }
  }
}
```

### 14. What is Azure Redis Cache?
Azure Redis Cache is a managed Redis service for caching data in memory. It's like a super-fast temporary storage that helps speed up your applications.

**When to use:** Use it to cache frequently accessed data to improve application performance and reduce database load.

**Real-time use case:** Caching user session data in a web application to avoid repeated database queries.

**Code snippet:** Using Azure Redis Cache in .NET:

```csharp
using StackExchange.Redis;

string connectionString = "your-redis-cache-connection-string";
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect(connectionString);
IDatabase db = redis.GetDatabase();

await db.StringSetAsync("user:123:name", "John Doe");
string name = await db.StringGetAsync("user:123:name");
```

### 15. What is Azure Front Door?
Azure Front Door is a global CDN and load balancer service. It's like a smart router that directs users to the best server location for fast access.

**When to use:** Use it for applications with global users to improve performance and provide high availability.

**Real-time use case:** Distributing traffic for a worldwide e-commerce site to reduce latency for users in different regions.

**Code snippet:** Front Door is configured through the portal, but here's how you might access it programmatically (not typical):

```csharp
// Front Door is usually configured via ARM templates or portal
// But for custom routing, you might use it indirectly in your app
// This is more of a configuration example:

// In ARM template:
{
  "type": "Microsoft.Network/frontDoors",
  "apiVersion": "2020-05-01",
  "name": "myFrontDoor",
  "location": "Global",
  "properties": {
    "backendPools": [
      {
        "name": "backendPool",
        "properties": {
          "backends": [
            {
              "address": "myapp.azurewebsites.net",
              "httpPort": 80,
              "httpsPort": 443
            }
          ]
        }
      }
    ]
  }
}
```

### 16. What is Azure Cosmos DB?
Azure Cosmos DB is a globally distributed, multi-model database service. It's like a super-flexible database that can handle different types of data and scale automatically across the world.

**When to use:** Use it for applications that need global distribution, low latency, and support for multiple data models (SQL, MongoDB, Cassandra, etc.).

**Real-time use case:** Storing user profiles and preferences for a global social media app that needs to serve users quickly from anywhere.

**Code snippet:** Connecting to Azure Cosmos DB with SQL API:

```csharp
using Microsoft.Azure.Cosmos;

string endpointUri = "https://yourcosmosdb.documents.azure.com:443/";
string primaryKey = "your-primary-key";
CosmosClient cosmosClient = new CosmosClient(endpointUri, primaryKey);

Database database = await cosmosClient.CreateDatabaseIfNotExistsAsync("UserDatabase");
Container container = await database.CreateContainerIfNotExistsAsync("UserProfiles", "/partitionKey");

UserProfile user = new UserProfile { Id = "123", Name = "John Doe", Email = "john@example.com" };
await container.CreateItemAsync(user, new PartitionKey(user.Id));
```

### 17. What is Azure Cognitive Services?
Azure Cognitive Services is a collection of AI services that add intelligence to your applications. It's like having pre-built AI capabilities for vision, speech, language, and decision-making.

**When to use:** Use it to add AI features like image recognition, text analysis, or speech-to-text without building your own AI models.

**Real-time use case:** Adding automatic image tagging to a photo-sharing app.

**Code snippet:** Using Azure Computer Vision for image analysis:

```csharp
using Azure.AI.Vision.ImageAnalysis;

string endpoint = "https://yourvision.cognitiveservices.azure.com/";
string key = "your-key";
ImageAnalysisClient client = new ImageAnalysisClient(new Uri(endpoint), new AzureKeyCredential(key));

ImageAnalysisResult result = await client.AnalyzeAsync(
    new Uri("https://example.com/image.jpg"),
    VisualFeatures.Caption | VisualFeatures.Tags);

Console.WriteLine($"Caption: {result.Caption.Text}");
foreach (var tag in result.Tags.Values)
{
    Console.WriteLine($"Tag: {tag.Name} (Confidence: {tag.Confidence})");
}
```

### 18. What is Azure SignalR Service?
Azure SignalR Service is a fully managed service for adding real-time web functionality to applications. It's like a hub that enables instant communication between server and clients.

**When to use:** Use it for applications that need real-time features like chat, live updates, or collaborative editing.

**Real-time use case:** Building a real-time chat feature for a customer support system.

**Code snippet:** Using Azure SignalR Service in ASP.NET Core:

```csharp
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
}

// In Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR().AddAzureSignalR("your-signalr-connection-string");
}

public void Configure(IApplicationBuilder app)
{
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapHub<ChatHub>("/chatHub");
    });
}
```

### 19. What is Azure Notification Hubs?
Azure Notification Hubs is a scalable push notification service for sending notifications to mobile devices and web browsers. It's like a central system for broadcasting messages to millions of users.

**When to use:** Use it to send push notifications to iOS, Android, Windows, and web platforms from any backend.

**Real-time use case:** Sending promotional notifications to users of a mobile shopping app.

**Code snippet:** Sending a push notification using Azure Notification Hubs:

```csharp
using Microsoft.Azure.NotificationHubs;

string connectionString = "Endpoint=sb://yournamespace.servicebus.windows.net/;SharedAccessKeyName=DefaultFullSharedAccessSignature;SharedAccessKey=yourkey";
string hubName = "yourhubname";

NotificationHubClient hub = NotificationHubClient.CreateClientFromConnectionString(connectionString, hubName);

string message = "New sale! 50% off all items!";
var notification = new AppleNotification(message);
await hub.SendNotificationAsync(notification);
```

### 20. What is Azure Container Instances?
Azure Container Instances (ACI) is a service for running containers on-demand without managing servers. It's like running Docker containers in the cloud instantly.

**When to use:** Use it for quick container deployments, testing, or running simple containerized applications without orchestration.

**Real-time use case:** Running a containerized background job for processing uploaded files.

**Code snippet:** Creating a container instance programmatically (using Azure SDK):

```csharp
using Azure.ResourceManager.ContainerInstance;
using Azure.ResourceManager.ContainerInstance.Models;

string subscriptionId = "your-subscription-id";
string resourceGroupName = "your-resource-group";
string containerGroupName = "my-container-group";

ContainerGroupData containerGroupData = new ContainerGroupData(AzureLocation.WestUS2, new ContainerInstanceContainer[]
{
    new ContainerInstanceContainer("my-container", "nginx:latest", new ContainerResourceRequirements(new ContainerResourceRequestsContent(1.0, 1.5)))
    {
        Ports = { new ContainerPort(80) }
    }
});

ContainerGroupResource containerGroup = await resourceGroup.GetContainerGroups().CreateOrUpdateAsync(WaitUntil.Completed, containerGroupName, containerGroupData);
```

### 21. What is Azure Virtual Machines?
Azure Virtual Machines is a service that provides on-demand, scalable computing resources. It's like renting virtual computers in the cloud where you can install and run your own software.

**When to use:** Use it when you need full control over the operating system and environment, or for running legacy applications.

**Real-time use case:** Setting up a development environment with specific tools and configurations for a team.

**Code snippet:** Creating a virtual machine using Azure SDK (simplified):

```csharp
using Azure.ResourceManager.Compute;
using Azure.ResourceManager.Compute.Models;

VirtualMachineData vmData = new VirtualMachineData(AzureLocation.WestUS2)
{
    HardwareProfile = new VirtualMachineHardwareProfile()
    {
        VmSize = VirtualMachineSizeTypes.StandardB1S
    },
    OsProfile = new VirtualMachineOSProfile()
    {
        ComputerName = "myvm",
        AdminUsername = "azureuser",
        AdminPassword = "P@ssw0rd123!"
    },
    StorageProfile = new VirtualMachineStorageProfile()
    {
        ImageReference = new ImageReference()
        {
            Publisher = "MicrosoftWindowsServer",
            Offer = "WindowsServer",
            Sku = "2019-Datacenter",
            Version = "latest"
        }
    },
    NetworkProfile = new VirtualMachineNetworkProfile()
    {
        NetworkInterfaces = { new VirtualMachineNetworkInterfaceReference() { Id = networkInterfaceId } }
    }
};

VirtualMachineResource vm = await resourceGroup.GetVirtualMachines().CreateOrUpdateAsync(WaitUntil.Completed, "myVM", vmData);
```

### 22. What is Azure Storage Queues?
Azure Storage Queues is a service for storing large numbers of messages that can be accessed from anywhere. It's like a simple message queue for decoupling application components.

**When to use:** Use it for reliable messaging between application components, especially for background processing.

**Real-time use case:** Queuing image processing tasks for a photo editing app.

**Code snippet:** Using Azure Storage Queues:

```csharp
using Azure.Storage.Queues;

string connectionString = "DefaultEndpointsProtocol=https;AccountName=youraccount;AccountKey=yourkey;EndpointSuffix=core.windows.net";
string queueName = "imageprocessing";

QueueClient queueClient = new QueueClient(connectionString, queueName);
await queueClient.CreateIfNotExistsAsync();

await queueClient.SendMessageAsync("Process image: image123.jpg");

QueueMessage[] messages = await queueClient.ReceiveMessagesAsync(maxMessages: 1);
if (messages.Length > 0)
{
    Console.WriteLine($"Processing: {messages[0].Body}");
    await queueClient.DeleteMessageAsync(messages[0].MessageId, messages[0].PopReceipt);
}
```

### 23. What is Azure Table Storage?
Azure Table Storage is a service for storing structured NoSQL data in the cloud. It's like a simple database table that can store massive amounts of data.

**When to use:** Use it for storing large amounts of structured data that doesn't require complex queries.

**Real-time use case:** Storing telemetry data from IoT devices.

**Code snippet:** Using Azure Table Storage:

```csharp
using Azure.Data.Tables;

string connectionString = "DefaultEndpointsProtocol=https;AccountName=youraccount;AccountKey=yourkey;EndpointSuffix=core.windows.net";
string tableName = "TelemetryData";

TableClient tableClient = new TableClient(connectionString, tableName);
await tableClient.CreateIfNotExistsAsync();

TelemetryEntity entity = new TelemetryEntity("Device001", "2023-10-01T10:00:00Z")
{
    Temperature = 25.5,
    Humidity = 60.0
};

await tableClient.AddEntityAsync(entity);

public class TelemetryEntity : ITableEntity
{
    public string PartitionKey { get; set; }
    public string RowKey { get; set; }
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }
    public double Temperature { get; set; }
    public double Humidity { get; set; }

    public TelemetryEntity(string partitionKey, string rowKey)
    {
        PartitionKey = partitionKey;
        RowKey = rowKey;
    }
}
```

### 24. What is Azure Cognitive Search?
Azure Cognitive Search is a cloud search service with built-in AI capabilities. It's like a powerful search engine that can understand and index your content.

**When to use:** Use it to add search functionality to your applications with features like fuzzy matching, filters, and AI-enriched indexing.

**Real-time use case:** Adding search to an e-commerce website to help customers find products.

**Code snippet:** Searching with Azure Cognitive Search:

```csharp
using Azure.Search.Documents;
using Azure.Search.Documents.Models;

string serviceName = "yoursearchservice";
string indexName = "products";
string apiKey = "your-api-key";

Uri serviceEndpoint = new Uri($"https://{serviceName}.search.windows.net/");
AzureKeyCredential credential = new AzureKeyCredential(apiKey);
SearchClient searchClient = new SearchClient(serviceEndpoint, indexName, credential);

SearchResults<Product> response = await searchClient.SearchAsync<Product>("laptop");
await foreach (SearchResult<Product> result in response.GetResultsAsync())
{
    Console.WriteLine($"Name: {result.Document.Name}, Price: {result.Document.Price}");
}

public class Product
{
    public string Id { get; set; }
    public string Name { get; set; }
    public double Price { get; set; }
}
```

### 25. What is Azure Backup?
Azure Backup is a service for protecting your data by backing it up to the cloud. It's like an insurance policy for your important files and applications.

**When to use:** Use it to create backups of virtual machines, SQL databases, and file shares for disaster recovery.

**Real-time use case:** Backing up a development database to prevent data loss during testing.

**Code snippet:** Azure Backup is typically configured through the portal or PowerShell, but here's a conceptual example using Azure SDK for backup vaults:

```csharp
using Azure.ResourceManager.RecoveryServices;
using Azure.ResourceManager.RecoveryServices.Models;

// Create a backup vault
RecoveryServicesVaultData vaultData = new RecoveryServicesVaultData(AzureLocation.WestUS2)
{
    Sku = new RecoveryServicesSku(RecoveryServicesSkuName.Standard)
};

RecoveryServicesVaultResource vault = await resourceGroup.GetRecoveryServicesVaults().CreateOrUpdateAsync(WaitUntil.Completed, "myBackupVault", vaultData);

// Configure backup for a VM (simplified - actual implementation is more complex)
BackupFabricResource fabric = await vault.GetBackupFabrics().GetAsync("Azure");
BackupProtectionIntentResource intent = await fabric.GetBackupProtectionIntents().CreateOrUpdateAsync(WaitUntil.Completed, "vm-backup-intent", new BackupProtectionIntentData()
{
    // Configuration details would go here
});
```

This Q&A covers the most commonly used Azure services for .NET development. Remember to always check the official Azure documentation for the latest features and best practices.