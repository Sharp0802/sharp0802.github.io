---
LCID: en-us
Layout: default
Title: "`GCHandle.AddrOfPinnedObject()` for strings and arrays"
Timestamps:
- 2024-09-24
Topic: [ "C#", ".Net", "GCHandle", "AddrOfPinnedObject", "GCHandle.AddrOfPinnedObject", "string", "System.String", "array" ]
---

For `System.String` and `System.Array`, `GCHandle.AddrOfPinnedObject()` returns address of its buffer.
So, You don't have to use `Encoding.Unicode.GetBytes`, `fixed` statements or `Span<T>` to aquire address of basic types.

How can it be done?
Until .NET Core 2.0, `GCHandle.AddrOfPinnedObject()` is implemented in native-side:

```c++
// Get the address of a pinned object referenced by the supplied pinned
// handle.  This routine assumes the handle is pinned and does not check.
FCIMPL1(LPVOID, MarshalNative::GCHandleInternalAddrOfPinnedObject, OBJECTHANDLE handle)
{
    FCALL_CONTRACT;

    LPVOID p;
    OBJECTREF objRef = ObjectFromHandle(handle);

    if (objRef == NULL)
    {
        p = NULL;
    }
    else
    {
        // Get the interior pointer for the supported pinned types.
        if (objRef->GetMethodTable() == g_pStringClass)
            p = ((*(StringObject **)&objRef))->GetBuffer();
        else if (objRef->GetMethodTable()->IsArray())
            p = (*((ArrayBase**)&objRef))->GetDataPtr();
        else
            p = objRef->GetData();
    }

    return p;
}
FCIMPLEND
```

If object is string or array type,
It returns its buffer's address by calling `GetBuffer()` and `GetDataPtr()`.

In contrast, After .NET Core 3.0, `GCHandle.AddrOfPinnedObject()` is implemented in managed-side:

```csharp
/// <summary>
/// Retrieve the address of an object in a Pinned handle.  This throws
/// an exception if the handle is any type other than Pinned.
/// </summary>
public IntPtr AddrOfPinnedObject()
{
    // Check if the handle was not a pinned handle.
    // You can only get the address of pinned handles.
    IntPtr handle = _handle;
    ThrowIfInvalid(handle);

    if (!IsPinned(handle))
    {
        ThrowHelper.ThrowInvalidOperationException_HandleIsNotPinned();
    }

    // Get the address.

    object target = InternalGet(GetHandleValue(handle));
    if (target is null)
    {
        return default;
    }

    unsafe
    {
        if (RuntimeHelpers.ObjectHasComponentSize(target))
        {
            if (target.GetType() == typeof(string))
            {
                return (IntPtr)Unsafe.AsPointer(ref Unsafe.As<string>(target).GetRawStringData());
            }

            Debug.Assert(target is Array);
            return (IntPtr)Unsafe.AsPointer(ref Unsafe.As<Array>(target).GetRawArrayData());
        }

        return (IntPtr)Unsafe.AsPointer(ref target.GetRawData());
    }
}
```

As described, raw address of string's buffer or array are returned by `String.GetRawStringData()` and `Array.GetRawArrayData`.
