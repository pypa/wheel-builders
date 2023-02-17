Proposal: Native libraries in wheels
====================================

(very draft! comments welcome!)

Motivation
----------

Wheels and the ecosystem around them (PyPI, pip, etc.) provide a
standard cross-platform way for Python projects to distribute
pre-built packages, including pre-built binary extensions. But
currently it can be rather awkward to distribut binary extension
modules when those modules depend on external native libraries -- for
example, ``numpy`` and ``scipy`` depend on ``blas``, and
``cryptography`` and ``psycopg2`` depend on ``openssl``. Currently,
there are two main approaches that are used: Option 1 is to include
('vendor') the native libraries into the Python package wheels, so
that e.g. ``cryptography`` and ``psycopg2`` each have their own copy
of ``openssl``. This creates bloated downloads and memory usage
(e.g. BLAS libraries can be very large, so shipping multiple copies
quickly adds up), and complicates the process of building and updating
and responding to security problems (every time a new ``openssl`` bug
comes out, the distributors of ``cryptography`` and ``psycopg2`` are
both separately responsible for rerolling their wheels). OTOH, it's
the only approach that currently works consistently and portably
across platforms. Option 2 is to assume/hope that the library will be
somehow provided externally by the system. This is mostly only done
with sdists on Linux and sometimes OS X, and works about as well as
you'd expect, i.e., sorta but not very well, and almost not at all on
Windows.

We can do better.


This proposal
-------------

Here we propose the ``pynativelib`` project: a set of conventions and
tools for packaging native libraries and native executables into
Python wheels, so that we can have wheels like
``openssl-py2.py3-none-win32.whl`` that can be
installed/upgraded/uninstalled via ``pip`` and used by projects like
``cryptography`` and ``psycopg2``; this allows them to share a single
copy of openssl, and when a security fix comes out, then only the
openssl wheel needs to be updated to fix all the dependent packages.

(Rationale for name: as we'll see below, this name will end up in
filenames and metadata attached to native binaries that might become
dissociated from the Python context; including "py" in the name gives
a hint as to where these files come from. And "nativelib" because we
aim to support native libraries, in any language.)


Principles
----------

The pynativelib design should be:

1) Portable: support all platforms that distribute wheels via PyPI
   (currently Windows, OS X, Linux), and provide consistent behavior
   across these platforms (e.g. encapsulate differences in how shared
   libraries work across platforms so that users don't have to learn
   the grim details).

2) Pythonic: these packages should be located using familiar
   sys.path-based lookup rules, so e.g. installing these wheels into a
   virtualenv or anywhere else on PYTHONPATH should Just Work.

3) Play well with others: we expect people will use a mixture of
   pynativelib wheels, vendored libraries, and external
   system-provided libraries, possibly in the same package and
   certainly in the same environment. This should "just
   work". Pynativelibs shouldn't need any special case treatment from
   the broader packaging ecosystem; they should just work with the
   existing system, and without requiring coordination between anyone
   except the pynativelib package builder and the downstream package
   that wants to use it.

   Also, we would like to play nicely with systems like Fedora or
   Debian that will want to package Python libraries and have them
   depend on external system libraries, without requiring lots of
   per-package work by distributors.

The rest of this document outlines my best attempt to meet all of
these requirements.


Example
-------

To orient ourselves, let's start with a quick look at a "typical"
package ``pypkg``, which contains an extension module ``pypkg._ext``
that is linked to the pynativelib ``libssl`` library.

First, in ``pypkg/__init__.py``::

    # This must be done before importing any code that is linked
    # against the pynativelib openssl libraries. It does whatever
    # platform-specific magic is needed to make the libraries
    # accessible; pypkg doesn't know or care about the details.
    import pynativelib_openssl
    pynativelib_openssl.enable()

    # Now we can import ._ext
    from ._ext import ...

