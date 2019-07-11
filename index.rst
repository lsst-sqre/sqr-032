:tocdepth: 1

.. sectnum::

Abstract
========

Examples and tutorials are important elements of successful documentation.
They show how something can be accomplished, rather than merely describing it.
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
It is through |15| that we are mandated to specify the conventions and formats for committing examples and scripts into the LSST code base, and provide the infrastructure for displaying these examples and scripts in end-user documentation.
It is also through |15| that we are mandated to provide infrastructure to test examples and scripts for correctness and reproducibility.
We interpret |13| as requesting that examples and scripts be conveniently displayed and listed through end-user documentation, to solve the issue that examples and scripts are generally not discoverable of both our own developers and the user community.
|13| can be accomplished in conjunction with |15|.

Finally, |14| requests that we establish a policy for how example code is maintained.
The issue is that examples can be broken when developers add or change APIs in the code base and it is not necessarily the responsibility of those authors to also update all examples and tutorials elsewhere in the LSST documentation and codebase.
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

.. [#taskindex] Tasks are listed in package documentation homepages and there are plans, described in :dmtn:`30` to list task by theme so that users can discover appropriate tasks without having to be familiar with the structure of the LSST code base.

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

We also see many executable scripts in :file:`examples` directories that associated with documentation.
An example is the runRepair.py_ script in the pipe_tasks_ package.
That script is associated with a page in the Doxygen documentation.
The reason it's a script is to be runnable: the script sets up a mock dataset, and then runs the ``lsst.pipe.tasks.repair.RepairTask`` on it.
This type of script fits the purpose of the original :file:`examples` framework, but there is a clear mandate from |15| to improve how these examples are created, managed, and tested.
Didactic script will be considered later in this technote as part of the examples portion of the work.

.. todo:: Add a link to that section.

Scripts in languages other than Python
--------------------------------------

Not all ad hoc scripts are written in Python.
For example, run_ci_dataset.sh_ in the :file:`bin` directory of ap_verify_ is a shell script that provides a higher-level interface to the ``ap_verify.py`` script.

Plan for consolidating and documenting scripts
==============================================

Based on the :ref:`review of existing ad hoc scripts <review-of-scripts>` in the LSST codebase, we can fulfill |13| (in relation to scripts) by introducing a systematic approach to including and documenting scripts in the LSST codebase.

Action: move all scripts to bin.src/ or bin/
--------------------------------------------

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

   This has the same effect as putting modules in :file:`bin.src` in an LSST EUPS package.

This will have the effect of making it possible for users to execute scripts without having to address EUPS environment variables.
Using the :file:`bin.src` directory specifically for Python-based scripts has the benefit of ensuring that the hash-bang is re-written properly to account for SIP security features in macOS.
Non-Python scripts can go directly in the :file:`bin` directory because shebangtron_ only updates the hash-bangs of Python scripts.

Action: mandate that the core code from scripts should reside in the main package for testability
-------------------------------------------------------------------------------------------------

Rather than putting the entirety of a script's code in the script module itself, which resides in :file:`bin.src`, we should encourage developers to put the entirely of a function's code in the main package.
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

In summary, by re-structuring scripts we can provide comprehensive unit tests for those scripts without having to treat scripts as a special case for testing.

Action: scripts are documented in package documentation
-------------------------------------------------------

Similar to how every function has a page in the Python API reference, and every task as a corresponding `Task topic page <Task topic type>`_, every script or command-line executable must have a corresponding documentation page.
The structure of this page should be designed and templated as a `topic type`_.
These documentation pages should be listed both on the package's homepage, and in a central index accessible from the https://pipelines.lsst.io homepage (to be specific) that gathers executables from all packages.
The script topic will use Sphinx extensions to auto-populate documentation from the script's code (see the :ref:`next action <adopt-argparse>`).

The script topic should also provide a way to add metadata about a script, such as a description or tags, to facilitate a useful index of scripts.

.. _adopt-argparse:

Action: adopt argparse for command-line scripts to enable auto-documentation
----------------------------------------------------------------------------

To facilitate automatic documentation of command-line interfaces, scripts should use standard frameworks for continuing their interface rather than working directly with arguments using `sys.argv`.
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
.. _Task topic type: https://developer.lsst.io/stack/task-topic-type.html
.. _topic type: https://developer.lsst.io/stack/package-documentation-topic-types.html
.. _sphinx-argparse: http://sphinx-argparse.readthedocs.io/en/latest/ 
.. _DM Python Style Guide: https://developer.lsst.io/python/style.html

.. |13| replace:: :ref:`QAWG-REC-13 <qawg-rec-13>`
.. |14| replace:: :ref:`QAWG-REC-14 <qawg-rec-14>`
.. |15| replace:: :ref:`QAWG-REC-15 <qawg-rec-15>`
