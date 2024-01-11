# System.ClientModel API

## Overview

The Azure SDK provides .NET clients that enable idiomatic development of .NET applications that communicate with Azure services. Many of our clients are created with code generation tools, and use concepts that the .NET platform doesn't necessarily provide features for, such as retry logic and credentials for authenticating with cloud services. To avoid generating the code for these concepts into every client library, we have created a shared library called Azure.Core that our generated clients depend on. We would now like to be able to create generated clients that can communicate with any cloud service, not only those in Azure. We believe putting these client building block types in a `System.` namespace is a good way to put them in a neutral bucket.

We selected the package name `System.ClientModel` for these types to indicate that the types are building blocks for clients that call cloud services.  The `System.ClientModel` package includes types in a root `System.ClientModel` namespace as well as lower-level APIs in a `System.ClientModel.Primitives` namespace.

We plan to maintain the source for the proposed `System.ClientModel` package, as well as the release pipelines needed to publish the package to NuGet, in the Azure/azure-sdk-for-net repo.

## Concepts

This section establishes the terminology we'll use to describe user groups, API categorization, and client-specific concepts.

### Two user groups

The APIs in ClientModel are designed for two different groups of users: the end-users of ClientModel clients, and the authors of ClientModel clients.  For this discussion, we will refer to these two user groups as **"client-users"** and **"client-authors"**.

### Convenience vs. control