And in ``pypkg``'s ``setup.py``::

    from setuptools import setup, Extension

    import pynativelib_openssl as pnlo

    setup(...
      # This doesn't actually work because of the usual problems with
      # setup_requires, but let's assume that some day it will
      # (see PEP 516/517)
      setup_requires=["pynativelib_openssl >= ..."],
      # ask pynativelib_openssl to tell us which versions are
      # ABI-compatible with the one we are building against
      install_requires=pnlo.install_requires(),
      ext_modules = [
        Extension("pypkg._ext", ["pypkg/_ext.c"],
                  # Ask pynativelib_openssl to tell us how to link to it
                  **pnlo.distutils_extension_kwargs()),
      ],
    )

That's it.

Now, to make this work, let's look at ``pynativelib_openssl``. It
might be laid out like::

    pynativelib_openssl/
        __init__.py
        _libs/
            libpynativelib_openssl__ssl.so.1.0.2
        _include/
            openssl.h

(or replace ``libpynativelib_openssl_ssl.so.1.0.2`` with
``libpynativelib_openssl_ssl.1.0.2.dylib`` on OSX and
``pynativelib_openssl_ssl-1.0.2.dll`` on Windows, and on Windows we'll
also need a ``pynativelib_openssl_ssl-1.0.2.lib`` import library. The
general rule is that whatever the library would normally be called, we
rename it to ``<pynativelib package name>__<regular name>``.)

**Rationale for this naming scheme:** (1) It allows tools like
``auditwheel`` to not only recognize that an external dependency of a
wheel is a pynativelib library, but also to extract the name of the
pynativelib package and check that the wheel has an
``Install-Requires`` entry for it. (2) It is possible that there will
be different pynativelib packages that contain different versions of
the same library (e.g., on Windows there are multiple C++ ABIs);
encoding the pynativelib package name into the library name ensures
that these will have distinct names. And the use of a double
underscore avoids ambiguity in case we ever end up with a package
where the ``<package name>`` and ``<regular name>`` portions both
contain underscores, on the assumption that no-one will ever use
double-underscores on purpose in package names.

And ``pynativelib_openssl/__init__.py`` contains code that looks something like::

    # openssl version 1.0.2 + our release 0
    __version__ = "1.0.2.0"

    import os.path
    # Find our directories
    _root = os.path.abspath(os.path.dirname(__file__))
    _libs = os.path.join(_root, "_libs")
    _include = os.path.join(_root, "_include")

    def enable():
        # platform specific code for adding _libs to search path
        # basically prepending the _libs dir to
        #    LD_LIBRARY_PATH on linux
        #    DYLD_LIBRARY_PATH on OS X
        #    PATH on Windows
        ...

    def install_requires():
        # For a library that maintains backwards (but not forward)
        # ABI compatibility:
        return ["pynativelib_openssl >= {}".format(__version__)]
        # In real life openssl's API/ABI is a mess, so we might want to
        # have different parallel-installable packages for different
        # openssl versions or other clever things like that.
        # Either way users don't really need to care.

    def library_name():
        # adjust as necessary for the platform
        return "libpynativelib_openssl__ssl.so.1.0.2"

    # -- WARNING --
    # This is not actually how I'd suggest implementing this function
    # in real life, and there are some functions missing as well.
    # This is just to give the general idea:
    def distutils_extension_kwargs():
        return {"include_dirs": [_include],
                "library_dirs": [_lib],
                # adjust as necessary for platform
                "libraries": ["pynativelib_openssl__ssl"],
                }

Of course there are also lots of other possible variations here -- the
person putting together the pynativelib package could use whatever
conventions they prefer for what exactly to name the library and
include directories, and how to handle version numbers in a consistent
way cross-platform, etc. The crucial thing here is to settle on an
interface between the pynativelib package and the downstream package
that uses it.


Does this work?
---------------

Yes! Not only does it allow us to distribute packages that link
against openssl and other libraries in a sensible way, but it fulfills
our principles: it works on all supported platforms, the lookup
happens via ``import`` so it follows standard Python rules, and it
plays well with others in that pynativelib-installed libraries can
coexist without interference with libraries coming from the external
system, or vendored into other projects' wheels, or whatever.

