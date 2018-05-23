- Feature Name: from_bits
- Start Date: 2018-05-23
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add the `FromBits<T>` unsafe marker trait. `U: FromBits<T>` indicates that they bytes any valid `T` can be safely interpreted as a `U`. Add the `SizeLeq<T>` and `AlignLeq<T>` unsafe marker traits. `U: SizeLeq<T>` and `U: AlignLeq<T>` indicate that `U` is less than or equal to `T` in size and alignment respectively. Add library- or compiler-supported custom derives for these traits, and various supporting library functions to make them more useful.

# Motivation
[motivation]: #motivation

## Generalized `from_bits`

Various standard library types have `from_bits` functions which construct a type from raw bits. However, none of these functions are generic. In domains such as SIMD, in which there are many conversions that are possible, and a given program will want to use many of them in practice, it is desirable to be able to write code which is generic on two types whose bits can be safely interchanged.

## Safe zero-copy parsing

In the domain of parsing, a common task is to take an untrusted sequence of bytes and attempt to parse those bytes into an in-memory representation without violating memory safety. Today, that requires either explicit runtime checking and copying or `unsafe` code and sound reasoning about the exact semantics of Rust's (undefined) memory model. The former is undesirable because it is slow, and there is a lot of boilerplate code, while the latter is undesirable because it introduces a high risk of memory unsafety. However, a type that implements `FromBits<[u8]>` can safely be deserialized from any sufficiently-long untrusted byte sequence without any runtime overhead or risk of memory unsafety. In many cases, this enables the holy grail of "zero-copy" parsing.

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A new unsafe marker trait, `FromBits<T>` is introduced. If `U: FromBits<T>`, then the bytes of any `T` of sufficient length can be safely reinterpreted as a `U`. A new custom derive is also introduced. Since `FromBits` takes a type parameter `T`, the custom derive also needs an argument indicating what values of `T` `FromBits` should be derived for. For example, the following asks the custom derive to emit `unsafe impl FromBits<[u8]> for MyStruct {}` or cause a compile error if it would be unsafe.

```rust
#[derive(FromBits)]
#[from_bits_derive(T = [u8])]
#[repr(C)]
struct MyStruct {
    a: u8,
    b: u16,
    c: u32,
}
```

Two new unsafe marker traits, `SizeLeq<T> where T: Sized` and `AlignLeq<T>` are introduced. If `U: SizeLeq<T>`, then `U`'s size is less than or equal to `T`'s size. If `U: AlignLeq<T>`, then `U`'s alignment requirements are less than or equal to `T`'s, implying that any address which satisfies `T`'s alignment also satisfies `U`'s.

Various library functions are introduced which add functionality for types implementing these traits. These include `coerce<T, U>`, which is a safe variant of `transmute` where `U: FromBits<T>`, and `coerce_ref` and `coerce_mut`, which coerce references rather than values.

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `FromBits`

A new unsafe auto marker trait, `FromBits<T>` is introduced. If `U: FromBits<T>`, then:
- If `T` and `U` are both `Sized`, then `std::mem::transmute::<T, U>(t)` does not cause UB.
- If `t: &T` and `size_of_val(t) >= size_of::<U>()`, then `std::mem::transmute::<&T, &U>(t)` does not cause UB.
- If `t: &mut T` and `size_of_val(t) == size_of::<U>()` *and* `T: FromBits<U>`, then `std::mem::transmute<&mut T, &mut U>(t)` does not cause UB.

In order for `U: FromBits<T>`, the following must hold:
- If `T` and `U` are both `Sized`, then `size_of::<T>() >= size_of::<U>()`.
- For any valid `t: T` such that `size_of_val(t) >= size_of::<U>()`, the first `size_of::<U>()` bytes of `t` are a valid instance of `U`.

These requirements have the following corrolaries:
- If it is possible to construct a value of `T` which is as large as `U` and which has uninitialized bytes (for example, from struct field padding) at byte offsets which correspond to actual values in `U`, then `U: !FromBits<T>`. This is because accessing uninitialized memory is undefined behavior. For example, the following `FromBits<T>` implementation is unsound:

```rust
#[repr(C)]
struct T {
    a: u8,
    // padding byte
    c: u16,
}

struct U {
    a: u8,
    b: u8, // at the same byte offset as the padding in T
    c: u16,
}

unsafe impl FromBits<T> for U {} // invokes UB
```

`FromBits<T>` is an auto trait, and is automatically implemented for any pair of types for which it's sound. Note that this precludes types with private fields - if a type has private fields, then implementing `FromBits` for it is unsound because the implementation might use unsafe code and rely on internal invariants that a `FromBits` implementation could violate. There is also a custom derive that allows a the author of a type with private fields to opt into `FromBits`. Here is an example of its usage:

```rust
#[derive(FromBits)]
#[derive_from_bits(T = Bar)] // derive 'unsafe impl FromBits<Bar> for Foo {}'
#[repr(C)]
struct Foo(...);
```

Note that because, without a `repr`, the memory layout of structs and enums are undefined, structs and enums without `repr` are never automatically included in `FromBits` impls, you can never derive `FromBits` for them, and they can never appear as the type parameter to `FromBits`.

