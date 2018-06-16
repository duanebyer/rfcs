- Feature Name: `reference_specialization_types`
- Start Date: 2018-06-16
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

This RFC proposes custom reference types, referred to as Reference
Specialization Types (RSTs). Most references in Rust are implemented as "thin
pointers" to the underlying type, but there is already limited support for
special reference types: the `&[T]`, `&str`, and `&Trait` types all have unique
"fat pointer" representations. This RFC proposes to allow for similar custom
reference types with unique representations. Consequently, this RFC also allows
for custom Dynamically Sized Types (DSTs). In addition, a mechanism is
introduced by which custom types parameterized by a lifetime can participate
more directly in borrow-checking. The philosophy behind this RFC is to remove
some of the magic around reference types by allowing regular types to behave
like references do.

# Motivation
[motivation]: #motivation

Creating analogs for reference types is quite common in Rust code. For example,
within the standard library, the `RefCell` type uses `Ref` and `RefMut` as
stand-ins for immutable and mutable references. Or, within the [`rulinalg`][1]
crate, there are the [`MatrixSlice`][2] and [`MatrixSliceMut`][3] types that
act like slice references. Proxy-references are also useful when designing
collections (see [these][4] [two][5] playground examples).

In general, there is little issue using these reference analog types, other
than that they do not appear to be references in the Rust type system. However,
there are some situations in which reference analogs are particularly painful
to use. Improving support for reference analog types would not make the
impossible become possible (except in specific circumstances outlined below),
but it would make many Rust library APIs more ergonomic and more consistent
with the built-in Rust reference types.

RSTs are a somewhat complicated idea, but for the average Rust programmer, RSTs
should only reduce what they need to understand. Ideally, RSTs should let
libraries provide types that act more like the standard types: a `Matrix` with
a `&MatrixSlice` slice that acts like a `Vec<T>` with a `&[T]` slice. Existing
reference analogs have many quirks that cannot be hidden by the library author,
and so must be understood and worked around by the user of the library. RSTs
should reduce this friction.

## Implementing traits on reference analogs
Some traits cannot be implemented in a satisfactory way for reference analogs.
In [this][6] playground example, several different problems are shown when
implementing an `Expression` trait for some reference analogs `PowerRef` and
`PowerRefMut`.

1. Line 67. The conversion from `PowerRefMut` to `PowerRef` is not implicit.
   There needs to be some kind of conversion from `PowerRefMut` to `PowerRef`,
   similar to how a `&mut Power` reference can be converted to a `&Power`
   reference. The conversion uses the `Into` trait, but the `Deref` trait could
   be used instead (with some unsafe code) to provide an implicit conversion.
   Although, this is not the intended use of the `Deref` trait.
2. Line 89. When the `Expression` trait is implemented for the `PowerRef` type,
   it is impossible to provide a reasonable implementation for methods that
   take `&mut self`. All the implementation can do is `panic`: basically an
   undesirable runtime mutability check.
3. Line 97. There is a lot of duplicated code when writing the `Expression`
   implementation for the `PowerRefMut` type. This duplicated code cannot be
   refactored by converting to a `PowerRef` and calling its methods because the
   conversion method consumes the `PowerRefMut`. So, the only way to refactor
   the duplicated code is to use an `unsafe` block.
4. Line 135. Mutating the `PowerList` through the `PowerRefMut` requires that
   the `PowerRefMut` be declared as `mut`. If we were using references, this
   would be similar to if mutating an `i32` through a reference required a
   `mut &mut i32` instead of just an `&mut i32`.

None of these problems make it impossible to use reference analogs, but they do
make it unpleasant and inelegant.

## Moving and borrowing
Reference analogs do not obey the same moving and borrowing rules as regular
references. In [this][7] playground example, a `ValueRefMut` becomes unusable
after being consumed to produce its value once, while a `&mut i32` reference
would not be consumed, but instead its contents would be borrowed temporarily.

