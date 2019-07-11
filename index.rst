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

References
==========

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa

.. _Developer Guide: https://developer.lsst.io

.. |13| replace:: :ref:`QAWG-REC-13 <qawg-rec-13>`
.. |14| replace:: :ref:`QAWG-REC-14 <qawg-rec-14>`
.. |15| replace:: :ref:`QAWG-REC-15 <qawg-rec-15>`
