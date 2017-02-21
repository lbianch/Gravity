---
layout: post
title:  "Iterating through a C++ enumeration"
date:   2017-02-20 20:00:00 -0500
categories: ["programming", "c++"]
author: L. Bianchini
---

To give a bit of motivation, when running a particle physics
analysis one needs to consider different systematic effects.  
The correct way to do this involves applying one systematic effect
at a time to the analysis procedure.  Thus, in order to produce
the analysis with systematics requires looping over all possible
systematic effects.  One could handle these systematics as a
string and loop over a container of all strings, or store these
in a JSON/XML file and parse it.  Of course in C++, this is 
describing a single particular value among a list of pre-defined
values which is perhaps best handled by an `enum`.  Still, when
faced with an `enum` type how does one loop over all possible
values?

The key to this can be taken from two perspectives - one is the
manner in which STL containers are looped over prior to C++11 and
the second is that multiple `enum` names can be backed by the same 
value using assignment.  Prior to C++11, one traditionally loops
over an iterator from the container's `begin()` method up to but
excluding the iterator from the container's `end()` method.
With an `enum`, rather than listing all the values if one adds
two extra values named `BEGIN` and `END` as follows:

```c++
enum Systematic {
    BEGIN,
    Nominal = BEGIN,
    Electron_Up,
    // ...
    END
};
```

then one can also loop over the values as

```c++
for(Systematic systematic(BEGIN); systematic != END; systematic=static_cast<Systematic>(systematic+1)) {
    run_analysis(systematic);
}
```

Of course, if one defines an `enum` with gaps in the values then the above 
cannot work as it clearly only increments by one.  With that limitation,
though, we've successfully iterated through an `enum` just by defining
two special values.  The downside is that the loop is downright awful.

We are free to define global operators which might be useful, though.
Consider

```c++
Systematic operator+(Systematic s, int n) {
    return static_cast<Systematic>(static_cast<int>(s) + n);
}
```
If one's confused by the inner `static_cast`, note that `s + n` by
itself would invoke this function.  With this we have at least eliminated
the cast in the loop:

```c++
for(Systematic systematic(BEGIN); systematic != END; systematic=systematic+1) {
    run_analysis(systematic);
}
```

Of course we can go a step further and define

```c++
Systematic& operator+=(Systematic& s, int n) {
    s = static_cast<Systematic>(s + n);
    return s;
}
```

which produces a loop like

```c++
for(Systematic systematic(BEGIN); systematic != END; systematic += 1) {
    run_analysis(systematic);
}
```

For an even more iterator-like loop, defining the pre-fix incrementation
operator as

```c++
Systematic& operator++(Systematic& s) {
    s = static_cast<Systematic>(s + 1);
    return s;
}
```

yields

```c++
for(Systematic systematic(BEGIN); systematic != END; ++systematic) {
    run_analysis(systematic);
}
```

Personally, I don't like the post-fix operator so I won't bother with that.
However, there are at least two things that we can improve upon here.

One is that the `enum` names are in the global scope.  It would be better
if one had to write `Systematic::BEGIN`.  This is also important if one ever
wants to have more than one iterable enumeration without resorting to gross
tricks such as looping over `BEGIN` to `END` for one type and `Begin` to `End`
for another.  Clearly something as simple as a `namespace` would solve this 
scope issue, as would C++11's `enum class`.  Another solution, which I favor, 
is placing the `enum` inside a `class`.

The second thing I don't like is actually having to specify that we're 
looping from `BEGIN` to `END`.  In my typical usage I wanted to loop over 
all systematics so I would like that to be the default behavior.  Ideally,
I would want to write code like this and then figure out how to support
that syntax:

```c++
for(Systematic systematic; systematic; ++systematic) {
    run_analysis(systematic);
}
```

Clearly this requires two additional things beyond just being able to
increment an enumeration value.  First, we need the default value to be
the `BEGIN` value, and we need to support truth testing.  Both of these
problems are easily solved with a `class` as the default constructor
can ensure the correct value is chosen, and truth testing can be handled
via the conversion operator to a boolean, `operator bool()`, which must 
be a member function.  