The interaction of the first and last points is somewhat subtle,
though: why does it work on all these different platforms? Their
linkers work differently in *lots* of ways, but they all (1) provide
some way to alter the library search order in a process-wide fashion
that also gets passed to child processes spawned with ``os.execv``,
and (2) use the library name as a crucial part of their symbol
resolution process.

On Windows and OS X this is straightforward: they resolve symbols
using a "two-level namespace", i.e., binaries don't just say "I need
``SSL_read``", they say "I need ``SSL_read`` from ``libssl``". So if
one binary says that, and another says "I need ``SSL_read`` from
``pynativelib_openssl__libssl``", then there's no danger of the two
``SSL_read``'s getting mixed up.

On Linux, it's a bit more complicated: on Linux, binaries do just say
"I need ``SSL_read``", and then there's a separate list of libraries
that get searched. But, thanks to the magic of ELF scopes (don't ask),
every Python extension module gets its own independent list of
libraries to be searched -- so if extension module A puts
``pynativelib_openssl__libssl`` into its list, and extension module B
puts ``libssl`` into its list, then they'll both end up finding the
version of ``SSL_read`` that they wanted.

These platforms' dynamic linkers also provide all kinds of other neat
features: ``RPATH`` variants, SxS manifests, etc., etc. But it turns
out that these are all unnecessary for our purposes, and are too
platform-specific to be useful in any case, so we ignore them and
stick to the basics: search paths and unique library names.

[EDIT, 5 years later: it turns out I *dramatically* underestimated
how messy macOS makes this. It's still doable, just way more complicated
-- for details see
https://discuss.python.org/t/native-dependencies-in-other-wheels-how-i-do-it-but-maybe-we-can-standardize-something/23913/11
]


Other niceties
..............

**What if I want to link to two pynativelib libraries, and I'm using
distutils/setuptools?** Then you should write some ugly thing like::

    def combined_extension_kwargs(*pynativelib_objs):
        combined = defaultdict(list)
        for po in pynativelib_objs:
            for k, v in po.items():
                combined[k] += v
        return dict(combined)

    Extension(...,
              **combined_extension_kwargs(
                   pynativelib_openssl,
                   ...))


**Isn't that kinda ugly?** Yeah, because interfacing with distutils is
just kinda intrinsically obnoxious -- the interface distutils gives
you is over-complicated and yet not rich enough. This is the best I
can come up with, and there's some more rationale details below, but
really the long-term solution is to modify setuptools and whatever
other build systems we care about so that you can just say "here's an
object that implements the standard pynativelib interface, pls make
that happen", like::

    Extension("pypkg/_ext", ["pypkg/_ext.c"],
              pynativelibs=[pynativelib_openssl])

Possibly we can/should ship the ``combined_extension_kwargs`` function
above in a helper library too as a stop-gap.


**What if I have two libraries that ship together?** Contrary to the
oversimplified example sketched above, openssl actually ships two
libraries: ``libssl`` and ``libcrypto``. We need a way to link to just
one or the other or both. Solution: well, this is Python, so instead
of restricting ourselves to thinking of the "pynativelib interface" as
something that only top-level modules can provide, we should think of
it as an interface that can be implemented by some arbitrary object,
and a single package can provide multiple objects that implement the
pynativelib interface. So e.g. the openssl package might decide to
expose two objects ``pynativelib_openssl.libssl`` and
``pynativelib_openssl.libcrypto`` and then one could write::

    Extension("pypkg/_ext", ["pypkg/_ext.c"],
              pynativelibs=[pynativelib_openssl.libssl])

or::

    Extension("pypkg/_ext", ["pypkg/_ext.c"],
              pynativelibs=[pynativelib_openssl.libcrypto])

or, if you want to link to both::

    Extension("pypkg/_ext", ["pypkg/_ext.c"],
              pynativelibs=[pynativelib_openssl.libcrypto,
                            pynativelib_openssl.libssl])

(Or if you're using a currently-existing version of setuptools that
doesn't yet support ``pynativelibs=``, then you can translate this
into the equivalent slightly-more-cumbersome code.)

Another case where this idea of pynativelib interface objects is
useful is numpy, which can and should provide an implementation for
projects that want to link to its C API, as ``numpy.pynativelib`` or
something. (This would also nudge downstream packages to start
properly tracking numpy's ABI versions in their ``install_requires``,
which is something that everyone currently gets wrong...)


**What if I want to access a pynativelib library via
``dlopen``/``LoadLibrary``?** See the ``library_name`` interface
below.


**What if my pynativelib library depends on another pynativelib
library?** Just make sure that your ``enable`` function calls their
``enable`` function, and everything else should work.


**What about executables, like the ``openssl`` command-line tool?** We
can handle this by stashing the executable itself in another hidden
directory::

    pynativelib_openssl/
        <...all the stuff above...>
        _bins/
            openssl

and then install a Python dispatch script (using the same mechanisms
as we'd use to ship any Python script with wheel), where the script
looks like::

    #!python

    import os
    import sys

    import pynativelib_openssl

    pynativelib_openssl.enable()
    os.execv(pynativelib_openssl._openssl_bin_path, sys.argv)


Rejected alternatives
---------------------

**Rationale for having ``enable`` but not ``disable``:** since one of
our goals is to avoid accidental collisions with other shared library
distribution strategies, I was originally hoping to use something like
a context manager approach: ``with pynativelib_openssl.enabled():
import ._ext``. The problem is that you can't effectively remove a
library from the search path once it's been loaded on Windows or Linux
(and maybe OS X too for all I know -- `the Linux behavior shocked me
<https://sourceware.org/bugzilla/show_bug.cgi?id=19884>`_ so I no
longer trust docs on this kind of thing). So ``disable`` is impossible
to implement correctly and there's no point trying to pretend
otherwise; the only viable strategy to avoid collisions is to give
libraries unique names.

**Rationale for modifying ``PATH`` on Windows instead of doing
something else:** SxS assemblies are `ruinously complex and yet
fatally limited <http://blog.tombowles.me.uk/2009/10/05/winsxs/>`_ --
in particular they can't be nested and it doesn't seem like they can
load from arbitrary run-time specified directories. It's not clear if
``AddDllDirectory`` is available on all supported Windows
versions. Preloading won't work for running executables (in fact, none
of these options will). And so long as the only things we add to
``PATH`` are directories which contain nothing besides libraries, and
these libraries all have carefully-chosen unique names, then modifying
``PATH`` should be totally safe.


Detailed specification
----------------------

An object implementing the *pynativelib interface* should provide the
following functions as attributes:

Run-time interface:

- ``enable()``: Client libraries must call this before attempting to
  load any code that was linked against this pynativelib. Must be
  idempotent, i.e. multiple calls have the same effect as one call.

- ``library_name()``: Returns a string suitable for passing to
  ``dlopen`` or ``LoadLibrary``. You must call ``enable()`` before
  actually calling ``dlopen``, though. (This is necessary because the
  library might itself depend on other pynativelib libraries, and
  loading those will fail if ``enable`` has not been called. To remind
  you of this, ``library_name`` implementations should probably avoid
  returning a full path even when they could, so as to make code that
  fails to call ``enable`` fail early.)

  (FIXME/QUESTION: are there any cases where it would make sense to
  combine two libraries into a single pynativelib interface object? I
  can't think of any, but if there are then this interface might be
  inadequate. Let's impose that as an invariant for now to keep things
  simple? Alternatively, any package that has a weird special case can
  just implement and document a different interface -- I don't expect
  there will be much generic code calling ``library_name()`` on random
  pynativelib packages; instead it will mostly be specialized code
  calling ``library_name()`` on a specific package for use with
  ``ctypes`` or ``cffi``, so there shouldn't be much problem if a
  particular pynativelib object needs to deviate.)

Build-time interface:

The tricky thing about build time metadata is that we want to support
different compilers simultaneously, and they use different
command-line argument syntaxes.

To address this problem, distutils `takes the approach
<https://docs.python.org/2/distutils/apiref.html#distutils.core.Extension>`_
of defining an abstract Python-level interface to common operations
(and some not-so-common operations), which then get translated into
whatever command-line arguments are appropriate for the compiler at
hand, plus providing the escape-valve ``extra_compile_args`` and
``extra_link_args``.

``pkg-config`` takes an interestingly different approach. It seems
like in practice there are two practically-important styles of passing
arguments to compilers: gcc and MSVC. Everyone else makes their
command-line arguments match one or the other of these -- at least
with regard to the basic arguments that you need to know in order to
link against something::

    | Function     | GCC-style | MSVC-style   |
    |--------------+-----------+--------------|
    | Include path | -I...     | -I...        |
    | Define       | -DFOO=1   | -DFOO=1      |
    | Library path | -L...     | /libpath:... |
    | Link to foo  | -lfoo     | foo.lib      |

(MSVC docs: `compiler
<https://msdn.microsoft.com/en-us/library/19z1t1wy.aspx>`_, `linker
<https://msdn.microsoft.com/en-us/library/y0zzbyt4.aspx>`_; note that
while the canonical option character is ``/``, ``-`` is also
accepted.)

So in a ``pkg-config`` file, you just write down all your compile
arguments and linker arguments (they call them ``cflags`` and
``libs``) using GCC style, and ``pkg-config`` knows about ``-I`` and
``-L`` and ``-l``, and if you really want just those parts of the
command line it can pull them out, and if you want MSVC-style syntax
then you pass ``--msvc`` and it will translate ``-L`` and ``-l`` for
you according to the above table.

(``pkg-config`` also has some special casing for ``-rpath``, but
``-rpath`` is never useful for us, because we never know where the
pynativelib library will be installed on the user filesystem. AFAICT
these are the only cases that ``pkg-config`` thinks need special
casing.)

What I like about the ``pkg-config`` approach is that as a user you
basically just say "give me {GCC, MSVC} {compile, link} flags", and
then the gory details are encapsulated. This is strictly more powerful
than the distutils approach. Consider the case of a library where if
you use it with GCC you need to pass ``--pthread`` as part of your
link flags -- but with MSVC you don't want to pass this. In the
distutils approach there's simply now way to do this.

Therefore, we adopt a more ``pkg-config``-style approach, plus a
convenience adaptor to the legacy distutils-style approach.

- ``compile_args(compiler_style)``, ``link_args(compiler_style)``:
  Mandatory argument ``compiler_style`` should be ``"gcc"`` or
  ``"msvc"`` (or potentially other styles later if it turns out to be
  relevant). Returns a list of strings like ``["-Ifoo", "-DFOO"]``.

  Note: Similarly to ``enable``, if successfully linking to
  pynativelib1 requires some arguments be added to also link to
  pynativelib2, then these functions should call their counterparts in
  pynativelib2 and merge them into the results.

- ``distutils_extension_kwargs()``: returns a dict of kwargs to pass
  to `the distutils ``Extension`` constructor
  <<https://docs.python.org/2/distutils/apiref.html#distutils.core.Extension>`_,
  or raises an error if this is impossible (e.g. if there are
  necessary arguments that cannot be translated into the distutils
  interface).

  This should only be used as a workaround for build systems where we
  can't use the ``{compile,link}_args`` API.

``pkg-config`` also provides some options for pulling out particular
parts of the args (e.g. ``--cflags-only-I`` + ``--cflags-only-other``
= ``--cflags``). I'm not sure why this is needed. We can add it later
if it turns out to be necessary.

FIXME: in some cases we might not be able to convince some code to
link against a specific library just by controlling the arguments
passed to the compiler; for example, there's no argument you can pass
to ``gfortran`` to tell it "please link against
``pynativelib_libgfortran__libgfortran`` instead of
``libgfortran``. Maybe we need some additional function in the
pynativelib interface that the build system is required to call after
it's finished building each binary, so that the pynativelib codes gets
a chance to do post-hoc patching of the binary? However, integrating
this into a distutils-based build system would be non-trivial (and
probably likewise for lots of other build systems)...


Tracking ABI compatibility
--------------------------

Most pynativelib packages will have multiple build variants, so we
need to make sure that the right variant is found at install- and
run-time.

The most fundamental example of this problem is distinguishing between
a pynativelib wheel containing Windows ``.dll`` files, and a
pynativelib wheel containing Linux ``.so`` files. This basic problem
can be solved directly by use of the wheel format's existing ABI
tags. E.g., a typical pynativelib wheel will have a name like::

    $(PKG)-$(VERSION)-py2.py3-none-$(PLATFORM TAG).whl

where ``py2.py3`` indicates that it can be used with any version of
Python, ``none`` indicates that it does not depend on any particular
version of the Python C API, and the platform tag will be something
like ``manylinux1_x86_64`` to indicate that this wheel contains
binaries for Linux x86-64.

The first complication on top of this is that on Windows, there are
multiple versions of the C runtime (CRT) with incompatible ABIs, and
some libraries have a public API that is dependent on the CRT ABI
(e.g. because they have public APIs that take or return stdio
``FILE*`` or file descriptors, or where they return ``malloc``'ed
memory and expect their caller to call ``free``). In practice, each
CPython release picks and sticks with a particular version of MSVC and
its corresponding CRT. So, for pynativelib libraries where the choice
of CRT is important, we can use a cute trick: by tagging the wheel
with the CPython version, we also pick out the right CRT version::

    # MSVC 2008, 32-bit
    $(PKG)-$(VERSION)-cp26.cp27.cp32.pp2-none-win32.whl
    # MSVC 2010, 32-bit
    $(PKG)-$(VERSION)-cp33.cp34-none-win32.whl
    # MSVC 2015 or greater ("universal" CRT), 32-bit
    $(PKG)-$(VERSION)-cp35-none-win32.whl

    # MSVC 2008, 64-bit
    $(PKG)-$(VERSION)-cp26.cp27.cp32.pp2-none-win_amd64.whl
    # MSVC 2010, 64-bit
    $(PKG)-$(VERSION)-cp33.cp34-none-win_amd64.whl
    # MSVC 2015 or greater ("universal" CRT), 64-bit
    $(PKG)-$(VERSION)-cp35-none-win_amd64.whl

Notes:

1) ``pp2`` is "any version of PyPy 2"; for compatibility with CPython
   they also use `MSVC 2008 for Windows builds
   <http://doc.pypy.org/en/latest/windows.html#translating-pypy-with-visual-studio>`_,
   and this seems unlikely to change. PyPy 3 is not widely used and
   may well change compiler versions at some point, so we leave PyPy 3
   out for now.

