# Example cases for JSON Merge Patch Proposal

The following examples describe the proposed implementation for .NET Patch models for [JSON Merge Patch](https://www.rfc-editor.org/rfc/rfc7396) operations in Azure services.

Each section begins with a C# code sample, followed by an illustration of how the service-side resource would change as a result of the C# code.  These are followed by an expandable diagram of HTTP traffic for completeness.

Most examples are complete, but some are discussions of tricky cases we want to handle in special ways and are open for discussion.  Each section begins with a short description of which purpose they serve.

These examples are part of the larger discussion of .NET Patch models, documented in the following places:

- [JSON Merge Patch arch board issue](https://github.com/Azure/azure-sdk/issues/5966)
- [.NET Patch Models design principles](https://gist.github.com/annelo-msft/ae16eda80b382cc3ae9428954c08e069)

## Examples

<details>
<summary><h3><b>1. Create a new resource</b></h3></summary>

This sample shows a basic example of resource creation with PATCH.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

</details>

```csharp
User user = new User("123");
user.FirstName = "Alice";
user.LastName = "Smith";
client.UpdateUser(user);
```

### Resource state

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

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After> }
```

</details>

</details>

<details>
<summary><h3><b>2. Update a top-level property</b></h3></summary>

This sample shows a basic example of how a user would update a top-level property on a model with PATCH.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

</details>

```csharp
User user = client.GetUser("123");
user.LastName = "Jones";
client.UpdateUser(user);
```

### Resource state

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
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith"
}
```

</td>
<td valign="top">

```json
{
  "lastName": "Jones"
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice", 
-  "lastName": "Smith"
+  "lastName": "Jones"
 } 
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After> }
```

</details>

</details>

<details>
<summary><h3><b>3. Update a property on a nested model</b></h3></summary>

This sample shows how a user would update a property on a child model (`Address`) nested under a parent model (`User`) using PATCH.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last, Address address) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public Address Address { get; set; }
}

public class Address
{
    public Address() { /****/ }
    internal Address(string street, string city, string state, string zip) { /****/ }

    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
}
```

</details>

```csharp
User user = client.GetUser("123");
user.Address.Street = "15010 NE 36th St";
client.UpdateUser(user);
```

### Resource state

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
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "address" : {
    "street": "One Microsoft Way",
    "city": "Redmond",
    "state": "WA",
    "zipCode": "98052"
  }
}
```

</td>
<td valign="top">

```json
{
  "address": {
    "street": "15010 NE 36th St"
  }
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "address" : {
-    "street": "One Microsoft Way",
+    "street": "15010 NE 36th St",
    "city": "Redmond",
    "state": "WA",
    "zipCode": "98052"
  }
}
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After> }
```

</details>

</details>

<details>
<summary><h3><b>4. Replace a nested model</b></h3></summary>

This example illustrates some of the challenges that can arise for users when attempting to replace nested models with PATCH.

Specifically, if a nested model has had a property added in some version (v2 in this example), using an earlier version client (v1 in this example) can result in a "torn write" -- i.e. a situation where the caller intended to fully replace the resource but unintentionally left v2 properties on the resource.

This section illustrates the problem in Example 1, then proposes a solution to mitigate it in Examples 2 and 3.

### C# code -- "torn write", Example 1

<details>
<summary><b>Model definitions</b></summary>

```csharp
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last, Address address) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public Address Address { get; set; }
}

// v1 model
public class Address
{
    public Address() { /****/ }
    internal Address(string street, string city, string state, string zip) { /****/ }

    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
}

// v2 model
public class Address
{
    public Address() { /****/ }
    internal Address(string street, string street2, string city, string state, string zip) { /****/ }

    public string Street { get; set; }

    // Note: Added in v2!
    public string StreetLineTwo { get; set;}

    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
}
```

</details>

```csharp
// v1 client code - results in "torn write" data integrity issue
User user = v1Client.GetUser("123");
user.Address = new Address() {
    Street = "One Microsoft Way",
    City = "Redmond",
    State = "WA",
    ZipCode = "98052"
}

v1Client.UpdateUser(user);
```

