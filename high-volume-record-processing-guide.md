# Handling Millions of Records - High Volume API Processing Guide

## Overview

When processing millions of records through a single API endpoint, you need strategies for batching, parallel processing, rate limiting, fault tolerance, and monitoring. This guide covers proven patterns for .NET Core applications.

---

## Architecture Overview

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Data Source    │────▶│   Batch Manager  │────▶│  Parallel        │────▶│   Target API     │
│   (Millions of   │     │   (Chunking)     │     │  Processor       │     │   Endpoint       │
│    Records)      │     │                  │     │  (Rate Limited)  │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘
                                                          │
                                                          ▼
                                                  ┌──────────────────┐
                                                  │  Retry Queue     │
                                                  │  (Failed Items)  │
                                                  └──────────────────┘
```

---

## 1. Install Required NuGet Packages

```bash
dotnet add package Polly
dotnet add package System.Threading.Channels
dotnet add package Microsoft.Extensions.Http.Polly
dotnet add package System.Linq.Async
```

---

## 2. Configuration

### appsettings.json

```json
{
  "BulkProcessing": {
    "TargetApiBaseUrl": "https://api.example.com",
    "TargetEndpoint": "/api/records",
    "BatchSize": 1000,
    "MaxParallelBatches": 10,
    "MaxRequestsPerSecond": 100,
    "MaxRetryAttempts": 3,
    "RetryDelaySeconds": 2,
    "TimeoutSeconds": 30,
    "ChannelCapacity": 10000
  }
}
```

### Configuration Class

```csharp
public class BulkProcessingSettings
{
    public string TargetApiBaseUrl { get; set; } = string.Empty;
    public string TargetEndpoint { get; set; } = string.Empty;
    public int BatchSize { get; set; } = 1000;
    public int MaxParallelBatches { get; set; } = 10;
    public int MaxRequestsPerSecond { get; set; } = 100;
    public int MaxRetryAttempts { get; set; } = 3;
    public int RetryDelaySeconds { get; set; } = 2;
    public int TimeoutSeconds { get; set; } = 30;
    public int ChannelCapacity { get; set; } = 10000;
}
```

---

## 3. Rate Limiter Implementation

```csharp
using System.Threading.RateLimiting;

public class ApiRateLimiter : IAsyncDisposable
{
    private readonly RateLimiter _rateLimiter;

    public ApiRateLimiter(int requestsPerSecond)
    {
        _rateLimiter = new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
        {
            TokenLimit = requestsPerSecond,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            QueueLimit = requestsPerSecond * 10,
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            TokensPerPeriod = requestsPerSecond,
            AutoReplenishment = true
        });
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action, CancellationToken cancellationToken = default)
    {
        using var lease = await _rateLimiter.AcquireAsync(1, cancellationToken);
        
        if (!lease.IsAcquired)
        {
            throw new RateLimitExceededException("Rate limit exceeded");
        }

        return await action();
    }

    public async ValueTask DisposeAsync()
    {
        _rateLimiter.Dispose();
    }
}

public class RateLimitExceededException : Exception
{
    public RateLimitExceededException(string message) : base(message) { }
}
```

---

## 4. Batch Processor with Channels (Producer-Consumer Pattern)

```csharp
using System.Threading.Channels;

public class BatchProcessor<T>
{
    private readonly Channel<BatchItem<T>> _channel;
    private readonly BulkProcessingSettings _settings;
    private readonly ILogger<BatchProcessor<T>> _logger;

    public BatchProcessor(
        IOptions<BulkProcessingSettings> settings,
        ILogger<BatchProcessor<T>> logger)
    {
        _settings = settings.Value;
        _logger = logger;
        
        _channel = Channel.CreateBounded<BatchItem<T>>(new BoundedChannelOptions(_settings.ChannelCapacity)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false
        });
    }

    public ChannelWriter<BatchItem<T>> Writer => _channel.Writer;
    public ChannelReader<BatchItem<T>> Reader => _channel.Reader;
}

