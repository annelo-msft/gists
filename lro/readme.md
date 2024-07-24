# System.ClientModel Long-Running Operations

## Requirements

### Azure requirements

- Start operation
- Get updates regarding operation progress by polling service
- Set polling interval per update via (1) retry-after header (2) custom interval (3) exponential backoff
- Get current status of operation
- Get outcome of completed operation
- Communicate to user when operation is completed
- Communicate to user when operation state is changed
- Rehydrate an operation from a separate process
- Get HTTP details from service responses

### New third-party service requirements

- Communicate to user when operation is suspended and requires action to continue
- Get updates regarding operation progress via service response stream
- Expose linked operations (e.g. Cancel) on operation type
- Protocol method return type enables adding convenience method overload without breaking changes

## System.ClientModel types

[System.ClientModel APIView](https://spa.apiview.dev/review/1b123e7a51d44ebe945f0212ee039c65?activeApiRevisionId=52c33ec0ef944f2984f69c9fa0f5af5c&diffApiRevisionId=3063cc5747204d499f9e8c212b84c0b3&diffStyle=trees)

## Third-party generated client types

Client libraries add public types derived from SCM `OperationResult`.  These add the following APIs over what `OperationResult` provides on the base type:

- Public properties for `Value`, `Status`, as applicable to the service operation
- Methods to get updates via polling or streaming, as applicable to the service operation
- Implementations of `Wait`, that return when operation is completed or suspended, as applicable to the service operation
- Protocol methods for "linked operations," including methods to resume, cancel the operation
- Convenience methods for "linked operations," including methods to resume, cancel the operation

Clients return these types from both protocol methods and convenience methods that start the operation on the service.  In addition, for convenience methods, clients add methods that accept a rehydration token and return an operation that has already been started, e.g. by a different process.

In OpenAI, an example is:

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

OpenAI also has the option to stream updates indicating the progress of an operation running on the service.  We need a way to support this at the protocol layer, as well as a way to provide access to "linked operations" such as resume and cancel at the convenience layer.  We can address this by having the client add a subtype of the polling `RunOperation` that implements the get updates requirement via streaming,  overrides polling implementations on its base type, and adds streaming convenience APIs.

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

For OpenAI's most complex scenario, submitting tool output when an assistant thread run operation is suspended in "requires_action" state, this can be implemented via either `Wait` or `GetUpdates` for both the polling and streaming cases, as follows.  
The samples look largely the same for both polling and streaming versions.

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

## Client evolution
