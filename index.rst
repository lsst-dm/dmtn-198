:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

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

Overview
--------

The USDF will serve as the Rucio site. Rucio will maintain the global state of 
replication of files/objects across sites in a single central database there. Rucio services 
will be used to transfer files to Rucio Storage Elements (RSE) at each site.  Note that sites with RSEs will include the Data Facilities for Data Release Production but will also include the Chilean Data Access Center and may also include Independent Data Access Centers (IDACs) that require copies of data products.  Rules set in Rucio determine how files are transferred between RSEs.

Each site has storage (file systems, object stores, tape, etc.) registered with Rucio as one or more RSEs, and each site (except possibly IDACs) will have its own local Butler registry.  This registry will maintain a view of all local datasets.
There are RSEs at the USDF, Chile, United Kingdom Data Facility (UKDF), and French Data Facility (FrDF). No Rucio services are running at the remote sites. All Rucio commands must call back to the USDF to perform actions.

After any action in Rucio is completed, it is logged in a Rucio database at the USDF.  Rucio has a daemon, Hermes, which periodically takes new database entries, changes each one to a STOMP message and sends them to a message broker.

DBB requires that some files be automatically ingested into Butler repositories at the RSE sites after completing a Rucio transfer. We can trigger this ingestion by receiving 
Rucio messages from Hermes, examining the message's contents, and sending a message to a service running at the DF which will ingest the file into a Butler repository at that site.
The metadata for this ingestion will be obtained from the files themselves, if they are self-describing, or else transferred along with the files in separate JSON or 
YAML documents (or in the message to the broker itself), with one per file or one per batch.  The exact mechanism for generating this "sidecar" metadata is TBD.


Note that Registries do not communicate directly with each other.
In particular, there is no database-to-database replication associated with Butler Registries.

Also note that there are files that are part of the permanent survey record that are not Butler datasets.
These files are also transferred via Rucio according to policy, primarily to the FrDF.

Message Brokers
---------------

The message brokers under consideration are ActiveMQ, RabbitMQ, and Kafka.

Rucio uses ActiveMQ as its default message broker, in examples and in the default demonstration containers it distributes. ActiveMQ has two main versions, "Classic" and "Artemis".
Members of the collaboration have used RabbitMQ with Rucio as well.  Kafka has been mentioned as a possible broker. ActiveMQ supports the STOMP protocol Hermes uses directly. 
Both RabbitMQ and Kafka support the STOMP protocol via a plug-in.

The broker software must be maintained throughout operations, so community support of the broker is essential. ActiveMQ, RabbitMQ, and Kafka all have active user bases, and we
expect support (new updates, bug fixes, security patches) to continue for the foreseeable future. In addition, we can write clients for each of these brokers in a variety of 
languages. There are also a variety of plug-ins we could leverage in the future.

Plug-ins for ActiveMQ and Kafka are written in Java. RabbitMQ plug-ins are written in Erlang.  ActiveMQ and Kafka it support Apache Camel (see below), which supports message 
filtering and routing as part of its distribution.

The broker we're using to prototype our solution is ActiveMQ.  This may be changed to use Kafka in the future.


Rucio Hermes Daemon Configuration
---------------------------------

The Hermes daemon can use "queues" or "topics" based on a parameter in the rucio.cfg file.  STOMP messages prefixed are prefixed with "/queue" or with "/topic." 
ActiveMQ strips this prefix and (if necessary) creates either a message queue or a message topic and sends the message to that destination.

Message Queues
-------------

A message sent to a queue remains in that queue until it is consumed and its consumption is acknowledged. Generally, this is used to ensure that messages 
can be consumed and processed by a client or group of clients working together. There is only one message, so the first client retrieves the message 
obtains it, and no other clients will receive it.

Topics and Durable vs Non-durable Topic subscriptions
-----------------------------------------------------

A message sent to a topic is broadcast to multiple clients subscribed to that topic. Each client gets its own copy of the message.  

How the broker treats that message depends on whether or not the topic subscriber is "durable" or "non-durable."  

In a durable topic subscription, if a message is sent and the client is down, the broker remembers that the subscription was 
durable and retains any unread messages until the client resubscribes. This type of subscription is helpful if the client comes and goes. 

In a non-durable topic subscription, if a message is sent and a client is down, it will not have the opportunity to receive the 
message, and that message is lost. This type of subscription is useful if receiving all messages isn't necessary, such as a client used for intermittent debugging. 

Note that topics aren't durable or non-durable; the topic subscriptions can either be durable or non-durable.

Message Filtering
-----------------

Message filtering allows a message broker or client to obtain a subset of messages from the main message flow. We will use this to identify messages that 
would trigger a butler ingest at a particular RSE site and only transmit those messages to that site. 

