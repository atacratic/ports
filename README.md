# ports

_ports_ is an experimental [Unison](https://unisonweb.org) library for composing stateful message-processing functions.

> Get the code in ucm with `pull https://github.com/atacratic/ports:.trunk .ports`

The premise is that you'd like to write functions with signatures like this:

```haskell
    -- Take an A, and emit zero or more Bs.
    -- We'll often think of A as an input message, and B as an output message.
    f : A ->{Stream B} ()

    -- Take a B, process it with the help of some state G, and emit zero or
    -- more Cs (using ability Foo as we go).
    g : B ->{Store G, Stream C, Foo} ()
```

You then use this library to compose and run these _stream functions_, by first transforming them into the corresponding _port functions_:

```haskell
    -- Take a B port (a consumer of Bs) and transform it into an A port (a
    -- consumer of As), using f.  If the B port has state s and effects g,
    -- then the A port has the same.
    f_ports : Port B s g -> Port A s g

    -- Take a C port and produce a B port, which has extra state G and
    -- uses an extra ability Foo.
    g_ports : Port C s g -> Port B (Tuple G s) {g, Foo}
```

These functions compose directly: `f_ports . g_ports : Port C s g -> Port A (Tuple G s) {g, Foo}`.  This corresponds to wiring up `g` to process the stream of `B`s coming out of `f`.

Once you've finished composing the functions, you can run the result like this:

```haskell
-- see example2.main in the repo for a runnable example, similar to this one
main : A -> G ->{Foo} [C]
main a g =
  go = 'let
    system = f_ports . g_ports
    port : Port A (Tuple G ()) {Stream C, Foo}
    port = system streamer
    Port.inject port <| a
    !Runtime.run
  Stream.toList '(handle !go with Runtime.handler (Cons g ()))
```

`Runtime.run` is a scheduler, which takes all the emitted values and dispatches them into the next function along.

## Motivation

I wanted to write this to explore the possibilities for [actor](https://en.wikipedia.org/wiki/Actor_model)-style programming in Unison, but under the constraint that the 'actors' be composable according to their types.  I'm excited that Unison's ability types can express the necessary stuff so directly.

I also wanted to see if I could get a work scheduler of some description to pass the Unison typechecker.  Unison doesn't have any escape hatches (no `unsafeCoerce`) - or at least, not yet - and it wasn't obvious to me how to write a scheduler without that.

## Assessment and learnings

It kinda works, but I wouldn't recommend you build anything on it just yet: there are some typechecker bugs which make it pretty painful - lots of bouncing between spurious ability check errors and infinite loops during type inference.  See [unison#1790](https://github.com/unisonweb/unison/issues/1790) and [unison#1791](https://github.com/unisonweb/unison/issues/1791).  `ports` makes lots of use of ability signatures that look like `{g, Run s g}`, where the same ability variable `g` appears both as a member of the ability list and as a parameter of another member; I think it's this that Unison is sometimes choking on.  At the time of writing ability typechecking is getting some TLC, and this related issue is under investigation [unison#1792](https://github.com/unisonweb/unison/issues/1792) so things will likely improve.  They probably will need to: the new proposed distributed programming API ([here](https://github.com/unisonweb/distributed/issues/1)) includes similar-looking signatures.  For now, the impact is that getting Unison to accept functions like `main` above requires some awkward contortions - see `example2.main.doc` in the repo for an explanation.

There's awkwardness if you want to write a function `A ->{Stream B, Stream C} ()` - Unison doesn't support functions that require two abilities with the same head type constructor, `Stream B` and `Stream C` in this case.  Currently it will accept the type signature, but choke when you try and call the ability's operator (`emit`), because it can't work out which of the two abilities you mean (even despite the argument type giving a good clue.)  I hope this works in some future version of Unison...  For now the workaround is to define a specialized version of `Stream`: `ability StreamB where emitB : B -> ()`, plus a few similarly specialized supporting functions.  Irritating, but not actually a deal-breaker.

`ports` isn't really the same as the actor model - for example, while an actor can create another actor and send a message to it, a port function can't dynamically change the graph of port functions being run in the same way.  An actor also has its own lifetime, whereas a port function doesn't.  My suspicion is that we're still left at a useful point in the design space, and we've kept an interesting part of the essence of the actor model, namely messages flowing through a graph of stateful processors.

The ports transformation (where we went from `f` to `f_ports`) feels like it touches on something fundamental.  Given a signature `A ->{Stream B, Stream C, Stream D} ()`, we see that actually it corresponds exactly with `P D -> P C -> P B -> P A` (writing `P x` for `Port x s g`).  We've moved the focus from 'functions that emit stuff' to 'functions that consume stuff', and we've flipped the direction of the function arrow.  (Maybe this is where people talk about 'codata'?  LMK if you have insights as to what's going on...)

Adding in state means that as you compose port functions, you also need to compose the type of their state.  So the ports transformation needs to explain how (for example) a `Port A (F, G) g` can crack the `F` and the `G` out of the overall state, and pass them separately to the constituent ports functions.  This is achieved with some lenses that you need to provide as you make the port transformation.  I tried for a while to avoid specifying the state type as an argument to `Port` - but then the scheduler doesn't typecheck, as it can't summon up the relevant state at the right type.

Overall I'm bullish on this general idea, with the following reservations:
- I can't yet explain how it relates to the various sophisticated functional streaming libraries that are out there (e.g. [fs2](https://fs2.io/))
  - Streams libraries seem to focus on mapping/filtering or otherwise transforming the streams of data; whereas with `ports` the focus is on the stateful processors and how they act on each new input - the streams of data flowing between them are not themselves first-class things to be manipulated.
- I fear that all the lensed access to state might be slow.

## Future directions

Here are some things it would be cool to investigate further.

- Demonstrating that you can have a port function wired up to send back into itself.  (You can; I just need to work up the example code again.)
- Scaling up to slightly larger example systems.
- Seeing what a request/response pattern ends up looking like.  How much wiring do you end up needing to do?
- Seeing how to group sets of port functions that all want access to the same state (~ 'actors that can receive multiple message types').
- Investigating analogues to the supervision trees found in actor systems.
- Seeing whether there maybe could actually be some analogue to dynamically creating an actor.
- Working up some effectful 'injector' port functions, e.g. injecting data received from a socket.
- Deciding how to model 'wake me up in x milliseconds'.

## Design

### `Port` and `Run`
The key definitions are the `Port` type and the `Run` ability:

```haskell
-- This is a wrapper for a function that is a 'consumer of a values'.
unique type Port a s g
  = Port (a ->{Run s g, g} ())

-- This ability is a combination of `Store s`, giving access to state, plus an `enqueue`
-- operation which allows further computations to be scheduled for later processing.
ability Run s g where
  put : s ->{Run s g} ()
  get : {Run s g} s
  enqueue : '{g, Run s g} () ->{Run s g} ()
```

### Ports transformation example
The `Run` ability is constructed to let us make the ports transformation - let's see how that works.  Let's take the example of transforming from this

```haskell
g : B ->{Store G, Stream C, Foo} ()
```

to this:

```haskell
g_ports : Port C s g -> Port B (Tuple G s) {g, Foo}
```

We need to wrap `g` with some handlers that deal with the `Store G` and `Stream C` abilities.  First we handle the `Stream C` ability by taking each `Stream.emit c` call that `f` makes, and mapping it through to a call to `Run.enqueue`.  To do this, we need to know what port is going to receive the `c` value.  The library provides the following function, which does the job of converting `Stream` to `Run`.

```haskell
consumeStream : Port a s g -> (x ->{h, Stream a} y) -> x ->{h, Run s g} y
```

Then we need to map `Store` to `Run`.  The interesting point here is that the port that `g_ports` is creating needs to have access not just to the state `G`, but also whatever state is used by the `Port C s g` that will consume `g`'s output stream of `C`s.  So while `g` just uses `Store G`, the port we're building has state `Tuple G s`.  The library provides the following function for this conversion.

```haskell
-- in this example o is (Tuple G s), a is G, b is s
addState : Lens o a -> Lens o b -> (x ->{h, Store a, Run b g} y) -> x ->{h, Run o g} y
```

Suppose we name our lenses as follows:

```haskell
Tuple.lens_a : Lens (Tuple a b) a
Tuple.lens_b : Lens (Tuple a b) b
```

Then the incantation to do the full ports transformation ends up being:

```haskell
g_ports : Port C s g -> Port B (Tuple G s) {g, Foo}
g_ports p =
  transform = Port . (addState lens_a lens_b . consumeStream p)
  transform g
```

The implementation of `g_ports` is mechanically derivable from the type - maybe it could be automatic if Unison adds metaprogramming support.

### Runtime

So far, all we've done is wrapped and transformed our stream functions (`f`/`g`) into ports functions (`f_ports`/`g_ports`) which compose in a more pluggable fashion.  Let's look at how to actually run them.

```haskell
unique type Runtime.State s g
  = Runtime.State ['{Run s g, g} ()] s

Port.inject : Port a s g -> a ->{g, Store (Runtime.State s g)} ()

Runtime.run : '{g, Store (Runtime.State s g)} ()

Runtime.handler : s -> Request (Store (Runtime.State s g)) v -> v
```

`Runtime.run` is a scheduler which dispatches work tracked by `Runtime.State` until there's none left.  The runtime state, which it acts on, is just a queue of functions to be run.  There are two ways that things get on that queue:

1) by being injected 'from outside', usually with a call to `Port.inject`

2) by port functions calling `Run.enqueue` when they are dispatched - i.e. the underlying stream functions emitting values on their output streams.

So to actually make something happen, the steps are:

1) write a function that injects some work and calls `Runtime.run` - that function is in ability `{Store (Runtime.State s g)}`

2) handle that function with `Runtime.handler`.

