# System.ClientModel-based client pattern for LROs

## LRO definition

A long-running operation (LRO) is typically an operation that should execute synchronously, but due to services not wanting to maintain long-lived connections (>1 seconds) and load-balancer timeouts, the operation must execute asynchronously. For this pattern:

1. The client initiates the operation on the service, and then 
2. the client repeatedly polls the service (via another API call) to track the operation's progress/completion.

From [Azure API Guidelines | Long-Running Operations & Jobs](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#long-running-operations--jobs)
    
- Client-side user experience
    1. User asks client to initiate the LRO through some means
        1. User can specify the following:
            1) The desire to be notified once the LRO has completed
            2) The desire to be notified once the LRO has started and obtain a type instance that enables more granular visibility and control over interaction with the service operation
    2. Upon receipt of the user's request to initiate the LRO, the client sends a request to the service to begin the operation.
    3. The client receives a response from the service containing values that indicate:
        1. How to send subsequent requests to determine the status of the operation and if the operation has completed.  
        2. Alternatively, the service request may not contain this information if it is fully specified in the service API definition
        3. Optionally, whether the operation has already completed.
    4. After the initial request, if the client has not determined that the operation has already completed, the client polls for updates regarding the operation status in the manner the service has specified.  This can happen in different ways based on values provided by the user:
        1. Client can poll for status updates at a default polling interval; user can customize the polling interval/strategy
        2. If the user obtained a type instance exposing APIs beyond those on the client used to initiate the operation, the user can use that type to:
            1) Check whether the operation has completed
            2) Wait for the operation to complete in a manner idiomatic to the language the client is implemented in
            3) Manually poll for status updates via the client affordance
            4) Obtain the HTTP details for each service response
            5) Persist a value that can be used by the same process or a different process to "rehydrate" the operation
    5. Once the client has determined that the operation has completed, it communicates this to the user in whatever manner the user specified it wished to receive that notification.  In addition, depending on the type of service operation and the details of its implementation on the service, the client may expose the following additional information:
        1. If the service operation computed a result (an "output"), the client makes that available to the user
            1) If the "output" value is useful at intermediate stages of the operation (as may be the case in some AI scenarios where the user wants to continue the computation until some confidence threshold has been reached, and then terminate the operation in order to save the cost of compute time), the client affordance can make these intermediate values available to the user as well.
            2) Otherwise, the "output" is made available when the LRO has successfully completed
        2. If the service exposes information regarding stages of the operation lifecycle while is it running (a "status"), the client affordance makes that visible to the user at the increment of the polling interval
        3. If the service exposes operations that target a given LRO instance (i.e. a "Cancel" operation that terminates execution of an in-progress LRO), the client affordance makes those available in such a way that a call to those methods can be successful, i.e. those APIs become available after the operation is started and not before then.
    6. A persisted identifier for the operation can be used to "rehydrate" the client affordance for the service operation, from either the same or a different process.
        1. The LRO can be rehydrated while it is in progress or after it has completed, for as long as the service retains information regarding its existence and status.
        2. If the service operation is still in in progress when the user rehydrates the client affordance, the user can wait for completion in the same way they would if they had started the operation themselves.
        3. If the service operation has completed when the user rehydrates the client affordance, they can interact with it in the same way they would it they had started the operation themselves, i.e. if an output value is available the user can obtain this, if a status that indicates why the operation completed, they can view this, etc.
        
    
- SCM-based client pattern elements
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