### Resource state

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
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "address" : {
    "street": "54 State Street",
    "streetLine2": "Suite 701",
    "city": "Albany",
    "state": "NY",
    "zipCode": "12207"
  }
}
```

</td>
<td valign="top">

```json
{
  "address": {
    "street": "One Microsoft Way",
    "city": "Redmond",
    "state": "WA",
    "zipCode": "98052"
  }
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "address" : {
-    "street": "54 State Street",
+    "street": "One Microsoft Way",
    "streetLine2": "Suite 701",
-    "city": "Albany",
+    "city": "Redmond",
-    "state": "NY",
+    "state": "WA",
-    "zipCode": "12207"
+    "zipCode": "98052"
  }
}
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After> }
```

</details>

### Comments

Note that in the above example, if the `user.Address` property is set to a new model instance, the user might have the intention of overwriting the full `Address` value.  In a forward-compatibility scenario, if they use an earlier client version and a property was added to the `Address` model in a later version, they could end up in a "torn write" state, with compromised data integrity.

To help .NET users who may not have a deep understanding of forward compatibility scenarios, we would like to apply the following principle: _if you would have to send multiple requests to achieve a desired resource state on the service, we will require that you send multiple requests to do this._

In this case, that principle results in the following developer experience.

If a caller tries to overwrite the value of a service resource that could have evolved across versions, the caller can only modify it as follows:

1. They can set it to `null` to delete the resource.
1. If they have retrieved the resource and the model they are holding in-memory confirms that the value is absent or has been deleted, they can set it to a new instance of the model.

Overwriting a nested model that has a non-null value will result in an exception being thrown that warns the caller about possible forward-compatibility data integrity issues.

### C# code - alternate approach, Example 2

```csharp
// v1 client code - "safe" because user doesn't believe they are completely replacing Address
User user = v1Client.GetUser("123");
user.Address.Street = "One Microsoft Way";
v1Client.UpdateUser(user);
```

### C# code - alternate approach, Example 3

```csharp
// v1 client code - "safe" because user is forced to delete the v2 Address completely before making an update
User user = v1Client.GetUser("123");

user.Address = null;
v1Client.UpdateUser(user);

user = v1Client.GetUser("123");
user.Address = new Address() {
    Street = "One Microsoft Way",
    City = "Redmond",
    State = "WA",
    ZipCode = "98052"
}

v1Client.UpdateUser(user);
```

### Resource state - Example 3

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
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "address" : {
    "street": "54 State Street",
    "streetLine2": "Suite 701",
    "city": "Albany",
    "state": "NY",
    "zipCode": "12207"
  }
}
```

</td>
<td valign="top">

```json
{
  "address": null
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
-  "address" : {
-    "street": "54 State Street",
-    "streetLine2": "Suite 701",
-    "city": "Albany",
-    "state": "NY",
-    "zipCode": "12207"
-  }
}
```

</td>
  </tr>
  <tr>
<td valign="top">

```json
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith"
}
```

</td>
<td valign="top">

```json
{
  "address" : {
    "street": "One Microsoft Way",
    "city": "Redmond",
    "state": "WA",
    "zipCode": "98052"
  }
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
+  "address" : {
+    "street": "One Microsoft Way",
+    "city": "Redmond",
+    "state": "WA",
+    "zipCode": "98052"
  }
}
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before - 1> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body - 1> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After - 1> }
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before - 2> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body - 2> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After - 2> }
```

</details>

</details>

<details>
<summary><h3><b>5. Update an array value - primitives</b></h3></summary>

This sample illustrates our proposal for array updates, and discusses the rationale for this approach in the **Comments** section below.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp
using System.Net.Http;

public class User
{
    public User(string id) { /****/ }
    internal User(string id, ETag eTag, string first, string last, IList<string> pets) { /****/ }