2) This works well for existing versions of CPython, but (as alluded
   to above) doesn't handle non-CPython builds as well, and it doesn't
   handle future CPython releases well either (when 3.6 is released
   we'll need to update our wheel tags). So in the long run we may
   want to come up with a better way of encoding this information
   (e.g., by adding a new ABI tag that directly indicates which CRT is
   in use, so that one could write ``py2.py3-msvc2008-win32`` or
   similar).

Then, there are all the other ways that ABIs can vary. This may be
especially an issue for libraries that expose C++ APIs. E.g., on
Windows, MSVC and gcc have incompatible C++ APIs, so a C++ pynativelib
package on Windows might want to provide two verions. (Or possibly
even more, if one wants to support the different mingw-w64 ABI
variants.) Or, on all platforms using gcc, APIs that use
``std::string`` or related objects will end up with different ABIs
depending on whether they were built with g++ <5.1 or >=5.1 (`details
<https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html>`_).

In these cases, it's plausible that a single Python environment might
want multiple variants of these pynativelib packages installed
simultaneously, and our job is to make sure that downstream libraries
get run using the same variant of the pynativelib library as they were
built against. Therefore, the solution in these cases will be to build
multiple packages with different names,
e.g. ``pynativelib_mycpplib_msvc`` versus
``pynativelib_mycpplib_mingww64``, so that they can be installed and
imported separately.


Practical challenges / notes towards a todo list
------------------------------------------------

- We're going to want a support library (maybe just ``import
  pynativelib``) that handles common boilerplate. Probably the thing
  to do is to start implementing some pynativelib packages then see
  what helper functions are useful, but I can guess already that we're
  going to want:

  - A generic function for adding a directory to the library search
    that does the right thing on different platforms.

  - Some code to handle the annoyances around compiler/library
    paths. Probably we should follow ``pkg-config``'s lead, and have
    pynativelib package authors provide the GCC-style options, and
    then the support library can automatically translate these into
    the various other representations that we care about.

