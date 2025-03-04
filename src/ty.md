# The `ty` module: representing types

<!-- toc -->

The `ty` module defines how the Rust compiler represents types internally. It also defines the
*typing context* (`tcx` or `TyCtxt`), which is the central data structure in the compiler.

## `ty::Ty`

When we talk about how rustc represents types,  we usually refer to a type called `Ty` . There are
quite a few modules and types for `Ty` in the compiler ([Ty documentation][ty]).

[ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/index.html

The specific `Ty` we are referring to is [`rustc_middle::ty::Ty`][ty_ty] (and not
[`rustc_hir::Ty`][hir_ty]). The distinction is important, so we will discuss it first before going
into the details of `ty::Ty`.

[ty_ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.Ty.html
[hir_ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Ty.html

## `rustc_hir::Ty` vs `ty::Ty`

The HIR in rustc can be thought of as the high-level intermediate representation. It is more or less
the AST (see [this chapter](hir.md)) as it represents the
syntax that the user wrote, and is obtained after parsing and some *desugaring*. It has a
representation of types, but in reality it reflects more of what the user wrote, that is, what they
wrote so as to represent that type.

In contrast, `ty::Ty` represents the semantics of a type, that is, the *meaning* of what the user
wrote. For example, `rustc_hir::Ty` would record the fact that a user used the name `u32` twice
in their program, but the `ty::Ty` would record the fact that both usages refer to the same type.

**Example: `fn foo(x: u32) → u32 { x }`**

In this function, we see that `u32` appears twice. We know
that that is the same type,
i.e. the function takes an argument and returns an argument of the same type,
but from the point of view of the HIR,
there would be two distinct type instances because these
are occurring in two different places in the program.
That is, they have two different [`Span`s][span] (locations).

[span]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_span/struct.Span.html

**Example: `fn foo(x: &u32) -> &u32`**

In addition, HIR might have information left out. This type
`&u32` is incomplete, since in the full Rust type there is actually a lifetime, but we didn’t need
to write those lifetimes. There are also some elision rules that insert information. The result may
look like  `fn foo<'a>(x: &'a u32) -> &'a u32`.

In the HIR level, these things are not spelled out and you can say the picture is rather incomplete.
However, at the `ty::Ty` level, these details are added and it is complete. Moreover, we will have
exactly one `ty::Ty` for a given type, like `u32`, and that `ty::Ty` is used for all `u32`s in the
whole program, not a specific usage, unlike `rustc_hir::Ty`.

Here is a summary:

| [`rustc_hir::Ty`][hir_ty] | [`ty::Ty`][ty_ty] |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Describe the *syntax* of a type: what the user wrote (with some desugaring).  | Describe the *semantics* of a type: the meaning of what the user wrote. |
| Each `rustc_hir::Ty` has its own spans corresponding to the appropriate place in the program. | Doesn’t correspond to a single place in the user’s program. |
| `rustc_hir::Ty` has generics and lifetimes; however, some of those lifetimes are special markers like [`LifetimeName::Implicit`][implicit]. | `ty::Ty` has the full type, including generics and lifetimes, even if the user left them out |
| `fn foo(x: u32) → u32 { }` - Two `rustc_hir::Ty` representing each usage of `u32`, each has its own `Span`s, and `rustc_hir::Ty` doesn’t tell us that both are the same type | `fn foo(x: u32) → u32 { }` - One `ty::Ty` for all instances of `u32` throughout the program, and `ty::Ty` tells us that both usages of `u32` mean the same type. |
| `fn foo(x: &u32) -> &u32)` - Two `rustc_hir::Ty` again. Lifetimes for the references show up in the `rustc_hir::Ty`s using a special marker, [`LifetimeName::Implicit`][implicit]. | `fn foo(x: &u32) -> &u32)`- A single `ty::Ty`. The `ty::Ty` has the hidden lifetime param. |

[implicit]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/enum.LifetimeName.html#variant.Implicit

**Order**

HIR is built directly from the AST, so it happens before any `ty::Ty` is produced. After
HIR is built, some basic type inference and type checking is done. During the type inference, we
figure out what the `ty::Ty` of everything is and we also check if the type of something is
ambiguous. The `ty::Ty` is then used for type checking while making sure everything has the
expected type. The [`astconv` module][astconv] is where the code responsible for converting a
`rustc_hir::Ty` into a `ty::Ty` is located. This occurs during the type-checking phase,
but also in other parts of the compiler that want to ask questions like "what argument types does
this function expect?"

[astconv]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir_analysis/astconv/index.html

**How semantics drive the two instances of `Ty`**

You can think of HIR as the perspective
of the type information that assumes the least. We assume two things are distinct until they are
proven to be the same thing. In other words, we know less about them, so we should assume less about
them.

They are syntactically two strings: `"u32"` at line N column 20 and `"u32"` at line N column 35. We
don’t know that they are the same yet. So, in the HIR we treat them as if they are different. Later,
we determine that they semantically are the same type and that’s the `ty::Ty` we use.

Consider another example: `fn foo<T>(x: T) -> u32`. Suppose that someone invokes `foo::<u32>(0)`.
This means that `T` and `u32` (in this invocation) actually turns out to be the same type, so we
would eventually end up with the same `ty::Ty` in the end, but we have distinct `rustc_hir::Ty`.
(This is a bit over-simplified, though, since during type checking, we would check the function
generically and would still have a `T` distinct from `u32`. Later, when doing code generation,
we would always be handling "monomorphized" (fully substituted) versions of each function,
and hence we would know what `T` represents (and specifically that it is `u32`).)

Here is one more example:

```rust
mod a {
    type X = u32;
    pub fn foo(x: X) -> u32 { 22 }
}
mod b {
    type X = i32;
    pub fn foo(x: X) -> i32 { x }
}
```

Here the type `X` will vary depending on context, clearly. If you look at the `rustc_hir::Ty`,
you will get back that `X` is an alias in both cases (though it will be mapped via name resolution
to distinct aliases). But if you look at the `ty::Ty` signature, it will be either `fn(u32) -> u32`
or `fn(i32) -> i32` (with type aliases fully expanded).

## `ty::Ty` implementation

[`rustc_middle::ty::Ty`][ty_ty] is actually a wrapper around
[`Interned<WithCachedTypeInfo<TyKind>>`][tykind].
You can ignore `Interned` in general; you will basically never access it explicitly.
We always hide them within `Ty` and skip over it via `Deref` impls or methods.
`TyKind` is a big enum
with variants to represent many different Rust types
(e.g. primitives, references, abstract data types, generics, lifetimes, etc).
`WithCachedTypeInfo` has a few cached values like `flags` and `outer_exclusive_binder`. They
are convenient hacks for efficiency and summarize information about the type that we may want to
know, but they don’t come into the picture as much here. Finally, [`Interned`](./memory.md) allows
the `ty::Ty` to be a thin pointer-like
type. This allows us to do cheap comparisons for equality, along with the other
benefits of interning.

[tykind]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html

## Allocating and working with types

To allocate a new type, you can use the various `mk_` methods defined on the `tcx`. These have names
that correspond mostly to the various kinds of types. For example:

```rust,ignore
let array_ty = tcx.mk_array(elem_ty, len * 2);
```

These methods all return a `Ty<'tcx>` – note that the lifetime you get back is the lifetime of the
arena that this `tcx` has access to. Types are always canonicalized and interned (so we never
allocate exactly the same type twice).

> N.B.
> Because types are interned, it is possible to compare them for equality efficiently using `==`
> – however, this is almost never what you want to do unless you happen to be hashing and looking
> for duplicates. This is because often in Rust there are multiple ways to represent the same type,
> particularly once inference is involved. If you are going to be testing for type equality, you
> probably need to start looking into the inference code to do it right.

You can also find various common types in the `tcx` itself by accessing its fields:
`tcx.types.bool`, `tcx.types.char`, etc. (See [`CommonTypes`] for more.)

[`CommonTypes`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/context/struct.CommonTypes.html

## `ty::TyKind` Variants

Note: `TyKind` is **NOT** the functional programming concept of *Kind*.

Whenever working with a `Ty` in the compiler, it is common to match on the kind of type:

```rust,ignore
fn foo(x: Ty<'tcx>) {
  match x.kind {
    ...
  }
}
```

The `kind` field is of type `TyKind<'tcx>`, which is an enum defining all of the different kinds of
types in the compiler.

> N.B. inspecting the `kind` field on types during type inference can be risky, as there may be
> inference variables and other things to consider, or sometimes types are not yet known and will
> become known later.

There are a lot of related types, and we’ll cover them in time (e.g regions/lifetimes,
“substitutions”, etc).

There are many variants on the `TyKind` enum, which you can see by looking at its
[documentation][tykind]. Here is a sampling:

- [**Algebraic Data Types (ADTs)**][kindadt] An [*algebraic data type*][wikiadt] is a  `struct`,
  `enum` or `union`.  Under the hood, `struct`, `enum` and `union` are actually implemented
  the same way: they are all [`ty::TyKind::Adt`][kindadt].  It’s basically a user defined type.
  We will talk more about these later.
- [**Foreign**][kindforeign] Corresponds to `extern type T`.
- [**Str**][kindstr] Is the type str. When the user writes `&str`, `Str` is the how we represent the
  `str` part of that type.
- [**Slice**][kindslice] Corresponds to `[T]`.
- [**Array**][kindarray] Corresponds to `[T; n]`.
- [**RawPtr**][kindrawptr] Corresponds to `*mut T` or `*const T`.
- [**Ref**][kindref] `Ref` stands for safe references, `&'a mut T` or `&'a T`. `Ref` has some
  associated parts, like `Ty<'tcx>` which is the type that the reference references.
  `Region<'tcx>` is the lifetime or region of the reference and `Mutability` if the reference
  is mutable or not.
- [**Param**][kindparam] Represents a type parameter (e.g. the `T` in `Vec<T>`).
- [**Error**][kinderr] Represents a type error somewhere so that we can print better diagnostics. We
  will discuss this more later.
- [**And many more**...][kindvars]

[wikiadt]: https://en.wikipedia.org/wiki/Algebraic_data_type
[kindadt]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Adt
[kindforeign]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Foreign
[kindstr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Str
[kindslice]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Slice
[kindarray]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Array
[kindrawptr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.RawPtr
[kindref]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Ref
[kindparam]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Param
[kinderr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Error
[kindvars]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variants

## Import conventions

Although there is no hard and fast rule, the `ty` module tends to be used like so:

```rust,ignore
use ty::{self, Ty, TyCtxt};
```

In particular, since they are so common, the `Ty` and `TyCtxt` types are imported directly. Other
types are often referenced with an explicit `ty::` prefix (e.g. `ty::TraitRef<'tcx>`). But some
modules choose to import a larger or smaller set of names explicitly.

## ADTs Representation

Let's consider the example of a type like `MyStruct<u32>`, where `MyStruct` is defined like so:

```rust,ignore
struct MyStruct<T> { x: u32, y: T }
```

The type `MyStruct<u32>` would be an instance of `TyKind::Adt`:

```rust,ignore
Adt(&'tcx AdtDef, SubstsRef<'tcx>)
//  ------------  ---------------
//  (1)            (2)
//
// (1) represents the `MyStruct` part
// (2) represents the `<u32>`, or "substitutions" / generic arguments
```

There are two parts:

- The [`AdtDef`][adtdef] references the struct/enum/union but without the values for its type
  parameters. In our example, this is the `MyStruct` part *without* the argument `u32`.
  (Note that in the HIR, structs, enums and unions are represented differently, but in `ty::Ty`,
  they are all represented using `TyKind::Adt`.)
- The [`SubstsRef`][substsref] is an interned list of values that are to be substituted for the
  generic parameters.  In our example of `MyStruct<u32>`, we would end up with a list like `[u32]`.
  We’ll dig more into generics and substitutions in a little bit.

[adtdef]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.AdtDef.html
[substsref]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/subst/type.SubstsRef.html

**`AdtDef` and `DefId`**

For every type defined in the source code, there is a unique `DefId` (see [this
chapter](hir.md#identifiers-in-the-hir)). This includes ADTs and generics. In the `MyStruct<T>`
definition we gave above, there are two `DefId`s: one for `MyStruct` and one for `T`.  Notice that
the code above does not generate a new `DefId` for `u32` because it is not defined in that code (it
is only referenced).

`AdtDef` is more or less a wrapper around `DefId` with lots of useful helper methods. There is
essentially a one-to-one relationship between `AdtDef` and `DefId`. You can get the `AdtDef` for a
`DefId` with the [`tcx.adt_def(def_id)` query][adtdefq]. `AdtDef`s are all interned, as shown
by the `'tcx` lifetime.

[adtdefq]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html#method.adt_def


## Type errors

There is a `TyKind::Error` that is produced when the user makes a type error. The idea is that
we would propagate this type and suppress other errors that come up due to it so as not to overwhelm
the user with cascading compiler error messages.

There is an **important invariant** for `TyKind::Error`. The compiler should
**never** produce `Error` unless we **know** that an error has already been
reported to the user. This is usually
because (a) you just reported it right there or (b) you are propagating an existing Error type (in
which case the error should've been reported when that error type was produced).

It's important to maintain this invariant because the whole point of the `Error` type is to suppress
other errors -- i.e., we don't report them. If we were to produce an `Error` type without actually
emitting an error to the user, then this could cause later errors to be suppressed, and the
compilation might inadvertently succeed!

Sometimes there is a third case. You believe that an error has been reported, but you believe it
would've been reported earlier in the compilation, not locally. In that case, you can invoke
[`delay_span_bug`] This will make a note that you expect compilation to yield an error -- if however
compilation should succeed, then it will trigger a compiler bug report.

[`delay_span_bug`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_session/struct.Session.html#method.delay_span_bug

For added safety, it's not actually possible to produce a `TyKind::Error` value
outside of [`rustc_middle::ty`][ty]; there is a private member of
`TyKind::Error` that prevents it from being constructable elsewhere. Instead,
one should use the [`TyCtxt::ty_error`][terr] or
[`TyCtxt::ty_error_with_message`][terrmsg] methods. These methods automatically
call `delay_span_bug` before returning an interned `Ty` of kind `Error`. If you
were already planning to use [`delay_span_bug`], then you can just pass the
span and message to [`ty_error_with_message`][terrmsg] instead to avoid
delaying a redundant span bug.

[terr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html#method.ty_error
[terrmsg]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html#method.ty_error_with_message

## Question: Why not substitute “inside” the `AdtDef`?

Recall that we represent a generic struct with `(AdtDef, substs)`. So why bother with this scheme?

Well, the alternate way we could have chosen to represent types would be to always create a new,
fully-substituted form of the `AdtDef` where all the types are already substituted. This seems like
less of a hassle. However, the `(AdtDef, substs)` scheme has some advantages over this.

First, `(AdtDef, substs)` scheme has an efficiency win:

```rust,ignore
struct MyStruct<T> {
  ... 100s of fields ...
}

// Want to do: MyStruct<A> ==> MyStruct<B>
```

in an example like this, we can subst from `MyStruct<A>` to `MyStruct<B>` (and so on) very cheaply,
by just replacing the one reference to `A` with `B`. But if we eagerly substituted all the fields,
that could be a lot more work because we might have to go through all of the fields in the `AdtDef`
and update all of their types.

A bit more deeply, this corresponds to structs in Rust being [*nominal* types][nominal] — which
means that they are defined by their *name* (and that their contents are then indexed from the
definition of that name, and not carried along “within” the type itself).

[nominal]: https://en.wikipedia.org/wiki/Nominal_type_system
