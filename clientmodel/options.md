# Configuring ClientModel clients

Two options types are provided: a `ServiceClientOptions` type that enables configuring the client instance, and a `RequestOptions` type that can be passed to protocol methods to change options for the duration of the service method invocation, or to change how the protocol method itself functions.  Both client-users and client-authors can use these types to change client and service method behavior, and precedence rules are built into these types.

In this section, we discuss how end users of ClientModel clients can configure those clients for their specific needs.

## Client-scope configuration

### Changing the service version

### Setting the network timeout

### Adding a policy to the client pipeline

### Overriding the default transport

## Operation-scope configuration

### Passing a CancellationToken to a service method

### Adding a header to a request

### Adding a policy to the pipeline for the duration of an operation

### Changing the behavior of a service method for an error response
