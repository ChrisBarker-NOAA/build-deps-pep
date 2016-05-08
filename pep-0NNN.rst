PEP: NNN
Title: Specifying build dependencies for Python Software Packages
Version: $Revision$
Last-Modified: $Date$
Author: Brett Cannon <brett@python.org>,
        Nathaniel Smith <njs@pobox.com>,
        Donald Stufft <donald@stufft.io>
BDFL-Delegate: XXX
Discussions-To:	distutils-sig <distutils-sig at python.org>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: NN-Mmm-2016
Post-History: NN-Mmm-2016


Abstract
========

This PEP specifies how Python software packages should specify their
build dependencies (e.g. what dependencies are required to go from
source checkout to built wheel). As part of this specification, a new
configuration file is introduced for software packages to use to
specify their build dependencies (with the expectation that the same
configuration file will be used for future configuration details).


Rationale
=========

When Python first developed its tooling for building distributions of
software for projects, distutils [#distutils]_ was the chosen
solution. As time went on, setuptools [#setuptools]_ gained popularity
to add some features on top of distutils. Both used the concept of a
``setup.py`` file that project maintainers executed to build
distributions of their software (as well as users to install said
distribution).

Using an executable file to specify build requirements under distutils
isn't an issue as distutils is part of Python's standard library.
Having the build tool as part of Python means that a ``setup.py`` has
no external dependency that a project maintainer needs to worry about
to build a distribution of their project. There was no need to specify
any dependency information as the only dependency is Python.

But when a project chooses to use setuptools, the use of an executable
file like ``setup.py`` becomes an issue. You can't execute a
``setup.py`` file without knowing its dependencies, but currently
there is no standard way to know what those dependencies are without
executing the ``setup.py`` file where that information is stored. It's
a catch-22 of a file not being readable without knowing its own
contents which can't be known unless you read the file.

Setuptools tried to solve this with a ``setup_requires`` argument to
its ``setup()`` function [#setup_args]_. The problem is that no tools
can read that information without executing the ``setup.py`` file, but
the ``setup.py`` file can't be executed necessarily without knowing
what that argument contains. In practice the field is more for human
beings to read, but few set it and since there is no verirication that
its contents are valid it's very easy for the field to be out-of-date.

All of this has led pip [#pip]_ to simply assume that setuptools is
necessary when executing a ``setup.py`` file, thus making sure it's
installed when building a project as e.g. a wheel [#wheel]_. The
problem with this, though, is it doesn't scale if another project
began to gain traction in the commnity as setuptools has. It also
prevents other projects from gaining traction due to the friction
required to use it with a project when pip can't infer the fact that
something other than setuptools is required.

This PEP specifies a way to list the build dependencies of a project
in a declarative fashion in a specific file. This allows a project
to list what build dependencies it has to go from e.g. source
checkout to wheel, while not falling into the catch-22 trap that a
``setup.py`` has where tooling can't infer what a project needs to
build itself. Implementing this PEP will allow projects to specify
what they depend on upfront so that tools like pip can make sure that
they are installed in order to build the project (the actual driving
of the build process is left for another PEP).


References
==========

.. [#distutils] distutils
   (https://docs.python.org/3/library/distutils.html#module-distutils)

.. [#setuptools] setuptools
   (https://pypi.python.org/pypi/setuptools)

.. [#setup_args] setuptools: New and Changed setup() Keywords
   (http://pythonhosted.org/setuptools/setuptools.html#new-and-changed-setup-keywords)

.. [#pip] pip
   (https://pypi.python.org/pypi/pip)

.. [#wheel] wheel
   (https://pypi.python.org/pypi/wheel)


Copyright
=========

This document has been placed in the public domain.



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
