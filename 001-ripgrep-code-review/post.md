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

As you can see, the `deque::new()` returns two objects. The queue is indeed
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
to use it from multiple threads, since you can yield the instance to another
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

This value is pushed by the main thread when all the files have been examined:

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
This allows to customize the behavior without annoying the user, forcing them
to provide a lot of parameters to the constructor:

```rust
let w = WalkBuilder::new(path).ignore(true).max_depth(Some(5)).build();
```

In this example we have specified only two parameters; all the others will be
defaulted.

The implementation of the type is not very interesting from our point of view.
It is basically an `Iterator` that walks through the filesystem by using the
`walkdir` crate, but ignores the files and directories listed in `.gitignore`
and `.ignore` files possibly present, with the help of the `Ignore` type. We
will look at that type a bit later. Let's look at the `Error` type first:

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
adds line information to an `Error`. In this case `Box<Error>`, since a
recursive type cannot embed itself, otherwise it would be impossible to compute
the size of the type.

Then the [error::Error](https://doc.rust-lang.org/std/error/trait.Error.html),
[fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) and
[`From<io::Error>`](https://doc.rust-lang.org/std/convert/trait.From.html)
traits are implemented, to make it a proper error type and to easily construct
it out an `io::Error`. Here the boilerplate necessary to crank up the error type
are handcrafted. Another possibility could have been to use the
[quick-error](https://github.com/tailhook/quick-error) macro, which reduces the
burden to implement error types to a minimum. A good reference on the error
handling topic is present in
[the Rust book](https://doc.rust-lang.org/stable/book/error-handling.html).

### Ignore patterns

Ignore patterns are handled within the `ignore` crate by the `Ignore` struct.
This type connects directory traversal with ignore semantics. In practice it
builds a tree-like structure that mimics the directories structure, in which
leaves are new ignore contexts. The implementation is quite complicated, but
let's have a brief look at it:

```rust
#[derive(Clone, Debug)]
pub struct Ignore(Arc<IgnoreInner>);

#[derive(Clone, Debug)]
struct IgnoreInner {
    compiled: Arc<RwLock<HashMap<OsString, Ignore>>>,
    dir: PathBuf,
    overrides: Arc<Override>,
    types: Arc<Types>,
    parent: Option<Ignore>,
    is_absolute_parent: bool,
    absolute_base: Option<Arc<PathBuf>>,
    explicit_ignores: Arc<Vec<Gitignore>>,
    ignore_matcher: Gitignore,
    git_global_matcher: Arc<Gitignore>,
    git_ignore_matcher: Gitignore,
    git_exclude_matcher: Gitignore,
    has_git: bool,
    opts: IgnoreOptions,
}
```

I have taken out the comments to make it short, but you can find them in
`ignore/src/dir.rs`. The `Ignore` struct is a wrapper around an atomic reference
counter to the actual data (i.e. `IgnoreInner`). A first interesting field
inside that struct is `parent`, that is an `Option<Ignore>`, so it points to a
parent if it is present. This is how the tree structure is implemented. Since
the `Arc` can be shared, multiple `Ignore` can share the same parent. But that
is not all; they can be also cached. The `compiled` field has a mouthful type:

```rust
Arc<RwLock<HashMap<OsString, Ignore>>>
```

but this is the cache of `Ignore` instances that is shared among all of them.
Let's try to break it down:

* the `HashMap` maps paths to `Ignore` instances (as expected);
* the `RwLock` allows the map to be shared and modified across different
  threads, without causing data races;
* and finally the `Arc` allow the cache to be owned safely by different threads.

every time a new `Ignore` instance has to be built and added to a tree, the
implementation first looks in the cache, trying to reuse the existing instances.
The tree is built dynamically, while crawling the directories, looking for the
specific ignore files (e.g. `.gitignore`, `.ignore`, `.rgignore`) and support
custom ignores.

Another interesting bit is the `add_parents` signature for `Ignore`:

```rust
pub fn add_parents<P: AsRef<Path>>(&self, path: P) -> (Ignore, Option<Error>);
```

instead of returning a `Result<Ignore, Error>` it uses a pair, returning always
a result and optionally an error. In this way partial failures are allowed. If
you remember, the error can also be an array of errors, so the function can
collect them all while working and do it's best, but then it can also return a
(maybe partial) result. I found this approach very interesting.

The search process
------------------

In this section we will look at how the regex search inside a file is
implemented. This process involves some modules in `ripgrep` and also the `grep`
crate.

Everything starts from `Worker::do_work` in `main.rs`. Based on the type of the
file passed in, `search` or `search_mmap` are in turn called. The first function
is used to read the input one chunk at a time and then search, while the second
is used to search into a memory mapped input. In this case there is no need to
read the file into a buffer, because it is already available in memory, or more
precisely, the kernel will take care of this illusion.

The `search` function just creates a new `Searcher` and calls `run` on it.

```rust
impl<'a, R: io::Read, W: Terminal + Send> Searcher<'a, R, W> {
    pub fn run(mut self) -> Result<u64, Error>;
}
```

The first interesting thing to note here is that the `run` function actually
consumes `self`, so you can't actually run it twice with the same instance. Why
is that? Let's have a look at the `new` method, that creates this struct:

```rust
impl<'a, R: io::Read, W: Terminal + Send> Searcher<'a, R, W> {
    pub fn new(inp: &'a mut InputBuffer,
               printer: &'a mut Printer<W>,
               grep: &'a Grep,
               path: &'a Path,
               haystack: R) -> Searcher<'a, R, W>;
}
```

It takes a bunch of arguments and stores them into a new `Searcher` instance.
All the arguments to `Searcher` are passed as reference, except `haystack` which
is the `Read` stream representing the file. This means that when this struct
will be destroyed, the file will be gone. Whenever you complete the search for a
file, you don't have to do it again, indeed. You can enforce this usage by
consuming the input file in the `run` function, or take its ownership in the
constructor and force the `run` function to consume `self`.

Since we cannot run the search twice using the same `Searcher` instance, why
don't we just use a function then? The approach used here has several advantages:

1. you get the behavior that the search cannot be run twice with the same file
   (nothing that a free function could not do);
2. you can split the function among different private functions, without passing
   around all the arguments; they will all take `self` by reference (maybe also
   `&mut self`) and just refer to the member variables.

So, instead of:

```rust
fn helper1(&self,
           inp: &mut InputBuffer,
           printer: &mut Printer<W>,
           grep: &Grep,
           path: &Path,
           haystack: &mut R)
{
    // do something with path, grep, etc
}
```

we have:

```rust
fn helper1(&mut self) {
    // do something with self.path, self.grep, etc
}
```

The end result is much nicer.
