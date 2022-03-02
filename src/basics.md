# Basics

_It's recommended to follow the example by retyping it in your favorite editor_

Imagine we're writing a web crawler that crawls posts from different blogs.
The crawler's output looks like this:

```rust
struct DiscoveredItem {
    blog_url: String,
    post_url: String,
}
```

Now we want to write a function filtering the web crawler's results to iterate
posts from some specific blog.


```rust
struct DiscoveredItem {
    blog_url: String,
    post_url: String,
}


fn posts_from_blog(items: &[DiscoveredItem], blog_url: &str) -> impl Iterator<Item = &str> {
    // Creating an iterator from the &[DiscoveredItem] slice
    items.iter().filter_map(move |item| {
        // Filtering items by blog_url and returning a post_url
        (item.blog_url == blog_url).then_some(item.post_url.as_str())
    })
}
```

Our function doesn't compile, the compiler complains it needs some lifetime
annotations.

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:6:86
  |
6 | fn posts_from_blog(items: &[DiscoveredItem], blog_url: &str) -> impl Iterator<Item = &str> {
  |                           -----------------            ----                          ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `items`
 or `blog_url`
help: consider introducing a named lifetime parameter
  |
6 | fn posts_from_blog<'a>(items: &'a [DiscoveredItem], blog_url: &'a str) -> impl Iterator<Item = &'a str> {
  |                   ++++         ++                              ++                               ++

For more information about this error, try `rustc --explain E0106`.
```

If we read the error carefully we can even see the suggestion on how to fix our
signature.

Let's apply it!

```rust
struct DiscoveredItem {
    blog_url: String,
    post_url: String,
}

fn posts_from_blog<'a>(items: &'a [DiscoveredItem], blog_url: &'a str) -> impl Iterator<Item = &'a str> {
    // Creating an iterator from the &[DiscoveredItem] slice
    items.iter().filter_map(move |item| {
        // Filtering items by blog_url and returning a post_url
        (item.blog_url == blog_url).then_some(item.post_url.as_str())
    })
}
```

Cool, we satisifed the borrow checker and now everything compiles and therefore
works... right? Well, not quite. The compiler actually tricked us. It provided
a suggestion which is semantically incorrect and which made our function
overly- restrictive. This means the borrow checker will strike back soon and
will make some developer's day insuffarable. Let's step into this developer's
shoes.

## Borrow checker strikes back

Now we're trying to use the function in some non-trivial context:

```rust
#struct DiscoveredItem {
#    blog_url: String,
#    post_url: String,
#}
#
#fn posts_from_blog<'a>(items: &'a [DiscoveredItem], blog_url: &'a str) -> impl Iterator<Item = &'a str> {
#    // Creating an iterator from the &[DiscoveredItem] slice
#    items.iter().filter_map(move |item| {
#        // Filtering items by blog_url and returning a post_url
#        (item.blog_url == blog_url).then_some(item.post_url.as_str())
#    })
#}
fn main() {
    // Assume the crawler returned the following results
    let crawler_results = &[
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/cooking/fried_eggs".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://blogs.com/".to_owned(),
            post_url: "https://blogs.com/travelling/death_mountain".to_owned(),
        },
        DiscoveredItem {
            blog_url: "https://successfulsam.xyz/".to_owned(),
            post_url: "https://successfulsam.xyz/keys_to_success/Just_use_AI_every_day".to_owned(),
        },
    ];

    // Reading the blog URL we're interested in from somewhere
    let blog_url = get_blog_url();

    // Collecting post URLs from this blog using our function
    let post_urls: Vec<&str> = posts_from_blog(crawler_results, &blog_url).collect();

    // Spawning a thread to do some further blog processing
    let handle = std::thread::spawn(move || calculate_blog_stats(blog_url));

    // Processing posts in parallel
    for url in post_urls {
        process_post(url);
    }

    handle.join().expect("Everything will be fine");
}

fn get_blog_url() -> String {
    "https://blogs.com/".to_owned()
}

fn process_post(url: &str) {
    println!("{}", url);
}

// Assume some requests being made to the blog_url to evaluate stats and save them to DB
fn calculate_blog_stats(_blog_url: String) {}
```

This code doesn't compile and the compiler error is very confusing:

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

How `blog_url` can stay borrwed in the loop when we `collect` the iterator
immediately and `post_urls: Vec<&str>` is a collection of references coming
directly from `crawler_results` which are unrelated to `get_blog_url()` call?
When you see such confusing errors it's often a sign that the problem is not in
your code but in one of the function signatures that your code uses. But how to
identify and fix malformed function signatures and what the compiler error is
actually communicating to us? To answer these questions, we need to understand
the way the borrow checker works.
