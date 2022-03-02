# Lifetime subtyping
`'a: 'b` reads as `'a is subtype of 'b`, but mixing types with
lifetimes is often confusing, so rusteceans prefer to say: `lifetime 'a outlives lifetime 'b`.
This `outlives` relationship implies 2 important things:

- It allows to implicitly cast references with `'a` lifetime into references with `'b` lifetime.
- The compiler must assert that `'a >= 'b` (region `'a` is the same or wider than region `'b`)

Given that `'a >= 'b` is required and that we're allowed to implicitly cast `'a` references into `'b` references,
I will call these implicit casts - lifetimes shortenings and will denote them like that: `'a ~> 'b`. Let's go through
some examples to get used to these concepts.

Let's start from this pseudocode:
```
given 'a: 'b

ref_a: 'a
ref_b: 'b

ref_b = ref_a // fine, 'a ~> 'b
ref_a = ref_b // not fine, requires 'b: 'a
```

We're given `'a: 'b` and we have 2 references: `ref_a` belonging to the region `'a` and `ref_b` belonging to the region `'b`.
`'a: 'b` implies `'a ~> 'b` allowing us to assign `ref_b` to `ref_a`. By doing that we're sort of moving a reference from a longer `'a` 
region into a shorter `'b` region and this allows us to do some stuff with it on a behalf of the `'b` region. However, assigning `ref_a` to `ref_b` 
results in a compile error. It requires a `'b: 'a` precondition to cast `'b` ref into `'a` ref, but we only have a `'a: 'b` precondition.

In the previous chapter we used a visual approach to show how the borrow checker infers regions. In reality it doesn't work like that. All it does
is it assumes a new region for every line of code, and then it infers `outlives` relationships for these regions, and then executes validations based
on this information. When the borrow checker encounters a function call it doesn't try to be smart and infer anything, it just reads regions
and relationships between them directly from the function signature and assigns references to those regions. 
That means when we're annotating our signatures with lifetimes we're doing the great part of the borrow checker's work manually. To get a brief
feel of how the borrow checker actually operates we'll go through the next example written in real Rust. To make it readable I won't assume a new region
for every line of code, but I'll assume it for every scope:

```rust
{ // 'a
    let a = 42;
    let ref_a = &a; // ref_a belongs to 'a
    { // 'b. 'b is subscope of a', so `'a: 'b`
        let b = 24;
        let mut ref_b = &b; // ref_b belongs to 'b
        ref_b = ref_a; // 'a: 'b => 'a ~> 'b
        println!("{}", ref_b); // prints 42
    }

    println!("{}", ref_a); // prints 42
}
```

The example compiles just fine. It corresponds to the next lines of the pseudocode we met before and works for the same reasons. 
The only difference is `'a: 'b` relationship is not given, but inferred from the function scopes:
```
inferred 'a: 'b

ref_a: 'a
ref_b: 'b

ref_b = ref_a // fine, 'a ~> 'b
```

Now let's try this variation.

```rust
{ // 'a
    let a = 42;
    let mut ref_a = &a; // ref_a belongs to 'a
    { // 'b. 'b is subscope of a', so `'a: 'b`
        let b = 24;
        let ref_b = &b; // ref_b belongs to 'b
        ref_a = ref_b; // compilation error. No `'b: 'a` relationship
        println!("{}", ref_b); // doesn't compile
    }

    println!("{}", ref_a); // doesn't compile
}
```

This code corresponds to these lines of the pseudocode above and doesn't compile:

```
inferred 'a: 'b

ref_a: 'a
ref_b: 'b

ref_a = ref_b // not fine, requires 'b: 'a
```

We can't assign `ref_a` to `ref_b` because we didn't infer `'b: 'a` relationship. We inferred only `'a: 'b`, so knowing
that and by further inferring the region boundaries within the function scope compiler was able to produce a user friendly `b doesn't live long enough`
error. 

I hope at this point you've already guessed why `'a: 'b` relationship is required to be able to implicitly cast `'a` references into `'b` references.
In short, we just can't guarantee safety after the cast if `'a >= 'b` condition is not met. If you don't get why, study the last 2 examples carefully.

## Specifying lifetime relationships in signatures

Returning to our `post_urls_from_blog` example we had this error

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

```
   Compiling playground v0.0.1 (/playground)
error[E0623]: lifetime mismatch
  --> src/main.rs:11:6
   |
10 |     blog_url: &'blog_url str,
   |               -------------- this parameter and the return type are declared with different lifetimes...
11 | ) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
   |      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |      |
   |      ...but data from `items` is returned here

For more information about this error, try `rustc --explain E0623`.
error: could not compile `playground` due to previous error
```

