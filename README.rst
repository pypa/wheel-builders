wheel-builders
==============

Mailing list:

* wheel-builders@python.org
* https://mail.python.org/mailman/listinfo/wheel-builders

`Wheels <http://pythonwheels.com/>`_ let us distribute pre-built
Python libraries -- which is great! Especially for libraries that are
difficult to build, contain binary extensions with complicated
dependencies, and/or want to support users on multiple platforms.

But actually building wheels is still something of a black art. There
are all kinds of little tricks needed, that vary between different
platforms, and traditionally there hasn't been one central place to go
to learn about these. (For example, the MacPython wiki is great, but
unless you're specifically a Mac developer then you may not even know
it exists.)

The wheel-builders mailing list is for everyone who wants to build and
distribute wheels for their packages, and for collaborating to improve
the relevant tooling and documentation. Everything discussed here
would also be in scope for distutils-sig or even pypa-dev, but here we
have a more narrow focus on the practicalities of getting individual
packages working and writing code. Newbie questions welcome!

Some example goals:

* Improving the tooling for building standalone wheels, like
  "delocate" and "auditwheel". (This is the home list for those
  packages.)
* Building common infrastructure for handling native library
  dependencies.
* Writing documentation for package developers on how to use all this
  stuff (perhaps in the form of a new chapter for
  `packaging.python.org <https://packaging.python.org/>`_)
* Developing better tools for automated builds (e.g., sharing scripts
  for getting free CI services to build our wheels for us).

This repository is a scratch space to go along with the mailing list;
in particular we have `a wiki
<https://github.com/pypa/wheel-builders/wiki>`_.


Code of Conduct
===============

Everyone interacting on the wheel-builders mailing list and associated
areas is expected to follow the `PyPA Code of Conduct`_.

.. _PyPA Code of Conduct: https://www.pypa.io/en/latest/code-of-conduct/
