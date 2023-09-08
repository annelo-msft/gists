## Code samples for JSON Merge Patch Proposal

The following illustrates how our implementation of .NET Patch models for JSON Merge Patch would affect resources on the service side.
It shows the before/after of the resource representation on the service, alongside the HTTP request message sent to the service that caused the change.
It also shows the C# code you would write to achieve the change.

### Samples

<details>
	<summary>Create a new resource</summary>

```mermaid
sequenceDiagram
    participant client
    participant server
    Note left of client: "resource 1"
    client->>server: GET /doc HTTP/1.1
    Note right of client: <code>change <br> hi</code>
    Note right of server: "resource 2"
    server-->>client: HTTP/1.1 200
```

</details>

<details>
	<summary>Update a top-level value</summary>
</details>