The error is a bit tricky because the root cause lies at this particular dot:

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
    items.iter().filter_map(move |item| {
// here---------^ 
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

Let's examine what happens in between of the `iter()` and `filter_map()` calls. `iter()` returns an 
`Iterator` from `items` and this iterator is bounded by `'post_urls` lifetime. `filter_map()` takes the `items`
iterator, but also captures the `blog_url` with `'blog_url` lifetime in the closure, and we expect the resulting iterator to be bounded by
'`blog_url` lifetime. We can represent what's happening with the following function:

```rust
fn dot<'post_urls, 'blog_url>(
    iter: impl Iterator<Item = ()> + 'post_urls,
) -> impl Iterator<Item = ()> + 'blog_url
{
    iter
}
```

The function doesn't compile. The cast from `Iterator + 'post_urls` into `Iterator + 'blog_url` is prohibited because
`'post_urls` and `'blog_url` lifetimes are unrelated. In order to make the cast possible we need to introduce a relationship
between the lifetimes. We want to cast(shorten) `'post_urls` into `'blog_url` therefore we need a `'post_urls: 'blog_url`
relationship. Let's type it out. 

```rust
fn dot<'post_urls, 'blog_url>(
    iter: impl Iterator<Item = ()> + 'post_urls,
) -> impl Iterator<Item = ()> + 'blog_url
where
    'post_urls: 'blog_url
{
    iter
}
```

Now, with this additional bit of information the funciton does compile. The relationships between lifetimes
aren't inferred between the function calls, we need to specify them manually in order to apply casts we want
in the function body. Adding `where 'post_urls: 'blog_url` to `post_urls_from_blog` makes `items.iter()` cast
into `Iterator + 'blog_url` cast valid and the function now compiles for the same reason.

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url 
where
    'post_urls: 'blog_url
{

    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

But what if we had used `'blog_url: 'post_urls` relationship instead?

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'post_urls
where
    'blog_url: 'post_urls
{

    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

Now instead of casting `items.iter()` we're casting the borrow of the `blog_url` in the `filter_map` closure `'blog_url ~> 'post_urls`, so the
resulting iterator appears to be `Iterator + 'post_urls` as shown in the updated function signature and this signature compiles too.
What's the difference? To understand why this is not what we want we need to remember the second implication of `'a: 'b` relationship:

- The compiler must assert that `'a >= 'b` (region `'a` is the same or wider than region `'b`)

Let's return to the caller site and infer the regions for this signature. We will continue to use the visual approach introduced
in the previous chapter because even if it's not what compiler actually does it works quite well for humans.
Ok, so we need to infer 2 regions by _the last usage_ rule and we actually already did that in the previous chapter:

```rust,noplayground
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'post_urls region
||/-post_urls_from_blog 'blog_url region
||| let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||-
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
```

But now we have an extra precondition in our `post_urls_from_blog` function signature that `'blog_url` must be as wide as `'post_urls`
region or wider, so we need to extend `post_urls_from_blog 'blog_url` region to meet this requirement.

```rust,noplayground
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'post_urls region
||/-post_urls_from_blog 'blog_url region
||| let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
|||
||| let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-||
 || for url in post_urls {
 ||     process_post(url);
 || }
 --
```

As the result `post_urls_from_blog 'blog_url` and `blog_url` regions are not aligned and we have a conflict and the same compiler
error we were struggling with from the beginning. We know that the region for the iterator must be shorter because, usually, iterators
live less then the items they yield, but we failed to communicate this to compiler and our signature requires the region for the `Iterator` 
to be as wide or wider then the region for its items which is wrong, so we must stick with the `'post_urls: 'blog_url` relationship.
The regions for it will look as we want:

```rust,noplayground
/---blog_url region 
|   let blog_url = get_blog_url(); 
|
|/--post_urls_from_blog 'post_urls region
||/-post_urls_from_blog 'blog_url region
||| let post_urls: Vec<_> = post_urls_from_blog(crawler_results, &blog_url).collect();
||-
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
```

`post_urls_from_blog 'post_urls > post_urls_from_blog 'blog_url`, so no extra region expansion is required and everything compiles just fine.

Hope, this example was sufficient to show that lifetimes subtyping isn't a rocket science and is very straighforward to work with. The important
thing to remember is when you define a signature you manually specify how many regions the compiler needs to infer and 
what relationships are between them. The relationships come from the "casts" you want to perform in your function body, and specifying them
results in possible extra region expansions on the caller site. However, to have a full picture, we also need to learn about the lifetime variance.

## Chapter exercises

Analyze and write down the equivalent to the following signature: 

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
fn post_urls_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem], 
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'blog_url 
where
    'post_urls: 'blog_url,
    'blog_url: 'post_urls
{

    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```