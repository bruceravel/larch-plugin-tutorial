# Writing your first plugin

Here is a very simple plugin

```python
print "hi!"

def addtwo(a=None, b=None):
    if a==None or b==None:
        Exception('The addtwo function takes two numerical arguments')
    addition = float(a)+float(b)
    return addition

def registerLarchPlugin():
    return ('_math', {'addtwo': addtwo})
```