- Most library build systems won't have a flag that tells them to
  generate a library called ``libpynativelib_openssl__ssl`` instead of
  ``libssl`` or whatever. So we need a way to rename
  libraries. Strategy:

  - On Linux, .so files do know what their name is and this is
    crucial, because if two libraries both think they have the same
    name then you hit `glibc bug #19884
    <https://sourceware.org/bugzilla/show_bug.cgi?id=19884>`_. However,
    ``patchelf --set-soname`` lets you rename a shared library and
    avoid this problem.

  - On OS X, shared libraries do know what their name is, but I'm not
    sure to what extent it affects things; however, my impression is
    that their linker is weirdly concerned about such things
    (FIXME). However, ``install_name_tool -id`` lets you change the
    name of a shared library and avoid any potential problems.

  - On Windows, .dll files do know what their name is, but I don't
    *think* that anything cares? FIXME: we should check this. (If they
    do care, then we can easily teach `redll
    <https://github.com/njsmith/redll>`_ to modify the embedded name
    strings, similar to ``patchelf --set-soname``.)

    A bigger problem though is that on Windows, you don't link
    directly against .dll files; instead, you pass the linker a .lib
    "import library" and the .lib file says which .dll to link to. So
    if you rename a .dll, you also need some way to modify all the
    references to it inside the .lib file.

    *Possible strategy 1:* rename the .dll files and then regenerate
     the .lib files using something like `dlltool
     <https://sourceware.org/binutils/docs/binutils/dlltool.html>`_.

    *Possible strategy 2:* rename the .dll files and then patch the
     .lib file to refer to the new name.

    *Possible strategy 3:* leave the .lib file alone, and patch the
     resulting binary using something like `redll
     <https://github.com/njsmith/redll>`_.

    Complicating strategies 1 & 2 is that there are actually two
    different formats for .lib files -- the simple format that MSVC
    traditionally uses, and the .a format traditionally used by
    mingw-based compilers. The .lib format is very simple and it would
    be easy to write a tool to patch it; also, it is the more
    widespread format and mingw-based compilers can nowadays
    understand it perfectly well, so that's probably what we want to
    be standardizing on. Also, dlltool only knows how to generate the
    .a format. Given all this, and the complexity/uncertainty
    associated of post-hoc patching (strategy 3), I'm leaning towards
    strategy 2: require everyone to use MSVC-style .lib files and
    write a tool to patch them.

    This also requires some way to actually generate .lib files when
    using the mingwpy buildchain. This could use the new ``genlib``
    program that's just been committed to ``mingw-w64`` upstream, or
    it actually wouldn't be terribly hard to implement our own .lib
    file writer (the format is very simple -- just an ``ar`` archive
    of ``ILF`` files, `details are here <
    http://www.microsoft.com/whdc/system/platform/firmware/PECOFF.mspx>`_.). This
    extra work is unfortunate, but we may need to do it anyway if we
    want to allow for people using MSVC to link to mingwpy-compiled
    libraries, and mingw-w64 upstream is already moving in this
    direction.

