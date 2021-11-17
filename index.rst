:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

Introduction
============

This document describes the decisions made in the course of implementing the Data Backbone design from :dmtn:`122`.

Replication and Transport
=========================

The Data Backbone (DBB) will be used to move file/object data for production purposes between Data Facilities, including the US Data Facility (USDF), French Data Facility (FrDF), and UK Data Facility (UKDF).
It will also be used to move "offline", non-time-sensitive raw data from the Observatory in Chile to the USDF and potentially to the FrDF.
Data products to be served directly from Data Access Centers in Chile or elsewhere (IDACs) will be moved via the DBB, as will data products sent to other partners, as described in :dmtn:`147`.

These large-scale movements will be coordinated via Rucio,

Policies will be used to direct transfers from appropriate sources and to appropriate destinations.
For example, the UKDF may obtain its data from the FrDF rather than from the USDF or the Observatory.
The UKDF will not receive all raw images but instead will only receive images from a certain pre-designated portion of the sky.
Appropriate metadata labels for the raw image files, as well as policies using those labels, will be configured into Rucio to enable this restrictive data transfer.

Deciding that intermediate data products are no longer needed will be handled by workflow execution or campaign management mechanisms.
Rucio will be used to execute this removal across all replicas.

Download of files to end users will be provided by Virtual Observatory and/or other services within the Rubin Science Platform; it is not expected to be handled by Rucio.

Tiering of files to nearline or archival storage is handled by storage infrastructure, not Rucio.

Similarly, in a hybrid cloud model where permanent storage is on-premises but a cache is maintained in the cloud to improve latency for users, cache maintenance is the responsibility of a dedicated caching service, not Rucio.


Location and Metadata
=====================

Rucio will maintain the global state of replication of files/objects across sites.
It is understood that this is kept in a single central database, which will be located at the USDF.

In addition, each Data Facility will have its own local Butler Registry.
This registry will maintain a view of all local datasets.
Data Access Centers that provide Butler access will also have their own Registry, which is expected to be separate from any processing Registry used by a co-located Data Facility.

As a file or batch of files is transferred by Rucio, it will need to trigger an ingestion into the local Butler.
The exact mechanism for triggering this is TBD.
Each file requires appropriate metadata, akin to that produced by the ``astro_metadata_translator`` package for raw images.
This includes identifying values for the appropriate dimensions for the dataset type.
The metadata for this ingestion will be obtained from the files themselves, if they are self-describing, or else transferred along with the files in separate JSON or YAML documents, either one per file or one per batch.
The exact mechanism for generating this "sidecar" metadata is TBD.

Note that Registries do not communicate directly with each other.
In particular, there is no database-to-database replication associated with Butler Registries.

Also note that there are files that are part of the permanent survey record that are not Butler datasets.
These files are also transferred via Rucio according to policy, primarily to the FrDF.


Files
=====

Most files are expected to be stored in an object store at each location.
Some locations may choose to use a filesystem instead.

The Large File Annex is currently thought of as containing two types of files: one type that is ingested into a Butler and used as a dataset and another type that remains as a read-only object only.
The files to be ingested are identified by looking for a known, pre-specified set of CSC/generator/MIME type values in largeFileObjectAvailable event messages.
The contents of the Large File Annex object store on the Summit are replicated by the DBB to the USDF; the LFA files to be ingested are then placed in the same Butler as images and data products.
The DBB should be able to be configured to replicate ingested LFA files (and ingest into a destination Butler) just like any other ingested dataset.
It should also be able to be configured to replicate non-ingested LFA files if need be (without ingestion at the destination, of course), maintaining the object name (with a different bucket/root).


Databases
=========

Qserv databases are not part of the DBB.
Instead, canonical Parquet files copied via the DBB are transformed, partitioned, and ingested into local Qservs.

The Alert Production Database is internal to the Alert Production and resides only at the USDF.

The Prompt Products Database (including Solar System Objects), the Transformed Engineering and Facilities Database, the Exposure Log, and any other databases within the Consolidated Database are replicated to other Data Access Centers via native database replication technology.
