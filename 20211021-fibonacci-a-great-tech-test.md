# So you want to make a poor candidate to implement the Fibonacci function

#### Beware here be dragons

Firstly tech tests are _very often_ a complete waste of time and sometimes outright damaging to your
attempt to find a good candidate for your opening.

And a _good_ tech test is a very involved project because you want to simulate the
real world conditions of the job in question which is at least challenging if not absolutely impossible.

Assuming you succeeded in doing so, you still get only a snapshot of your candidates abilities, it is like
criticizing a movie from one frame, although I guess that some critics work exactly in that way...

But just for the sake of the argument let us explore would could/should/might happen if you give such tech tests
and you decide to use _Fibonacci Numbers_ for that purpose...

Did I already mention? _Here be dragons_

## The one response where you just might get lucky and eliminate a bad candidate

is when you get this response

, but let us generously overlook that fact

So we want to have a function fibo(n) so that

    fibo(0) -> 0
    fibo(1) -> 1
    fibo(n) -> fibo(n-1) + fibo(n-2)

### Failing ...

```haskell
  fibo(0) = 0
  fibo(1) = 1
  fibo(n) = fibo(n-1) + fibo(n-2)
```

... unless you can convince your interviewer into believing that with
lazy evaluation and memoization the runtime of `O(n)` and space complexity of `O(n)`
justifies this "readable"  code 

### Passing ...

```haskell
    fibo(n) = fibo'(n, 1, 0) where
       fibo(0, _a, b) = b
       fibo(1, a, _b) = a
       fibo(n, a, b)  = fibo(n-1, a+b, a)
```

this code has `O(n)` runtime but `O(0)` space complexity.

However what if `O(log n)` runtime and space complexity was desired (maybe you should have asked?)

### Failing with style

The observation is that the matrix

        ((1, 1), (1, 0)) ^ n  == ((fibo(n+1), fibo(n)), (fibo(n), fibo(n-1)))

therefore the following implementation has runtime `O(log n)` and space complexity `O(log n)`

```haskell
    fibo(n) = x in ((_, x), (_, _)) <- pow ((1, 1)(1, 0)) n where
       pow m 1 = m
       pow m 2*n = mul (pow m n) (pow m n)
       pow m 2*n + 1 = mul m (mul pmn) in pmn <- pow m n where = Matrix Multiplication
```

so you are king until your interviewer tells you that this is a very nice solution but unfortunately he
dislikes the unnecessary multiplications in your algorithm (actually 3/8 are unnecessary)

### Passing with style

There are two key observations about the matrices we use:

1. They are all of the form `((a, b), (b, c))`, therefore  we can write the product of such matrices as

        ((a, b),  ((x, y),         ((a*x + b*y), (a*y + b*z),
                   *              = 
          b, c))   (y, z))          (b*x + c*y), (b*y + c*z))

    which allows us to save one multipilcation, and...

1. We also now that `a = b + c` and `x = y + z` with which we can write the product as

        a*x + b*y,    (b+c)*y + b*z
        b*(y+z) + b*y, b*y + c*z

    which is

        a*x + b*y,       b*y + c*y + b*z
        b*y + b*z + c*y, b*y + c*z

    and has only 5 products to be computed

As a matter of fact we can forget matrices and define `(fibo(n+1), fibo(n), fibo(n-1) = (1, 1, 0) ^ n` where
multiplication is defined as follows

          (a, b, c) * (x, y, z)  = (a*x + b*y, b*y + c*y + b*z, b*y + c*z)

```elixir
    @base {1, 1, 0}
    def fibo(n) do
      with {_, x, _} <- pow(@base, n), do: x
    end
    def mul({a, b, c}, {x, y, z}), do: {a*x + b*y, b*y + c*y + b*z, b*y + c*z}
    def pow(triple, n)
    def pow(_triple, 0), do: {0, 0, 0}
    def pow(triple, 1), do: triple
    def pow(triple, n) do
      t_2 = pow(triple, floor(n/2))
      squared = mul(t_2, t_2) # Multiplication # 1
      case rem(n, 2) do
        0 -> squared
        1 -> mul(triple, squared) # Multiplication # 2
      end
    end
```

### Correcting a (Stupid) Mistake

_Obviously_ it is a bad idea to compute `pow(@base, n)` when `pow(@base, n-1)` will do, right?

Now wait a minute, how many multiplications do we have to calculate `pow(@base, n)` .
`log(n)` for Multiplication #1 and Multiplication #2 is executed for each 1 digit in the binary
representation of `n`.

So the costly case is `n = 2^k - 1` and by subtracting 1 from n we might just hit that case from the ideal case `n = 2^k - 1`
However there seems to be no amortisation for that worst case scenario because by subtracting 1 from `2^k - 1` we only save one 1 in
the digital representation of n.

Well this is not a valid argument, as we will add n+1 `1`s in one case, but remove one `1` in many cases, let us have a look to get a better 
understanding of this:

     n |  Number of 1's in n |   n-1     |      Number of 1's in n-1  |    gain/loss
     --|---------------------|-----------|----------------------------|----------------
   10000  |   1              | 1111      |           4                | -3
    1111  |   4              | 1110      |           3                |  1
    1110  |   3              | 1101      |           3                |  0
    1101  |   3              | 1100      |           2                |  1
    1100  |   2              | 1011      |           3                | -1
    1011  |   3              | 1010      |           2                |  1
    1010  |   2              | 1001      |           2                |  0
    1001  |   2              | 1000      |           1                |  1
                                                              SUM:       0

so we do get amortisation (which could probably also be seen by a statistical argument)

However we have the luxury that we can chose any k near n and in the following sense:

   fibo(n) = {1, 1, 0}^n -> take middle element or
   fibo(n) = {1, 1, 0}^(n-1) -> take first element or
   fibo(n) = {1, 1, 0}^(n+1) -> take last element

With that information in hand it might turn out interesting to see if calculating which of these three numbers has the least 1s and use it for
our calculation might be a real winner for large `n`s, especially as this is **one** instruction on modern chipsets, or at least a very 
efficient calculation.

That said, you might not have access to this efficient implementation in the language you are using though.

If we look at it in a different way it becomes even clearer, subtracting 1 from a binary number can at most remove one 1 but add
an arbitrary amount to it.

If we follow this logic we still made a _stupid_ mistake, but the other way round, seems that we should have computed
`pow(@base, n+1)`!
### Benchmarks

### Final Thoughts

The solution using matrix multiplication seems very elegant and asymptotically efficient.
However by reflecting on the special properties of these matrices we can come up with a
simpler algebra of a multiplicative representation of Fibonacci Numbers that allow for
`O(log(n))` complexity algorithms based on the nth power.

But proposing even that algorithm without either clarifying what kind of optimality is looked for, space or runtime, or
on the other hand presenting the constant space, `O(n)` runtime algorithm might be considered a bad approach to this
test.
