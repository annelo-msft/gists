# System.ClientModel-based client pattern for LROs

## LRO definition

A long-running operation (LRO) is an operation that a cloud *service* executes asynchonously.  For HTTP-based services, this means that the service will typically respond to a request to start the operation prior to the completion of the operation.

See [Azure API Guidelines | Long-Running Operations & Jobs](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#long-running-operations--jobs) for further details.

From a *client* perspective, a client pattern for LROs must support these steps:

1. The client initiates the operation on the service
1. The client polls the service to track the progress of the operation
1. The client indicates to its user that the operation is complete and any computed results can be obtained

### Messages sent between client and service

![image](https://gist.github.com/user-attachments/assets/fcf2cdb9-2f4d-4be4-ae21-fddc99cac566)

## SCM-based client pattern

The `System.ClientModel`-based client pattern for long-running operations in .NET clients consists of three elements:

1. Service methods on the client used to initiate the LRO
1. A base abstraction [OperationResult](https://learn.microsoft.com/en-us/dotnet/api/system.clientmodel.primitives.operationresult?view=azure-dotnet)
1. A type derived from `OperationResult`, implemented as a public type in the client assembly (the *_LRO subclient_*)

### C# usage samples

<details>
<summary><h3><b> 1. Start LRO, return when completed </b></h3></summary>

```csharp
VectorStore vectorStore = client.CreateVectorStore(waitUntilCompleted: true).Value;
```

</details>

<details>
<summary><h3><b> 2. Start LRO, wait for completion via LRO subclient </b></h3></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
createOperation.WaitForCompletion();
VectorStore vectorStore = createOperation.Value;
```

</details>

<details>
<summary><h3><b> 3. Start LRO, wait for completion using custom polling interval </b></h3></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
createOperation.WaitForCompletion(pollingInterval: TimeSpan.FromSeconds(2));
VectorStore vectorStore = createOperation.Value;
```

</details>

<details>
<summary><h3><b> 4. Start LRO, manually poll for updates (advanced) </b></h3></summary>

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
<summary><h3><b> 5. Start LRO, view HTTP response details (advanced) </b></h3></summary>

```csharp
CreateVectorStoreOperation createOperation = client.CreateVectorStore(waitUntilCompleted: false);
PrintHttpDetauls(createOperation.GetRawResponse());
while (!createOperation.HasCompleted)
{
    await Task.Delay(2000);
    createOperation.UpdateStatus();
    PrintHttpDetauls(createOperation.GetRawResponse());
}
VectorStore vectorStore = createOperation.Value;

void PrintHttpDetails(PipelineResponse response)
{
    Console.WriteLine("Status code: " + response.Status);
}
```

</details>

<details>
<summary><h3><b> 6. Start LRO, wait for completion from a different process ("Rehydrate") (advanced) </b></h3></summary>

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