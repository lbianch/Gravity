---
layout: post
title:  "ROOT: Directories, Histograms, and unique_ptr"
date:   2016-12-19 11:00:00 -0500
categories: programming
author: "L. Bianchini"
---

If we follow the ROOT user's guide, then code like the following ends up being
fairly common:

```c++
TFile* data = TFile::Open("data.root", "read");
TTree* tree = (TTree*)data->Get("tree_name");
/* Code to read in desired branches */
TFile* outputFile = TFile::Open("output.root", "recreate");
TH1D* elPt = new TH1D("el_pt", "Electron p_{T} (GeV)", 100, 20., 120.);
/* Lots of analysis code */
outputFile->Write();
outputFile->Close();
data->Close();
```

Part of the magic of ROOT is that this code doesn't contain memory leaks.
Besides the explicit heap allocation for the histogram, the static method
`TFile::Open` returns heap allocated memory and even the declaration for
`tree` incurs heap allocation as the object is streamed from disk into memory
allocated on-demand.  Yet the garbage collection mechanism handles this just fine
(assuming there are no exceptions which go uncaught).

The way that ROOT avoids the need for such explicit `delete` calls is with a 
relatively simple garbage collector.  When objects are created via heap allocation, the
objects are typically registered in a global object, retrievable via `gROOT`. When an
object goes out of scope it is unregistered and deleted.  Here we can see one problem:
the objects never went out of scope, only the pointers to the objects go out of scope. 

So far that's not too bad, but the situation with histograms is more complicated.
Here, when a histogram is created it is automatically registered to whatever
object is the current directory.  This is handled via a global variable, `gDirectory`.
In a decision that may or may not be obvious, a `TFile` is in fact a subclass of
a `TDirectory` (via `TDirectoryFile`), which means that the call to `TFile::Open`
alters the global `gDirectory` and makes the just-opened file the current directory.
Any histograms created immediately after opening a file are therefore automatically
registered to that file.

This leads to a situation I have seen with relatively new ROOT users: declaration
order matters.  For example, this code:

```c++
TFile* data = TFile::Open("data.root", "read");
TTree* tree = (TTree*)data->Get("tree_name");
TH1D* elPt = new TH1D("el_pt", "Electron p_{T} (GeV)", 100, 20., 120.);
/* Code to read in desired branches */
TFile* outputFile = TFile::Open("output.root", "recreate");
/* Lots of analysis code */
outputFile->Write();
outputFile->Close();
data->Close();
```

does not do the same thing.  In the former code, the histogram is written out
to `output.root`, but here the histogram is not saved at all.  This may be surprising,
but perhaps the idea that `outputFile->Write();` manages to write only some of the
objects in the above code may be just as surprising.  The reason is that in moving
the declaration of `outputFile` after the creation of the histogram, the histogram is
now registered to the input file rather than the output file. The only way to save
the histogram is with an explicit request: `outputFile->WriteTObject(elPt)`.  At this 
point it's also worth mentioning that the fact the former code actually did result in 
writing the histogram to the output file is performed implicitly. The `Write` method 
called on a `TFile` writes all the objects that have been registered to the `TFile`. 
Curiously, though, histograms automatically register themselves to the current directory 
but not all related types do. For example, `TGraph` objects do not and any `TGraph` 
objects that were created would not be saved - even in the correctly formed original code.

