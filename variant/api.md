# Azure.Variant

`Azure.Variant` is an implementation of a [tagged union](https://en.wikipedia.org/wiki/Tagged_union).  It provides a type we can use for both primitives and value types to avoid boxing .NET intrinsic types.  For example, in libraries like KeyVault mentioned below, it could be used to hold both a string and an int without boxing the int.

- [Tracking issue]( https://github.com/Azure/azure-sdk-for-net/issues/32978)
- [Original API proposal](https://github.com/dotnet/runtime/issues/28882)

## How does it work?

As a principle, it behaves much like `object`, in the sense that you can assign a value of any type to it.  The major difference is that it doesn't box most intrinsic value types.

<details>
<summary>Value types that Variant won't box</summary>

- `byte`
- `byte?`
- `sbyte`
- `sbyte?`
- `bool`
- `bool?`
- `char`
- `char?`
- `short`
- `short?`
- `int`
- `int?`
- `long`
- `long?`
- `ushort`
- `ushort?`
- `uint`
- `uint?`
- `ulong`
- `ulong?`
- `float`
- `float?`
- `double`
- `double?`
- `DateTimeOffset`
- `DateTimeOffset?`
- `DateTime`
- `DateTime?`
- Enums

</details>

Because it's not `object`, you do need to use a different set of APIs to get the same functionality in some cases.  For example:

### Assign a value to Variant

Variant has implicit cast operators for each of the primitives listed above.  So, you can simply assign a value of those types to Variant, and it will create a new Variant instance holding an unboxed copy of the value.

```csharp
Variant v = 3L;

// v.Type is System.Int64
```

Since you can't have an implicit cast operator from `object`, for reference types, you have to create a new instance of `Variant` passing the value to Variant's constructor.

```csharp
MemoryStream s = new();
Variant v = new(s);

// v.Type is System.IO.MemoryStream
```

Variant will box value types that aren't on the list above, because it only stores 16 bytes and a user-defined value type could be arbitrarily large.  The one exception is for enums, which it will hold without boxing.  Because we can't add implicit cast operators for unknown enum types, you have to call the `Create` method to create a Variant that holds an enum without boxing.

```csharp
Value v = Value.Create(System.Color.Blue);

// v.Type is System.Color
```

### Get the value from Variant

Variant has explicit cast operators for each of the primitives listed above, as well as to `string`.  That means you can assign a Variant holding one of these types to a variable of that type by casting it:

```csharp
Variant v = 3;
int i = (int)v;
```

If you don't know the type of the value that a Variant is holding, you can read that from Variant's `Type` property.

```csharp
switch (v.Type)
{
    case Type s when s == typeof(string):
        Console.WriteLine($"string: {v}");
        break;
    case Type i when i == typeof(int):
        Console.WriteLine($"int: {(int)v}");
        break;
}
```

If you already know the type the Variant is holding, you can use the `As<T>` method to get its value as that type.

```csharp
MemoryStream s = new();
Variant v = new (s);

MemoryStream streamValue = v.As<MemoryStream>();
```

Of course, if the Variant isn't holding a value of the type that you ask for, it will throw an `InvalidCastException`.  If you'd prefer to retrieve the value without risking an exception being thrown, you can use the `TryGetValue` method instead.

```csharp
if (v.TryGetValue(out string s))
{
    Console.WriteLine($"string: {s}");
}

if (v.TryGetValue(out int i))
{
    Console.WriteLine($"int: {i}");
}
```

### Handling nulls

### Numeric conversions and allowable casts