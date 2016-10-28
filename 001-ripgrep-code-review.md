Ripgrep code review
===================

main.rs
-------

Duties:

* args parsing and actual searching delegated to mods;
* multi-threading handling;
* matches together args and workers;

What's interesting in there?

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

      ```
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

* At the end of the work, to stop the workers, the main thread just enqueues a
  bunch of "quit" messages (precisely equal to the number of threads); you can
  be sure they will all quit because when one thread picks a quit message it
  immediately terminate, so the rest of them will eventually pick the remaining
  quit messages.

Suggestions:

* Remove `Quit` from the enum, and use instead `Option<Work>`; it feels much
  more ideomatic.
* In `run` refactor out the last block of code into `run_multiple_threads`, so
  it's more consistent with the previous blocks.
* Instead of cluttering the `MultiWorker` class with platform specific members,
  they could have used a wrapper module exporting the correct type, or provide
  a good `type` directive:

      ```
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

Overall notes
-------------

* extensive usage of `#[inline(always)]` and `#[inline(never)]` directives; I
  wander if those have been added after profiling and I'd be interesting to know
  why the compiler failed to identify those correctly.
* why `eprintln!` macro instead of log macros?
