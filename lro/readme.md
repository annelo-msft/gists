# System.ClientModel Long-Running Operations

Cloud services use long-running operations to implement asynchronous HTTP operations.  A client sends a request to the service to start an operation, gets updates from the service until the operation completes, and then retrieves any value computed by the operation.

## Requirements

The `Operation` and `Operation<T>` types in Azure.Core provide general-purpose subclient types returned from client service methods.  Third-party services have greater variation in how they are implemented than Azure services, which adds requirements that System.ClientModel types and third-party generated libraries must satisfy.

### Azure requirements

- Start operation
- Get updates regarding operation progress by polling service
- Set polling interval per update via (1) retry-after header (2) custom interval (3) exponential backoff
- Get current status of operation
- Get outcome of completed operation
- Communicate to user when operation is completed
- Rehydrate an operation from a separate process
- Get HTTP details from service responses

### New third-party service requirements

- Get updates regarding operation progress from service response stream
- Communicate to user when operation state is changed
- Communicate to user when operation is suspended and requires action to continue
- Expose linked operations (e.g. Cancel) on operation type
- Protocol method return type enables adding convenience method overload without breaking changes

## System.ClientModel types

To account for variation in operation implementations across third-party cloud services, System.ClientModel provides minimal base types that expose only APIs to indicate whether an operation has completed, to enable rehydration, and a `Wait` method that returns when a polling LRO should stop polling, or an LRO update stream ends.  Client libraries will add public operation-specific types derived from `OperationResult` to add APIs to address additional requirements specific to the operation.

### SCM OperationResult API

```csharp
namespace System.ClientModel.Primitives
{
    public abstract partial class OperationResult : System.ClientModel.ClientResult
    {
        protected OperationResult() { }
        protected OperationResult(System.ClientModel.Primitives.PipelineResponse response) { }
        public abstract bool IsCompleted { get; protected set; }
        public abstract System.ClientModel.ContinuationToken? RehydrationToken { get; protected set; }
        public abstract void Wait(System.Threading.CancellationToken cancellationToken = default(System.Threading.CancellationToken));
        public abstract System.Threading.Tasks.Task WaitAsync(System.Threading.CancellationToken cancellationToken = default(System.Threading.CancellationToken));
    }
}
```