- We need some scripted way to build these wheels, which will involve
  running the underlying build system (e.g. openssl's Makefile-based
  system), patching the resulting binaries, and packing them up into a
  wheel. It's not 100% clear what the best way to do that is -- maybe
  some hack on top of (dist/setup)utils, maybe some ad hoc thing where
  we just generate the wheels directly (maybe using `flit
  <https://flit.readthedocs.org/en/latest/>`_ for the packing).

- Practically speaking, we're probably going to want some shared
  infrastructure to make it easier to maintain the collection of
  pynativelib packages. `conda-forge
  <https://conda-forge.github.io/>`_ provides an interesting model
  here: they have a `shared github organization
  <https://github.com/conda-forge>`_, with one repo per project. And
  there's some `shared tooling
  <https://github.com/conda-forge/conda-smithy>`_ for automatically
  building this using Appveyor + CircleCI + Travis to cover Windows /
  Linux / OS X respectively, and automatically push new versions into
  the conda channel. We'll need to figure out how to do this.


Unanswered questions
--------------------

- **The "gfortran problem" (also known as "the libstdc++ problem"):**

  Compiler runtimes are particularly challenging to ship as
  pynativelib wheels, because:

  - There's no straightforward way to tell the linker to link against
    the renamed versions of these libraries.

  - It may not be easy to even figure out what version of the runtime
    is needed. For GCC's runtimes you generally need to make sure you
    have a runtime available that's at least as new as the one used to
    build your code -- so you need some way to figure out what was
    used to build the code, which may require some sort of non-trivial
    build system integration.

  - It's not even 100% clear that anyone promises to maintain
    backwards compatibility for GCC runtimes on Windows or OS X. (On
    Linux they use ELF symbol versioning; on Windows and OS X
    there's... no such thing as ELF symbol versioning, and no formal
    promises I can find in the documentation.) But after some
    investigation, it looks like we're probably OK here in practice --
    libgcc and libgfortran have never actually depended on symbol
    versioning, and the only time libstdc++ did was back in the
    ancient past. In particular, for the big c++11 ABI-breaking
    transition, they used a strategy (different mangling for the old
    and new symbols) that basically does the same thing as ELF symbol
    versioning but that is more portable.

  All in all it seems like we'll need to handle these specially
  somehow.

  In the short term, auditwheel-style vendoring works fine. In the
  long run, maybe it will work to follow an auditwheel-style strategy
  where we provide a special tool that postprocesses a built distro to
  convert it into one that depends on a pynativelib package (instead
  of just vendoring the runtime library)? Anyway, we can defer this
  for the moment...

  (Right now this mostly affects GCC-based toolchains, but now that MS
  has switched to a more traditional approach we may run into the
  problem with MSVC in the future too. While MSVC 2015 moved all the
  crucial C runtime stuff out of the compiler runtime and into the
  operating system, `there still is a runtime library
  <http://stevedower.id.au/blog/building-for-python-3-5-part-two/>`_
  (``vcruntime140.dll`` in MSVC 2015). CPython ships
  ``vcruntime140.dll``, so we don't need to. But if MSVC 2016-built
  code needs ``vcruntime150.dll``, and you want to use MSVC 2016 to
  build a package for CPython 3.5, then we'll need a way to ship
  ``vcruntime150.dll``.)

