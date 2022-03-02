# Borrow checking

The most important thing to understand about the borrow checker is it analyzes
each function completely independently(in isolation) from other functions. This
means when we encounter a call to our `posts_from_blog` the borrow checker
doesn't look inside it to validate the usage of references, all it does is it
reads the function signature and evaluates its lifetimes. But what does it mean
to evaluate a lifetime? Let's go back to our example and figure this out.


## Infering regions

The borrow checker analyzes the `main` function and encounters a line of code
with `posts_from_blog` function invocation.

```rust,noplayground
let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
```

First of all, the borrow checker looks at the function signature:

```rust,noplayground
fn posts_from_blog<'a>(
    items: &'a [DiscoveredItem],
    blog_url: &'a str,
) -> impl Iterator<Item = &'a str> {
    // We're looking at the function from the borrow checker's perspective.
    // It doesn't see the impl :(
}
```

`'a` is a generic lifetime. It is similar to a generic type `T` in a sense that
it acts like a placeholder the compiler needs to infer and validate statically
at every place we call the function. To do that the compiler must adhere the
following rules:

1. An inferred lifetime must be as small as possible.
1. References with this inferred lifetime must stay valid for the whole
   lifetime (no dangling pointers!)

Ok, but that still sounds vague. What exactly is a lifetime and what the
compiler is actually infering? Well, the lifetime is nothing more than a
continuous[^continuous] region of code(e.g. from line 3 to line 8). A region
can also be just a single line of code:

```rust,noplayground
    // Processing posts in parallel
    for url in post_urls {
/---process_post region. Holds: `&url`
|        process_post(url);
-
    }
```

_In the example above `url` reference is assigned to the `'process_post` region
which is inferred to be only 1 line long._

Region boundaries define where items assigned to the region can be accessed and
for references - where the borrows of the referents end.

Compiler infers regions for owned values and local references fully
automatically, you can reason about these inferences using Rust scoping rules
with only exception that references are getting dropped immediately after the
last use - not at the end of the scope.

```rust
fn f() {
/----num region lasts till num is moved out of scope or till the end of scope
|    let num = Num(42);
|/---ref_mut region ends right after the `.add` call(last use rule)
||   let ref_mut = &mut num;
||   ref_mut.add(1);
|-
|/---assert_eq region holds temp &num for the duration of assert call
||   assert_eq!(&Num(43), &num)
|-
-
}
```

However, when encountering another function call within the analyzed function
the compiler doesn't do any smart automatic inferences and safety checks, it
doesn't study the another function body, it doesn't track function inputs and
function outputs, it simply relies on generic parameters you defined in the
function signature to create regions for a particular call.

To study this, let's return to our main example and read the `posts_from_blog`
signature through regions lenses:

```rust,noplayground
fn posts_from_blog<'a>(
    items: &'a [DiscoveredItem],
    blog_url: &'a str,
) -> impl Iterator<Item = &'a str>;
```

_This signature reads as: "Dear Rust compiler, please infer a single region `'a` at the call site and assign to it the following references: `items`, `blog_url`, `impl Iterator`, `Iterator::Item`."_

Notice that the `impl Iterator` belongs to the region `'a` too, this is because
the complete return type actually looks like this: `impl Iterator<Item = &'a
str> + 'a`[^iterator] but the compiler elided the last `'a` according to
lifetime elision rules.

Now at the call site we need to infer a region `'a` for the `posts_from_blog`
such that the region is as small as possible for all 4 references it holds. How
wide the region should be? As wide as all references it holds must stay valid.
How long the references must stay valid? As long as they're used. So, the size
of a region is basically determined by the last used reference belonging to the
region.

Let's apply this rule in practice. Our function returns an iterator
`impl Iterator<_> + 'a`. How is it used?

```rust,noplayground
let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
```

