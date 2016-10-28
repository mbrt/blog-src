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

      ```
      enum Work {
          Stdin,
          File(DirEntry),
          Quit,
      }
      ```

    So, it could either be:

      * standard input;
      * a file path;
      * indication to terminate.

* At the end of the work, to stop the workers, the main thread just enqueues a
  bunch of "quit" messages (precisely equal to the number of threads); you can
  be sure they will all quit because when one thread picks a quit message it
  immediately terminate, so the rest of them will eventually pick the remaining
  quit messages.
