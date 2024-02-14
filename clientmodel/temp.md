# Response.Content

## Problem

Azure.Core-based client protocol methods return `Response`.  Without reading the docs, a caller of such a method may not know whether the response content has been buffered or not.  Today, in Azure.Core, if the response content has not been buffered, accessing the `Response.Content` property will throw an `InvalidOperationException`.

The problem with throwing from the `Response.Content` property is that it is not idiomatic to .NET.  In .NET, APIs are either fully polymorphic (i.e. they just work), or we supply "testers" to know in advance whether calling the API will throw.

## Proposals

### 1. Make Response.Content "always work"

In this proposal:

- For buffered responses
  - If the caller accesses Content, we return a BinaryData wrapping the byte[] content
  - If the caller accesses ContentStream, we return a MemoryStream wrapping the byte[] content

- For unbuffered responses
  - If the caller accesses Content first, we read the live stream, buffer into byte[] and return a BinaryData wrapping the byte[] content
    - If a subsequent call accesses ContentStream, we return a MemoryStream wrapping the byte[] content
  - If the caller accesses ContentStream first, we return the live stream.
    - If a subsequent call accesses ContentStream again, we return the same stream, but if it was a network stream, it can no longer be read from.
    - If a subsequent call accesses Content, the Content property will throw because we cannot buffer the content after the network stream was read.

Advantages:

- In most cases, callers of protocol methods can access response.Content and it will just work.  This is idiomatic to .NET.

Disadvantages:

- To read the bytes from the network stream in the Content property in the unbuffered case, we must read the stream synchronously.
- If the network stream is too large to fit in memory in the unbuffered case, we will throw an IOException.  This is a different exception type than we throw today in Azure.Core.

### 2. Continue to throw from Response.Content

- For buffered responses
  - If the caller accesses Content, we return a BinaryData wrapping the byte[] content
  - If the caller accesses ContentStream, we return a MemoryStream wrapping the byte[] content

- For unbuffered responses
  - If the caller accesses Content, throw an `InvalidOperationException`
  - If the caller accesses ContentStream, we return the live stream.

Advantages:

- We do not try to synchronously read from a network stream, or risk loading too much data into memory.

Disadvantages:

- We ship a type in System.ClientModel that is not idiomatic to .NET because it not fully polymorphic and it does not provide a tester.

### 3. Add a tester to Response

In this approach, we use either option #1 or option #2 above, and we add a property `IsBuffered` to `PipelineResponse` that is inherited by `Response`.  Callers of protocol methods who are uncertain whether to call `Content` or `ContentStream` can call `IsBuffered` to understand whether it is safe to access `Content`.

Advantages:

- The type is idiomatic to .NET in that it provides a "tester" API to prevent accessing a member on the type that might throw an exception

Disadvantages:

- End users might think they always have to check `IsBuffered` before accessing the `Content` property when it is uncommon that responses are unbuffered.  This could make call-sites overly complex.

### 4. Introduce a StreamingResponse type

In this approach, we introduce a `StreamingPipelineResponse` type with only the `ContentStream` property, and make it the base type of the `Response` hierarchy.  Azure.Core-based clients would return `StreamingPipelineResponse` from protocol methods.  ClientModel-based clients would continue to return `ClientResult` from protocol methods.

This would look as follows:

```csharp
public abstract class StreamingPipelineResponse {
    Stream? ContentStream { get; set; }
}

public abstract class PipelineResponse : StreamingPipelineResponse { 

    // Inherits ContentStream
    BinaryData Content { get; }
}

public abstract class Response : PipelineResponse { 
    // Inherits both ContentStream and Content
}
```

Azure.Core protocol methods would need to return `StreamingPipelineResponse`, which would have different header types than Azure.Core `Response` header types, which could lead to confusion.

It may also not work with protocol methods on ClientModel-based clients.  This is because ClientModel-based clients return `ClientResult` from protocol methods, not `PipelineResponse`.  `ClientResult.GetRawResponse()` would have to return the base type -- i.e. a `StreamingPipelineResponse`, which the user would have to down-cast to a `PipelineResponse` for the majority of use cases.

## Recommendation

Given how easy it would be to add an `IsBuffered` property to `PipelineResponse` later on, I would recommend we go with option 1 and add the tester if we find that users have a problem with not knowing whether to call `ContentStream`.