Well, we just collect it immediately into a vector therefore, our region with respect of iterator
is a single line of code(the iterator is consumed and can't be used anywhere else):

```rust,noplayground
fn main() {
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];

    let blog_url = get_blog_url();

/---posts_from_blog 'a region
|   let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
-
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}
```

The consumed iterator yields references `Item=&'a str` that also belong to our region.
We store them in the `post_urls` vector. Now we need to find the last usage of those references.
It's here:

```rust,noplayground
    for url in post_urls {
        process_post(url);
    }
```

So `post_urls` references must be valid at least till the end of this loop. Expanding the region accordingly:

```rust,noplayground
fn main() {
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];

    let blog_url = get_blog_url();

/---posts_from_blog 'a region
|   let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
|
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
|
|   for url in post_urls {
|       process_post(url);
|   }
-
    handle.join().expect("Everything will be fine");
}
```

As for input arguments, usually they don't affect the region expansion because
they must be valid only for the duration of a function call, but later we will
study some cases when they do. Let's evaluate `items: &'a [DiscoveredItem]` and
`blog_url: &'a str` together. They're just regular input references without any
quirks, so they must be valid only at the line with the function invocation. If
we had started our analysis from input arguments, our region would look like
this:

```rust,noplayground
fn main() {
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];

    let blog_url = get_blog_url();

/---posts_from_blog 'a region
|   let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
-
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}
```

But we've already analyzed the outputs and know that our region must be wider,
hence the resulting `'a` region of the `posts_from_blog` function looks like
this:

```rust,noplayground
fn main() {
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];

    let blog_url = get_blog_url();

/---posts_from_blog 'a region
|   let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
|
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
|
|   for url in post_urls {
|       process_post(url);
|   }
-
    handle.join().expect("Everything will be fine");
}
```

The region holds: a copy of the `crawler_results` reference, a reference to
`blog_url`, a vector of `post_urls` references, and the consumed iterator(yes,
it's consumed at the first line of the region but accessing the iterator must
still be valid for the whole region scope). Note that we didn't analyze any
relationships between references. At this point we don't understand how inputs
and outputs are connected and where the references point to. All we did is we
inferred a region for them within which accessing __any__ of those references
must be possible.

That's it for the function inferences, the inference for structs containing
lifetimes works basically the same with the only addition that the usage of the
struct itself expands **all** regions defined in struct, e.g.:

```rust,noplayground
struct RefPair<'a> {
    a: &'a str,
    b: &'a str,
}

impl<'a> RefPair<'a> {
    fn concat(&self) -> String {
        format!("{}{}", self.a, self.b)
    }
}

fn main() {
     let x = String::from("X");
     let y = String::from("Y");
/----RefPair 'a region
|    let pair = RefPair {
|        a: x.as_str(),
|        b: y.as_str(),
|    }
|
|    println!("{}", pair.concat());
-
     println!("values: {x}, {y}");
}
```

And now if we move the last usage of the struct further down the main function
the `RefPair 'a` region gets expanded:

```rust,noplayground
fn main() {
     let x = String::from("X");
     let y = String::from("Y");
/----RefPair 'a region
|    let pair = RefPair {
|        a: x.as_str(),
|        b: y.as_str(),
|    }
|
|
|    println!("values: {x}, {y}");
|
|    println!("{}", pair.concat());
-
}
```


## Validating regions

It's time to ensure the safety. After we inferred all regions in the analyzed
function we need to explore relationships between __the regions__ (not
variables) looking for potential conflicts. Let's start at the point where the
`crawler_results` reference is copied to be passed as an argument to the
`posts_from_blog` function. Can we create this copy?

```rust,noplayground
fn main() {
/---crawler results region
|   let crawler_results = &[
|       DiscoveredItem {
|           blog_url: "https://blogs.com/".to_owned().to_owned(),
|           post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
|       },
|       DiscoveredItem {
|           blog_url: "https://blogs.com/".to_owned(),
|           post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
|       },
|       DiscoveredItem {
|           blog_url: "https://successfulsam.xyz/".to_owned(),
|           post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
|       },
|   ];
|
|   let blog_url = get_blog_url();
|
|/--posts_from_blog 'a region
||  let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
||
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
||
||  for url in post_urls {
||      process_post(url);
||  }
|-
|   handle.join().expect("Everything will be fine");
-
}
```

`crawler_resutls` region fills the whole `main` body clearly outliving our
`posts_from_blog 'a` region meaning we can dereference the copy of
`crawler_results` reference at **any** line within the `posts_from_blog 'a`
region (note that we use `crawler_results` borrow only at the line with the
function call, but it lasts till the end of the `posts_from_blog 'a` region
anyway).

Then we're taking a reference to `blog_url`. Can we do that?

```rust,noplayground
fn main() {
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];
/---blog_url region
|   let blog_url = get_blog_url();
|
|/--posts_from_blog 'a region
||  let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
||
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
    handle.join().expect("Everything will be fine");

}
```

No, we can't. We must be able to derefence this `blog_url` reference at **any**
line within the `posts_from_blog 'a` region, but there is no way of doing that
around the for loop because `blog_url` region ends(variable moves out of scope)
right before the loop.

Now we should be able to decipher the error message from the previous chapter:

```
error[E0505]: cannot move out of `blog_url` because it is borrowed
  --> src/main.rs:41:37
   |
35 |     let blog_url = get_blog_url();
   |         -------- binding `blog_url` declared here
...
38 |     let post_urls: Vec<&str> = posts_from_blog(crawler_results, &blog_url).collect();
   |                                                              --------- borrow of `blog_url` occurs here
...
41 |     let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
   |                                     ^^^^^^^                      -------- move occurs due to use in closure
   |                                     |
   |                                     move out of `blog_url` occurs here
...
44 |     for url in post_urls {
   |                --------- borrow later used here
   |
help: consider cloning the value if the performance cost is acceptable
   |
38 |     let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url.clone()).collect();
   |                                                                       ++++++++

For more information about this error, try `rustc --explain E0505`.
```

Effectively it tells us that at line 38 compiler tries to create a reference to
`blog_url` in a region that ends at line 44, but `blog_url` region ends at line
41, so the compiler can't do that. How can we fix this error? One way is to put
the for loop before `std::thread::spawn`. This way the regions will be aligned
and `blog_url` will be safe to use at any line of the `posts_from_blog 'a`
region. But our code is not executing in parallel this way.


```rust,noplayground
fn main() {
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];
/---blog_url region
|   let blog_url = get_blog_url();
|
|/--posts_from_blog 'a region
||  let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
||
||  for url in post_urls {
||      process_post(url);
||  }
|-
|   let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-
    handle.join().expect("Everything will be fine");
}
```

Another way is to get away with clones. But let's look at our regions closely:

```rust,noplayground
/---blog_url region
|   let blog_url = get_blog_url();
|
|/--posts_from_blog 'a region
||  let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
||
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
```

We don't really need the `blog_url` reference to be valid inside the for loop,
we only care about `post_urls` there. This `posts_from_blog 'a` region is
essentially a region for our `post_urls`, the `blog_url` region could be much
smaller, but the function signature asks the compiler to infer only a single
region, so the `blog_url` reference ends up coupled with the `post_urls`
references. What we actually want is to ask the compiler to infer 2 regions for
this function: the one for `post_urls` and the one for `blog_url`, so regions
in `main` would look like this

```rust,noplayground
/---blog_url region
|   let blog_url = get_blog_url();
|
|/--posts_from_blog 'post_urls region
||/-posts_from_blog 'blog_url region
||| let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();
||-
||  let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));
-|
 |  for url in post_urls {
 |      process_post(url);
 |  }
 -
```

This way `posts_from_blog 'blog_url` region is only 1-line long and borrowing
`blog_url` for this line is fine, when `posts_from_blog 'post_urls` region
holds only `post_urls` references and doesn't care about the `blog_url` region
at all. Let's try to split this `'a` region!

## Splitting `posts_from_blog 'a` region
To ask the compiler to infer 2 regions instead of 1 we just need to introduce a second lifetime parameter in the function signature:

```rust,noplayground
fn posts_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem],
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> {
    // ...
}
```

We're taking post urls from input items, so `items` clearly belong to
`post_urls` region. We use `blog_url` only for filtering, so it belongs to its
own `blog_url` region. Iterator returns post urls from the input `items`, so
`Item = &str` must belong to `post_urls` region. But what about an `Iterator`
itself? We're iterating `items`, so let's assign it to `post_urls` region.

```rust,noplayground
fn posts_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem],
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'post_urls {
    // ...
}
```

Now we will infer new regions for the updated signature, but in a bit different
context to emphasize that borrow checker analyzes each function completely
independetly:

```rust,noplayground
fn uaf(options: &CrawlerOptions) {
    let items = crawler::run(options);

    let blog_url = get_blog_url();
    let iterator = posts_from_blog(&items, &blog_url);
    drop(blog_url);

    for url in iterator {
        do_stuff(url);
    }
}
```

Let's infer regions in the `uaf` function:

```rust,noplayground
fn uaf(options: &CrawlerOptions) {
/---items region
|   let items = crawler::run(options);
|
|   let blog_url = get_blog_url();
|/--posts_from_blog 'post_urls region
||  let iterator = posts_from_blog(&items, &blog_url);
||  drop(blog_url);
||
||  for url in iterator {
||      do_stuff(url);
||  }
--
}
```

`posts_from_blog 'post_urls` holds an iterator, and a reference to the `items`
variable and it can dereference them at any line of this region because `items`
region outlives the `'post_urls` region.

```rust,noplayground
fn uaf(options: &CrawlerOptions) {
    let items = crawler::run(options);
/---blog_url region
|   let blog_url = get_blog_url();
|/--posts_from_blog 'blog_url region
||  let iterator = posts_from_blog(&items, &blog_url);
|-  drop(blog_url);
-
    for url in iterator {
        do_stuff(url);
    }

}
```

`posts_from_blog 'blog_url` region holds just a reference to the `blog_url`
variable. It's an input argument, therefore the reference should be valid only
for the time of the function call, so the region is 1-line long. `blog_url`
region clearly outlives this 1-line region, so it's safe to create a borrow
there. As the result the `uaf` function passes the borrow checking just
perfectly, but if we think about what's going on here we quickly realize
that the iterator holds a reference to `blog_url` internally to do the
comparisons, so in fact **we have a use after free memory bug here**.
`posts_from_blog` function signature doesn't tell anything about this
internal borrow, so the borrow checker can't spot any issues while
analyzing the `uaf` function. Luckily for us it can spot the issue during
the analysis of the `posts_from_blog` function body which is done only once
and completely independently from the `uaf`.

```rust
# struct DiscoveredItem {
#     blog_url: String,
#     post_url: String,
# }
fn posts_from_blog<'post_urls, 'blog_url>(
    items: &'post_urls [DiscoveredItem],
    blog_url: &'blog_url str,
) -> impl Iterator<Item = &'post_urls str> + 'post_urls {
    items.iter().filter_map(move |item| {
        if item.blog_url == blog_url {
            Some(item.post_url.as_str())
        } else {
            None
        }
    })
}
```

The borrow checker emits the following error for this implementation:

```
error[E0623]: lifetime mismatch
  --> src/main.rs:11:6
   |
10 |     blog_url: &'blog_url str,
   |               -------------- this parameter and the return type are declared with different lifetimes...
11 | ) -> impl Iterator<Item = &'post_urls str> + 'post_urls {
   |      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |      |
   |      ...but data from `blog_url` is returned here

For more information about this error, try `rustc --explain E0623`.
error: could not compile `playground` due to previous error
```

It was able to spot that the iterator borrows `blog_url` from the `'blog_url`
region, but the signature suggests that the iterator borrows only from the
`'post_urls` region, so the borrow checker threw a `lifetime mismatch` error.
In order to fix it we must reflect this `blog_url` borrow in our signature by
assigning the iterator to the `'blog_url` region.

```rust
# struct DiscoveredItem {
#     blog_url: String,
#     post_url: String,
# }
fn posts_from_blog<'post_urls, 'blog_url>(
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

But compilation fails with the same error. Hmm... It's time to resort to magic!


```rust
# struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
# }
fn posts_from_blog<'post_urls, 'blog_url>(
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

> [!WARNING]
> Since `Rust 2024` this resort to magic is not a correct approach anymore,
> with the new edition we can easily express precisely what we want:
> ```rust
> # struct DiscoveredItem {
> #     blog_url: String,
> #     post_url: String,
> # }
> fn posts_from_blog<'post_urls, 'blog_url>(
>     items: &'post_urls [DiscoveredItem],
>     blog_url: &'blog_url str,
> ) -> impl Iterator<Item = &'post_urls str> + use<'post_urls, 'blog_url>
> {
>     items.iter().filter_map(move |item| {
>         if item.blog_url == blog_url {
>             Some(item.post_url.as_str())
>         } else {
>             None
>         }
>     })
> }
> ```
>
> However, this served as a nice introduction point to the lifetime subtyping
> and the next chapter is built on further exploring this code snippet so it
> will stay here until the next chapter is rewritten with better examples.

And now everything compiles, including the example from the previous chapter.
Check it out:

```rust
# struct DiscoveredItem {
#     blog_url: String,
#     post_url: String,
# }
# fn posts_from_blog<'post_urls, 'blog_url>(
#     items: &'post_urls [DiscoveredItem],
#     blog_url: &'blog_url str,
# ) -> impl Iterator<Item = &'post_urls str> + 'blog_url
# where
#     'post_urls: 'blog_url
# {
#     items.iter().filter_map(move |item| {
#         if item.blog_url == blog_url {
#             Some(item.post_url.as_str())
#         } else {
#             None
#         }
#     })
# }
fn main() {
    // Assume the crawler returned the following results
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned().to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_do_this_one_thing_every_day".to_owned(),
        },
    ];

    // Reading the blog URL we're interested in from somewhere
    let blog_url = get_blog_url();

    // Collecting post URLs from this blog using our function
    let post_urls: Vec<_> = posts_from_blog(crawler_results, &blog_url).collect();

    // Spawning a thread to do some further blog processing
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    // Processing posts in parallel
    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}

// Returns a predefined value
fn get_blog_url() -> String {
    "https://blogs.com/".to_owned()
}

// Just prints URL out
fn process_post(url: &str) {
    println!("{}", url);
}

// Actually does nothing
fn calculate_blog_stats(_blog_url: String) {}
```

We will demistify the added `where` clause and will understand the last
compilation error in the next chapter, but before going further make sure you
understood the material from this chapter.

## Chapter exercises

> #### Exercise 1
> The chapter says when we encounter a function call we need to infer minimal regions for it at the invocation point.
> Why do we want these regions to be minimal?


> #### Exercise 2
> Assume this signature compiles:
> ```rust,noplayground
> fn posts_from_blog<'post_urls, 'blog_url>(
>     items: &'post_urls [DiscoveredItem],
>     blog_url: &'blog_url str,
> ) -> impl Iterator<Item = &'post_urls str> + 'blog_url {
>     // ...
> }
> ```
>
> Go back to the `uaf` example. Infer and validate regions for the `uaf` using
> this `posts_from_blog` signature. Does `uaf` compile? Try to come up with an
> example that causes use after free again.

> #### Exercise 3
> - Manually infer regions for the following function and explain why this code doesn't compile
> ```rust
>
> struct RefMutPair<'a, 'b> {
>     a: &'a mut String,
>     b: &'b mut String,
> }
>
> impl<'a, 'b> RefMutPair<'a, 'b> {
>     fn concat(&self) -> String {
>         format!("{}{}", self.a, self.b)
>     }
> }
>
> fn main() {
>     let mut x = String::from("X");
>     let mut y = String::from("Y");
>
>     let pair = RefMutPair {
>         a: &mut x,
>         b: &mut y,
>     };
>
>     println!("{}", pair.concat());
>
>     println!("{}", x);
>     pair.b.push('c');
>     println!("{}", y);
> }
> ```
>
> - The same results can be achieved with the simplified struct definition, can
> you write it down?
>
> - Now replace the struct above with the following:
>
> ```rust
> struct RefMutPair<'a, 'b> {
>    a: &'a String,
>    b: &'b mut String,
> }
> ```
>
> Try to explain why the example suddenly compiles.

> #### Exercise 4
>
> The new `+ use<_generics_>` syntax effectively allows to define generics for
> the existential types so the correct modern solution to this chapter's problem
>
> ```rust,noplayground
>
> fn posts_from_blog<'post_urls, 'blog_url>(
>     items: &'post_urls [DiscoveredItem],
>     blog_url: &'blog_url str,
> ) -> impl Iterator<Item = &'post_urls str> + use<'post_urls, 'blog_url> {
>     items.iter().filter_map(move |item| {
>         if item.blog_url == blog_url {
>             Some(item.post_url.as_str())
>         } else {
>             None
>         }
>     })
> }
> ```
>
> turns out to be quite similar to the `RefPair` example:
>
> ```rust,noplayground
> struct InferredIteratorType<'post_urls, 'blog_url> {
>     post_urls: &'post_urls [DiscoveredItem],
>     blog_url: &'blog_url str,
> }
>
> impl<'post_urls, 'blog_url> Iterator for InferredIteratorType<'post_urls, 'blog_url> {
>     type Item = &'a post_urls str;
>     //...
> }
>
> fn posts_from_blog<'post_urls, 'blog_url>(
>     items: &'post_urls [DiscoveredItem],
>     blog_url: &'blog_url str,
> ) -> InferredIteratorType<'post_urls, 'blog_url>
>     items.iter().filter_map(move |item| {
>         if item.blog_url == blog_url {
>             Some(item.post_url.as_str())
>         } else {
>             None
>         }
>     })
> }
> ```
>
> Can you explain why unlike in `RefPair` example it is absolutely required to
> pass 2 distinct lifetimes(infer 2 regions) for the main example to work?

[^continuous]: The book was written in pre-Polonius era.

[^iterator]: Since edition 2024 the correct `Iterator` signature is `impl Iterator<Item = &'a str>> + use<'a>`.
