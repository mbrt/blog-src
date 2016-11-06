Ripgrep code review
===================

TOC
---

1. Introduction
2. Big picture
3. Main
4. The file listing
    1. Ignore patterns
5. The search process
6. Output handling


Introduction
------------

I've been playing around with [Rust](https://www.rust-lang.org) for a year and a
half, and as many other people already said, the best of it is the very
welcoming and helpful community. There are a lot of online resources that help
you to get started: the [Rust book](https://doc.rust-lang.org/book/), the
[Rustonomicon](https://doc.rust-lang.org/nomicon/) and many blog posts and stack
overflow questions. After I learned the basics I felt a bit lost though, since
not many advanced topics are available. As a C++ developer I miss indeed books
like [Effective C++](http://www.aristeia.com/books.html) from Scott Meyers, and
people like [Herb Sutter](https://herbsutter.com/). They provide a lot of
insights besides the basics. They teach you how to get the best from the
language, how to use it properly, and how to structure your code to be more
clear and effective.

How do you learn those things then? You look at how the best developers work.
I think code reviews are incredibly useful; you can see how other people reason
about problems you also struggled with, and how they have solved them. This post
is an attempt to do so, by looking at the
[ripgrep](https://github.com/BurntSushi/ripgrep) crate by Andrew Gallant, which
is a great example of good Rust, in my opinion.

I'm not going to explain everything about the crate, since there is already a
very good [blog post](http://blog.burntsushi.net/ripgrep/) by Andrew himself,
explaining how the application works from a functional perspective, compared to
GNU grep, the Silver Searcher (`ag`), and other similar tools. Our perspective
is instead more implementation driven. We are going to walk through the crate
architecture. As a side note, this post is not intended for complete beginners
since I'm going to take for granted the basics. If you need a refresher you can
take a look at the resources I mentioned before.

We are going to look at this specific version of the crate:

```
$ git describe
0.2.5-4-gf728708
```

which is the last one at the time of writing. By the time you are reading, the
code might have evolved, so if you want to follow the code live, it's better to
checkout this specific version:

```
$ git clone https://github.com/BurntSushi/ripgrep.git
$ cd ripgrep
$ git checkout f728708
```

So, without further ado, let's get started.

The big picture
---------------

Ripgrep is a command line tool for searching file contents using regular
expressions, similarly to GNU grep. The tool is split across four crates: the
main one, `ignore`, `grep` and `globset`.

![Crates](https://cdn.rawgit.com/mbrt/blog/master/001-ripgrep-code-review/crates.svg)

The `grep` crate provides line-by-line regex searching from a buffer and it is
used only by the main crate. The `globset` crate uses regex to perform
[glob matching](https://en.wikipedia.org/wiki/Glob_(programming)) over paths. It
is used by the main and the `ignore` crates. The `ignore` crate implements
directory walking, ignore and include patterns. It uses the `glob` crate for
that. Finally, the main crate, which glues everything together, implementing
also command line argument parsing, output handling and multi-threading.

One clear advantage of splitting an application in multiple crates is that it
forces you to keep your code scoped. It's easy to create a mess of dependencies
among the components if everything is in the same crate. If you instead take a
part of your application and try to give it a meaning by itself, you end up with
a more generic, usable and clearer interface. Embrace the
[Single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle)
and let it be your guide.

Main
----

Everything starts from the `ripgrep` main function:

```rust
fn main() {
    match Args::parse().and_then(run) {
        Ok(count) if count == 0 => process::exit(1),
        Ok(_) => process::exit(0),
        Err(err) => {
            eprintln!("{}", err);
            process::exit(1);
        }
    }
}
```

It is very concise: it parses the command line arguments and then passes them to
the `run` function. In between them there is the
[`Result::and_then`](https://doc.rust-lang.org/std/result/enum.Result.html#method.and_then)
combinator, so the `match` statement gets to the `Ok` branch only if both
operations did not fail, otherwise it selects the `Err` branch, handling errors
for both the first and the second operation. Then the main exit code depends on
whether there have been matches or not.

```rust
fn run(args: Args) -> Result<u64>;
```

The file listing
----------------

The default operation mode of `ripgrep` is to search recursively for non-binary,
non-ignored files, starting from the current directory. To feed the search
engines for those files, it uses the `ignore` crate.
