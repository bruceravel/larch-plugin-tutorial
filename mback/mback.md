# A useful plugin: the MBACK algorithm

Here we will walk through the creation of a Larch plugin for
[the MBACK algorithm by Tsu-Chien Weng, et al.](http://dx.doi.org/10.1107/S0909049504034193).
MBACK is a method of normalizing XAS $$\mu(E)$$ by matching it to
tabulated values of atomic cross sections.

To accommodate instumental and sample-related effects in the
background of the measured data, this background function is used:

$$\mu_{back}(E) = \left[\sum_0^m C_i(E-E_0)^i\right] + A\cdot\mathrm{erfc}\left((E-E_{em}\right)/\xi)$$

This includes a Legendre polynomial of order *m* added to a
complementary error function used to approximate the effect of Compton
scattering in the discriminator window of an energy-discriminating
detector, which is often respo0nsible for a highly non-linear shape to
the pre-edge region of the measured $$\mu(E)$$ data.

This normalization function has *m+2* parameters -- *m* Legendre
polynomial coefficients and the amplitude (*A*) and width ($$\xi$$) of
the complementary error function.  The centroid of the error function
($$E_{em}$$) is set to the centroid of the emission line associated
with the measured absorption edge.  This function is then minimized to
determine the background function parameters:

$$\frac{1}{n_1} \sum_{1}^{n_1} \left[\mu_{tab}(E) + \mu_{back}(E) + s\mu_{data}(E)\right]^2 + \frac{1}{n_2} \sum_{n_1}^{N} \left[\mu_{tab}(E) + \mu_{back}(E) + s\mu_{data}(E)\right]^2$$

Here, $$\mu_{data}(E)$$ is the measured spectrum and $$\mu{tab}(E)$$
is the tabulated cross section.

The plugin that implements this is 


```python
from larch import (Group, Parameter, isgroup, use_plugin_path, Minimizer)
use_plugin_path('math')
from mathutils import index_of
use_plugin_path('xray')
from cromer_liberman import f1f2
from xraydb_plugin import xray_edge, xray_line, f2_chantler
use_plugin_path('std')
from grouputils import parse_group_args
use_plugin_path('xafs')
from xafsutils import set_xafsGroup
from pre_edge import find_e0

import numpy as np
from scipy.special import erfc

MAXORDER = 6

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
                   em     = Parameter(xray_line(z, n, _larch=_larch)[0], vary=False, _larch=_larch), # erfc centroid
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
            normalization_function  = normalization_function + getattr(getattr(params, attr), 'value') * eoff**j
    
    group.fpp = params.s*mu - normalization_function
    group.mback_params = params


def registerLarchPlugin(): # must have a function with this name!
    return ('_xafs', { 'mback': mback })
    

```




[Larch's First Argument Group convention](http://cars.uchicago.edu/xraylarch/xafs/utilities.html#first-argument-group-convention)