To use terminology described by Richard Lander in the [The convenience of .NET](https://devblogs.microsoft.com/dotnet/the-convenience-of-dotnet/) series, we will use the terms **convenience** and **control** to indicate the difference between APIs designed to enable writing compact and straightforward code vs those that provide more detailed options and enable lower-level control of implemented functionality.

### Service methods

Finally, building on the user and API categorizations above, we introduce terminology for the two types of methods that ClientModel types are designed to enable on ClientModel clients.  A **service method** is a method on a ClientModel client that sends a message to a cloud service, receives the service's response, and returns a value to the client-user that called the method, for use in their application. ClientModel clients can expose two different types of service methods: "convenience methods" and "protocol methods".

**Convenience methods** are service methods that take a strongly-typed model representing schematized data needed to communicate with the cloud service as input, and return a strongly-typed model as output.  Having strongly-typed models that represent service concepts provides a layer of convenience over working with raw JSON, and unifies the experience for users of ClientModel clients when cloud services differ in wire-formats.  That is, a client-user can learn the patterns for strongly-typed models that ClientModel clients provide, and use them together without having to reason about whether a cloud service represents resources using JSON or XML.

**Protocol methods** are service methods that provide very little convenience over the raw HTTP APIs a cloud service exposes.  They have the advantage of being straightforward to generate from a service's API description, and represent request and response message bodies using types that are very thin layers over raw JSON/binary/other formats.  They require users to read and understand the service's HTTP API documentation directly, rather than relying on the client to provide developer conveniences via strongly-typing service schemas.  In the usage samples we'll share, we show that convenience methods are implemented by calling through to the lower-level protocol methods.

## Client-user APIs

This section first describes the intended pattern for using ClientModel clients in application code, and then identifies the client-user types that enable this pattern.

### Application pattern for cloud clients

The following shows an example of a minimal application that could be implemented using a ClientModel client.  It illustrates some usage of types intended for client-users.

```csharp
using Maps;
using System;
using System.Net;
using System.ClientModel;

string key = Environment.GetEnvironmentVariable("MAPS_API_KEY") ?? string.Empty;
ApiKeyCredential credential = new ApiKeyCredential(key);
MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");
    ClientResult<IPAddressCountryPair> output = client.GetCountryCode(ipAddress);

    IPAddressCountryPair ipCountryPair = output.Value;

    Console.WriteLine($"Response status code: '{output.GetRawResponse().Status}'");
    Console.WriteLine($"IPAddress: '{ipCountryPair.IpAddress}', Country code: '{ipCountryPair.CountryRegion.IsoCode}'");
}
catch (ClientResultException e)
{
    Console.WriteLine($"Error: Response status code: '{e.Status}'");
}
```

We highlight the following patterns intended to be common across ClientModel clients:

1. Construction of a ClientModel client, taking a service endpoint and credential
1. Calling a client's service method (the convenience method `GetCountryCode`)
1. Obtaining a strongly-typed model that represents a service resource (here, `IPAddressCountryPair`)
1. Retrieving lower-level details of the protocol if desired, (`result.GetRawResponse()`)
1. Handling service errors (by catching `ClientResultException`)

### Client-user types

Types intended for use by client-users in application code are grouped in the `System.ClientModel` namespace.  These include:

- `ApiKeyCredential` - supports (1)
- `ClientResult<T>` - supports (2), (3), (4)
- `ClientResult` - supports (4)
- `ClientResultException` - supports (5)

## Client-author types

This section describes ClientModel client implementation patterns that the types in the `System.ClientModel.Primitives` namespace have been designed to enable.  The patterns shown here include patterns for implementing a client constructor, patterns for implementing both convenience and protocol service methods -- both illustrating usage of types intended for client-authors -- and concludes with a discussion of APIs designed to support lower-level control and flexibility for both client-users and client-authors.

The first example here continues from the application sample in the prior section.  It sketches an implementation of the ClientModel client `MapsClient` that was used in the prior example.

At a high level, a ClientModel client can have the following parts:

```csharp
namespace Maps;

public class MapsClient
{
    //
    // 1. Private members initialized by the client constructor
    //

    //
    // 2. Client constructor
    //

    //
    // 3. Service methods
    //   - Convenience method
    //   - Protocol method
    //

    //
    // 4. Service method helpers
    //
}
```

The following sections will provide sample code filling in the parts of this client implementation.

### Client construction

In a ClientModel client constructor, client-authors create an instance of a `ClientPipeline` that the client will use to send and receive messages to/from a cloud service.  The constructor also stores any information needed to later create requests in the format expected by the service.

The following illustrates the constructor implementation for our example client:

```csharp
    public MapsClient(Uri endpoint, ApiKeyCredential credential, MapsClientOptions options = default)
    {
        if (endpoint is null) throw new ArgumentNullException(nameof(endpoint));
        if (credential is null) throw new ArgumentNullException(nameof(credential));

        options ??= new MapsClientOptions();

        _endpoint = endpoint;
        _credential = credential;
        _apiVersion = options.Version;

        var authenticationPolicy = ApiKeyAuthenticationPolicy.CreateHeaderApiKeyPolicy(credential, "subscription-key");
        _pipeline = ClientPipeline.Create(options, authenticationPolicy);
    }
```

Note that:

1. The service URI is stored in `_endpoint` for use later when creating requests
1. The user's credential is stored in `_credential` to use in the authentication policy and/or later request creation
1. The service's API version is stored so it can be added as a query parameter to request URIs
1. A `ApiKeyAuthenticationPolicy` is added to pipeline options provided by the user so it can add headers for authenticating the request when it is sent in the pipeline
1. The message pipeline is created and stored for later use in service methods.

### Service methods

As discussed in prior sections, ClientModel clients provide two types of service methods: convenience methods and protocol methods.  Examples for both are given in this section.

#### Convenience method

In our `MapsClient` example, the following shows how the `GetCountryCode` convenience method might be implemented:

```csharp
    public virtual ClientResult<IPAddressCountryPair> GetCountryCode(IPAddress ipAddress)
    {
        if (ipAddress is null) throw new ArgumentNullException(nameof(ipAddress));

        ClientResult output = GetCountryCode(ipAddress.ToString());

        PipelineResponse response = output.GetRawResponse();
        IPAddressCountryPair value = IPAddressCountryPair.FromResponse(response);

        return ClientResult.FromValue(value, response);
    }
```

Note that this method calls through to the protocol method `GetCountryCode` and passes a string instead of an `IPAddress`.  It receives the returned `ClientResult`, obtains the lower-level `PipelineResponse`, and creates a strongly-typed `IPAddressCountryPair` return value from the response content before creating a `ClientResult<T>` to return to the client-user.

The common patterns highlighted here include:

1. Converting strongly-typed inputs to `BinaryContent` (not shown in this sample)
1. Calling the protocol method corresponding to the service operation
1. Converting the response `BinaryContent` to a strongly-typed output model
1. Packaging the output model with the protocol-level response to return to the caller

#### Protocol method

The protocol method for the `GetCountryCode` operation corresponds closely to the operation as defined in the service's REST API for the [Geolocation - Get IP To Location](https://learn.microsoft.com/rest/api/maps/geolocation/get-ip-to-location?tabs=HTTP) operation.  Path and query parameters in the service API correspond to primitive-type input parameters in the client's service method, and any content meant to go in the message body would correspond to a `BinaryContent` input parameter (not shown here).  The return type is a `ClientResult`, on which `GetRawResponse()` can be called to get access to the `PipelineResponse.Content` property, where `BinaryContent` provides a thin layer of convenience over the raw response content.

The following shows how the `GetCountryCode` protocol method could be implemented on the example client:

```csharp
    public virtual ClientResult GetCountryCode(string ipAddress, RequestOptions options = null)
    {
        if (ipAddress is null) throw new ArgumentNullException(nameof(ipAddress));

        options ??= new RequestOptions();

        using PipelineMessage message = CreateGetLocationRequest(ipAddress, options);

        _pipeline.Send(message);

        PipelineResponse response = message.Response;

        if (response.IsError && options.ErrorOptions == ClientErrorBehaviors.Default)
        {
            throw new ClientResultException(response);
        }

        return ClientResult.FromResponse(response);
    }
```

Note that this method adds a `PipelineMessageClassifier` to the passed-in `RequestOptions`, creates a message with a request, sends the message via the pipeline, checks whether the response is an error response and throws an exception if needed, and finally returns the `ClientResult` to the caller.

Protocol methods are public methods and can be called from convenience methods or by client-users.

The common patterns highlighted here include:

1. Indicating which response status codes the service considers successful for this operation by setting a `PipelineMessageClassifier`
1. Calling a helper method to create the `PipelineMessage` and `PipelineRequest` specific to the service operation
1. Sending the message by calling `ClientPipeline.Send`
1. Checking `PipelineResponse.IsError` and throwing an `ClientResultException` if needed
1. Packaging the low-level `PipelineResponse` into a `ClientResult` to return to the caller.

#### Message and request creation

In the example, the protocol method calls a private `CreateGetLocationRequest` helper method to create the message populated with a `PipelineRequest` that will work with the transport that the pipeline contains.  This method could be implemented as follows:

```csharp
    private PipelineMessage CreateGetLocationRequest(string ipAddress, RequestOptions options)
    {
        PipelineMessage message = _pipeline.CreateMessage();
        message.Apply(options, new ResponseStatusClassifier(stackalloc ushort[] { 200 }));

        PipelineRequest request = message.Request;
        request.Method = "GET";

        UriBuilder uriBuilder = new(_endpoint.ToString());

        StringBuilder path = new();
        path.Append("geolocation/ip");
        path.Append("/json");
        uriBuilder.Path += path.ToString();

        StringBuilder query = new();
        query.Append("api-version=");
        query.Append(Uri.EscapeDataString(_apiVersion));
        query.Append("&ip=");
        query.Append(Uri.EscapeDataString(ipAddress));
        uriBuilder.Query = query.ToString();

        request.Uri = uriBuilder.Uri;

        request.Headers.Add("Accept", "application/json");

        return message;
    }
```

Note the following common patterns:

1. Creating the message via the pipeline
1. Applying `RequestOptions` to the message
1. Setting the request method
1. Setting the request URI
1. Setting the request headers
1. Setting the request content (not shown)

Although the example `MapsClient` only illustrates reading response content and construct a strongly-typed output model, there is a lot of complexity in how generated model code supports writing request content as well.  The next section focuses on APIs for types that handle writing input model content to requests and reading response content to create output models.

#### Reading and writing message content

In the example `MapsClient`, we've included a strongly-typed output model `IPAddressCountryPair`.  This represents the schema for the service's response to the [Geolocate IP Operation](https://learn.microsoft.com/rest/api/maps/geolocation/get-ip-to-location?tabs=HTTP).  If you consult the [REST API documentation for the response body schema](https://learn.microsoft.com/rest/api/maps/geolocation/get-ip-to-location?tabs=HTTP#ipaddresstolocationresult), you'll see that the response schema describes a pair of values -- the input `IPAddress` is paired with the `CountryRegion` that the service returns to "geolocate" the IP address, i.e. to identify the country where the device using the specified IP address is located.

The following sample sketches out what an implementation of the `IPAddressCountryPair` output model might look like:

```csharp
public class IPAddressCountryPair : IJsonModel<IPAddressCountryPair>
{
    internal IPAddressCountryPair(CountryRegion countryRegion, IPAddress ipAddress)
    {
        CountryRegion = countryRegion;
        IpAddress = ipAddress;
    }

    public CountryRegion CountryRegion { get; }

    public IPAddress IpAddress { get; }

    internal static IPAddressCountryPair FromResponse(PipelineResponse response)
    {
        // Read JSON from response.Body and return IPAddressCountryPair
    }

    public IPAddressCountryPair Create(ref Utf8JsonReader reader, ModelReaderWriterOptions options)
    {
        // Read JSON values and return IPAddressCountryPair
    }

    public IPAddressCountryPair Create(BinaryData data, ModelReaderWriterOptions options)
    {
        // Read JSON from BinaryData and return IPAddressCountryPair
    }

    public void Write(Utf8JsonWriter writer, ModelReaderWriterOptions options)
    {
        // Write JSON representing model to Utf8JsonWriter
    }

    public BinaryData Write(ModelReaderWriterOptions options)
    {
        // Write JSON representing model to returned BinaryData value
    }
}
```

Common patterns highlighted here include:

1. Model implements `IJsonModel<T>` interface
1. Model provides implementations of `IJsonModel<T>.Create` and `IPersistableModel<T>.Create` methods
1. Model provides implementations of `IJsonModel<T>.Write` and `IPersistableModel<T>.Write` methods

`IJsonModel<T>.Create` is ultimately what's called to create the strongly-typed output model from the response body in ClientModel client code, albeit typically through convenience APIs such as cast operators or by using the `ModelReaderWriter` type.  For input models, `IJsonModel<T>.Write` is what's ultimately called to create the request body to send to the service.

Further details of the APIs used to read and write ClientModel clients' strongly-typed models can be found in [ModelReaderWriter](https://gist.github.com/m-nash/626adc8b6e79eaa8c835f93fa156370c).

#### Handling the service response

Much of the logic needed to handle the service response is managed in the `PipelineTransport` and policies in the pipeline such as `ResponseBufferingPolicy`.  Today, `PipelineTransport` is responsible for invoking the `MessageClassifier.IsErrorResponse` method after the response is set on the message in order to populate the `PipelineResponse.IsError` property.  The `ResponseBufferingPolicy` does the work needed to read the content out of the response network stream into a buffered `MemoryStream`, to make it easier for client-authors and client-users accessing lower-level APIs when reading the response body.

Options for configuring the message classifier and network timeouts are exposed on `ClientPipelineOptions` and `RequestOptions` types.  Opting-out of response buffering is uncommon, so APIs enabling this are exposed only on the `ResponseBufferingPolicy` itself.

Please note that these options APIs are still in preview and subject to change.

Logic for deciding how to expose an error response to the client-user, and package successful responses into strongly typed `ClientResult<T>` return types is shown in client implementation samples in the preceding sections.

### Lower-level control and flexibility

As discussed in the introductory concepts section, some of the types in ClientModel are intended for use as conveniences, and some are designed to provide greater control in the implementation.  We discuss several types that enable client extensibility and customization in this section.

#### Options

Two options types are provided: a `ClientPipelineOptions` type that enables configuring the client instance, and a `RequestOptions` type that can be passed to protocol methods to change options for the duration of the service method invocation, or to change how the protocol method itself functions.  Both client-users and client-authors can use these types to change client and service method behavior, and precedence rules are built into these types.

A full discussion of options types available to both client-users and client-authors can be found in [Configuring ClientModel clients](https://github.com/annelo-msft/gists/blob/main/clientmodel/options.md).

#### Customizing the pipeline

Both client-authors and client-users can customize the pipeline, either for the entire client or for the duration of a service method's invocation.  This is enabled by setting `PipelinePolicy` instances on `ClientPipelineOptions` or `RequestOptions` via the different policy members on those types.  When `ClientPipeline.Create` is called in a ClientModel client constructor, policies specified on the `ClientPipelineOptions` instance passed to `Create` will be built into the client's pipeline.

Implementors of custom policies inherit from `PipelinePolicy` and implement its abstract `Process` and `ProcessAsync` methods.  Both these methods take a `PipelineMessage` and are expected to call `PipelinePolicy.ProcessNext` or `PipelinePolicy.ProcessNextAsync` to pass control to the next policy in the pipeline.  A policy implementation can modify the request (e.g. to add a header) before calling `ProcessNext`, and will have access to the buffered response after control returns from the `ProcessNext` call.  Policies can be inserted into the pipeline before the retry policy by adding them to `PipelineOptions.PerCallPolicies` or after the retry policy by adding them to `PipelineOptions.PerTryPolicies`.