Rust will automatically reborrow the contents of a reference passed to a
a function, but it will not do this for a custom type parameterized by a
lifetime (see the [Extension](extension) section for far more detail). Again,
this problem is not enough to make reference analogs unusable, only less
ergonomic.

## Using reference analogs in generic code
Generic functions that take their arguments by reference do not always treat
reference analogs correctly. Consider a function that wants a mutable and an
immutable reference of the same underlying type `T`:

```rust
fn do_stuff<'a, 'b, T>(target: &'a mut T, other: &'b T) where /* ... */;
```

With reference analogs, such a generic function can't be used at all, since the
immutable and mutable reference analogs are distinct types. A similar problem
arises when using functions that are generic over reference lifetimes:

```rust
fn do_stuff<'a, T>(value: &'a T, /* ... */) where /* ... */;
```

Since the reference analog types take their lifetimes as a type parameter (such
as `ValueRef<'a>`), it is impossible for them to be passed to functions with
certain lifetime bounds.

This is a hard limitation of reference analog types in rare situations.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC allows for custom reference types to be defined just like any other
type can be defined. These types are called Reference Specialization Types
(RSTs). Several examples of RSTs exist in the core Rust language, namely
`&[T]`, `&str`, and `&Trait`. Custom RSTs can be used in a similar way to these
built-in types.

The purpose of an RST is to wrap an existing reference type together with a
small amount of metadata. For example, an RST might wrap a reference to a `Vec`
together with an index into that vector. Then, the RST can act like a normal
(thin) reference to an element of the vector without actually being a thin
reference (in most cases).

There are a few restrictions on how a custom RST may be defined:

* Since references come in two forms, `&T` and `&mut T`, RSTs always come in
  pairs: an immutable RST and a mutable RST.
* The immutable RST must be `Copy`.
* The immutable and mutable RSTs must have pointer-equivalent representations.
  The precise definition of this is discussed in more detail later, but in
  general this means that they must contain the same members as each other,
  except that the immutable RST must have `&` references while the mutable RST
  may have `&mut` references.

Keeping this in mind, the definition of a pair of RSTs for some underlying type
`T` looks like:

```rust
#[derive(Clone, Copy)]
struct &'a T {
    some_wrapped_ref: &'a SomeType,
    some_copy_data: SomeData,
}

struct &'a mut T {
    some_wrapped_ref: &'a mut SomeType,
    some_copy_data: SomeData,
}
```

_Aside: You might wonder why two redundant definitions are needed. This is
because many of the restrictions on the contents of an RST could be lifted in
the future, allowing immutable and mutable RSTs to have different forms. This
is discussed further under [Extension](extension)._

The type `T` is not given its own definition. You might think of RSTs as
indirectly defining the type `T` by defining the reference type `&T` instead.
Then, every type can either be defined on the value level (`struct T {}`) or on
the reference level (`struct &'a T {}`). Regardless of how you think about it,
since the underlying type `T` has no representation, it cannot be put on the
stack, and so `T` is unsized.

In many ways, `&T` can be used just like a regular value type. It can have
public members, have its members accessed with dot notation, and even be
constructed:

```rust
let rst = &T {
    some_wrapped_ref: &some_value,
    some_copy_data: SomeData { /* ... */ },
};

println!("{}", rst.some_copy_data);
```

Since the reference type `&T`&mdash;not the value type `T`&mdash;is defined by
an RST, functions that take the type `&T` are taking the immutable RST by
value. So, traits that never take `self` by value can be implemented for the
underlying unsized type `T`.

```rust
trait Trait {
    fn takes_ref(&self) -> i32;
    fn takes_ref_mut(&mut self) -> i32;
    // If this method were present, the trait could not be implemented for the
    // unsized underlying type of an RST.
    // fn takes_value(self) -> i32;
}

// Assume `T` is an unsized underyling type for some RST `&T`, `&mut T`.
impl Trait for T {
    // This method takes the immutable variant of the RST by value.
    fn takes_ref(&self) { /* ... */ }
    // This method takes the mutable variant of the RST by value.
    fn takes_ref_mut(&mut self) { /* ... */ }
}
```

