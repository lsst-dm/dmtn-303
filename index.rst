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

This technote is explicitly not about so-called "user batch" services, for which the architecture is still too uncertain to design against.
It does reference user batch in a few places where commonalities are clear.

Minimum Viable Satellite Butlers
================================

QuantumGraph Generation on RemoteButler
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

We need high-level command-line and notebook-friendly tools for at least the following tasks:

- Create a new, empty satellite data repository from the official data repository.
  This needs to set up a datastore that can reference official datasets (with URL signing), and it may be convenient for it to always register a few instruments and/or skymaps, and maybe even write curated calibrations.

- Create and run a ``QuantumGraph`` with inputs from the official data repository or the personal data repository and outputs going to the personal data repository, using ``QuantumBackedButler``.
  In the minimum-viable system the input source would strictly be one repository OR the other, but in the future we may be able to support combinations, and should plan for that in the UI.
  I would recommend splitting this up into multiple subcommands, centered around a user-visible directory that holds the ``QuantumGraph`` file and some transfer metadata, similar to how BPS interacts with its "submit" directory.
  I think BPS is overall a better UI starting point than ``pipetasks``, but this is something we should put some real design thought and vetting into.
  The working directory for run state could even be the directory of the new RUN collection within the datastore root, since deleting that manually prior to the transfer job would actually be fine, but it may be a little close to the edge what would not be fine for the user to do manually.

- Explicitly transfer (by referencing) datasets from the official data repository to a personal one, using user-provided queries to identify the datasets.
  This may be best implemented by augmenting ``butler transfer-from`` with a notebook-friendly interface and a way to default the source data repository, since the personal repository has to know something about the official repository anyway to sign URLs.


Further Extensions
==================

Multi-Butler QuantumGraph Generation
------------------------------------

Generating a ``QuantumGraph`` whose overall inputs come from more than one data repository is a hard problem because the algorithm does not know in advance which of several constraints (the user-provided query string, the existence of various dimension records, and the existence of the input datasets in the collection search path) best constrains the set of data IDs that will go into the graph.
Inferring which constraint to start with is entirely analogous to planning the execution of a SQL query, so our ``QuantumGraph`` generation algorithm delegates to the butler database by forming a single query with all of those constraints.
When there are multiple databases, this is impossible.

This approach already fails for some hard ``QuantumGraph`` generation problems, leading to the creation of the ``--dataset-query-constraint`` argument to ``pipetask``, which allows the user to indicate which overall-input datasets (if any) are the best ones to use to constrain the graph.
Using this option successfully typicall requires expert help, however, so it's more of a workaround than a real solution.

In order to get multi-Butler ``QuantumGraph`` generation working well for end users, we need to come up with and test heuristics for splitting up the "best constraint" problem into multiple queries, often one for each data repository, that must be executed in a particular order.
It may also involve requiring the user to provide better hints about which constraints are likely to be relevant, but if so, they need to be more intuitive than ``--dataset-query-constraint``.
This work should be driven by the concrete ``QuantumGraph`` problems we expect to encounter in practice, and we expect it to proceed incrementally, with support for *some* kinds of multi-Butler ``QuantumGraph`` problems being available well before others, and fully-general support may never happen.
Many common problems should be solvable with a single constraint query against one of the two repositories (most often the personal data repository, because it is smaller), and the challenge is recognizing these cases and identify which of the two repositories to use.
This needs to be done with care, because a bad initial constraint query can be extremely expensive, and too many of these could overload the ``RemoteButler`` server.


Publishing Collections
----------------------

To facilitate sharing between science users, we would ideally provide a way for users to "publish" datasets and collections back to the official data repository as a federated data product.
This does not need to be fully automated, in the sense that we may want project staff to sign off on any publishing request, and it does not need to be immediate.
This greatly mitigates the problem of writing to the (in general) highly-replicated official data repository database; while the ideal scenario is a fully frozen database, a system comprised of a single read-write database servers and many read-only replicas should be possible as well.

Aside from the work involved in setting up the appropriate kind of database replication, we need to make sure publishing does not break any caching in the ``RemoteButler`` server, and then write a new API endpoints and client-side UIs for initiating, reviewing, and completing publish requests.

A major open question here is how the actual transfers of file artifacts and metadata would actually work.
The simplest scenario involves the client packaging up all of the content to be transferred into a ``zip`` or ``tar`` file and pushing that to the ``RemoteButler`` server, where it would have to be staged somewhere until the transfer is approved and completed.
More efficient options in which the server pulls the content from the personal data repository are much more likely to be viable if we provide personal PostgreSQL databases and/or object storage, as discussed in the next section.

The publishing mechanism is probably something we should try to share with any future user-batch implementation in which user-generated data products land in a butler data repository, regardless of whether that data repository is the same RSP personal data repository this technote describes.

We ultimately want the publishing system to support multiple levels of access control, in which users or groups retain ownership of the datasets and collections they have published, and can grant access to other users and groups instead of making them world-readable.
These access controls need to be implemented in the ``RemoteButler`` server (queries should not return datasets a user does not have access to) and the URL signing server.
Where to store permission state like access control lists - in the ``RemoteButler`` database vs. a separate one - is an open question.

External Storage
================

Personal PostgreSQL Databases
-----------------------------

Providing personal PostgreSQL database space to science users (with a direct SQL driver, not some HTTP intermediary) is something the project is considering for reasons other than just Butler support, and if that functionality is available we should strongly consider using this database storage to back personal data repositories, instead of relying on SQLite.
Having a separate namespace for Butler and other personal tables (i.e. no "one namespace per user" rule) is the only requirement we believe that Butler usage would impose on a general personal-PostgreSQL system.
PostgreSQL-backed data repositories are much more scalable and have received most of the focus in Butler optimization work.
They can also be centrally managed, which may help us provide user support, and unlike RSP SQLite databases it is plausible that they could also be used for user batch.

Personal PostgreSQL-backed butler databases do have some disadvantages:

- They are much harder to completely delete and reset (something inexpert users will want to do quite often).
- They are harder to share with other users (sharing a full SQLite data repository via filesystem permissions or copying is not ideal, but it may be fine for simple, common cases where a full "publish" request seems like overkill).
- If file artifact storage is still on the RSP filesystem, it may be hard to maintain data repository consistency, since the database could be somewhat centrally-managed but the file artifacts will not be at all centrally-managed.

Personal Object Storage
-----------------------

Personal space in an object store could be much cheaper than RSP filesystem storage, but it requires more sophisticated URL signing and permissions to allow users and possibly groups to own files, with access mediated by the Butler client (note that this is still a ``DirectButler``, interacting with a server or URL signing only, as in the case of personal data repositories referencing official datasets).
This is at least similar to the functionality needed for user and group ownership and sharing of published datasets, but it may not be identical.

As noted earlier, personal object storage works best when paired with personal PostgreSQL database space rather than a SQLite database.
At the very least, the SQLite database itself cannot be accessed through object storage (POSIX filesystem access is required), and maintaining consistency between database and file storage will be easier if both are on the RSP filesystem or both are more centrally managed.
