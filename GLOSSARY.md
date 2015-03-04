# Group

A generic container for holding the symbols of functions, constants,
and data.  This is implemented as a python class.

# symbol

A word used by Larch to point to a Group, function, constant, or piece
of data.  This is a word with special meaning at the Larch command
line.  The collection of symbols recognized by Larch is called the
"symbol table".

# XANES

X-ray-Absorption Near-Edge Structure

# XAS

X-ray-Absorption Spectroscopy

# EXAFS

Extended X-ray-Absorption Fine-Structure

# diffKK

Differential Kramers-Kronig transform, an algorithm for invoking the
causality relationship to relate the real part of the
energy-dependent, complex correction to the Thompson scattering
($f'(E)$) to the imaginary part ($f''(E)$), which is obtained from a
measurement of the XAS $\mu(E)$.

# MBACK

A method to normalize X-ray absorption data to tabulated mass
absorption coefficients conceived by T.-C. Weng, G. S. Waldo and
J. E. Penner-Hahn and published in Journal of Synchrotron Radiation
(2005). 12, 506-510, DOI: 10.1107/S0909049504034193

# vectorization

The conversion of a scalar calculation -- processing a single pair of
operands at a time -- to a vector implementation -- processing one
operation on many pairs of operands at once.  This allows you to
optimize algorithms by using your computer's architecture along with
highly optimised vector routines, in this case supplied by NumPy.
