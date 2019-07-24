.. image:: https://img.shields.io/badge/sqr--032-lsst.io-brightgreen.svg
   :target: https://sqr-032.lsst.io
.. image:: https://travis-ci.org/lsst-sqre/sqr-032.svg
   :target: https://travis-ci.org/lsst-sqre/sqr-032
..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

##################################################################
Rendering and testing examples and tutorials in LSST documentation
##################################################################

SQR-032
=======

Examples and tutorials are important elements of successful documentation.
They show the reader how something can be accomplished, which is often more powerful and effective than a mere description.
Such examples are only useful if they are correct, though.
Automated testing infrastructure is the best way to ensure that examples are correct when they are committed, and remain correct as a project evolves.
Recently, in DMTN-085, the QA Strategy Working Group (QAWG)  issued specific recommendations to improve how examples are managed and tested in LSST's documentation.
This technote analyzes these recommendations and translates them into technical requirements.
Subsequently, this technote also provides an overview of how example code management and testing has been implemented.

**Links:**

- Publication URL: https://sqr-032.lsst.io
- Alternative editions: https://sqr-032.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-032
- Build system: https://travis-ci.org/lsst-sqre/sqr-032


Build this technical note
=========================

You can clone this repository and build the technote locally with `Sphinx`_:

.. code-block:: bash

   git clone https://github.com/lsst-sqre/sqr-032
   cd sqr-032
   pip install -r requirements.txt
   make html

.. note::

   In a Conda_ environment, ``pip install -r requirements.txt`` doesn't work as expected.
   Instead, ``pip`` install the packages listed in ``requirements.txt`` individually.

The built technote is located at ``_build/html/index.html``.

Editing this technical note
===========================

You can edit the ``index.rst`` file, which is a reStructuredText document.
The `DM reStructuredText Style Guide`_ is a good resource for how we write reStructuredText.

Remember that images and other types of assets should be stored in the ``_static/`` directory of this repository.
See ``_static/README.rst`` for more information.

The published technote at https://sqr-032.lsst.io will be automatically rebuilt whenever you push your changes to the ``master`` branch on `GitHub <https://github.com/lsst-sqre/sqr-032>`_.

Updating metadata
=================

This technote's metadata is maintained in ``metadata.yaml``.
In this metadata you can edit the technote's title, authors, publication date, etc..
``metadata.yaml`` is self-documenting with inline comments.

Using the bibliographies
========================

The bibliography files in ``lsstbib/`` are copies from `lsst-texmf`_.
You can update them to the current `lsst-texmf`_ versions with::

   make refresh-bib

Add new bibliography items to the ``local.bib`` file in the root directory (and later add them to `lsst-texmf`_).

.. _Sphinx: http://sphinx-doc.org
.. _DM reStructuredText Style Guide: https://developer.lsst.io/restructuredtext/style.html
.. _this repo: ./index.rst
.. _Conda: http://conda.pydata.org/docs/
.. _lsst-texmf: https://lsst-texmf.lsst.io
