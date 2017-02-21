---
layout: post
title:  "Randomly selecting from weighted samples"
date:   2017-01-19 20:00:00 -0500
categories: ["programming", "python"]
author: L. Bianchini
---

Recently at an interview I was given a small coding question to write a 
function which would return values with differing probability.  The setup
was this:

```python
data = {'A': 8, 'B': 2, 'C': 5}
def random_sample(data):
```

Depending on the real dictionary, more than one solution is possible.  The
option I ultimately ended up with was

```python
import random

def random_sample(data):  # Version 0
    data = {k: v/sum(data.values()) for k, v in data.items()}
    cdf = 0.
    rv = random.random()
    for k, v in data.items():
        cdf += v
        if rv < cdf:
            return k
```

I also mentioned that it would be faster to not call `sum(data.values())`
so many times.  This would lead to, eg,

```python
import random

def random_sample(data):  # Version 1
    cdf = 0.
    rv = random.random()
    n = sum(data.values())
    for k, v in data.items():
        cdf += v / n
        if rv < cdf:
            return k
```

The interviewer said that their followup question, had we had time, would be 
whether the less-than should be strict less than or not.  Based on this,
four things come to mind.  One is that trying to do calculations with `float`
and comparisons with `<` vs `<=` typically are going to be limited by floating
point precision errors so the notion that we can have strictly less than is itself
not exactly realistic.  Second is that it's possible in the above code that
`cdf` never reaches `1.0` at all and there would be a correspondingly small chance
that `rv < cdf` is always `False` in which case this function would return `None`.
Upon reflection this is rather interesting as the interviewer said this was the 
exact correct answer, yet clearly it's potentially bugged.  The solution to that is 
at least trivial: add `return k` after the `for` loop, thanks to Python's scope rules.
Third is that if we consider integers, then we could have solved this without using
probabilities at all:

```python
import random

def random_sample(data):  # Version 2
    n = sum(data.values())
    rv = random.randint(0, n - 1)
    cdf = 0
    for k, v in data.items():
        cdf += v
        if rv < cdf:
            return k
```

Here, the rounding error is impossible so there's no need for a `return k` at the 
end as this function would always return a key and should only return `None` if the
empty dictionary was passed to the function.  Thinking about the test data, however,
suppose that `'A'` is the first key, and on the first iteration we then have `cdf == 8`
meaning we want to return `'A'` for `8` possible values of `rv`.  Since `rv` is chosen from
the range `[0, n)`, this corresponds to the values `[0, 7]` and thus we do want strictly
less than.

Afterwards, a one line solution came to mind and, at the cost of memory, it completely
sidesteps the potential of rounding error resulting in a return value of `None` or the
question of strictly less than.  Instead, we can exploit `random.choice` and list
comprehension:

```python
from random import choice

def random_sample(data):  # Version 3
    return choice([k for k, v in data.items() for _ in range(v)])
```

Going a step further we can even enlist some help from the `collections` module:

```python
from random import choice
from collections import Counter

def random_samples(data):  # Version 4
    return choice(list(Counter(data).elements()))
```

In my opinion, this code is the most expressive though we have to acknowledge the
memory (and therefore computational) cost here.  The memory usage scales linearly 
with `sum(data.values())`.  The method `collections.Counter.elements` actually 
returns an `itertools.chain` which cannot be used in a `len` expression that's 
needed by `random.choice`, explaining the conversion to a `list`.  This technically 
results in potentially unbound memory usage (eg, `{'A': 10**9}` should result in a
`list` that consumes many gigabytes of memory).

Now for a reality check: performance.  I tested this with 100k iterations for
each version and the dictionary `{'A': 10, 'B': 3, 'C': 4}` on Python 3.5.2.  The
results are:

| Version | Time (s)|
|---------|---------|
|       0 |   0.366 |
|       1 |   0.157 |
|       2 |   0.450 |
|       3 |   0.621 |
|       4 |   1.416 |

In version 0, it's either the sum or the dictionary comprehension which is causing
slower performance relative to version 1.  Moving the summation out but retaining
the dictionary comprehension, performance is instead 0.268s meaning roughly half
of the performance difference is the repeated calls to `sum` and half is the
comprehension itself.  Timing `random.random()` versus `random.randint(0, 16)`
reveals that a large difference between generating random floating point numbers
and random integers exists, yielding the decrease in performance seen in version 2.
Version 3 is a little trickier to explain, but there is an overhead in creating
the `list` from the comprehension.  Timing the list comprehension in version 3 vs 
the call to `list(Counter(data).elements())` I find that the former takes 0.40s while 
the latter takes 1.16s.

Cranking up the number of items with `{'A': 350, 'B': 30, 'C': 140}` I found that
versions 3 and 4 start to greatly underperform the other solutions while also switching
their relative ordering:

| Version | Time (s)|
|---------|---------|
|       0 |   0.284 |
|       1 |   0.158 |
|       2 |   0.465 |
|       3 |   2.548 |
|       4 |   2.320 |

Do note that here version 0 has been updated to move the summation out, so the comparable
entry from the previous table is the 0.268s discussed there, indicating that all
solutions are at least somewhat dependent upon the number of items in the collection, 
with the issue clearly affecting versions 3 and 4 the most.  Adding more keys to the
input dictionary while preserving the total number should impact only version 0 as it
needs to store more keys.  Versions 3 and 4 wouldn't be affected by this as they depend
only upon `sum(data.values())`, and versions 1 and 2 clearly don't create any new 
containers to begin with.