An RST can be used just like a thin reference in most situations. It follows
the same moving and borrow-checking rules, can be converted to and from a
pointer, and can be used in generic code wherever a reference type is expected.
However, because `T` is an unsized type, there are some situations where an RST
cannot be used in place of a thin reference. For example, an RST (and its
pointer) cannot be moved from, dereferenced (except to reborrow with `&*`), or
placed into a trait object.

Defining an RST also defines two associated fat pointer types `*const T` and
`*mut T`. Just like thin pointers, a `*const T` can be converted to a `*mut T`,
and vice versa.

# Detailed explanation
[detailed-explanation]: #detailed-explanation

## Reborrowing
References behave in a very different way than any other type in Rust because
of the borrow checking rules that govern when a reference can be used. Because
of this, references cannot be thought of as a regular type parameterized by a
lifetime and an underlying type (like some type `FakeRef<'a, T: 'a>`). Most of
the magic around references comes from a mechanic which will be referred to as
"reborrowing". Through reborrowing, a reference can be copied with a new
lifetime `'b` that is smaller or equal to the original lifetime `'a`.
Reborrowing happens implicitly whenever references are passed to functions, and
can be done explicitly with `&*`.

```rust
fn some_function(mut_ref: &mut i32) { /* ... */ }

let mut value = 10;
let mut_ref_1 = &mut value;
{
    // An explicit mutable reborrow of `mut_ref_1`
    let mut_ref_2 = &mut *mut_ref_1;
    // Reborrowing can also be done by letting the lifetime be inferred.
    let mut_ref_3: &mut i32 = mut_ref_2;
    // Functions will reborrow their reference arguments implicitly.
    some_function(mut_ref_3);
    // A mutable reference can be immutably reborrowed.
    let const_ref_1 = &*mut_ref_3;
    // The following line does not reborrow! It copies instead, with the same
    // lifetime parameter.
    let const_ref_2 = const_ref_1;
}
```

Reborrowing cannot be done with custom types (see the [Extension](extension)
section for more thoughts on this).

```rust
struct FakeRef<'a, T: 'a> { _phantom: PhantomData<&'a mut T> }
fn some_function(fake_ref: FakeRef<i32>) { /* ... */ }

let fake_ref_1 = FakeRef::<i32> { _phantom: PhantomData };
{
    // When we try to reborrow, it moves `FakeRef` instead. A new lifetime
    // parameter is not inferred!
    let fake_ref_2: FakeRef<i32> = fake_ref_1;
}
{
    // Functions calls cannot reborrow either.
    some_function(fake_ref_1);
}
// So now when `FakeRef` is used here, we get a "use after move" error.
println!("{:?}", fake_ref_1._phantom);
```

Reborrowing is the single largest distinguishing feature between references and
other types. So, it must be supported for RSTs. To ensure that an
immutable/mutable RST pair can be safely reborrowed, the pair must be
pointer-equivalent.

## Pointer-equivalence
The concept of pointer-equivalent representation is necessary for RSTs to be
used safely (although it could be replaced with other concepts in the future).
Since an immutable RST `&T` and a mutable RST `&mut T` have pointer-equivalent
representations, then `&'a mut T` can be reborrowed as `&'b T` where `'a: 'b`.
Pointer-equivalent representations also allow for the fat pointer type
`*const T` to be converted by a simple copy to `*mut T`, and vice versa.

For two types to be pointer-equivalent with respect to a lifetime `'a`, the
members of the two types must have the same names and be in the same order.
Further, any pair of identically-named members must be pointer-equivalent with
respect to the lifetime `'a`.

The following pairs of types are also pointer-equivalent with respect to the
lifetime `'a`:

