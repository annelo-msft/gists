# System.ClientModel-based client pattern for LROs

`System.ClientModel`-based .NET clients implement long-running operations (LROs) via a base abstraction in the `System.ClientModel` library, public types derived from this abstraction in client libraries, and API patterns for service methods on the client.  

The benefit of having a base abstraction and consistency in client patterns is that users of SCM-based clients can learn the pattern once and use it to work with LROs implemented by any cloud service, without having to understand the specific details of different service APIs.  This can save users time in authoring applications that consume cloud services, and enable engineering efficiencies such as code generation, the ability to build reusable components in layers above the abstraction, and community benefits around readability, extensibilty, and maintainability of open source implementations based on these patterns.

## LRO definition

A long-running operation (LRO) is an operation that a cloud *service* executes asynchonously.  For HTTP-based services, this means that the service will typically respond to a request to start the operation prior to the completion of the operation.

See [Azure API Guidelines | Long-Running Operations & Jobs](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#long-running-operations--jobs) for further details.

From a *client* perspective, a client pattern for LROs must support the steps:

1. The client initiates the operation on the service
1. The client polls the service to track the progress of the operation
1. The client indicates to its user that the operation is complete and any computed results can be obtained

### Messages sent between client and service

![image](https://gist.github.com/user-attachments/assets/fcf2cdb9-2f4d-4be4-ae21-fddc99cac566)

## SCM-based client pattern

The `System.ClientModel`-based client pattern for long-running operations in .NET clients consists of three elements:

1. Service methods on the client used to initiate the LRO
1. A base abstraction [OperationResult](https://learn.microsoft.com/en-us/dotnet/api/system.clientmodel.primitives.operationresult?view=azure-dotnet)
1. A type derived from `OperationResult`, implemented as a public type in the client assembly (the *LRO subclient*)

<details>
<summary><h3><b> Usage samples </b></h3></summary>

<details>
<summary><h4><b> 1. Start LRO, return when completed </b></h4></summary>

```csharp
VectorStore vectorStore = client.CreateVectorStore(waitUntilCompleted: true).Value;
```

</details>

<details>
<summary><h4><b> 2. Start LRO, wait for completion via LRO subclient </b></h4></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
createOperation.WaitForCompletion();
VectorStore vectorStore = createOperation.Value;
```

</details>

<details>
<summary><h4><b> 3. Start LRO, wait for completion using custom polling interval </b></h4></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
createOperation.WaitForCompletion(pollingInterval: TimeSpan.FromSeconds(2));
VectorStore vectorStore = createOperation.Value;
```

</details>

<details>
<summary><h4><b> 4. Start LRO, manually poll for updates (advanced) </b></h4></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
while (!createOperation.HasCompleted)
{
    await Task.Delay(2000);
    createOperation.UpdateStatus();
}
VectorStore vectorStore = createOperation.Value;
```

</details>

<details>
<summary><h4><b> 5. Start LRO, view HTTP response details (advanced) </b></h4></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
PrintHttpDetails(createOperation.GetRawResponse());
while (!createOperation.HasCompleted)
{
    await Task.Delay(2000);
    createOperation.UpdateStatus();
    PrintHttpDetails(createOperation.GetRawResponse());
}
VectorStore vectorStore = createOperation.Value;

void PrintHttpDetails(PipelineResponse response)
{
    Console.WriteLine("Status code: " + response.Status);
}
```

</details>

<details>
<summary><h4><b> 6. Start LRO, wait for completion from a different process ("Rehydrate") (advanced) </b></h4></summary>

From first process:

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
PersistValue(createOperation.RehydrationToken);
```

From second process:

```csharp
ContinuationToken rehydrationToken = ReadPersistedValue(createOperation.RehydrationToken);
CreateVectorStoreOperation createOperation = CreateVectorStoreOperation(client, rehydrationToken);
createOperatino.WaitForCompletion();
VectorStore vectorStore = createOperation.Value;
```

</details>

</details>
<details>
<summary><h3><b> Client APIs </b></h3></summary>

The sections below illustrate an example implementation for the OpenAI LRO to create a vector store.  These samples are based on the implementation in the .NET OpenAI library (see: [CreateVectorStoreOperation.cs](https://github.com/openai/openai-dotnet/blob/main/src/Custom/VectorStores/CreateVectorStoreOperation.cs) and related types for specifics of current implementation details).

<details>
<summary><h4><b> Client service methods </b></h4></summary>

The `VectorStoreClient` provides service methods to start the LRO.  Usage samples to call these APIs are provided in a prior section.

```csharp
public class VectorStoreClient {
    // ...

    // Convenience methods    
    public virtual Task<CreateVectorStoreOperation> CreateVectorStoreAsync(bool waitUntilCompleted, VectorStoreCreationOptions vectorStore = null, CancellationToken cancellationToken = default);
    public virtual CreateVectorStoreOperation CreateVectorStore(bool waitUntilCompleted, VectorStoreCreationOptions vectorStore = null, CancellationToken cancellationToken = default);

    // Protocol methods
    public virtual Task<CreateVectorStoreOperation> CreateVectorStoreAsync(BinaryContent content, bool waitUntilCompleted, RequestOptions options = null);
    public virtual CreateVectorStoreOperation CreateVectorStore(BinaryContent content, bool waitUntilCompleted, RequestOptions options = null);

    // ...
}
```

</details>

<details>
<summary><h4><b> OperationResult APIs </b></h4></summary>

The `OperationResult` base abstraction is provided in the `System.ClientModel` library.  It has an abstract `UpdateStatus` method that derived types must implement.  Its `WaitForCompletion` method calls `UpdateStatus` internally, at a default polling interval.

```csharp
public abstract partial class OperationResult
{
    protected OperationResult(System.ClientModel.Primitives.PipelineResponse response) { }
    public bool HasCompleted { get { throw null; } protected set { } }
    public abstract System.ClientModel.ContinuationToken? RehydrationToken { get; protected set; }
    public System.ClientModel.Primitives.PipelineResponse GetRawResponse() { throw null; }
    protected void SetRawResponse(System.ClientModel.Primitives.PipelineResponse response) { }
    public abstract System.ClientModel.ClientResult UpdateStatus(System.ClientModel.Primitives.RequestOptions? options = null);
    public abstract System.Threading.Tasks.ValueTask<System.ClientModel.ClientResult> UpdateStatusAsync(System.ClientModel.Primitives.RequestOptions? options = null);
    public virtual void WaitForCompletion(System.Threading.CancellationToken cancellationToken = default(System.Threading.CancellationToken)) { }
    public virtual System.Threading.Tasks.ValueTask WaitForCompletionAsync(System.Threading.CancellationToken cancellationToken = default(System.Threading.CancellationToken)) { throw null; }
}
```

</details>

<details>
<summary><h4><b> LRO subclient APIs </b></h4></summary>

The `CreateVectorStoreOperation` class is dervied from SCM `OperationResult`.  It implements the abstract members `UpdateStatus` and `RehydrationToken`.  The implementation of `UpdateStatus` calls through to internal generated service methods to obtain the status of the operation, and sets `HasCompleted` to `true` once the operation has completed.

```csharp
public class CreateVectorStoreOperation : OperationResult {
    public override ContinuationToken? RehydrationToken { get; protected set; }
    public VectorStore? Value { get; }
    public static CreateVectorStoreOperation Rehydrate(VectorStoreClient client, ContinuationToken rehydrationToken, CancellationToken cancellationToken = default);
    public static Task<CreateVectorStoreOperation> RehydrateAsync(VectorStoreClient client, ContinuationToken rehydrationToken, CancellationToken cancellationToken = default);
    public override ClientResult UpdateStatus(RequestOptions? options = null);
    public override ValueTask<ClientResult> UpdateStatusAsync(RequestOptions? options = null);
}
```

</details>

</details>
<details>
<summary><h3><b> Implementation samples </b></h3></summary>

<details>
<summary><h4><b> CreateVectorStoreOperation </b></h4></summary>

The following is an example implementation of `CreateVectorStoreOperation`, whose public APIs were detailed in a prior section.

```csharp
using System;
using System.ClientModel;
using System.ClientModel.Primitives;
using System.Diagnostics.CodeAnalysis;
using System.Threading;
using System.Threading.Tasks;

#nullable enable

namespace OpenAI.VectorStores;

[Experimental("OPENAI001")]
public partial class CreateVectorStoreOperation : OperationResult
{
    private readonly ClientPipeline _pipeline;
    private readonly Uri _endpoint;

    internal CreateVectorStoreOperation(ClientPipeline pipeline, Uri endpoint, ClientResult<VectorStore> result)
        : base(result.GetRawResponse())
    {
        _pipeline = pipeline;
        _endpoint = endpoint;

        Value = result;
        HasCompleted = GetHasCompleted(Value.Status);
        RehydrationToken = new CreateVectorStoreOperationToken(VectorStoreId);
    }
    
    /// <inheritdoc/>
    public override ContinuationToken? RehydrationToken { get; protected set; }

    /// <summary>
    /// The current value of the create <see cref="VectorStore"/> operation in progress.
    /// </summary>
    public VectorStore? Value { get; private set; }

    /// <summary>
    /// Recreates a <see cref="CreateVectorStoreOperation"/> from a rehydration token.
    /// </summary>
    /// <param name="client"> The <see cref="VectorStoreClient"/> used to obtain the 
    /// operation status from the service. </param>
    /// <param name="rehydrationToken"> The rehydration token corresponding to 
    /// the operation to rehydrate. </param>
    /// <param name="cancellationToken"> A token that can be used to cancel the 
    /// request. </param>
    /// <returns> The rehydrated operation. </returns>
    /// <exception cref="ArgumentNullException"> <paramref name="client"/> or <paramref name="rehydrationToken"/> is null. </exception>
    public static async Task<CreateVectorStoreOperation> RehydrateAsync(VectorStoreClient client, ContinuationToken rehydrationToken, CancellationToken cancellationToken = default)
    {
        Argument.AssertNotNull(client, nameof(client));
        Argument.AssertNotNull(rehydrationToken, nameof(rehydrationToken));

        CreateVectorStoreOperationToken token = CreateVectorStoreOperationToken.FromToken(rehydrationToken);

        ClientResult result = await client.GetVectorStoreAsync(token.VectorStoreId, cancellationToken.ToRequestOptions()).ConfigureAwait(false);
        PipelineResponse response = result.GetRawResponse();
        VectorStore vectorStore = VectorStore.FromResponse(response);

        return client.CreateCreateVectorStoreOperation(ClientResult.FromValue(vectorStore, response));
    }

    /// <summary>
    /// Recreates a <see cref="CreateVectorStoreOperation"/> from a rehydration token.
    /// </summary>
    /// <param name="client"> The <see cref="VectorStoreClient"/> used to obtain the 
    /// operation status from the service. </param>
    /// <param name="rehydrationToken"> The rehydration token corresponding to 
    /// the operation to rehydrate. </param>
    /// <param name="cancellationToken"> A token that can be used to cancel the 
    /// request. </param>
    /// <returns> The rehydrated operation. </returns>
    /// <exception cref="ArgumentNullException"> <paramref name="client"/> or <paramref name="rehydrationToken"/> is null. </exception>
    public static CreateVectorStoreOperation Rehydrate(VectorStoreClient client, ContinuationToken rehydrationToken, CancellationToken cancellationToken = default)
    {
        Argument.AssertNotNull(client, nameof(client));
        Argument.AssertNotNull(rehydrationToken, nameof(rehydrationToken));

        CreateVectorStoreOperationToken token = CreateVectorStoreOperationToken.FromToken(rehydrationToken);

        ClientResult result = client.GetVectorStore(token.VectorStoreId, cancellationToken.ToRequestOptions());
        PipelineResponse response = result.GetRawResponse();
        VectorStore vectorStore = VectorStore.FromResponse(response);

        return client.CreateCreateVectorStoreOperation(ClientResult.FromValue(vectorStore, response));
    }

    /// <inheritdoc/>
    public override async ValueTask<ClientResult> UpdateStatusAsync(RequestOptions? options = null)
    {
        ClientResult result = await GetVectorStoreAsync(options).ConfigureAwait(false);

        PipelineResponse response = result.GetRawResponse();
        VectorStore value = VectorStore.FromResponse(response);

        ApplyUpdate(response, value);

        return result;
    }

    /// <inheritdoc/>
    public override ClientResult UpdateStatus(RequestOptions? options = null)
    {
        ClientResult result = GetVectorStore(options);

        PipelineResponse response = result.GetRawResponse();
        VectorStore value = VectorStore.FromResponse(response);

        ApplyUpdate(response, value);

        return result;
    }

    internal async Task<CreateVectorStoreOperation> WaitUntilAsync(bool waitUntilCompleted, RequestOptions? options)
    {
        if (!waitUntilCompleted) return this;
        await WaitForCompletionAsync(options?.CancellationToken ?? default).ConfigureAwait(false);
        return this;
    }

    internal CreateVectorStoreOperation WaitUntil(bool waitUntilCompleted, RequestOptions? options)
    {
        if (!waitUntilCompleted) return this;
        WaitForCompletion(options?.CancellationToken ?? default);
        return this;
    }

    private void ApplyUpdate(PipelineResponse response, VectorStore value)
    {
        Value = value;
        Status = value.Status;

        HasCompleted = GetHasCompleted(value.Status);
        SetRawResponse(response);
    }

    private static bool GetHasCompleted(VectorStoreStatus status)
    {
        return status == VectorStoreStatus.Completed ||
            status == VectorStoreStatus.Expired;
    }

    internal virtual async Task<ClientResult<VectorStore>> GetVectorStoreAsync(CancellationToken cancellationToken = default)
    {
        ClientResult result = await GetVectorStoreAsync(cancellationToken.ToRequestOptions()).ConfigureAwait(false);
        return ClientResult.FromValue(VectorStore.FromResponse(result.GetRawResponse()), result.GetRawResponse());
    }

    internal virtual ClientResult<VectorStore> GetVectorStore(CancellationToken cancellationToken = default)
    {
        ClientResult result = GetVectorStore(cancellationToken.ToRequestOptions());
        return ClientResult.FromValue(VectorStore.FromResponse(result.GetRawResponse()), result.GetRawResponse());
    }

    internal virtual async Task<ClientResult> GetVectorStoreAsync(RequestOptions? options)
    {
        using PipelineMessage message = CreateGetVectorStoreRequest(_vectorStoreId, options);
        return ClientResult.FromResponse(await _pipeline.ProcessMessageAsync(message, options).ConfigureAwait(false));
    }

    internal virtual ClientResult GetVectorStore(RequestOptions? options)
    {
        using PipelineMessage message = CreateGetVectorStoreRequest(_vectorStoreId, options);
        return ClientResult.FromResponse(_pipeline.ProcessMessage(message, options));
    }
}

```

</details>

<details>
<summary><h4><b> OperationResult </b></h4></summary>

The base abstraction `OperationResult` is implemented in the `System.ClientModel` library.  See [OperationResult.cs](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/core/System.ClientModel/src/Convenience/OperationResult.cs) for details.

</details>

</details>

### Derived type variations

Based on the details of the service operation and customer needs from the client, the implementation of the LRO subclient (the type derived from `OperationResult`) can use one or more of the following pattern variants:

1. Add a `Value` property if the operation computes a result
1. Add a `Status` property if the operation has statuses other than in-progress and completed
1. Add other applicable public properties as needed
1. Add service methods that operate on the service-side LRO itself, e.g. a `Cancel` operation if the service provides it.  If the service exposes a "Get Status"-like operation, this method is not exposed as a public service method on the LRO subclient.  This is because it is functionally equivalent to the `UpdateStatus` method on the base `OperationResult` type.
1. Add an overload of the `Rehydrate` method taking simple parameters
1. Throw an exception from the `UpdateStatus` method to indicate that a terminal error was encountered during the execution of the operation
1. Provide an overload of the `WaitForCompletion` method that takes a custom polling interval or alternate polling strategy
1. Override the virtual `WaitForCompletion` method if needed for a specific service implementation

### Further reading

For a higher-level discussion of this client pattern, see [System.ClientModel-based client pattern for LROs](https://gist.github.com/annelo-msft/bba35dde939e50370b76163d1da35593).