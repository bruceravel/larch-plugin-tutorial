# Boilerplate for the MBACK plugin

First off, at the very end of the file are these lines:

```python
def registerLarchPlugin(): # must have a function with this name!
    return ('_xafs', { 'mback': mback })
```

As discussed in the last chapter, this special construct is what
allows Larch to recognize this file as a plugin rather than just a
file containing python code.  Specifically, it registers the symbol
`mback` as referring to the `mback` function, which we will discuss
below, and places that symbol into the `_xafs` Group.

At the top of the file are these lines:

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
```

The point of the various forms of python's
[import](https://docs.python.org/2/tutorial/modules.html) statement is
to make features from various python modules available to our plugin.

The first line imports some functionality from Larch itself.  The
various `use_plugin_path` lines tell Larch where to find other Larch
plugins.  The lines following the `use_plugin_path` lines tell Larch
to import symbols from other Larch plugins.

We import [NumPy](http.www.numpy.org) so its functionality is
available to our plugin.  Finally, we import the symbol for the
[complementary error function](http://en.wikipedia.org/wiki/Error_function),
`erfc`, from [SciPy](http://scipy.org/).

The last line defines a constant which will be used in several places
in the MBACK plugin.

