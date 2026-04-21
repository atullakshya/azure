# Azure Service Bus Integration Guide for .NET Core

## Overview

Azure Service Bus is a fully managed enterprise message broker that enables reliable cloud messaging between applications and services. It supports queues (point-to-point) and topics/subscriptions (publish-subscribe).

---

## When to Use Azure Service Bus

| Scenario | Why Service Bus |
|----------|----------------|
| **Decoupling microservices** | Producers and consumers operate independently; if a downstream service is offline, messages are retained until it recovers |
| **Load leveling** | Smooths out traffic spikes by buffering messages in the queue so consumers process at their own pace |
| **Reliable async processing** | Guarantees at-least-once delivery with dead-letter support, unlike fire-and-forget HTTP calls |
| **Ordered message processing** | Sessions provide FIFO ordering when message sequence matters (e.g., account transactions) |
| **Publish-subscribe patterns** | Topics and subscriptions let multiple consumers react to the same event (e.g., order placed → inventory, billing, notifications) |
| **Cross-boundary communication** | Securely connect on-premises systems to cloud services, or bridge different teams/organizations |
| **Long-running workflows** | Offload work that takes seconds or minutes (report generation, video encoding) instead of blocking an HTTP response |
| **Competing consumers** | Scale out processing by adding more consumer instances that share the same queue |

### When **Not** to Use Service Bus

- **Simple request/response** – Direct HTTP/gRPC calls are simpler when you need a synchronous reply.
- **Low-latency real-time streaming** – Consider Azure Event Hubs or SignalR for high-throughput telemetry or live UI updates.
- **Lightweight in-process pub/sub** – `MediatR` or in-memory channels may suffice within a single process.
- **Basic task scheduling** – Azure Functions timer triggers or Hangfire are better fits for cron-style jobs.

---

## 1. Prerequisites

- Azure subscription
- Azure Service Bus namespace created in Azure Portal
- .NET Core 6.0 or later
- Visual Studio / VS Code

---

## 2. Create Service Bus Namespace in Azure

