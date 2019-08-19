:tocdepth: 1

.. sectnum::

Abstract
========

Examples and tutorials are important elements of successful documentation.
They show the reader how something can be accomplished, which is often more powerful and effective than a mere description.
Such examples are only useful if they are correct, though.
Automated testing infrastructure is the best way to ensure that examples are correct when they are committed, and remain correct as a project evolves.
Recently, in :dmtn:`085` :cite:`DMTN-085`, the QA Strategy Working Group (QAWG)  issued specific recommendations to improve how examples are managed and tested in LSST's documentation.
This technote analyzes these recommendations and translates them into technical requirements.
Subsequently, this technote also provides an overview of how example code management and testing has been implemented.

.. _recs:

QAWG Recommendations relevant to examples in documentation
==========================================================

The work planned and described in this technote was prompted by recommendations made by the QAWG in :dmtn:`085`.
In short, the QAWG recommended that LSST DM adopted a systematic approach to producing, maintaining, and testing examples in our end-user documentation projects.
For reference, the relevant recommendations are:

.. _qawg-rec-13:

QAWG-REC-13
    | Provide a central location where examples, scripts and utilities which are not fundamental to pipeline execution are indexed and made discoverable.

.. _qawg-rec-14:

QAWG-REC-14
    | The Project should adopt a documented (in the `Developer Guide`_) policy on the maintenance of example code.

.. _qawg-rec-15:

QAWG-REC-15
    | The Project should prioritize the development of a documentation system which makes it convenient to include code examples and that tests those examples as part of a documentation build.

Of these recommendations, the bulk of the work is associated with |15|.
It is through |15| that we are mandated to specify the conventions and formats for committing examples and scripts into the LSST codebase, and provide the infrastructure for displaying these examples and scripts in end-user documentation.
It is also through |15| that we are mandated to provide infrastructure to test examples and scripts for correctness and reproducibility.
We interpret |13| as requesting that examples and scripts be conveniently displayed and listed through end-user documentation, to solve the issue that examples and scripts are generally not discoverable of both our own developers and the user community.
|13| can be accomplished in conjunction with |15|.

Finally, |14| requests that we establish a policy for how example code is maintained.
The issue is that examples can be broken when developers add or change APIs in the codebase and it is not necessarily the responsibility of those authors to also update all examples and tutorials elsewhere in the LSST documentation and codebase.
The work done to fulfill |13| and |15|, by creating a systematic infrastructure for handling examples and scripts, can assist with this process by alerting developers when their code changes are affecting documentation.
Beyond that, in |14| we must still provide a system for organizing the work associated with breakages in example code.

.. _review-of-scripts:

Review of LSST's existing ad hoc scripts
========================================

Although the main focus of this work is associated with examples and tutorials, |13| also specifically requests that LSST's ad hoc scripts be systematically included in the documentation.
This section summarizes the state of ad hoc scripts in the LSST codebase.

*What are these ad hoc scripts?*