    public string Id { get; }
    public ETag ETag { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public IList<string> Pets { get; }
}
```

</details>

```csharp
using System.Net.Http;

Response<User> response;

do 
{
    User user = client.GetUser("123");
    user.Pets.Add("rizzo");

    response = client.UpdateUser(user, onlyIfUnchanged: true);
}
while (response.Status == HttpStatusCode.PreconditionFailed);
```

### Resource state

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
  "id": "123",
  "ETag": "abc",
  "firstName": "Alice",
  "lastName": "Smith",
  "pets": [
    "statler",
    "waldorf"
  ]
}
```

</td>
<td valign="top">

```txt
If-Match: "abc"
```

```json
{
  "pets": [
    "statler",
    "waldorf",
    "rizzo"
  ]
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
-  "ETag": "abc",
+  "ETag": "def",
  "firstName": "Alice", 
  "lastName": "Smith",
- "pets": [
-   "statler",
-   "waldorf"
+ ]
+ "pets": [
+   "statler",
+   "waldorf",
+   "rizzo"
+ ]
 } 
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: ETag="abc"<br>{ <Resource Before> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: If-Match="abc"<br>{ <Request Body> }
    service->>client: 412 Precondition Failed
    deactivate service
    Note left of service: ETag="xyz"
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: ETag="xyz"<br>{ <Modified Resource> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: If-Match="xyz"<br>{ <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: ETag="def"<br>{ <Resource After> }
```

</details>

### Comments

To help .NET users who may not have a deep understanding of the full details of the [JSON Merge Patch RFC](https://www.rfc-editor.org/rfc/rfc7396), we would like to apply the following principle: _if we need to send values in the Patch request body that the user did not explicitly modify in their application code, we should prevent accidental data loss from the result of sending these values._  Users should either use conditional requests to prevent unintentionally overwriting data on the server, or use a protocol method instead of a convenience method and hand-author a request body.  Either of these interventions lets the user opt-in to the responsibility of handling nuances of the JSON Merge Patch RFC themselves, and promotes transparency about what the client is sending.

The reason we do this is to say: “if you use APIs that are easy, what we do will be easy to understand; if you want to do something that might be surprising, we need you to grok the RFC, so the complexity of the API will indicate that you need to do more work to understand what’s going on.”

In this instance, these principle results in the following developer experience.

If a caller modifies an array value in a Patch model and doesn't send an ETag in the update request, we will throw an exception with one of the following messages:

1. If the service supports conditional requests, the message will direct the user to set the `If-Match` header on the PATCH request.  In the example above, this is accomplished by retrieving the resource value before updating it or setting the ETag property on the model manually.  Setting the optional `onlyIfUnchanged` parameter in the `UpdateUser` method adds the value of the ETag property from the model to the `If-Match` header on the request.  The sample C# code for this is shown above.
1. If the service does not support conditional requests, the message will instruct the user to compose the PATCH JSON payload by hand and send it using the corresponding protocol method.  In this case, no `onlyIfUnchanged` parameter will be in the method signature of the update method, since the service does not support it.  The sample C# code for this is shown below.

Note that if the array value in the `User` model is not modified, no exception will be thrown from the update method because the client did not try to send values that the user did not modify.

#### C# code - when service does not support conditional requests

```csharp
User user = client.GetUser("123");

var patch = new {
    "pets" = new string[] {
        "statler",
        "waldorf",
        "rizzo"
    }
}

response = client.UpdateUser(user.Id, patch);
```

#### References

For further details of conditional requests, see:

- [If-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Match)
- [Avoiding mid-air collisions](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/412#avoiding_mid-air_collisions)

</details>

<details>
<summary><h3><b>6. Update an array value - objects</b></h3></summary>

This sample closely mirrors the **Update an array value - primitives** example above, but illustrates the implications of resources that hold JSON arrays of objects.  Specifically, we want to call out the increased payload size to make a small change to a single property of an object held in an array when using JSON Merge Patch, i.e. the entire array must be sent to make this change.

The models in this sample are simplifications of resources used by the ACS JobRouter service in its [Upsert Job](https://learn.microsoft.com/rest/api/communication/jobrouter/job-router/upsert-job?tabs=HTTP) operation.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp

public class RouterJob
{
    public RouterJob(string id) { /****/ }
    internal RouterJob(string id, string channelId, int priority, List<RouterWorkerSelector> selectors) { /****/ }

    public string Id { get; }
    public ETag ETag { get; set; }
    public string ChannelId { get; set; }
    public int Priority { get; set; }
    public IList<RouterWorkerSelector> Selectors { get; }
}

public class RouterWorkerSelector
{
    public RouterWorkerSelector() { /****/ }
    internal RouterWorkerSelector(string key, bool expedite) { /****/ }

    public string Key { get; set; }
    public bool Expedite { get; set; }
}
```

</details>

```csharp
using System.Net.Http;

Response<RouterJob> response;

do 
{
    RouterJob job = client.GetJob("123");
    job.Selectors[0].Expedite = true;

    response = client.UpdateJob(job, onlyIfUnchanged: true);
}
while (response.Status == HttpStatusCode.PreconditionFailed);
```

### Resource state

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
  "id": "123",
  "ETag": "abc",
  "channelId": "ChatChannel",
  "priority": "2",
  "selectors": [
    {
        "key": "A",
        "expedite": false
    },
    {
        "key": "B",
        "expedite": false
    },
    {
        "key": "C",
        "expedite": false
    }
  ]
}
```

</td>
<td valign="top">

```txt
If-Match: "abc"
```

```json
{
  "selectors": [
    {
      "key": "A",
      "expedite": true
    },
    {
      "key": "B",
      "expedite": false
    },
    {
      "key": "C",
      "expedite": false
    }
  ]
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
-  "ETag": "abc",
+  "ETag": "def",
  "channelId": "ChatChannel",
  "priority": "2",
  "selectors": [
    {
        "key": "A",
-       "expedite": false
+       "expedite": true
    },
    {
        "key": "B",
        "expedite": false
    },
    {
        "key": "C",
        "expedite": false
    }
  ]
}
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /jobs/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: ETag="abc"<br>{ <Resource Before> }
    client->>service: PATCH /jobs/123
    activate service
    Note right of client: If-Match="abc"<br>{ <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: ETag="def"<br>{ <Resource After> }
```

</details>

### Comments

Please see **Comments** section in **Update an array value - primitives**](#update-an-array-value---primitives)** section above.

</details>

<details>
<summary><h3><b>7. Update values in a dictionary</b></h3></summary>

This sample shows a basic example of how a user would update a dictionary value with PATCH.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last, Dictionary<string, string> petTypes) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public IDictionary<string, string> PetTypes { get; }
}
```

</details>

```csharp
User user = client.GetUser("123");

// Update existing value
user.PetTypes["statler"] = "dog";

// Add new
user.PetTypes["rizzo"] = "rat";

client.UpdateUser(user);
```

### Resource state

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
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "petTypes" : {
    "statler": "cat",
    "waldorf": "dog"
  }
}
```

</td>
<td valign="top">

```json
{
  "petTypes" : {
    "statler": "dog",
    "rizzo": "rat"
  }
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "petTypes" : {
-   "statler": "cat",
+   "statler": "dog",
    "waldorf": "dog",
+   "rizzo": "rat",
  }
}
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After> }
```

</details>

</details>

</details>

<details>
<summary><h3><b>8. Clear a dictionary</b></h3></summary>

This example illustrates how we would handle clearing a dictionary.  In this case, the dictionary is found on a service resource that has a property that is a JSON object with unnamed key-value pairs, which we would model as an `IDictionary` property in a .NET model.

We discuss some challenges involved with this approach and propose a solution in the **Comments** section after the example.

### C# code

<details>
<summary><b>Model definitions</b></summary>

```csharp
public class User
{
    public User(string id) { /****/ }
    internal User(string id, string first, string last, Dictionary<string, string> petTypes) { /****/ }

    public string Id { get; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public IDictionary<string, string> PetTypes { get; }
}
```

</details>

```csharp

User user = client.GetUser("123");

// Clear the dictionary
user.PetTypes.Clear();

// Add new item to empty dictionary
user.PetTypes["rizzo"] = "rat";

client.UpdateUser(user);
```

### Resource state

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
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "petTypes" : {
    "statler": "dog",
    "waldorf": "dog"
  }
}
```

</td>
<td valign="top">

```json
{
  "petTypes" : {
    "statler": null,
    "waldorf": null,
    "rizzo": "rat"
  }
}
```

</td>
<td valign="top">

```diff
{
  "id": "123",
  "firstName": "Alice",
  "lastName": "Smith",
  "petTypes" : {
-   "statler": "dog",
-   "waldorf": "dog",
+   "rizzo": "rat",
  }
}
```

</td>
  </tr>
</table>

<details>
<summary><b>HTTP traffic</b></summary>

(Please click the `<->` icon to see the diagram rendered correctly.)

```mermaid
sequenceDiagram
    client->>service: GET /users/123
    activate service
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource Before> }
    client->>service: PATCH /users/123
    activate service
    Note right of client: { <Request Body> }
    service->>client: 200 OK
    deactivate service
    Note left of service: { <Resource After> }
```

</details>

### Comments

Similar to the forward-compatibility issues described in example **Replace a nested model**, if a user wants to clear a dictionary, they need to ensure that every item in the dictionary is deleted.  If the dictionary has been modified since their last GET, there may be values in the dictionary that would not get deleted.

We can address this by requiring the use of ETags with a `Dictionary.Clear` operation as described in the **Update an array value - primitives** example below.  Alternatively, we could allow users to do this under the principle that _users should know that someone else may have updated a resource since they last retrieved it and so they should use "update without ETags" at their own risk_.  This is the principle that guides us not to require ETags for updates to primitive properties.  However, since a user may not understand how our models have implemented `Dictionary.Clear`, it would be safer to help users prevent data integrity issues by requiring them to either use an ETag or work at the protocol level as described in the **Update an array value - primitives** example below.

</details>
