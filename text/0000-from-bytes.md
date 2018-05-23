- Feature Name: from_bytes
- Start Date: 2018-12-03
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add unsafe marker traits which indicate that byte-level conversions between a
given pair of types is safe, and thus can be done with no runtime overhead. Add
auto-derives for these traits.

In particular:
- Add the `FromBytes<T>` unsafe marker trait. `U: FromBytes<T>` indicates that
  the bytes of any valid `T` correspond to the bytes of a valid instance of `U`.
- Add the `AlignedTo<T>` unsafe marker trait. `U: AlignedTo<T>` indicates that
  `U`'s alignment requirement is at least as strict as `T`'s, and so any memory
  address which satisfies the alignment of `U` also satisfies the alignment of
  `T`.
- Add the `AbiStable` marker trait. `T: AbiStable` indicates that `T`'s ABI is
  part of its API, and that ABI changes are considered API changes.

# Motivation
[motivation]: #motivation

## Generalized `from_bits`

Various standard library types have `from_bits` functions which construct a type
from raw bits. However, none of these functions are generic. In domains such as
SIMD, in which there are many conversions that are possible, and a given program
will want to use many of them in practice, it is desirable to be able to write
code which is generic on two types whose bits can be safely interchanged.

## Safe zero-copy parsing

In the domain of parsing and serialization, a common task is to take an
untrusted sequence of bytes and attempt to parse those bytes into an in-memory
representation without violating memory safety. Today, that requires either
explicit runtime checking and copying or `unsafe` code and sound reasoning about
the exact semantics of Rust's (undefined) memory model. The former is
undesirable because it is slow, and there is a lot of boilerplate code, while
the latter is undesirable because it introduces a significant risk of memory
unsafety. However, a type that implements `FromBytes<[u8]>` can safely be
deserialized from any sufficiently-long untrusted byte sequence without any
runtime overhead or risk of memory unsafety. Similarly, a type, `T`, which
satisfies `[u8]: FromBytes<T>`, can be safely serialized without any runtime
overhead or risk of memory unsafety. In many cases, these two properties enable
the holy grail of "zero-copy" parsing and serialization.

## Generic atomics

Currently, the standard library provides types like `AtomicU8` and
`AtomicUsize`. This proposal would unlock `Atomic<T>` - so long as
`usize: FromBytes<T>`, then instances of `T` can be operated on using atomic
operations intended for word-sized values.

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## ABI stability

Since the semantics of the `FromBytes` and `AlignedTo` traits depend on the ABI
of the types that implement them, elements of the API surface such as
`U: FromBytes<T>` become dependent on the ABI of `U` and `T`. Thus, ABI-breaking
changes become API-breaking changes. This is not the case for most other Rust
APIs.

In order to make sure that we don't introduce new, backwards-incompatible
requirements for what constitutes an API-breaking change, we introduce the
`AbiStable` marker trait. `T: AbiStable` indicates that the author of `T` has
opted in to `T`'s ABI becoming part of its API, and accepting that ABI-breaking
changes will be considered API-breaking changes.

The exact definition of "ABI" for the purposes of this RFC is discussed in the
[reference-level-explanation] section.

## `FromBytes`

A new unsafe marker trait, `FromBytes<T>`, is introduced. If `U: FromBytes<T>`,
then the bytes of any `T` of appropriate length can be reinterpreted as a `U`.
The Rust compiler is modified to automatically derive `U: FromBytes<T>` for all
(`T`, `U`) pairs where both `T` and `U` implement `AbiStable`, and where `T` and
`U` satisfy the requirements of the trait. We expect this to be efficient in
practice because the derivation need only happen when code makes use of the
trait bound for a particular pair of types.

The details of the definition of `FromBytes` differ depending on whether `T` and
`U` are `Sized`. In particular,
- `T: Sized`, `U: Sized` - `U: FromBytes<T>` implies that 
  `size_of::<T>() == size_of::<U>()`
- `T: Sized`, `U: !Sized` - `U: FromBytes<T>` implies that all instances of `T`
  correspond to a valid instance of `U` of length `size_of::<T>()`
- `T: !Sized`, `U: Sized` - `U: FromBytes<T>` implies that, given `t: T`, so
  long as `size_of_val(t) == size_of::<U>()`, the bytes of `t` correspond to a
  valid instance of `U`

