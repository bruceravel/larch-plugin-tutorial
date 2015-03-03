# The objective function for minimization

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
