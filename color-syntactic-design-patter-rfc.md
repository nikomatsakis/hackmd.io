---
title: 'RFC: The "color" syntactic design pattern'

---

# RFC: The "color" syntactic design pattern

[#67792]: https://github.com/rust-lang/rust/issues/67792
[RFC #3628]: https://github.com/rust-lang/rfcs/pull/3628
[RFC #3668]: https://github.com/rust-lang/rfcs/pull/3668
[RFC #243]: https://github.com/rust-lang/rfcs/pull/243
[RFC #2071]: https://github.com/rust-lang/rfcs/blob/master/text/2071-impl-trait-existential-types.md
[asyncfg]: https://rust-lang.github.io/rust-project-goals/2024h2/async.html
[droporder]: https://github.com/rust-lang/rust/blob/0b16baa570d26224612ea27f76d68e4c6ca135cc/compiler/rustc_ast_lowering/src/item.rs#L1179-L1210
[WCIF]: https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/

# Summary
[summary]: #summary

This RFC unblock the stabilization of async closures by committing to `K $Trait` (where `K` is some keyword like `async` or `const`) as a pattern that we will use going forward to define a "K-variant of `Trait`". This commitment is made as part of committing to a larger *syntactic design pattern* called "colors".

In the color pattern, each color is tied to a specific keyword `K`. Colors share the "infectious property": code with color `K` interacts naturally with other code with color `K` but only interacts in limited ways with code without the color `K`.

Using `async` as the driving example, the RFC enumerates a complete "syntactic pattern" for how `K` can be applied to various pieces of code in order to enable full interop. Many parts of this pattern exist in Rust today. Other parts are specified by accepted RFCs. Still other parts are only described in pending RFCs. 

The complete set of syntactic patterns is as follows:

* `K fn $name() -> $ty` to define a K-function (e.g., async, const, etc)
* `K { $expr }` to define a K-block (and potentially `K move { $expr }`)
* `K $Trait` to reference the K-variant of a trait `$Trait`
* `K [move] |$args| $expr` to define a K-closure (which implements a `K Fn{,Mut,Once}` trait and returns the result of a `K move { $expr }` block)
* a still TBD syntax `🚲K<$ty>` that indicates the "colorized" result type that you get from executing a K-block or calling a K-function or K-closure

The color pattern also describes transformations and "rough equivalences" that users can expect:

* A `K`-function `K fn $name($args) -> $ty { $expr }` is equivalent to a regular function that wraps its return type and body with `K`:
    * `fn $name($args) -> 🚲K<$ty> { K { $expr } }`
* `K`-colored traits should offer at least the same methods, associated types, and other trait items as an uncolored version of the trait. Some of the items will be `K`-colored, but not necessarily all of them.
* A `K`-closure `K |$args| $expr` implements `K Fn` traits. Where possible, `K`-closures should be equivalent to a regular closure returning a `K`-block (but this is not always possible).
* Some kind of operation to take a `🚲K<$ty>` value and extracts the underlying `$ty`. This operation may or may not be valid outside of `K`-blocks.

Put together, this pattern enables fully functional interop for code with color `K`. The RFC therefore recommends that keywords that act "like a color" generally try to implement as much of the pattern as possible. But not all keywords require or fit the entire pattern. In addition to async, the RFC explores `const`, showing how it can be viewed as a color and which parts of the pattern fit and which do not. The [Future Possibilities](#future-possibilities) section considers how other keywords like `unsafe` could be made to fit the color pattern more fully.

The RFC's binding recommendations are as follows:

* Commit to `K $Trait` as the syntax for applying colors to traits.
* Commit to adding a TBD syntax `🚲async<$ty>` that will meet the equivalences described in this RFC.
* Consider whether current or feature languages are "color-like". If so, follow as much of the pattern as is applicable, and justify any divergences from the pattern.

These recommendations are ultimately fairly limited. Most of the design work related to colors is yet to be done and will be addressed in future RFCs. Examples of things that this RFC does NOT specify (but which early readers thought it might):

* Any form of "effect" or "color" generics:
    * Colors in this RFC are a pattern for Rust designers to keep in mind as we explore possible language features, not a first-class language feature; this RFC also does not close the door on making them a first-class feature in the future.
* Whether or how `async` can be used with traits beyond the `Fn` traits:
    * For example, the RFC specifies that **if** we add an async version of the `Read` trait, it will be referred to as `async Read`, but the RFC does **not** specify whether to add such a trait nor how such a trait would be defined or what its contents would be.
* How `const Trait` ought to work ([under active exploration][#67792]):
    * The RFC only specifies that the syntax for naming a `const`-colored trait should be `const Trait`; it does not specify what a `const`-colored trait would mean or when that syntax can be used.
* What specific syntax we should use for `🚲K<$ty>`:
    * We are committed to adding this syntax at least for `async`, but the precise syntax still needs to be pinned down. [RFC #3628][] contains one possibility.

# Motivation
[motivation]: #motivation

The Rust Project has a [flagship goal][asyncfg] of bringing the async Rust experience closer to parity with sync Rust, and one of the most important deliverables for that goal is [async closures][RFC #3668]. The only unresolved question in the async closure RFC is the syntax for async closure bounds, which could be spelled like `F: async Fn()` or `F: AsyncFn()` (or many other ways). For reasons covered more in depth in that RFC, neither of these is an obvious choice given the Rust we have today.

## Consensus: `async Fn` is nice, but only as part of a larger design pattern

The lang team discussed the syntax question [in a design meeting][DM] and concluded that we prefer offering the `async Fn` syntax as a means of requesting the "async version of a trait", but we would want it to be part of a consistent **syntactic design pattern** for selecting variants of traits.

[DM]: https://hackmd.io/@rust-lang-team/rJxAOyWaC

In other words, if users wrote `async Fn` to get an async version of a closure but `AsyncRead` to get the async version of the `Read` trait, that would be confusing. In contrast, if the pattern to get the async version of a trait is always to write the `async` keyword (e.g., `async Fn` and `async Read`), that is appealing. This is true even if the `async Read` trait is defined in a very separate way from its sync counterpart.

We also observed that there is a need for having "variants of traits" in other similar contexts, most notably `const`, where there is [active experimentation for a const-trait design][#67792]. If we further extend the pattern to cover those cases, so that one writes (e.g.) `const Fn` or `const Default` to the "const versions of the `Fn` or `Default` traits" respectively, then these two instances of the same pattern reinforce one another, helping to create a coherent whole. Any future instances we add would only strenghten this effect.

## A latent set of "colors" exists in Rust today

Looking more closely at `async` and `const`, we see that there is a latent concept within Rust that we refer to as **colors**. The term color is a playful reference to the famous ["What color is your function?"][WCIF] blog post.

Each color has an associated keyword `K` that can be applied to functions and blocks. Code with the color `K` interoperates fully with other code with the same color; working across colors is generally more limited:

* `async` functions return an `impl Future` which can only be `await`'d from another `async` block.
* `const` functions can only call other `const` functions.

## Role of this RFC: identify and describe this pattern

Code with a given color `K` is meant to interoperate fully with other code with the same color, but in practice that is not the case. This is precisely the gaps we are aiming to close by considering features like [async closures][RFC #3668] or [const traits][#67792]. The role of this RFC is to identify the "color pattern" and carry it to its logical conclusion.

We apply the pattern to the `async` keyword and use it to justify that certain features ought to be added. We are therefore committing to adding those features with this RFC; but we are not presenting a concrete design, that will be done in follow-up RFCs.

## Tenet: consistent syntax and transformations makes Rust easier to learn

The premise of this RFC is that color keywords like `const` and `async` "feel similar" to users and should be used in syntactically similar ways, even though their purpose and semantics diverge. The goal is to make these colors, and any future "color-like" features, feel as consistent and predictable as possible, so that users can develop "muscle memory" that helps them to predict syntax they haven't used or seen yet.  The RFC describes the syntax to use for a color keyword applied to various constructs as well as some of the "approximate equivalences" that colors should generally hold.

## Tenet: all parts of the color pattern should have explicit syntax centered on the keyword `K`

One of the recommendations of this RFC is committing to add a `K`-colored type syntax in the future, similar to what is proposed in [RFC #3628][]. The motivation for this is that it is important when teaching colors to be able to teach the *color specifically*, without having to reference additional language features like `impl Trait`. This addresses feedback we have received from Rust trainers which is that one of the difficulties in teaching Rust is that users can't learn Rust in "layers", they have to learn all parts of Rust at once before any of it makes sense.

## Tenet: no false equivalence, but partial consistency is better than none

Just because colors share many elements doesn't make them equivalent. We also describe how the color pattern applies to `const` and show why some parts of the pattern don't make sense in that context. We think that `const` should use the pattern where it makes sense, but avoid it (or use a distinct pattern) where it does not fit.

## Conclusion: follow the pattern for present and future colors, or explain why you don't

Looking forward, this RFC doesn't make binding statements about [future language features](#future-possibilities), but it does strongly suggest that we follow the color pattern where it makes sense. If a feature has some parts of the pattern, we should either add the other parts or offer an explanation why they don't apply.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Our guide-level explanation proceeds in three phases:

1. We begin by **defining the color pattern**, using the example of `async`.
2. Next, we develop a subset of this pattern appropriate for **effect-carrying colors**, using the example of `const` and examining how it differs from `async`.
3. Finally, we include an example of **teaching `async`**, with the goal of demonstrating how the parts of the color pattern work together to form a coherent whole. Note that colors themselves are not a user-facing concept that we teach explicitly.

## Defining the "color pattern"

### When is a keyword `K` a color?

We begin by defining the criteria where we, as language designers, should consider using the color pattern for a keyword `K`. Given the color patterns informality, there is no hard and fast rule for this, but the primary criteria to consider are as follows:

1. The keyword `K` is used to "color" code, in the form of functions and blocks.
2. `K`-colored code works best when used with other `K`-colored code; working with "uncolored" code is limited.

Apart from these minimal criteria, colors can encompass a broad range of keywords with diverse effects.

#### Example: `async`

Applying these concepts to `async`:

* Rust supports `async` functions and blocks.
* `async` functions and blocks return an `impl Future`. Awaiting futures is only possible from within other `async` functions or blocks.
    * It is possible to "block on" a future or interact with it in other ways from outside of `async` code, but it's difficult.

### `K`-colored functions, blocks, traits, and closures

The syntax to use when applying colors to functions, blocks, traits, and closures is as follows:

* `K fn $name($args) -> $ty { $expr }` for functions
* `K { $statements; $expr }` (and in some cases `K move { $statements; $expr }`) for blocks
* `K $Trait` for traits (generalizing the precedent set by [RFC #3668][])
* `K |$args| $expr` for closures (generalizing the precedent set by [RFC #3668][])

#### Example: `async`

Function and block syntax lines up with existing, stable `async` syntax:

* `async fn foo() { }`
* `let future = async move { /* ... */ };`

Trait and closure syntax follows the precedent described in [RFC #3668][]:

* `async Fn` as the async version of the `Fn` trait
* `async || /* ... */` as the syntax for an async closure.

Across all examples we see a consistent pattern that `async` appears as a prefix, consistent with its role as a kind of adjective (which in English grammar appears first).

#### What is new in this RFC

1. Resolving the unresolved question from [RFC #3668][] in the affirmative (yes, we will use the `async Fn` syntax).
2. Committing to use `async $Trait` as the syntax for any future `async`-colored traits we may add.
3. Committing to use `K $Trait` as the syntax for any other form of `K`-colored traits we may add (e.g., `const`-colored).

#### What this RFC does NOT specify

This RFC should NOT be understood as adding any form of `K`-colored traits. For now, the only example of a `K`-colored trait are the `async Fn*` traits introduced by [RFC #3668][]. This RFC does not permit users to name an `async`-colored version of any other trait. But it does specify that if we support `async`-colored versions of other trait, they should use the syntax `async $TraitName`. For example, if and when we add support for an async version of the `Drop` trait, it should be named `async Drop`. Furthermore, if and when an RFC emerges from thej [the ongoing experimentation with const traits](#66792), that RFC should use the syntax `const $Trait`.

### `K`-colored types

#### Background: what is a `K`-colored type and why might we want it?

When you tag a block or function with `async`, it changes its result type to `impl Future<Output = $ty>`, where `$ty` is the type that would normally have been produced. This characteristic is shared with some other prospective colors (e.g., [`try` and `gen`](#future-possibilities)), but not all: it corresponds to cases where the code executes differently in some way from ordinary code.

Currently there is no special syntax for this type translation. That is, the only way to reference the "type produced by an async block" is to write an explicit `impl Future`. One example where this comes up frequently is when users wish to write an `async fn` that will execute *some* code eagerly and defer the rest to execute when the future is awaited. The way to do this today is to write a regular function that returns `impl Future`:

```rust
fn do_something() -> impl Future<Output = ()> {
    // ... actions take here execute immediately ...
    async move {
        // ... actions here execute when future is awaited ...
    }
}
```


Having users write `impl Future<Output = T>` explicitly in cases like this has downsides. To start, it is verbose. But also, as a new user of Rust, it requires understanding several unnecessary concepts (the trait system, the `Future` trait, `impl Trait` syntax) in order to write async code like this.

To address these issues, [RFC #3628][] proposes adding a new syntax for "the type that results from an `async` block", but that RFC has not been accepted. Accepting the RFC would require answering two questions:

1. Does it make sense to add some syntax beyond `impl Future<Output = T>` for the result of an async block?
2. If so, precisely what syntax should we use?
    * The RFC proposes `async<T>`, but other syntaxes have been proposed, such as `async -> T`, `impl async -> T`, and `impl Future -> T`.

#### Conclusion: we do want a syntax, but we don't know what it should be yet

In discussion the lang team concluded that having an explicit syntax for "the result of an async block" that was tied to the `async` keyword was a good idea. However, similarly to the question about trait syntax, the syntax is most desirable if it is part of a larger pattern. In this case, it is part of two larger patterns: one is using the `async` keyword as a prefix to transform all things related to async (functions, blocks, traits, closures, and now types). The second is that other future color keywords may employ a similar syntax (e.g., [`try`](#try-as-a-color) or [`gen`](#gen-as-a-color)).

#### What is new in this RFC

This RFC commits us to adding *some* form of this syntax for `async` but defer the bikeshed for [RFC #3668][] or some future RFC. We refer to this future syntax `🚲async<$ty>`. Usage of this syntax should be equivalent to typing `impl Future<Output = $ty>`. Note that `🚲async<$ty>` is not a Rust type alias but a syntactic substitution, and therefore the `impl` keyword here has a different role depending on where it appears (in [function arguments][apit], it corresponds to a new generic argument to the fn; in [return position][rpit], it corresonds to an [abstract return type][rpit], etc).

[apit]: https://doc.rust-lang.org/stable/reference/types/impl-trait.html#anonymous-type-parameters
[rpit]: https://doc.rust-lang.org/stable/reference/types/impl-trait.html#abstract-return-types

This RFC also recommends that future colors which "transform" the result of a block or function introduce a corresponding syntax.

Where `🚲K<$ty>` exists it should meet the following invariants:

1. A `K`-function `K fn foo() -> T { $expr }` has a body that returns a value of type `T`. When called, however, its caller receives a `🚲K<T>` value.
2. A `K`-block `K { $expr }` where `$expr: $ty` has type `$ty` produces a result of type `🚲K<T>` value.
3. A `K`-closure `K |$args| -> T { $expr }` has a body that returns a value of type `T`. It implements the trait `K Fn<Output = T>`. This trait ensures that when the closure is called its caller receives a `🚲K<T>` value.

### Relationship between `K`-colored functions, blocks, and types

Looking at async functions and the proposed `🚲async<$ty>` notation, we observe the following relationship. An `async` function like `count_input`...

```rust
async fn count_input(urls: &[Url]) -> usize {
    // ... iterate over urls and load data ...
}
```

...can be transformed into an "ordinary" function that (a) returns `🚲async<$ty>` and (b) uses an `async move` block (caveat: the true transformation is slightly more subtle [in order to preserve the drop order for method arguments][droporder]):

```rust
fn count_input(urls: &[Url]) -> 🚲async<usize> {
    async move {
        // ... iterate over urls and load data ...
    }
}
```

#### What is new in this RFC:

This RFC recommends that future colors that define `🚲K<$ty>` meet this invariant:

* A `K fn foo() -> T` should be equivalent to a (uncolored) `fn` that returns `🚲async<T>` and uses some form of `K`-colored block.

In some cases, colors may meet a different invariant, where the color `K` is preserved on the `fn`:

* A `K fn foo() -> T` should be equivalent to a `K fn` that returns `🚲async<T>` and uses some form of `K`-colored block.

This is appropriate in cases where the presence of color `K` is flagged some form of pervasive effect.

## The "effect-carrying color" pattern

The previous section defined the color pattern in full with reference to `async`. Examining the criteria for colors, there are two other existing keywords in Rust that ought to be considered: `unsafe` and `const`. This section explores `const` specifically; `unsafe` is discussed in the [Future Possibilities](#future-possibilities) section. We begin by comparing `const` and `async`, identifying some key differences, and based on those differences identifying the subset of the color pattern that applies to const. We suggest that this subset may also apply to other colors that share these characteristics.

### At first glance, `const` and `async` seem very different

Both async and const can be applied to blocks and functions, but there are important differences between them. Async changes how a function body is compiled and the type of value that is produced; the result of async functions (and async blocks) are only usable from other async contexts.

In contrast, `const` limits the operations a function can perform to those that the compiler can execute at runtime. It does not change how a function is compiled at runtime nor does it change the type of value that is produced.

The relationship between `async` functions and `async` blocks is also different in kind than the relationship between `const` functions and `const` blocks. An `async` function is sugar for a function with an `async` block in its body (and a different return type). In contrast, a `const` function is something that can be run at compilation or runtime, and a `const` block is code that *only* runs at compilation time.

And yet, despite all these differences, both `const` and `async` intuitively *feel* very similar -- they both *feel* like colors to users, as argued in the motivation. How to understand this?

### `const` is best understood as an inverted default for a `runtime` color

The natural orientation for colors is additive: having a `K`-colored block gives access to additional capabilities. `const`, in contrast, is a *subtractive* color. A `const` function can do fewer things than an ordinary function. To see the difference, imagine an alternate form of Rust, Runtime-Rust. Runtime-Rust works exactly like Rust, but the defaults are flipped: code blocks, by default, execute at compilation time. To execute at runtime, they must be tagged with `runtime { ... }`, which in turn gives them access to capabilities that are only permitted at runtime, like FFI. This `runtime` color matches `async` much more closely: just as `async` blocks can only fully be used from inside another `async` block, `runtime` blocks can only be used from inside `runtime` blocks. What this thought experiment shows is that `const` is actually an inverted default, like `?Sized`.

### What parts of the pattern work for `runtime`?

Continuing with the RuntimeRust hypothetical, most parts of the color pattern apply equally well to `runtime` and `async`:

* `K`-colored functions, blocks, and closures are prefixed with the keyword `K`.
* `K`-colored traits are prefixed with the keyword `K`.
    * A `K`-colored trait `Trait` include (at least) the members of `Trait`, with some of them colored by `K`.
* `K`-colored functions and closures implement `K`-colored variants of the `Fn` trait.

There is one part however that doesn't really fit:

* `K`-colored types can be written with the syntax `🚲K<T>`:
    * `K`-colored functions, closures, and blocks return/produce `🚲K<T>` values, where `T` is their original type.

Introducing a `🚲runtime<T>` syntax to describe a "runtime-colored type" doesn't make sense, because `runtime` blocks don't change the type of value that is produced. So `🚲runtime<T> = T` always. We don't need syntax for this and having it would be confusing.

The next that doesn't fit is the equivalence between a `K`-colored function and a function returning a `K`-block:

* `K fn $name() -> T { $expr }` is equivalent to `fn $name() -> 🚲K<T> { K { $expr } }`

This equivalence doesn't work because the `runtime` color cannot be *encapsulated*. You can't have a compilation time function that uses a `runtime` block internally but hides it from its callers. In contrast, you *can* have a synchronous Rust function that creates an `async` block internally and executes it by polling in place or some other means.

### The "effect-carrying color" subset of the color pattern

Based on our discussion of `runtime`, we can identify a subset of colors which we will call *effect-carrying* colors. Effect-carrying colors indicate blocks of code that are capable of performing "extra" operations beyond what is available in an uncolored block. Because these colors don't change how code is compiled, they don't change the type of value that is produced. Further, the transformation from `K fn $name` to a function with a `K`-block doesn't work, because the point of a `K fn` is to signal to callers that the effect will take place.

For effect-carrying colors, the parts of the color pattern which make sense are the following:

* `K`-colored functions, blocks, and closures are prefixed with the keyword `K`.
* `K`-colored traits are prefixed with the keyword `K`.
    * A `K`-colored trait `Trait` include (at least) the members of `Trait`, with some of them colored by `K`.
* `K`-colored functions and closures implement `K`-colored variants of the `Fn` trait.

The other points do not apply. The implication of this is that we should support `const`-colored traits and `const`-colored closures and we should use the same prefix syntax used by `async`.

## Teaching async via the "color pattern"

> The "color pattern" is meant to guide Rust designers as we extend the language, not to be something directly taught to end users. To illustrate how it might feel, this section covers an example of teaching `async` leveraging the syntax and concepts of the color pattern (but not teaching the pattern explicitly). To help illustrate where the overall vision for Rust, we assume that `🚲K<$T>` is `K -> $T` (a variant of [RFC #3628][] currently preferred by the author), that `async -> T` and its equivalent `impl Trait<Output = T>` are supported in local variable declarations ([RFC #2071][]), and that there is some form of `async Iterator` trait available in std.

### Async functions

Rust's `async` functions are a built-in feature designed for building concurrent applications. Async functions are just like regular functions, but prefix with the `async` keyword:

```rust
async fn load_data(input_urls: &[Url]) -> Vec<Data> {
    let mut data = Vec::new();
    for url in input_urls {
        // Load the data from `url` and merge it into `data`
        data.push(load_url(url).await);
    }
    data
}

async fn load_url(url: &Url) -> Data {
    // ... do something ...
}
```

When you call an `async` function, the function does not execute immediately. Instead, it returns a suspended function execution, called a *future*. This future can later be *awaited* to cause it to execute. We see this in `load_data`, which invokes a helper async function `load_url` and then awaits the result before pushing it into `data`.

If we were break that call to `load_url` out into separate statements, it would look like this:

```rust
for url in input_urls {
    // Calling `load_url` yields a future, the type of which is denoted as `async -> Data`:
    let url_future: async -> Data = load_url(url);

    // Awaiting the future causes it to execute synchronously.
    // Once it completes, we have a `Data`.
    let url_data: Data = url_future.await;

    // Load the data from `url` and merge it into `data`
    data.push(url_data);
}
```

Some things to note:

* The value returned by `load_url` is not of type `Data`, it's of type `async -> Data`.
  This notation indicates that `url_future` is in a fact a future that, when awaited, will yield a `Data` value.
* The notation `url_future.await` is used to await a future.
  This causes the current task to block and execute the function.

In our original example, we immediately awaited the result of the function call (`load_url(url).await`). This is equivalent to calling a synchronous function.

Where futures become powerful is when you combine them to create concurrent patterns. For example, suppose the caller has a list of URLs `urls` and would like to split it in half and process the data from the two halves concurrently:

```rust
// Split list of URLs into two pieces:
let mid_point = urls.len() / 2;
let (urls_1, urls_2) = urls.split(mid_point).unwrap();

// Create two futures by loading these halves.
// Nothing happens at this point, we just create
// two suspended computations.
let future_1: async -> Vec<Data> = load_data(urls_1);
let future_2: async -> Vec<Data> = load_data(urls_1);

// Join those two futures together into a new future.
let future: async -> (Vec<Data>, Vec<Data>) = join!(future_1, future_2);

// Await the joined future. This will process both
// halves concurrently and yield up a tuple with
// the two vectors.
let (data_1, data_2): (Vec<Data>, Vec<Data>) = future.await; 
```

### Async blocks

In addition to async functions, Rust supports async *blocks*. These are a lighterweight alternative to defining an async function and are useful when you want to suspend execution of a small piece of code that references various things from its environment, similar to a closure. For example, we might like to make a future that invokes `load_data`, awaits the result, and then does some light pre-processing:

```rust
let processed_data: async -> ProcessedData = async {
    load_data(urls).await.process()
};
```

Creating an async block yields an `async -> T` future representing the suspended computation, just like the futures that result from calling an async function.

Or, to continue with our `join!` example, we might load and process the data from two sets of URLs concurrently:

```rust
let (data_1, data_2) = join!(
    async { load_data(urls_1).await.process() },
    async { load_data(urls_2).await.process() },
).await;
```

Here, the futures supplied to the `join!` macro are two async blocks, instead of just calls to `load_data`.

#### `async` vs `async move`

Async blocks resemble closures in some ways. Just as a Rust closure by default takes references to variables from the surrounding stack frame, async blocks yield futures that store references to the variables they need whenever possible. You can however use the `async move` syntax to force the future to take ownership of any data that is uses.

### Async closures

We previous defined `load_data` using a `for` loop:

```rust
async fn load_data(input_urls: &[Url]) -> Vec<Data> {
    let mut data = vec![];
    for url in input_urls {
        data.push(load_url(url).await);
    }
    data
}
```

We saw in Chapter XX that we rewrite those for-loops to use iterators with a `map`/`collect` call. If you try that in `load_data`, however, it won't compile, because you can't use `await` in a synchronous closure:

```rust
async fn load_data(input_urls: &[Url]) -> Vec<Data> {
    input_urls
        .iter()
        .map(|url| load_url(url).await)
        //                       ^^^^^
        //          Error: await from a synchronous closure
        .collect()
}
```

To use `await` from inside the `map` closure, the closure needs to be tagged as `async`. This in turn requires you to use an *async* iterator, which you can get by invoking `async_iter`. Once you make these changes, we can rewrite `load_data` to use an async iterator as follows:

```rust
async fn load_data(input_urls: &[Url]) -> Vec<Data> {
    input_urls
        .async_iter()
        .map(async |url| load_url(url).await)
        .collect()
        .await // <-- collect yields a future, must await
}
```

> **Meta note:** We are assuming here that some kind of `async Iterator` trait is eventually added that supports functionality similar to [`futures::Stream`](https://docs.rs/futures/latest/futures/prelude/trait.Stream.html). This functionality is not specified in this RFC.

### Desugaring async functions into async blocks

The `async fn` syntax is actually a shorthand for a more general form of declaration that makes use of `async -> T` types and `async` blocks. The more general declaration is formed by moving the `async` keyword from before the `fn` to the return type and then transforming the body to use an `async move` block:

```rust
fn load_data(urls: &[Url]) -> async -> Data {
    async move {
        // ... same as before ...
    }
}
```

This more general declaration gives you more control over what happens when `load_data` is called. Among other things, it can let you take some actions right away, while deferring others to run when the resulting future is awaited. To do this, simply add some statements before the `async move` block:

```rust
fn load_data(urls: &[Url]) -> async -> Data {
    // ... this code executes immediately ...
    async move {
        // ... this code executes when the future is awaited ...
    }
}
```

### Comparing async-await in Rust to async-await in other languages

Many languages have some variant of async-await notation and so async functions in Rust likely look familiar to you. That's good! But be aware that async-await works differently in Rust than most other languages.

The most obvious difference is that where most languages put `await` as a prefix, Rust makes it into a suffix. So instead of writing `await foo` you write `foo.await`. This is more convenient when writing chained operations, like `data.load().await.load_more().await`, and especially when dealing with futures that yield `Result`, as one can use `?` like `load_data().await?`.

The other, more subtle, difference has to do with Rust's execution model. In most languages, calling an async function implicitly starts up a background task that will continue executing. Rust, like Kotlin, takes a different approach: calling an async function returns a suspended computation, which on its own is inert. That future can then by combined with other futures to form aggregates, like the `join!` operation that we saw in the previous example. Eventually, the future or the aggregate that it is embedded into must be awaited -- and, when it is, that will block your current task until it completes.

If you'd like the async functon to execute in the background, then you need to use a `spawn` function to create a new task (analogous to spawning a new thread in synchronous code). Rust itself does not provide a spawn function, but one is typically provided by your async runtime. The most common choice here is `tokio`, which offers [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html):

```rust
let data_future = tokio::spawn(async move { load_data(urls).await });
// ... `data` is not being loaded in the background, in parallel ...
```

### Digging deeper: the `Future` trait

We have thus far avoided saying precisely what a future *is*, apart from a "suspended computation". The more precise answer is that a future is a value that implements the `Future` trait. The notation `async<T>` that we have been using is in fact a shorthand for `impl Future<Output = T>`, meaning "a value of some type that implements `Future<Output = T>`". We introduced the `impl Future` notation in Chapter XX, and `async -> T` works the same way:

* When used in function argument position like `fn method(data: async -> Data)`, `async -> T` is equivalent to adding a new anonymous type parameter `A` where `A: Future<Output = T>`.
* When used in return position like `fn method() -> async -> Data` or in a type alias like `type DataFuture = async -> Data`, `async<T>` desugars to an opaque type whose precise value is inferred from the function body or uses of the type alias, respectively.
* When used in local variable position like `let data: async -> Data = ...`, `async -> Data` desugars to an assertion that the type of `data` implements `Future<Output = Data>`.
* In other positions, `async -> T` (like `impl Future<Output = T>`) is an error.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section collects the precise statements of this RFC.

## When is a keyword `K` a color?

There is no hard and fast rule for this, but the primary criteria to consider are as follows:

1. The keyword `K` should be applicable to functions and blocks of code.
    * `const fn foo()`, `const { .. }`
    * `async fn foo()`, `async { .. }`, `async move { .. }`
2. `K`-colored code should generally only be usable with other `K`-colored code.
    * `const` functions can only call other `const` functions
    * `async` functions return futures that can only be awaited from async blocks

## Syntactic patterns

Color keywords are not required to be accepted in all locations. But when they are accepted, they should generally prefix the syntax they are colorizing:

* `K fn $name($args) -> $ty { $expr }` for functions
* `K { $statements; $expr }` for blocks
* `K $Trait` for traits
* `K |$args| $expr` for closures

## Rules for rewrite colors

A **rewrite color** is a color where `K`-blocks produce values of different types than an unmodified block. A future RFC will specify a consistent syntax, currently denoted as `🚲K<$ty>`, that can be used to specify the type resulting from a `K`-colored block, function, or closure that would otherwise produce a value of type `$ty`.

The type `🚲K<$ty>` relates to the type of blocks, functions, and closures as follows:

1. A `K`-block `K { $expr }` produces a value of type `🚲K<$ty>`, where `$ty` is the type of `$expr`.
2. A `K`-function `K fn foo() -> $ty { $expr }` has a body that returns a value of type `$ty`. When called, however, its caller receives a `🚲K<$ty>` value.
3. A `K`-closure `K |$args| -> $ty { $expr }` has a body that returns a value of type `$ty`. It implements the trait `K Fn<Output = $ty>`. This trait ensures that when the closure is called its caller receives a `🚲K<$ty>` value.

For every rewrite color `K`, there should be a transformation from a `K`-function to an uncolored function that returns `🚲K<$ty>` and uses a `K`-block in its body:

```rust
K fn example($args) -> $ty {
    $body
}

// becomes

fn example($args) -> 🚲K<$ty> {
    K { $body }
}
```

This translation is however not precise. For example, for `async`, the `K`-block is an `async move` block and includes [`let` statements to preserve drop order][droporder]. Other rewrite colors may work slightly differently.

# Rationale, alternatives, and FAQ
[rationale-and-alternatives]: #rationale-and-alternatives

## Why include `🚲K<$T>` syntax?

Most parts of the color pattern already exist in Rust today or at least in accepted RFCs. The `🚲K<$T>` syntax for colored types stands out as the exception. It was included in the RFC because it forms an important part of the overall story (witness how prominent it is in the guide section). Some members of the lang team felt that, without `🚲K<$T>`, they didn't feel good about the color pattern overall.

## Why not use the camel-case named for K-colored traits, like `AsyncFn` instead of `async Fn`?

We considered a number of alternatives to `K Trait` for denoting K-colored traits. Perhaps the most obvious was a camelcased identifier like `AsyncFn`. While this is a perfectly viable option, it has downsides as well:

* First, the story of how to transition something from sync to async gets more complicated. It's not a story of "just add `async` keywords in the right places".
* Second, this convention does not offer an obvious way to support const traits like `const Default`, unless we are going to produce `ConstDefault` variants as well somehow. And if there are more variants in the future, e.g., `AsyncSendSomeTrait` or `AsyncTrySendSomeTrait`, it becomes very unwieldy.
* Third, although we are not committing to any form of "color generics" in this RFC, we would also prefer not to close the door entirely; using an entirely distinct identifier like `AsyncFn` would make it very difficult to imagine how one function definition could be made generic over a color.

Ultimately, the killer argument was that this felt like everybody's preferred "second choice option", but nobody's *favorite*. It's not a bold design.

## Why not use a notation like `T: async fn()` instead of `T: async Fn()`?

One advantage of today's sugar for trait bounds (`T: Fn()`, `T: FnMut()`, etc) is that it more closely resembles function declarations. Adding the async keyword continues that, in that one now puts `async` in front of the `fn` or closure and then likewise in front of the bound (`T: async Fn()`). Iterating on that vein, we considered going further with bound notation, so that one would write `T: fn()` instead of `T: Fn()` and then `T: async fn()` instead of `T: async Fn()`. The following objections were raised:

* There is no obvious way to encode `T: FnMut()` or `FnOnce()`. Should the notation be `T: fn mut()`and `T: fn once()`? Then we are losing the parallel to function declarations and introducing some ad-hoc keywords.
* Changing the notation for `T: Fn()` bounds is a massive change affects virtually all existing Rust code. To make a change like that, we have to be very confident the change will be a win, and many were not (particularly given the previous bullet).

## `const` functions can't be rewritten to have a `const` block in their body. Is that inconsistent with `async`?

The point of this RFC to enable syntactic transformations and intutions that apply across colors. And yet one of the  transformations we describe, moving the `K` keyword from the `fn` to its return type and body, only applies to one color (`async`) and not the other (`const`).

This inconsistency is skin deep, however. `const` is best thought of as a "subtractive" color, one that indicates a block that *denies* operations an ordinary block would permit.

If we imagine an alternative Rust where `const fn` was the default, then we might say that *regular* functions have to be tagged as a `runtime fn`, indicating that they can only be used at runtime, and that a `runtime { .. }` block would be one that can contain runtime operations. In this case, a `runtime fn foo() { $body }` could be transformed to a `runtime fn foo() { runtime { $body } }`. Note that we still need the `runtime` on the fn as a signal to callers that there is a `runtime { .. }` block in the body, and hence this function cannot be called from const code; this is a difference from rewrite colors, where the effect on the caller is expressed via a different return type.

## Can `🚲K<$T>` be used in closure return types like it can for regular functions?

The answer to this is complicated and left to be resolved by future RFCs that actually add the `🚲K<$T>` notation. Recall that we specified that `K`-colors can be migrated from a `K`-function to its return type and body:

```rust
K fn foo() -> T { ... }
// becomes
fn foo() -> 🚲K<T> { K { ... } }
```

The question then is whether it is possible to transform a a `K`-closure in a similar way:

```rust
K |args| -> T { ... }
// could maybe become
|args| -> 🚲K<T> { K { ... } }
```

Supporting this notation is tricky however because closure return types are often elided. Given that `K`-closure types implement `K Fn` rather than the typical `Fn` trait, we need to be able to determine if a closure has color `K` to decide what traits it implements. But there is no way to distinguish whether an expression like `|| K { .. }` is meant to be a `K`-closure that implements `K Fn<Output = T>` or an uncolored closure that implements `Fn<Output = 🚲K<T>>` value. As described in [RFC #3668][], these two traits are not equivalent for `async` and likely not for other future rewrite colors.

We've left the behavior here to be specified in a future RFC, but we note that it would be simple to declare it as an error for now. Note that `-> impl Future` is also not accepted in this position.

# Prior art
[prior-art]: #prior-art

The term "color" is taken from the ["What color is your function?"][WCIF] blog post. As that post indicates, many languages have added colors of one form or another. This RFC is focused on the syntactic patterns around colors, specifically using the color as a prefix (e.g., `async fn`, `🚲async<T>`, `async Trait`). This pattern of prefixing functions with colors is very common and is very natural in English as the color plays the role of an adjective, describing the kind of function one has.

Although the RFC is focused on syntax, it does also suggest some equivalences that colors must maintain, such as how a `K fn foo() -> T { expr }` can be transformed to `fn foo() -> 🚲K<T> { K move { expr } }` for any rewrite color `K`. These equivalences hint at an underlying semantics for colors. There are two related precedents in academia:

* [Haskell]'s monads are comparable in some ways to rewrite colors, and the [monadic laws][] suggest similar equivalences. A "[monad]" is like a color made up of two operations, one that "wraps" a value `v` to yield a `Wrapped<V>` (analogous to `K-type!(v)`) and one that does an `and_then` operation, often called "fold", taking a `Wrapped<V>`and a `V => Wrapped<W>` closure and yielding a `Wrapped<W>` value. The effect is kind of like a "programmable semicolon", allowing [Haskell] programs to introduce operations in in between each statement, and to potentially carry along "user data" with the wrapped `V` value. Haskell includes generic syntax ("do"-notation) designed to work with monads, allowing one to define generic functions that can be used with any monad `Wrapped`.
    * Monads are famously challenging to learn ("The sheer number of different monad tutorials on the internet is a good indication of the difficulty many people have understanding the concept", says the Haskell wiki). Colors run the risk of being similarly complicated. However, this RFC is not introducing colors as a concept that users would be taught, but rather as a syntactic pattern that they will pick up on as they learn about different existing parts of Rust (async, unsafe, etc).
    * Ergonomic challenges with monads often arise when composing or converting between monads. These same problems arise with colors (e.g., how to call an async function from a sync function), but those problems already exist and are not made worse by this RFC.)
* Effects in languages like [Koka][] can be compared to colors. Effects have evolved over the years from a way of signalling side effects (from which the name derived) to an expressive construct for injecting new operations into functions. An effect in [Koka][] is a function that is threaded down through context, the usage of which must be declared. Because [Koka][] is based on [continuation passing style][cps], effect functions are able to abort computation (modeling exceptions) or pause and resume it (modeling generators).

[Koka]: https://koka-lang.github.io/koka/doc/index.html
[Haskell]: https://www.haskell.org/
[monadic laws]: https://wiki.haskell.org/Monad_laws
[monad]: https://wiki.haskell.org/All_About_Monads
[cps]: https://en.wikipedia.org/wiki/Continuation-passing_style


# Unresolved questions
[unresolved-questions]: #unresolved-questions

## What syntax to use for `🚲K<$T>`?

This question is purposefully left unresolved by this RFC and is meant to be addressed in follow-up RFCs, such as [RFC #3628][].

# Future possibilities
[future-possibilities]: #future-possibilities

## Expanding on our existing colors

This RFC suggests a number of future changes to "fill out" the support for colors:

* Async should add a `🚲async<T>` syntax (see [RFC #3628][]).
* We will eventually need a mechanism for defining async-colored traits like `async Iterator` or `async Default`. For the short term this is not urgent as the traits we are focused on (such as `Fn` and `Drop`) are special.
* Const-colored traits should use a notation like (e.g.) `const Default`.
* Unsafe-colored traits like `unsafe Default`, which would mean a `Default` trait with an unsafe `default` method, are a possibility, but they could be easily confused for an unsafe trait.

## More complex colors

In authoring this RFC, we also explored what future colors might look like. Three potential future colors are `unsafe`, `try` (for fallible functions that yield a `Result`, most commonly at least) and `gen` (for generators). `try` and `gen` would be rewrite colors.

Unlike `async` and `const`, these three keywords are not themselves  *colors* but more like *families of colors*. `unsafe` for example indicates that the function has safety predicates that must be proven before it can be called and hence the "true color" conceptually includes those predicates (though we don't write them explicitly in our notation). `try` and `gen` both have other types associated with them beyond the main output type.

### `unsafe` as an effect-carrying color

`unsafe` can be considered an effect-carrying color, although it has semantic differences from an effect like `runtime`. `unsafe` doesn't itself indicate some kind of action that takes place but rather the requirements to safely use a piece of code. Furthermore the `unsafe` color, despite being 1 keyword, is really a "set" of colors, in that each `unsafe fn` has its own safety requirements that are distinct from one another.

If we decided to treat `unsafe` as a color, we would add the concept of an `unsafe Trait`, which would be a version of `Trait` where some items are `unsafe`. This could be used for `unsafe` closures, which would implement `unsafe Fn`, and which would signal to their callers that they require special requirements to be invoked. Rust already has a concept like this in the form of types like `unsafe fn()`. Unlike a type like `fn()`, the `unsafe fn()` type does not implement the `Fn` trait because there is no way to reflect that unsafety in the trait.

There is a complication to considering `unsafe` as a color because we already have a notion of an `unsafe` trait, with a distinct meaning. An unsafe trait is one where implementors prove a safety predicate relied on by the caller, as opposed to an `unsafe`-colored trait, which is a trait where the caller proves a safety predicate relied upon by the impl.

It's also worth noting the `unsafe_op_in_unsafe_fn` lint, which encourages the use of `unsafe` blocks even within an `unsafe` function. This is likely to be different from other effect-carrying colors that could be added (e.g., perhaps one that tracks whether allocation or panics can occur in a function). This difference stems in part from the fact that `unsafe` is not tracking an "effect" that occurs at runtime but rather an aid to static reasoning.

### `try` as a color

The `try` keyword was introduced in [RFC #243][], along with the `?` operator. It has been unstable since [RFC #243][] was merged in 2016 and has seen significant evolution.

As described in [RFC #243][], a `try { $expr }` block executes `$expr` and "captures" any `?` operations that occur within. On successful completion, the result of the try block is not the result of `$expr` but rather an "ok-wrapped" variant. The term "ok wrapping" refers to the common case where the `try` block produces a `Result`; the idea is that `try { 22 }` would be equivalent to `Ok(22)`. During the course of `try`'s long history, the design [temporarily changed *not* to `Ok`-wrap][#41414]. However, [the original design was ultimately restored][#70941].

[#41414]: https://github.com/rust-lang/rust/issues/41414
[#70941]: https://github.com/rust-lang/rust/issues/70941

Looking at `try` as a color, it is clear that ok-wrapping is the correct and consistent behavior. `try`, like `async`, is a fully general color that transforms its result type, and the color pattern specifies that the type of an expression in a block should be "ok-wrapped" to produce the block's final output type. Another point of controversy about `try` has been how to support it syntactically at the function level. This RFC suggests that the right answer is `try fn foo() -> T`, where `T` represents the "ok" type. The final result of calling `foo()` would therefore be `🚲try<T>`, the ok-wrapped version of `T`.

While treating `try` as a color is appealing, it does have one important difference from `async`: `try` and `?` are intended to be usable with many types, including `Result`, `Option`, and others. In terms of the color pattern, it is therefore unclear what the syntax `🚲try<T>` should expand to.

[`Try`]: https://doc.rust-lang.org/std/ops/trait.Try.html

One possibility is permitting the user to explicitly declare (and important) `try` colors themselves in the form of a type alias. For example, the `std::io::Result<T>` pattern might instead be:

```rust
// Older alias, deprecated:
type Result<T> = Result<T, std::io::Error>;

// in std::io, we define `try` as an alias to transformed type,
// given the Ok type:
type try<T> = Result<T, std::io::Error>;
```

Users could then import `try` from `std::io` and use it:

```rust
use std::io::try;

try fn load_configuration_string() -> String {
    std::fs::read_to_string("config.txt")?
}
```

This would be equivalent to the following uncolored function returning `🚲try<T>`:

```rust
use std::io::try;

fn load_configuration_string() -> 🚲try<String> {
    try { std::fs::read_to_string("config.txt")? }
}
```

This is only one possibility. There are several other ideas for how try could be integrated:

* Define `type try = Result<!, std::io::Error>` as an alias for the residual and use `try -> T` to mean `impl Try<Output = T, Residual = X>` (this assumes `🚲try<T>` is defined as `try -> T`).
* Extend `try fn` with a clause like `try fn foo() -> T throws X` and have it expand to `impl Try<Output = T, Residual = X>` or something like that.

### `gen` as a color

The `gen` keyword has been proposed for introducing generator blocks, which are a syntactic pattern to make it easier to write iterators (and perhaps to fill other use cases, there is a range of design space still to be covered). One notation proposed for `gen` is `gen<T>`, which appears at first glance to resemble the proposed `async<T>` type notation from [RFC #3628]. However, as proposed, `gen<T>` is actually quite different, as the `T` here represents the type of value that is yielded by the iterator. Therefore the generator may produce any number of `T` instances, whereas `async<T>` produces exactly one `T` instance. This means that they are used very differently by users, and it is unclear whether what parts of the color pattern should apply to `gen` in this case.

That said, there is some precedent for treating "generators" as color-*like* things. In Haskell, the `List` monad performs implicit `flat_map` operations, meaning that a given piece of code may execute many times. In Rust terms, this would correspond to nested `for` loops. Consider the following Rust-like pseudocode:

```rust
gen fn even() yields u32 {
    // In Haskell-like pseudocode, this might be something like
    //
    // do
    //   i <- all()
    //   if i % 2 == 0 { return i }
    //
    // Here, the fact that there is a for-loop is "hidden" in the
    // monad itself, and not apparent in the `do`.
    //
    // In contrast, under the proposed gen designs, the `for` loop
    // in Rust code would be explicit, not something that happens
    // via an implicit mechanism:
    for i in all() {
        if i % 2 == 0 {
            yield i;
        }
    }
}

gen fn all() yields u32 {
    for i in 0.. {
        yield i;
    }
}
```

This understanding of gen does not map well to colors. Compare to `async`: it makes total sense to define an `async Default` trait -- that is an `async`-colored version of `Default` where the `default` method is an `async fn`. But what does `gen Default` mean: the `default` method returns a generator yielding `Self` values? Is that a useful transformation to be able to make?

Another way to look at `gen` is to look at yielding as a kind of *side effect* that the function performs along the way to computing its final value. The iterator trait doesn't have a final value (or, equivalently, only has a value of `()`), but other generators do have this distinction (a `Future<Output = T>` can be seen as a generator that yields up `()` values whenever it needs to suspend and produces a `T` once complete, for example). Looking at `gen` from this perspective, the generator "color" is not just a keyword but includes the type of value that will be yielded as the generator executes. The `🚲try<$ty>` definition for a generating yielding values of type `Y` is therefore `impl Generator<Yield = Y, Output = $ty>`. Given a generator, a `yield_all` operator could be defined (analogous to `await` for `async` or `?` for `try`) that takes every item from the generator `v` and yields it in turn, before returning the final output.

Here is an example using the hypothesized "yield all" construct, showing how you could work with `gen` as a color. In this case, the `yield_all` 'unwraps' the generator effect, propagating the yielded values, and returning the final value. Calling `outer` would then yield up [`0`, `1`, `2`, `6`, `7`, `8`] and return 42. Note how `.yield_all` appears at the same places that `.await` would appear if these were async functions:

```rust
gen(yields u32) fn outer() -> u32 {
    // Yields 0, 1, 2 and returns 6 = (0 + 1 + 2) * 2
    let mid = starting_at(0).yield_all;

    // Yields 6, 7, 8 and returns 42 = (6 + 7 + 8) * 2
    starting_at(mid).yield_all
}

gen(yields u32) fn starting_at(x: u32) -> u32 {
    let mut sum = 0;
    for i in x .. x+3 {
        sum = sum + i;
        yield i;
    }
    sum * 2
}
```

It is unclear whether this perspective of `gen<Yields = T>` as a color is a useful one for users (and that syntax is not the most elegant).