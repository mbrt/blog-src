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
filesystem, and pushes them into a [deque](https://crates.io/crates/deque).
This is a Single-producer / Multiple-consumers queue, from which multiple worker
threads can pull at the same time, and perform the search operations. Here is
the workers initialization in the `run` function:

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

As you can see, the `deque::new()` returns two objects. The queus is indeed
composed by two ends. One is the `workq` from which the main thread can push,
and the other end is the `stealer`, from which all the workers can pull. In
every iteration of the loop a new worker is created and moved to a new thread,
along with a `stealer`. Note that the `stealer` is
[cloneable](https://doc.rust-lang.org/std/clone/trait.Clone.html),
but this doesn't mean that the queue itself is cloned. Internally indeed the
`stealer` contains an
[Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) to the queue:

```rust
pub struct Stealer<T: Send> {
    deque: Arc<Deque<T>>,
}
```

To note here is the beauty of the `deque` interface. To express the fact that
the producer is only one, but the consumers can be multiple, the type is split
in two: the producer is then
[Send](https://doc.rust-lang.org/std/marker/trait.Send.html) but not
[Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html), nor
[Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html). There is no way
to use it from multiple threads, since you can yeld the instance to another
thread, but you can't keep another reference to it. The `Stealer`, which
is the other end, is instead both `Send` and `Clone`. You can then pass them
around by cloning and sending them off to other threads. They will all refer to
the same queue. There is no way to use this interface incorrectly.

Another thing to note here is that the result of the block is just the producer
part of the `deque`. The worker threads are pushed into a vector. The workers
along with their stealers are moved into the threads. The use of a block that
just returns what is needed for the rest of the function is a good practice. In
this way the `run` function is not polluted with variables that are not usable
anymore because their values have been moved.

This is the `MultiWorker` struct, that runs in a separate thread:

```rust
struct MultiWorker {
    chan_work: Stealer<Work>,
    quiet_matched: QuietMatched,
    out: Arc<Mutex<Out>>,
    #[cfg(not(windows))]
    outbuf: Option<ColoredTerminal<term::TerminfoTerminal<Vec<u8>>>>,
    #[cfg(windows)]
    outbuf: Option<ColoredTerminal<WindowsBuffer>>,
    worker: Worker,
}
```

the first field is the stealer. As you can see from its type, the items it
contains are `Work` structs:

```rust
enum Work {
    Stdin,
    File(DirEntry),
    Quit,
}
```

The main thread will push them from its `workq` variable:

```rust
for dent in args.walker() {
    if quiet_matched.has_match() {
        break;
    }
    paths_searched += 1;
    if dent.is_stdin() {
        workq.push(Work::Stdin);
    } else {
        workq.push(Work::File(dent));
    }
}
```

The `args.walker()` is an iterator over the files to search, or standard input
if the `-` argument is passed. In the former case it pushes a `Work::File` entry
with the path, in the latter it pushes a `Work::Stdin` entry.

The `MultiWorker::run` function is a loop that pops from the `deque` entries to
process one by one:

```rust
loop {
    if self.quiet_matched.has_match() {
        break;
    }
    let work = match self.chan_work.steal() {
        Stolen::Empty | Stolen::Abort => continue,
        Stolen::Data(Work::Quit) => break,
        Stolen::Data(Work::Stdin) => WorkReady::Stdin,
        Stolen::Data(Work::File(ent)) => {
            match File::open(ent.path()) {
                Ok(file) => WorkReady::DirFile(ent, file),
                Err(err) => {
                    eprintln!("{}: {}", ent.path().display(), err);
                    continue;
                }
            }
        }
    };
    // ...
}
```

The `steal()` method tries to pop from the `deque` and returns a `Stolen`
instance:

```rust
pub enum Stolen<T> {
    /// The deque was empty at the time of stealing
    Empty,
    /// The stealer lost the race for stealing data, and a retry may return more
    /// data.
    Abort,
    /// The stealer has successfully stolen some data.
    Data(T),
}
```

The outcome is matched against the different possibilities, and only
`Stolen::Data` contains a `Work` entry. Both `Stdin` and `File` are then
translated into a `WorkReady` instance. In the second case the file is then
opened with an `std::fs::File`. The `work` variable is later consumed by a
`Worker` instance:

```rust
self.worker.do_work(&mut printer, work);
```

We'll get back to that afterwords, but let's first backtrack to the
`MultiWorker::run` loop. The `Work::Quit` case breaks it, so the thread
terminates:

```rust
let work = match self.chan_work.steal() {
    // ...
    Stolen::Data(Work::Quit) => break,
    // ...
```

This value is pushed by the main thread when all the files have been examinated:

```rust
for _ in 0..workers.len() {
    workq.push(Work::Quit);
}
let mut match_count = 0;
for worker in workers {
    match_count += worker.join().unwrap();
}
```

The threads are all guaranteed to terminate, because the number of pushed `Quit`
messages is the same as the number of workers. A worker can only consume one of
them and then quit. This implies that, since no messages can be lost, all the
workers will get the message at some point and then terminate. All the workers
threads are then joined, waiting for completion.

This is a good multi-threading pattern to follow:

* a `deque` in between a producer (that doesn't have to do intensive jobs) and a
  bunch of consumers (that do the heavy lifting) in separate threads;
* the `deque` carries an enumeration of the things to do, and one of them is the
  `Quit` action;
* the producer will eventually push a bunch of `Quit` messages to terminate the
  worker threads.

In case you just have one type of job, it makes perfect sense to use an
`Option<Stuff>` as work item. The workers will terminate in case `None` is
passed. The `Option` can be used also in the `ripgrep` case, replacing the
`Quit` message but I'm not sure the code would be more readable.

This was the multi-threading operational mode. `ripgrep` can however operate in
a single thread, in case there is only one file to search or only one core to
use, or the user says so. The `run` function checks that:

```rust
let threads = cmp::max(1, args.threads() - 1);
let isone =
    paths.len() == 1 && (paths[0] == Path::new("-") || paths[0].is_file());
// ...
if threads == 1 || isone {
    return run_one_thread(args.clone());
}
```

and calls the `run_one_thread` function for the single-threaded case (I have
removed some uninteresting details):

```rust
fn run_one_thread(args: Arc<Args>) -> Result<u64> {
    let mut worker = Worker {
        args: args.clone(),
        inpbuf: args.input_buffer(),
        grep: args.grep(),
        match_count: 0,
    };
    // ...
    for dent in args.walker() {
        // ...
        if dent.is_stdin() {
            worker.do_work(&mut printer, WorkReady::Stdin);
        } else {
            let file = match File::open(dent.path()) {
                Ok(file) => file,
                Err(err) => {
                    eprintln!("{}: {}", dent.path().display(), err);
                    continue;
                }
            };
            worker.do_work(&mut printer, WorkReady::DirFile(dent, file));
        }
    }
    // ...
}
```

As you can see, the function uses a single `Worker`. If you remember, this struct
is also used by `MultiWorker`. The files to search are iterated by
`args.walker()` as before, and each entry is passed to the `worker`, as before.
The use of `Worker` across these two cases allows code reuse at a great extent.

The file listing
----------------

We are now going to look over the file listing functional block.

The default operation mode of `ripgrep` is to search recursively for non-binary,
non-ignored files starting from the current directory (or from the given
paths). To enumerate the files and feed the search engine, `ripgrep` uses the
`ignore` crate.

But let's start from the beginning. The `walker` function provided by `Args`
returns a `Walk` struct:

```rust
pub fn walker(&self) -> Walk;
```

`Walk` is just a simple wrapper around the `ignore::Walk` struct. A value of
this struct can be created by using the `new` method:

```rust
pub fn new<P: AsRef<Path>>(path: P) -> Walk;
```

or with a `WalkBuilder`, that implements the
[builder pattern](https://doc.rust-lang.org/book/method-syntax.html#builder-pattern).
This allows to customize the behavior without annoying the user to provide a lot
of parameters to the constructor:

```rust
let w = WalkBuilder::new(path).ignore(true).max_depth(Some(5)).build();
```

In this example we have specified only two parameters; all the others will be
defaulted.

The implementation of the type is not very interesting from our point of view.
It is basically an `Iterator` that walks through the filesystem by using the
`walkdir` crate, but ignores the files and directories listed in `.gitignore`
and `.ignore` files possibly present, with the help of the `Ignore` type.

An interesting bit is instead the `Error` type:

```rust
#[derive(Debug)]
pub enum Error {
    Partial(Vec<Error>),
    WithLineNumber { line: u64, err: Box<Error> },
    WithPath { path: PathBuf, err: Box<Error> },
    Io(io::Error),
    Glob(String),
    UnrecognizedFileType(String),
    InvalidDefinition,
}
```

This error type has an interesting recursive definition. The `Partial` case of
the enumeration contains a vector of `Error`s, for example. `WithLineNumber`
adds line information to an `Error`. In this case `Box<Error>` since a recursive
type cannot embed itself, otherwise it would be impossible to compute the size
of the type.

Then the [error::Error](https://doc.rust-lang.org/std/error/trait.Error.html),
[fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) and
[`From<io::Error>`](https://doc.rust-lang.org/std/convert/trait.From.html)
traits are implemented, to make it a proper error type, and to easily convert an
`io::Error` into it. Here the boilerplate necessary to crank up the error type
are handcrafted. Another possibility would have been to use the
[quick-error](https://github.com/tailhook/quick-error) macro, which reduces the
burden to implement error types to a minimum. A good reference on the error
handling topic is present in
[the Rust book](https://doc.rust-lang.org/stable/book/error-handling.html).

### Ignore patterns

Ignore patterns are handled within the `ignore` crate by the `Ignore` struct.
This type connects directory traversal with ignore semantics. In practice it
builds a tree-like structure that mimics the directories structure, in which
leaves are new ignore contexts. The implementation is quite complicated.
