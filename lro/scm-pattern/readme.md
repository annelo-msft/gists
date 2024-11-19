# System.ClientModel-based client pattern for LROs

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

<details>
<summary><h4><b> Client service methods </b></h4></summary>

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