* Any `Copy` type is pointer-equivalent with itself (even if it is
  parameterized by the lifetime `'a`).
* Any thin reference/pointer (with any mutability) with lifetime `'a` is
  pointer-equivalent to any other thin reference/pointer to the same type (such
  as `&'a mut i32` and `*const i32`).

## Fat pointers
Any RST pair `&T` and `&mutT` comes with two fat pointer types `*const T` and
`*mut T` that have the same representation in memory. Mechanically, an RST is
converted into a pointer using a simple copy.

Fat pointer types are completely opaque. While the original RST may have had
public members and methods that could be accessed, fat pointers do not provide
access to these. The only way to look inside a fat pointer is to convert it
back into an RST using the `*` operator.

# Extension
[extension]: #extension

_The ideas in this section are closely related but not crucial for the core
proposals in this RFC. If implemented, they do allow for RSTs and custom
lifetime-parameterized types to be far more powerful. This is the reason for
requiring separate pointer-equivalent `&T` and `&mut T` definitions, as it
leaves some possibilities open for the future._

This RFC introduces the concept of reborrowing of custom RST types. Until now,
we have assumed that only references are allowed to be reborrowed (whether thin
or RST references), but it may be possible to extend the borrow-checker to
operate on all lifetime-parameterized types once Higher Kinded Types (HKT) are
available in Rust.

Consider the following trait:

```rust
// Let's make up an HKT syntax.
unsafe trait Reborrow<|'b| Target<'b>> {
    fn reborrow<'b>(self) -> Target<'b>;
}
```

This trait encodes the basic operation of reborrowing: a type possibly
parameterized by some lifetime `'a` is changed into a type with a some lifetime
`'b`. The implementation of this trait for the regular reference types might
be:

```rust
unsafe impl<'r, T> Reborrow<|'b| &'b T> for &'r T where 'r: 'b {
    fn reborrow<'b>(self: &'r T) -> &'b T { &*self }
}
unsafe impl<'r, T> Reborrow<|'b| &'b T> for &'r mut T where 'r: 'b {
    fn reborrow<'b>(self: &'r mut T) -> &'b T { &*self }
}
unsafe impl<'r, T> Reborrow<|'b| &'b mut T> for &'r mut T where 'r: 'b {
    fn reborrow<'b>(self: &'r mut T) -> &'b mut T { &mut *self }
}
```

The `reborrow` method is called implicitly whenever a lifetime parameter must
inferred, such as in a function call or an assignment.

```rust
struct Reborrowable<'a, T> { /* ... */ }
let reborrowable_1 = Reborrawable::<i32> { /* ... */ }
// Lifetime parameter must be inferred here.
let reborrowable_2: Reborrowable<i32> = reborrowable_1;
// Equivalent to:
// let reborrowable_2 = reborrowable_1.reborrow();
```

The `reborrow` method is able to implicitly convert between types when
necessary, just like mutable references can be implicitly converted to
immutable references. Although Rust generally disfavours implicit conversions,
in this case it is mostly harmless since a type annotation missing its lifetime
(`Reborrowable<_, i32>`) is required for the implicit conversion.

The `Reborrow` trait indicates how some implementing type works with the borrow
checker. It is closely related to the `Copy` trait, in that a `Reborrow` type
can be used after it is moved&mdash;but only after the reborrowed lifetime has
expired.

There is no `ReborrowMut` trait because the difference between immutable and
mutable references is encoded in the `Copy` trait. Since immutable references
can be used after move, calling `reborrow` on an immutable reference does not
prevent the original reference from being used. On the other hand, the
non-`Copy` mutable reference cannot be used immediately after being moved into
the `reborrow` function. Instead, it must wait for the reborrowed lifetime to
expire.

The `Reborrow` trait must be `unsafe`, as it allows types to be used after
move. It can be auto-implemented for some types with a lifetime parameter `'a`
for which it is safe to do so, just like the `Send` and `Sync` types.

