# Introduction

Lifetimes are one of Rust's most distinctive feature. They are what makes the
language so valuable in the systems programming domain, and it's essential to
master them to use Rust effectively. Unfortunately, they're not very well
explained topic: to have a really solid understanding of how lifetimes work you
have to learn about them bit by bit from scattered blogs, `rustlang` GitHub
issues, compiler source-code, and Zulip discussions &ndash; so this book is my
attempt to cover everything in a single place.

We will start from the [Basics](basics.md) chapter where we will explore how lifetime
annotations affect the borrow checking, after finishing this chapter you should
be able to annotate your code with a confidence and an understanding of what
you're doing. Then we will touch various topics that are nice to know about in
certain situations and which are not as important as [Basics](basics.md) so
you can always revisit them when you need to.


### The book was never finished

Originally, I planned to cover a broad list of lifetime-related topics but
due to some unfortunate circumstances this project had to be freezed. If you
find the information here valuable you can help resurrecting the book with your
[contributions](https://github.com/a1akris/lifetimes-book)

- Fix typos, improve wording, suggest improvements to existing chapters
- Contribute more examples and exercises
    - **right now the [lifetime subtyping](lifetime_subtyping.md) chapter
      desperately needs a good intuitive example justifying the use of
      outlives relationship in function signatures**
- Contribute misc chapters about:
    - Reborrowing and stacked borrows
    - Lifetime ellision rules, different rules for functions and closures,
      tricks that help to fix incorrectly ellided lifetimes
    - Late and early bounds and the use of `for<'a>`
    - When the use of lifetimes in trait definitions is justified
    - Polonius(intuitive model) and the future of borrowchk
    - Lifetimes in GATs
    - Binding lifetimes in unsafe code
    - _...etc..._
