PEP: NNN
Title: Specifying build dependencies for Python Software Packages
Version: $Revision$
Last-Modified: $Date$
Author: Brett Cannon <brett@python.org>,
        Nathaniel Smith <njs@pobox.com>,
        Donald Stufft <donald@stufft.io>
BDFL-Delegate: Nick Coghlan
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
beings to read, but few set it and since there is no verification that
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

Specification
=============

XXX TOML [#toml]_
XXX metadata-version
XXX build dependencies https://www.python.org/dev/peps/pep-0508/
XXX what if outdated dependencies are already installed?
XXX https://mail.python.org/pipermail/distutils-sig/2016-May/028825.html


Rejected Ideas
==============

Other file formats
------------------

Several other file formats were put forward for consideration, all
rejected for varying reasons. Key requirements were that the format
be editable by human beings and have an implementation that can be
vendored easily by projects. This outright exluded certain formats
like XML which are not friendly towards people who need to may need to
edit configuration files by hand.


JSON
''''

The JSON format [#json]_ was initially considered but quickly
rejected. While great as a human-readable, string-based data exchange
format, the syntax does not lend itself to easy editing by a human
being (e.g. the syntax is more verbose than necessary while not
allowing for comments).

An example JSON file for the proposed data would be::

    {
        "metadata version": 1,
        "dependencies": {
            "build": [
                "setuptools",
                "wheel>=0.27"
            ]
        }
    }


YAML
''''

The YAML format [#yaml]_ was designed to be a superset of JSON
[#json]_ while being easier to work with by hand. There are three main
issues with YAML.

One is that the specification is large: 86 pages if printed on
letter-sized paper. That leaves the possibility that someone may use a
feature of YAML that works with one parser but not another.

Two is that YAML itself is not safe by default. The specification
allows for the arbitrary execution of code which is best avoided when
dealing with configuration data. While this PEP is focused on
the building of projects which inherently involves code execution,
other configuration data such as project name and version number may
end up in the same file someday where arbitrary code execution is not
desired.

And finally, the most popular Python implemenation of YAML is
PyYAML [#pyyaml]_ which is a large project of a few thousand lines of
code and an optional C extension module. While in and of itself this
isn't necessary an issue, this becomes more of a problem for projects
like pip where they would most likely need to vendor PyYAML as a
dependency so as to be fully self-contained (otherwise you end up
with your install tool needing an install tool to work).

An example YAML file is::

    metadata-version: 1  # Only 1 is valid ATM.
    dependencies:
        build:
            - setuptools
            - wheel>=0.27


configparser
''''''''''''

An INI-style configuration file based on what
configparser [#configparser]_ accepts was considered. Unfortunately
there is no specification of what configparser accepts, leading to
support skew between versions. For instance, what ConfigParser in
Python 2.7 accepts is not the same as what configparser in Python 3
accepts. While one could standardize on what Python 3 accepts and
simply vendor the backport of the configparser module, that does mean
this PEP would have to codify that the backport of configparser must
be used by all project wishes to consume the metadata specified by
this PEP. This is overly restrictive and could lead to confusion if
someone is not aware of that a specific version of configparser is
expected.

An example INI file is::

    [metadata]
    # Only 1 is valid ATM.
    version = 1

    [dependencies]
    build = setuptools, wheel>=0.27


Python literals
'''''''''''''''

Someone proposed using Python literals as the configuration format.
All Python programmers would be used to the format, there
would implicitly be no third-party dependency to read the
configuration data, and it can be safe if something like
``ast.literal_eval()`` [#ast_literal_eval]_. The problem is that
to user Python literals you either end up with something no
better than JSON, or you end up with something like what
Bazel [#bazel]_ uses. In the former the issues are the same as JSON.
In the latter, you end up with people consistently asking for more
flexibility as users have a hard time ignoring the desire to use some
feature of Python that they think they need (one of the co-authors has
direct experience with this from the internal usage of Bazel at
Google).

There is no example format as one was never put forward for
consideration.


Other file names
----------------

Several other file names were considered and rejected (although this
is very much a bikeshedding topic, and so the decision comes down to
mostly taste).

pypa.toml
  While it makes sense to reference the PyPA [#pypa]_, it is a
  somewhat niche term. It's better to have the file name make sense
  without having domain-specific knowledge.

pybuild.toml
  From the restrictive perspective of this PEP this filename makes
  sense, but if any non-build metadata ever gets added to the file
  then the name ceases to make sense.

pip.toml
  Too tool-specific.

meta.toml
  Too generic; project may want to have its own metadata file.

setup.toml
  While keeping with traditional thanks to ``setup.py``, it does not
  necessarily match what the file may contain in the future (.e.g is
  knowing the name of a project inerhently part of its setup?).

pymeta.toml
  Not obvious to newcomers to programming and/or Python.

pypackage.toml & pypackaging.toml
  Name conflation of what a "package" is (project versus namespace).

pydevelop.toml
  XXX

pysource.toml
  XXX

pytools.toml
  XXX

pysettings.toml
  XXX


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

.. [#toml] TOML
   (https://github.com/toml-lang/toml)

.. [#json] JSON
   (http://json.org/)

.. [#yaml] YAML
   (http://yaml.org/)

.. [#configparser] configparser
   (https://docs.python.org/3/library/configparser.html#module-configparser)

.. [#pyyaml] PyYAML
   (https://pypi.python.org/pypi/PyYAML)

.. [#pypa] PyPA
   (https://www.pypa.io)

.. [#bazel] Bazel
   (http://bazel.io/)

.. [#ast_literal_eval] ``ast.literal_eval()``
   (https://docs.python.org/3/library/ast.html#ast.literal_eval)


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
