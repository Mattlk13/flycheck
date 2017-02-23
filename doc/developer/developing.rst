.. _flycheck-developers-guide:

=================
Developer's Guide
=================

So you want to extend Flycheck, but have no idea where to start?  This guide
will give you an overview of Flycheck internals.

Well, you
shouldn't be afraid to dive into the code itself, as it is well documented.
Nevertheless, it can help to first have an overview of how Flycheck works on the
inside.  This guide will give you an idea of how Flycheck operates on the
inside.

An overview of Flycheck
=======================

The goal of Flycheck is to display errors from external checker programs
directly in the buffer you are editing.  Instead of you manually invoking
``make`` or the compiler for your favorite language, Flycheck takes care of it
for you, collects the errors and displays them right there in the buffer.

How Flycheck does it is rather straightforward.  Whenever a syntax check is
started (see :ref:`flycheck-syntax-checks`), the following happens:

1. First, Flycheck runs the external program as an asynchronous process using
   ``start-process``.  While this process runs, Flycheck accumulates its output.
   See :infonode:`(elisp)Asynchronous Processes` for more information on running
   processes from inside Emacs.
2. When the process exits, Flycheck parses its output in order to collect the
   errors.  The raw output is turned into a list of ``flycheck-error`` objects
   containing, among others, the filename, line, column, message and severity of
   the error.
3. Flycheck then filters the collected errors to keep only the relevant ones.
   For instance, errors directed at other files than the one you are editing are
   discarded.
4. Relevant errors are highlighted by Flycheck in the buffer, according to user
   preference.  By default, each error adds a mark in the fringe at the line it
   occurs, in addition to underlining the symbol at the position of the error
   using overlays (see :infonode:`(elisp)Overlays`).
5. Finally, Flycheck rebuilds the error list buffer.

Flycheck follows this process for all the :ref:`many different syntax checkers
<flycheck-languages>` that are provided by default.

.. note::

   Specifically, the above describes the process of *command checkers*, i.e.,
   checkers that run external programs.  All the checkers defined in
   ``flycheck-checkers`` are command checkers, but command checkers are actually
   instances of *generic checkers*.  See ``flycheck-merlin`` for a case where a
   generic checker is preferred.

Adding a syntax checker to Flycheck
===================================

To add a syntax checker to Flycheck, you need to answer a few questions:

- How to invoke the checker?  What is the name of its program, and what
  arguments should Flycheck pass to it?
- How to parse the checker output?
- What language (or languages) will the checker be used for?

Once you have answered these questions, you merely have to translate the answers
to Emacs Lisp.  Here is the full definition of the ``scala`` checker you can
find in ``flycheck.el``:

.. code-block:: elisp

  (flycheck-define-checker scala
     "A Scala syntax checker using the Scala compiler.

  See URL `http://www.scala-lang.org/'."
    :command ("scalac" "-Ystop-after:parser" source)
    :error-patterns
      ((error line-start (file-name) ":" line ": error: " (message) line-end))
    :modes scala-mode
    :next-checkers ((warning . scala-scalastyle)))

The code is rather self-explanatory; but we'll go through it nonetheless.

- First, we define a checker using ``flycheck-define-checker``.  The argument to
  pass is the name of the checker, as a symbol.  The name is used to refer to
  checker in the documentation, so it should usually be the name of the language
  to check, or of the program used to do the checking, or a combination of both.
  Here, ``scalac`` is the program, but the checker is named ``scala``.  There is
  another Scala checker using ``scalastyle``, with the symbol
  ``scala-scalastyle``.  See ``flycheck-checkers`` for the full list of symbols.

- After the symbol comes the docstring.  This is a documentation string
  answering three questions: 1) What language is this checker for?  2) What is
  the program used? 3) Where can users get this program?

- The rest of the arguments are keyword arguments; their order does not matter,
  but they are usually given in the fashion above.

  - `:command` describes what command to run, and what arguments to pass.  Here,
    we tell Flycheck to run ``scalac -Ystop-after:parser`` on ``source``.  In
    Flycheck, we usually want to get error feedback as fast as possible, that is
    why we will pass any flags that will speed up the invocation of a compiler,
    even at the cost of missing out on some errors.  Here, we are telling
    ``scalac`` to stop after the parsing phase to ensure we are getting syntax
    errors quickly.

    The ``source`` argument is special: it instructs Flycheck to create a
    temporary file containing the content of the current buffer, and to pass
    that temporary file as argument to ``scalac``.  That way, ``scalac`` can be
    run on the content of the buffer, even when the buffer has not been saved
    yet.  The other ways to pass the content of the buffer to the command are
    documented in the docstring of ``flycheck-substitute-argument``.

  - `:error-patterns` describes how to parse the output, using `rx` patterns.
    Here, we expect ``scalac`` to return error messages of the form::

      foo.scala:1: error: Syntax error, unexpected ...

    This is a common output format for compilers.  With `:error-patterns`, we
    tell Flycheck to extract three parts from each line in the output: the
    ``file-name``, the ``line`` number, and the ``message`` content.  These
    three parts are then used by Flycheck to create a ``flycheck-error`` of the
    ``error`` severity.

  - `:modes` is the list of Emacs major modes in which this checker can run.
    Here, we want the checker to run only in buffers with ``scala-mode`` active.

That's it!  This definition alone contains everything Flycheck needs to run
``scalac`` on a Scala buffer and parse its output in order to give error
feedback to the user.

Usually though, you'll want to register the checker as well (see :ref:`Select
checkers`).  For that, you just need to add the checker symbol to
``flycheck-checkers``.  The order of checkers do matter, as only one checker can
be enabled in a buffer at a time.  Usually you want to put the most useful
default as the first checker for that mode.  For instance, Flycheck has a
checker for ``bash`` and ``zsh`` scripts in shell-mode, but `sh-bash` comes
before `sh-zsh` in the list.

Maybe send a PR?  See contributor's guide.

.. code-block:: elisp

  (flycheck-define-checker protobuf-protoc
    "A protobuf syntax checker using the protoc compiler.

  See URL `https://developers.google.com/protocol-buffers/'."
    :command ("protoc" "--error_format" "gcc"
              (eval (concat "--java_out=" (flycheck-temp-dir-system)))
              ;; Add the file directory of protobuf path to resolve import directives
              (eval (concat "--proto_path=" (file-name-directory (buffer-file-name))))
              source-inplace)
    :error-patterns
    ((info line-start (file-name) ":" line ":" column
           ": note: " (message) line-end)
     (error line-start (file-name) ":" line ":" column
            ": " (message) line-end)
     (error line-start
            (message "In file included from") " " (file-name) ":" line ":"
            column ":" line-end))
    :modes protobuf-mode
    :predicate (lambda () (buffer-file-name)))


Writing an extension
====================