## `SizeLeq` and `AlignLeq`

Two new unsafe marker traits, `SizeLeq<T> where T: Sized` and `AlignLeq<T>` are introduced. If `U: SizeLeq<T>`, then `T` and `U` are both `Sized`, and `size_of::<U>() <= size_of::<T>()`. If `U: AlignLeq<T>`, then `U`'s minimum alignment requirement is less than or equal to `T`'s, and so any address which satisfies `T`'s alignment also satisfies `U`'s.

These are both auto traits, and are automatically implemented for any pair of types for which the property holds. For struct and enum types, this requires a `repr` as with `FromBits`. Struct field privacy does not affect the auto trait implementation, so you cannot `derive` `SizeLeq` or `AlignLeq`.

## Library functions

The following safe library functions are added. They are all guaranteed to never panic.
- `coerce<T, U>(x: T) -> U where U: FromBits<T>` - interpret the first `size_of::<U>()` bytes of `x` as `U` and forget `x`
- `coerce_ref<T, U>(x: &T) -> &U where U: FromBits<T> + AlignLeq<T>`
- `coerce_ref_size_checked<T, U>(x: &T) -> Option<&U> where T: ?Sized, U: FromBits<T> + AlignLeq<T>` - like `coerce_ref`, but `x`'s size is checked at runtime, and `None` is returned if `size_of_val(x) < size_of::<U>()`
- `coerce_ref_align_checked<T, U>(x: &T) -> Option<&U> where U: FromBits<T>` - like `coerce_ref`, but `x`'s alignment is checked at runtime, and `None` is returned if it does not satisfy `U`'s alignment requirements
- `coerce_ref_size_align_checked<T, U>(x: &T) -> Option<&U> where T: ?Sized, U: FromBits<T>` - like `coerce_ref`, but `x`'s size and alignment are both checked at runtime, and `None` is returned if `x` is insufficient in either respect
- `coerce_mut<T, U>(x: &mut T) -> &mut U where T: FromBits<U>, U: FromBits<T> + AlignLeq<T>` - note that this differs from `coerce_ref` in also requiring that `T: FromBits<U>`, which is necessary because the caller might write new values of `U` to the returned reference
- `coerce_mut_size_checked<T, U>(x: &mut T) -> Option<&mut U> where T: FromBits<U> + ?Sized, U: FromBits<T> + AlignLeq<T>` - unlike `coerce_ref_size_checked`, returns `None` if `size_of_val(x) != size_of::<U>()`
- `coerce_mut_align_checked<T, U>(x: &mut T) -> Option<&mut U> where T: FromBits<U>, U: FromBits<T>`
- `coerce_mut_size_align_checked<T, U>(x: &mut T) -> Option<&mut U> where T: FromBits<U> + ?Sized, U: FromBits<T>` - unlike `coerce_ref_size_align_checked`, returns `None` if `size_of_val(x) != size_of::<U>()`

The following unsafe library functions are added. They are all equivalent to their checked counterparts, except that it is the caller's responsibility to ensure the unchecked property, and if the property does not hold, it may cause UB.
- `coerce_ref_size_unchecked<T, U>(x: &T) -> &U where T: ?Sized, U: FromBits<T> + AlignLeq<T>`
- `coerce_ref_align_unchecked<T, U>(x: &T) -> &U where U: FromBits<T>`
- `coerce_ref_size_align_unchecked<T, U>(x: &T) -> &U where T: ?Sized, U: FromBits<T>`
- `coerce_mut_size_unchecked<T, U>(x: &mut T) -> &mut U where T: FromBits<U> + ?Sized, U: FromBits<T> + AlignLeq<T>`
- `coerce_mut_align_unchecked<T, U>(x: &mut T) -> &mut U where T: FromBits<U>, U: FromBits<T>`
- `coerce_mut_size_align_unchecked<T, U>(x: &mut T) -> &mut U where T: FromBits<U> + ?Sized, U: FromBits<T>`

Note that, for all functions where `T: Sized`, the `U: FromBits<T>` bound implies that `T` is at least as large as `U` and, if a `T: FromBits<U>` bound is also present, that `T` and `U` have equal size. The `T: FromBits<U>` bound is used on `_mut` functions since, without it, there's no guarantee that all valid instances of `U` are valid instances of `T`, and so code like `*coerce_mut(&mut my_t) = U::constructor()` would not be sound.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[alternatives]: #alternatives

