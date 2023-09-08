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
    <td><b>Request Body: Merge Patch JSON</b></td>
    <td><b>Resource After</b></td>
  </tr>
  <tr>
<td valign="top">

```json
{
}
```

</td>
<td valign="top">

```json
{
  "firstName": "Alice", 
  "lastName": "Smith"
}
```

</td>
<td valign="top">

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
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

User user = new User("123");
user.FirstName = "Alice";
user.LastName = "Smith";
client.UpdateUser(user);
```

#### HTTP traffic

```mermaid
sequenceDiagram
    client->>service: PATCH /users/123
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    Note left of service: { <Resource After> }
```

### Update a top-level property</summary>

### Update a property on a nested model</summary>

### Replace a nested model</summary>

### Update a dictionary value</summary>

### Clear a dictionary</summary>

### Update an array value - primitives</summary>

### Update an array value - objects</summary>

### Update an array using ETags</summary>
