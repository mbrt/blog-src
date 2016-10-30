Ripgrep code review
===================

Reviewed version: 0.2.3-15-g0156967.

Big picture
-----------

* The search process is split in two implementations, in the `search_buffer` and
  `search_stream` modules.
* Command line parsing and options are handled in `args.rs`.
* Output is handled by...

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


search_buffer.rs
----------------


Overall notes
-------------

* extensive usage of `#[inline(always)]` and `#[inline(never)]` directives; I
  wonder if those have been added after profiling and if so, why the compiler
  have failed to identify those correctly.
* why `eprintln!` macro instead of log macros?
