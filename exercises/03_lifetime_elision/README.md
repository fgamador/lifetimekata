# Why Lifetime Annotations Are Rarely Used

In the last chapter, we saw why we needed lifetimes. We saw that the compiler
was unable to tell automatically how references in the arguments or return
values might relate to each other. This is why we needed to tell the compiler
how the references related to each other.

That said, you've probably written a function in Rust that used a reference
(likely a `&str`), without annotating its lifetime. Why didn't you have to do
so then? There are some common patterns in Rust that make it obvious to
the compiler what the lifetimes should be, so you can omit ("elide") them. Let's explore some of them.

## Example 1: No Output References

``` rust
fn add(a: &i32, b: &i32) -> i32 {
    *a + *b
}

// fn main() {
//     assert_eq!(add(&3, &4), 7);
// }
```

The lifetimes of `a` and `b` in this function don't need to relate to each other. Assuming there's only one thread,
and assuming safe code, there's no way that the variables they are referencing could possibly be dropped during the
function, and after the function, they can live for as long or as short as they like.

## Example 2: Only one reference in the input


``` rust
fn identity(a: &i32) -> &i32 {
    a
}

// fn main() {
//     let x = 52;
//     assert_eq!(&x, identity(&x));
// }
```

It's important to note that it isn't possible[1] to create a reference and pass it out of a
function if it wasn't given to you. This is because a reference must refer
to something you own, but everything you own is dropped at the end of your function.
Therefore, nothing you own can be referenced after the function ends, and the only reference you can return
is one you were passed.

For this reason, if you have only one reference in your parameters, the only reference you
could return is that one, so the lifetime of your parameter has to be the same as
the lifetime of the reference you return.

[1]: Actually, it is possible with static types like string literals, but we'll cover those later.

# What to do

Rust could have encoded these examples as specific exceptions, but there would have been many such exceptions, making the rules confusing.

Instead, the Rust project settled on a procedure that the compiler follows to guess lifetimes.

The compiler first splits all the references in a function signature into two types: *input* and *output*.
Input references are those in the parameters of the function (i.e. its arguments). Output references
are those in the return type of the function.

The two rules that we'll learn in this chapter are:

1. Each place where an input lifetime is omitted (elided) is given its own lifetime.
2. If there's exactly one lifetime across all the input references, that lifetime is assigned to *every* output lifetime.

Let's see how these rules affect the above two examples and then re-examine an example from the last chapter.

## Example 1: No Output References

We had:

``` rust,ignore
fn add(a: &mut i32, b: &mut i32) -> i32 {
    *a + *b
}
```

There are two input lifetimes, the ones needed in the types of `a` and `b`. Each gets allocated its own lifetime:

``` rust
fn add<'elided1, 'elided2>(a: &'elided1 i32, b: &'elided2 i32) -> i32 {
    *a + *b
}

// fn main() {
//     assert_eq!(add(&3, &4), 7);
// }
```

There are no output lifetimes, so we are done.

This example is now correct: the two lifetimes of `a` and `b` can be entirely unrelated,
and the output is an owned value, so it doesn't depend on any lifetimes at all.

## Example 2: Only one reference in the input

We had:

``` rust
fn identity(a: &i32) -> &i32 {
    a
}

// fn main() {
//    let x = 52;
//     assert_eq!(&x, identity(&x));
// }
```

There is only one input lifetime (needed for the type of `a`):

``` rust
fn identity<'elided1>(a: &'elided1 i32) -> &i32 {
    a
}

// fn main() {
//     let x = 52;
//     assert_eq!(&x, identity(&x));
// }
```

There is only one output lifetime, and all the input lifetimes share the same lifetime (`'elided1`),
so we can allocate all output lifetimes that lifetime:

``` rust
fn identity<'elided1>(a: &'elided1 i32) -> &'elided1 i32 {
    a
}

// fn main() {
//     let x = 52;
//     assert_eq!(&x, identity(&x));
// }
```

This now makes sense: the only possible way you could return a `&i32` is if you got it from a parameter,
and we can see that the input and output must share a lifetime.

## Example 3: The Limits of Elision

Now let's have another look at this example from the last chapter:

``` rust,ignore
fn max_of_refs(a: &i32, b: &i32) -> &i32 {
    if *a > *b {
        a
    } else {
        b
    }
}
```

As with Example 1, there are two input lifetimes, so we give them
distinct lifetimes:

``` rust,ignore
fn max_of_refs<'elided1, 'elided2>(a: &'elided1 i32, b: &'elided2 i32) -> &i32 {
    if *a > *b {
        a
    } else {
        b
    }
}
```

But unlike Example 1, we need an output lifetime! By the second rule, we can
elide the output lifetime only if we have exactly one input lifetime, but here we have two. Therefore,
Rust considers it an error to elide lifetimes here -- the user has to give more
information!

## Exercise: Apply These Rules

In this exercise, there are four functions that are missing some lifetime annotations.
Your task is to follow the lifetime elision rules manually and give these
functions lifetimes in the same way that the compiler would.

In a future release of lifetimekata, these will be checked automatically.
For now, when you're done, compare your answer to the solutions.
