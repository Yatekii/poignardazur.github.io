---
layout: post
title: Analysising variadics, and how to add them to Rust
---

*\[Note - This is republishing of an article that was previous [posted on github gist](https://gist.github.com/PoignardAzur/aea33f28e2c58ffe1a93b8f8d3c58667), and advertised on reddit in January 2021.\]*

This is an analysis of how variadic generics could be added to [the Rust programming language](https://www.rust-lang.org/). It's not a proposal so much as a summary of existing work, and a toolbox for creating an eventual proposal.

## Introduction

Variadic generics (aka variadic templates, or variadic tuples), are an often-requested feature that would enable traits, functions and data structures to be generic over a variable number of types.

To give a quick example, a Rust function with variadic generics might look like this:

```rust
fn make_tuple_sing<...T: Sing>(t: (...T)) {
    for member in ...t {
        member.sing();
    }
}

let kpop_band = (KPopStar::new(), KPopStar::new());
let rock_band = (RockStar::new(), RockStar::new(), RockStar::new(), RockStar::new());
let mixed_band = (KPopStar::new(), RockStar::new(), KPopStar::new());

make_tuple_sing(kpop_band);
make_tuple_sing(rock_band);
make_tuple_sing(mixed_band);
```

**Note that variadic generics are a broader feature than variadic functions.** There are many languages implementing a feature that lets users call a function with an arbitrary number of parameters; this feature is usually called a [variadic function](https://en.wikipedia.org/wiki/Variadic_function). The extra parameters are dynamically typed (C, JS, Python) or a shorthand for passing a slice (ex: Java, Go, C#).

As far as I'm aware, there are only two widespread languages implementing variadic generics (C++ and D), which is what this document is about. (Zig has similar features, but it doesn't really have "templates" the way C++ or Rust understand it)

### About post-monomorphization errors

This document sometimes refers to "post-monomorphization errors". If you're not familiar with Rust generics, this term might confuse you.

**Post-monomorphization errors** refer to any compiler error in the code of a generic function, that isn't triggered by compiling the generic function, but is triggered by instantiating it.

A major strength of Rust compared to C++ or D is that post-monomorphization errors are virtually non-existent. If your generic function compiles when you write it, it will compile when someone else uses it.


## Prior work

### RFCs

- https://github.com/fredpointzero/rfcs/blob/variadic_tuples/text/0000-variadic-tuples.md
- https://github.com/alexanderlinne/rfcs/blob/variadic_generics/text/0000-variadic-generics.md
- https://github.com/memoryleak47/variadic-generics/blob/master/0000-variadic-generics.md
- https://github.com/rust-lang/rfcs/issues/376
- https://github.com/rust-lang/lang-team/issues/61

### Other languages

- [C++ parameter packs](https://en.cppreference.com/w/cpp/language/parameter_pack)
- [D variadics](https://dlang.org/articles/variadic-function-templates.html)
- [Zig comptime](https://ziglang.org/documentation/0.6.0/#Introducing-the-Compile-Time-Concept) - powerful reflection, with similar benefits as variadics.


## Motivations

### Implementing traits for tuples

This has been the focus of much attention, as the use case is a natural extension of the one [that motivated const-generics](https://without.boats/blog/shipping-const-generics/).

Currently, implementing a trait for a tuple is usually done by writing a macro implementing the trait over a given number of fields, then calling the macro multiple times.

This has the same problems that array implementations of traits without const generics had:

- It requires lots of awkward boilerplate.
- It's unpleasant to code and maintain; compiler errors are a lot less readable than with regular generics.
- It only implements the trait up to a fixed number of fields, usually 12. That means that a user with a 13-uple is out of luck.

For instance, right now the [implementation of Hash for tuples](https://github.com/rust-lang/rust/blob/master/library/core/src/hash/mod.rs#L612-L649) looks like:

```rust
    macro_rules! impl_hash_tuple {
        // ...

        ( $($name:ident)+ ) => (
            #[stable(feature = "rust1", since = "1.0.0")]
            impl<$($name: Hash),+> Hash for ($($name,)+) where last_type!($($name,)+): ?Sized {
                #[allow(non_snake_case)]
                fn hash<S: Hasher>(&self, state: &mut S) {
                    let ($(ref $name,)+) = *self;
                    $($name.hash(state);)+
                }
            }
        );
    }

    // ...

    impl_hash_tuple! { A }
    impl_hash_tuple! { A B }
    impl_hash_tuple! { A B C }
    impl_hash_tuple! { A B C D }
    impl_hash_tuple! { A B C D E }
    impl_hash_tuple! { A B C D E F }
    impl_hash_tuple! { A B C D E F G }
    impl_hash_tuple! { A B C D E F G H }
    impl_hash_tuple! { A B C D E F G H I }
    impl_hash_tuple! { A B C D E F G H I J }
    impl_hash_tuple! { A B C D E F G H I J K }
    impl_hash_tuple! { A B C D E F G H I J K L }
```

Variadic generics would provide an idiomatic way to implement a trait for arbitrary tuples.

### More powerful helper functions

The ecosystem has a few utility methods that operate on pairs of objects, such as `Iterator::zip` or async-std's [`Future::join`](https://docs.rs/async-std/1.9.0/async_std/future/trait.Future.html#method.join).

There are often use cases where one might want to call these methods to [combine more than two objects](https://blog.yoshuawuyts.com/future-join-and-const-eval/); currently, the default way to do so is to call the method multiple times and chain the results, eg `a.join(b).join(c)` which essentially returns `Joined(Joined(a, b), c)` (kind of like "cons lists" in Lisp).

A common workaround is instead implement the utility method as a macro, which can take an arbitrary number of parameters, but this isn't always convenient.

### Easier `#[derive]` macros

I have never seen this use case suggested before, but it seems like an obvious feature to me; it's also the main use case for variadics in D.

Rust has a lot of crates centered around providing `#[attribute]` and `#[derive]` macros to enhance your types. These macros often follow a pattern of "list your type's fields, and then for each of the fields, do something similar". For instance:

```rust
#[derive(serde::Serialize, serde::Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}
```

The Serialize, Deserialize and Debug macros all follow the same principle of "do something with `x`, then do something with `y`", where the "something" in question can be easily defined with traits. Both built-in traits like Debug and custom traits Serialize and Deserialize do this by generating a string of tokens that compiles to a Serialize/Deserialize implementation for Point.

By contrast, a serialization function in D will look like

```d
void serialize_struct(T)(Writer writer, string name, const T value)
{
    writer.startObject(name);

    static foreach (memberName; __traits(allMembers, T))
    { {
        auto memberValue = __traits(getMember, value, member);
        alias PlainMemberT = typeof(cast() memberValue);

        static if (isStruct!PlainMemberT)
        {
            serialize_struct(writer, memberName, memberValue);
        }
        else
        {
            // Serialize leaf types (eg integers, strings, etc)
            // ...
        }
    } }
}
```

There are a lot of subtle differences between D's semantics and Rust's, which means some concepts can't be trivially ported. I personally think these differences are under-studied, like much of D's generics, but the specific details are outside the scope of this document.

The gist of it is that D's generics are closer to Rust macros than Rust generics. They feel like a scripting language, where entire chunks of code can be disabled or reinterpreted based on template parameters; so naturally post-monomorphization errors are much more frequent.

That said, the enthusiastic adoption of `static foreach` in D shows that there is a strong demand for an easy-to-use, idiomatic feature to write code that applies to each field of a data structure.

### Other use cases

Some variadic generics proposals mention other possible use cases. I think they are less compelling, so I'll cover them very briefly:

- **Variadic functions:** Right now this use case is covered by macros. Eg where C has `printf`, and D has `printfln`, Rust has the `println!` macro. With variadic generics we could write a `println` function, which would be better-integrated with the language (better error messages, maybe faster compile times, etc); though it probably wouldn't replace the `println!` macro (for instance, it wouldn't be able to check the format string at compile-time).

- **Fn traits:** Fn traits currently work with compiler magic, so that they be called with arbitrary numbers and types of arguments. Proposals [mention](https://github.com/memoryleak47/variadic-generics/blob/master/0000-variadic-generics.md#motivation) that implementing variadic tuples would offer more flexibility when working with higher-order functions, though I'm not sure that's still the case.

- **Tuple manipulation:** Some proposals mention that tuples could be enhanced to be flattened, concatenated, etc. This would allow more flexible code in some situations, eg:

  ```rust
  let (package_name, ...fields) = get_some_config();
  use_config(package_name, (fields), ...get_other_config());
  ```

# Design elements

This section is not so much a proposal for variadic templates, so much as a list of questions that need to be answered to implement the feature.

It's not as structured as I'd like. I'll start with what the syntax might look like, what operations we'll want to allow, and branch out from there.


## Basic syntax

First and foremost, we want:

- To express "this template function/type/trait takes a parameter that can represent an arbitrary number of types".
- To require that each of those types implements a given trait.
- To declare tuples of variadic types, and pass them around (eg `let my_tuple : /* VARIADIC_TYPES */; foobar(my_tuple);`)
- In some cases, to "flatten" tuples, and interpret them as a comma-separated list of values, like the [spread operator in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax#examples).

Common suggestions to represent variadic types include: `Ts...`, `...Ts`, `Ts..`, `..Ts`, `Ts @ ..`. Using `...` is closer to existing C++/D syntax, but `..` is closer to existing Rust syntax for "and then a bunch of things". The `Ts @ ..` syntax in particular mimics existing [subslice patterns](https://blog.rust-lang.org/inside-rust/2020/03/04/recent-future-pattern-matching-improvements.html#subslice-patterns-head-tail--).

Finally, we want to concisely express "Execute this code for every member of this variadic tuple". There are two common approaches for this:

- Split the tuple into a "head" and a "tail" binding (eg `let (head, ...tail) = my_tuple`), process the head, and recursively call the function on the tail binding.
- A special `for` loop which iterates over tuples. Syntax could be `for member in ...tuple` or `for member ..in tuple` or something similar.

This document will analyze both approaches later.

**Note:** I don't really care what the exact syntax is, for any of these features. The examples in this document just use an arbitrary syntax, but any other could work. I ask that commenters focus on the trade-offs between features, and avoid bikeshedding at first. Thank you!


## Type inference

Let's consider the `future::join` use-case again:

```rust
let a = some_future(1u8);
let b = some_future("hello");
let c = some_future(42.0);
assert_eq!(future::join(a, b, c).await, (1u8, "hello", 42.0));
```

The`join` function takes `(SomeFuture<u8>, SomeFuture<&str>, SomeFuture<f32>)` and returns `JoinedFuture<(u8, str, f32)>`. To enable this, we need a way for a function declaration to apply a mapping to the types of a variadic tuple. The declaration of join might look like:

```rust
fn join<...Fs: Future>(futures: ...Fs) -> JoinedFuture<...Fs::Output>;
```

(Here we're adding the `...` token before a type-expression that includes a variadic argument, and instantiating the entire type-expression for each member of the variadic; this is similar to how [macro_rules! repetitions](https://doc.rust-lang.org/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming) work)

Conceptually, we are mapping our variadic parameter to a **type constructor** that looks like `<type T where T: Future> => <T as Future>::Output` (pseudocode).

The mapping should carry over all the usual rules of type inference. For instance:

```rust
fn unwrap_all<...Ts>(options: ...Option<Ts>) -> (...Ts);

let (a, b, c) = unwrap_all(Some(1), Some("hello"), Some(false));
```

In the above, type inference should understand that `unwrap_all` expects Options, and match the types of Ts to the option payloads (that is, `Ts` should be `i32, &str, bool`).

### Multiple mappings

The above examples are fairly simple, with a single mapping. Some use-cases are likely to be more complex. For instance, let's imagine a function that splits a tuple of [Eithers](https://docs.rs/either/1.6.1/either/) into tuples of Lefts and Rights.

```rust
fn split<...Lefts, ...Rights>(eithers: ...Either<Lefts, Rights>)
    -> (...Option<Lefts>, ...Option<Rights>);
```

The syntax used in this example is simplified, and makes some implicit assumptions: that `Lefts` and `Rights` variadics have the same tuple size, and that our `...` syntax iterates through both of them "simultaneously" (like `Iterator::zip`).

We can imagine use-cases where these assumptions don't hold; for instance, functions that take multiple independent variadic parameter but don't zip them, or a syntax where `...(As, Bs)` with `As = i8, i16`, `Bs = str, bool` evaluates to `(i8, str), (i8, bool), (i16, str), (i16, bool)`. It's unclear how often there would be a real-life need for those use-cases, though.

Either way, the language spec should either document these assumptions, or provide a syntax to include them in a declaration, that both the developer and the compiler can reason about, eg:

```rust
fn split<...Lefts, ...Rights>(eithers: ...Either<Lefts, Rights>)
  -> (...Option<Lefts>, ...Option<Rights>)
  where Lefts: SAME_TUPLE_SIZE_AS<Rights>;
```

(though I'm personally not super hot on the above syntax; when working with arrays, we usually don't write `fn foobar<A: Array<T>, B: Array<T2>>(...) where A: SameSizeAs<B>;`, we use const-generics)

### Concatenating variadics

Some users might want to use variadics to concatenate tuples, eg:

```rust
fn concat<...As, ...Bs>(a: (...As), b: (...Bs)) -> (...As, ...Bs);
```

Much of the previous section applies here. It's easy to come up with a syntax that leaves a lot of edge-cases unspecified.

In particular, if users are allowed to use multiple variadics in a single type list, this may make patterns harder to reason about, unless sensible limitations are specified:

```rust
fn cut_in_two<...As, ...Bs>(ab: (...As, ...Bs)) -> (...As), (...Bs);

cut_in_two((0, 1, 2, 3, 4)); // Where does As stop and Bs start?
```

### Other transformations

In theory, variadics can enable a statically-typed language to treat types almost like first-class values. Beyond `map`, a language could port most of the classic list transformations (`filter`, `fold`, `any`, `all`, `sort`, `dedup`, etc) to sequences of types (and this is indeed what D does).

It's unclear how much real-world use these transformations would have. One could, for instance, write a type constructor `IntegerTypes<Ts...>` that would return a subset of the input tuple with only integer types; but it's not obvious what practical applications this constructor would have that cannot be achieved now.

Rust generally frowns upon adding complex type operations for the sake of making them possible, the way some functional programming languages do (no Higher-Kinded Types for you). Anybody wanting to push towards D-style non-linear variadics (that is, type constructors with variadics that aren't N-to-N), will have an uphill climb ahead of them. Among other things, they'll be expected to research how these additions would impact type inference, undecidability issues, and post-monomorphization errors.


## Implementing variadic functions

Declaring variadic types in only one-half of the feature. The other is how to use them.

### Tuple `for`

A `for` loop is the most idiomatic way to express "I want to do the same thing, over and over again, with each item of this collection".

Adapting it for variadics is straightforward:

```rust
fn hash_tuple<...Ts: Hash, S: Hasher>(tuple: (...Ts), state: &mut S) {
  // hash each field
  for item ...in tuple {
    item.hash(state);
  };
}
```

We also need tuple-for to return the data at the end of its block, for its respective members:

```rust
fn unwrap_all<...Ts>(options: ...Option<Ts>) -> (...Ts) {
  for option ...in options {
    option.unwrap()
  }
}
```

(That last feature would probably be incompatible with `break` and `continue` statements; I won't go into details, but there are several ways to adress that)

And in some cases, we need to iterate over types as well as values:

```rust
fn to_vecs<...Ts>(options: (...Option<Ts>)) -> (...Vec<Ts>) {
  // Again, this is just one possible syntax
  for option, type T ...in options {
    if let Some(value) = option {
      vec![value]
    }
    else {
      Vec::new::<T>()
    }
  }
}
```

(Though I don't expect these cases to be common; Rust type inference is strong, and the example above is kind of dumb)

### Iterating over references

Given the following code:

```rust
let array = [1, 2, 3];

for x in &array {
    do_stuff(x);
}
```

we expect `x` to be of type `&i32`, not `i32`.

Similarly, with variadics:

```rust
// tuple == (1, 2, 3);

for x ...in &tuple {
    do_stuff(x);
}
```

`x` should also be of type `&i32`.

Since tuple-for probably won't be built on a trait the way existing for-loops are, the exact rules will probably be a little more magical than with iterators.

### Iterating over multiple tuples

Some use-cases might require the implementation to iterate over multiple tuples simultaneously. For instance:

```rust
fn zip<...As, ...Bs>(tuple_a: (...As), tuple_b: (...Bs)) -> (...(As, Bs)) {
  for a, b ...in tuple_a, tuple_b {
    (a, b)
  }
}
```

Another possibility is to have the `zip` function above be purely hardcoded, and have other use-cases zip their tuples before using them.

### Fold expressions

C++ provides a convenient way to fold variadics over binary operators:

```c++
template<typename ...Ts> auto sum_of(Ts&& ...ts) {
    return (0 + ... + ts);
}
```

Rust could implement something similar.

### Spread operator

Users may want to be able to create new tuples by "flattening" existing tuples, and treating them as if they were a comma-separated list of their members. In JavaScript, this is known as [the spread operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax#examples), though Rust has a similar concept known as [the struct update syntax](https://doc.rust-lang.org/stable/book/ch05-01-defining-structs.html#creating-instances-from-other-instances-with-struct-update-syntax).

This would make it possible to concatenate tuples or pass them inside arguments list, eg:

```rust
let my_tuple = (1, 2, 3, ...previous_tuple);
my_variadic_function(arg1, ...my_tuple, arg2, ...my_other_tuple, more_args);
```

This is a powerful feature, and as such, it might be hard to reason about. In particular, the type inference system would need to account for cases where the number of arguments a tuple "spreads" into is dependent on template parameters:

```rust
fn foo<T1, T2, T3>(t1: T1, t2: T2, t3: T3);

fn bar<...Ts>(tuple: (...Ts) {
  // Should probably be an error, there's no guarantee that Ts is 3 members long
  foo(...tuple);
}
```

### Destructuring patterns

Destructuring patterns are the opposite of the spread operator:

```rust
// prev_tuple == (1, 2, 3, 4, 5, 6);
let (a, ...my_tuple, b, c) = prev_tuple;
// a, b, c == 1, 5, 6
// my_tuple == 2, 3, 4
```

(Also, some destructuring syntax [already exists](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring/destructure_tuple.html) in the language)

This is often suggested as a means to enable C++11-style recursive variadics, eg:

```rust
fn print_all<...Ts>(args: ...Ts) {
  match args {
    (arg0, ...others_args) => {
      print(arg0);
      print_all(others_args);
    }
    () => ()
  }
}
```

But, generally speaking, it's a powerful feature with a lot of potential uses (and thus, much like concatenating and spreading tuples, it makes type inference harder).

Note that tuples have an implementation-defined internal layout. This means that a sequence of fields in a tuple aren't guaranteed to be a subslice of that tuple, which is why binding to destructuring patterns isn't currently allowed in Rust.

In practice, that means `let (head, ...tail) = args` might work, but `let (ref head, ref ...tail) = &args` would not.

Also, interactions with macros are non-obvious. For instance, it's not immediately clear whether `println!("somestufff {}", ...my_tuple)` would compile.

## Derives

I think this is the most under-analyzed potential benefit of variadic generics.

There is a large consensus that slow compile times are one of the major pain points of Rust (though, interestingly, as of 2021, there doesn't seem to be a recent analysis on the subject). The lifetime system can be tamed after a learning period, but the compiler stays slow, incremental improvements notwithstanding.

I'd wager that for the majority of projects, having to compile `syn` and proc-macros takes a big chunk of compile times. There are *a lot* of Rust projects that use extremely elaborate proc macros to generate trait implementations that boil down to "for every member of your struct, do X".

This is a *really inefficient* way to produce generic code. We're parsing a token-stream, performing expensive operations on the resulting AST, then producing another token tree that needs to be parsed, type-checked and borrow-checked all over again. The process is fiddly and library maintainers can easily introduce compile errors that don't show up in their tests because they can only be triggered by certain inputs (though [solutions are being explored](https://github.com/8BitMate/reflect) to adress that).

In fact, I'd be interested to see a survey of the most often used proc macros in published crates, because I suspect the vast majority of use-cases could be easily implemented with variadic generics. \[EDIT - [See here](https://gist.github.com/PoignardAzur/aea33f28e2c58ffe1a93b8f8d3c58667#gistcomment-3609667) for an informal survey. Short version, yes, they could.\]

For instance, assume that we had variadic generics, as well as a `GET_FIELDS` builtin that transforms a struct into a tuple of its fields. The Debug derive could be implemented as:

```rust
use syn::*;

#[proc_macro_derive(Debug)]
pub fn derive_debug(input: TokenStream) -> TokenStream {
    let mut derive_input = parse_macro_input!(input as DeriveInput);

    let struct_name = derive_input.ident.to_string();
    let struct_data;
    if let Struct(data) = derive_input.data {
      struct_data = data;
    }
    else {
      unimplemented!();
    }
    let field_names : VecDequeue<String> = struct_data.fields.map(|field|
      field.ident.unwrap().to_string()
    ).collect();

    let generics = add_trait_bounds(derive_input.generics);
    let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();

    let expanded = quote! {

        impl #impl_generics fmt::Debug for #struct_name #ty_generics #where_clause {
          fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
            debug_struct(#struct_name, GET_FIELDS!(self), #field_names, f);
          }
        }

    };
    proc_macro::TokenStream::from(expanded)
}

fn debug_struct_fields<...Ts: Debug>(
  struct_name: String,
  fields: &(...Ts),
  field_names: VecDequeue<String>,
  f: &mut fmt::Formatter<'_>
)
  -> Result<(), fmt::Error>
{
  f.debug_struct(struct_name);
  for field ...in fields {
    f.field(field_names.pop_front(), field);
  }
  f.finish()
}
```

* **Note:** This is an example of a simplified derive macro. Actualy macros usually have more complicated rules regarding attributes; for instance, `Derivative(Debug)` will have rules like "use the debug implementation of all these fields, but skip this one". I believe this could be adressed with the rules already described, though I won't go into details here.

The example still uses syn, because derives can only be written with proc_macros and the entire system is a tower of duct tape. With further language improvements, syn could be removed entirely (with derives based on macros-by-example, or a slightly more ergonomic TokenStream, or a `#[derive_with_variadic_generics]` builtin).

Regardless, the example is still a lot leaner than the traditional derive macro. The macro only generates 5 lines of tokens, and the heavy lifting is done by `debug_struct_fields`, which only needs to be parsed, type-checked and borrow-checked once. Any errors in the implementation will show up at compile time, not when the macro is being instantiated, with easier-to-read error messages.

How hard would the `GET_FIELDS` builtin be to implement? The main problem is that a struct layout is implementation-defined, and there is no guarantee that `struct Point{ x: i32, y: i32 };` has the same layout as `(i32, i32)`. In practice, we might want a builtin trait like `HasVariadicFields<T...>`; so that the prototype of `debug_struct_fields` would actually be:

```rust
fn debug_struct_fields<...Ts: Debug, Struct: HasVariadicFields<...Ts>>(
  struct_name: String,
  fields: &Struct,
  field_names: VecDequeue<String>,
  f: &mut fmt::Formatter<'_>
)
```

## First-class types and reflection

While they're not strictly speaking variadics, first-class types are an idea that comes up often enough.

The general idea would be to treat types as values (no different than integers or string) that can be passed to and returned from const functions, eg:

```rust
const fn do_something_with(arg: type) -> type;

type my_type = i32;
type my_other_type = do_something_with(my_type);
```

This is extremely powerful beyond the scope of variadics; first-class types make it possible to create languages inside the language, and to generate entirely new type semantics inside const functions, while using the familiar syntax of Rust code.

In principle, we can even use them to create variadic tuples and variadic generics without any dedicated syntax, extending existing features:

```rust
// Hardcoded by the compiler
impl const Iterator<Type> for Tuple {
    const LEN: usize;
    // ...
}

let my_tuple = ...;
for member in my_tuple.iter() {
    member.do_stuff();
}

struct MyTypes<const Types: Vec<Type>> {
    things: (Types),
    optional_things: (Types.map(|Type| Option<Type>)),
}
```

In general, these proposals are suggested as a replacement for variadic generics. The idea being that we don't need variadics if we can use existing syntax instead, applied to types instead of values.


# How to proceed

Everything I wrote so far has been a summary of different proposals and design ideas. I've tried to make a balanced account, without actually recommending anything.

This section is where I actually explain how I think the Rust language should implement variadics.

## Minimum viable product

The rust team recently decided to ship [a minimum version of const generics](https://github.com/rust-lang/rust/issues/74878), which is currently available on stable. This version only kept the features that were unanimously agreed upon, while postponing the features that required non-trivial design decisions for later stabilization.

I think something similar should be done with variadic generics. A first implementation should be released, focusing on the use case of implementing traits for tuples. This means:

- No zipping, flattening, concatenating, or applying type constructors to tuples. `(...Ts)` is allowed, `(...As, ...Bs)`, `...Option<As>`, `...(As, Bs)`, etc, aren't.
- No variadic argument lists. Eg `fn foobar<...Ts>(tuple: (...Ts))` is allowed, `fn foobar<...Ts>(args: ...Ts)` isn't.
- Implement tuple `for`.
- No spread operator or destructuring patterns (eg `let (a, ...my_tuple, b, c) = prev_tuple;` isn't allowed).
- No `GET_FIELDS` or `HasVariadicFields` builtin for transforming a struct into a tuple.

This is still more than enough to implement eg the `Hash` trait for any arbitrary tuple.

## Implement tuple `for`

Even before an MVP is stabilized, the language should add a tuple-for syntax for any tuple, including non-variadic tuples.

It's a micro-feature that could pull its weight even without variadics. For instance, proc-macros could generate code with tuple-for, instead of copy-pasting the same bit multiple times. The macros would still have to be called for every tuple size, but the code actually being generated would be simpler.

## Restrict tuple arithmetic

While I've tried to be balanced, I think it's clear by now that I don't think non-linear tuple arithmetic is worth implementing, even past the MVP stage.

By "non-linear", I mean any operation that takes a N-tuple and doesn't return a N-tuple: the spread operator, destructuring pattern, tuple flattening/concatenation, tuple indexing, etc.

These features are required to implement C++11-style recursive variadics, but C++11-style variadics are absolutely awful. They only work in a language resigned to horrible long post-monomorphization errors and officially-sanctionned hacks to work around the language's own limitations.

There's an interesting conversation about what we want Rust generics to be like; some of that conversation has already started, with the question of whether to allow maybe-panicking code in const generics. Whether to bring tuples closer to reified types, with flattening and indexing and so on, should be part of that conversation, which is broader than this document.

## Focus on the compile error story

This wouldn't be part of any formal proposal, but it's something to keep in mind.

This document has referred to post-monomorphization errors a lot. A major preprequisite of any variadics proposal should be that it doesn't add any.

More generally, variadics should be designed with Rust's compile-error story in mind. A major feature of Rust is that the compiler helps you locate where an error comes from quickly. A feature that works when you use it right isn't good enough; it must also be easily corrected when you use it wrong.

This should be the case on the declaration side ("your for-loop is invalid because the member type X doesn't match the member type Y of variadic types ...Xs and ...Ys") and on the user side ("you are using unwrap_all wrong because you're giving it `Option<X>, Option<Y>` but you're expecting `X, ()`, the second parameter should be `Y`).

In practice, keeping that focus requires keeping the semantics simple; it's also one of the reasons I think non-linear tuple arithmetic should be avoided.

## Avoid first-class types

First-class types are sometimes [brought up as a counter-argument](https://internals.rust-lang.org/t/analysis-pre-rfc-variadic-generics-in-rust/13879/15?u=poignardazur) against implementing variadics. The reasoning goes that we can implement all of the desired use-cases without adding any new syntax; if we do add a special `...Ts` syntax, that syntax will become a special-case, technical debt that will  need to be maintained in the compiler and taught to newcomers, without pulling its weight.

Using first-class types, we can be much more flexible about how we handle our types. We can have generics that use imperative rust code to define the types they handle, which means they can be defined on a much broader set of types without additional syntax.

But, for the reasons I listed in previous sections, I'm extremely skeptical of any such proposal.

I think the above reasoning is flawed. While first-class types may appear simpler at a glance, they come with a slew of corner cases, and subtle semantic changes, that would be much more difficult to implement (and maintain in the compiler, and teach to newcomers) than variadics alone. And, not to belabor a point, but **post-monomorphization errors are bad and first-class types would bring millions of post-monomorphization errors**.

Bluntly speaking, I don't think they're on the table.

(Speaking generally, we shouldn't add reflection the way Zig or D do it, by saying "How can we expose as much information from the type system as we can?". Instead we should ask "What are the common reflection use-cases, and how can we provide tools to implement them, such that the path-of-least-resistance for the tools is one that degrades gracefully?")

## Improve the derive use-case

Sections of this document have laid out the base of how derive macros could be improved incrementally, using variadic generics.

But in my dream future, the more generics improve, the less macros will be needed. For instance, after enough progress, we may replace the `#[proc_macro_derive]` builtin entirely with a `#[variadic_derive]` builtin:

```rust
// See https://gist.github.com/PoignardAzur/4795888034e8b40b6b312b1d9da2cf3c
// for a more complete example

#[variadic_derive(struct)]
impl<
    const Name: String,
    const ...FIELDS_INFO: FieldData,
    ...Ts: Debug,
    Struct: HasVariadicFields<...Ts, ...FieldsInfo>,
> Debug for Struct {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
      let mut f = f.debug_struct(Name);
      for field, field_info ...in self.fields, FIELDS_INFO {
        let (field_name, _field_attributes) = field_info;
        f.field(field_name, field);
      }
      f.finish()
    }
}
```

It will take a lot of intermediary features for this to be possible, beyond those described in this document. Among others:

- Advanced variadic generics.
- Advanced const generics.
- The ultimate fusion, variadic const generics.
- Some form of compile-time reflection.
- A more thought-out post-monomorphization error story, so that the derive can raise custom errors when a field's attribute doesn't match its type, in a human-readable way.

This will take a lot of time to implement, but once it *is* implemented, the vast majority of crates will finally repent.

(as in, remove their dependency on `syn`)

And that is something we can all aspire to.

---

[Discussion on reddit](https://www.reddit.com/r/rust/comments/l221r0/analysis_variadic_generics_in_rust/)