If the `Reborrow` trait were to be added to Rust, then every RST pair would be
required to implement it. In the special case where the RST pair have
pointer-equivalent representations, the `Reborrow` trait could be
auto-implemented.

# Prior art
[prior-art]: #prior-art

Several other concepts for custom Dynamically Sized Types (DSTs) have touched
upon similar ideas as in this RFC. In particular, see [this][8] RFC (and its
closed [pull request][9]) and [this][10] thread on the discussion forum (please
take the time to read these&mdash;there are some valuable ideas in there).
These previous approaches have been more focused on the mechanical details of
fat pointers, and so they require large amounts of unsafe code. Furthermore,
there is limited-to-no support for references. I believe that a reference-based
approach as outlined in this RFC fits more neatly into Rust's type system, and
allows for safer DSTs.

Proxy-references as they are used in C++ are very similar to RSTs.
Proxy-references are used in the standard library in the infamous
`vector<bool>` bitvector specialization so that individual bits can act like a
`bool` reference. Proxy-references are not very nice to use in C++ as they fail
to completely mimic real references. Unlike C++, Rust has the capacity to offer
proxy-references (in the form of RSTs) that are completely indistinguishable
from regular references. Also, as an RST cannot wrap an existing type, it would
be impossible to create a proxy `bool` reference type that could be confused
with the regular `bool` reference.

# Drawbacks
[drawbacks]: #drawbacks

This RFC aims to reduce difficulty in working with reference types and
dynamically sized types, but it does not allow anything to be accomplished that
cannot already be accomplished in Rust today (except some rare edge cases with
generic functions taking references). As such, it has a lower priority than
many other RFCs.

The RFC benefits significantly from higher kinded types. The `Reborrow` trait
is what ties much of the proposal together by allowing custom types to
completely mimic the built-in reference types, but they require HKT to be
imlemented. Since HKT will take some time to be part of Rust, this RFC may need
to be postponed for a while.

The `Reborrow` trait may have unexpected consequences for how borrowing works
in Rust. At first, reborrowing of references should work as it does now, and
minor additional reborrowing support should be added for `Reborrow` types. This
support can be gradually extended until `Reborrow` types are just as integrated
with the borrow-checker as reference types. Of course, RSTs must be fully
integrated with the borrow-checker right from the beginning.

There are some other RFCs that may be affected. Although I don't know the
details, the [`pin`][11] RFC will probably interact in some way with this one.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should the `Reborrow` trait remain as part of this RFC? This RFC is all about
  making reference types less special, and the `Reborrow` trait is the natural
  conclusion of this approach. At the same time, this trait could be moved into
  its own RFC, since it can be implemented independently of RSTs.
* Should the mutable RST definition be mandatory? It's possible that some RSTs
  won't be able to provide a suitable mutable version.

[1]:  https://crates.io/crates/rulinalg
[2]:  https://athemathmo.github.io/rulinalg/doc/rulinalg/matrix/struct.MatrixSlice.html
[3]:  https://athemathmo.github.io/rulinalg/doc/rulinalg/matrix/struct.MatrixSliceMut.html
[4]:  https://play.rust-lang.org/?gist=820f41718d0145fe54bfbd0952aa55a8
[5]:  https://play.rust-lang.org/?gist=f09120537a05a4fb9851ad214c242986
[6]:  https://play.rust-lang.org/?gist=7bfa00e67ffffacf3686d44b4173a316
[7]:  https://play.rust-lang.org/?gist=3863892745914f78de613ea7d2d88857
[8]:  https://github.com/ubsan/rfcs/blob/fa32227cea5a1c3a0b2f850f33b21cb7c9efd8c3/text/0000-custom-dst.md
[9]:  https://github.com/rust-lang/rfcs/pull/1524
[10]: https://internals.rust-lang.org/t/pre-erfc-lets-fix-dsts/6663
[11]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md

