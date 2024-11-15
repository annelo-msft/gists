# System.ClientModel-based client pattern for LROs

## LRO definition

The following definition comes from [Azure API Guidelines | Long-Running Operations & Jobs](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#long-running-operations--jobs):

_A long-running operation (LRO) is typically an operation that should execute synchronously, but due to services not wanting to maintain long-lived connections (>1 seconds) and load-balancer timeouts, the operation must execute asynchronously. For this pattern:_

1. _The client initiates the operation on the service, and then_
2. _The client repeatedly polls the service (via another API call) to track the operation's progress/completion._
    
## Client-side experience

The following description is intended to be independent of the programming language a client is implemented in, but since it comes from the .NET pattern, it is possible it contains elements that are too specific to .NET for other language clients to adopt.  That is fine -- my expectation is that we would want to align on a similar feature set across languages, but implement them in a way that is idiomatic to a given language.

### 1. User starts the LRO

The user asks client to initiate the LRO through some means.  When doing so, the user can specify the following:

1. The desire to be notified once the LRO has completed
2. The desire to be notified once the LRO has started and obtain a type instance that enables more granular visibility and control over interaction with the service operation

In .NET, users call service methods on the client to achieve this.  The details of this service method are outlined below.

### 2. Client sends initial request to the service

Upon receipt of the user's request to initiate the LRO, the client sends a request to the service to begin the operation.

In .NET, the clients creates a message and sends it via the client pipeline.  The implementation of this is internal to the client.

### 3. Client receives initial response from the service

In response to the client request, the service sends a response containing values that in some way indicate the operation has started.

It may be fully specified in the service API how the client can send subsequent requests for updates regarding the status of the operation and whether the operation has completed.  
Alternatively, the service's initial response may contain values that tell the client how to make these subseqent requests.
The service response may optionally indicate to the client that the operation has already completed.

In .NET, the service response is obtained from the message once the client pipeline's send operation has completed.  The client's service method internally creates an _LRO subclient_ as described below an returns this to the user.

### 4. Client polls for status updates

After the initial request, if the client has not determined that the operation has already completed, the client polls for updates regarding the operation status in the manner the service has specified.

This can happen in different ways based on values provided by the user:

1. Client can poll for status updates at a default polling interval; user can customize the polling interval/strategy
2. If the user obtained a type instance exposing APIs beyond those on the client used to initiate the operation, the user can use that type to:
    1. Check whether the operation has completed
    2. Wait for the operation to complete in a manner idiomatic to the language the client is implemented in
    3. Manually poll for status updates via the client affordance
    4. Obtain the HTTP details for each service response
    5. Persist a value that can be used by the same process or a different process to "rehydrate" the operation

In .NET, these different approaches are achieved by either the way the user calls the service method, or how they interact with the subtype of `OperationResult` that the service method returns.  Details of this pattern for `System.ClientModel`-based clients are described below.

### 5. Client indicates LRO has completed

Once the client has determined that the operation has completed, it communicates this to the user in the manner the user specified it wished to receive that notification.

In addition, depending on the type of service operation and the details of its implementation on the service, the client may expose the following additional information for the completed LRO:

1. If the service operation computed a result (an _"output"_), the client makes that available to the user
    1. If the "output" value is useful at intermediate stages of the operation (as may be the case in some AI scenarios where the user wants to continue the computation until some confidence threshold has been reached, and then terminate the operation in order to save the cost of compute time), the client affordance can make these intermediate values available to the user as well.
    2. Otherwise, the "output" is made available when the LRO has successfully completed
2. If the service exposes information regarding stages of the operation lifecycle while is it running (a _"status"_), the client affordance makes that visible to the user at the increment of the polling interval
3. If the service exposes operations that target a given LRO instance (i.e. a "Cancel" operation that terminates execution of an in-progress LRO), the client affordance makes those available in such a way that a call to those methods can be successful, i.e. those APIs become available after the operation is started and not before then.

In .NET, the client affordance provided for viewing outputs, operation status, and operations related to an in-progress LRO, is an LRO subclient, or a public type derived from `OperationResult`.

### 6. LRO rehydration

The client makes it possible for the user to persist and identifier for the operation that can be used to "rehydrate" the client affordance for the service operation.  This enables recreating the client affordance from either the same or a different process.

1. The LRO can be rehydrated while it is in progress or after it has completed, for as long as the service retains information regarding its existence and status.
2. If the service operation is still in in progress when the user rehydrates the client affordance, the user can wait for completion in the same way they would if they had started the operation themselves.
3. If the service operation has completed when the user rehydrates the client affordance, they can interact with it in the same way they would it they had started the operation themselves, i.e. if an output value is available the user can obtain this, if a status that indicates why the operation completed, they can view this, etc.

In .NET, clients provide a client-side continuation token object (not identical to a service-side continuation token, but a service-side continuation token may be used in the implementation of the client-side type), and a static `Rehydrate` method on the LRO subclient.  Details of this are described below.

## SCM-based client pattern

The `System.ClientModel`-based client pattern for long-running operations consists of three elements: the base abstraction type `OperationResult`, the public LRO subclient, and the service methods the client exposes to start the LRO.

    1. Base abstraction type -- OperationResult
        1. Provides public `HasCompleted` property, which the implementation must set to true in its implementation of `UpdateStatus` when the service update indicates the operation has completed.
        2. Provides a `WaitForCompletion` method that can be called synchronously or asynchronously to wait for the operation to complete from users' application code.
        3. Provides a `UpdateStatus` method called from `WaitForCompletion`, but public to enable low-level control of polling for advanced scenarios
        4. Provides a `GetRawResponse` method to provide access to HTTP response details for advanced scenarios
        5. Provides a `RehydrationToken` that can be persisted and used to rehydrate the OperationResult derived type as a runtime instance.
    2. Derived type -- LRO subclient
        1. Adds `Value` property if applicable
        2. Adds `Status` property if applicable
        3. Adds other applicable public properties as needed
        4. Adds static `Rehydrate` method taking relevant client, client-side continuation token
            1) Convenience overloads can be added for this has needed per user scenarios
        5. Overrides abstract `UpdateStatus` methods to
            1) Obtain status of the operation on the service, as specified by the service API and possibly values obtained from the first service response
            2) Update public properties on the LRO subclient from the service response
            3) Make the details of the HTTP response available by calling `SetRawResponse`
        6. May optionally override `WaitForCompletion` 
        7. Add any service methods that operate on the service-side LRO itself, e.g. a `Cancel` operation
            1) If the service exposes a `GetStatus` operation, this method is not exposed as a public service method on the LRO subclient.  This is because it is functionally equivalent to the `UpdateStatus` method on the base `OperationResult` type.
    3. Client service methods
        1. Clients expose service methods that users call to start the operation on the service
        2. These methods are named in a way that indicates what the long-running operation on the service does, in the following pattern:
                a) Service method name: <MethodName>, e.g. CreateVectorStore
                b) Derived type name: <MethodNameOperation>, e.g. CreateVectorStoreOperation
        3. Inputs to these methods include:
            1) Parameters needed to create the HTTP request to start the operation
            2) An additional boolean parameter `waitUntilCompleted` that indicates whether the method should return the derived type instance after the first request is sent and the service response is received, or whether the service method should internally call `WaitForCompletion` on the derived type instance before returning
        4. Method return types are:
            1) For context, .NET clients expose high- and low-level service methods for a given service endpoint: convenience methods take model types and `CancellationToken` parameters as input, and protocol methods take primitive values corresponding to query and path parameters in the HTTP request, a `BinaryContent` value corresponding to the HTTP request body, and `RequestOptions` parameters.  
            2) Service methods that start LROs return the same type from both convenience and protocol methods.  This is the type derived from OperationResult for the specific operation.  
            3) The reason for this is that the derived type is a subclient itself, and evolves in the same way as other .NET clients do.  It can be shipped quickly in an initial release with only protocol methods.  Convenience methods can be added in later versions, and that investment in design effort can be dialed according to customer needs.
                a) Protocol-only LRO subclients do not expose strongly-typed `Value` or `Status` properties -- similar to how protocol methods are used, end users can obtain these values by parsing the HTTP details obtained from calling `GetRawResponse`
                b) Protocol-only LRO subclients can defer addition of `ContinuationToken` and `Rehydration` methods until the convenience layer is added, but this is not recommended. -->