public class BatchItem<T>
{
    public int BatchId { get; set; }
    public List<T> Items { get; set; } = new();
    public int RetryCount { get; set; } = 0;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

---

## 5. High Volume Record Processor Service

```csharp
using System.Collections.Concurrent;
using System.Diagnostics;

public interface IBulkRecordProcessor
{
    Task<ProcessingResult> ProcessRecordsAsync<T>(
        IEnumerable<T> records, 
        CancellationToken cancellationToken = default);
    
    Task<ProcessingResult> ProcessRecordsFromStreamAsync<T>(
        IAsyncEnumerable<T> records, 
        CancellationToken cancellationToken = default);
}

public class BulkRecordProcessor : IBulkRecordProcessor
{
    private readonly HttpClient _httpClient;
    private readonly BulkProcessingSettings _settings;
    private readonly ILogger<BulkRecordProcessor> _logger;
    private readonly ApiRateLimiter _rateLimiter;
    private readonly SemaphoreSlim _semaphore;

    public BulkRecordProcessor(
        IHttpClientFactory httpClientFactory,
        IOptions<BulkProcessingSettings> settings,
        ILogger<BulkRecordProcessor> logger)
    {
        _httpClient = httpClientFactory.CreateClient("BulkApiClient");
        _settings = settings.Value;
        _logger = logger;
        _rateLimiter = new ApiRateLimiter(_settings.MaxRequestsPerSecond);
        _semaphore = new SemaphoreSlim(_settings.MaxParallelBatches);
    }

    public async Task<ProcessingResult> ProcessRecordsAsync<T>(
        IEnumerable<T> records, 
        CancellationToken cancellationToken = default)
    {
        var stopwatch = Stopwatch.StartNew();
        var result = new ProcessingResult();
        var failedBatches = new ConcurrentQueue<BatchItem<T>>();

        // Split into batches
        var batches = records
            .Select((item, index) => new { item, index })
            .GroupBy(x => x.index / _settings.BatchSize)
            .Select((group, batchId) => new BatchItem<T>
            {
                BatchId = batchId,
                Items = group.Select(x => x.item).ToList()
            })
            .ToList();

        result.TotalBatches = batches.Count;
        result.TotalRecords = records.Count();

        _logger.LogInformation(
            "Starting processing of {TotalRecords} records in {TotalBatches} batches",
            result.TotalRecords, result.TotalBatches);

        // Process batches in parallel with rate limiting
        var tasks = batches.Select(async batch =>
        {
            await _semaphore.WaitAsync(cancellationToken);
            try
            {
                var success = await ProcessBatchWithRetryAsync(batch, cancellationToken);
                if (success)
                {
                    Interlocked.Add(ref result.SuccessfulRecords, batch.Items.Count);
                    Interlocked.Increment(ref result.SuccessfulBatches);
                }
                else
                {
                    failedBatches.Enqueue(batch);
                    Interlocked.Add(ref result.FailedRecords, batch.Items.Count);
                    Interlocked.Increment(ref result.FailedBatches);
                }
            }
            finally
            {
                _semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);

        stopwatch.Stop();
        result.ProcessingTime = stopwatch.Elapsed;
        result.RecordsPerSecond = result.TotalRecords / stopwatch.Elapsed.TotalSeconds;
        result.FailedItems = failedBatches.SelectMany(b => b.Items.Cast<object>()).ToList();

        _logger.LogInformation(
            "Completed processing: {SuccessfulRecords}/{TotalRecords} records successful in {Time:F2}s ({RPS:F0} records/sec)",
            result.SuccessfulRecords, result.TotalRecords, 
            result.ProcessingTime.TotalSeconds, result.RecordsPerSecond);

        return result;
    }

    public async Task<ProcessingResult> ProcessRecordsFromStreamAsync<T>(
        IAsyncEnumerable<T> records, 
        CancellationToken cancellationToken = default)
    {
        var stopwatch = Stopwatch.StartNew();
        var result = new ProcessingResult();
        var buffer = new List<T>(_settings.BatchSize);
        var batchId = 0;
        var tasks = new List<Task<bool>>();

        await foreach (var record in records.WithCancellation(cancellationToken))
        {
            buffer.Add(record);
            result.TotalRecords++;

            if (buffer.Count >= _settings.BatchSize)
            {
                var batch = new BatchItem<T>
                {
                    BatchId = batchId++,
                    Items = new List<T>(buffer)
                };
                buffer.Clear();
                result.TotalBatches++;

                await _semaphore.WaitAsync(cancellationToken);
                var task = ProcessBatchWithRetryAsync(batch, cancellationToken)
                    .ContinueWith(t =>
                    {
                        _semaphore.Release();
                        if (t.Result)
                        {
                            Interlocked.Add(ref result.SuccessfulRecords, batch.Items.Count);
                            Interlocked.Increment(ref result.SuccessfulBatches);
                        }
                        else
                        {
                            Interlocked.Add(ref result.FailedRecords, batch.Items.Count);
                            Interlocked.Increment(ref result.FailedBatches);
                        }
                        return t.Result;
                    }, cancellationToken);

                tasks.Add(task);
            }

            // Report progress every 100,000 records
            if (result.TotalRecords % 100000 == 0)
            {
                _logger.LogInformation("Progress: {Count} records queued", result.TotalRecords);
            }
        }

        // Process remaining records
        if (buffer.Count > 0)
        {
            var finalBatch = new BatchItem<T>
            {
                BatchId = batchId,
                Items = buffer
            };
            result.TotalBatches++;
            
            await _semaphore.WaitAsync(cancellationToken);
            var finalTask = ProcessBatchWithRetryAsync(finalBatch, cancellationToken)
                .ContinueWith(t =>
                {
                    _semaphore.Release();
                    if (t.Result)
                    {
                        Interlocked.Add(ref result.SuccessfulRecords, finalBatch.Items.Count);
                        Interlocked.Increment(ref result.SuccessfulBatches);
                    }
                    else
                    {
                        Interlocked.Add(ref result.FailedRecords, finalBatch.Items.Count);
                        Interlocked.Increment(ref result.FailedBatches);
                    }
                    return t.Result;
                }, cancellationToken);
            tasks.Add(finalTask);
        }

        await Task.WhenAll(tasks);

        stopwatch.Stop();
        result.ProcessingTime = stopwatch.Elapsed;
        result.RecordsPerSecond = result.TotalRecords / stopwatch.Elapsed.TotalSeconds;

        return result;
    }

    private async Task<bool> ProcessBatchWithRetryAsync<T>(
        BatchItem<T> batch, 
        CancellationToken cancellationToken)
    {
        var retryCount = 0;
        
        while (retryCount <= _settings.MaxRetryAttempts)
        {
            try
            {
                await _rateLimiter.ExecuteAsync(async () =>
                {
                    var content = new StringContent(
                        JsonSerializer.Serialize(batch.Items),
                        Encoding.UTF8,
                        "application/json");

                    var response = await _httpClient.PostAsync(
                        _settings.TargetEndpoint, 
                        content, 
                        cancellationToken);

                    response.EnsureSuccessStatusCode();
                    return response;
                }, cancellationToken);

                _logger.LogDebug("Batch {BatchId} processed successfully", batch.BatchId);
                return true;
            }
            catch (HttpRequestException ex) when (IsTransientError(ex))
            {
                retryCount++;
                if (retryCount <= _settings.MaxRetryAttempts)
                {
                    var delay = TimeSpan.FromSeconds(
                        _settings.RetryDelaySeconds * Math.Pow(2, retryCount - 1));
                    
                    _logger.LogWarning(
                        "Batch {BatchId} failed (attempt {Attempt}), retrying in {Delay}s: {Error}",
                        batch.BatchId, retryCount, delay.TotalSeconds, ex.Message);
                    
                    await Task.Delay(delay, cancellationToken);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Batch {BatchId} failed with non-transient error", batch.BatchId);
                return false;
            }
        }

        _logger.LogError("Batch {BatchId} failed after {MaxRetries} attempts", 
            batch.BatchId, _settings.MaxRetryAttempts);
        return false;
    }

    private static bool IsTransientError(HttpRequestException ex)
    {
        if (ex.StatusCode == null) return true; // Network error
        
        return ex.StatusCode switch
        {
            System.Net.HttpStatusCode.RequestTimeout => true,
            System.Net.HttpStatusCode.TooManyRequests => true,
            System.Net.HttpStatusCode.InternalServerError => true,
            System.Net.HttpStatusCode.BadGateway => true,
            System.Net.HttpStatusCode.ServiceUnavailable => true,
            System.Net.HttpStatusCode.GatewayTimeout => true,
            _ => false
        };
    }
}
```

---

## 6. Processing Result Model

```csharp
public class ProcessingResult
{
    public int TotalRecords { get; set; }
    public int SuccessfulRecords { get; set; }
    public int FailedRecords { get; set; }
    public int TotalBatches { get; set; }
    public int SuccessfulBatches { get; set; }
    public int FailedBatches { get; set; }
    public TimeSpan ProcessingTime { get; set; }
    public double RecordsPerSecond { get; set; }
    public List<object> FailedItems { get; set; } = new();

    public string GetSummary() => $"""
        Processing Summary:
        - Total Records: {TotalRecords:N0}
        - Successful: {SuccessfulRecords:N0} ({(double)SuccessfulRecords/TotalRecords:P2})
        - Failed: {FailedRecords:N0}
        - Processing Time: {ProcessingTime:hh\:mm\:ss\.fff}
        - Throughput: {RecordsPerSecond:N0} records/second
        """;
}
```

---

## 7. Register Services in Program.cs

```csharp
using Polly;
using Polly.Extensions.Http;

var builder = WebApplication.CreateBuilder(args);

// Configuration
builder.Services.Configure<BulkProcessingSettings>(
    builder.Configuration.GetSection("BulkProcessing"));

// HttpClient with Polly retry policy
builder.Services.AddHttpClient("BulkApiClient", (sp, client) =>
{
    var settings = sp.GetRequiredService<IOptions<BulkProcessingSettings>>().Value;
    client.BaseAddress = new Uri(settings.TargetApiBaseUrl);
    client.Timeout = TimeSpan.FromSeconds(settings.TimeoutSeconds);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Services
builder.Services.AddSingleton<IBulkRecordProcessor, BulkRecordProcessor>();

var app = builder.Build();

// Polly Policies
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
}
```

---

## 8. API Controller for Bulk Processing

```csharp
[ApiController]
[Route("api/[controller]")]
public class BulkProcessingController : ControllerBase
{
    private readonly IBulkRecordProcessor _processor;
    private readonly ILogger<BulkProcessingController> _logger;

    public BulkProcessingController(
        IBulkRecordProcessor processor,
        ILogger<BulkProcessingController> logger)
    {
        _processor = processor;
        _logger = logger;
    }

    /// <summary>
    /// Process records from request body (for smaller datasets up to 100K)
    /// </summary>
    [HttpPost("process")]
    public async Task<IActionResult> ProcessRecords(
        [FromBody] List<RecordDto> records,
        CancellationToken cancellationToken)
    {
        if (records == null || !records.Any())
            return BadRequest("No records provided");

        var result = await _processor.ProcessRecordsAsync(records, cancellationToken);
        return Ok(result);
    }

    /// <summary>
    /// Process records from database (for millions of records)
    /// </summary>
    [HttpPost("process-from-database")]
    public async Task<IActionResult> ProcessFromDatabase(
        [FromQuery] string tableName,
        CancellationToken cancellationToken)
    {
        // Stream records from database
        var records = GetRecordsFromDatabaseAsync(tableName, cancellationToken);
        var result = await _processor.ProcessRecordsFromStreamAsync(records, cancellationToken);
        return Ok(result);
    }

    /// <summary>
    /// Process records from file upload
    /// </summary>
    [HttpPost("process-from-file")]
    [RequestSizeLimit(500_000_000)] // 500MB limit
    public async Task<IActionResult> ProcessFromFile(
        IFormFile file,
        CancellationToken cancellationToken)
    {
        if (file == null || file.Length == 0)
            return BadRequest("No file provided");

        var records = ReadRecordsFromFileAsync(file, cancellationToken);
        var result = await _processor.ProcessRecordsFromStreamAsync(records, cancellationToken);
        return Ok(result);
    }

    private async IAsyncEnumerable<RecordDto> GetRecordsFromDatabaseAsync(
        string tableName,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        // Example: Stream from database using Dapper or EF Core
        // This prevents loading all records into memory
        
        using var connection = new SqlConnection("your-connection-string");
        await connection.OpenAsync(cancellationToken);

        var command = new SqlCommand($"SELECT * FROM {tableName}", connection);
        using var reader = await command.ExecuteReaderAsync(cancellationToken);

        while (await reader.ReadAsync(cancellationToken))
        {
            yield return new RecordDto
            {
                Id = reader.GetInt64(0),
                Data = reader.GetString(1)
                // Map other fields
            };
        }
    }

    private async IAsyncEnumerable<RecordDto> ReadRecordsFromFileAsync(
        IFormFile file,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        using var stream = file.OpenReadStream();
        using var reader = new StreamReader(stream);
        
        // Skip header
        await reader.ReadLineAsync(cancellationToken);

        string? line;
        while ((line = await reader.ReadLineAsync(cancellationToken)) != null)
        {
            var parts = line.Split(',');
            yield return new RecordDto
            {
                Id = long.Parse(parts[0]),
                Data = parts[1]
            };
        }
    }
}

public class RecordDto
{
    public long Id { get; set; }
    public string Data { get; set; } = string.Empty;
}
```

---

## 9. Background Job Processing (For Very Large Datasets)

### Using Background Service with Queue

```csharp
public class BulkProcessingJob
{
    public string JobId { get; set; } = Guid.NewGuid().ToString();
    public string SourceType { get; set; } = string.Empty; // "database", "file", "blob"
    public string SourcePath { get; set; } = string.Empty;
    public JobStatus Status { get; set; } = JobStatus.Pending;
    public ProcessingResult? Result { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? CompletedAt { get; set; }
}

public enum JobStatus
{
    Pending,
    Processing,
    Completed,
    Failed
}

public class BulkProcessingBackgroundService : BackgroundService
{
    private readonly Channel<BulkProcessingJob> _jobChannel;
    private readonly IBulkRecordProcessor _processor;
    private readonly IServiceProvider _serviceProvider;
    private readonly ConcurrentDictionary<string, BulkProcessingJob> _jobs;
    private readonly ILogger<BulkProcessingBackgroundService> _logger;

    public BulkProcessingBackgroundService(
        IBulkRecordProcessor processor,
        IServiceProvider serviceProvider,
        ILogger<BulkProcessingBackgroundService> logger)
    {
        _processor = processor;
        _serviceProvider = serviceProvider;
        _logger = logger;
        _jobs = new ConcurrentDictionary<string, BulkProcessingJob>();
        _jobChannel = Channel.CreateUnbounded<BulkProcessingJob>();
    }

    public string QueueJob(BulkProcessingJob job)
    {
        _jobs[job.JobId] = job;
        _jobChannel.Writer.TryWrite(job);
        return job.JobId;
    }

    public BulkProcessingJob? GetJobStatus(string jobId)
    {
        _jobs.TryGetValue(jobId, out var job);
        return job;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _jobChannel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                job.Status = JobStatus.Processing;
                _logger.LogInformation("Processing job {JobId}", job.JobId);

                var records = await GetRecordsForJobAsync(job, stoppingToken);
                job.Result = await _processor.ProcessRecordsFromStreamAsync(records, stoppingToken);
                job.Status = JobStatus.Completed;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Job {JobId} failed", job.JobId);
                job.Status = JobStatus.Failed;
            }
            finally
            {
                job.CompletedAt = DateTime.UtcNow;
            }
        }
    }

    private async IAsyncEnumerable<RecordDto> GetRecordsForJobAsync(
        BulkProcessingJob job,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        // Implementation based on job.SourceType
        yield break; // Placeholder
    }
}
```

### Job Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class JobsController : ControllerBase
{
    private readonly BulkProcessingBackgroundService _backgroundService;

    public JobsController(BulkProcessingBackgroundService backgroundService)
    {
        _backgroundService = backgroundService;
    }

    [HttpPost("submit")]
    public IActionResult SubmitJob([FromBody] BulkProcessingJob job)
    {
        var jobId = _backgroundService.QueueJob(job);
        return Accepted(new { JobId = jobId, Status = "Queued" });
    }

    [HttpGet("{jobId}/status")]
    public IActionResult GetStatus(string jobId)
    {
        var job = _backgroundService.GetJobStatus(jobId);
        if (job == null)
            return NotFound();

        return Ok(new
        {
            job.JobId,
            job.Status,
            job.Result,
            job.CreatedAt,
            job.CompletedAt,
            Duration = job.CompletedAt.HasValue 
                ? (job.CompletedAt.Value - job.CreatedAt).TotalSeconds 
                : (DateTime.UtcNow - job.CreatedAt).TotalSeconds
        });
    }
}
```

---

## 10. Memory-Efficient Database Streaming with EF Core

```csharp
public class RecordRepository
{
    private readonly AppDbContext _context;

