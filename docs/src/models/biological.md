# Chemical Reaction Models

The biological models functionality is provided by DiffEqBiological.jl and helps
the user build discrete stochastic and differential equation based systems
biological models. These tools allow one to define the models at a high level
by specifying reactions and rate constants, and the creation of the actual problems
is then handled by the modeling package.

## The Reaction DSL

The `@reaction_network` DSL allows you to define reaction networks in a more
scientific format. Its input is a set of chemical reactions and from them it
generates a reaction network object which can be used as input to `ODEProblem`,
`SDEProblem` and `JumpProblem` constructors.

The basic syntax is:

```julia
rn = @reaction_network rType begin
  2.0, X + Y --> XY               
  1.0, XY --> Z            
end
```

where each line corresponds to a chemical reaction. The input `rType` designates
the type of this instance (all instances will inherit from the abstract type
`AbstractReactionNetwork`).

The DSL can handle several types of arrows, in both backwards and forward
direction. If a bi-directional arrow is used two reaction rates must be
designated. These two reaction networks are identical:

```julia
rn1 = @reaction_network rType begin
  2.0, X + Y → XY               
  1.0, XY > Z       
  1.0, X + Y ← XY               
  0.5, XY < Z           
end
rn1 = @reaction_network rType begin
  (2.0,1.0), X + Y ↔ XY               
  (1.0, 0.5), XY ⟷ Z       
end
```

The empty set can be used for production or degradation and is declared using
either `0` or `∅`. Integers denote the number of each reactant partaking in the
reaction.

```julia
rn1 = @reaction_network rType begin
  2.0, 2X --> 0        
  2.0, ∅ --> X  
end
```

Multiple reactions can be declared in a single line:

```julia
rn = @reaction_network rType begin
  2.0, (X,Y) --> 0                   #Identical to reactions [2.0, X --> 0] and [2.0, Y --> 0]
  (2.0, 1.0), (X,Y) --> 0            #Identical to reactions [2.0, X --> 0] and [1.0, X --> 0]
  2.0, (X1,Y1) --> (X2,Y2)           #Identical to reactions [2.0, X1 --> X2] and [2.0, Y1 --> Y2]
  (2.0,1.0), X + Y ↔ XY              #Identical to reactions [2.0, X + Y --> XY] and [1.0, XY --> X + Y].
  ((2.0,1.0),(1.0,2.0)), (X,Y) ↔ 0   #Identical to reactions [(2.0,1.0), X ↔ 0] and [(1.0,2.0), Y ↔ 0].
end
```

Parameters can be added to the network by declaring them after the reaction
network. Parameters can only exist in the reaction rate and not as a part of the
reaction.

```julia
rn = @reaction_network rType begin
    (kB, kD), X + Y ↔ XY
end kB kD
p = [2.0, 1.0]
```

The parameter set `p` must be passed to the problem constructor. The parameter
values can be changed after the reaction network is defined.

The reaction rate do not need to be constant, but maybe depend on the
concentration of the reactants.

```julia
rn = @reaction_network rType begin
    (1.0,2XY), X + Y ↔ XY
end
```

The hill function `hill(x,v,K,n) = v*(x^n)/(x^n + K^n)` can be used, as well as
the Michaelis Menten function (the hill function with `n = 1`).

```julia
rn = @reaction_network rType begin
    (1.0,hill(XY,1.5,2.0,2)), X + Y ↔ XY
end
```

By using the `@reaction_func` macro it is possible to define your own functions,
which may then be used when creating new reaction networks.

```julia
@reaction_func hill2(x, v, k) = v*x^2/(k^2+x^2)    
rn = @reaction_network rType begin
  (1.0,hill2(XY,1.5,2.0)), X + Y ↔ XY
end
```

Reaction rates are automatically adjusted according mass kinetics, including
taking special account of higher order terms like `2X -->`. This can be disabled
using any non-filled arrow (`⇐, ⟽, ⇒, ⟾, ⇔, ⟺`), in which case the reaction
rate will be exactly as input. E.g the two reactions in

```julia
rn = @reaction_network rType begin
    2.0, X + Y --> XY
    2.0*X*Y, X + Y ⟾ XY
end
```

will both have reaction rate equal to `2[X][Y]`.

Once a reaction network has been created it can be passed as input to either
one of the `ODEProblem`, `SDEProblem` or `JumpProblem` constructors.

```julia
  probODE = ODEProblem(rn, args...; kwargs...)      
  probSDE = SDEProblem(rn, args...; kwargs...)
  probJump = JumpProblem(prob,aggregator::Direct,rn)
```

the output problems may then be used as normal input to the solvers of the `DifferentialEquations` package.

The noise used by the SDEProblem will correspond to the Chemical Langevin Equations.
However it is possible to scale the amount of noise be declaring a noise parameter.
This will be done after declaring the type but before the network.

```julia
rn = @reaction_network BiDir begin
    2.0, X + Y ↔ XY
end
p = [0.5]
```

The noise term is then added as an additional parameter to the network (by default
the last parameter in the parameter array, unless also declared after the reaction network among the other parameters). By reducing (or increasing) the noise term the
amount stochastic fluctuations in the system can be reduced (or increased).

It is possible to access expressions corresponding to the functions determining the deterministic and stochastic development of the network using.

```julia
  f_expr = rn.f_func
  g_expr = rn.g_func
```

This can e.g. be used to generate LaTeX code corresponding to the system.

### Example: Birth-Death Process

```julia
rs = @reaction_network begin
  c1, X --> 2X
  c2, X --> 0
  c3, 0 --> X
end c1 c2 c3
p = (2.0,1.0,0.5)
prob = DiscreteProblem([5], (0.0, 4.0), p)
jump_prob = JumpProblem(prob, Direct(), rs)
sol = solve(jump_prob, Discrete())
```

### Example: Michaelis-Menten Enzyme Kinetics

```julia
rs = @reaction_network begin
  c1, S + E --> SE
  c2, SE --> S + E
  c3, SE --> P + E
end c1 c2 c3
p = (0.00166,0.0001,0.1)
# S = 301, E = 100, SE = 0, P = 0
prob = DiscreteProblem([301, 100, 0, 0], (0.0, 100.0), p)
jump_prob = JumpProblem(prob, Direct(), rs)
sol = solve(jump_prob, Discrete())
```