[System.ClientModel APIView](https://spa.apiview.dev/review/1b123e7a51d44ebe945f0212ee039c65?activeApiRevisionId=52c33ec0ef944f2984f69c9fa0f5af5c&diffApiRevisionId=3063cc5747204d499f9e8c212b84c0b3&diffStyle=trees)

## Third-party generated client types

Client libraries add public types derived from SCM `OperationResult`.  These add the following APIs over what `OperationResult` provides on the base type:

- Public properties for `Value`, `Status`, as applicable to the service operation
- Methods to get updates via polling or streaming, as applicable to the service operation
- Implementations of `Wait`, that return when operation is completed or suspended, as applicable to the service operation
- Protocol methods for "linked operations," including methods to resume, cancel the operation
- Convenience methods for "linked operations," including methods to resume, cancel the operation
- Static `Rehydrate` method taking a client or pipeline instance and a rehydration token

Clients return these types from both protocol methods and convenience methods that start the operation on the service.

## Comparison of how Azure.Core and SCM clients address requirements

| Requirement  | Azure.Core | System.ClientModel |
| ------------- | ------------- | --- |
| Start operation  | Service method on client | Service method on client |
| Automatically poll for updates | `Operation.WaitForCompletion` | `OperationResult.Wait` |
| Manually poll for updates | `Operation.UpdateStatus` | Generated type `GetUpdates` |
| Set polling interval | `Operation.WaitForCompletion` parameter | Generated type `Wait` overload parameter |
| Get current status of operation | Parse JSON from `Operation.UpdateStatus(...).GetRawResponse()` | Generated type `Status` property |
| Get outcome of completed operation | `Operation<T>.Value` | Generated type `Value` property |
| Communicate to user when operation is completed | `Operation.WaitForCompletion` | `OperationResult.Wait` |
| Rehydrate an operation from a separate process | `Operation.Rehydrate` static method | Generated type `Rehydrate` static method |
| Get HTTP details from service responses | `Operation.GetRawResponse` | `OperationResult.GetRawResponse` |
| Stream updates | not supported | Generated type `GetUpdates` or `GetUpdatesStreaming` |
| Communicate to user when operation state is changed | not supported | Generated type `GetUpdates` |
| Communicate to user when operation is suspended and requires action to continue | not supported | `OperationResult.Wait` |
| Expose linked operations on operation type | not supported | Methods generated on LRO subclient |
| Add convenience methods without breaking changes | not supported | Protocol and convenience methods return same type |

## Example SCM-based client types

### Polling operation subclients

In OpenAI, an example is the `RunOperation` type, returned from both protocol and convenience `AssistantClient.CreateRun` methods.

```csharp
public class RunOperation : OperationResult
{
    internal RunOperation()
    {
        // Store reference to pipeline, etc
    }

    public ThreadRun? Value { get; protected set; }
    public RunStatus? Status { get; protected set; }

    public override ContinuationToken? RehydrationToken { get; protected set; }
    public override bool IsCompleted { get; protected set; }

    public override void Wait(CancellationToken cancellationToken = default)
    {
        foreach (ThreadRun update in GetUpdates())
        {
            if (update.Status == RunStatus.RequiresAction)
            {
                return;
            }
        }
    }

    public virtual IEnumerable<ThreadRun> GetUpdates(TimeSpan? pollingInterval = null, CancellationToken cancellationToken = default)
    {
        IEnumerator<ClientResult<ThreadRun>> enumerator = new RunUpdateEnumerator();

        while (enumerator.MoveNext())
        {
            ApplyUpdate(enumerator.Current);

            yield return enumerator.Current;

            // Wait polling interval
        }
    }

    private void ApplyUpdate(ClientResult<ThreadRun> result)
    {
        Value = result.Value;
        Status = result.Value.Status;
        SetRawResponse(result.GetRawResponse());
    }

    // Generated convenience methods, including
    public virtual ClientResult<ThreadRun> CancelRun(CancellationToken cancellationToken = default)
    {
        // ...
    }

    // Generated protocol methods, including
    public virtual ClientResult CancelRun(string threadId, string runId, RequestOptions? options)
    {
        // ...
    }
}
```

### Streaming operation subclients

OpenAI also has the option to stream updates indicating the progress of an operation running on the service.  We need a way to support this at the protocol layer, as well as a way to provide access to "linked operations" such as resume and cancel at the convenience layer.  We can address this by having the client add a subtype of the polling `RunOperation` that implements the get updates requirement via streaming, overrides polling implementations on its base type, and adds streaming convenience APIs.

```csharp
public class StreamingRunOperation : RunOperation
{
    // Using IEnumerator instead of IAsyncEnumerator for illustration purposes
    private IEnumerator<StreamingUpdate> _enumerator;

    public virtual IEnumerable<ThreadRun> GetUpdatesStreaming(CancellationToken cancellationToken = default)
    {
        // Implemented via SseParser implementation
        _enumerator = new StreamingUpdateEnumerator();

        try
        {
            while (_enumerator.MoveNext())
            {
                if (_enumerator.Current is RunUpdate update)
                {
                    ApplyUpdate(update);
                }

                yield return _enumerator.Current;
            }
        }
        finally
        {
            if (_enumerator != null)
            {
                _enumerator.Dispose();
                _enumerator = null;
            }
        }
    }

    public override IEnumerable<ThreadRun> GetUpdates(TimeSpan? pollingInterval = null, CancellationToken cancellationToken = default)
    {
        foreach (StreamingUpdate update in GetUpdatesStreaming(cancellationToken))
        {
            if (update is RunUpdate runUpdate)
            {
                yield return runUpdate.Value;
            }
        }
    }

    public override void Wait(CancellationToken cancellationToken = default)
    {
        foreach (StreamingUpdate update in GetUpdatesStreaming(cancellationToken))
        {
            // Terminates naturally if completed or requires action
        }
    }

    // Streaming convenience methods, including
    public virtual void SubmitToolOutputsToRunStreaming(IEnumerable<ToolOutput> toolOutputs, CancellationToken cancellationToken = default)
    {
        // Sends request
        // Updates _enumerator used by GetUpdatesStreaming to parse events from the new response stream
    }
}
```

[OpenAI APIView](https://spa.apiview.dev/review/2413cd41f35b43d7bbc60a5588dc103f?activeApiRevisionId=2c9579ce805d4158ad957ec8a2bbecc2&diffApiRevisionId=dd8c0659b2aa467b99585ce1688a7398&diffStyle=trees)

## Usage samples

For OpenAI's most complex scenario, submitting tool output when an assistant thread run operation is suspended in "requires_action" state, this can be implemented via either `Wait` or `GetUpdates` for both the polling and streaming cases, as follows.  The samples look largely the same for both polling and streaming versions.

### Wait, polling

```csharp
// Create run polling
RunOperation runOperation = client.CreateRun(
    ReturnWhen.Started,
    thread, assistant,
    new RunCreationOptions()
    {
        AdditionalInstructions = "Call provided tools when appropriate.",
    });

while (!runOperation.IsCompleted)
{
    runOperation.Wait();

    if (runOperation.Status == RunStatus.RequiresAction)
    {
        IEnumerable<ToolOutput> outputs = new List<ToolOutput> {
            new ToolOutput(runOperation.Value.RequiredActions[0].ToolCallId, "tacos")
        };

        runOperation.SubmitToolOutputsToRun(outputs);
    }
}
```

### GetUpdates, streaming

```csharp
// Create run streaming
StreamingRunOperation runOperation = client.CreateRunStreaming(thread, assistant,
    new RunCreationOptions()
    {
        AdditionalInstructions = "Call provided tools when appropriate.",
    });


IAsyncEnumerable<StreamingUpdate> updates = runOperation.GetUpdatesStreamingAsync();

await foreach (StreamingUpdate update in updates)
{
    if (update is RunUpdate &&
        runOperation.Status == RunStatus.RequiresAction)
    {
        IEnumerable<ToolOutput> outputs = new List<ToolOutput> {
            new ToolOutput(runOperation.Value.RequiredActions[0].ToolCallId, "tacos")
        };

        await runOperation.SubmitToolOutputsToRunStreamingAsync(outputs);
    }
}
```

## Subclient evolution
