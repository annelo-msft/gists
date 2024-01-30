# Configuring ClientModel clients

Two options types are provided: a `ClientPipelineOptions` type that enables configuring the client instance, and a `RequestOptions` type that can be passed to protocol methods to change options for the duration of the service method invocation, or to change how the protocol method itself functions.  Both [client-users and client-authors](https://github.com/annelo-msft/gists/blob/main/clientmodel/readme.md#two-user-groups) can use these types to change client and service method behavior, and precedence rules are built into these types.

## Client-user options

### Client-scope options

ClientModel clients provide a type that inherits from `ClientPipelineOptions` to provide a client-specific options type used to set options with a client-wide scope including:

- Pipeline customizations
  - Custom per-call and per-retry policies via the `options.AddPolicy` method
  - Custom RetryPolicy via `options.RetryPolicy`
  - Custom Transport via `options.Transport`
- Service-specific options like `ServiceVersion` via client options constructor or properties
  - Client-wide `NetworkTimeout` override via `options.NetworkTimeout`

### Method-scope options
  
ClientModel clients take a `RequestOptions` parameter on protocol methods.  This is used specify options for the duration of a single service method invocation.  This includes:

- Adding headers to a request via `options.AddHeader` or `options.SetHeader`.
- Providing method-scope cancellation token via `options.CancellationToken`
- Setting non-default behavior when the service returns an error response via `options.ErrorOptions`
- Per-invocation pipeline customizations
  - Custom per-call and per-retry policies via the `AddPolicy` method

## Client-author options

Client-authors need the same knobs as end-users to override Core ClientModel defaults for each of the configurable settings described above.  The options passed by client-users take precedence over client-author settings.

### Client-scope options

- Custom per-call and per-retry pipeline policies
  - Passed to `pipeline.Create` as separate parameters from `ClientPipelineOptions`
- Custom RetryPolicy and custom transport in pipeline
  - Not enabled today, but if needed, an overload of `ClientPipeline.Create` can be provided that takes these values
- Service-specific options like `ServiceVersion`
  - Client-wide defaults are set in the `ClientPipelineOptions` subtype itself
- Client-wide `NetworkTimeout` override
  - Not enabled today, but since this is a parameter passed to the ResponseBufferingPolicy constructor, we could add an overload of  `ClientPipeline.Create` that allows overriding the default buffering policy and let the client-author construct it directly to set this value.

### Method-scope options

- Adding headers to a request
  - Set on `PipelineRequest.Headers` directly when request is created
- `CancellationToken` is not needed to be specified by client-author
- Method behavior on error response is written into the method implementation directly
- Per-invocation pipeline customizations
  - Not enabled today, but if needed, we could expose these APIs via:
    - Overloads of `message.Apply(options)` that take custom policies
    - Letting client-authors add custom policies to the message property bag
- Response error classification
  - A `PipelineMessageClassifier` specific to the service operation is passed via `message.Apply(options)`

## Tabular Overview of Options

Compiling the above into a table view to make it easy to compare how options can be specified by different users in a side-by-side format.

Precedence rules are: [client-user](https://github.com/annelo-msft/gists/blob/main/clientmodel/readme.md#two-user-groups) option overrides [client-author](https://github.com/annelo-msft/gists/blob/main/clientmodel/readme.md#two-user-groups) option overrides the ClientModel Core default.

| **Option**                   | **client-user**                    | **client-author**       | **Core default**     |
|------------------------------|------------------------------------|-------------------------|----------------------|
| Client-scope pipeline policy | `ClientPipelineOptions.AddPolicy`   | `ClientPipeline.Create` | n/a                  |
| Client-scope retry policy    | `ClientPipelineOptions.RetryPolicy` | n/a                     | `ClientRetryPolicy` |
| Client-scope transport       | `ClientPipelineOptions.Transport`   | n/a                     |  `HttpClientPipelineTransport`    |
| Client-scope ServiceVersion     | `ClientPipelineOptions` ctor | `ClientPipelineOptions` internal member                    |  n/a   |
| Client-scope network timeout | `ClientPipelineOptions.NetworkTimeout`  |  `ResponseBufferingPolicy` ctor   |  internal `ResponseBufferingPolicy.DefaultNetworkTimeout`  |
| Client-scope error response classification | n/a | n/a | internal `PipelineMessageClassifier.Default` |
| Method-scope request headers | `RequestOptions.AddHeader` or `RequestOptions.SetHeader` | `PipelineRequest.Headers` |  n/a  |
| Method-scope CancellationToken | `RequestOptions.CancellationToken` | n/a |  n/a   |
| Method-scope error response behavior | `RequestOptions.ErrorOptions` | service method implementation | `ClientErrorBehaviors.Default` |
| Method-scope pipeline policy | `RequestOptions.AddPolicy` | n/a |  n/a |
| Method-scope error response classification | n/a | `PipelineMessage.Apply(options)` |  n/a |

## Usage Samples

In this section, we share samples illustrating how end-users of ClientModel clients ([client-users](https://github.com/annelo-msft/gists/blob/main/clientmodel/readme.md#two-user-groups)) can configure those clients for their specific needs.

### Client-scope configuration

#### Changing the service version

```csharp
// Service version is available on client subtype of ServiceClientOptions
MapsClientOptions options = new MapsClientOptions(MapsClientOptions.ServiceVersion.V1);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential, options);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");
    ClientResult<IPAddressCountryPair> output = client.GetCountryCode(ipAddress);

    Assert.AreEqual("US", output.Value.CountryRegion.IsoCode);
    Assert.AreEqual(IPAddress.Parse("2001:4898:80e8:b::189"), output.Value.IpAddress);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

#### Setting the network timeout

```csharp
MapsClientOptions options = new MapsClientOptions();
options.NetworkTimeout = TimeSpan.FromSeconds(2);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential, options);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");
    ClientResult<IPAddressCountryPair> output = client.GetCountryCode(ipAddress);

    Assert.AreEqual("US", output.Value.CountryRegion.IsoCode);
    Assert.AreEqual(IPAddress.Parse("2001:4898:80e8:b::189"), output.Value.IpAddress);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

#### Adding a policy to the client pipeline

```csharp
MapsClientOptions options = new MapsClientOptions();
CustomPolicy customPolicy = new CustomPolicy();
options.AddPolicy(customPolicy, PipelinePosition.PerCall);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential, options);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");
    ClientResult<IPAddressCountryPair> output = client.GetCountryCode(ipAddress);

    Assert.IsTrue(customPolicy.ProcessedMessage);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

#### Overriding the default transport

```csharp
MapsClientOptions options = new MapsClientOptions();
options.Transport = new CustomTransport();

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential, options);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");
    ClientResult<IPAddressCountryPair> output = client.GetCountryCode(ipAddress);

    PipelineResponse reponse = output.GetRawResponse();

    Assert.AreEqual("CustomTransportResponse", reponse.ReasonPhrase);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

### Method-scope configuration

#### Passing a CancellationToken to a service method

```csharp
string key = Environment.GetEnvironmentVariable("MAPS_API_KEY") ?? string.Empty;
ApiKeyCredential credential = new ApiKeyCredential(key);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");

    RequestOptions options = new RequestOptions();
    options.CancellationToken = new CancellationToken();

    // Call protocol method in order to pass RequestOptions
    ClientResult output = client.GetCountryCode(ipAddress.ToString(), options);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

#### Adding a header to a request

```csharp
string key = Environment.GetEnvironmentVariable("MAPS_API_KEY") ?? string.Empty;
ApiKeyCredential credential = new ApiKeyCredential(key);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");

    RequestOptions options = new RequestOptions();
    options.RequestHeaders.Add("CustomHeader", "CustomHeaderValue");

    // Call protocol method in order to pass RequestOptions
    ClientResult output = client.GetCountryCode(ipAddress.ToString(), options);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

#### Adding a policy to the pipeline for the duration of an operation

```csharp
string key = Environment.GetEnvironmentVariable("MAPS_API_KEY") ?? string.Empty;
ApiKeyCredential credential = new ApiKeyCredential(key);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");

    RequestOptions options = new RequestOptions();
    CustomPolicy customPolicy = new CustomPolicy();
    options.AddPolicy(customPolicy, PipelinePosition.PerCall);

    // Call protocol method in order to pass RequestOptions
    ClientResult output = client.GetCountryCode(ipAddress.ToString(), options);

    Assert.IsTrue(customPolicy.ProcessedMessage);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```

#### Changing the behavior of a service method for an error response

```csharp
string key = Environment.GetEnvironmentVariable("MAPS_API_KEY") ?? string.Empty;
ApiKeyCredential credential = new ApiKeyCredential(key);

MapsClient client = new MapsClient(new Uri("https://atlas.microsoft.com"), credential);

try
{
    IPAddress ipAddress = IPAddress.Parse("2001:4898:80e8:b::189");

    RequestOptions options = new RequestOptions();
    options.ErrorOptions = ErrorBehavior.NoThrow;

    // Call protocol method in order to pass RequestOptions
    ClientResult output = client.GetCountryCode(ipAddress.ToString(), options);
}
catch (ClientResultException e)
{
    Assert.Fail($"Error: Response status code: '{e.Status}'");
}
```
