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
Created: 10-May-2016
Post-History: 10-May-2016


Abstract
========

This PEP specifies how Python software packages should specify their
build dependencies (i.e. what dependencies are required to go from
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
there is no standard way to know what those dependencies are in an
automated fashion without executing the ``setup.py`` file where that
information is stored. It's a catch-22 of a file not being runnable
without knowing its own contents which can't be known programmatically
unless you run the file.

Setuptools tried to solve this with a ``setup_requires`` argument to
its ``setup()`` function [#setup_args]_. This solution has a number
of issues, such as:

* No tooling (besides setuptools itself) can access this information
  without executing the ``setup.py``, but ``setup.py`` can't be
  executed without having these items installed.
* While setuptools itself will install anything listed in this, they
  won't be installed until *during* the execution of the ``setup()``
  function, which means that the only way to actually use anything
  added here is through increasingly complex machinations that delay
  the import and usage of these modules until later on in the
  execution of the ``setup()`` function.
* This cannot include ``setuptools`` itself nor can it include a
  replacement to ``setuptools``, which means that projects such as
  ``numpy.distutils`` are largely incapable of utilizing it and
  projects cannot take advantage of newer setuptools features until
  their users naturally upgrade the version of setuptools to a newer
  one.
* The items listed in ``setup_requires`` get implicily installed
  whenever you execute the ``setup.py`` but one of the common ways
  that the ``setup.py`` is executed is via another tool, such as
  ``pip``, who is already managing dependencies. This means that
  a command like ``pip install spam`` might end up having both
  pip and setuptools downloading and installing packages and end
  users needing to configure *both* tools (and for ``setuptools``
  without being in control of the invocation) to change settings
  like which repository it installs from. It also means that users
  need to be aware of the discovery rules for both tools, as one
  may support different package formats or determine the latest
  version differently.

This has cumulated in a situation where use of ``setup_requires``
is rare, where projects tend to either simply copy and paste snippets
between ``setup.py`` files or they eschew it all together in favor
of simply documenting elsewhere what they expect the user to have
manually installed prior to attempting to build or install their
project.

All of this has led pip [#pip]_ to simply assume that setuptools is
necessary when executing a ``setup.py`` file. The problem with this,
though, is it doesn't scale if another project began to gain traction
in the commnity as setuptools has. It also prevents other projects
from gaining traction due to the friction required to use it with a
project when pip can't infer the fact that something other than
setuptools is required.

This PEP attempts to rectify the situation by specifying a way to list
the build dependencies of a project in a declarative fashion in a
specific file. This allows a project to list what build dependencies
it has to go from e.g. source checkout to wheel, while not falling
into the catch-22 trap that a ``setup.py`` has where tooling can't
infer what a project needs to build itself. Implementing this PEP will
allow projects to specify what they depend on upfront so that tools
like pip can make sure that they are installed in order to build the
project (the actual driving of the build process is not within the
scope of this PEP).


Specification
=============

The build dependencies will be stored in a file named
``pyproject.toml`` that is written in the TOML format [#toml]_. This
format was chosen as it is human-usable (unlike JSON [#json]_), it is
flexible enough (unlike configparser [#configparser]_), stems from a
standard (also unlike configparser [#configparser]_), and it is not
overly complex (unlike YAML [#yaml]_). The TOML format is already in
use by the Rust community as part of their
Cargo package manager [#cargo]_ and in private email stated they have
been quite happy with their choice of TOML. A more thorough
discussion as to why various alternatives were not chosen can be read
in the `Other file formats`_ section.

A top-level ``[package]`` table will represent details specific to the
package that the project contains.

A ``semantics-version`` key within the ``[package]`` table will
represent the semantic version that the ``[package]`` table
represents. It will always be set to an integer and will default to a
value of ``1`` if unspecified. The version will only be updated when
the semantics or absence of a key or sub-table in the ``[package]``
table cannot be interpreted in a backwards-compatible fashion (e.g.
the version does not need to change if a new table is added to the
semantics of the file, but if a pre-existing field changes its
meaning or the behavior when a field is absent changes then the
semantic version will need to change). Changes to the meaning of the
versions are expected to occur through a PEP.

Projects are expected to check the value of the ``semantics-version``
field in their code appropriately. The expectation is tools will do
something along the lines of::

  if data["package"].get("semantics-version", 1) != 1:
      raise Exception  # Whatever exception is appropriate for the tool.

There will be a ``[package.build-system]`` sub-table in the
configuration file to store build-related data (although the exact
name of the sub-table is an
`open issue <#name-of-the-build-related-sub-table>`__ as
``[package.build]`` is another possibility). Initially only one key of
the table will be valid: ``requires``. That key will have a value of a
list of strings representing the PEP 508 dependencies required to
build the project as a wheel [#wheel]_ (currently that means what
dependencies are required to execute a ``setup.py`` file to generate a
wheel).

For the vast majority of Python projects that rely upon setuptools,
the ``pyproject.toml`` file will be::

  [package.build-system]
  requires = ['setuptools', 'wheel']  # PEP 508 specifications.

Or, the equivalent but more verbose::

  [package]
  semantics-version = 1  # Optional; defaults to 1.

      # Indentation is optional in TOML and has no semantic meaning.
      [package.build-system]
      requires = ['setuptools', 'wheel']  # PEP 508 specifications.

Because the use of setuptools and wheel are so expansive in the
community at the moment, build tools are expected to use the example
configuration file above as their default semantics when a
``pyproject.toml`` file is not present.

All other top-level keys and tables are reserved for future use by
other PEPs except for the ``[tool]`` table. Within that table, tools
can have users specify configuration data as long as they use a
sub-table within ``[tool]``, e.g. the `flit <https://pypi.python.org/pypi/flit>`_
tool might store its configuration in ``[tool.flit]``.

We need some mechanism to allocate names within the ``tool.*``
namespace, to make sure that different projects don't attempt to use
the same sub-table and collide. Our rule is that a project can use
the subtable ``tool.$NAME`` if, and only if, they own the entry for
``$NAME`` in the Cheeseshop/PyPI.


Open Issues
===========

Name of the build-related sub-table
-----------------------------------

The authors of this PEP couldn't decide between the names
``[package.build]`` and ``[package.build-system]``, and so it is an
open issue over which one to go with.


Rejected Ideas
==============

Other semantic version key names
--------------------------------

Names other than ``semantics-version`` were considered to represent
the version of semantics that the configuration file was written for.
Both ``configuration-version`` and ``metadata-version`` were both
considered, but were rejected due to how people may confuse the
key as representing a version of the files contents instead of the
version of semantics that the file is interpreted under.


A flatter namespace
-------------------

An earlier draft of this PEP lacked the ``[package]`` table and had
all of its contained values one level higher. In the end it was
decided it would be better to scope package-related details to its own
table for more clear scoping and easier expansion of this file for
future use.


Other file formats
------------------

Several other file formats were put forward for consideration, all
rejected for various reasons. Key requirements were that the format
be editable by human beings and have an implementation that can be
vendored easily by projects. This outright exluded certain formats
like XML which are not friendly towards human beings and were never
seriously discussed.


JSON
''''

The JSON format [#json]_ was initially considered but quickly
rejected. While great as a human-readable, string-based data exchange
format, the syntax does not lend itself to easy editing by a human
being (e.g. the syntax is more verbose than necessary while not
allowing for comments).

An example JSON file for the proposed data would be::

    {
        "build": {
            "requires": [
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
feature of YAML that works with one parser but not another. It has
been suggested to standardize on a subset, but that basically means
creating a new standard specific to this file which is not tractable
long-term.

Two is that YAML itself is not safe by default. The specification
allows for the arbitrary execution of code which is best avoided when
dealing with configuration data.  It is of course possible to avoid
this behavior -- for example, PyYAML provides a ``safe_load`` operation
-- but if any tool carelessly uses ``load`` instead then they open
themselves up to arbitrary code execution. While this PEP is focused on
the building of projects which inherently involves code execution,
other configuration data such as project name and version number may
end up in the same file someday where arbitrary code execution is not
desired.

And finally, the most popular Python implemenation of YAML is
PyYAML [#pyyaml]_ which is a large project of a few thousand lines of
code and an optional C extension module. While in and of itself this
isn't necessarily an issue, this becomes more of a problem for
projects like pip where they would most likely need to vendor PyYAML
as a dependency so as to be fully self-contained (otherwise you end
up with your install tool needing an install tool to work). A
proof-of-concept re-working of PyYAML has been done to see how easy
it would be to potentially vendor a simpler version of the library
which shows it is a possibility.

An example YAML file is::

    build:
        requires:
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

    [build]
    requires =
        setuptools
        wheel>=0.27


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

pysettings.toml
  Most reasonable alternative.

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
  The file may contain details not specific to development.

pysource.toml
  Not directly related to source code.

pytools.toml
  Misleading as the file is (currently) aimed at project management.

dstufft.toml
  Too person-specific. ;)


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

.. [#cargo] Cargo, Rust's package manager
   (http://doc.crates.io/)


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
