# Your First Larch Plugin

We will begin with a very simple example of a plugin that provides a
function for adding two numbers together.  I admit that this is a bit
silly, since such a thing already exists and is accessed using the `+`
sign, as in:

```
larch> print 2+3
5
```
That said, we will create a function called `addtwo` and put in a file
called `addtwonumbers.py`.  The idea is that, once this defined, we
will be able to add together two numbers like so:
```
larch> print addtwo(2,3)
5
```
While this certainly is a trivial and unnecessary example, it wil
allow us to examine the contents of a Larch plugin without having to
spend any attention on understanding what the plugin does.

## A simple example

Here is the entire contents of the file `addtwonumbers.py`:

```python
print "hi!"

def addtwo(a=None, b=None):
	"""
	Add two numbers, returning their sum
	"""
    if a==None or b==None:
        Exception('The addtwo function takes two numerical arguments')
    addition = float(a)+float(b)
    return addition

def registerLarchPlugin():
    return ('_math', {'addtwo': addtwo})
```

The definition of the `addtwo` function begins at line 3.  The
signature of the function specifies that it is looking for two
parameters (i.e. the two numbers to be added together) and sets theor
default values to `None`.  The next three lines are the document
string explaining the purpose of the function.  Any user-facing
function, that is an function that you intend a Larch user to interact
with, should have a document string

The next two lines do some simple testing of the input parameters.  If
both parameters are not specified when the function is called, an
Exception will be returned.  This allows someone using the `addtwo`
function to write some code which verifies the successful return of
the function before moving on.

At line 9, the two input numbers are converted to floating point
numbers.  This guarantees that the value returned by `addtwo` will
itself be a floating point number.

Finally at line 10, the sum of the two numbers is returned.

Lines 12 and 13 are the magic lines which turn this from a file
containing python code into an actual Larch plugin.  When Larch reads
this file, it looks for a function called `registerLarchPlugin`.  If
found, it puts symbols into its symbol table.

## The symbol table

"Symbol table" -- that's a bit of jargon.  To understand the symbol
table, let's fire up Larch at the command line and enter this command:

```
larch> show _main
== Group _main: 8 symbols ==
  _builtin: <Group _builtin>
  _io: <Group _io>
  _math: <Group _math>
  _plotter: <Group _plotter>
  _sys: <Group _sys>
  _xafs: <Group _xafs>
  _xray: <Group _xray>
  _xrf: <Group _xrf>
```

This is Larch's master list of symbols, which are all the things --
functions, scalars, arrays, objects, and so on -- that Larch
understands.  In essence, it is the master dictionary of words that
the Larch interpreter can understand at the command line.

Each of the items listed is a Group, which is best thought of as a
simple container for other symbols.  We can examine each of these
Groups, for instance:

```
larch> show _main._plotter
```

This will show the contents of the `_plotter` group.  It is not
actually necessary to specify `_main`.  When using the `show` command,
Larch will assume that you mean to display a symbol in the `_main`
group.  Thus you can simply type:

```
larch> show _plotter
```

and this will show:

```
  close_all_displays: <function close_all_displays, file=/usr/local/share/larch/plugins/wx/plotter.py>
  contour: <function contour, file=/usr/local/share/larch/plugins/wx/plotter.py>
  dtcorrect_viewer: <function dtcorrect_viewer, file=/usr/local/share/larch/plugins/wx/gse_dtcorrect.py>
  get_cursor: <function get_cursor, file=/usr/local/share/larch/plugins/wx/plotter.py>
  get_display: <function get_display, file=/usr/local/share/larch/plugins/wx/plotter.py>
  imshow: <function imshow, file=/usr/local/share/larch/plugins/wx/plotter.py>
  larchgui: <function larchgui, file=/usr/local/share/larch/plugins/wx/scanviewer.py>
  mapviewer: <function mapviewer, file=/usr/local/share/larch/plugins/wx/scanviewer.py>
  newplot: <function newplot, file=/usr/local/share/larch/plugins/wx/plotter.py>
  oplot: <function oplot, file=/usr/local/share/larch/plugins/wx/plotter.py>
  plot: <function plot, file=/usr/local/share/larch/plugins/wx/plotter.py>
  plot_arrow: <function plot_arrow, file=/usr/local/share/larch/plugins/wx/plotter.py>
  plot_axhline: <function plot_axhline, file=/usr/local/share/larch/plugins/wx/plotter.py>
  plot_axvline: <function plot_axvline, file=/usr/local/share/larch/plugins/wx/plotter.py>
  plot_marker: <function plot_marker, file=/usr/local/share/larch/plugins/wx/plotter.py>
  plot_text: <function plot_text, file=/usr/local/share/larch/plugins/wx/plotter.py>
  save_image: <function save_image, file=/usr/local/share/larch/plugins/wx/plotter.py>
  save_plot: <function save_plot, file=/usr/local/share/larch/plugins/wx/plotter.py>
  scanviewer: <function scanviewer, file=/usr/local/share/larch/plugins/wx/scanviewer.py>
  scatterplot: <function scatterplot, file=/usr/local/share/larch/plugins/wx/plotter.py>
  update_trace: <function update_trace, file=/usr/local/share/larch/plugins/wx/plotter.py>
  xrf_oplot: <function xrf_oplot, file=/usr/local/share/larch/plugins/wx/plotter.py>
  xrf_plot: <function xrf_plot, file=/usr/local/share/larch/plugins/wx/plotter.py>
```

Each of the items in the `_plotter` group is function implementing a
task related to displaying data.  For example, `newplot` and `plot`
are the basic functions for plotting line data in two dimensions.

For a function, larch tells you thename of the file in which the
function is defined.  This makes it possible to examine the code which
defines that feature of Larch.

You can see the document string associated with a function like so:

```
larch> print _plotter.newplot.__doc__

    Plot 2-D trace of x, y arrays in a Plot Frame, clearing any
    plot currently in the Plot Frame.

    This is equivalent to
    plot(x, y[, win=1[, new=True[, options]]])

    See Also: plot, oplot
    
```

Note that there are *two* underscores at the beginning *and* the end
of `__doc__`.

## Placing our function in the _math group

Our toy function is a mathematical operation, so it makes sense to
place it in the `_math` group.  If you show the `_math` group, a few
hundred lines of stuff is displayed.  There are a lot of functions,
constants (like `pi`), and other things in that Group.

When we register our plugin at lines 12 and 13 of the example above,
the `addtwo` function is placed in the `_math` group under the symbol
`addtwo`.  That's what the syntax means.

Now copy your `addtwonumbers.py` file to `$HOME/.larch/plugins` (on
Linux or the Mac)  or to `C:\Program Files\larch\plugins` (on
Windows).  That is a location where you can place a plugin file and
load it dynamically from the larch command line.

Fire up larch and do:

```
larch> add_plugin('addtwonumbers')
hi!
True
larch> show _math.addtwo
<function addtwo, file=/home/bruce/.larch/plugins/addtwonumbers.py>
larch> print addtwo(2,3)
5.0
```

The `add_plugin` command is the function which loads a new plugin
dynamically into a running Larch interpreter.  At the next line, we
see that the print statement was executed, verifying that our plugin
was imported into Larch.  Larch then tells us that importing the
plugin was successful by printing `True`.  Had there been a syntax
error or other problem, Larch would have alerted you by printing
`False` and may also provide additional information about the problem.

The `show` line verifies that our `addtwo` function is now in the
symbol table.  Finally, we verify that `addtwo` works without fancy
test case.

## Making Larch recognize our function at startup

Once the plugin is working as expected, we want to put it someplace
where Larch will recognize it and import it at startup without
needing to use the `add_plugin` command.

**:FIXME: get the file locations right!**

Move your file from `$HOME/.larch/plugins/addtwonumbers.py` to
`$HOME/.larch/plugins/math/addtwonumbers.py` or from 
`C:\Program Files\larch\plugins\addtwonumbers.py`
`C:\Program Files\larch\plugins\math\addtwonumbers.py`.  The `math`
folder in the plugins location is where Larch looks to find plugin
files for the `_math` Group.

Now when you start Lacrh, you will immediately find that you do
```
larch> show _math.addtwo
<function addtwo, file=/usr/local/share/larch/plugins/math/addtwonumbers.py>
larch> addtwo(2,3)
5.0
```

That's a plugin.  Now let's do something a bit more interesting.