    public RecordRepository(AppDbContext context)
    {
        _context = context;
    }

    /// <summary>
    /// Stream millions of records without loading all into memory
    /// </summary>
    public IAsyncEnumerable<RecordEntity> GetAllRecordsStreamAsync(
        CancellationToken cancellationToken = default)
    {
        return _context.Records
            .AsNoTracking()  // Important: Prevents tracking overhead
            .AsAsyncEnumerable();
    }

    /// <summary>
    /// Stream records in chunks with cursor-based pagination
    /// </summary>
    public async IAsyncEnumerable<RecordEntity> GetRecordsWithCursorAsync(
        long? lastId = null,
        int pageSize = 10000,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var currentLastId = lastId ?? 0;

        while (true)
        {
            var batch = await _context.Records
                .AsNoTracking()
                .Where(r => r.Id > currentLastId)
                .OrderBy(r => r.Id)
                .Take(pageSize)
                .ToListAsync(cancellationToken);

            if (!batch.Any())
                break;

            foreach (var record in batch)
            {
                yield return record;
            }

            currentLastId = batch.Last().Id;
        }
    }
}
```

---

## 11. Progress Reporting and Monitoring

```csharp
public interface IProgressReporter
{
    Task ReportProgressAsync(ProgressReport report);
}

public class ProgressReport
{
    public string JobId { get; set; } = string.Empty;
    public long ProcessedRecords { get; set; }
    public long TotalRecords { get; set; }
    public long FailedRecords { get; set; }
    public double PercentComplete => TotalRecords > 0 
        ? (double)ProcessedRecords / TotalRecords * 100 
        : 0;
    public TimeSpan ElapsedTime { get; set; }
    public double RecordsPerSecond { get; set; }
    public TimeSpan EstimatedTimeRemaining { get; set; }
}

public class ConsoleProgressReporter : IProgressReporter
{
    private readonly ILogger<ConsoleProgressReporter> _logger;