The only conundrum left is how to actually see some kind of result from all this.  For this example we're going to supply the following as our `Port C s g` - the thing that actually consumes the `C` produced at the end of the chain.

```haskell
streamer : Port a () {Stream a}
streamer = Port emit
```

After all the work we've done to eliminate and handle all the `Stream` abilities in `f` and `g`, we're now going to go the other way and resurface our `C` outputs back as a `Stream C`.

Let's see the example again, and look how it fits together:

```haskell
-- see example2.main in the repo for a runnable example, similar to this one
main : A -> G ->{Foo} [C]
main a g =
  go = 'let
    system = f_ports . g_ports
    port : Port A (Tuple G ()) {Stream C, Foo}
    port = system streamer
    Port.inject port <| a
    !Runtime.run
  Stream.toList '(handle !go with Runtime.handler (Cons g ()))
```

- `system` is our composed port function, of type `Port C s g -> Port A (Tuple G s) {g, Foo}`
- `streamer` is our `Port C () {Stream C}` which we pass to `system`, to yield the `A` port, of type `Port A (Tuple G ()) {Stream C, Foo}`
- We use `Port.inject` to put (a port-transformed version of) `f a` onto our runtime scheduler queue
- We call `Runtime.run` to turn the handle and see what happens.
- All this give us a function `go : '{Store (Runtime.State (Tuple G ()) {Stream C, Foo}), Stream C, Foo} ()`.
- We handle that runtime `Store` ability with `Runtime.handler`, passing in `Cons g ()` as the initial `Tuple G ()` state.
- And then we collect the `Stream C` into a list.

