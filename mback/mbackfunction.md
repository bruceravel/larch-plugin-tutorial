# The implementation of the MBACK function

Here is the entire `mback` function:

```python
def mback(energy, mu, group=None, order=3, z=None, edge='K', e0=None, emin=None, emax=None,
          whiteline=None, form='mback', tables='cl', fit_erfc=False, return_f1=False,
          _larch=None):
    """
    Match mu(E) data for tabulated f"(E) using the MBACK algorithm or the Lee & Xiang extension

      energy, mu:    arrays of energy and mu(E)
      order:         order of polynomial [3]
      group:         output group (and input group for e0)
      z:             Z number of absorber
      edge:          absorption edge (K, L3)
      e0:            edge energy
      emin:          beginning energy for fit
      emax:          ending energy for fit
      whiteline:     exclusion zone around white lines
      form:          'mback' or 'lee'
      tables:        'cl' or 'chantler'
      fit_erfc:      True to float parameters of error function

    References:
      * MBACK (Weng, Waldo, Penner-Hahn): http://dx.doi.org/10.1086/303711
      * Lee and Xiang: http://dx.doi.org/10.1088/0004-637X/702/2/970
      * Cromer-Liberman: http://dx.doi.org/10.1063/1.1674266
      * Chantler: http://dx.doi.org/10.1063/1.555974
    """
    order=int(order)
    if order < 1: order = 1 # set order of polynomial
    if order > MAXORDER: order = MAXORDER

    ### implement the First Argument Group convention
    energy, mu, group = parse_group_args(energy, members=('energy', 'mu'),
                                         defaults=(mu,), group=group,
                                         fcn_name='mback')
    group = set_xafsGroup(group, _larch=_larch)

    if e0 is None:              # need to run find_e0:
        e0 = xray_edge(z, edge, _larch=_larch)[0]
    if e0 is None:
        e0 = group.e0
    if e0 is None:
        find_e0(energy, mu, group=group)


    ### theta is an array used to exclude the regions <emin, >emax, and
    ### around white lines, theta=0.0 in excluded regions, theta=1.0 elsewhere
    (i1, i2) = (0, len(energy)-1)
    if emin != None: i1 = index_of(energy, emin)
    if emax != None: i2 = index_of(energy, emax)
    theta = np.ones(len(energy)) # default: 1 throughout
    theta[0:i1]  = 0
    theta[i2:-1] = 0
    if whiteline:
        pre     = 1.0*(energy<e0)
        post    = 1.0*(energy>e0+float(whiteline))
        theta   = theta * (pre + post)
    if edge.lower().startswith('l'):
        l2      = xray_edge(z, 'L2', _larch=_larch)[0]
        l2_pre  = 1.0*(energy<l2)
        l2_post = 1.0*(energy>l2+float(whiteline))
        theta   = theta * (l2_pre + l2_post)


    ## this is used to weight the pre- and post-edge differently as
    ## defined in the MBACK paper
    weight1 = 1*(energy<e0)
    weight2 = 1*(energy>e0)
    weight  = np.sqrt(sum(weight1))*weight1 + np.sqrt(sum(weight2))*weight2


    ## get the f'' function from CL or Chantler
    if tables.lower() == 'chantler':
        f1 = f1_chantler(z, energy, _larch=_larch)
        f2 = f2_chantler(z, energy, _larch=_larch)
    else:
        (f1, f2) = f1f2(z, energy, edge=edge, _larch=_larch)
    group.f2=f2
    if return_f1: group.f1=f1

    n = edge
    if edge.lower().startswith('l'): n = 'L'
    params = Group(s      = Parameter(1, vary=True, _larch=_larch),     # scale of data
                   xi     = Parameter(50, vary=fit_erfc, min=0, _larch=_larch), # width of erfc
                   em     = Parameter(xray_line(z, n, _larch=_larch)[0], vary=False, _larch=_larch),
                   e0     = Parameter(e0, vary=False, _larch=_larch),   # abs. edge energy
                   en     = energy,
                   mu     = mu,
                   f2     = group.f2,
                   weight = weight,
                   theta  = theta,
                   form   = form,
                   _larch = _larch)
    if fit_erfc:
        params.a = Parameter(1, vary=True,  _larch=_larch) # amplitude of erfc
    else:
        params.a = Parameter(0, vary=False, _larch=_larch) # amplitude of erfc

    for i in range(order): # polynomial coefficients
        setattr(params, 'c%d' % i, Parameter(0, vary=True, _larch=_larch))

    fit = Minimizer(match_f2, params, _larch=_larch, toler=1.e-5) 
    fit.leastsq()

    eoff = energy - params.e0.value
    normalization_function = params.a.value*erfc((energy-params.em.value)/params.xi.value) + params.c0.value
    for i in range(MAXORDER):
        j = i+1
        attr = 'c%d' % j
        if hasattr(params, attr):
            normalization_function = normalization_function + getattr(getattr(params, attr),'value') * eoff**j
    
    group.fpp = params.s*mu - normalization_function
    group.mback_params = params
```

It's kind of long, so let's break it down into pieces.

## The function signature

