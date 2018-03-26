===========
 ``z3.js``
===========

This repo contains a build script to compile `Z3 <https://github.com/Z3Prover/z3/>`_ to WebAssembly using `emscripten <https://github.com/kripken/emscripten/>`_.  To make things more reproducible, the entire build happens in a `Vagrant <https://www.vagrantup.com/>` VM (but the build script should be fairly platform-agnostic).

Loading Z3 is fairly slow (~15 seconds on Chrome, &less than 1 second on Firefox), but verification typically is within a factor 2 to 5 of native performance.

Building
========

Install `Vagrant <https://www.vagrantup.com/>`_, then run this::

   vagrant up
   vagrant ssh
   ./z3.sh

A detailed log is written to ``provision.log``, with an outline printed to stdout.  The first build can take up to two hours (emscripten requires a custom build of LLVM, Z3 is large, and all of this is running in a VM).

The output is written to ``z3w.js``, ``z3w.wasm``, ``z3smt2w.js``, and ``z3wsmt2.js``.

Using the generated code
========================

The build script produces two artefacts: a copy of the full Z3 command line application (``z3w.js``, ``z3w.wasm``), and a Javascript version of a small SMT2 REPL-style API, built by linking ``z3smt2.c`` with ``libz3``.

``z3w.js``
----------

There is a small example script using nodejs for demonstration purposes in the repo::

  $ nodejs z3test.js /nodefs/test.smt2
  sat
  (model
    (define-fun y () Int
      3)
    (define-fun x () Int
      2)
    (define-fun z () Int
      1)
    (define-fun g ((x!0 Int) (x!1 Int)) Int
      (ite (and (= x!0 2) (= x!1 2)) 0
        0))
    (define-fun f ((x!0 Int)) Int
      (ite (= x!0 2) (- 1)
      (ite (= x!0 3) 1
        (- 1))))
  )
  unsat
  (error "line 14 column 10: model is not available")
  unsat

From a webpage the process is roughly the same: write Z3's input to a file using emscripten's virtual file system, and use ``callMain(args)`` to run Z3.  To capture Z3's output, you can pass a custom ``print`` function to the emscripten module upon initialization.

This is all demoed in ``html/z3.html`` (that example also uses a WebWorker to run Z3, keeping the page responsive while it runs).  Try it like this::

  cd html
  cp ../z3-js/z3/z3w.js ../z3-js/z3/z3w.wasm ./
  python3 -m http.server
  # Open your browser to http://localhost:8000/z3.html

``z3smt2w.js``
--------------

This script exposes a small API useful for verifiers (like F* or Dafny) that interact with Z3 on through pipes.  It exposes the following functions::

  // Create a fresh SMT2 context
  Z3_context smt2Init()

  // Globally set `option` to `value`.
  void smt2SetParam(const char* option, const char* value)

  // Send `query` to `ctx` and fetch the response as a string.
  const char* smt2Ask(Z3_context ctx, const char* query)

  // Release the resources used by `ctx`.
  void smt2Destroy(Z3_context ctx)

There is a small example script using nodejs for demonstration purposes in the repo::

  $ nodejs ./z3smt2test.js
  sat
  (model
    (define-fun y () Int
      4)
    (define-fun x () Int
      2)
    (define-fun z () Int
      0)
    (define-fun g ((x!0 Int) (x!1 Int)) Int
      (ite (and (= x!0 2) (= x!1 2)) 0
        0))
    (define-fun f ((x!0 Int)) Int
      (ite (= x!0 2) (- 1)
      (ite (= x!0 4) 1
        (- 1))))
  )
  unsat
  (error "line 1 column 11: model is not available")
  unsat


Check the source code of F*.js for an example of how to use this in a larger application.

Known issues
============

Chrome precompiles WebAssembly programs before running them — this makes startup slow, though verification after that is fast.  The recommendation is to cache compiled modules, but Chrome doesn't (2018-03) allow that yet.

Firefox is much better at this, though the code eventually does run a bit slower.