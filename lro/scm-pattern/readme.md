# System.ClientModel-based client pattern for LROs

## LRO definition

A long-running operation (LRO) is an operation that a cloud *service* executes asynchonously.  For HTTP-based services, this means that the service will typically respond to a request to start the operation prior to the completion of the operation.

See [Azure API Guidelines | Long-Running Operations & Jobs](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#long-running-operations--jobs) for further de

From a *client* perspective, a client pattern for LROs must support these steps:

1. The client initiates the operation on the service
1. The client polls the service to track the progress of the operation
1. The client indicates to its user that the operation is complete and any computed results can be obtained