```python
def mback(energy, mu, group=None, order=3, z=None, edge='K', e0=None, emin=None, emax=None,
          whiteline=None, form='mback', tables='cl', fit_erfc=False, return_f1=False,
          _larch=None):
```

The first three lines provide
[the signature of the function](http://docs.python-guide.org/en/latest/writing/style/#function-arguments).
Function signatures are no different in Larch than in normal python.

The next several lines are the document string for the `mback`
function.  This can be read at any time from the larch command line by
`print _xafs.mback.__doc__`.

## Managing the function arguments

The next three lines enforce the type (integer) and value
(1<`order`<6) of the `order` argument, which sets the order of the
Legendre polynomial used in the normalization.

```python
    order=int(order)
    if order < 1: order = 1 # set order of polynomial
    if order > MAXORDER: order = MAXORDER
```

The next four lines enforce
[Larch's First Argument Group convention](http://cars.uchicago.edu/xraylarch/xafs/utilities.html#first-argument-group-convention),
which is a way of tersely specifying the data Group on which the
function is to operate.  This allows the user to specify a Group and
have Larch use the `.energy` and `.mu` attributes of the group in the
function.  While this is fine, my personal preference is not to use
this convention.  For one thing, it seems more clear to me to specify
the energy and mu arrays.  For another, it allows the user to use
Group attributes with names other than `.energy` and `.mu` as the
arguments for the `mback` function.  Those attribute names are not
guaranteed when using Larch's `read_ascii` function or some of its
other IO functionality.

```python
    energy, mu, group = parse_group_args(energy, members=('energy', 'mu'),
                                         defaults=(mu,), group=group,
                                         fcn_name='mback')
    group = set_xafsGroup(group, _larch=_larch)
```

The next six lines are used to set the `e0` argument if it is not
provided.  The default is to use the tabulated value for the specified
`z` and `edge`.  If that doesn't work and no `e0` value is set for the
input data group, Larch's `find_e0` function will be run to determine
a guess for `e0` based on the content of the data.

```python
    if e0 is None:              # need to run find_e0:
        e0 = xray_edge(z, edge, _larch=_larch)[0]
    if e0 is None:
        e0 = group.e0
    if e0 is None:
        find_e0(energy, mu, group=group)
```

## Excluding portions of the data

In certain situations, it is useful to exclude data from the MBACK
fit.  The `emin` and `emax` arguments allow you to exclude data from
the beginning or end of the data range.  That is used, for instance,
if the data contain an edge step from another element, which is a data
feature that this implementation of the MBACK algorithm is not
designed to handle.

Another use of the exclusion array is to exclude the region around a
white line from the fit.  In some cases, the white line carries so
much spectral weight that it skews the fit results, causing the
matched data to be "slanted" relative to the tabulated data.

```python
    (i1, i2) = (0, len(energy)-1)
    if emin != None: i1 = index_of(energy, emin)
    if emax != None: i2 = index_of(energy, emax)
    theta = np.ones(len(energy)) # default: 1 throughout
    theta[0:i1]  = 0
    theta[i2:-1] = 0
    if whiteline:
        pre     = 1.0*(energy<e0)
        post    = 1.0*(energy>e0+float(whiteline))
        theta   = theta * (pre + post)
    if edge.lower().startswith('l'):
        l2      = xray_edge(z, 'L2', _larch=_larch)[0]
        l2_pre  = 1.0*(energy<l2)
        l2_post = 1.0*(energy>l2+float(whiteline))
        theta   = theta * (l2_pre + l2_post)
```

The exclusion array, `theta` is initialized as an array for which each
element is 1.0 and of the same length as the data.  That is, it is
intended to be interpreted on the same energy grid as the data.  For
regions of energy to be excluded from the fit, `theta` is set to 0.0.
This array can then be multiplied by the minimization function, which
has the effect of removing data in the exclusion regions from the
determination of the parameters.

## Weights for the fit

In the minimization function given above, the function is split into
two regions, the first $$n_1$$ data points and all the rest.  Often
the pre-edge region contains far fewer datapoints than the region
above the edge.  To make sure that the normalization function does a
good job of matching the pre-edge data, a different weighting is used.
That is the purpose of the $$\frac{1}{n_1}$$ and $$\frac{1}{n_2}$$
terms in the minimization function.  The weight array accommodates
this feature of the MBACK algorithm.

```python
    weight1 = 1*(energy<e0)
    weight2 = 1*(energy>e0)
    weight  = np.sqrt(sum(weight1))*weight1 + np.sqrt(sum(weight2))*weight2
```

## Tabulated cross section data

The next seven lines gather tabulated values for bare-atom cross
sections from either the Cromer-Liberman of Chatler tables.  These
arrays are gnerated on the data grid, broadened by the core-hole
lifetime, and placed in the data group.  If the `return_cl` flag is
set to True, the real part of the energy-dependent cross-section is
also placed in the data group.

```python
    if tables.lower() == 'chantler':
        f1 = f1_chantler(z, energy, _larch=_larch)
        f2 = f2_chantler(z, energy, _larch=_larch)
    else:
        (f1, f2) = f1f2(z, energy, edge=edge, _larch=_larch)
    group.f2=f2
    if return_f1: group.f1=f1
```

Placing these arrays in the data group is an important feature of this
plugin.  Remember that a Group is simply a container for symbols.  It
should be a **useful** container.  By placing the cross section data
in the group, the user is easily able to make plots of the normalized
data along with the normalization standard.

A plugin which operates on a data Group should always put useful and
relevant arrays, constants, and other symbols in the data Group.

## Setting up the fit parameters

The next 19 lines establish the parameters of the fit.

First, we set a throw-away parameter based on the input `edge` argument.

```python
    n = edge
	if edge.lower().startswith('l'): n = 'L'
```

This will be used to set the fixed value of $$E_{em}$$ in the
normalization function.

Then we make a Group called `params`, which is our container for
holding all the variables, constants, and arrays required to evaluate
the minimization function.

```python
    params = Group(s      = Parameter(1, vary=True, _larch=_larch),     # scale of data
                   xi     = Parameter(50, vary=fit_erfc, min=0, _larch=_larch), # width of erfc
                   em     = Parameter(xray_line(z, n, _larch=_larch)[0], vary=False, _larch=_larch), # erfc centroid
                   e0     = Parameter(e0, vary=False, _larch=_larch),   # abs. edge energy
                   en     = energy,
                   mu     = mu,
                   f2     = group.f2,
                   weight = weight,
                   theta  = theta,
                   form   = form,
                   _larch = _larch)
```

The first parameter set is the scale for the measured data, *s*.  This
is made into a Larch Parameter and flagged as a variable.

If the complementary error function is included in the fit by setting
`fit_ercf` to True, then the width $$\xi$$ is marked as variable
parameter.  The complementary error function centroid $$E_{em}$$ and
the edge energy are set as constants of the fit.  $$E_{em}$$ is set to
the centroid of the emission lines associated with the absorption
edge.

`en`, `mu`, `f2`, `weight`, and `theta` are the arrays needed to
evaluate the minimization function.  It is convenient to place these
arrays in the `params` Group.  `form` is a string.  If it is set to
`lee` then the modification to the minimization function given my
[Lee and Xiang](http://dx.doi.org/10.1088/0004-637X/702/2/970) will be
made.

Next the *a* parameter -- the amplitude of the complementary error
function is set to 0 (which has the effect of setting the entire
complementary error function to 0) if the input arguments so specify.
Other wise, *a* is guessed

```python
    if fit_erfc:
        params.a = Parameter(1, vary=True,  _larch=_larch) # amplitude of erfc
    else:
        params.a = Parameter(0, vary=False, _larch=_larch) # amplitude of erfc
```

Finally, the parameters of the Legendre polynomial are guessed:

```python
    for i in range(order): # polynomial coefficients
        setattr(params, 'c%d' % i, Parameter(0, vary=True, _larch=_larch))
```

This is a tricky bit.  For each order of polynomial specified in the
input arguments, a parameter is guessed.  We want parameters with
names like `c1`, `c2`, and so on.  Iteratively constructing parameter
names like that requires using python's
[`setattr`](https://docs.python.org/2/library/functions.html#setattr)
function.

First the parameter name is made using the
[string formatting operator](https://docs.python.org/2/library/stdtypes.html#string-formatting).
Like the parameters defined above, the Larch's `Parameter` function is
used and the parameter is defined to be a variable of the fit.  The
`setattr` function then makes a parameter symbol, gives it a name like
`c1` or `c2`, and places it in the `params` group.

## Performing the minimization

Finally, the minimization happens.

```python
    fit = Minimizer(match_f2, params, _larch=_larch, toler=1.e-5) 
    fit.leastsq()
```

A Minimizer object is made.  This takes as arguments the name of the
objective function (see below) and the name of Group containing the
parameters of the fit.  Once the Minimizer object is made, the fit is
performed by the call to `leastsq()`.


## Finishing up

Once the fit is finished, we reconstruct the normalization function
using the best fit values.  As before, the normalization function is
constructed as the sum of a complementary error function and a
Legendre polynomial.

```python
    eoff = energy - params.e0.value
    normalization_function = params.a.value*erfc((energy-params.em.value)/params.xi.value) + params.c0.value
    for i in range(MAXORDER):
        j = i+1
        attr = 'c%d' % j
        if hasattr(params, attr):
            normalization_function  = normalization_function + getattr(getattr(params, attr), 'value') * eoff**j
    
    group.fpp = params.s*mu - normalization_function
    group.mback_params = params
```

The
[hasattr](https://docs.python.org/2/library/functions.html#hasattr)
function is used to restrict the order of the polynomial to be the
same as was used in the fit.  If the order was 2 in the fit, there
will not be a symbol in `params` called `c3`.  `hasattr` returns true
only for symbols in `params`.  The repeated use of
[getattr](https://docs.python.org/2/library/functions.html#getattr)
digs the best fit value for each Legendre polynomial coefficient out
of `params`.  That's a bit unwieldy, but is required when probing the
contents of a Group in a computed manner, as shown here.

Finally, the matched data are placed in the data Group, as is Group
containing the parameters of the fit.