First, ad hoc scripts *are not* command-line tasks (soon to be overhauled with the Generation 3 middleware project), which are included in the :file:`bin.src` directories of packages and that use the `lsst.pipe.base.CmdLineTask` infrastructure.
These scripts are already documented using the `Task topic type`_ in LSST documentation.
Such task scripts are consistently documented, and centrally indexed. [#taskindex]_

.. [#taskindex] Tasks are listed in package documentation homepages and there are plans, described in :dmtn:`30` to list task by theme so that users can discover appropriate tasks without having to be familiar with the structure of the LSST codebase.

.. _scripts-in-binsrc:

Python scripts in bin.src
-------------------------

Other scripts, outside the task framework, are also included in the :file:`bin.src` directories of packages.
We put Python scripts in :file:`bin.src` because they are processed by SCons at build time and installed into the user's ``$PATH`` so that they can be executed directly on the command line.
For example, the verify_ package has three scripts in its :file:`bin.src` directory: dispatch_verify.py_, inspect_job.py_, and lint_metrics.py_.
Some of these scripts use the standard `argparse` package to process command-line arguments (dispatch_verify.py_ and lint_metrics.py_), while others directly parse command line arguments in an ad hoc manner (`inspect_job.py`_).

.. note that lint metrics does not its argparse parser in an importable place.

.. _general-purpose-scripts-in-examples:

General-purpose scripts in examples/
------------------------------------

Some useful utility scripts are distributed in the :file:`eamples` directory of packages, rather than :file:`bin.src`.
For example, the plotSkyMap.py_ script, found in the :file:`examples` directory of the skymap_ package uses `argparse` and is a general purpose utility that is potentially testable and usable.
The problem with scripts like plotSkyMap.py_ is that they are not installed for users.
Instead, users need to reference them by their absolute path, often using environment variables created by EUPS:

.. code-block:: sh

   python $SKYMAP_DIR/examples/plotSkyMap.py

It makes sense to move all command-line scripts into the :file:`bin.src` directory so that they can be addressed by users without having to know about their association with EUPS packages:

.. code-block:: sh

   plotSkyMap.py

.. _non-reusable-utilities-in-examples:

Non-reusable utility scripts in examples/
-----------------------------------------

Another category of scripts in :file:`examples` non-reusable utility scripts.
For example, the pipe_analysis_ package includes a script called parseLogs.py_ in its :file:`examples` directory.
parseLogs.py_ isn't intended to be used directly since it is hard-coded in a way that is very specific at not reproducible:

.. code-block:: py

   logRootDir = "/tigress/HSC/users/lauren/"
   lsstTicket = "DM-6816"
   hscTicket = "HSC-1382"
   rcFields = ["cosmos", "wide"]
   bands = ["g", "r", "i", "z", "y", "n921"]
   allBands = "HSC-G^HSC-R^HSC-I^HSC-Z^HSC-Y^NB0921"
   bandStrDict = {"g": "HSC-G", "r": "HSC-R", "i": "HSC-I", "z": "HSC-Z", "y": "HSC-Y", "n921": "NB0921", "HSC-G^HSC-R^HSC-I^HSC-Z^HSC-Y^NB0921": "GRIZY9", "HSC-G^HSC-R^HSC-I^HSC-Z^HSC-Y": "GRIZY"}
   colorsList = ["gri", "riz", "izy", "z9y"]

That said, parseLogs.py_ is clearly meant to be an executable utility script rather than an :ref:`example for documentation <examples-as-scripts>` because it solves a specific problem and doesn't seem to have a broader didactic purpose.

Example scripts such as this one pose a problem for fulfilling |15| because that code can only be executed in a single, non-reproducible environment.
Such code needs to be re-engineered, including providing a proper command-line interface, if we can hope to use and test it.

.. _examples-as-scripts:

Examples as scripts in examples/
--------------------------------

We also see many executable scripts in :file:`examples` directories that are associated with documentation.
An example is the runRepair.py_ script in the pipe_tasks_ package.
That script is associated with a page in the Doxygen documentation.
The reason it's a script is to be runnable: the script sets up a mock dataset and then runs the ``RepairTask`` task on it.
This type of script fits the purpose of the original :file:`examples` framework, but there is a clear mandate from |15| to improve how these examples are created, managed, and tested.
Didactic scripts will be considered :ref:`later in this technote <examples-review>` as part of the examples portion of the work.

Scripts in languages other than Python
--------------------------------------

Not all ad hoc scripts are written in Python.
For example, run_ci_dataset.sh_ in the :file:`bin` directory of ap_verify_ is a shell script that provides a higher-level interface to the ``ap_verify.py`` script.

Plan for consolidating and documenting scripts
==============================================

Based on the :ref:`review of existing ad hoc scripts <review-of-scripts>` in the LSST codebase, we can fulfill |13| (in relation to scripts) by introducing a systematic approach to including and documenting scripts in the LSST codebase.

Action: move all utility scripts to bin.src/ or bin/
----------------------------------------------------

The first improvement we can realize is by ensuring that any executable script provided with an LSST package is shipped in its :file:`bin.src` or :file:`bin` directory. [#setuptoolsscripts]_

.. [#setuptoolsscripts]

   This plan of action applies to EUPS packages.
   LSST code that is packaged for PyPI with setuptools should instead use the ``console_scripts`` entrypoints feature to install scripts from a package's modules:

   .. code-block:: py

      setup(
          # ...
          entry_points={
              'console_scripts': [
                  'cliname = package.module:main_function',
              ]
          }
      )

   This has the same effect as putting scripts in the :file:`bin.src` directory of an LSST EUPS package.

This will have the effect of making it possible for users to execute scripts without having to address EUPS environment variables.
Using the :file:`bin.src` directory specifically for Python-based scripts has the benefit of ensuring that the hash-bang is re-written properly to account for SIP security features in macOS.
Non-Python scripts can go directly in the :file:`bin` directory because shebangtron_ only updates the hash-bangs of Python scripts.

Action: mandate that the core code from scripts should reside in the main package for testability
-------------------------------------------------------------------------------------------------

Rather than putting the entirety of a script's code in the script module itself, which resides in :file:`bin.src`, we should encourage developers to put the entirely of a script's code in the main package.
A useful pattern might be to create a standardized subpackage called ``scripts`` [#scripts-name]_ within any Stack package.
Then inside the ``scripts`` subpackage, each "script" is a module that's named after the command line executable in the :file:`bin.src` directory.
Then the script imports that main entrypoint:

.. code-block:: py

   #!/usr/bin/env python
   from lsst.some.package.scripts.a import main

   if __name__ == '__main__':
       main()

The ``main`` function is then responsible for parsing command-line arguments and running the business logic, though ideally ``main`` itself is factored such that the core logic is performed in functions that are independent of the command-line context.
With this architecture, the script's internal logic can be tested using the existing unit testing infrastructure (`unittest` tests run by pytest_ though SCons).
This architecture is already effectively used by command-line tasks.
Their executable command-line scripts look like this:

.. code-block:: py

    #!/usr/bin/env python
   from lsst.pipe.tasks.processCcd import ProcessCcdTask

   ProcessCcdTask.parseAndRun()

The interaction of command-line arguments with script flow can even be tested within `unittest`-based tests by mocking the output of `argparse.ArgumentParser.parse_args`.
Interactions with other types of external resources can also be mocked.

In summary, by restructuring scripts we can provide comprehensive unit tests for those scripts without having to treat scripts as a special case for testing.

.. [#scripts-name]

   An alternative choice to ``scripts`` would be ``bin``.
   The ``lsst.verify`` package has a subpackage called ``bin``.

Action: scripts are documented in package documentation
-------------------------------------------------------

Similar to how every function has a page in the Python API reference, and every task has a corresponding `Task topic page <Task topic type>`_, every script or command-line executable must have a corresponding documentation page.
The structure of this page should be designed and templated equivalently to a `topic type`_.
These documentation pages should be listed both on the package's homepage, and in a central index accessible from the https://pipelines.lsst.io homepage (to be specific) that gathers executables from all packages.
The script topic will use Sphinx extensions to auto-populate documentation from the script's code (see the :ref:`next action <adopt-argparse>`).

The script topic should also provide a way to add metadata about a script, such as a description or tags, to facilitate a useful index of scripts.

.. _adopt-argparse:

Action: adopt argparse for command-line scripts to enable auto-documentation
----------------------------------------------------------------------------

To facilitate automatic documentation of command-line interfaces, scripts should use standard frameworks for establishing their interface rather than directly parsing `sys.argv`.
For example, the `sphinx-argparse`_ Sphinx extension can automatically document a command-line interface based on the `argparse.ArgumentParser` configuration for a script.
Since `argparse is already adopted <https://developer.lsst.io/python/style.html#the-argparse-module-should-be-used-for-command-line-scripts>`_ by the DM Style Guide, this recommendation should be non-controversial.
Nevertheless, some simple scripts have been written to avoid `argparse`, and those scripts should be ported to `argparse` to facilitate documentation.

To work with `sphinx-argparse`_, scripts need to be written such that the `argparse.ArgumentParser` is returned by a function that takes no arguments:

.. code-block:: py

   def main():
       parser = parse_args()
       # ...


   def parse_args():
       parser = argparse.ArgumentParser(description='Documentation for the script')
       parser.add_argument('--verbose', default=False, help='Enable verbose output)
       return parser

Such a requirement will need to be added to the `DM Python Style Guide`_.

.. _examples-review:

Survey of examples and tutorials in LSST documentation
======================================================

In the second part of this technote, we consider example and tutorial code that appears in documentation, and attempt to provide a road map for fulfilling |13| and |15|.
As with the first part, concerning utility scripts, we first review the current landscape of example code, and in later sections identify technologies and actionable steps towards meeting |13| and |14|.

.. _review-examples-in-examples:

Examples in examples/ directories
---------------------------------

The |examples| directory does, in fact, contain example code (though see also :ref:`general-purpose-scripts-in-examples`).
Examples exist in many forms: C++ source and header files (``.cc`` and ``.h``), Python modules, and Jupyter notebooks.

In the most successful cases, the Python and C++ files are referenced from documentation text.
Originally, LSST Science Pipelines documentation was written in Doxygen and the ``.dox`` files and docstrings included the contents of files from |examples|.
For example, the `measAlgTasks.py`_ example is referenced from the docstring of the SourceDetectionTask_.
Newer documentation being written in reStructuredText is merely linking to the GitHub blob URLs of files in |examples| because the multi-package build prevents |examples| from being available at a fixed relative URL during the build process.\ [#examples-sphinx-build]_

.. [#examples-sphinx-build]

   See the `Overview of the pipelines.lsst.io build system <https://documenteer.lsst.io/pipelines/build-overview.html>`_ in Documenteer's documentation.

Many of the Python examples, and generally all of the C++ examples, are structured as executables.
In the case of the Python examples, the command-line interface provides a toggle for activating the debug framework or to optionally open a display (see `measAlgTasks.py`_).
Thus these examples are distinct from :ref:`ad hoc scripts that are also found in the examples directory <general-purpose-scripts-in-examples>`.

Not all examples are referenced from the documentation, though.
For example, the `statisticsMaskedImage.py`_ module in the |examples| directory of ``afw`` is not referenced in any documentation, despite seeming to be genuine example.

.. _review-tests-in-examples:

Test programs in examples/ directories
--------------------------------------

In addition to examples that are associated with documentation, some files in |examples| directories (typically C++) are neither :ref:`true examples <review-examples-in-examples>` nor :ref:`ad hoc utilities <review-tests-in-examples>`.
These files seem to be tests of an ad hoc nature.
Examples of this are the `ticket647.cc`_ and `maskIo2.cc`_ programs in ``afw``.
The former appears to reference a ticket from the deprecated Trac system, and `maskIo2.cc`_ seems to test memory management in C++ code.

Part of the issue here is that DM doesn't write unit tests in C++.
Instead, all unit tests are written in Python, though those tests may exercise C++ code through Pybind11 bindings.

.. _review-data-in-examples:

Data files in examples/ directories
-----------------------------------

In rare cases, data files are located in the |examples| directories of packages.
One such file is NewSuprimeCam.paf_ in ``afw``, which has no references anywhere in the ``afw`` codebase.

.. _review-doctests:

Python doctests
---------------

Another category of example code that is commonly found in Python are doctests_, which are supported by Python itself through the `doctest` standard library package.
doctests_ are formatted like interactive Python sessions, and show both the input that a user might enter at a Python prompt, along with the expected output.
For example:

>>> a = [1, 2, 3]
>>> a.append(4)
>>> a
[1, 2, 3, 4]

doctests_ can be found in docstrings (particularly the `Examples section <https://developer.lsst.io/python/numpydoc.html#py-docstring-examples>`__ of a Numpydoc-formatted Python docstring), as well as in reStructuredText files.
Because docstrings show both inputs and inputs, they work well as testable examples because test harnesses can run the example and verify that the output matches the expected output.

.. _review-rst-examples:

Untested Python and shell samples in reStructuredText
-----------------------------------------------------

Besides doctests_, code samples can also be added to documentation with reStructuredText directives such as ``code-block``, ``literalinclude``, and ``prompt``.
For example, the `Getting Started`_ tutorial series in the Pipelines documentation uses ``code-block`` directives to include both shell commands and their output, along with Python scripts.

.. _review-jupyter-notebooks:

Jupyter notebooks
-----------------

`Jupyter Notebooks`_, like doctests_, are well suited for creating testable documentation because of how they mix prose, code, and output cells.
`Jupyter Notebooks`_ are particularly notable for the immediacy and interactivity of their writing process.

LSST uses `Jupyter Notebooks`_ in a number of contexts, including as documentation.
As :ref:`mentioned before <review-examples-in-examples>`, Jupyter Notebooks appear in the |examples| directories of packages.
Entire Git repositories are also dedicated to collecting Jupyter Notebooks.
For example, the `notebook-demo`_ repository contains demo notebooks for LSST's Nublado platform.
At the moment, Notebooks aren't part of Sphinx documentation builds.

.. _examples-consolidation:

Consolidation of approaches to examples
=======================================

In the :ref:`previous section <examples-review>`, we reviewed the various types of examples that exist in LSST codebases.
Given the number of formats that examples can currently be found in, it's beneficial to consolidate our usage to a defined set of technologies and methodologies that are both convenient to integrate into documentation (addressing |13|) and test (addressing |15|).
The QAWG recommended that one technology should be adopted:

    There are various technologies which could be adopted to address this goal.\ [#wg-techs]_
    The WG suggests that standardizing upon a single technology is essential, but takes no position as to which technology is most appropriate.

    .. [#wg-techs] For example, Jupyter notebooks or Sphinx doctests.

Although adopting a single technology is appealing, such a limitation may prove inappropriate for the types of documentation that LSST writes, and the types of things that are documented.
The approach that we will pursue in this technote is to address the specific scenarios where examples are written for LSST documentation, and to associate a specific approach with that scenario.
This way, even though we support multiple technologies, only one is permitted for each documentation scenario.
We believe that most scenarios of writing examples in documentation can be covered with two technologies: Python doctests and Jupyter Notebooks.
C++ examples remain a special case, and will be supported by a third approach.
The following sections review these adopted technologies and the scenarios that they support.

Doctests
--------

Doctests are standard in Python, have have robust support in both Sphinx_ (the tool that generates our documentation websites) and in pytest_ (the tool that runs our Python unit tests).
The Astropy_ project is an excellent example of doctest-driven documentation.
By using doctests, the Astropy project provides ample examples of their APIs, and these examples are tested automatically as part of their continuous integration process.

Doctests excel in their integration with existing software and documentation development practices.
Doctests are convenient to add to docstrings of Python functions, classes, and methods.
Doctests are also convenient to add to reStructuredText files, which is where the bulk of LSST's user and conceptual documentation is already written.
For example, each task already has a `task topic`_ page written in reStructuredText.
Doctests demonstrating that task as a Python API can be added directory into that reStructuredText file.
Being plain text, doctests are convenient to review as part of a pull request.

Compared to Jupyter Notebooks, doctests are slightly less convenient to write.
Instead of the writing and execution environment being combined, developers may choose to write Python statements in a scratch Jupyter Notebook or IPython shell and copy the source and output into a doctest.
Testing the doctest also requires running a command.
However, given the success and abundant use of doctests in projects ranging from the Python documentation to Astropy_, it would appear that workflow issues are not significant.

We recommend that doctests be adopted for docstrings and for how-to documentation written in reStructuredText where it is important for the example to integrate seamlessly with the existing documentation.

Jupyter Noteboooks
------------------

Jupyter Notebooks are the second technology that would be good to consolidate towards.
In many ways, Jupyter Notebooks have similar attributes to doctests in that they combine prose, source code, and outputs.
Compared to doctests, notebooks add a few additional capabilities: support for running shell commands, and integration as a development and execution environment for both writers and readers.
Given the adoption of Jupyter notebooks by the LSST Science Platform, it's also obvious that notebooks cannot be ignored as a platform for creating examples.

Notebooks do have some disadvantages compared to doctests that prevent us from adopting them as the dominant technology for all examples.
First, their JSON format is difficult to integrate into Pull Request workflows where merge conflicts can be expected.
Similarly, notebooks require a working Jupyter server to view and edit, as opposed to the minimalist requirements of doctests.
As the LSST Science Platform becomes more mature, it will become easier to include Jupyter Notebooks in a development workflow.

Second, notebooks are also difficult to integrate into Sphinx documentation.
Markdown is the primary prose format for Jupyter Notebooks.
Although it's possible to write in reStructuredText, it won't be rendered in the browser.
This means that notebooks cannot take advantage of Sphinx's cross-referencing syntax.
Nor can reStructuredText files use cross-reference syntax to link *to* a Jupyter Notebook.
For this reason we suggest that it's better to not deeply integrate Jupyter Notebooks within a Sphinx documentation page.
Instead, Jupyter Notebooks ought to be standalone pages.

In other words, Jupyter Notebooks work well for delivering *tutorials*.
In `What nobody tells you about documentation`_, the author Daniele Procida describes four distinct types of documentation:

Reference
    A comprehensive description of the product.

Explanations
    Content that helps build understanding, background, and context.

How-to guides
    Specific recipes, often with steps, that describe how to accomplish a common task.

Tutorials
    A learning-oriented lesson.

Doctests work well when integrated with reference documentation (examples in docstrings, for example), in how-to guides, and to a lesser extent in explanatory guides as well.
That type of documentation is deeply integrated with reStructuredText and Sphinx.

Jupyter notebooks on the other hand are excellent for tutorials because, as a lesson, they can stand apart from the main body of the documentation.
Sphinx's features, such as custom syntax, are not generally needed for tutorials.
The reader's ability to download the notebook itself and follow along and make experimental adjustments to the tutorial is hugely beneficial.
Lastly, tutorials experience less churn during regular development than other types of documentation, which makes the notebook's requirement that it cannot be edited in a text editor more acceptable.
Thus, we recommend Jupyter notebooks as an ideal medium for producing tutorials.

Jupyter notebooks are also useful for other types of documentation that benefits from an integration with software.
For example, technical notes could be written as Jupyter notebooks.
The nbreport_ platform is also built around the concept of using Jupyter notebooks to generate regular reports augmented with templated computations.

C++ examples
------------

Together, doctests and Jupyter Notebooks cover scenarios for most of the examples that LSST might want to write.
One scenario that isn't addressed, though, are C++ examples that are currently found in the |examples| directories of packages.
Neither doctests nor Jupyter Notebooks support C++ code.

For C++ examples, it may best to continue the existing pattern of placing source files in the |examples| directory, having scons run the compilation of those examples, and referencing those examples from the documentation.
Note that displaying files from the |examples| directory still needs to be accommodated in the Sphinx builds as mentioned in :ref:`review-examples-in-examples`.

.. _summary-of-example-scenarios:

Summary of documentation scenarios and technologies
---------------------------------------------------

In summary, these are technologies that DM should adopt to produce examples in, and the appropriate scenarios for each technology:

Python doctests
    - "Examples" sections of Python docstrings.
    - Python API demos and how-tos integrated with reStructuredText/Sphinx documentation.
    - Pure-Python tutorials written in reStructuredText/Sphinx.

Jupyter Notebooks
    - Standalone tutorials that use Python and/or the shell, which are associated with a Sphinx documentation site.
    - nbreport_ :cite:`SQR-023` templates and instances.
    - Technical notes written entirely as a Jupyter Notebook.

Files in |examples|
    - Examples written in C++ that are referenced from reStructuredText/Sphinx documentation.

.. _examples-not-covered:

Types of examples not directly covered by adopted technologies
--------------------------------------------------------------

Some scenarios are not well covered by the adopted technologies.
These are:

- UI-based tutorials
- Project-building tutorials

LSST will use UI-based tutorials in documentation of the Science Platform.
There isn't a technology that combines the content of a UI-based tutorial with a machine-testable plan.
In industry, UI-based tutorials are often periodically reviewed and updated by a QA or documentation team.
It's conceivable that UI tutorials could be co-developed with a Selenium test script (or similar).
Selenium is also commonly used in industry to generate screenshots for UI-based tutorials since it's often the *appearance* of the UI that changes most frequently.

Project-building tutorials are a common format for developer tutorials where the reader is guided through building a project consisting of multiple source files that are incrementally built up.
tut_ is a Sphinx extension that provides an approach to creating a project-based tutorial in Sphinx_.
It works with a Git repository where each branch contains the code at each stage of the project.

.. _pytest-approaches:

Approaches for integrating doctests with Stack testing
======================================================

Doctests are one of our adopted technologies for writing testable examples.
This section considers the various approaches for running and validating doctests as part of either the general software testing process or the documentation build.
In general, there are two types of approaches: run doctests through pytest while the software is being tested, or run doctests through Sphinx when the documentation is built.

.. _doctest-pytest:

Running doctests through pytest
-------------------------------

The main testing command for the Stack (of which the LSST Science Pipelines is part of) is :command:`scons test`.
SCons, in turn, runs pytest_, which provides test discovery, execution, and reporting.
Integrating doctests with pytest_, and thus :command:`scons test` is appealing because it would enable us to test doctests without changing developer workflows.

Pytest `supports doctests`__ through a ``--doctest-modules`` command-line argument.
In principle, pytest should discover all Python modules in the package and run their doctests, similarly to how it discovers test modules and executes them.

.. __: https://docs.pytest.org/en/latest/doctest.html

As a case study, the verify_ package uses doctests that are run by pytest using its ``--doctest-modules`` argument.
Note that in order for modules to be discovered, we had to specify the :file:`python`, :file:`tests`, and :file:`bin.src` directories where modules can be found in a standard LSST EUPS package.
In the :file:`setup.cfg` file, pytest is configured as:

.. code-block:: ini

   [tool:pytest]
       addopts = --flake8 --doctest-modules python bin.src tests
       flake8-ignore = E133 E226 E228 N802 N803 N806 N812 N815 N816 W504

The disadvantage of this approach is that the specification of ``python bin.src tests`` as default options through the :file:`setup.cfg` file prevents a developer from easily running pytest against a single test module.
Additional work is needed to understand why pytest cannot automatically discover LSST's Python modules by default.

In addition to Python modules, pytest can also gather and run doctests in reStructuredText files using the ``--doctest-glob`` argument.
For example: ``--doctest-glob=doc/**/*.rst`` would test all reStructuredText files in a package's documentation.

.. _pydoctestplus:

Enhancing pytest with Astropy's pytest-doctestplus
--------------------------------------------------

Astropy created a extension for pytest called pytest-doctestplus_ that enhances pytest-based doctest testing.
Its main features are:

- Processing doctests in reStructuredText files (which is now handled natively by Pytest).
- Approximate floating point comparison.
- Advanced doctest skipping control for modules.
- Integration with the pytest-remotedata_ plugin to enable skipping tests that require remote connections.

The floating point comparison feature is useful for avoiding test failures because of small floating point rounding differences between a doctest and an execution.
It handles cases like this:

.. code-block:: rst

   >>> 1. / 3.  # doctest: +FLOAT_CMP
   0.333333333333333311

Such a directive is likely useful to enable by default.

pytest-doctestplus_ allows developers to indicate that any doctests associated with a Python class, function, method, or whole module should be skipped through a ``__doctest_skip__`` module-level variable.

For example, to skip all doctests in the function ``get_http`` in a module:

.. code-block:: python

   __doctest_skip__ = ['get_http']

It also supports wildcard matching of names:

.. code-block:: python

   __doctest_skip__ = ['HttpClient.http_*']

An entire module can be skipped with a module-level wildcard:

.. code-block:: python

   __doctest_skip__ = ['*']

pytest-doctestplus_ provides a similar module-level variable to configure doctests that should be skipped if an optional dependency is not present.

In the context of reStructuredText pages, and within docstrings, pytest-doctestplus_ also provides several directives [#doctestplus-directives]_ for skipping tests and creating invisible doctests for environmental set up.

.. [#doctestplus-directives]

   These reStructuredText directives are not actually part of pytest-doctestplus_.
   In fact, they're shipped as part of the `sphinx-astropy`_ project as ``sphinx_astropy.ext.doctest``.
   This means that adding pytest-doctestplus_ also requires adding sphinx-astropy_ as a dependency.

The ``testsetup`` directive runs the code within the doctest, but does not display the code in the published documentation:

.. code-block:: rst

   .. testsetup::

      import lsst.afw.table as afwTable

``testsetup``, as the name implies, is useful for running setup code that would distract from the narrative of the page itself.

The ``doctest-skip`` directive renders the doctest, but skips executing and therefore testing, the code:

.. code-block:: rst

   .. doctest-skip::

      >>> 1 / 0

Finally, there is also a special comment that ignores all doctests on a page:

.. code-block:: rst

   .. doctest-skip-all

Overall, pytest-doctestplus_ appears to be a useful extension of pytest's basic doctest capability, and should be part of our solution for testing doctests.

.. _sybil-pytest:

Sybil
-----

An alternative to pytest_\ ’s ``--doctest-modules`` mode and pytest-doctestplus_ is Sybil_.
Sybil_ provides additional features for testing Python examples in reStructuredText/Sphinx documentation.

Features
^^^^^^^^

The main use case for Sybil over other systems is being able to build custom example parsers.
Whereas pytest_ and pytest-doctestplus_ only check for traditional doctests, Sybil_ provides additional parsers to check examples written in other types of syntax, such as ``code-block`` directives.

This flexibility is useful in cases where an author might write a function or class in a ``code-block`` directive, and then use a doctest to exercise that example class or function.
The code from both the ``code-block`` and doctest are treated as part of the same namespace.

In addition to the ``code-block`` parser, Sybil_ provides an `API for additional parsers`__ should we wish to provide examples in custom reStructuredText directives or in different languages or syntaxes.
For example, Sybil_ could operate on bash scripts or commands:

.. code-block:: rst

   .. code-block:: sh

      $ echo Hello world
      Hello world

.. __: https://sybil.readthedocs.io/en/latest/parsers.html#developing-your-own-parsers

Similarly, Sybil_ could also be extended to validate YAML or JSON-format code blocks.

Sybil also provides an ``invisible-code-block`` reStructuredText directive:

.. code-block:: rst

   .. invisible-code-block:: python

      import lsst.afw.table as afwTable

This directive can be used to execute code within the namespace of the page’s tests without rendering content onto the page.
Used judiciously, ``invisible-code-block`` could be useful for adding setup code and environment assertions to ensure that the examples are testable without adding distractions for readers.

In addition, Sybil provides a flexible skipping mechanism.
Using a ``skip`` reStructuredText comment, single examples or ranges of examples can be skipped.

.. code-block:: rst

   .. skip: next

   >>> 1 / 0

A range of examples can also be skipped:

.. code-block:: rst

   .. skip: start

   >>> 1+1
   2

   Some text...

   >>> 40+2
   42

   .. skip: end

Examples can also be skipped based on a logical test (the ``invisible-code-block`` directives provide a place to write auxiliary code for these tests).

.. code-block:: rst

   .. invisible-code-block::

      import os
      is_travis = os.getenv('TRAVIS') == 'true'

   .. skip: is_travis is False

   >>> os.getenv('TRAVIS')
   'true'

Integration with pytest
^^^^^^^^^^^^^^^^^^^^^^^

Sybil_ integrates with pytest_, among other Python test runners.
To use Sybil_ with pytest_, we would add a :file:`conftest.py` file to the doc directories of packages (or any other documentation project).
In this :file:`conftest.py` file we can configure features like the parsers mentioned mentioned in the previous section.

By executing pytest_ from a package’s root directory, as SCons already does, pytest_ should automatically discover the :file:`doc/conftest.py` file and begin testing the doctests in the reStructuredText source.
Thus Sybil can integrate well into DM’s existing pytest_\ -based testing system.

Finally, as alluded to above, Sybil_ is pitched squarely at running doctests in reStructuredText files, not in docstrings within Python modules.
Thus Sybil_ would need to be used in conjunction with pytest_ itself or pytest-doctestplus_ to test docstrings.

.. _extdoctest:

Testing doctests with sphinx.ext.doctest
----------------------------------------

Another method of testing doctests in documentation is as part of the Sphinx documentation build, rather than as part of the unit testing with pytest_.
sphinx.ext.doctest_, a Sphinx extension included with Sphinx, provides this capability.

sphinx.ext.doctest_ provides three methods for writing doctests:

1. Regular doctests that use the ``>>>`` syntax.
2. A ``doctest`` directive that provides control over test groupings, what doctest directives are applied to process the doctest, and whether or not to hide the doctest in the build site.
3. A ``testcode`` and ``testoutput`` directive pairing that allow writers to separate the block that displays the example code from the block that displays output.

This third method is unique to sphinx.ext.doctest_.
It gives authors the flexibility to separately introduce the input and output.
On the other hand, readers are used to seeing code and output paired together (not only do doctests pair code and output, but Jupyter Notebooks as well).

In addition to directives for writing the examples themselves, sphinx.ext.doctest_ also provides ``testsetup`` and ``testcleanup`` directives.
The content of these directives is automatically hidden, and are automatically run before and after, respectively, the test groups that they are associated with.
Similar to the ``invisible-code-block`` directive provided by Sybil_, the ``testsetup`` directive can both run preparatory code and also add variables to the namespace that can be used by the examples.

Finally, sphinx.ext.doctest_ provides means for conditionally skipping tests of examples.

.. _doctest-summary:

Summary of doctest testing approaches
-------------------------------------

There are several approaches to validating doctest-based examples that differ based on how they are run, where they run, and what features they add beyond the basic doctest library.

Of the options listed, we immediately dismiss the strictly pytest-driven approach as pytest-doctestplus_ provides an equivalent, but improved, feature set.
We also dismiss the sphinx.ext.doctest_ approach because it occurs during the documentation build, rather than during regular testing (driven by pytest).
We believe that documentation testing with pytest-based testing brings documentation integrity to the forefront of developer culture, and helps ensure that breakages of examples are more visible.
This reasoning also weights heavily in our :ref:`chosen approach to testing Jupyter Notebooks <testing-notebooks-summary>`.

The two approaches that are worthwhile to pursue are pytest-doctestplus_ and Sybil_.
pytest-doctestplus_ is advantageous because it can be used for both doctest examples in docstrings and in reStructuredText pages.
As such, pytest-doctestplus_ provides a uniform system for testing all doctests that developers write.

On the other hand, Sybil_ only operates on reStructuredText pages.
Compared to pytest-doctestplus_, though, Sybil_ has an extensible parsing system that can allow us to test examples in other languages, such as shell scripts or even HTTP calls.
Sybil_ also has a more flexible syntax for constructing invisible code blocks and for skipping tests.

There are fundamentally two options for applying doctests, then:

1. Use pytest-doctestplus_ to test all doctests.

2. Use pytest-doctestplus_ to only test doctests in docstrings, and then use Sybil to test reStructuredText pages.

From a capability standpoint, the second option is more appealing as it allows greater flexibility in testing examples in reStructuredText documentation.
The drawback is that developers must be aware of the distinction of writing doctests in these two environments, and learn the skip syntax for each.

.. _testing-notebooks:

Testing Jupyter Notebooks
=========================

.. _nbval-intro:

nbval for testing notebooks with pytest
---------------------------------------

nbval_ enables pytest_ to run on Jupyter Notebooks.
It determines whether the Jupyter Notebook, when re-executed, can reproduce the outputs stored in the notebook.
In this way, nbval_ treats Jupyter Notebooks as sophisticated doctests.

nbval_ provides different levels of control over how the output stored in the Jupyter Notebook is compared against output from executing the notebook in a test environment:

1. All cells can be tested by running as :command:`pytest --nbval`.
2. Only cells specially marked with a ``# NBVAL_CHECK_OUTPUT`` marker comment can be tested by running as :command:`pytest --nbval-lax`.
3. Checking all cells, but only after “sanitizing” the reproduced and stored outputs to avoid testing outputs that are intrinsically semi-random.

The ``--nbval-lax`` mode is a low buy-in means of testing notebooks by allowing authors to mark just those cells that are representative and known to be testable.
Note that their is a companion comment, ``# NBVAL_IGNORE_OUTPUT`` that causes nbval_ to skip testing a cell.
This is a useful escape valve for cells that are difficult or impossible to test.

The sanitization approach is more technically involved.
In this mode, we would provide an ini-format file with regular expressions and replacement strings.
nbval_ runs these regular expressions over the outputs and replaces the matched strings with a simplified replacement string that is tested against.

nbval_ has additional advanced features that are useful.
One is the ``# NBVAL_RAISES_EXCEPTION`` code comment that permits a cell to raise an exception, and directs nbval_ to test the traceback.

Instead of using Python code comments (which are user-visible), cells can also be annotated with Jupyter-native tags.

Finally, nbval_ can be configured to skip certain output types, such as ``stderr`` or ``application/javascript``.

nbval_ is known to be compatible with the xdist plugin for running tests in parallel.
In this case, an entire notebook would be run together as a unit.

It is not currently known how to control which Jupyter kernel pytest_ or nbval_ runs notebooks with, or whether this is configurable.

Overall, nbval_ is an excellent and comprehensive solution for ensuring that Jupyter Notebooks are reproducible.
One caveat that must be recognized, though, is that nbval_ requires that notebooks be committed into documentation repositories with their outputs committed.
This pattern runs counter to the practice of always stripping notebooks of outputs before committing them to a Git repository. 
Committing outputs increases the probability of merge conflicts should multiple authors be working on the same notebook simultaneously in separate branches.
This reinforces the notion that Jupyter Notebooks should only be used in special circumstances, such as tutorials, rather than as the primary format for all of LSST’s documentation and examples.

.. _nbpages-for-testing:

nbpages and nbconvert as a testing device
-----------------------------------------

The one potential drawback of the nbval_/pytest_ approach, described :ref:`above <nbval-intro>`, is that notebooks must be committed with their outputs.
As discussed, committing outputs makes Git diffs more difficult to interpret and increases the difficulty associated with resolving merge conflicts.
nbpages_, developed at the Space Telescope Science Institute, attempts to work around this issue by creating a notebook publishing workflow that rigorously uses notebooks committed to Git without outputs.

Essentially, nbpages_ is a front-end to nbconvert_.
nbpages_ runs nbconvert_ on each notebook in a repository, which executes the notebook programatically, and then converts the notebook into an output format such as HTML or reStructuredText.
As such, nbpages_ provides smoke-test level testing of notebooks.

Obviously, without having existing outputs, it is impossible for this method to discern whether the outputs are correct or not.
However, simply running the notebook programatically protects against notebooks that outright do not run.

.. _testing-notebooks-summary:

Summary of notebook testing approaches
--------------------------------------

Overall, nbval_ is a comprehensive solution for testing Jupyter Notebooks intended for documentation, and integrates into our existing pytest_ workflow.
An nbconvert_\ -based testing approach is not as compelling as actually validating the reproducibility of a notebook's outputs.
Further, the fact that nbval_ relies on notebooks where the outputs are included can be seen as a feature since it means that a documentation site can be regenerated without having to re-run the notebook itself.
This is a useful capability so that developer builds can run without a massive computational investment.
In summary, we should adopt nbval_ for projects such as the `pipelines.lsst.io <https://pipelines.lsst.io>`_ and `nb.lsst.io <https://nb.lsst.io>`_ documentation projects.

.. _notebook-execution:

Execution environments for testing examples and scripts
=======================================================

Since examples (whether they are are doctests or notebooks) and scripts are tested through pytest_, the environment where they are tested corresponds to the environment where the corresponding software is tested or documentation is built.
This section considers the specific execution for different LSST DM projects.

The LSST Science Pipelines test environment
-------------------------------------------

The LSST Science Pipelines is tested by ci.lsst.codes, a Jenkins CI deployment.
Developers run the stack-os-matrix job to test the LSST Science Pipelines during their regular development.
Since stack-os-matrix run the :command:`scons` command, which in turn runs pytest_, tests for doctests, notebooks, and scripts will be executed in the stack-os-matrix environment.

Parallelism is provided "for free" by virtue of the pytest xdist plugin.

Any datasets that are referenced by the examples must be avilable in the stack-os-matrix job as well.

The Nublado test environment
----------------------------

Examples for the LSST Science Platform generally assume access to the capabilities of the LSP.
For example, an example notebook for the notebook aspect will assume access to mounted filesystems and data access APIs.
These example notebooks will also assume that certain software is installed, whether they are command-line tools or Python packages.
This necessitates that examples for the LSP should be tested on the LSP itself.
Mocking the LSP is not practical.

To do this, we envision adding a capability to Nublado (the LSST JupyterLab-based platform) that allows API-based execution.
That is, through an HTTP API, the documentation build platform should be able to request that Nublado start up a headless pod running JupyterLab and execute a command.
Such a command might do this:

#. Clone a Git repository (at a specific commit) that contains the documentation being tested.
#. Temporarily install additional Python dependencies needed for the documentation and its tests.
#. Run a shell command (likely :command:`pytest`) to execute the tests.
#. Gather the output of the tests and return it to the caller.

In this scenario, the documentation would be built in a CI environment (likely ci.lsst.codes), but the tests would run in Nublado itself.
Another interesting possibility is that the same API access to Nublado could also be used to build the documentation itself, in addition to testing the documentation.
This would be useful for enabling custom Sphinx extensions, that LSST writes, to introspect the Nublado environment and automatically document features such as available Python packages, command-line tools, and datasets on shared file storage.

All of this, however, requires that LSST develop API access to Jupyter to spawn and operate headless JupyterLab pods.

References
==========

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa

.. _Developer Guide: https://developer.lsst.io
.. _verify: https://github.com/lsst/verify
.. _pipe_analysis: https://github.com/lsst-dm/pipe_analysis
.. _skymap: https://github.com/lsst/skymap
.. _pipe_tasks: https://github.com/lsst/skymap
.. _ap_verify: https://github.com/lsst/ap_verify
.. _dispatch_verify.py: https://github.com/lsst/verify/blob/master/bin.src/dispatch_verify.py
.. _inspect_job.py: https://github.com/lsst/verify/blob/master/bin.src/inspect_job.py
.. _lint_metrics.py: https://github.com/lsst/verify/blob/master/bin.src/lint_metrics.py
.. _parseLogs.py: https://github.com/lsst-dm/pipe_analysis/blob/master/examples/parseLogs.py
.. _plotSkyMap.py: https://github.com/lsst/skymap/blob/master/examples/plotSkyMap.py
.. _runRepair.py: https://github.com/lsst/pipe_tasks/blob/master/examples/runRepair.py
.. _run_ci_dataset.sh: https://github.com/lsst/ap_verify/blob/master/bin/run_ci_dataset.sh
.. _pytest: https://pytest.readthedocs.io/en/latest/
.. _shebangtron: https://github.com/lsst/shebangtron
.. _task topic:
.. _Task topic type: https://developer.lsst.io/stack/task-topic-type.html
.. _topic type: https://developer.lsst.io/stack/package-documentation-topic-types.html
.. _sphinx-argparse: http://sphinx-argparse.readthedocs.io/en/latest/ 
.. _DM Python Style Guide: https://developer.lsst.io/python/style.html
.. _measAlgTasks.py: https://github.com/lsst/meas_algorithms/blob/master/examples/measAlgTasks.py
.. _SourceDetectionTask: https://github.com/lsst/meas_algorithms/blob/ab9750a333ea586c47a619fd46eaf45e9985dcec/python/lsst/meas/algorithms/detection.py
.. _statisticsMaskedImage.py: https://github.com/lsst/afw/blob/master/examples/statisticsMaskedImage.py
.. _ticket647.cc: https://github.com/lsst/afw/blob/master/examples/ticket647.cc
.. _maskIo2.cc: https://github.com/lsst/afw/blob/master/examples/maskIo2.cc
.. _NewSuprimeCam.paf: https://github.com/lsst/afw/blob/7c9aa26c256174da0e9beb77f5fd941289262869/examples/NewSuprimeCam.paf
.. _doctest:
.. _doctests: https://docs.python.org/3/library/doctest.html
.. _Getting Started: https://pipelines.lsst.io/getting-started/index.html
.. _Jupyter Notebooks: https://jupyter-notebook.readthedocs.io/en/latest/
.. _notebook-demo: https://github.com/lsst-sqre/notebook-demo
.. _Astropy: http://docs.astropy.org/en/stable/
.. _Sphinx: http://www.sphinx-doc.org/en/master/
.. _nbreport: https://nbreport.lsst.io
.. _tut: https://github.com/nyergler/tut
.. _`What nobody tells you about documentation`: https://www.divio.com/blog/documentation/
.. _pytest-doctestplus: https://github.com/astropy/pytest-doctestplus
.. _pytest-remotedata: https://github.com/astropy/pytest-remotedata
.. _Sybil: https://sybil.readthedocs.io/en/latest/index.html
.. _sphinx.ext.doctest: http://www.sphinx-doc.org/en/master/usage/extensions/doctest.html
.. _nbval: https://github.com/computationalmodelling/nbval
.. _nbpages: https://github.com/eteq/nbpages
.. _nbconvert: https://nbconvert.readthedocs.io/
.. _sphinx-astropy: https://github.com/astropy/sphinx-astropy

.. |13| replace:: :ref:`QAWG-REC-13 <qawg-rec-13>`
.. |14| replace:: :ref:`QAWG-REC-14 <qawg-rec-14>`
.. |15| replace:: :ref:`QAWG-REC-15 <qawg-rec-15>`
.. |examples| replace:: :file:`examples`