- **How can we make this play better with other distribution
  mechanisms?** E.g. Debian or conda package builders who when
  building ``cryptography`` or ``numpy`` actually *want* them to link
  to an external non-pynativelib version of libssl and libblas.

  This is the one place where I'm most uncertain about the design
  described above.

  The minimal answer is "well, Debian/conda/... can carry a little
  patch to their copy of ``cryptography``/``numpy``, and that's not
  wrong, but it would be really nice if we could do better than
  that. I'm not sure how. The biggest issue is that in the design
  above, downstream packages like ``numpy`` have a hardcoded ``import
  pynativelib_...`` in their ``__init__.py``, which should be removed
  when linked against a non-pynativelib version of the library. What
  would be very nice is if we could make the public pynativelib
  interface be entirely a build-time thing -- so that instead of a
  hard-coded runtime import, the build-time interface would provide
  the code that needed to be called at run-time. (Perhaps it could
  drop it into a special ``.py`` file as part of the build process,
  and then the package ``__init__.py`` would have to import that.)
  This would also potentially be useful in other cases too like
  separation of development and runtime libraries and header-only
  libraries, and maybe if we had some sort of general machinery for
  saying "run this code when you're imported" then it would also help
  with vendoring libraries on Windows. Unfortunately, build-time
  configuration is something that distutils is extremely terrible
  at...


Copyright
---------

This document has been placed into the public domain.
