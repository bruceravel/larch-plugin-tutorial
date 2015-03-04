# The objective function for minimization #

Larch's `minimize` function and its use of objective functions
[is discussed here](http://xraypy.github.io/xraylarch/fitting/minimize.html).
This is a really neat part of Larch because it allows the user to do
almost any kind of fit to almost any kind of data.  The objective
function that implements the MBACK fit is this:

```python
def match_f2(p):
    """
    Objective function for matching mu(E) data to tabulated f"(E) using the MBACK
    algorithm or the Lee & Xiang extension.
    """
    s      = p.s.value
    a      = p.a.value
    em     = p.em.value
    xi     = p.xi.value
    c0     = p.c0.value
    eoff   = p.en - p.e0.value

    norm = a*erfc((p.en-em)/xi) + c0 # erfc function + constant term of polynomial
    for i in range(MAXORDER):        # successive orders of polynomial
        j = i+1
        attr = 'c%d' % j
        if hasattr(p, attr):
            norm = norm + getattr(getattr(p, attr), 'value') * eoff**j
    func = (p.f2 + norm - s*p.mu) * p.theta / p.weight
    if p.form.lower() == 'lee':
        func = func / s*p.mu
    return func
```

There are several aspects of this function worthy of discussion.

First, it takes a single argument, which is the `params` Group from
the `mback` function.  That Group contains all of the parameters,
constants, and arrays needed to evaluate the fit.

Second, it is *not* registered in the call to `registerLarchPlugin`.
This means that `match_f2` is not a symbol that will be readily
available to the user of the Larch command line.  That's OK.  The user
is expected to interact with the `mback` function, which uses this
function.

Third, the return value of the function is the array to be minimized.
The Minimizer's `leastsq` function will alter the values of the
Parameters in the input Group until the sum of squares of the return
array is as small as possible.

Fourth, the evaluation of the return array is slightly different than
the evaluation of the normalization function at the end of the `mback`
function due to the presence of the `theta` and `weight` arrays.  The
`weight` array is used to adjust the significance of different regions
of the data to the evaluation of the fit.  This is also the purpose of
the
[Lee and Xiang extension](http://dx.doi.org/10.1088/0004-637X/702/2/970)
to the MBACK algorithm.  It optionally weights the minimization
function by shape of the $$\mu(E)$$ function, as seen in the next to
last line.  The `theta` array is used to exclude regions of the data
(the beginning or end of the array or the regions around white lines)
from the minimization function by multiplying the minimization
function by 0.0 in those regions.

Finally, this function makes use of [NumPy's](http://www.numpy.org)
vectorized calculations.  Rather than iterating through the energy
range (as a Fortran programmer might do), arrays are added and
multiplied together, relying on NumPy conventions to do the iterations
correctly and efficiently.  As we will see in the
[next chapter](../diffkk/README.md), employing vectorization is a huge
win in terms of execution time.