One of the significant benefits of this proposal is that it allows for zero-copy parsing. Consider, for example, the following simplified API for zero-copy parsing, a fuller version of which is implemented in a production system [here](https://fuchsia-review.googlesource.com/c/garnet/+/155615/):

```rust
pub struct LayoutVerified<'a, T, U>(&'a T, PhantomData<U>);

impl<'a, T, U> LayoutVerified<'a, T, U> {
    pub fn new(x: &'a T) -> Option<LayoutVerified<'a, T, U>> { ... }
}

impl<'a, T, U> LayoutVerified<'a, T, U> where T: Sized, U: SizeLeq<T> {
    pub fn new_sized(x: &'a T) -> Option<LayoutVerified<'a, T, U>> { ... }
}

impl<'a, T, U> LayoutVerified<'a, T, U> where U: AlignLeq<T> {
    pub fn new_aligned(x: &'a T) -> Option<LayoutVerified<'a, T, U>> { ... }
}

impl<'a, T, U> LayoutVerified<'a, T, U> where T: Sized, U: SizeLeq<T> + AlignLeq {
    pub fn new_sized_aligned(x: &'a T) -> LayoutVerified<'a, T, U> { ... }
}

impl<'a, T, U> Deref for LayoutVerified<'a, T, U> where U: FromBits<T> {
    type Target = U;
    fn deref(&self) -> &U { ... }
}
```

The idea is that `LayoutVerified` carries a `&T` which is guaranteed to be safe to convert to a `&U` using the implementation of `Deref`. Even though the `Deref` impl itself only requires that `U: FromBits<T>`, every `LayoutVerified` constructor ensures that both size and alignment are valid before producing a `LayoutVerified`. Thus, ownership of a `LayoutVerified` is proof that the appropriate checks have been performed, whether at runtime or via the type system at compile time, and so `Deref` can be implemented soundly (though unsafe code is required in `Deref::deref`). Note that, since `U: FromBits<T>` for `T: Sized, U: Sized` implies `U: SizeLeq<T>`, we don't use `SizeLeq` in any of the library functions proposed above. However, we see its utility in this example: It allows us to write the `new_sized` and `new_sized_aligned` constructors. Without this, a `LayoutVerified`-style type that used static compile-time size verification would need to be a distinct type.

Using `LayoutVerified`, we can write zero-copy parsing code like the following, which is taken from a [production networking stack](https://fuchsia.googlesource.com/garnet/+/sandbox/joshlf/recovery-netstack/bin/recovery-netstack/src/wire/udp.rs), and requires no unsafe code (note that, in the real version, the derives are not implemented, and so unsafe code is required for implementing `FromBits` and `AlignLeq`):

```rust
#[derive(FromBits, AlignLeq)
#[derive_from_bits(T = [u8])]
#[derive_align_leq(T = [u8])]
#[repr(C)]
struct Header { ... }

pub struct UdpPacket<'a> {
    header: LayoutVerified<'a, [u8], Header>,
    body: &'a [u8],
}

impl<'a> UdpPacket<'a> {
    pub fn parse(bytes: &'a [u8]) -> Result<UdpPacket<'a>, ()> {
        let header = LayoutVerified::new_aligned(bytes).ok_or(())?;
        ...
    }
}
```

## Alternatives
- Use a trait like [`Pod`](https://docs.rs/pod/0.5/pod/trait.Pod.html) which simply guarantees that any byte sequence of length `size_of::<T>()` is a valid `T`. This is less general and, crucially, does not support the SIMD use case.
- Omit `SizeLeq` trait. We could still have all of the library functions proposed here. However, we would loose the ability to decouple the verification and utilization steps as described in the `LayoutVerified` example, which would be a significant loss.
- Omit `AlignLeq` trait. We could still coerce by value, and we could still coerce by reference so long as alignment were checked at runtime. However, we could not express infallible reference coercions, which would significantly reduce the utility of this system for zero-copy parsing, as it would re-introduce a class of bugs that could not be caught at compile-time.
- Handle endianness. We don't currently include endianness in the model of what is considered "safe" because by "safe" we simply mean "not causing undefined behavior." However, under that definition "safe" doesn't mean "unsurprising." I think that the behavior that you want with respect to endianness is likely to be highly use case-specific, so I don't think it's appropriate in a general-purpose mechanism.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

## Related Rust proposals
- [pre-RFC FromBits/IntoBits](https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071)
- [Pre-RFC: Trait for deserializing untrusted input](https://internals.rust-lang.org/t/pre-rfc-trait-for-deserializing-untrusted-input/7519)
- [`Pod` trait](https://docs.rs/pod/0.5/pod/trait.Pod.html)
- [Safe conversions for DSTs](https://internals.rust-lang.org/t/safe-conversions-for-dsts/7379)
- [Bit twiddling pre-RFC](https://internals.rust-lang.org/t/bit-twiddling-pre-rfc/7072?u=scottmcm)

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved]: #unresolved-questions

- Is it possible to have `U: FromBits<T>` if either `T` or `U` are structs or enums that don't have a `repr`?
- What does `U: FromBits<T>` mean if `U: !Sized`?
- If `U: FromBits<T>`, `t: &mut T`, and `size_of_val(t) > size_of::<U>()` (note the strict inequality), is it safe to convert `t` to a `&mut U`? I don't think so.
- If we have a composite type with internal padding, `T`, is `T: FromBits<[u8]>` sound? In other words, is it OK to read arbitrary but meaingful bytes from a `[u8]` into the padding bytes of a `T`?
- We say that `FromBits`, `SizeLeq`, and `AlignLeq` only work for struct and enum types with `repr`s. Do any `repr`s work, or do we need to narrow it down to specific `repr`s?
- Bikeshed "coerce" to mean "safe transmute".

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