1. Go to [Azure Portal](https://portal.azure.com)
2. Click **Create a resource** → Search **Service Bus**
3. Fill in details:
   - **Subscription**: Select your subscription
   - **Resource Group**: Create new or select existing
   - **Namespace name**: Unique name (e.g., `myapp-servicebus`)
   - **Location**: Select region
   - **Pricing tier**: Standard or Premium (Basic doesn't support topics)
4. Click **Review + Create** → **Create**

### Get Connection String

1. Navigate to your Service Bus namespace
2. Go to **Shared access policies** → **RootManageSharedAccessKey**
3. Copy the **Primary Connection String**

---

## 3. Install NuGet Packages

```bash
dotnet add package Azure.Messaging.ServiceBus
dotnet add package Microsoft.Extensions.Azure
```

---

## 4. Configuration

### appsettings.json

```json
{
  "AzureServiceBus": {
    "ConnectionString": "Endpoint=sb://your-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=your-key",
    "QueueName": "orders-queue",
    "TopicName": "notifications-topic",
    "SubscriptionName": "email-subscription"
  }
}
```

### Configuration Class

```csharp
public class ServiceBusSettings
{
    public string ConnectionString { get; set; } = string.Empty;
    public string QueueName { get; set; } = string.Empty;
    public string TopicName { get; set; } = string.Empty;
    public string SubscriptionName { get; set; } = string.Empty;
}
```

---

## 5. Register Service Bus in Program.cs

```csharp
using Azure.Messaging.ServiceBus;

var builder = WebApplication.CreateBuilder(args);

// Bind configuration
builder.Services.Configure<ServiceBusSettings>(
    builder.Configuration.GetSection("AzureServiceBus"));

// Register ServiceBusClient as singleton
builder.Services.AddSingleton(sp =>
{
    var settings = builder.Configuration
        .GetSection("AzureServiceBus")
        .Get<ServiceBusSettings>();
    return new ServiceBusClient(settings!.ConnectionString);
});

// Register sender and processor services
builder.Services.AddSingleton<IMessageSenderService, MessageSenderService>();
builder.Services.AddHostedService<MessageReceiverService>();

var app = builder.Build();
```

---

## 6. Sending Messages (Producer)

### Message Sender Service

```csharp
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.Options;

public interface IMessageSenderService
{
    Task SendMessageAsync<T>(T message, string? queueOrTopic = null);
    Task SendBatchMessagesAsync<T>(IEnumerable<T> messages, string? queueOrTopic = null);
}

public class MessageSenderService : IMessageSenderService, IAsyncDisposable
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSettings _settings;
    private ServiceBusSender? _sender;

    public MessageSenderService(
        ServiceBusClient client,
        IOptions<ServiceBusSettings> settings)
    {
        _client = client;
        _settings = settings.Value;
    }

    public async Task SendMessageAsync<T>(T message, string? queueOrTopic = null)
    {
        var destination = queueOrTopic ?? _settings.QueueName;
        _sender = _client.CreateSender(destination);

        var jsonMessage = JsonSerializer.Serialize(message);
        var serviceBusMessage = new ServiceBusMessage(jsonMessage)
        {
            ContentType = "application/json",
            MessageId = Guid.NewGuid().ToString(),
            Subject = typeof(T).Name
        };

        await _sender.SendMessageAsync(serviceBusMessage);
        Console.WriteLine($"Message sent to {destination}");
    }

    public async Task SendBatchMessagesAsync<T>(IEnumerable<T> messages, string? queueOrTopic = null)
    {
        var destination = queueOrTopic ?? _settings.QueueName;
        _sender = _client.CreateSender(destination);

        using ServiceBusMessageBatch batch = await _sender.CreateMessageBatchAsync();

        foreach (var message in messages)
        {
            var jsonMessage = JsonSerializer.Serialize(message);
            if (!batch.TryAddMessage(new ServiceBusMessage(jsonMessage)))
            {
                throw new Exception("Message too large for batch");
            }
        }

        await _sender.SendMessagesAsync(batch);
        Console.WriteLine($"Batch of {batch.Count} messages sent");
    }

    public async ValueTask DisposeAsync()
    {
        if (_sender != null)
            await _sender.DisposeAsync();
    }
}
```

---

## 7. Receiving Messages (Consumer)

### Message Receiver as Background Service

```csharp
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Options;

public class MessageReceiverService : BackgroundService
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSettings _settings;
    private readonly IServiceProvider _serviceProvider;
    private ServiceBusProcessor? _processor;

    public MessageReceiverService(
        ServiceBusClient client,
        IOptions<ServiceBusSettings> settings,
        IServiceProvider serviceProvider)
    {
        _client = client;
        _settings = settings.Value;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _processor = _client.CreateProcessor(
            _settings.QueueName,
            new ServiceBusProcessorOptions
            {
                AutoCompleteMessages = false,
                MaxConcurrentCalls = 5,
                PrefetchCount = 10
            });

        _processor.ProcessMessageAsync += ProcessMessageHandler;
        _processor.ProcessErrorAsync += ProcessErrorHandler;

        await _processor.StartProcessingAsync(stoppingToken);

        // Keep running until cancellation
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private async Task ProcessMessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        Console.WriteLine($"Received: {body}");

        // Process the message - call APIs, update database, etc.
        using var scope = _serviceProvider.CreateScope();
        var apiService = scope.ServiceProvider.GetRequiredService<IApiCallerService>();
        
        await apiService.ProcessMessageAsync(body);

        // Complete the message (remove from queue)
        await args.CompleteMessageAsync(args.Message);
    }

    private Task ProcessErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine($"Error: {args.Exception.Message}");
        return Task.CompletedTask;
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        if (_processor != null)
        {
            await _processor.StopProcessingAsync(cancellationToken);
            await _processor.DisposeAsync();
        }
        await base.StopAsync(cancellationToken);
    }
}
```

---

## 8. Calling Multiple APIs Through Service Bus

### Message Models

```csharp
public class ApiCallMessage
{
    public string MessageId { get; set; } = Guid.NewGuid().ToString();
    public string ApiEndpoint { get; set; } = string.Empty;
    public string HttpMethod { get; set; } = "GET";
    public object? Payload { get; set; }
    public Dictionary<string, string> Headers { get; set; } = new();
    public string CorrelationId { get; set; } = string.Empty;
}

public class MultiApiRequest
{
    public string CorrelationId { get; set; } = Guid.NewGuid().ToString();
    public List<ApiCallMessage> ApiCalls { get; set; } = new();
}
```

### API Caller Service

```csharp
public interface IApiCallerService
{
    Task ProcessMessageAsync(string messageBody);
    Task<HttpResponseMessage> CallApiAsync(ApiCallMessage apiCall);
}

public class ApiCallerService : IApiCallerService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<ApiCallerService> _logger;

    public ApiCallerService(
        IHttpClientFactory httpClientFactory,
        ILogger<ApiCallerService> logger)
    {
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }

    public async Task ProcessMessageAsync(string messageBody)
    {
        var request = JsonSerializer.Deserialize<MultiApiRequest>(messageBody);
        
        if (request?.ApiCalls == null || !request.ApiCalls.Any())
        {
            _logger.LogWarning("No API calls in message");
            return;
        }

        _logger.LogInformation(
            "Processing {Count} API calls for correlation {CorrelationId}",
            request.ApiCalls.Count,
            request.CorrelationId);

        // Process APIs in parallel
        var tasks = request.ApiCalls.Select(apiCall => CallApiAsync(apiCall));
        var results = await Task.WhenAll(tasks);

        _logger.LogInformation(
            "Completed {Success}/{Total} API calls",
            results.Count(r => r.IsSuccessStatusCode),
            results.Length);
    }

    public async Task<HttpResponseMessage> CallApiAsync(ApiCallMessage apiCall)
    {
        var client = _httpClientFactory.CreateClient();

        try
        {
            // Add headers
            foreach (var header in apiCall.Headers)
            {
                client.DefaultRequestHeaders.TryAddWithoutValidation(header.Key, header.Value);
            }

            HttpResponseMessage response;

            switch (apiCall.HttpMethod.ToUpper())
            {
                case "GET":
                    response = await client.GetAsync(apiCall.ApiEndpoint);
                    break;

                case "POST":
                    var postContent = new StringContent(
                        JsonSerializer.Serialize(apiCall.Payload),
                        Encoding.UTF8,
                        "application/json");
                    response = await client.PostAsync(apiCall.ApiEndpoint, postContent);
                    break;

                case "PUT":
                    var putContent = new StringContent(
                        JsonSerializer.Serialize(apiCall.Payload),
                        Encoding.UTF8,
                        "application/json");
                    response = await client.PutAsync(apiCall.ApiEndpoint, putContent);
                    break;

                case "DELETE":
                    response = await client.DeleteAsync(apiCall.ApiEndpoint);
                    break;

                default:
                    throw new NotSupportedException($"HTTP method {apiCall.HttpMethod} not supported");
            }

            _logger.LogInformation(
                "API {Endpoint} returned {StatusCode}",
                apiCall.ApiEndpoint,
                response.StatusCode);

            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error calling API {Endpoint}", apiCall.ApiEndpoint);
            throw;
        }
    }
}
```

### Register HttpClient in Program.cs

```csharp
builder.Services.AddHttpClient();
builder.Services.AddScoped<IApiCallerService, ApiCallerService>();
```

---

## 9. Example: Sending Multi-API Request

### Controller Example

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMessageSenderService _messageSender;

    public OrdersController(IMessageSenderService messageSender)
    {
        _messageSender = messageSender;
    }

    [HttpPost("process")]
    public async Task<IActionResult> ProcessOrder([FromBody] OrderRequest order)
    {
        // Create multi-API request to be processed via Service Bus
        var multiApiRequest = new MultiApiRequest
        {
            CorrelationId = Guid.NewGuid().ToString(),
            ApiCalls = new List<ApiCallMessage>
            {
                new ApiCallMessage
                {
                    ApiEndpoint = "https://inventory-api.example.com/api/reserve",
                    HttpMethod = "POST",
                    Payload = new { ProductId = order.ProductId, Quantity = order.Quantity }
                },
                new ApiCallMessage
                {
                    ApiEndpoint = "https://payment-api.example.com/api/charge",
                    HttpMethod = "POST",
                    Payload = new { Amount = order.TotalAmount, CustomerId = order.CustomerId }
                },
                new ApiCallMessage
                {
                    ApiEndpoint = "https://shipping-api.example.com/api/schedule",
                    HttpMethod = "POST",
                    Payload = new { OrderId = order.OrderId, Address = order.ShippingAddress }
                },
                new ApiCallMessage
                {
                    ApiEndpoint = "https://notification-api.example.com/api/email",
                    HttpMethod = "POST",
                    Payload = new { To = order.CustomerEmail, Subject = "Order Confirmed" }
                }
            }
        };

        // Send to Service Bus for async processing
        await _messageSender.SendMessageAsync(multiApiRequest);

        return Accepted(new { CorrelationId = multiApiRequest.CorrelationId });
    }
}
```

---

## 10. Using Topics and Subscriptions (Pub/Sub)

### Topic Sender

```csharp
// Send to topic instead of queue
await _messageSender.SendMessageAsync(order, "orders-topic");
```

### Topic Subscription Processor

```csharp
// In MessageReceiverService, use topic + subscription
_processor = _client.CreateProcessor(
    _settings.TopicName,
    _settings.SubscriptionName,
    new ServiceBusProcessorOptions
    {
        AutoCompleteMessages = false,
        MaxConcurrentCalls = 5
    });
```

### Create Multiple Subscriptions

Each subscription can have filters to receive specific messages:

```csharp
// Using Azure.Messaging.ServiceBus.Administration
var adminClient = new ServiceBusAdministrationClient(connectionString);

// Create subscription with SQL filter
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("orders-topic", "high-priority-orders"),
    new CreateRuleOptions("HighPriorityFilter", new SqlRuleFilter("Priority = 'High'")));
