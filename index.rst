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
These scripts are already documented using the `Task topic type <https://developer.lsst.io/stack/task-topic-type.html>`_ in LSST documentation.
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

.. |13| replace:: :ref:`QAWG-REC-13 <qawg-rec-13>`
.. |14| replace:: :ref:`QAWG-REC-14 <qawg-rec-14>`
.. |15| replace:: :ref:`QAWG-REC-15 <qawg-rec-15>`
