## ML Experiment Log - NN + Polynomial Solvers

This approach doesn't seem to produce good results.

The idea is to create a neural network completely from polynomials (doesn't really require much), differentiate it and solve for critical points.  Use the critical points to quickly find the weight values that minimize loss.
 

## Rational

Neural networks work well as a universal approximation function.  

Gradient descent is intuitive and simple, but results in a massive amount of wasted work on "guess and check".   Furthermore this tends to find a single minima, get caught up on flat-ish areas of loss.  Gradient descents inefficiency in some ways helps protect against overfitting, and I think is a little bit of a bad hack as a fix for this.

I wondered if we could find a way to imitate neural networks in a mathematical construct which we can quickly find the actual minima of?  I thought polynomial solvers may get us close to this, though I suspected a neural network would have NumVars! number of minima and we'd need a way to prune them (didn't get this far in this article though).

Also, while gradient descent is considered superior to many mathematical methods since in some ways it helps with overfitting, I think there exists some subset of problems that aren't really possible to overfit.  Furthermore, I think with a sufficiently fast training loop more exploration of architecture could be done to find architectures that don't overfit the given problem.


## Experiment

To test out the idea I tried to implement a simple XOR network, with no bias and 2 hidden nodes.


### The "neural network"

This function handles creating a node from some inputs, with a weight symbol per input variable. 
```
def addNode(inputVariables):
  global vnum
  nodeVal = 0
  variables = []

  i = 0
  for iv in inputVariables:
    w = symbols('w{vnum}_{i}'.format(vnum=vnum, i=i), real=True)
    variables.append(w)
    nodeVal = nodeVal + w * iv
    i = i + 1
  
  # should return new expression, new output variable, new weight/bias variables
  vnum = vnum + 1
  return nodeVal, variables
```

Build the actual network and output.
```
  x,y = symbols('x y', real=True)

  vars = []
  o1, v = addNode([x,y])
  vars.extend(v)
  o2, v = addNode([x,y])
  vars.extend(v)

  o21, v = addNode([o1, o2])
  vars.extend(v)
  o22, v = addNode([o1, o2])
  vars.extend(v)
  out, v = addNode([o21, o22])
  vars.extend(v)
```

We can then use `out.subs(values)` to calculate output.

### Loss function

To calculate our loss we'll loop through all our examples and add them together.
```
  # where t[0] is x, t[1] is y, and t[2] is expected output
  examples = ((0,0,0), (0,1,1),(1,0,1),(1,1,0))

  loss = 0
  for ex in examples:
    print('ex', ex, out.subs(x, ex[0]).subs(y, ex[1]))
    loss += ((out.subs(x, ex[0]).subs(y, ex[1]) - ex[2]))**2
```


## Solving

I've tried quite a few variations on this, but don't seem to be getting good results.
```
solution = solve(equations, vars, dict=True)
```

An interesting note, replacing `vars` with `vars.reverse()` causes no solutions to be found.  I think what's happening here is that solve doesn't find an exhaustive list, and vars/vars.reverse changes the order the variables are used to calculate Gröbner basis. 


## Results

Applying polynomial solvers with sympy is not working for a few reasons.

1. Not all results returned (the ones we're interested in).
2. Order of variables in solve matters.
3. Runtime calculation already taking awhile for small network.


## Future attempts

A few thoughts for future work on this approach
- Try numerical solvers, gradient descent probably qualifies here but there may be more effecient options to experiment with.
- Learn more about polynomial solving, and read through the sympy code that handles this.
- Try a layer by layer solve, this would be basically a middle ground of polynomial solve and gradient descent.