# System.ClientModel-based client pattern for LROs

## LRO definition

A long-running operation (LRO) is an operation that a cloud *service* executes asynchonously.  For HTTP-based services, this means that the service will typically respond to a request to start the operation prior to the completion of the operation.

See [Azure API Guidelines | Long-Running Operations & Jobs](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#long-running-operations--jobs) for further details.

From a *client* perspective, a client pattern for LROs must support these steps:

1. The client initiates the operation on the service
1. The client polls the service to track the progress of the operation
1. The client indicates to its user that the operation is complete and any computed results can be obtained

## SCM-based client pattern

The `System.ClientModel`-based client pattern for long-running operations in .NET clients consists of three elements:

1. The base abstraction type `OperationResult`
2. The LRO subclient implemented as a public type in the client assembly
3. The service methods the client exposes to initiate the LRO

Before moving to the details of this pattern, consider the following illustration.

![image](https://gist.github.com/user-attachments/assets/fcf2cdb9-2f4d-4be4-ae21-fddc99cac566)