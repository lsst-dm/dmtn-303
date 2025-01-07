###################################
Writeable Butlers for Science Users
###################################

.. abstract::

   The Butler is a major piece of Rubin's data access system, and both the project team and advanced early adopters have grown accustomed to the full read-write capabilities of the current ``DirectButler`` implementation, which connects directly to a SQL database.  For both scaling and security reasons, science users accessing official Rubin data products will instead go through the new ``RemoteButler``, which instead interacts with a server via a REST API and can support at most very limited write operations.  To provide science users with complete Butler support, this technote proposes augmenting ``RemoteButler`` with affiliated personal data repositories with ``DirectButler`` access.  These personal repositories would store user-generated data products and be able to reference official data products from the main repository.

Overview
========

In addition to being able to access official data products via a read-only ``RemoteButler`` client, science users in the notebook aspect of the RSP can already create their own personal butler data repositories in their personal storage area, using a small file-based SQLite database.
These personal data repositories can be accessed with the full read-write ``DirectButler`` client, and can be used to run any LSST- or user-developed pipeline, provided the necessary input datasets are present in the personal data repository.
For a sufficiently expert and patient user who can copy those input datasets from the official data repository in advance, this is arguably all we need to provide - but of course most users are not expert or patient, and a few new features would dramatically improve the usability of this scheme even for patient experts.
In particular:

- Populating a personal data repository with all of input datasets needed to run a pipeline is difficult; it's very similar to the process of building a ``QuantumGraph``.
  It would make much more sense to instead first build a ``QuantumGraph`` against the official data repository (via a ``RemoteButler`` client), and then use the ``QuantumGraph`` to drive the transfer.
  It would likely be even better to to use the ``QuantumGraph`` itself to drive processing (via the limited, execution-oriented ``QuantumBackedButler``), and transfer outputs (and optionally inputs) into the personal data repository only at the end.

- Transferring datasets from the official data repository to a personal data repository by copying is inefficient and wasteful.
  It would be much better to instead have a way for the personal butler repository to *reference* datasets in the official data repository.
  This is mostly already doable via the ``transfer=None`` mode, but only when a usuable permanent URL is available (i.e. not for signed URLs with finite lifetimes).

- SQLite does not handle concurrency well, especially on filesystems (like NFS) that do not provide a conformant implementation of POSIX locking.
  Using ``QuantumBackedButler`` to drive all processing largely sidesteps this problem, but while this is technically already doable it is not advertized or frequently exercised as something users should interact with directly; it probably needs UI improvements and possibly a new, higher-level interface.

With this additional functionality in place, we should have a minimum-viable "satellite" butler system, in which a personal data repository works in concert with the official data repository to provide most of the functionality we expect most users to need.
This system still would still have some rough edges, however, and here the problems become more difficult to solve:

- While users can nominally share personal data repositories by leveraging the same filesystem permissions they would use to share non-butler data, this exacerbates the SQLite concurrency problem and it does not provide a way for users to "publish" their data products to a broad audience (e.g. via the official data repository or something similarly accessed via a ``RemoteButler``).

- While the ``QuantumGraph`` describing a processing run can be built against either the offical data repository or the personal data repository, it isn't a straightforward way to built a ``QuantumGraph`` against the combination, in which some inputs are provided by each repository.
  Manual transfers into the personal repository provide an (inconvenient) workaround for this problem, and a system that allows personal data products to be published to the official data repository might provide another option for some use cases.

Finally, using RSP notebook-aspect storage for both file artifacts and databases limits the degree to which this solution can scale.
This may not be an immediate problem; as long as user resources (number of CPUs, storage, memory) are limited to what's available in the RSP anyway, the personal data repository architecture is unlikely to be a bottleneck, and the relationship between this system and potential user-batch services is not clear.
But a more scalable architecture would certainly be desirable if the costs (especially development costs, at this stage) are not too high.
One clear possibility is providing object storage resources for users in addition to filesystem storage; a personal butler repository could handle this transparently, and it could make it much cheaper to provide more butler-friendly storage to users.
Another would be using personal PostgreSQL database space to hold butler database records - especially if we already need to provide personal database space to users for other reasons.

Each of the pieces of missing functionality introduced above is described in greater detail later in this technical note.

Minimum Viable Satellite Butlers
================================

QuantumGraph generation on RemoteButler
---------------------------------------

The current ``QuantumGraph`` generation algorithm relies extensively on temporary tables, which are emulated in ``RemoteButler`` by re-running the original query each time the temporary table is used.
This means that ``QuantumGraph`` generation *probably* already works on ``RemoteButler``, but inefficiently, and to our knowledge this hasn't been tried.
It would nevertheless be better for ``QuantumGraph`` generation to have an option to avoid temporary tables, by instead re-uploading the data IDs it would select from the temporary table each time they are needed (the ``Query.join_data_coordinates`` method we already have).
This may actually be faster for ``DirectButler`` as well, since we often want to join in data IDs that are a small subset of those in the temporary table (and a subset that will have already been computed in the client).
If that turns out to be the case, we should probably drop temporary table support from the query system, as ``QuantumGraph`` generation was the driving user case for it.
We should probably drop it from ``RemoteButler`` regardless, as emulated support is probably worse than no support.

QuantumBackedButler Transfers from RemoteButler to DirectButler
---------------------------------------------------------------

After a ``QuantumGraph`` is generated against the official data repository, we need to be able to execute that graph such that:

- input datasets are fetched from the ``RemoteButler`` to the offical data repository (with URL signing as needed);
- output datasets are written into the datastore root of the personal data repository;
- the "transfer" stage that inserts rows into the personal data repository's database can (optionally) transfer inputs as well as outputs, including setting up the collection structure in which the inputs were found in the offical data repository.

This requires additional persistent information that would have to be put into the ``QuantumGraph`` or some other file.

Signed URLs in DirectButler
---------------------------

In order for a personal data repository to reference an official data product (instead of copying it), the ``DirectButler`` client will need to be able to obtain signed URLs from the ``RemoteButler`` server.
This could be implemented as a new ``Datastore`` subclass (to be chained in with a regular ``FileDatastore`` for the personal datasets in that repository), an extension to the current ``FileDatastore``.
It could also be implemented directly within ``DirectButler``, but since we want this functionality in ``QuantumBackedButler`` as well, implementing it in ``Datastore`` seems to make more sense.

User Interfaces
---------------

TODO

Further Extensions
==================

Multi-Butler QuantumGraph Generation
------------------------------------

TODO

Publishing Collections
----------------------

TODO


External Storage
================

Personal PostgreSQL Databases
-----------------------------

TODO

Personal Object Storage
-----------------------

TODO
