# Why Lifetimes Are Mostly Never Used

In the last chapter, we saw why we needed lifetimes. We saw that the compiler was unable
to automatically tell which inputs to the function might be used to produce references,
and therefore it wasn't sure how long the outputs of functions needed to live for.

This said, you've probably written a function in Rust that needed a reference (likely a `&str`).
Why didn't you have to annotate lifetimes then? There are some common patterns in rust that
make it obvious to the compiler what the lifetimes should be. Let's explore some of them:

## Example 1: No Output References

``` rust
fn add(a: &i32, b: &i32) -> i32 {
    *a + *b
}

# fn main() {
#     assert_eq!(add(&3, &4), 7);
# }
```

The lifetimes of `a` and `b` in this function don't need to relate to eachother. Assuming there's only one thread,
and assuming safe code, there's no way that the variable they are referencing could possibly be dropped during the
function, and after the function, they can live for as long as they like.

## Example 2: Only one reference in the input


``` rust
fn identity(a: &i32) -> &i32 {
    a
}

# fn main() {
#     let x = 52;
#     assert_eq!(&x, identity(&x));
# }
```

Remember that it (generally) isn't possible to create a reference and pass it out of a
function if it wasn't given to you. If you're giving it away, but you created it, who
owns the underlying data?

(footnote: this is possible with static types, like string literals, but we'll cover those later)

For this reason, if you only have one reference in your parameters, the only reference you
could return is that one -- so the lifetime of your parameter has to be the same as
the reference you return.

# What to do

Rust could have specifically encoded these examples as exceptions; but then
there may have been many cases which were excepted and they would have
ended up with confusing rules.

Instead, the Rust project has settled on a procedure the compiler will follow to try and guess lifetimes.

The compiler first splits all the references in a function signature into two types: 'input'
references are those in the parameters of the function (i.e. it's arguments). 'output' references
are those in the return type of the function.

The two rules that we'll learn in this chapter are:

1. Each place that an input lifetime is left out (a.k.a 'elided') is filled in with it's own lifetime.
2. If there's only one lifetime on all the input references, that lifetime is assigned to all output lifetimes.

Let's see how those rules affect the above two examples, and an example from the last chapter:

## Example 1: No Output References

We had:

``` rust,ignore
fn add(a: &mut i32, b: &mut i32) {
    *a + *b
}
```

There are two input lifetimes, the ones needed in the types of `a` and `b`. Each get allocated their own lifetime:

``` rust
fn add<'elided1, 'elided2>(a: &'elided1 i32, b: &'elided2 i32) -> i32 {
    *a + *b
}

# fn main() {
#     assert_eq!(add(&3, &4), 7);
# }
```

There are no output lifetimes, so we end there.

This example is now correct -- the two lifetimes of `a` and `b` can be entirely unrelated;
and the output is an owned value, so it doesn't depend on any lifetimes at all.

## Example 2: Only one reference in the input

We had:

``` rust
fn identity(a: &i32) -> &i32 {
    a
}

# fn main() {
#     let x = 52;
#     assert_eq!(&x, identity(&x));
# }
```

There is only one input lifetime (needed for the type of `a`):

``` rust
fn identity<'elided1>(a: &'elided1 i32) -> &i32 {
    a
}

# fn main() {
#     let x = 52;
#     assert_eq!(&x, identity(&x));
# }
```

And there is only one output lifetime; but all the input lifetimes share the same lifetime (`'elided1`);
therefore we can allocate all output lifetimes that lifetime:

``` rust
fn identity<'elided1>(a: &'elided1 i32) -> &'elided1 i32 {
    a
}

# fn main() {
#     let x = 52;
#     assert_eq!(&x, identity(&x));
# }
```

This now makes sense: the only possible way you could return a `&i32` is if you got it from a parameter;
and we can see that the input and output share a lifetime.

## Example 3: The Limits of Elision

Let's have another look at this example from the last chapter.

``` rust,ignore
fn max_of_refs(a: &i32, b: &i32) -> &i32 {
    if *a > *b {
        a
    } else {
        b
    }
}
```

Just like Example 1, there are two input lifetimes, so we give them
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

But unlike Example 1, there are now two input lifetimes! The second rule
only works if we have one input lifetime. Therefore, rust considers it an
error to elide lifetimes here -- the user has to give more information!

## Exercise: Apply These Rules

In this exercise, there are four functions which have no manual lifetimes.
Your task is to manually follow the lifetime elision rules, and give these
functions lifetimes.

If you are unable to give them a lifetime, put `#[elision_not_allowed]`
above the function, and leave it unedited.