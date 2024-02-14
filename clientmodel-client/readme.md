# Differences between Azure.Core-based clients and System.ClientModel-based clients

## Overview

Differences fall into the following categories:

- [Type name changes](#type-names)
- [Type usage changes](#type-usage)
- [Type removals](#type-removal)
- [Client API changes](#client-apis)
- [Nullable annotations](#nullable-annotations)

## Type names

| Azure.Core type | System.ClientModel type |
| ------------- | ------------- |
| `AzureKeyCredential` | `ApiKeyCredential` |
| `AzureKeyCredentialPolicy` | `ApiKeyAuthenticationPolicy` |
| `ClientOptions` | `ClientPipelineOptions` |
| `ErrorOptions` | `ClientErrorBehaviors` |
| `HttpClientTransport` | `HttpClientPipelineTransport` |
| `HttpMessage` | `PipelineMessage` |
| `HttpPipeline` | `ClientPipeline` |
| `HttpPipelinePolicy` | `PipelinePolicy` |
| `HttpPipelinePosition` | `PipelinePosition` |
| `HttpPipelineTransport` | `PipelineTransport` |
| `NullableResponse<T>` | `ClientResult<T?>` |
| `Request` | `PipelineRequest` |
| `RequestContent` | `BinaryContent` |
| `RequestContext` | `RequestOptions` |
| `RequestHeaders` | `PipelineRequestHeaders` |
| `RequestFailedException` | `ClientResultException` |
| `Response` | `PipelineResponse` |
| `Response<T>` | `ClientResult<T>` |
| `ResponseClassifier` | `PipelineMessageClassifier` |
| `ResponseHeaders` | `PipelineResponseHeaders` |
| `RetryPolicy` | `ClientRetryPolicy` |

[Azure.Core APIView](https://apiview.dev/Assemblies/Review/ba87b735158144eea6cabe21a2c58dde)
[System.ClientModel APIView](https://apiview.dev/Assemblies/Review/1b123e7a51d44ebe945f0212ee039c65)

## Type usage

Usage of ClientModel types differ from Azure.Core types in the following areas:

- Call to `HttpPipelineBuilder.Build` is replaced by `ClientPipeline.Create`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L32-L35).
- Client endpoint is a local variable and not a property on client options. [Client example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L27). [Client options example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClientOptions.cs#L9).
- `ClientPipeline.Send` does not take a cancellation token. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L60).
- Generated `PipelineMessageClassifier` instances are created using `PipelineMessageClassifier.Create` instead of classifier constructor. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/OpenAIClient.cs#L99).
- `PipelineMessage.Apply(options)` is called at the end of a `CreateRequest` method instead of at the beginning. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L132).
- `PipelineRequest.Method` is set from a string value instead of an extensible enum value. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L110).
- Instances of `ClientResultException` created in async contexts use `ClientResultException.CreateAsync` instead of exception constructor. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L66).

## Type removal

A number of types have been removed from System.ClientModel.  As a result client method implementations will change as follows.

- `ClientDiagnostics` is removed.  As a result, calls `CreateScope`, `scope.Start` and try/catch block around method implementation are removed. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L72).
- `RequestUriBuilder` is replaced by BCL `Uri` type.  `CreateRequest` methods use BCL `UriBuilder` to create the request URI.
- Shared source type `RawRequestUriBuilder` is removed.  Generated clients escape URI query parameter values inline. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L112-L126).
- Shared source type `HttpPipelineExtensions` is removed.  Generated clients call `pipeline.Send` and check `response.IsError` inline in the protocol method. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L92-L101).
- Shared source type `Argument` is internal to ClientModel code.  Generated clients inline argument checks.  [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L74).
- There are no pageable or long-running operations in `System.ClientModel`-based clients today.

## Client APIs

System.ClientModel-based clients should have the same APIs as Azure.Core-based clients (modulo type name changes) in all areas, except the following.

- Convenience methods do not take `CancellationToken` parameter. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L72).
  - As a result, convenience methods do not create an instance of `RequestOptions` to pass to the nested protocol method call. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L76).
- Protocol methods return `ClientResult` instead of `PipelineResponse`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L84).
  - As a result, convenience methods first call `result.GetRawResponse` to get the `PipelineResponse` they use to create the method return value. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L78).
  - Similarly, protocol methods call `ClientResult.FromResponse` to create the return value instead of returning the message.Response. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L81).

## Nullable annotations

System.ClientModel-based clients use nullable annotations to indicate that a convenience method may return null from `result.Value` instead of a model instance.  Whether or not a client has such a service method in its first release, because enabling nullable annotations for an assembly is a breaking change, nullable annotations must be turned on from the start in case such an operation is ever added by the service.

The changes needed to support nullable annotations are:

- Client's `.csproj` file contains `<Nullable>enable</Nullable>`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/System.ClientModel.Tests.Client.csproj#L7).
- Optional `ClientPipelineOptions` parameter to client constructor is nullable. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L20).
- Optional `RequestOptions` parameter to protocol methods is nullable. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L51).
- Protocol methods set `response` to non-nullable `PipelineResponse` using null-forgiving operator `!`.  This is allowed because transport validates response is non-null before returning a message to pipeline policies.  [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/MapsClient.cs#L62).
- Lazily-initialized `PipelineMessageClassifier` private field is nullable. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/OpenAIClient.cs#L98).
- Model files must either remove `// <auto-generated/>` or add `#nullable enabled`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/Choice.Serialization.cs#L1-L6).
- Optional `ModelReaderWriterOptions` parameters to internal methods calling `ModelReaderWriter.Write` are nullable.
- Optional properties that are reference types use nullable annotations. Affects constructor [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/Choice.cs#L21) and property definition [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/Choice.cs#L61).
- Model deserialization routines throw `JsonException` if `(element.ValueKind == JsonValueKind.Null)`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/CountryRegion.cs#L23).
- Nested models that can be deserialized correctly from JSON `null` are set to null by the parent model deserialization routine. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/Choice.Serialization.cs#L49-L58).
- Model deserialization routines declare variables used to read required properties from JSON as nullable. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/CountryRegion.cs#L26).
  - These variables are checked to validate that they have been populated prior to creating type instance.  If they do not have values, throw `JsonException`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/MapsClient/CountryRegion.cs#L37-L40).
- Model deserialization routines use `valueTypeVariable!.Value` to pass required value type property to type constructor. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/Completions.Serialization.cs#L94).
- Model deserialization routines deserializing collections of non-nullable values check for null when enumerating collection values and throw `JsonException` if a value is null. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/CompletionsLogProbabilityModel.Serialization.cs#L31-L33).
- Model serialization routines use null-forgiving operator `!` for optional properties that have already been checked for null by call to `OptionalProperty.IsDefined`. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/CompletionsOptions.Serialization.cs#L26).
- Models that are extensible enums use nullable annotation in `Equals` method override. [Example](https://github.com/Azure/azure-sdk-for-net/blob/feature/core-experiment/sdk/core/System.ClientModel/tests/client/OpenAIClient/CompletionsFinishReason.cs#L46).