Thus, the solution looks something like:

```c++
class Systematic {
public:
    enum Mode { BEGIN,
        Nominal = BEGIN,
        Electron_Up,
        Electron_Down,
        // ...
        END
    };
	
    Systematic() : mode(BEGIN) { }
    Systematic(Mode m) : mode(m) { }
	
    operator bool() const {
        return mode != END; 
    }
	
    Systematic& operator++() {
        mode = static_cast<Mode>(mode + 1);
        return *this;
    };
	
private:
    Mode mode;
};
```

This solves most of the problems.  The last points to consider are
what do we want something like `if(systematic == Systematic::Electron_Up)` 
to actually do.  As-is, there's no comparison operator between a 
`Systematic` and a `Systematic::Mode`.  The comparison would require that
a temporary `Systematic` is implicitly constructed via the second constructor
and the comparison is made between two `Systematic` objects using the compiler generated default comparison operator, which simply compares the 
`mode` members.  We could add a function such as

```c++
bool Systematic::operator==(Systematic::Mode m) {
    return mode == m;
}
```

and remove the implicitly constructed `Systematic`.

As a very useful extension of this `enum` inside a `class` pattern is
the ability to then define helper methods that are clearly part of the
class.  For example, a method that returns the name of the systematic mode
as a string, or returns the name of the `TTree` for writing/reading data.
Combining such methods with a `switch` statement is even more powerful
as the compiler can automatically detect missing cases.  In the case that
a particular change to the analysis is made for a variety of modes then
one can end up with long boolean chains, like

````c++
if(systematic == Systematic::Electron_Up || systematic == Systematic::Electron_Down)
```

but a simple member function can fix all that:

```c++
bool Systematic::IsElectron() const {
    switch(mode) {
    case Electron_Up:
    case Electron_Down:
        return true;
    default:
        return false;
    }
}
```

Now, of course with the plain-old `enum` we always had the freedom to
define something like

```c++
bool IsElectron(Systematic s) {
    switch(s) {
    case Electron_Up:
    case Electron_Down:
        return true;
    default:
        return false;
    }
}
```

and use `if(IsElectron(systematic))`.  The object-oriented change to 
`if(systematic.IsElectron())` isn't fundamentally different, aside from
the C++ class being closed to such modifications.

With C++11 syntax, is there opportunity for improvement?  To an extent, yes.
It is possible to enable the following code to work the same as our previous
loop:

```c++
for(Systematic systematic: Systematic()) {
    run_analysis(systematic);
}
```

Perhaps the first approach one would have is to define a function which
essentially just aggregrates all possible systematics using a pre-C++11
style loop, eg,

```c++
std::vector<Systematic> GetSystematics() {
    std::vector<Systematic> systematics;
    for(Systematic systematic; systematic; ++systematic) {
        systematics.push_back(systematic);
    }
    return systematics;
}
```

which generates a loop like

```c++
for(Systematic systematic : GetSystematics()) {
    run_analysis(systematic);
}
```

This, of course, is just iteration through a STL container and therefore
boring.  However, digging into how the C++11 loops work we know that they rely
on a `begin` and `end` function and a dereference operator.  If we choose 
to define these as:

```c++
// static  
Systematic Systematic::begin() {
    return Mode::BEGIN;
}

// static 
Systematic Systematic::end() {
    return Mode::END;
}

const Systematic& Systematic::operator*() const {
    return *this;
}
```

then the loop 

```c++
for(Systematic systematic : Systematic())
```

ends up working as one wishes.  This particular implementation is still
relatively incomplete.  For example, we can only start at the beginning
and we can only stop at the end.  Iterating through just a subset is not 
possible directly - one needs an `if` statement in the body to weed out
undesired cases.  Also note the above `begin` and `end` member methods 
do not need to be `static`; they are only suggested as `static` because
they do not depend on an instance of the class.

Taking this just one step further, we can reduce the repetitiveness of
the C++11 loop with

```c++
for(auto systematic: Systematic())
```

