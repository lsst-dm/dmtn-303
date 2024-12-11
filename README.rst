.. image:: https://img.shields.io/badge/dmtn--303-lsst.io-brightgreen.svg
   :target: https://dmtn-303.lsst.io
.. image:: https://github.com/lsst-dm/dmtn-303/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-303/actions/

###################################
Writeable Butlers for Science Users
###################################

DMTN-303
========

The Butler is a major piece of Rubin's data access system, and both the project team and advanced early adopters have grown accustomed to the full read-write capabilities of the current DirectButler implementation, which connects directly to a SQL database.  For both scaling and security reasons, science users accessing official Rubin data products will instead go through the new RemoteButler, which instead interacts with an server via a REST API and can support at most very limited write operations.  To provide science users with complete Butler support, this technote proposes augmenting RemoteButler with affiliated personal data repositories with DirectButler access.  These personal repositories would store user-generated data products and be able to reference official data products from the main repository.

**Links:**

- Publication URL: https://dmtn-303.lsst.io
- Alternative editions: https://dmtn-303.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-303
- Build system: https://github.com/lsst-dm/dmtn-303/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-303
   cd dmtn-303
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-303.lsst.io whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-303.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
