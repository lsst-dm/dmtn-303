###################################
Writeable Butlers for Science Users
###################################

.. abstract::

   The Butler is a major piece of Rubin's data access system, and both the project team and advanced early adopters have grown accustomed to the full read-write capabilities of the current DirectButler implementation, which connects directly to a SQL database.  For both scaling and security reasons, science users accessing official Rubin data products will instead go through the new RemoteButler, which instead interacts with an server via a REST API and can support at most very limited write operations.  To provide science users with complete Butler support, this technote proposes augmenting RemoteButler with affiliated personal data repositories with DirectButler access.  These personal repositories would store user-generated data products and be able to reference official data products from the main repository.

Add content here
================

See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.
