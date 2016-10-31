Ripgrep code review
===================

Reviewed version: 0.2.5-4-gf728708

Big picture
-----------

* The search process is split in two implementations, in the `search_buffer` and
  `search_stream` modules.
* Command line parsing and options are handled in `args.rs`.
* Output is handled by the `term` crate, `printer.rs`, `out.rs`, `terminal`,
  `terminal_win` and `atty.rs`.
* Directory walking, ignore and include patterns are implemented in the `ignore`
  crate, with the help of the `walkdir` crate.

main.rs
-------

Duties:

* args parsing and actual searching delegated to mods;
* multi-threading handling;
* matches together args and workers;

What's interesting in there?

Multi-threaded or not:

* Depending on the given options and on the inputs, the `run` function decides
  whether it's better to run the search procedure by using multiple threads or
  not.

* In the single threaded case, the `Worker` struct is used:

      ```rust
      struct Worker {
          args: Arc<Args>,
          inpbuf: InputBuffer,
          grep: Grep,
          match_count: u64,
      }
      ```

The work stealing mechanism:

* The main thread starts the worker threads, walks over the directories and
  feeds the queue with files to search;

* A worker picks up a job from the queue and starts working on it;

* A job is described by this enum:

      ```rust
      enum Work {
          Stdin,
          File(DirEntry),
          Quit,
      }
      ```

    So, it could be either:

      * standard input;
      * a file path;
      * indication to terminate.

* A worker is described by this struct:

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

    It forwards the jobs to a `Worker`. So, the actual work is performed by the
    single-threaded version.

* At the end of the work, to stop the workers, the main thread just enqueues a
  bunch of "quit" messages (precisely equal to the number of threads); you can
  be sure they will all quit because when one thread picks a quit message it
  immediately terminate, so the rest of them will eventually pick the remaining
  quit messages.

How does the output is managed?

* There's a printer that refers to a terminal;
* This is passed to the worker when the actual job has to be performed (the
  `do_work` function);
* The worker forwards the printer to the actual searcher.

How the files are managed:

* The files are opened by the main thread directly, and passed to the workers;
* The worker decides whether to use a memory map or not and forwards the search
  process to an actual `Searcher`.

Suggestions:

* Remove `Quit` from the enum, and use instead `Option<Work>`; it feels much
  more ideomatic.

* In `run` refactor out the last block of code into `run_multiple_threads`, so
  it's more consistent with the previous blocks.

* Instead of cluttering the `MultiWorker` class with platform specific members,
  they could have used a wrapper module exporting the correct type, or provide
  a good `type` directive:

      ```rust
      #[cfg(not(windows))]
      type Terminal = Option<ColoredTerminal<term::TerminfoTerminal<Vec<u8>>>>;
      #[cfg(windows)]
      type Terminal = Option<ColoredTerminal<WindowsBuffer>>;
      ```

    and then use it across the module right away.

args.rs
-------

Duties:

* parse command line arguments;
* convert the options into something closer to the rest of the crate (no dumb
  options struct).

What's interesting in there?

* The `unescape` function is an interesting state machine;
* Usage of the awesome [docopt](https://crates.io/crates/docopt) crate.
* There is the usage string from which docopt parses the command line and put
  the result int a `RawArgs` structure. That structure contains a `to_args`
  function that converts the argument to an `Args` structure, which is higher
  level. It involves some duplication because you need to report the same
  information in three different formats: the usage string, the raw arguments
  and the final options.
* The `Args` struct also provides some builder methods that given its options,
  construct things like `Searcher`, `Printer` which do the actual search and
  print work.

search_stream.rs
----------------

Duties:

* search?

What's interesting in there?

* the `InputBuffer` struct is a buffer used to search into. It buffers data read
  from an input and has the option to keep part of the data across multiple
  calls:

      ```
               keep   end
                 |     |
                 v     v
      ---------------------
      | | | | | |x|y|z|w| |
      ---------------------
      ```

  becomes:

      ```
              pos   end
               |     |
               v     v
      ---------------------
      |x|y|z|w|a|b|c| | | |
      ---------------------
      ```

  after the buffer is filled again with new data.

* all the searches are done within this buffer. Additional data is kept when
  it's needed when printing context lines.

* the `Searcher` class fills the buffer, runs the `Grep` matcher and if it
  matches, prints the output according to the options (context lines, line
  numbers) and counts the number of matches.

* note that the `InputBuffer` is taken by reference, so it is meant to be reused
  across different `run`. The `run` itself goes through the whole file and
  consumes `self`. So, to run again you need a new `Searcher` instance. This is
  smart because it reduces the possibility to misuse the struct, by providing a
  new input file without updating the file path accordingly.


search_buffer.rs
----------------

Duties:

* search within a memory mapped file;
* it doesn't support showing context;

What's interesting in there?

* reuses some functions and types of the `search_stream` module; it doesn't need
  the `InputBuffer` struct, since it can freely seek into the memory. It doesn't
  actually seek back because context is not implemented.

Suggestions:

* instead of depending directly on `search_stream` it would have been better to
  move the common parts into a separate parent module.

printer.rs
----------

Duties:

* handles the ouptut logic for the crate;
* supports options and forwards writes to the inner `Terminal` type, which must
  be `Send` also (I still don't understand why).

grep crate
----------

Duties:

* takes a line and performs a regex search into it;

What's interesting in there?

* it builds on top of the `regex` crate and adds some optimizations in
  `literals.rs`.
* BurntSushi blog post already explains very well the implementation details,
  which are very interesting.
* the result is simply a `Match` instance (namely a start and end positions).
* `Grep` is clonable, so it needs to be built only once and then cloned for all
  the workers.

Overall notes
-------------

* extensive usage of `#[inline(always)]` and `#[inline(never)]` directives; I
  wonder if those have been added after profiling and if so, why the compiler
  have failed to identify those correctly. The only possible use case is
  intra-crate inlining, but compiling with `rustc -C lto` already allows to
  inline everything (by slowing down compilation). See [When should I use
  inline](https://internals.rust-lang.org/t/when-should-i-use-inline/598).
* why `eprintln!` macro instead of log macros?