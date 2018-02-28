## quickcheck-state-machine

[![Hackage](https://img.shields.io/hackage/v/quickcheck-state-machine.svg)](https://hackage.haskell.org/package/quickcheck-state-machine)
[![Stackage Nightly](http://stackage.org/package/quickcheck-state-machine/badge/nightly)](http://stackage.org/nightly/package/quickcheck-state-machine)
[![Stackage LTS](http://stackage.org/package/quickcheck-state-machine/badge/lts)](http://stackage.org/lts/package/quickcheck-state-machine)
[![Build Status](https://api.travis-ci.org/advancedtelematic/quickcheck-state-machine.svg?branch=master)](https://travis-ci.org/advancedtelematic/quickcheck-state-machine)

`quickcheck-state-machine` is a Haskell library, based
on [QuickCheck](https://hackage.haskell.org/package/QuickCheck), for testing
stateful programs. The library is different from
the
[`Test.QuickCheck.Monadic`](https://hackage.haskell.org/package/QuickCheck/docs/Test-QuickCheck-Monadic.html) approach
in that it lets the user specify the correctness by means of a state machine
based model using pre- and post-conditions. The advantage of the state machine
approach is twofold: 1) specifying the correctness of your programs becomes less
adhoc, and 2) you get testing for race conditions for free.

The combination of state machine based model specification and property based
testing first appeard in Erlang's proprietary QuickCheck. The
`quickcheck-state-machine` library can be seen as an attempt to provide similar
functionality to Haskell's QuickCheck library.

### Example

As a first example, let's implement and test programs using mutable
references. Our implementation will be using `IORef`s, but let's start with a
representation of what actions are possible with programs using mutable
references. Our mutable references can be created, read from, written to and
incremented:

```haskell
data Action (v :: * -> *) :: * -> * where
  New   ::                                     Action v (Opaque (IORef Int))
  Read  :: Reference v (Opaque (IORef Int)) -> Action v Int
  Write :: Reference v (Opaque (IORef Int)) -> Int -> Action v ()
  Inc   :: Reference v (Opaque (IORef Int)) -> Action v ()
```

When we generate actions we won't be able to create arbitrary `IORef`s, that's
why all uses of `IORefs` are wrapped in `Reference v`, where the parameter `v`
will let us use symbolic references while generating (and concrete ones when
executing).

In order to be able to show counterexamples, we need a show instance for our
actions. `IORef`s don't have a show instance, thats why we wrap them in
`Opaque`; which gives a show instance to a type that doesn't have one.

Next, we give the actual implementation of our mutable references. To make
things more interesting, we parametrise the semantics by a possible problem.

```haskell
data Problem = None | Bug | RaceCondition
  deriving Eq

semantics :: Problem -> Action Concrete resp -> IO resp
semantics _   New           = Opaque <$> newIORef 0
semantics _   (Read  ref)   = readIORef  (opaque ref)
semantics prb (Write ref i) = writeIORef (opaque ref) i'
  where
  -- One of the problems is a bug that writes a wrong value to the
  -- reference.
  i' | i `elem` [5..10] = if prb == Bug then i + 1 else i
     | otherwise        = i
semantics prb (Inc   ref)   =
  -- The other problem is that we introduce a possible race condition
  -- when incrementing.
  if prb == RaceCondition
  then do
    i <- readIORef (opaque ref)
    threadDelay =<< randomRIO (0, 5000)
    writeIORef (opaque ref) (i + 1)
  else
    atomicModifyIORef' (opaque ref) (\i -> (i + 1, ()))
```

Note that above `v` is instantiated to `Concrete`, which is essentially the
identity type, so while writing the semantics we have access to real `IORef`s.

We now have an implementation, the next step is to define a model for the
implementation to be tested against. We'll use a simple map between references
and integers as a model.

```haskell
newtype Model v = Model [(Reference v (Opaque (IORef Int)), Int)]

initModel :: Model v
initModel = Model []
```

The pre-condition of an action specifies in what context the action is
well-defined. For example, we can always create a new mutable reference, but
we can only read from references that already have been created. The
pre-conditions are used while generating programs (lists of actions).

```haskell
precondition :: Model Symbolic -> Action Symbolic resp -> Bool
precondition _         New           = True
precondition (Model m) (Read  ref)   = ref `elem` map fst m
precondition (Model m) (Write ref _) = ref `elem` map fst m
precondition (Model m) (Inc   ref)   = ref `elem` map fst m
```

The transition function explains how actions change the model. Note that the
transition function is polymorphic in `v`. The reason for this is that we use
the transition function both while generating and executing.

```haskell
transition :: Ord1 v => Model v -> Action v resp -> v resp -> Model v
transition (Model m) New           ref = Model (m ++ [(Reference ref, 0)])
transition m         (Read  _)     _   = m
transition (Model m) (Write ref i) _   = Model (update ref i         m)
transition (Model m) (Inc   ref)   _   = Model (update ref (old + 1) m)
  where
  Just old = lookup ref m

update :: Eq a => a -> b -> [(a, b)] -> [(a, b)]
update ref i m = (ref, i) : filter ((/= ref) . fst) m
```

Post-conditions are checked after we executed an action and got access to the
result.

```haskell
postcondition :: Model Concrete -> Action Concrete resp -> resp -> Bool
postcondition _         New         _    = True
postcondition (Model m) (Read ref)  resp = lookup ref m == Just resp
postcondition _         (Write _ _) _    = True
postcondition _         (Inc _)     _    = True
```

Finally, we have to explain how to generate and shrink actions.

```haskell
generator :: Model Symbolic -> Gen (Untyped Action)
generator (Model m)
  | null m    = pure (Untyped New)
  | otherwise = frequency
      [ (1, pure (Untyped New))
      , (8, Untyped .    Read  <$> elements (map fst m))
      , (8, Untyped <$> (Write <$> elements (map fst m) <*> arbitrary))
      , (8, Untyped .    Inc   <$> elements (map fst m))
      ]

shrinker :: Action v resp -> [Action v resp]
shrinker (Write ref i) = [ Write ref i' | i' <- shrink i ]
shrinker _             = []
```

To be able to fit the code on a line we pack up all of them above into a
record.

```haskell
sm :: Problem -> StateMachine Model Action IO
sm prb = stateMachine
  generator shrinker precondition transition
  postcondition initModel (semantics prb) id
```

We can now define a sequential property as follows.

```haskell
prop_references :: Problem -> PropertyOf (Program Action)
prop_references prb = monadicSequentialC sm' $ \prog -> do
  (hist, _, res) <- runProgram sm' prog
  prettyProgram sm' hist $
    checkActionNames prog (res === Ok)
  where
  sm' = sm prb
```

If we run the sequential property without introducing any problems to the
semantics function, i.e. `quickCheck (prop_references None)`, then the property
passes. If we however introduce the bug problem, then it will fail with the
minimal counterexample:

```
> quickCheck (prop_references Bug)
*** Failed! Falsifiable (after 19 tests and 3 shrinks):

Model []

    New --> Opaque

Model [(Reference Concrete Opaque,0)]

    Write (Reference (Symbolic (Var 0))) 5 --> ()

Model [(Reference Concrete Opaque,5)]

    Read (Reference (Symbolic (Var 0))) --> 6

Model [(Reference Concrete Opaque,5)]

PostconditionFailed /= Ok
```

Recall that the bug problem causes the write of values ``i `elem` [5..10]`` to
actually write `i + 1`.

Running the sequential property with the race condition problem will not uncover
the race condition.

If we however define a parallel property as follows.

```haskell
prop_referencesParallel :: Problem -> Property
prop_referencesParallel prb = monadicParallel (sm prb) $ \prog ->
  prettyParallelProgram prog =<< runParallelProgram (sm prb) prog
```

And run it using the race condition problem, then we'll find the race
condition:

```
> quickCheck (prop_referencesParallel RaceCondition)
*** Failed! (after 8 tests and 6 shrinks):

Couldn't linearise:

┌────────────────────────────────┐
│ Var 0 ← New                    │
│                       → Opaque │
└────────────────────────────────┘
┌─────────────┐ │
│ Inc (Var 0) │ │
│             │ │ ┌──────────────┐
│             │ │ │ Inc (Var 0)  │
│        → () │ │ │              │
└─────────────┘ │ │              │
                │ │         → () │
                │ └──────────────┘
                │ ┌──────────────┐
                │ │ Read (Var 0) │
                │ │          → 1 │
                │ └──────────────┘
Just 2 /= Just 1
```

As we can see above, a mutable reference is first created, and then in
parallel (concurrently) we do two increments of said reference, and finally we
read the value `1` while the model expects `2`.

Recall that incrementing is implemented by first reading the reference and
then writing it, if two such actions are interleaved then one of the writes
might end up overwriting the other one -- creating the race condition.

We shall come back to this example below, but if your are impatient you can
find the full source
code
[here](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/MutableReference.hs).

### How it works

The rough idea is that the user of the library is asked to provide:

  * a datatype of actions;
  * a datatype model;
  * pre- and post-conditions of the actions on the model;
  * a state transition function that given a model and an action advances the
    model to its next state;
  * a way to generate and shrink actions;
  * semantics for executing the actions.

The library then gives back a bunch of combinators that let you define a
sequential and a parallel property.

#### Sequential property

The *sequential property* checks if the model is consistent with respect to the
semantics. The way this is done is:

  1. generate a list of actions;

  2. starting from the initial model, for each action do the the following:

       1. check that the pre-condition holds;
       2. if so, execute the action using the semantics;
       3. check if the the post-condition holds;
       4. advance the model using the transition function.

  3. If something goes wrong, shrink the initial list of actions and present a
     minimal counterexample.

#### Parallel property

The *parallel property* checks if parallel execution of the semantics can be
explained in terms of the sequential model. This is useful for trying to find
race conditions -- which normally can be tricky to test for. It works as
follows:

  1. generate a list of actions that will act as a sequential prefix for the
     parallel program (think of this as an initialisation bit that setups up
     some state);

  2. generate two lists of actions that will act as parallel suffixes;

  3. execute the prefix sequentially;

  4. execute the suffixes in parallel and gather the a trace (or history) of
     invocations and responses of each action;

  5. try to find a possible sequential interleaving of action invocations and
     responses that respects the post-conditions.

The last step basically tries to find
a [linearisation](https://en.wikipedia.org/wiki/Linearizability) of calls that
could have happend on a single thread.

### More examples

Here are some more examples to get you started:

  * The water jug problem from *Die Hard 3* -- this is a
    simple
    [example](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/DieHard.hs) of
    a specification where we use the sequential property to find a solution
    (counterexample) to a puzzle from an action movie. Note that this example
    has no meaningful semantics, we merely model-check. It might be helpful to
    compare the solution to the
    Hedgehog
    [solution](http://clrnd.com.ar/posts/2017-04-21-the-water-jug-problem-in-hedgehog.html) and
    the
    TLA+
    [solution](https://github.com/tlaplus/Examples/blob/master/specifications/DieHard/DieHard.tla);

  * The
    union-find
    [example](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/UnionFind.hs) --
    another use of the sequential property, this time with a useful semantics
    (imperative implementation of the union-find algorithm). It could be useful
    to compare the solution to the one that appears in the paper *Testing
    Monadic Code with
    QuickCheck* [[PS](http://www.cse.chalmers.se/~rjmh/Papers/QuickCheckST.ps)],
    which the
    [`Test.QuickCheck.Monadic`](https://hackage.haskell.org/package/QuickCheck/docs/Test-QuickCheck-Monadic.html) module
    is based on;

  * Mutable
    reference
    [example](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/MutableReference.hs) --
    this is a bigger example that shows both how the sequential property can
    find normal bugs, and how the parallel property can find race conditions.
    Several metaproperties, that for example check if the counterexamples are
    minimal, are specified in a
    separate
    [module](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/MutableReference/Prop.hs);

  * Circular buffer
    [example](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/CircularBuffer.hs)
    -- another example that shows how the sequential property can find help find
    different kind of bugs. This example is borrowed from the paper *Testing the
    Hard Stuff and Staying Sane*
    [[PDF](http://publications.lib.chalmers.se/records/fulltext/232550/local_232550.pdf),
    [video](https://www.youtube.com/watch?v=zi0rHwfiX1Q)];

  * Ticket
    dispenser
    [example](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/TicketDispenser.hs) --
    a simple example where the parallel property is used once again to find a
    race condition. The semantics in this example uses a simple database file
    that needs to be setup and cleaned up. This example also appears in the
    *Testing a Database for Race Conditions with QuickCheck* and *Testing the
    Hard Stuff and Staying
    Sane*
    [[PDF](http://publications.lib.chalmers.se/records/fulltext/232550/local_232550.pdf),
    [video](https://www.youtube.com/watch?v=zi0rHwfiX1Q)] papers;

  * CRUD webserver where create returns unique
    ids
    [example](https://github.com/advancedtelematic/quickcheck-state-machine/blob/master/example/src/CrudWebserverDb.hs) --
    create, read, update and delete users in a postgres database on a webserver
    using an API written
    using [Servant](https://github.com/haskell-servant/servant). Creating a user
    will return a unique id, which subsequent reads, updates, and deletes need
    to use. In this example, unlike in the last one, the server is setup and
    torn down once per property rather than generate program.

All examples have an associated `Spec` module located in
the
[`example/test`](https://github.com/advancedtelematic/quickcheck-state-machine/tree/master/example/test) directory.
These make use of the properties in the examples, and get tested as part
of
[Travis CI](https://travis-ci.org/advancedtelematic/quickcheck-state-machine).

To get a better feel for the examples it might be helpful to `git clone` this
repo, `cd` into the `example/` directory and fire up `stack ghci` and run the
different properties interactively.

### How to contribute

The `quickcheck-state-machine` library is still very experimental.

We would like to encourage users to try it out, and join the discussion of how
we can improve it on the issue tracker!

### See also

  * The QuickCheck
    bugtrack [issue](https://github.com/nick8325/quickcheck/issues/139) -- where
    the initial discussion about how how to add state machine based testing to
    QuickCheck started;

  * *Finding Race Conditions in Erlang with QuickCheck and
    PULSE*
    [[PDF](http://www.cse.chalmers.se/~nicsma/papers/finding-race-conditions.pdf),
    [video](https://vimeo.com/6638041)] -- this is the first paper to describe
    how Erlang's QuickCheck works (including the parallel testing);

  * *Linearizability: a correctness condition for concurrent
    objects* [[PDF](https://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf)], this
    is a classic paper that describes the main technique of the parallel
    property;

  * Aphyr's blogposts about [Jepsen](https://github.com/jepsen-io/jepsen), which
    also uses the linearisability technique, and has found bugs in many
    distributed systems:

    - [Knossos: Redis and linearizability](https://aphyr.com/posts/309-knossos-redis-and-linearizability);
    - [Strong consistency models](https://aphyr.com/posts/313-strong-consistency-models);
    - [Computational techniques in Knossos](https://aphyr.com/posts/314-computational-techniques-in-knossos);
    - [Serializability, linearizability, and locality](https://aphyr.com/posts/333-serializability-linearizability-and-locality).

  * The use of state machines to model and verify properties about programs is
    quite well-established, as witnessed by several books on the subject:

      - [Specifying
        Systems](https://www.microsoft.com/en-us/research/publication/specifying-systems-the-tla-language-and-tools-for-hardware-and-software-engineers/):
        The TLA+ Language and Tools for Hardware and Software Engineers.
        Parts of this book are also presented by the author, Leslie
        Lamport, in the following video
        [course](https://lamport.azurewebsites.net/video/videos.html);

      - [Modeling in Event-B](http://www.event-b.org/abook.html): System
        and Software Engineering. Parts of this book are covered in the
        following (video) course given at Microsoft Research by the
        author, Jean-Raymond Abrial, himself:

          + [Lecture 1](https://www.youtube.com/watch?v=2GP1pJINVT4):
            introduction to modeling and Event-B (chapter 1 of the
            book) and start of "controlling cars on bridge" example
            (chapter 2);

          + [Lecture 2](https://www.youtube.com/watch?v=M8nvVaZ74wA):
            refining the "controlling cars on a bridge" example
            (sections 2.6 and 2.7);

          + [Lecture 3](https://www.youtube.com/watch?v=Y5OUtq8cdV8):
            design patterns and the "mechanical press controller"
            example (chapter 3);

          + [Lecture 4](https://www.youtube.com/watch?v=ku-lfjxM4WI):
            sorting algorithm example (chapter 15);

          + [Lecture 5](https://www.youtube.com/watch?v=C0tpgPOKAyg):
            designing sequential programs (chapter 15);

          + [Lecture 6](https://www.youtube.com/watch?v=i-GKHZAWWjU):
            status report of the hypervisor that Microsoft Research are
            developing using Event-B.

      - [Abstract State Machines](http://www.di.unipi.it/~boerger/AsmBook/): A
        Method for High-Level System Design and Analysis.

    The books contain general advice how to model systems using state machines,
    and are hence relevant to us. For shorter texts on why state machines are
    important for modeling, see:

      - Lamport's
        [*Computation and State Machines*](https://www.microsoft.com/en-us/research/publication/computation-state-machines/);

      - Gurevich's
        [*Evolving Algebras 1993: Lipari Guide*](https://www.microsoft.com/en-us/research/publication/103-evolving-algebras-1993-lipari-guide/) and
        *Sequential Abstract State Machines Capture Sequential
        Algorithms*
        [[PDF](http://delta-apache-vm.cs.tau.ac.il/~nachumd/models/gurevich.pdf)].

  * Other similar libraries:

      - Erlang QuickCheck, [eqc](http://quviq.com/documentation/eqc/), the first
        property based testing library to have support for state machines
        (closed source);

      - The Erlang library [PropEr](https://github.com/manopapad/proper) is
        *eqc*-inspired, open source, and has support for state
        machine [testing](http://propertesting.com/);

      - The Haskell
        library [Hedgehog](https://github.com/hedgehogqa/haskell-hedgehog), also
        has support for state machine based testing;

      - [ScalaCheck](http://www.scalacheck.org/), likewise has support for state
        machine
        based
        [testing](https://github.com/rickynils/scalacheck/blob/master/doc/UserGuide.md#stateful-testing) (no
        parallel property);

      - The Python
        library [Hypothesis](https://hypothesis.readthedocs.io/en/latest/), also
        has support for state machine
        based
        [testing](https://hypothesis.readthedocs.io/en/latest/stateful.html) (no
        parallel property).

### License

BSD-style (see the file LICENSE).