Both versions of ActiveMQ (Classic and Artemis) support both server-side and client-side message filtering using a simple SELECT-like syntax for data in a message 
header.  ActiveMQ Artemis can filter messages in the body of the message, but the body of the message must be in XML. Hermes transmits this information in JSON in the 
body of the message.

Since we can not directly filter data kept in the body of messages, we will use Apache Camel in a broker plug-in. Apache Camel will allow us to examine message body
information and route messages to message brokers at RSE sites as appropriate. This will be specified as a combination of XML in the ActiveMQ configuration file and a custom 
Java plug-in to ActiveMQ.

We will filter message keys "event_type:transfer-done" (indicating the file transfer has completed) and "payload:scope:<scope of the transferred data>," and then send to the 
broker at an RSE site based on the contents of "payload:dst-rse:<destination RSE>."  We might be able to use other event_types to for sets of files, but this is still TBD.

Issues
------

Each RSE site should have a message broker associated with it, so messages sent from Hermes to the USDF broker can forward those messages to satellite DF 
message brokers. This approach relies on the message brokers themselves synchronizing the messages properly, allowing access to the message broker queue locally. 
ActiveMQ has several strategies to connect brokers over a WAN and how best to pass traffic between brokers. The topology of the brokers, the queues,
and the plug-ins defined for each site needs to be explored.  

We need to keep in mind that network bandwidth requirements between sites and the extra traffic that brokers add to running message brokers. Therefore, 
message TTL should either be set high enough to not expire before a client or not set at all.

We should also note the maintenance of custom broker plug-ins.  ActiveMQ and Kafka plug-ins are written in Java and RabbitMQ plug-ins are written in 
Erlang. It would probably be a lot easier to find a Java programmer than an Erlang programmer if software features were needed to be added or bug fixes implemented.

Approach
--------

The topology of the broker network should be hub/spoke, meaning that we should configure all RSE site message brokers to connect directly to the 
message broker at the USDF.  In this way, the brokers handle the message transaction traffic, and so consumption of messages is dealt with locally,
rather than having client programs connect to the USDF's message broker. This configuration also permits us to set up local monitoring of message traffic.

Butler ingest clients should use durable topic subscriptions instead of queues or non-durable topic subscriptions. Using a durable topic subscription 
effectively allows the messages to be read as a queue. If the Butler ingest service went down, the message broker would still retain messages for the service 
until it reconnected. We could use non-durable topic subscriptions to the same topic and for monitoring clients.  We will write a plug-in message filter using 
Apache Camel. We will use this plug-in in conjunction with routing mechanisms to forward messages to the appropriate site.

Federated Message Broker Diagram
--------------------------------

.. figure:: /_static/FederatedBrokerDiagram.png
   :name: fig-federated-broker-diagram

   Federated Message Broker Diagram

This diagram shows the file transfer paths and messaging paths for DBB services. The diagram also shows the federation of message brokers, one at each 
satellite DF connected to the primary message broker at the USDF.  

All file state changes in a local RSE are transmitted from that site using the Rucio utilities (or APIs) to communicate to RUCIO at the USDF. This
activity happens in all cases. For example, when a file changes state in RSE at UKDF, it must register directly to the USDF; it doesn't proxy through the 
FrDF, even though the UKDF will be transferring files to the FrDF, not the USDF directly.

The Rucio system uses a daemon, Hermes, to send notification messages about changes in file status. The Hermes daemon periodically queries a database,
retrieves recent entries, formats them as STOMP messages, and sends them to the message broker at the USDF. The broker at the USDF then forwards the 
messages to the appropriate broker at the satellite sites. Messages are filtered so only the messages that are meant for a destination and indicate that a 
file is meant for Butler ingest are forwarded. 

Each satellite site has a Butler ingest daemon that reads messages from the local broker and ingests files into the Butler at that site. The Butler ingest 
daemon should batch incoming messages so ingests can be grouped.



Files
=====

Most files are expected to be stored in an object store at each location.
Some locations may choose to use a filesystem instead.

The Large File Annex is currently thought of as containing two types of files: one type that is ingested into a Butler and used as a dataset and another type that remains as a read-only object only.


Databases
=========

Qserv databases are not part of the DBB.
Instead, canonical Parquet files copied via the DBB are transformed, partitioned, and ingested into local Qservs.

The Alert Production Database is internal to the Alert Production and resides only at the USDF.

The Prompt Products Database (including Solar System Objects), the Transformed Engineering and Facilities Database, the Exposure Log, and any other databases within the Consolidated Database are replicated to other Data Access Centers via native database replication technology.