    public ConsoleProgressReporter(ILogger<ConsoleProgressReporter> logger)
    {
        _logger = logger;
    }

    public Task ReportProgressAsync(ProgressReport report)
    {
        _logger.LogInformation(
            "[{JobId}] Progress: {Processed:N0}/{Total:N0} ({Percent:F1}%) | " +
            "Speed: {Speed:N0}/s | Failed: {Failed:N0} | ETA: {ETA:hh\\:mm\\:ss}",
            report.JobId,
            report.ProcessedRecords,
            report.TotalRecords,
            report.PercentComplete,
            report.RecordsPerSecond,
            report.FailedRecords,
            report.EstimatedTimeRemaining);

        return Task.CompletedTask;
    }
}

// SignalR Hub for real-time progress updates
public class ProgressHub : Hub
{
    public async Task JoinJobGroup(string jobId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, jobId);
    }

    public async Task LeaveJobGroup(string jobId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, jobId);
    }
}

public class SignalRProgressReporter : IProgressReporter
{
    private readonly IHubContext<ProgressHub> _hubContext;

    public SignalRProgressReporter(IHubContext<ProgressHub> hubContext)
    {
        _hubContext = hubContext;
    }

    public async Task ReportProgressAsync(ProgressReport report)
    {
        await _hubContext.Clients.Group(report.JobId)
            .SendAsync("ProgressUpdate", report);
    }
}
```

---

## 12. Handling Failures and Dead Letter Queue

```csharp
public class FailedRecordHandler
{
    private readonly ILogger<FailedRecordHandler> _logger;
    private readonly string _failedRecordsPath;