## Repo contents

Here's what you see when you pull:

```
.> pull https://github.com/atacratic/ports:.trunk .ports

  Here's what's changed in .ports after the merge:

  Added definitions:

    1.  type Lens o i (+3 metadata)
    2.  unique type Port a s g (+3 metadata)
    3.  ability Run s g (+3 metadata)
    4.  unique type Runtime.State s g (+3 metadata)
    5.  unique type example1.A (+2 metadata)
    6.  unique type example1.B (+2 metadata)
    7.  unique type example1.C (+2 metadata)
    8.  unique type example2.A (+2 metadata)
    9.  unique type example2.B (+2 metadata)
    10. unique type example2.C (+2 metadata)
    11. unique type example2.F (+2 metadata)
    12. unique type example2.G (+2 metadata)
    13. Lens.Lens                   : (o -> i) -> (i -> o -> o) -> Lens o i
    14. Port.Port                   : (a ->{g, Run s g} ()) -> Port a s g
    15. Run.enqueue                 : '{g, Run s g} () ->{Run s g} ()
    16. Run.get                     : {Run s g} s
    17. Run.put                     : s ->{Run s g} ()
    18. Runtime.State.Runtime.State : ['{g, Run s g} ()] -> s -> State s g
    19. example1.A.A                : Nat -> example1.A
    20. example1.B.B                : Nat -> example1.B
    21. example1.C.C                : Nat -> example1.C
    22. example2.A.A                : Nat -> example2.A
    23. example2.B.B                : Nat -> example2.B
    24. example2.C.C                : Nat -> example2.C
    25. example2.F.F                : Nat -> F
    26. example2.G.G                : Nat -> G
    27. Lens.doc                    : Doc (+2 metadata)
    28. Lens.get                    : Lens o i -> o -> i (+2 metadata)
    29. Lens.get.modify             : ((o -> i) ->{g} o -> i)
                                    -> Lens o i
                                    ->{g} Lens o i (+2 metadata)
    30. Lens.get.set                : (o -> i) -> Lens o i -> Lens o i (+2 metadata)
    31. Lens.set                    : Lens o i -> i -> o -> o (+2 metadata)
    32. Lens.set.modify             : ((i -> o -> o) ->{g} i -> o -> o)
                                    -> Lens o i
                                    ->{g} Lens o i (+2 metadata)
    33. Lens.set.set                : (i -> o -> o) -> Lens o i -> Lens o i (+2 metadata)
    34. Port.doc                    : Doc (+2 metadata)
    35. Port.inject                 : Port a s g -> a ->{g, Store (State s g)} () (+3 metadata)
    36. Port.inject.doc             : Doc (+2 metadata)
    37. Port.unwrap                 : Port a s g -> a ->{g, Run s g} () (+2 metadata)
    38. Run.doc                     : Doc (+2 metadata)
    39. Run.lens                    : Lens o i
                                    -> (x ->{h, Run i g} y)
                                    -> x
                                    ->{h, Run o g} y (+2 metadata)
    40. Run.lens.handler            : Lens o i -> Request (Run i g) x ->{Run o g} x (+2 metadata)
    41. Runtime.State.doc           : Doc (+2 metadata)
    42. Runtime.State.queue         : State s g -> ['{g, Run s g} ()] (+2 metadata)
    43. Runtime.State.queue.modify  : ∀ g s h g1.
                                      (['{g1, Run s g1} ()] ->{h} ['{g, Run s g} ()])
                                      -> State s g1
                                      ->{h} State s g (+2 metadata)
    44. Runtime.State.queue.set     : ∀ g s g1. ['{g, Run s g} ()] -> State s g1 -> State s g (+2 metadata)
    45. Runtime.State.state         : State s g -> s (+2 metadata)
    46. Runtime.State.state.modify  : (o ->{h} o) -> State o g ->{h} State o g (+2 metadata)
    47. Runtime.State.state.set     : state1 -> State state1 g -> State state1 g (+2 metadata)
    48. Runtime.handler             : s -> Request (Store (State s g)) v -> v (+3 metadata)
    49. Runtime.handler.doc         : Doc (+2 metadata)
    50. Runtime.inject              : (x ->{h, Run s g} y)
                                    -> x
                                    ->{h, Store (State s g)} y (+3 metadata)
    51. Runtime.inject.doc          : Doc (+2 metadata)
    52. Runtime.inject.handler      : Request (Run s g) a ->{Store (State s g)} a (+2 metadata)
    53. Runtime.run                 : '{g, Store (State s g)} () (+3 metadata)
    54. Runtime.run.doc             : Doc (+2 metadata)
    55. Store.intoRun               : (x ->{h, Store a, Run a g} y)
                                    -> x
                                    ->{h, Run a g} y (+2 metadata)
    56. Store.intoRun.handler       : Request (Store s) x ->{Run s g} x (+2 metadata)
    57. Store.lens                  : Lens o i
                                    -> (x ->{h, Store i} y)
                                    -> x
                                    ->{h, Store o} y (+2 metadata)
    58. Store.lens.handler          : Lens o i -> Request (Store i) x ->{Store o} x (+2 metadata)
    59. Stream.ToPort.doc           : Doc (+3 metadata)
    60. Stream.ToPort.handler       : Port a s g
                                    -> Request (Stream a) x
                                    ->{Run s g} x (+2 metadata)
    61. Tuple.lens_a                : Lens (Tuple a b) a (+2 metadata)
    62. Tuple.lens_b                : Lens (Tuple a b) b (+2 metadata)
    63. addState                    : Lens o a
                                    -> Lens o b
                                    -> (x ->{h, Store a, Run b g} y)
                                    -> x
                                    ->{h, Run o g} y (+2 metadata)
    64. consumeStream               : Port a s g
                                    -> (x ->{h, Stream a} y)
                                    -> x
                                    ->{h, Run s g} y (+3 metadata)
    65. example1.f                  : example1.A ->{Stream example1.B} () (+2 metadata)
    66. example1.f_ports            : Port example1.B s g -> Port example1.A s g (+3 metadata)
    67. example1.g                  : example1.B ->{Stream example1.C} () (+2 metadata)
    68. example1.g_ports            : Port example1.C s g -> Port example1.B s g (+3 metadata)
    69. example1.main               : [example1.C] (+3 metadata)
    70. example1.main.doc           : Doc (+2 metadata)
    71. example2.f                  : example2.A ->{Stream example2.B, Store F} () (+2 metadata)
    72. example2.f_ports            : Port example2.B s g
                                    -> Port example2.A (Tuple F s) g (+2 metadata)
    73. example2.g                  : example2.B ->{Stream example2.C, Store G} () (+2 metadata)
    74. example2.g_ports            : Port example2.C s g
                                    -> Port example2.B (Tuple G s) g (+2 metadata)
    75. example2.main               : [example2.C] (+3 metadata)
    76. example2.main.doc           : Doc (+2 metadata)
    77. streamer                    : Port a () {Stream a} (+3 metadata)
    78. streamer.doc                : Doc (+2 metadata)

  Tip: You can use `todo` to see if this generated any work to do in this namespace and `test` to
       run the tests. Or you can use `undo` or `reflog` to undo the results of this merge.
```
