# Example cases for JSON Merge Patch Proposal

The following illustrates how our implementation of .NET Patch models for JSON Merge Patch would affect resources on the service side.
It shows the before/after of the resource representation on the service, alongside the HTTP request message sent to the service that caused the change.
It also shows the C# code you would write to achieve the change.

## Samples

### Create a new resource</summary>

#### Resource state

<table>
  <tr>
    <td><b>Resource Before</b></td>
    <td><b>PATCH Body</b></td>
    <td><b>Resource After</b></td>
  </tr>
  <tr>
<td>

```json
{
}
```

</td>
<td>

```json
{
  "firstName": "Alice", 
  "lastName": "Smith"
}
```

</td>
<td>

```diff
{
+  "id": "123",
+  "firstName": "Alice", 
+  "lastName": "Smith"
 } 
```

</td>
  </tr>
</table>

#### C# code

```csharp
User user = new User("123");
user.FirstName = "Alice";
user.LastName = "Smith";
client.UpdateUser(user);
```

#### HTTP traffic

```mermaid
sequenceDiagram
    participant client
    participant service
    client->>server: PATCH /123
    Note right of client: { /** see above **/ }
    server-->>client: HTTP/1.1 200
```

### Update a top-level property</summary>

### Update a property on a nested model</summary>

### Replace a nested model</summary>

### Update a dictionary value</summary>

### Clear a dictionary</summary>

### Update an array value - primitives</summary>

### Update an array value - objects</summary>

### Update an array using ETags</summary>
