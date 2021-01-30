# ports

_ports_ is an experimental [Unison](https://unisonweb.org) library for composing stateful message-processing functions.

> Get the code from ucm with `pull https://github.com/atacratic/ports:.trunk .ports`

The premise is that you'd like to write functions with signatures like this:

```haskell
    -- Take an A, and emit zero or more Bs
    f : A ->{Stream B} ()

    -- Take a B, process it with the help of some state G, and emit zero or
    -- more Cs (using ability Foo as we go)
    g : B ->{Store G, Stream C, Foo} ()
```

You then use this library to compose and run these functions, by first transforming them into the corresponding _port functions_:

```haskell
    -- Take a B port (a consumer of Bs) and transform it into an A port (a
    -- consumer of As), using f.  If the B port has state s and effects g,
    -- then the A port has the same.
    f_ports : Port B s g -> Port A s g

    -- Take a C port and produce a B port, which has extra state (G) and
    -- ability (Foo).
    g_ports : Port C s g -> Port B (Tuple G s) {g, Foo}
```

These functions compose directly: `f_ports . g_ports : Port C s g -> Port A (Tuple G s) {g, Foo}`.

Once you've finished composing the functions, you can run the result like this:

```haskell
-- see @example2.main in the repo for a runnable example, similar to this one
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

`Runtime.run` is a scheduler, which is taking all the emitted values and dispatching them into the next function along.

### Repo contents

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
