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

# Summary
[summary]: #summary

This RFC unblock the stabilization of async closures by committing to `K $Trait` (where `K` is some keyword like `async` or `const`) as a pattern that we will use going forward to define a "K-variant of `Trait`". This commitment is made as part of committing to a larger *syntactic design pattern* called "`K`-color". The `K`-color pattern codifies how a "color keyword" `K` (e.g., `async`) can be used in various contexts within Rust contexts with consistent syntax:

* `K fn $name() -> $type` to define a K-function (e.g., async, const, etc)
* `K [move] { $expr }` to define a K-block
* `K $Trait` to reference the K-variant of a trait `$Trait`
* `K [move] |$args| $expr` to define a K-closure (which implements a `K Fn{,Mut,Once}` trait and returns the result of a `K move { $expr }` block)
* a still TBD syntax `🚲K<$T>` that indicates the "colorized" result type that you get from executing a K-block or calling a K-function or K-closure

The RFC uses this color pattern to describe two existing keywords, `async` and `const`. The RFC includes (non-binding) recommendations for future language work that would help the pattern to apply more fully. The future work section explores how other existing or planned features could be included in this framework.

This RFC itself does NOT define any particular semantics nor add new features. Instead, it describes a latent syntactic pattern that exists in Rust today, expands on that pattern to reach a logical conclusion, and commits to that pattern going forward. This RFC does not commit us to adding any particular feature but it does define the syntax that we would use for some future features that are under consideration and it suggest other features we might consider adding (and constrains the syntax we would use if we were to do so).

# Motivation
[motivation]: #motivation