NOTE: This RFC does not propose to define `U: FromBytes<T>` in the case that
both `T` and `U` are unsized, as the design space for that case is significantly
larger, getting it right is more complex, and there are fewer use cases.
Hopefully a future RFC will address that case.

Note that, as a consequence of this definition, `FromBytes` is transitive. In
other words, given `U: FromBytes<T>` and `V: FromBytes<U>`, we can infer that
`V: FromBytes<T>`.

### Implications

Given this definition of `FromBytes`, we can see a number of interesting implied
implementations which can be useful for our use cases:
- If `U: FromBytes<T>`, then `&U: FromBytes<&T>`
- If `U: FromBytes<T>` and `T: FromBytes<U>`, then `&mut U: FromBytes<&mut T>`

## `AlignedTo`

A new unsafe marker trait, `AlignedTo<T>`, is introduced. If `U: AlignedTo<T>`,
then `align_of::<T>() <= align_of::<U>()`, and so any valid instance of `U`
satisfies the alignment requirements of `T`. The Rust compiler is modified to
automatically derive `U: AlignedTo<T>` for all (`T`, `U`) pairs where both `T`
and `U` implement `AbiStable`, and where `align_of::<T>() <= align_of::<U>()`.

Note that, as a consequence of this definition, `AlignedTo` is transitive. In
other words, given `U: AlignedTo<T>` and `V: AlignedTo<U>`, we can infer that
`V: AlignedTo<T>`.

## Implications

Given these definitions of `FromBytes` and `AlignedTo`, we can see a number of
interesting and useful implied implementations:
- If `U: FromBytes<T>` and `T: AlignedTo<U>`, then `&U: FromBytes<&T>`
- If `U: FromBytes<T>`, `T: FromBytes<U>`, and `T: AlignedTo<U>`, then `&mut U: FromBytes<&mut T>`

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## ABI stability

The exact definition for ABI stability is a bit more complicated than was
described in the [guide-level-explanation] section. In particular, without an
explicit `repr` attribute, types' ABIs are not defined in the Rust spec, and so
cannot be stable. Thus, a type is considered to have a stable ABI so long as all
of the following hold:
- It implements `AbiStable`
- It has a `repr` attribute which guarantees its ABI according to the Rust spec
- All of its fields/elements are also ABI stable by this definition

There's one exception: The `repr` attribute requirement is dropped in the case
of types whose ABIs are defined by Rust already (namely, primitives such as
`usize`, and certain composite types such as slices or arrays).

## `FromBytes`

The definition of `U: FromBytes<T>` given in the [guide-level-explanation]
section simply said that the bytes of any instance of `T` correspond to a valid
instance of `U`. In this section, we define precisely what that means.

### Layouts

