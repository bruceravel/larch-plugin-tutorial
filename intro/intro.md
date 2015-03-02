
# Introduction to Larch plugins

Larch uses a plugin architecture to implement most of its features.
The reading and writing of various kinds of file, most math functions,
all functionality for moving motors and reading detectors, and all
functionality for managing the processing and analysis of synchrotron
data -- these are all managed via plugins.

A plugin is a file written in python with a few lines of special code
which allow the Larcg interpretter to recognize it as a plugin.  While
[the Larch document](http://xraypy.github.io/xraylarch/) includes
[a section](http://xraypy.github.io/xraylarch/devel/index.html) on
proramming with Larch and the many plugins that ship with Larch can
serve as examples, it is a bit challenging to get started writing
Larch plugins.  There are some strange and surprising aspects to Larch
plugins.

After writing the `feffrunner` and `diffkk` plugins as well as the
unit testing framework for the
[feff85exafs](https://github.com/xraypy/feff85exafs) project, I
decided to write this tutorial in hopes of lowering the barrier for
the next participant in Larch.  To Matt's credit, he has built a
powerful platform for doing chores at the beamline and with data
obtained at the beamline.  There is real potential for Larch to become
part of the synchrotron infrastructure.  That is particularly true as
more people make contributions to do more things with larch.

In this tutorial you will learn how plugins are used by Larch and what
goes into creating a new plugin.  We will start with a toy problem,
allowing us to understand what distinguishes a Larch plugin from any
other file containing python code.  We will also see how to
incorporate the plugin into Larch for testing and use.

Then we will build a substantial plugin which implements
[the MBACK algorithm by Tsu-Chien Weng, et al.](http://dx.doi.org/10.1107/S0909049504034193).
This is a way of normalizing XANES data by matching the measured
spectrum to tabulated values of the atomic cross sections.

Finally we will examine the diffkk plugin, which implements a
differential Kramers-Kronig transform, which is used to generate the
energy-dependent atomic scattering factors for elements in real
materials starting from a XAS measurement.  This will demonstrate how
to use object-oriented programming as part of your Larch plugin as
well as demonstrate how to write code which efficiently handled large
amounts of numerical data using [NumPy](http://www.numpy.org/),
python's package for scientific computing.

