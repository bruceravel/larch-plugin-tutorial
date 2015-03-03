# A useful plugin: the MBACK algorithm

Here we will walk through the creation of a Larch plugin for
[the MBACK algorithm by Tsu-Chien Weng, et al.](http://dx.doi.org/10.1107/S0909049504034193).
MBACK is a method of normalizing XAS $$\mu(E)$$ by matching it to
tabulated values of atomic cross sections.

To accommodate instumental and sample-related effects in the
background of the measured data, this background function is used:

$$\mu_{back}(E) = \left[\sum_0^m C_i(E-E_0)^i\right] + A\cdot erfc\left((E-E_{em}\right)/\xi)$$

This includes a Legendre polynomial of order *m* added to a
complementary error function used to approximate the effect of Compton
scattering in the discriminator window of an energy-discriminating
detector, which is often respo0nsible for a highly non-linear shape to
the pre-edge region of the measured $$\mu(E)$$ data.

This normalization function has *m+3* parameters -- *m* Legendre
polynomial coefficients, the amplitude (*A*) and width ($$\xi$$) of
the complementary error function, and a scaling factor (*s*) for the
measured data.  The centroid of the error function ($$E_{em}$$) is set
to the centroid of the emission line associated with the measured
absorption edge.  This function is then minimized to determine the
background function parameters:

$$\frac{1}{n_1} \sum_{1}^{n_1} \left[\mu_{tab}(E) + \mu_{back}(E) + s\mu_{data}(E)\right]^2 + \frac{1}{n_2} \sum_{n_1}^{N} \left[\mu_{tab}(E) + \mu_{back}(E) + s\mu_{data}(E)\right]^2$$

Here, $$\mu_{data}(E)$$ is the measured spectrum and $$\mu_{tab}(E)$$
is the tabulated cross section.


The plugin that implements this is at
https://github.com/xraypy/xraylarch/blob/mback/plugins/xafs/mback.py .
Here, we will step through the parts of that file.