    public FailedRecordHandler(ILogger<FailedRecordHandler> logger)
    {
        _logger = logger;
        _failedRecordsPath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            "BulkProcessing",
            "FailedRecords");
        
        Directory.CreateDirectory(_failedRecordsPath);
    }

    public async Task SaveFailedRecordsAsync<T>(
        string jobId, 
        IEnumerable<T> failedRecords,
        string reason)
    {
        var fileName = $"{jobId}_{DateTime.UtcNow:yyyyMMddHHmmss}_failed.json";
        var filePath = Path.Combine(_failedRecordsPath, fileName);

        var failedData = new
        {
            JobId = jobId,
            Timestamp = DateTime.UtcNow,
            Reason = reason,
            RecordCount = failedRecords.Count(),
            Records = failedRecords
        };

        await File.WriteAllTextAsync(
            filePath, 
            JsonSerializer.Serialize(failedData, new JsonSerializerOptions { WriteIndented = true }));

        _logger.LogWarning(
            "Saved {Count} failed records to {Path}", 
            failedRecords.Count(), 
            filePath);
    }

    public async Task<IEnumerable<T>> LoadFailedRecordsAsync<T>(string filePath)
    {
        var json = await File.ReadAllTextAsync(filePath);
        var data = JsonSerializer.Deserialize<JsonElement>(json);
        var records = data.GetProperty("Records").Deserialize<List<T>>();
        return records ?? Enumerable.Empty<T>();
    }
}
```

---

## 13. Performance Optimization Tips

| Strategy | Description | Impact |
|----------|-------------|--------|
| **Batch Size Tuning** | Start with 1000, adjust based on API response time | High |
| **Parallel Batches** | Use 5-20 concurrent batches based on API limits | High |
| **Connection Pooling** | Reuse HttpClient instances | Medium |
| **Compression** | Enable GZIP for request/response | Medium |
| **Streaming** | Use `IAsyncEnumerable` to avoid loading all data | High |
| **No Tracking** | Use `AsNoTracking()` for read-only queries | Medium |
| **Rate Limiting** | Respect API limits to avoid throttling | High |
| **Circuit Breaker** | Fail fast when API is down | Medium |
| **Cursor Pagination** | Better than offset for large datasets | High |

### Compression Configuration

```csharp
builder.Services.AddHttpClient("BulkApiClient")
    .ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
    {
        AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate
    });
