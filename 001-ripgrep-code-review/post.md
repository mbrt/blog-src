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
not many advanced resources are available. As a C++ developer I miss indeed
books like [Effective C++](http://www.aristeia.com/books.html) from Scott
Meyers, and people like [Herb Sutter](https://herbsutter.com/) that provide a
lot of insights besides the basics. They teach you how to get the best from the
language, how to use it properly, and how to structure your code to be more
clear and effective. Those resources are not completely absent in the Rust
community, but neither common.

How do you learn those things then? You look at how the best developers work.
I think code reviews are incredibly useful; you can see how other people reason
about problems you also struggled with, and how they have solved them. This post
attempts to target a more navigated audience, by looking at the
[ripgrep](https://github.com/BurntSushi/ripgrep) crate by Andrew Gallant, which
is a great example of good Rust, in my opinion.

I'm not going to explain everything about the crate, since there is already a
very good [blog post](http://blog.burntsushi.net/ripgrep/) by Andrew himself,
explaining how the application works from a functional perspective. We are going
instead to walk through the crate architecture. I'm going to take for granted
some of the basics, so if you need a refresher you can take a look at the
resources I mentioned before.

We are going to look at this specific version of the crate:

```
$ git describe
0.2.5-4-gf728708
```

which is the last one at the time of writing. By the time you are reading the
code might have evolved, so if you want to follow the code live, it's better to
checkout this specific version:

```
$ git clone https://github.com/BurntSushi/ripgrep.git
$ cd ripgrep
$ git checkout f728708
```

and without further ado, let's get started.

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
fn run(args: Args) -> Result<u64> {
    // ...
}
```

The `run` function at first decides if it's worth to spawn threads or not, and
if so, this is the way it setups the things:

![Main](https://rawgit.com/mbrt/blog/master/001-ripgrep-code-review/main.svg)

The main thread, controlled by the `run` function digs files from the
filesystem, and pushes them into a [`deque`](https://crates.io/crates/deque).
This is a Single-producer / Multiple-consumers queue, from which multiple worker
threads can pull at the same time:

```rust
let workq = {
    let (workq, stealer) = deque::new();
    for _ in 0..threads {
        let worker = MultiWorker {
            chan_work: stealer.clone(),
            // initialize other fields...
        };
        workers.push(thread::spawn(move || worker.run()));
    }
    workq
};
```

As you can see, the `deque` is composed by two objects, one end is the `workq`
from which the main thread can push, and the other end is the `stealer`, from
which all the workers can pull. In every iteration of the loop a new worker is
created and moved to a new thread, along with a `stealer`. Note that the
`stealer` is [cloneable](https://doc.rust-lang.org/std/clone/trait.Clone.html),
but this doesn't mean that the queue itself is cloned. Internally indeed the
`stealer` contains an
[`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html) to the queue:

```rust
pub struct Stealer<T: Send> {
    deque: Arc<Deque<T>>,
}
```

To note here is the beauty of the `deque` interface. To express the fact that
the producer is only one, but the consumers can be multiple, the type is split
in two: the producer is then
[`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html) but not
[`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html), nor
[`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html). There is no way
to use it from multiple threads, since you can yeld the instance to another
thread, but you can't keep another reference to it. The `Stealer`, which
is the other end, is instead both `Send` and `Clone`. You can then pass them
around by cloning and sending them off to other threads. They will all refer to
the same queue. There is no way to use this interface incorrectly.

Another thing to note is that the block returns only the producer end of the
`deque`, and the workers are pushed into a vector. A good suggestion here: try
to keep the variables the most scoped as possible. The `run` function is not
polluted with a lot of variables used only in this place. They are inside this
block that returns only the minimum needed by the rest of the function.

The file listing
----------------

The default operation mode of `ripgrep` is to search recursively for non-binary,
non-ignored files, starting from the current directory. To feed the search
engines for those files, it uses the `ignore` crate.
