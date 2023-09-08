## Code samples for JSON Merge Patch Proposal

The following illustrates how our implementation of .NET Patch models for JSON Merge Patch would affect resources on the service side.
It shows the before/after of the resource representation on the service, alongside the HTTP request message sent to the service that caused the change.
It also shows the C# code you would write to achieve the change.

### Samples

<details open>
    <summary>Create a new resource</summary>

### C# Code

```csharp
Resource resource = new Resource("abc");
resource.A = "aa";
client.UpdateResource(resource);
```

</details>

<details>
    <summary>Update a top-level property</summary>
</details>

<details>
    <summary>Update a property on a nested model</summary>
</details>

<details>
    <summary>Replace a nested model</summary>
</details>

<details>
    <summary>Update a dictionary value</summary>
</details>

<details>
    <summary>Clear a dictionary</summary>
</details>

<details>
    <summary>Update an array value - primitives</summary>
</details>

<details>
    <summary>Update an array value - objects</summary>
</details>

<details>
    <summary>Update an array using ETags</summary>
</details>