```

---

## 14. Sample Throughput Benchmarks

| Configuration | Records/Second | Notes |
|--------------|----------------|-------|
| Sequential, no batching | ~50 | Baseline |
| Batch size 100, 1 thread | ~500 | 10x improvement |
| Batch size 1000, 5 threads | ~5,000 | 100x improvement |
| Batch size 1000, 10 threads | ~10,000 | With rate limiting |
| Batch size 5000, 20 threads | ~25,000 | Optimal for most APIs |

**Processing 1 Million Records:**
- At 10,000/sec: ~100 seconds (~1.5 minutes)
- At 25,000/sec: ~40 seconds

---

## Summary

1. **Batch records** - Group into chunks of 500-5000
2. **Process in parallel** - Use `SemaphoreSlim` to control concurrency
3. **Rate limit** - Use `TokenBucketRateLimiter` to respect API limits
4. **Stream data** - Use `IAsyncEnumerable` for memory efficiency
5. **Implement retry** - Use Polly for transient failure handling
6. **Use circuit breaker** - Fail fast when API is overwhelmed
7. **Track progress** - Report completion percentage and ETA
8. **Handle failures** - Save failed records for retry
9. **Monitor throughput** - Log records/second metrics
10. **Background jobs** - Use for very large datasets with async status polling