The Rust Project has a [flagship goal][asyncfg] of bringing the async Rust experience closer to parity with sync Rust, and one of the most important deliverables for that goal is [async closures][RFC #3668]. The only unresolved question in the async closure RFC is the syntax for async closure bounds, which could be spelled like `F: async Fn()` or `F: AsyncFn()` (or many other ways). For reasons covered more in depth in that RFC, neither of these is an obvious choice given the Rust we have today.

## Consensus: `async Fn` is nice, but only as part of a larger design pattern

The lang team discussed the syntax question [in a design meeting][DM] and concluded that we prefer offering the `async Fn` syntax as a means of requesting the "async version of a trait", but we would want it to be part of a consistent **syntactic design pattern** for selecting variants of traits.

[DM]: https://hackmd.io/@rust-lang-team/rJxAOyWaC

In other words, if users wrote `async Fn` to get an async version of a closure but `AsyncDefault` to get the async version of the `Default` trait, that would be confusing. In contrast, if the pattern to get the async version of a trait is always to write the `async` keyword (e.g., `async Fn` and `async Default`), that is appealing.

We also observed that there is a need for having "variants of traits" in other similar contexts, most notably `const`, where there is [active experimentation for a const-trait design][#67792]. If we further extend the pattern to cover those cases, so that one writes (e.g.) `const Fn` or `const Default` to the "const versions of the `Fn` or `Default` traits" respectively, then these two instances of the same pattern reinforce one another, helping to create a coherent whole. Any future instances we add would only strenghten this effect.

## Tenet: syntactic consistency has value even if semantics diverge

The heart of this RFC is defining a **syntactic design pattern** called the *color* pattern. The pattern is defined by a series of rules that specify what syntax to use for each place the color can be applied. The term color is a playful reference to the famous ["What color is your function?"][WCIF] blog post.

[WCIF]: https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/

A *color* as defined by this RFC is a keyword `K` that can be applied to (at least) blocks and functions. The characteristic of a color is that code with color `K` is typically only usable from other code with color `K`. The RFC explores two colors in depth (`async` and `const`) and includes discussion of other potential colors in the [future possibilities](#future-possibilities) section. 

The premise of this RFC is that color keywords like `const` and `async` "feel similar" to users and should be used in syntactically similar ways, even though their semantics diverge. The goal is to make these colors, and any future "color-like" features, feel as consistent and predictable as possible, so that users can develop "muscle memory" that helps them to predict syntax they haven't used or seen yet.  The RFC describes the syntax to use for a color keyword applied to various constructs as well as some of the "approximate equivalences" that colors should generally hold.

## This RFC says what syntax to use when adding features; it doesn't add them

The goal of the RFC is NOT to define particular semantics for any particular color. Rather, it defines syntactic patterns that we should use going forward when we have "color-like" features. For example, if we wish to have a colorized version of a trait, the RFC specifies that this should be done with the syntax `K $Trait`, where `K` is the color keyword.

**The RFC does not require that all colors be usable on traits:** for the moment, the only "colorized" traits would be the `async Fn` traits defined by [RFC #3668][]. However, the RFC would be binding on future async traits: e.g., if we define an async version of the `Drop` trait in the future, it should be referred to as `async Drop`; when/if we gain support for const-ified traits, they should be written `const $Trait`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This section introduces the rules of the "color pattern" more precisely as a series of five steps:

1. When is a keyword `K` a color?
2. `K`-colored functions, blocks, traits, and closures
3. `K`-colored types
4. Relationship between `K`-colored functions, blocks, closures, and types

The "color pattern" is meant to guide Rust designers as we extend the language, not to be something directly taught to end users. To illustrate how it might feel, we also include an example of teaching `async` leveraging the syntax and concepts of the color pattern (but not teaching the pattern explicitly).

## Defining the "color pattern"

### When is a keyword `K` a color?

We begin by defining the criteria where we, as language designers, should consider using the color pattern for a keyword `K`. Given the color patterns informality, there is no hard and fast rule for this, but the primary criteria to consider are as follows:

1. The keyword `K` should be applicable to functions and blocks of code.
    * `const fn foo()`, `const { .. }`
    * `async fn foo()`, `async { .. }`, `async move { .. }`
2. `K`-colored code should generally only be usable with other `K`-colored code.
    * `const` functions can only call other `const` functions
    * `async` functions return futures that can only be awaited from async blocks

Colors can encompass a broad range of keywords with diverse effects. Both `async` and `const` fit the definition for a color above despite being very different on a semantic level. The other current keyword in Rust that could be considered a color is `unsafe`; we explore what it might mean to use the color pattern for `unsafe` in the [Future Possibilities](#future-possibilities) section.

### `K`-colored functions, blocks, traits, and closures

The syntax to use when applying colors to functions, blocks, traits, and closures is as follows:

* `K fn $name($args) -> $ty { $expr }` for functions
* `K { $statements; $expr }` (and in some cases `K move { $statements; $expr }`) for blocks
* `K $Trait` for traits (generalizing the precedent set by [RFC #3668][])
* `K |$args| $expr` for closures (generalizing the precedent set by [RFC #3668][])

The first two rules are the way that `async` and `const` are used today, and they both demonstrate the precedent that color keywords `K` are applied as a *prefix* to the things they modify. The second two rules are generalizing the precedent set by [RFC #3668][], which introduces the `async Fn` traits and  the `async ||` closure syntax. 

#### What is new in this RFC

1. Resolving the unresolved question from [RFC #3668][] in the affirmative (yes, we will use the `async Fn` syntax).
2. Committing to extend that syntax to any other colors, notably `const`, that may be applied to traits in the future.

#### What this RFC does NOT specify

This RFC should NOT be understood as adding any form of `K`-colored traits. For now, the only example of a `K`-colored trait are the `async Fn*` traits introduced by [RFC #3668][]. This RFC does not permit users to name an `async`-colored version of any other trait. But it does specify that if we support `async`-colored versions of other trait, they should use the syntax `async $TraitName`. For example, if and when we add support for an async version of the `Drop` trait, it should be named `async Drop`. Furthermore, if and when an RFC emerges from thej [the ongoing experimentation with const traits](#66792), that RFC should use the syntax `const $Trait`.

### `K`-colored types

The `async` color is different from `const` in that it is a **rewrite color**. Code colored by a rewrite color executes differently than ordinary code and produces a value of a different type. So `async { 22 }` does not produce an `i32` but rather a `impl Future<Output = u32>`. We expect there to be future colors (e.g., `try`, described under [future possibilities](#future-possibilities)) that operate in a similar fashion.

Users writing `async fn` sugar don't have to write out the `impl Future` syntax directly, but they do sometimes encounter it nonetheless. One reason might be writing a function that takes a future as argument. Another common reason is to write an async function that performs some actions synchronously and others when awaited:

```rust
fn do_something() -> impl Future<Output = ()> {
    // ... actions take here execute immediately ...
    async move {
        // ... actions here execute when future is awaited ...
    }
}
```

Having users write `impl Future<Output = T>` explicitly in cases like this has downsides. To start, it is verbose. But also, as a new user of Rust, it requires understanding an unrelated concept (`impl Trait` syntax) in order to write async code.

To address these issues, [RFC #3628][] proposes adding a new syntax for "the type that results from an `async` block", but that RFC has not been accepted. Accepting the RFC would require answering two questions:

1. Does it make sense to add some syntax beyond `impl Future<Output = T>` for the result of an async block?
2. If so, precisely what syntax should we use?
    * The RFC proposes `async<T>`, but other syntaxes have been proposed, such as `async -> T`, `impl async -> T`, and `impl Future -> T`.

Similarly to the question of `async Fn` syntax, members of the lang team felt that the case for [RFC #3628][] was stronger if the syntax was not something specific to `async` but rather part of a broader pattern used by multiple colors. Therefore, this RFC proposes to settle this first question ("should we add a syntax beyond `impl Future`") in the affirmative. However, we do not yet know what syntax we would like (see [future possibilities](#future-possibilities) for more considerations) and therefore we are leaving the precise syntax TBD to be specified by a future RFC.

#### What is new in this RFC

This RFC commits us to adding, for every rewriting color `K` (currently only `async`), **some (TBD) syntax `🚲K<$ty>` for describing the result of a `K`-colored block**. This syntax should involve the keyword `K` in some form. The behavior of this syntax may vary depending on where it is used, just like `impl Trait`. The notation `🚲K<$ty>` will be used in this RFC as a placeholder for a precise syntax to be specified in some future RFC.

The type `🚲K<$ty>` relates to the type of functions and closures as follows:

1. A `K`-function `K fn foo() -> T { $expr }` has a body that returns a value of type `T`. When called, however, its caller receives a `🚲K<T>` value.
2. A `K`-closure `K |$args| -> T { $expr }` has a body that returns a value of type `T`. It implements the trait `K Fn<Output = T>`. This trait ensures that when the closure is called its caller receives a `🚲K<T>` value.

#### Example for async

`🚲async<$ty>` is therefore defined as `impl Future<Output = $ty>`. Note that `🚲async<$ty>` is not a Rust type alias but a syntactic substitution, and therefore the `impl` keyword here has a different role depending on where it appears (in [function arguments][apit], it corresponds to a new generic argument to the fn; in [return position][rpit], it corresonds to an [abstract return type][rpit], etc).

[apit]: https://doc.rust-lang.org/stable/reference/types/impl-trait.html#anonymous-type-parameters
[rpit]: https://doc.rust-lang.org/stable/reference/types/impl-trait.html#abstract-return-types

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

To ensure consistency between colors, this general relationship should hold for future rewrite colors: 

* A `K fn foo() -> T` should be equivalent to a (uncolored) `fn` that returns `🚲async<T>` and uses some form of `K`-colored block.

## Teaching async via the "color pattern"

> The "color pattern" is meant to guide Rust designers as we extend the language, not to be something directly taught to end users. To illustrate how it might feel, this section covers an example of teaching `async` leveraging the syntax and concepts of the color pattern (but not teaching the pattern explicitly). To help illustrate where the overall vision for Rust, we assume that `🚲K<$T>` is `K -> $T` (a variant of [RFC #3628][] currently preferred by the author), that `async -> T` and its equivalent `impl Trait<Output = T>` are supported in local variable declarations ([RFC #2071][]), and that there is some form of `async Iterator` trait available in std.

## Async functions

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

## Async blocks

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

### `async` vs `async move`

Async blocks resemble closures in some ways. Just as a Rust closure by default takes references to variables from the surrounding stack frame, async blocks yield futures that store references to the variables they need whenever possible. You can however use the `async move` syntax to force the future to take ownership of any data that is uses.

## Async closures

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

## Desugaring async functions into async blocks

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

## Comparing async-await in Rust to async-await in other languages

Many languages have some variant of async-await notation and so async functions in Rust likely look familiar to you. That's good! But be aware that async-await works differently in Rust than most other languages.

The most obvious difference is that where most languages put `await` as a prefix, Rust makes it into a suffix. So instead of writing `await foo` you write `foo.await`. This is more convenient when writing chained operations, like `data.load().await.load_more().await`, and especially when dealing with futures that yield `Result`, as one can use `?` like `load_data().await?`.

The other, more subtle, difference has to do with Rust's execution model. In most languages, calling an async function implicitly starts up a background task that will continue executing. Rust, like Kotlin, takes a different approach: calling an async function returns a suspended computation, which on its own is inert. That future can then by combined with other futures to form aggregates, like the `join!` operation that we saw in the previous example. Eventually, the future or the aggregate that it is embedded into must be awaited -- and, when it is, that will block your current task until it completes.

If you'd like the async functon to execute in the background, then you need to use a `spawn` function to create a new task (analogous to spawning a new thread in synchronous code). Rust itself does not provide a spawn function, but one is typically provided by your async runtime. The most common choice here is `tokio`, which offers [`tokio::spawn`](XXX):

```rust
let data_future = tokio::spawn(async move { load_data(urls).await });
// ... `data` is not being loaded in the background, in parallel ...
```

## Digging deeper: the `Future` trait

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

### `unsafe` as a color

`unsafe` can be considered a color somewhat like `const` (or, more precisely, like the imaginary [`runtime` color](#const-functions-cant-be-rewritten-to-have-a-const-block-in-their-body-is-that-inconsistent-with-async)) described in the FAQ). Because unsafe blocks execute in the same way as ordinary blocks, it would not be a rewrite color.

Considering `unsafe` as a color would suggest supporting `unsafe`-colored traits like `unsafe Default`. This would indicate a type for which creating a `Default` value is unsafe.

There are a few complications to considering `unsafe` as a color:

* First, there is not just one notion of unsafe. Consider `unsafe Default`: presumably every type implementing `unsafe Default` has its own safety predicate that must be proven before its `unsafe Default` impl could be used. This implies that the "true" color is something like `unsafe(predicate)`, where the `predicate` is implied and unwritten.
* Second, we already have a notion of unsafe traits with a different meaning. An unsafe trait is one where implementors prove a safety predicate relied on by the caller, as opposed to an `unsafe`-colored trait, which is a trait where the caller proves a safety predicate relied upon by the impl.
* Third, the `unsafe_op_in_unsafe_fn` lint is moving from allow-by-default to warn- or deny-by-default. This means that migrating `unsafe fn foo() { ... }` is not exactly equivalent to `unsafe fn foo() { unsafe { ... } }`, as we might expect for [non-rewrite colors](#const-functions-cant-be-rewritten-to-have-a-const-block-in-their-body-is-that-inconsistent-with-async).

### `try` as a color

This RFC suggests that, to be stabilized, `try` blocks should become a rewrite color. This would mean extending to `try fn` and `🚲try<T>` notation.

The big challenge for `try` as a color is that `?` and try blocks can work with so many different types. `🚲try<T>` could be a result, an option, etc, and it will not be the same across all programs, as e.g. error types vary.

One possibility that we discussed was permitting the user to explicitly declare (and important) `try` colors themselves in the form of a type alias. For example, the `std::io::Result<T>` pattern might instead be:

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

The `gen` keyword has been proposed for introducing generator blocks, which are a syntactic pattern to make it easier to write iterators (and perhaps to fill other use cases, there is a range of design space still to be covered). One notation proposed for `gen` is `gen<T>`, which appears at first glance to resemble the proposed `async<T>` type notation from [RFC #3628]. However, as proposed, `gen<T>` is actually quite different, as the `T` here represents the type of value that is yielded by the iterator. Therefore the generator may produce any number of `T` instances, whereas `async<T>` produces exactly one `T` instance. This means that they are used very differently by users, and it is unclear whether they should be considered a "color" -- for example, does `gen Default` make sense? If so, what would it mean?

There is some precedent for treating "generators" as color-*like* things. In Haskell, the List monad performs implicit `flat_map` operations, meaning that a given piece of code may execute many times. In Rust terms, this would correspond to nested for loops. Consider the following Rust-like pseudocode:

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

This understanding of gen does not map well to rewrite colors. Compare to `async`: it makes total sense to define an `async Default` trait -- that is an `async`-colored version of `Default` where the `default` method is an `async fn`. But what does `gen Default` mean: the `default` method returns a generator yielding `Self` values? That is quite a different trait and doens't feel like a "color", which is basically a function that is only compatible with other functions of the same color.

There is however a way to make `gen` fit well into the color system. Generators can be thought of as a color if the yielded values are considered a side channel. Like `try`, then, the generator "color" is not just a keyword but includes the type of value that will be yielded as the generator executes. The `gen-type!($ty)` definition for a generating yielding values of type `Y` is therefore `impl Generator<Yield = Y, Output = $ty>`. The `gen-do!(v)` operation becomes "yield-all", taking every item from the generator `v` and yielding it in turn, before returning the final output.

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