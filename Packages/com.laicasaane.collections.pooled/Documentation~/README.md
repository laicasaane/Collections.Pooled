# Collections.Pooled

- Inspired by [jtmueller/Collections.Pooled](https://github.com/jtmueller/Collections.Pooled)
- The source code of `List`, `Queue`, `Stack`, `HashSet`, `Dictionary` are copied from [dotnet/runtime](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/), along with other utilities to accomodate their functionality


# Installation

## Install via Open UPM

You can install this package from the [Open UPM](https://openupm.com/packages/com.laicasaane.collections.pooled/) registry.

More details [here](https://github.com/openupm/openupm-cli#installation).

```
openupm add com.laicasaane.collections.pooled
```


## Install via Package Manager

Or, you can add this package by opening the **Package Manager** window and entering

```
https://github.com/laicasaane/Collections.Pooled.git?path=Packages/com.laicasaane.collections.pooled
```

from the `Add package from git URL` option.


# Collections.Pooled

## 1. Class-based collections

- `List<T>`
- `Queue<T>`
- `Stack<T>`
- `HashSet<T>`
- `Dictionary<TKey, TValue>`

Their functionality is the same as provided by the standard collections in `System.Collections.Generic` namespace.

There are also some additional APIs for high performant scenarios.

Designed as quick drop-in replacements for their standard counterparts.

## 2. Struct-based collections

- `ValueArray<T>`
- `ValueList<T>`
- `ValueQueue<T>`
- `ValueStack<T>`
- `ValueHashSet<T>`
- `ValueDictionary<TKey, TValue>`

Functionally the same as their class-based counterparts.

Designed as `struct`s to reduce GC allocation of the collection itself.

:warning: Must be created using the static `Create` methods.

## 3. Temporary struct-based collections

- `TempArray<T>`
- `TempList<T>`
- `TempQueue<T>`
- `TempStack<T>`
- `TempHashSet<T>`
- `TempDictionary<TKey, TValue>`

Functionally the same as their struct-based counterparts.

Designed as `ref struct`s to prevent unintentional GC allocations.

:warning: Must be created using the static `Create` methods.

## Usage

At the end of their lifetime, the collections should be `Dispose`d in order to return their internal arrays to the pool.

```cs
using Collections.Pooled.Generic;

void ManualDisposing()
{
    var lst = new List<int>();
    using var arr = ValueArray<int>.Create(10);
    ...
    lst.Dispose();
    arr.Dispose();
}

void AutomaticDisposing()
{
    using var lst = new List<int>();
    using var arr = TempArray<int>.Create(10);
    ...
} // at the end of this method, `lst` and `arr` will be disposed automatically
```

If the lifetime of the collections is extended, they should be disposed through the use of `IDisposable` interface.

```cs
using System;
using Collections.Pooled.Generic;

class ExtendedLifetime : IDisposable
{
    // The lifetime of these collections is extended
    private Queue<int> _queue = new Queue<int>();
    private ValueDictionary<int, string> _dict = new ValueDictionary<int, string>()

    public void Dispose()
    {
        _queue.Dispose();
        _dict.Dispose();
    }
}

void UseExtendedLifetime()
{
    using var xyz = new ExtendedLifetime();
    ...
} // at the end of this method, `xyz` will be disposed automatically, along with its internal collections
```

# Internals APIs

To enable the scenario of developing high performant extension modules, this library allows direct access to the internal fields of each collection.

## Internals data structures

- Collections.Pooled.Generic
  - `*Internals`
  - `*InternalsRef`
  - `*InternalsRefUnsafe`

- Collections.Pooled.Generic.StructBased
  - `Value*Internals`
  - `Value*InternalsRef`
  - `Value*InternalsRefUnsafe`

- Collections.Pooled.Generic.Temporary
  - `Temp*Internals`
  - `Temp*InternalsRef`
  - `Temp*InternalsRefUnsafe`

## Safe context

Safe internals accessing APIs are exposed through the static `*CollectionsInternals` classes.

### `TakeOwnership` methods

```cs
CollectionInternals      .TakeOwnership(collection)
ValueCollectionInternals .TakeOwnership(valueCollection)
TempCollectionInternals  .TakeOwnership(tempCollection)
```

The procedure follows these steps:

1. Creates an `*Internals` structure to hold references to internal arrays of a source collection.

2. Removes the source references by assigning them to `null`.

3. `Dispose` the source collection.

4. Returns that structure to the outside.

:arrow_forward: Consequentially, this structure has overtaken the **ownership** of those internal arrays.

:x: **DO NOT** use the source collection after this procedure to avoid unknown behaviours, because it has already been disposed.

```cs
using Collections.Pooled.Generic;
using Collections.Pooled.Generic.Internals;

void InternalsDisposing()
{
    using var list = new List<int>();
    using var internals = CollectionInternals.TakeOwnership(list);
    // `list` has already been disposed
    // DO NOT use `list` after this line
    ...
} // at the end of this method, `internals` will be disposed automatically
```

Sometimes it is desired for `*Internals` structures to not be disposed because the ownership is going to be transferred to other places. In that case, do not use `using` or call `Dispose` on the returned structure.

```cs
using Collections.Pooled.Generic;
using Collections.Pooled.Generic.Internals;

void Internals_Ownership_Transferring()
{
    using var list = new List<int>();
    var internals = CollectionInternals.TakeOwnership(list);
    var customList = SomeCustomList<int>.From(internals);
    // `customList` is holding ownership of the internals of `list`
    // no need to declare `using var internals = ...`
    // no need to call `internals.Dispose()`
    ...
} // at the end of this method, `internals` WON'T be disposed automatically
```

### `GetRef` methods

```cs
CollectionInternals      .GetRef(collection)
ValueCollectionInternals .GetRef(valueCollection)
TempCollectionInternals  .GetRef(tempCollection)
```

- This method returns an `*InternalsRef` structure that holds **indirect** references to internal arrays of a source collection.

- The indirect references are represented as `ReadOnlySpan`s to only expose high performant read operations.

- `*InternalsRef`s are designed as `ref struct`s to prevent unintentional lingering effect of the references (could cause memory leaks).

:warning: The source collection is **NOT** disposed, it still holds the ownership of its internal arrays.

```cs
using Collections.Pooled.Generic;
using Collections.Pooled.Generic.Internals;

void InternalsDisposing()
{
    using var list = new List<int>();
    var internalsRef = CollectionInternals.GetRef(list);
    // `list` is NOT disposed
    // `list` is USABLE even after this line
    ...
}
```

### `AsReadOnlySpan` methods

```cs
CollectionInternals      .AsReadOnlySpan(collection)
ValueCollectionInternals .AsReadOnlySpan(valueCollection)
TempCollectionInternals  .AsReadOnlySpan(tempCollection)
```

- This method returns an indirect reference to the internal main array of a source collection.

- The indirect references are represented as `ReadOnlySpan` to only expose read operations.

- For `Dictionary` and `HashSet`, it returns the `_entries` array.

- For other collections, it returns `_items` or `_array`.

## Unsafe context

Unsafe internals accessing APIs are exposed through the static `*CollectionInternalsUnsafe` classes.

### `GetRef` methods

```cs
CollectionInternalsUnsafe      .GetRef(collection)
ValueCollectionInternalsUnsafe .GetRef(valuecollection)
TempCollectionInternalsUnsafe  .GetRef(tempCollection)
```

- This method returns an `*InternalsRefUnsafe` structure that holds **indirect** references to internal arrays of a source collection.

- The indirect references are represented as `Span`s to expose high performant read and write operations.

- `*InternalsRefUnsafe`s are designed as `ref struct`s to prevent unintentional lingering effect of the references (could cause memory leaks).

:warning: The source collection is **NOT** disposed, it still holds the ownership of its internal arrays.

```cs
using Collections.Pooled.Generic;
using Collections.Pooled.Generic.Internals.Unsafe;

void InternalsDisposing()
{
    using var list = new List<int>();
    var internalsRefUnsafe = CollectionInternalsUnsafe.GetRef(list);
    // `list` is NOT disposed
    // `list` is USABLE even after this line
    ...
}
```

### `AsSpan` methods

```cs
CollectionInternals      .AsSpan(collection)
ValueCollectionInternals .AsSpan(valueCollection)
TempCollectionInternals  .AsSpan(tempCollection)
```

- This method returns an **indirect** reference to the internal main array of a source collection.

- The indirect references are represented as `Span` to expose read and write operations.

- For `Dictionary` and `HashSet`, it returns the `_entries` array.

- For other collections, it returns `_items` or `_array`.

### `GetUnsafe` methods

```cs
collection      .GetUnsafe(out array, out count)
valueCollection .GetUnsafe(out array, out count)
tempCollection  .GetUnsafe(out array, out count)
```

- This method returns a **direct** reference to the internal main array of a source collection, along with the size of that array (named `count`).

- For `Dictionary` and `HashSet`, it returns the `_entries` array.

- For other collections, it returns `_items` or `_array`.

- For `Queue`, it also returns `head` and `tail` values.

```cs
queue      .GetUnsafe(out array, out count, out head, out tail)
valueQueue .GetUnsafe(out array, out count, out head, out tail)
tempQueue  .GetUnsafe(out array, out count, out head, out tail)
```

