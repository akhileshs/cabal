cabal-testsuite is a suite of integration tests for Cabal-based
frameworks.

How to run
----------

1. Build `cabal-tests` (`cabal new-build cabal-tests`)
2. Run the `cabal-tests` executable. It will scan for all tests
   in your current directory and subdirectories and run them.
   To run a specific set of tests, use `cabal-tests PATH ...`.  You can
   control parallelism using the `-j` flag.

There are a few useful flags which can handle some special cases:

* `--builddir DIR` can be used to manually specify the dist directory
  that was used to build `cabal-tests`; this can be used if
  the autodetection doesn't work correctly (which may be the
  case for old versions of GHC.)

* `--with-ghc PATH` can be used to specify an alternate version of
  GHC to ask the tests to compile with.

* `--with-cabal PATH` can be used to specify the path of a
  `cabal-install` executable.  In this case, tests involving
  this executable will also get run.

How to write
------------

1. Create the package(s) that you need for your test in a
   new directory.  (Currently, tests are stored in `PackageTests`
   and `tests`; we might reorganize this soon.)

2. Create one or more `.test.hs` scripts in your directory, using
   the template:
   ```
   import Test.Cabal.Prelude
   main = setupAndCabalTest $ do
       -- your test code here
   ```

   The general API is that the test is considered to succeed if
   it returns exit 0, and failed if it returned exit code 1.
   Standard output/error are purely for diagnostic purposes.
   `setupAndCabal` test indicates that invocations of `setup`
   should work both for a raw `Setup` script, as well as
   `cabal-install` (if your test works only for one or the
   other, use `setupTest` or `cabalTest`).

   Code runs in the `TestM` monad, which manages some administrative
   environment (e.g., the test that is running, etc.)
   `Test.Cabal.Prelude` contains a number of useful functions
   for testing implemented in this monad, including ways to invoke
   `Setup`, `ghc-pkg`, and other important programs.  For other
   ideas of how to write tests, look at existing `.test.hs`
   scripts.  If you don't see something anywhere, that's probably
   because it isn't implemented. Implement it!

3. Run your tests using `cabal-tests` (no need to rebuild when
   you add or modify a test; it is automatically picked up.)
   The first time you run a test, assuming everything else is
   in order, it will complain that the actual output doesn't match
   the expected output.  Use the `--accept` flag to accept the
   output if it makes sense!

We also support a `.multitest.hs` prefix; eventually this will
allow multiple tests to be defined in one file but run in parallel;
at the moment, these just indicate long running tests that should
be run early (to avoid straggling.)

Hermetic tests
--------------

By default, we run tests directly on the source code that is checked into the
source code repository.  However, some tests require programatically
modifying source files, or interact with Cabal commands which are
not hermetic (e.g., cabal freeze).  In this case, cabal-testsuite
supports opting into a hermetic test, where we first make copy of all
the relevant source code before starting the test.  You can opt into
this mode using the 'withSourceCopy' combinator (search for examples!)
This mode is subject to the following limitations:

* You must be running the test inside a valid Git checkout of the test
  suite (withSourceCopy uses Git to determine which files should be copied.)

* You must 'git add' all files which are relevant to the test, otherwise
  they will not be copied.

* The source copy is still made at a well-known location, so running
  a test is still not reentrant. (See also Known Limitations.)

Design notes
------------

This is the second rewrite of the integration testing framework.  The
primary goal was to use Haskell as the test language (letting us take
advantage of a real programming language, and use utilities provided to
us by the Cabal library itself), while at the same time compensating for
two perceived problems of pure-Haskell test suites:

* Haskell test suites are generally compiled before they run
  (for example, this is the modus operandi of `cabal test`).
  In practice, this results in a long edit-recompile cycle
  when working on tests. This hurts a lot when you would
  like to experimentally edit a test when debugging an issue.

* Haskell's metaprogramming facilities (e.g., Template Haskell)
  can't handle dynamically loading modules from the file system;
  thus, there ends up being a considerable amount of boilerplate
  needed to "wire" up test cases to the central test runner.

Our approach to address these issues is to maintain Haskell test scripts
as self-contained programs which are run by the GHCi interpreter.
This is not altogether trivial, and so there are a few important
technical innovations to make this work:

* Unlike a traditional test program which can be built by the Cabal
  build system, these test scripts must be interpretable at
  runtime (outside of the build system.)  Our approach to handle
  this is to link against the same version of Cabal that was
  used to build the top-level test program (by way of a Custom
  setup linked against the Cabal library under test) and then
  use this library to compute the necessary GHC flags to pass
  to these scripts.

* The startup latency of `runghc` can be quite high, which adds up
  when you have many tests.  To solve this, in `Test.Cabal.Server`
  we have an implementation an GHCi server, for which we can reuse
  a GHCi instance as we are running test scripts.  It took some
  technical ingenuity to implement this, but the result is that
  running scripts is essentially free.

Here is the general outline of how the `cabal-tests` program operates:

1. It first loads the cached `LocalBuildInfo` associated with the
   host build system (which was responsible for building `cabal-tests`
   in the first place.)  This information lets us compute the
   flags that we will use to subsequently invoke GHC.

2. We then recursively scan the current working directory, looking
   for files suffixed `.test.hs`; these are the test scripts we
   will run.

3. For every thread specified via the `-j`, we spawn a GHCi
   server, and then use these to run the test scripts until all
   test scripts have been run.

The new `cabal-tests` runner doesn't use Tasty because I couldn't
figure out how to get out the threading setting, and then spawn
that many GHCi servers to service the running threads.  Improvements
welcome.

Expect tests
------------

An expect test is a test where we read out the output of the test
and compare it directly against a saved copy of the test output.
When test output changes, you can ask the test suite to "accept"
the new output, which automatically overwrites the old expected
test output with the new.

Supporting expect tests with Cabal is challenging, because Cabal
interacts with multiple versions of external components (most
prominently GHC) with different variants of their output, and no
one wants to rerun a test on four different versions of GHC to make
sure we've picked up the correct output in all cases.

Still, we'd like to take advantage of expect tests for Cabal's error
reporting.  So here's our strategy:

1. We have a new verbosity flag +markoutput which lets you toggle the emission
   of '-----BEGIN CABAL OUTPUT-----' and  '-----END CABAL OUTPUT-----'
   stanzas.

2. When someone requests an expect test, we ONLY consider output between
   these flags.

The expectation is that Cabal will only enclose output it controls
between these stanzas.  In practice, this just means we wrap die,
warn and notice with these markers.

An added benefit of this strategy is that we can continue operating
at high verbosity by default (which is very helpful for having useful
diagnostic information immediately, e.g. in CI.)

We also need to deal with nondeterminism in test output in some
situations.  Here are the most common ones:

* Dependency solving output on failure is still non-deterministic, due to
  its dependence on the global package database.  We're tracking this
  in https://github.com/haskell/cabal/issues/4332 but for now, we're
  not running expect tests on this output.

* Tests against Custom setup will build against the Cabal that shipped with
  GHC, so you need to be careful NOT to record this output (since we
  don't control that output.)

* We have some munging on the output, to remove common sources of
  non-determinism: paths, GHC versions, boot package versions, etc.
  Check normalizeOutput to see what we do.  Note that we save
  *normalized* output, so if you modify the normalizer you will
  need to rerun the test suite accepting everything.

* The Setup interface gets a `--enable-deterministic` flag which we
  pass by default.  The intent is to make Cabal more deterministic;
  for example, with this flag we no longer compute a hash when
  computing IPIDs, but just use the tag `-inplace`.  You can manually
  disable this using `--disable-deterministic` (as is the case with
  `UniqueIPID`.)

Some other notes:

* It's good style to put default-language in all your stanzas, so
  Cabal doesn't complain about it (that warning is marked!)  Ditto
  with cabal-version at the top of your Cabal file.

* If you can't get the output of a test to be deterministic, no
  problem: just exclude it from recording and do a manual test
  on the output for the string you're looking for.  Try to be
  deterministic, but sometimes it's not (easily) possible.

Non-goals
---------

Here are some things we do not currently plan on supporting:

* A file format for specifying multiple packages and source files.
  While in principle there is nothing wrong with making it easier
  to write tests, tests stored in this manner are more difficult
  to debug with, as they must first be "decompressed" into a full
  folder hierarchy before they can be interacted with.  (But some
  of our tests need substantial setup; for example, tests that
  have to setup a package repository.  In this case, because there
  already is a setup necessary, we might consider making things easier here.)

Known limitations
-----------------

* Tests are NOT reentrant: test build products are always built into
  the same location, and if you run the same test at the same time,
  you will clobber each other.  This is convenient for debugging and
  doesn't seem to be a problem in practice.
