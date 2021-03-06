﻿# Function Pointers

## Summary

This proposal provides language constructs that expose IL opcodes that cannot currently be accessed efficiently,
or at all, in C# today: `ldftn` and `calli`. These IL opcodes can be important in high performance code and developers
need an efficient way to access them.

## Motivation

The motivations and background for this feature are described in the following issue (as is a
potential implementation of the feature):

https://github.com/dotnet/csharplang/issues/191

This is an alternate design proposal to [compiler intrinsics](https://github.com/dotnet/csharplang/blob/master/proposals/intrinsics.md)

## Detailed Design

### Function pointers

The language will allow for the declaration of function pointers using the `delegate*` syntax. The full syntax is described
in detail in the next section but it is meant to resemble the syntax used by `Func` and `Action` type declarations.

``` csharp
unsafe class Example {
    void Example(Action<int> a, delegate*<int, void> f) {
        a(42);
        f(42);
    }
}
```

These types are represented using the function pointer type as outlined in ECMA-335. This means invocation
of a `delegate*` will use `calli` where invocation of a `delegate` will use `callvirt` on the `Invoke` method.
Syntactically though invocation is identical for both constructs.

The ECMA-335 definition of method pointers includes the calling convention as part of the type signature (section 7.1).
The default calling convention will be `managed`. Alternate forms can be specified by adding the appropriate modifier
after the `delegate*` syntax: `managed`, `cdecl`, `stdcall`, `thiscall`, or `unmanaged`. Example:

``` csharp
// This method will be invoked using the cdecl calling convention
delegate* cdecl<int, int>;

// This method will be invoked using the stdcall calling convention
delegate* stdcall<int, int>;
```

Conversions between `delegate*` types is done based on their signature including the calling convention.

``` csharp
unsafe class Example {
    void Conversions() {
        delegate*<int, int, int> p1 = ...;
        delegate* managed<int, int, int> p2 = ...;
        delegate* cdecl<int, int, int> p3 = ...;

        p1 = p2; // okay p1 and p2 have compatible signatures
        Console.WriteLine(p2 == p1); // True
        p2 = p3; // error: calling conventions are incompatible
    }
}
```

A `delegate*` type is a pointer type which means it has all of the capabilities and restrictions of a standard pointer
type:

- Only valid in an `unsafe` context.
- Methods which contain a `delegate*` parameter or return type can only be called from an `unsafe` context.
- Cannot be converted to `object`.
- Cannot be used as a generic argument.
- Can implicitly convert `delegate*` to `void*`.
- Can explicitly convert from `void*` to `delegate*`.

Restrictions:

- Custom attributes cannot be applied to a `delegate*` or any of its elements.
- A `delegate*` parameter cannot be marked as `params`
- A `delegate*` type has all of the restrictions of a normal pointer type.

### Function pointer syntax

The full function pointer syntax is represented by the following grammar:

```antlr
pointer_type
    : ...
    | funcptr_type
    ;

funcptr_type
    : 'delegate' '*' calling_convention? '<' (funcptr_parameter_modifier? type ',')* funcptr_return_modifier? return_type '>'
    ;

calling_convention
    : 'cdecl'
    | 'managed'
    | 'stdcall'
    | 'thiscall'
    | 'unmanaged'
    ;

funcptr_parameter_modifier
    : 'ref'
    | 'out'
    | 'in'
    ;

funcptr_return_modifier
    : 'ref'
    | 'ref readonly'
    ;
```

The `unmanaged` calling convention represents the default calling convention for native code on the current platform, and is encoded as winapi.
All `calling_convention`s are contextual keywords when preceded by a `delegate*`.

``` csharp
delegate int Func1(string s);
delegate Func1 Func2(Func1 f);

// Function pointer equivalent without calling convention
delegate*<string, int>;
delegate*<delegate*<string, int>, delegate*<string, int>>;

// Function pointer equivalent with calling convention
delegate* managed<string, int>;
delegate*<delegate* managed<string, int>, delegate*<string, int>>;
```

### Function pointer conversions

In an unsafe context, the set of available implicit conversions (Implicit conversions) is extended to include the following implicit pointer conversions:
- [_Existing conversions_](https://github.com/dotnet/csharplang/blob/master/spec/unsafe-code.md#pointer-conversions)
- From _funcptr\_type_ `F0` to another _funcptr\_type_ `F1`, provided all of the following are true:
    - `F0` and `F1` have the same number of parameters, and each parameter `D0n` in `F0` has the same `ref`, `out`, or `in` modifiers as the corresponding parameter `D1n` in `F1`.
    - For each value parameter (a parameter with no `ref`, `out`, or `in` modifier), an identity conversion, implicit reference conversion, or implicit pointer conversion exists from the parameter type in `F0` to the corresponding parameter type in `F1`.
    - For each `ref`, `out`, or `in` parameter, the parameter type in `F0` is the same as the corresponding parameter type in `F1`.
    - If the return type is by value (no `ref` or `ref readonly`), an identity, implicit reference, or implicit pointer conversion exists from the return type of `F1` to the return type of `F0`.
    - If the return type is by reference (`ref` or `ref readonly`), the return type and `ref` modifiers of `F1` are the same as the return type and `ref` modifiers of `F0`.
    - The calling convention of `F0` is the same as the calling convention of `F1`.

### Allow address-of to target methods

Method groups will now be allowed as arguments to an address-of expression. The type of such an
expression will be a `delegate*` which has the equivalent signature of the target method and a managed
calling convention:

``` csharp
unsafe class Util {
    public static void Log() { }

    void Use() {
        delegate*<void> ptr1 = &Util.Log;

        // Error: type "delegate*<void>" not compatible with "delegate*<int>";
        delegate*<int> ptr2 = &Util.Log;

        // Okay. Conversion to void* is always allowed.
        void* v = &Util.Log;
   }
}
```

In an unsafe context, a method `M` is compatible with a function pointer type `F` if all of the following are true:
- `M` and `F` have the same number of parameters, and each parameter in `D` has the same `ref`, `out`, or `in` modifiers as the corresponding parameter in `F`.
- For each value parameter (a parameter with no `ref`, `out`, or `in` modifier), an identity conversion, implicit reference conversion, or implicit pointer conversion exists from the parameter type in `M` to the corresponding parameter type in `F`.
- For each `ref`, `out`, or `in` parameter, the parameter type in `M` is the same as the corresponding parameter type in `F`.
- If the return type is by value (no `ref` or `ref readonly`), an identity, implicit reference, or implicit pointer conversion exists from the return type of `F` to the return type of `M`.
- If the return type is by reference (`ref` or `ref readonly`), the return type and `ref` modifiers of `F` are the same as the return type and `ref` modifiers of `M`.
- The calling convention of `M` is the same as the calling convention of `F`.
- `M` is a static method.

In an unsafe context, an implicit conversion exists from an address-of expression whose target is a method group `E` to a compatible function pointer type `F` if `E` contains at least one method that is applicable in its normal form to an argument list constructed by use of the parameter types and modifiers of `F`, as described in the following.
- A single method `M` is selected corresponding to a method invocation of the form `E(A)` with the following modifications:
   - The arguments list `A` is a list of expressions, each classified as a variable and with the type and modifier (`ref`, `out`, or `in`) of the corresponding _formal\_parameter\_list_ of `D`.
   - The candidate methods are only those methods that are applicable in their normal form, not those applicable in their expanded form.
   - The candidate methods are only those methods that are static.
- If the algorithm of Method invocations produces an error, then a compile-time error occurs. Otherwise, the algorithm produces a single best method `M` having the same number of parameters as `F` and the conversion is considered to exist.
- The selected method `M` must be compatible (as defined above) with the function pointer type `F`. Otherwise, a compile-time error occurs.
- The result of the conversion is a function pointer of type `F`.

An implicit conversion exists from an address-of expression whose target is a method group `E` to `void*` if there is only one static method `M` in `E`.
If there is one static method, then the single best method from `E` is `M`.
Otherwise, a compile-time error occurs.

This means developers can depend on overload resolution rules to work in conjunction with the
address-of operator:

``` csharp
unsafe class Util {
    public static void Log() { }
    public static void Log(string p1) { }
    public static void Log(int i) { };

    void Use() {
        delegate*<void> a1 = &Log; // Log()
        delegate*<int, void> a2 = &Log; // Log(int i)

        // Error: ambiguous conversion from method group Log to "void*"
        void* v = &Log;
    }
```

The address-of operator will be implemented using the `ldftn` instruction.

Restrictions of this feature:

- Only applies to methods marked as `static`.
- Non-`static` local functions cannot be used in `&`. The implementation details of these methods are
  deliberately not specified by the language. This includes whether they are static vs. instance or
  exactly what signature they are emitted with.


### Operators on Function Pointer Types

The section in unsafe code on operators is modified as such:

> In an unsafe context, several constructs are available for operating on all _pointer\_type_s that are not _funcptr\_type_s:
>
> *  The `*` operator may be used to perform pointer indirection ([Pointer indirection](unsafe-code.md#pointer-indirection)).
> *  The `->` operator may be used to access a member of a struct through a pointer ([Pointer member access](unsafe-code.md#pointer-member-access)).
> *  The `[]` operator may be used to index a pointer ([Pointer element access](unsafe-code.md#pointer-element-access)).
> *  The `&` operator may be used to obtain the address of a variable ([The address-of operator](unsafe-code.md#the-address-of-operator)).
> *  The `++` and `--` operators may be used to increment and decrement pointers ([Pointer increment and decrement](unsafe-code.md#pointer-increment-and-decrement)).
> *  The `+` and `-` operators may be used to perform pointer arithmetic ([Pointer arithmetic](unsafe-code.md#pointer-arithmetic)).
> *  The `==`, `!=`, `<`, `>`, `<=`, and `=>` operators may be used to compare pointers ([Pointer comparison](unsafe-code.md#pointer-comparison)).
> *  The `stackalloc` operator may be used to allocate memory from the call stack ([Fixed size buffers](unsafe-code.md#fixed-size-buffers)).
> *  The `fixed` statement may be used to temporarily fix a variable so its address can be obtained ([The fixed statement](unsafe-code.md#the-fixed-statement)).
> 
> In an unsafe context, several constructs are available for operating on all _funcptr\_type_s:
> *  The `&` operator may be used to obtain the address of static methods ([Allow address-of to target methods](function-pointers.md#allow-address-of-to-target-methods))
> *  The `==`, `!=`, `<`, `>`, `<=`, and `=>` operators may be used to compare pointers ([Pointer comparison](unsafe-code.md#pointer-comparison)).

Additionally, we modify all the sections in `Pointers in expressions` to forbid function pointer types, except `Pointer comparison` and `The sizeof operator`.

### Better function member

The better function member specification will be changed to include the following line:

> A `delegate*` is more specific than `void*`

This means that it is possible to overload on `void*` and a `delegate*` and still sensibly use the address-of operator.

## Metadata representation of `in`, `out`, and `ref readonly` parameters and return types

Function pointer signatures have no parameter flags location, so we must encode whether parameters and the return type are `in`, `out`, or `ref readonly` by using modreqs.

### `in`

We reuse `System.Runtime.InteropServices.InAttribute`, applied as a `modreq` to the ref specifier on a parameter or return type, to mean the following:
* If applied to a parameter ref specifier, this parameter is treated as `in`.
* If applied to the return type ref specifier, the return type is treated as `ref readonly`.

### `out`

We use `System.Runtime.InteropServices.OutAttribute`, applied as a `modreq` to the ref specifier on a parameter type, to mean that the parameter is an `out` parameter.

### Errors

* It is an error to apply `OutAttribute` as a modreq to a return type.
* It is an error to apply both `InAttribute` and `OutAttribute` as a modreq to a parameter type.
* If either are specified via modopt, they are ignored.

## Open Issues

### NativeCallableAttribute

This is an attribute used by the CLR to avoid the managed to native prologue when invoking. Methods marked by this
attribute are only callable from native code, not managed (can’t call methods, create a delegate, etc …). The attribute
is not special to mscorlib; the runtime will treat any attribute with this name with the same semantics.

It's possible for the runtime and language to work together to fully support this. The language could choose to treat
address-of `static` members with a `NativeCallable` attribute as a `delegate*` with the specified calling convention.

``` csharp
unsafe class NativeCallableExample {
    [NativeCallable(CallingConvention.CDecl)]
    static void CloseHandle(IntPtr p) => Marshal.FreeHGlobal(p);

    void Use() {
        delegate*<IntPtr, void> p1 = &CloseHandle; // Error: Invalid calling convention

        delegate* cdecl<IntPtr, void> p2 = &CloseHandle; // Okay
    }
}

```

Additionally the language would likely also want to:

- Flag any managed calls to a method tagged with `NativeCallable` as an error. Given the function can't be invoked from
managed code the compiler should prevent developers from attempting such an invocation.
- Prevent method group conversions to `delegate` when the method is tagged with `NativeCallable`.

This is not necessary to support `NativeCallable` though. The compiler can support the `NativeCallable` attribute as is
using the existing syntax. The program would simply need to cast to `void*` before casting to the correct `delegate*`
signature. That would be no worse than the support today.

``` csharp
void* v = &CloseHandle;
delegate* cdecl<IntPtr, bool> f1 = (delegate* cdecl<IntPtr, bool>)v;
```

### Extensible set of unmanaged calling conventions

The set of unmanaged calling conventions supported by the current ECMA-335 encodings is outdated. We have seen requests to add support
for more unmanaged calling conventions, for example:

- [vectorcall](https://docs.microsoft.com/cpp/cpp/vectorcall) https://github.com/dotnet/coreclr/issues/12120
- StdCall with explicit this https://github.com/dotnet/coreclr/pull/23974#issuecomment-482991750

The design of this feature should allow extending the set of unmanaged calling conventions as needed in future. The problems include
limited space for encoding calling conventions (12 out of 16 values are taken in `IMAGE_CEE_CS_CALLCONV_MASK`) and number of places
that need to be touched in order to add a new calling convention. A potential solution is to introduce a new encoding that represents
the calling convention using [`System.Runtime.InteropServices.CallingConvention`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.callingconvention) enum.

For reference, https://github.com/llvm/llvm-project/blob/master/llvm/include/llvm/IR/CallingConv.h has the list of calling conventions
supported by LLVM. While it is unlikely that .NET will ever need to support all of them, it demonstrates that the space of calling
conventions is very rich.

## Considerations

### Allow instance methods

The proposal could be extended to support instance methods by taking advantage of the `EXPLICITTHIS` CLI calling
convention (named `instance` in C# code). This form of CLI function pointers puts the `this` parameter as an explicit
first parameter of the function pointer syntax.

``` csharp
unsafe class Instance {
    void Use() {
        delegate* instance<Instance, string> f = &ToString;
        f(this);
    }
}
```

This is sound but adds some complication to the proposal. Particularly because function pointers which differed by the
calling convention `instance` and `managed` would be incompatible even though both cases are used to invoke managed
methods with the same C# signature. Also in every case considered where this would be valuable to have there was a
simple work around: use a `static` local function.

``` csharp
unsafe class Instance {
    void Use() {
        static string toString(Instance i) = i.ToString();
        delgate*<Instance, string> f = &toString;
        f(this);
    }
}
```

### Don't require unsafe at declaration

Instead of requiring `unsafe` at every use of a `delegate*`, only require it at the point where a method group is
converted to a `delegate*`. This is where the core safety issues come into play (knowing that the containing assembly
cannot be unloaded while the value is alive). Requiring `unsafe` on the other locations can be seen as excessive.

This is how the design was originally intended. But the resulting language rules felt very awkward. It's impossible to
hide the fact that this is a pointer value and it kept peeking through even without the `unsafe` keyword. For example
the conversion to `object` can't be allowed, it can't be a member of a `class`, etc ... The C# design is to require
`unsafe` for all pointer uses and hence this design follows that.

Developers will still be capable of presenting a _safe_ wrapper on top of `delegate*` values the same way that they do
for normal pointer types today. Consider:

``` csharp
unsafe struct Action {
    delegate*<void> _ptr;

    Action(delegate*<void> ptr) => _ptr = ptr;
    public void Invoke() => _ptr();
}
```

### Using delegates

Instead of using a new syntax element, `delegate*`, simply use existing `delegate` types with a `*` following the type:

``` csharp
Func<object, object, bool>* ptr = &object.ReferenceEquals;
```

Handling calling convention can be done by annotating the `delegate` types with an attribute that specifies
a `CallingConvention` value. The lack of an attribute would signify the managed calling convention.

Encoding this in IL is problematic. The underlying value needs to be represented as a pointer yet it also must:

1. Have a unique type to allow for overloads with different function pointer types.
1. Be equivalent for OHI purposes across assembly boundaries.

The last point is particularly problematic. This mean that every assembly which uses `Func<int>*` must encode
an equivalent type in metadata even though `Func<int>*` is defined in an assembly though don't control.
Additionally any other type which is defined with the name `System.Func<T>` in an assembly that is not mscorlib
must be different than the version defined in mscorlib.

One option that was explored was emitting such a pointer as `mod_req(Func<int>) void*`. This doesn't
work though as a `mod_req` cannot bind to a `TypeSpec` and hence cannot target generic instantiations.

### Named function pointers

The function pointer syntax can be cumbersome, particularly in complex cases like nested function pointers. Rather than
have developers type out the signature every time the language could allow for named declarations of function pointers
as is done with `delegate`.

``` csharp
func* void Action();

unsafe class NamedExample {
    void M(Action a) {
        a();
    }
}
```

Part of the problem here is the underlying CLI primitive doesn't have names hence this would be purely a C# invention
and require a bit of metadata work to enable. That is doable but is a significant about of work. It essentially requires
C# to have a companion to the type def table purely for these names.

Also when the arguments for named function pointers were examined we found they could apply equally well to a number of
other scenarios. For example it would be just as convenient to declare named tuples to reduce the need to type out
the full signature in all cases.

``` csharp
(int x, int y) Point;

class NamedTupleExample {
    void M(Point p) {
        Console.WriteLine(p.x);
    }
}
```

After discussion we decided to not allow named declaration of `delegate*` types. If we find there is significant need for
this based on customer usage feedback then we will investigate a naming solution that works for function pointers,
tuples, generics, etc ... This is likely to be similar in form to other suggestions like full `typedef` support in
the language.

## Future Considerations

### static local functions

This refers to [the proposal](https://github.com/dotnet/csharplang/issues/1565) to allow the
`static` modifier on local functions. Such a function would be guaranteed to be emitted as
`static` and with the exact signature specified in source code. Such a function should be a valid
argument to `&` as it contains none of the problems local functions have today

### static delegates

This refers to [the proposal](https://github.com/dotnet/csharplang/issues/302) to allow for the declaration of
`delegate` types which can only refer to `static` members. The advantage being that such `delegate` instances can be
allocation free and better in performance sensitive scenarios.

If the function pointer feature is implemented the `static delegate` proposal will likely be closed out. The proposed
advantage of that feature is the allocation free nature. However recent investigations have found that is not possible
to achieve due to assembly unloading. There must be a strong handle from the `static delegate` to the method it refers
to in order to keep the assembly from being unloaded out from under it.

To maintain every `static delegate` instance would be required to allocate a new handle which runs counter to the goals
of the proposal. There were some designs where the allocation could be amortized to a single allocation per call-site
but that was a bit complex and didn't seem worth the trade off.

That means developers essentially have to decide between the following trade offs:

1. Safety in the face of assembly unloading: this requires allocations and hence `delegate` is already a sufficient
option.
1. No safety in face of assembly unloading: use a `delegate*`. This can be wrapped in a `struct` to allow usage outside
an `unsafe` context in the rest of the code.