```

---

## 11. Dead Letter Queue Handling

```csharp
public class DeadLetterProcessor : BackgroundService
{
    private readonly ServiceBusClient _client;
    private ServiceBusReceiver? _receiver;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Access dead letter queue
        _receiver = _client.CreateReceiver(
            "orders-queue",
            new ServiceBusReceiverOptions
            {
                SubQueue = SubQueue.DeadLetter
            });

        while (!stoppingToken.IsCancellationRequested)
        {
            var messages = await _receiver.ReceiveMessagesAsync(
                maxMessages: 10,
                maxWaitTime: TimeSpan.FromSeconds(5),
                cancellationToken: stoppingToken);

            foreach (var message in messages)
            {
                Console.WriteLine($"Dead letter reason: {message.DeadLetterReason}");
                Console.WriteLine($"Body: {message.Body}");
                
                // Process or log dead letter message
                await _receiver.CompleteMessageAsync(message, stoppingToken);
            }
        }
    }
}
```

---

## 12. Best Practices

| Practice | Description |
|----------|-------------|
| **Use Managed Identity** | Avoid connection strings in production; use `DefaultAzureCredential` |
| **Implement Retry Logic** | Use Polly for resilient API calls |
| **Set Message TTL** | Configure `TimeToLive` to avoid stale messages |
| **Monitor with App Insights** | Track message processing metrics |
| **Use Sessions** | For ordered message processing (FIFO) |
| **Implement Idempotency** | Handle duplicate messages gracefully |
| **Batch Operations** | Use batch sending/receiving for better throughput |

### Using Managed Identity (Recommended for Production)

```csharp
using Azure.Identity;