With C++11 came `std::unique_ptr`, and there are reasons to prefer to use this type
with ROOT rather than rely on the garbage collection offered by ROOT.  The first
reason that comes to mind is that when the analysis code encounters an error, as such
code is often prone to doing, the stack begins to unwind.  However, the objects in
the stack are just pointers - the pointer itself is deallocated but the underlying
object is now potentially lost.  With a perfect garbage collector, what should happen
is that the destructor is called on `gROOT` as ROOT itself quits and with it the lists
of all ROOT objects go through and invoke the destructor on all of those objects.  In
practice, this is hard to ensure and `TGraph` is a counter-example as objects of that
type are not registered anywhere and could not be garbage collected.  Making it worse,
`gROOT` isn't even a variable - it's a C macro
[(line 374)](https://root.cern.ch/doc/master/TROOT_8h_source.html#l00364)
; the actual pointer is `ROOT::Internal::gROOTLocal`. However, when an exception is
encountered global objects are not destroyed.  This fact is relatively easy to see, eg,

```c++
#include <iostream>
class Foo {
public:
    Foo() { std::cout << "Foo::Foo()" << std::endl; }
    ~Foo() { std::cout << "Foo::~Foo()" << std::endl; }
};

Foo foo;
int main() {
    std::cout << "Entered main()" << std::endl;
    std::cout << "Exiting main()" << std::endl;
    return 0;
}
```

works as one would expect, but changing `return 0;` to `throw 0;` suppresses the
destructor from being called on `foo` and we do not see the `Foo::~Foo()` message.
The same situation occurs with objects that have `static` allocation and an 
uncaught exception is thrown.

The second reason I can see is that we get zero feedback from the ROOT garbage
collector about when objects are destroyed.  By using `std::unique_ptr` we can
actually understand when the object themselves cease to exist.  For example, this
code:

```c++
TH1D* GetHistogram(const std::string& fileName, const std::string& histogramName) {
    TFile* f = TFile::Open(fileName.c_str(), "read");
    TH1D* h = (TH1D*)f->Get(histogramName.c_str());
    f->Close();
    return h;
}
```

will never produce any usable histogram.  The line `f->Close();` invokes the
destructor on the histogram, causing the function to return a pointer to, ultimately,
a random memory address.  If we remove the `f->Close();` line then the histogram
is not destroyed and the pointer returned is perfectly usable, however, we incur
the cost of maintaining an open file handle.  Using such a function repeatedly
would exhaust the available file handles of the OS leading to a crash.  

Luckily ROOT is completely capable of providing a sane solution to this problem.
The answer comes from the `TH1::SetDirectory(TDirectory*)` method, which unregisters
the histogram from whichever directory it was originally registered and registers it
to the directory passed in via pointer.  In the case that `nullptr` is passed in,
the histogram is not registered anywhere.  This behavior can also be turned on by default
by calling `TH1::AddDirectory(kFALSE);`, though even here one needs to be careful to
make this call prior to including any headers which may contain global variables that
contain histograms, which could include reweighting/rescaling/smearing utilities.  Making
this proposition more difficult, in practice one uses utilities provided by other authors
and while the analysis code author may not use globals, their colleagues may not be as
opposed to global variables.  As the analysis code author, then, one would need to also
check all the utilities for global variables as turning off the automatic registration
of histograms may either cause a memory leak -- or the utility author may have relied
on the registration and retrieves histograms via `gROOT->Get(histogramName)`.

As such, I would generally not advise to use `TH1::AddDirectory(kFALSE);` as this leads
to a potential mixed ownership of histograms, where histograms allocated prior to that
do not need to be deleted and should not be stored in `std::unique_ptr` wrappers
while objects allocated after that call must be managed.  Instead, I would advise changing
the `GetHistogram` function to this:

```c++
std::unique_ptr<TH1D> GetHistogram(const std::string& fileName, const std::string& histogramName) {
    std::unique_ptr<TFile> f(TFile::Open(fileName.c_str(), "read"));
    std::unique_ptr<TH1D> h(dynamic_cast<TH1D*>(f->Get(histogramName.c_str())));
    h->SetDirectory(nullptr);
    return h;
}
```

which is still making the assumption that the file exists, is readable, is not corrupt,
and contains a TH1D with the specified name.  The important part, however, is that this
new function will not exhaust file handle resources, will not leak memory associated
with the `TFile`, and will return a pointer to a histogram which does exist and is usable.

Now we can begin to understand the way that `std::unique_ptr` can bring back an old
problem that moderately experienced ROOT users have almost certainly encountered:
a double free error. Consider this code:

```c++
TFile* data = TFile::Open("data.root", "read");
TTree* tree = (TTree*)data->Get("tree_name");
/* Code to read in desired branches */
TFile* outputFile = TFile::Open("output.root", "recreate");
TH1D* elPt = new TH1D("el_pt", "Electron p_{T} (GeV)", 100, 20., 120.);
/* Lots of analysis code */
outputFile->Write();
delete outputFile;
delete elPt;
delete data;
```

At this point, the author of such code has probably realized that the lifetime of
the allocated objects exceeds the lifetime of the function, which assuming it isn't
`main` is non-ideal.  They've also realized that the destructor for a `TFile` first
calls the `Close` method so in fact `delete outputFile;` and `delete data;` will
properly close the file first and unregister the objects from ROOT's garbage collection
system before actually freeing the memory.

Instead, the trouble comes from the order of the first two `delete` calls.  Here,
`delete outputFile;` also deletes all objects registered to it, which
is the histogram pointed to by `elPt`. Therefore, `delete outputFile;` also
calls `delete elPt;` so the explicit call to `delete elPt;` is redundant, despite
the fact that pointer was created via an explicit call to `new`.  This, of course,
is the source of the double free error.  The solution is trivial, either reorder
the first two `delete` calls so that `delete elPt;` is first, or recognize that
it was automagically garbage collected as part of `delete outputFile;` and we
don't need to handle the memory anymore.  Perhaps even setting `elPt = nullptr;`
is a good solution to avoid using an invalid pointer.

How does `std::unique_ptr` bring up this problem again?  If we simply switch
to using `std::unique_ptr<TH1D>` for `elPt` alone, then the destructors are called
in the reverse order of the variable declarations.  This means that the destructor
for `elPt` is called first, then the "destructors" for the raw pointers.  With the
`delete outputFile;` in place, the histogram held by `elPt` is also registered with
the output file, so `delete outputFile;` actually deletes the resource meant to be
managed by `elPt`.  This again results in a double free error, invoked by the destructor
for the `std::unique_ptr` as some other object has ownership.  Even trying to use
`std::shared_ptr` to better represent the memory management of the histogram would
not work as the histogram is not registered as a `std::shared_ptr` inside ROOT.

It's clear that if we want to use `std::unique_ptr` we need to do that
for all ROOT objects.  Going back to the stack unwinding argument, if there is
an uncaught exception then only the raw pointers themselves are deallocated, the
underlying object does not have its destructor called.  Thus, open files are not
properly closed if held by a raw `TFile*`.  With the above code, it can be seen
that switching all pointers to `std::unique_ptr` would result in destructors
being called in the following order:

* elPt
* outputFile
* tree
* data

which is exactly what we want in this case.  However, it's also clear that declaration
order also matters since reversing any two objects would also reverse their destructors.
In the case of `elPt` and `outputFile`, this matters, as does `tree` and `data` for much
the same reason (`tree` is registered with `data`).

The simplest possible code to run interactively in ROOT to generate a double free error
that I have found is:

```c++
root [0] unique_ptr<TH1D> h(new TH1D("h", "test", 10, 0., 10.));
root [1] .q
```

Here, the interactive ROOT session itself takes ownership of the histogram along with the
`std::unique_ptr`.  Thus, there is a valid memory address for `gDirectory` which points to
the `TRint` created by the ROOT session and this is where the created histogram is
registered.  The interactive ROOT session is destroyed when using the ROOT command `.q`
to quit, and the `std::unique_ptr` goes out of scope afterwards.  First the registered
ROOT objects are destroyed, and then the destructor for the `std::unique_ptr` is
invoked which triggers the double free error.
