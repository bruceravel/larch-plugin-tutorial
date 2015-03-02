# Writing your first plugin

We will begin with a very simple example of a plugin that provides a
function for adding two numbers together.  I admit that this is a bit
silly, since such a thing already exists and is accessed using the `+`
symbol, as in:

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

Here is the entire contents of the file `addtwonumbers.py`:
```prettyprint lang-python linenums
print "hi!"

def addtwo(a=None, b=None):
    if a==None or b==None:
        Exception('The addtwo function takes two numerical arguments')
    addition = float(a)+float(b)
    return addition

def registerLarchPlugin():
    return ('_math', {'addtwo': addtwo})
```
The addtwo function 