builder.Services.AddSingleton(sp =>
{
    var fullyQualifiedNamespace = "your-namespace.servicebus.windows.net";
    return new ServiceBusClient(fullyQualifiedNamespace, new DefaultAzureCredential());
});
```

---

## 13. Architecture Diagram

```
┌─────────────────┐     ┌───────────────────────┐     ┌─────────────────┐
│   Web API       │────▶│   Azure Service Bus   │────▶│  Background     │
│   (Producer)    │     │   Queue/Topic         │     │  Service        │
└─────────────────┘     └───────────────────────┘     │  (Consumer)     │
                                                       └────────┬────────┘
                                                                │
                        ┌───────────────────────────────────────┼───────────────────┐
                        │                                       │                   │
                        ▼                                       ▼                   ▼
                ┌───────────────┐                     ┌──────────────┐      ┌──────────────┐
                │  Inventory    │                     │   Payment    │      │  Notification│
                │  API          │                     │   API        │      │  API         │
                └───────────────┘                     └──────────────┘      └──────────────┘
```

---

## Summary

1. **Create** Azure Service Bus namespace and queue/topic
2. **Install** `Azure.Messaging.ServiceBus` NuGet package
3. **Configure** connection string in appsettings.json
4. **Register** `ServiceBusClient` in DI container
5. **Implement** sender service to publish messages
6. **Implement** receiver as `BackgroundService` to process messages
7. **Call multiple APIs** by deserializing messages and using `HttpClient`
8. **Use topics** for pub/sub scenarios with multiple subscribers

This pattern decouples your API calls, provides reliable message delivery, and enables scalable async processing.