In this section, we use the term *layout* to refer to the order and offset of
fields (including padding) in a type. We sometimes refer to "all possible
layouts of a type." For most types, this refers to its only valid layout. For
union types, however, it refers to the set of layouts for various variants of
the union. (Non-C-like enums can also have multiple layouts, but as explained
below, we don't support non-C-like enums.)

### Uninitialized bytes

According to LLVM, it is undefined behavior to operate on uninitialized memory
other than to copy it from one location to another. However, some Rust types'
layouts are defined such that some of their bytes are always uninitialized. The
case where this matters for our purposes is inter-field padding in structs and
union variants.

Since copying uninitialized memory around is OK, it's OK to only initialize some
of the bytes of an object so long as no code ever operates on the uninitialized
bytes other than to copy them. This is why it's OK to leave padding bytes
uninitialized and still copy struct or union objects - we never do anything
*other* than copy them.

This implies the following rule: In order for `U: FromBytes<T>`, it must be the
case that no uninitialized memory in any layout of `T` corresponds to
initialized memory in any layout of `U`. For example, consider the following
types:

```rust
#[repr(C)]
struct A {
    one: u8,
    // one byte of padding between `one` and `two`
    two: u16,
    three: u16,
}

#[repr(C)]
struct B {
    one: u8,
    // one byte of padding between `one` and `two`
    two: [u16; 2],
}

#[repr(C)]
struct C {
    one: u8,
    two: u8, // same location as padding in A and B
    three: [u16; 2],
}
```

It is OK for `B: FromBytes<A>` because both the `one` field and the `two` field
correspond entirely to initialized bytes in every instance of `A`. The only
uinitialized byte in `A` is the padding between the `one` and `two` fields, but
`B` has padding in the same location, so leaving it uninitialized is OK. On the
other hand, `C` has a field at that byte location, and so it is *not* OK for
`C: FromBytes<A>`.

### Self

For all `T`, `T: FromBytes<T>`.

### Primitive types

If `U` is a [numeric primitive
types](https://doc.rust-lang.org/book/first-edition/primitive-types.html#numeric-types)
(`iXXX`, `uXXX`, or `fXXX`), then `U: FromBytes<T>` if:
- For all layouts of `T`, all bytes are initialized
- Either `T: !Sized` or `size_of::<T>() == size_of::<U>()`

### Enums

The only types of enums which can be ABI stable are C-like enums (in which no
variant has any fields). If `U` is a C-like enum, then `U: FromBytes<T>` so long
as either of the following hold:
- `T` is a C-like enum, and every discriminant of `T` is a valid discriminant of `U`
- `T` is not an enum, and both of the following hold:
  - None of the bytes of any layout of `T` are uninitialized
  - Every possible discriminant of `U` is a valid one. This implies that, if
  `s == size_of::<U>()`, `U` has `2^s` variants.

### Composite types

For any composite type, `U` (where "composite types" are arrays, slices, tuples,
and structs), `U: FromBytes<T>` so long as the following holds:

For all layouts of `U`, consider each element or field of that layout. Let its
type be `UE`. Let the element's offset in bytes from the beginning of the layout
be `o`. Either of the following properties must hold:
- For all layouts of `T`, there exists an element or field at byte offset `o`
  within the layout of type `TE` such that `UE: FromBytes<TE>`
- For all layouts of `T`, both of the following properties hold:
  - There does not exist any byte offset, `b`, such that the byte at offset
    `o + b` in the layout of `T` is uninitialized, but the byte at offset `b` in
    `UE` is initialized
  - Any byte pattern is a valid instance of `UE` (in other words,
    `UE: FromBytes<[u8]>`)

### Examples

To illustrate these rules, here are a few examples:

## Example enablements

In order to demonstrate the power of the `FromBytes` and `AlignedTo` traits,
consider the following useful functions which could be written. They are all
guaranteed to never panic.
- `coerce<T, U>(x: T) -> U where U: FromBytes<T>`
- `coerce_ref<T, U>(x: &T) -> &U where U: !Sized + FromBytes<T>, T: AlignedTo<U>`
- `coerce_ref_from_unsized<T, U>(x: &T) -> Option<&U> where U: FromBytes<T>, T: !Sized + AlignedTo<U>` -
  like `coerce_ref`, but allows unsized `T` by checking that
  `size_of_val(x) == size_of::<U>()` at runtime
- `coerce_mut<T, U>(x: &mut T) -> &mut U where U: FromBytes<T>, T: FromBytes<U> + AlignedTo<U>`
- `coerce_mut_from_unsized<T, U>(x: &mut T) -> Option<&mut U> where U: FromBytes<T>, T: !Sized + FromBytes<U> + AlignedTo<U>` -
  like `coerce_mut`, but allows unsized `T` by checking that
  `size_of_val(x) == size_of::<U>()` at runtime

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

## Zero-Copy Parsing

One of the significant benefits of this proposal is that it allows for zero-copy
parsing. Consider, for example, the following simplified API for zero-copy
parsing, a fuller version of which is implemented in a production system
[here](https://docs.rs/zerocopy/0.1.0/zerocopy/):

```rust
pub struct LayoutVerified<'a, T, U>(&'a T, PhantomData<U>);

impl<'a, T, U> LayoutVerified<'a, T, U> {
    pub fn new(x: &'a T) -> Option<LayoutVerified<'a, T, U>> { ... }
}

impl<'a, T, U> LayoutVerified<'a, T, U> where T: !Sized {
    pub fn new_from_unsized(x: &'a T) -> Option<LayoutVerified<'a, T, U>> { ... }
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

The idea is that `LayoutVerified` carries a `&T` which is guaranteed to be safe
to convert to a `&U` using the implementation of `Deref`. Even though the
`Deref` impl itself only requires that `U: FromBytes<T>`, every `LayoutVerified`
constructor ensures that both size and alignment are valid before producing a
`LayoutVerified`. Thus, ownership of a `LayoutVerified` is proof that the
appropriate checks have been performed, whether at runtime or via the type
system at compile time, and so `Deref` can be implemented soundly (though unsafe
code is required in `Deref::deref`). Note that, since `U: FromBits<T>` for `T:
Sized, U: Sized` implies `U: SizeLeq<T>`, we don't use `SizeLeq` in any of the
library functions proposed above. However, we see its utility in this example:
It allows us to write the `new_sized` and `new_sized_aligned` constructors.
Without this, a `LayoutVerified`-style type that used static compile-time size
verification would need to be a distinct type.

Using `LayoutVerified`, we can write zero-copy parsing code like the following,
which is taken from a [production networking
stack](https://fuchsia.googlesource.com/garnet/+/sandbox/joshlf/recovery-netstack/bin/recovery-netstack/src/wire/udp.rs),
and requires no unsafe code (note that, in the real version, the derives are not
implemented, and so unsafe code is required for implementing `FromBytes` and
`AlignedTo`):

```rust
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

## Atomics

`AsBits` can be used to support arbitrary-type atomic operations. Any word-sized
value can be the target of an atomic CPU instruction, and so a generic atomic
type could be written as `Atomic<T> where T: AsBits<usize>`.

## Alternatives
- Use a trait like [`Pod`](https://docs.rs/pod/0.5/pod/trait.Pod.html) which
  simply guarantees that any byte sequence of length `size_of::<T>()` is a valid
  `T`. This is less general and, crucially, does not support the SIMD or atomic
  use cases.
- Omit `AlignedTo` trait. We could still coerce by value, and we could still
  coerce by reference so long as alignment were checked at runtime. However, we
  could not express infallible reference coercions, which would significantly
  reduce the utility of this system for zero-copy parsing, as it would
  re-introduce a class of bugs that could not be caught at compile-time.
- Handle endianness. We don't currently include endianness in the model of what
  is considered "safe" because by "safe" we simply mean "not causing undefined
  behavior." However, under that definition, "safe" doesn't mean "unsurprising."
  I think that the behavior that you want with respect to endianness is likely
  to be highly use case-specific, so I don't think it's appropriate in a
  general-purpose mechanism.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

- As part of development of the Fuchsia OS, a more limited form of this proposal
  was implemented [here](https://docs.rs/zerocopy/0.1.0/zerocopy/). It only
  supports going to/from byte slices (its `FromBytes` trait is analogous to this
  proposal's `FromBytes<[u8]>`, and its `AsBytes` trait is analogous to this
  proposal's `[u8]: FromBytes<T>`). It's currently used to support zero-copy
  parsing and serialization in Fuchsia's [Rust
  netstack](https://fuchsia.googlesource.com/garnet/+/master/bin/recovery_netstack).
- The [structview](https://crates.io/crates/structview) crate
- The [plain](https://crates.io/crates/plain) crate
- The [pod](https://crates.io/crates/pod) crate

## Related Rust proposals
- [pre-RFC
  FromBits/IntoBits](https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071)
- [Pre-RFC: Trait for deserializing untrusted
  input](https://internals.rust-lang.org/t/pre-rfc-trait-for-deserializing-untrusted-input/7519)
- [`Pod` trait](https://docs.rs/pod/0.5/pod/trait.Pod.html)
- [Safe conversions for
  DSTs](https://internals.rust-lang.org/t/safe-conversions-for-dsts/7379)
- [Bit twiddling
  pre-RFC](https://internals.rust-lang.org/t/bit-twiddling-pre-rfc/7072?u=scottmcm)

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

- It's a little ugly that `AlignedTo` goes in the reverse direction of
  `FromBytes` in the sense that, in order to convert a `&T` to a `&U`, you need
  `U: FromBytes<T>`, but `T: AlignedTo<U>`; is there a reasonable name we could
  come up with that would allow us to write something like `U: AlignedFrom<T>`?

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
