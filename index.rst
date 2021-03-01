..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::


Introduction
============

Rubin Observatory will publish records to about 249 Telemetry, 390 Commands, and 533 Event Logs topics from about 60 different subsystems of the telescope producing ~200GB/day.
This data is recorded in the  :abbr:`EFD (Engineering and Facility Database)` at the Summit in Chile and needs to be replicated to the project-wide accessible instance of the EFD at :abbr:`LDF (LSST Data Facility)`.

The current `EFD architecture`_ is based on Apache Kafka and InfluxDB.
At the time of this writing, there are at least five solutions for data replication in Kafka:

* `Confluent Replicator`_ from Confluent (commercial offering)
* `Brooklin`_ from Linkedin (open source)
* `Uber uReplicator`_ from Uber (open source)
* `Mirus`_ from Salesforce (open source)
* `MirrorMaker 2`_ (open source)

.. _EFD architecture: https://sqr-034.lsst.io/#introduction
.. _Confluent Replicator: https://docs.confluent.io/current/connect/kafka-connect-replicator/index.html
.. _Brooklin: https://github.com/linkedin/brooklin
.. _Uber uReplicator: https://github.com/uber/uReplicator
.. _Mirus: https://github.com/salesforce/mirus
.. _MirrorMaker 2: https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0


Confluent Replicator was the first solution we tried (see :ref:`confluent-replicator`), but the cost to purchase the `Confluent enterprise license`_ makes it prohibitive for the project.

Notice that we did not include MirrorMaker on the list.
MirrorMaker is a tool to copy data from one Kafka cluster to another, and does not perform a **full replication** as we discuss later.
In fact, all the five solutions above were developed to overcome `limitations found in MirrorMaker`_.

.. _Confluent enterprise license: https://docs.confluent.io/5.5.0/control-center/installation/licenses.html#enterprise-connectors-lm
.. _limitations found in MirrorMaker: https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0#KIP382:MirrorMaker2.0-Motivation

In this technote, we investigate these solutions and justify adopting **MirrorMaker 2** to replace Confluent Replicator in the EFD replication service.



Requirements
============

A good replacement for the Confluent Replicator must satisfy the following requirements:

1. Preferentially be based on the Kafka Connect framework.
2. Support the active/standby and active/active replication scenarios.
3. Automatically create kafka topics at the destination kafka cluster if they do not exist.
4. Automatically sync topic configuration between clusters.
5. Configure topics to be replicated based on a regex expression and have the ability to exclude topics from replication.
6. Provide metrics to monitor the replication throughput and the replication latency.
7. Provide error policy configuration, for example, if the replication fails retry or ignore, etc.

Advantages of using the Kafka Connect framework
-----------------------------------------------

- Kafka Connect offers dynamic configuration via a REST API and has a framework for distributing work across the Kafka cluster when more throughput is needed.
- We are already using Kafka Connect to run the InfluxDB Sink, Amazon S3 Sink and the JDBC source and sink connectors.
- We are familiar with the Kafka Connect management API, and we can reuse some of the tools we already have in place.
- We are also familiar with the Kafka Connect metrics for monitoring.


Replicating topics and schemas
------------------------------

In the EFD, the creation of new kafka topics in the source cluster is done by SAL Kafka.
That happens dynamically as SAL Kafka subscribes to new CSCs.
In the destination cluster, that must be done by the replication solution.
That means the replication solution must be able to automatically detect new topics and changes in topic configuration in the source cluster and propagate them to the destination cluster.

We use Avro to encode records in Kafka. SAL Kafka is also responsible for creating the Avro schemas and register them with the Schema registry in the source cluster. The replication solution also must be able to replicate the schemas to the destination cluster.


Active/standby and active/active replication scenarios
------------------------------------------------------

In the active/standby replication scenario, data replication happens in only one direction, from the active Kafka cluster (Summit EFD) to the standby Kafka cluster (LDF EFD).
The active/standby scenario works well for the EFD use case. In the EFD use case, there is no need to modify the data at LDF so we don't need to replicate it back to the Summit.
An important simplification of the active/standby scenario is that we can preserve topic names in the source and destination Kafka clusters.
By preserving the topic names, we can execute the same queries, dashboards and notebooks at the Summit EFD and at the LDF EFD transparently.\ [#preserving-topic-names]_
As discussed in section :ref:`schema-migration`, in the active/standby scenario we can migrate Avro schemas without modification.

The active/active scenario may be important for other applications like the :abbr:`OWL (Observatory Wide Log)` system.
In the OWL use case, the data can be modified at LDF and therefore it must be replicated back to the Summit.
The active/active replication scenario requires topic renaming and consequently schema translation when synchronizing the topic schemas between the Kafka clusters.

.. [#preserving-topic-names] If topics are renamed in Kafka, to achieve the same result we would have to rename them back to their original names in each connector individually when writing to other formats.


Evaluation
==========

Brooklin
--------

Brooklin is developed at LinkedIn and was designed as a generic bridge for streaming data.
It is comparable to solutions in the streaming ecosystem like Facebook’s Wormhole and Kafka Connect.
You can learn more about Brooklin from their engineering blog post `Streaming Data Pipelines with Brooklin`_

.. _Streaming Data Pipelines with Brooklin: https://engineering.linkedin.com/blog/2017/10/streaming-data-pipelines-with-brooklin

It has successfully replaced MirrorMaker at Linkedin (see also the `Waifair use case`_).
It does more than we need and it is not based on the Kafka connect framework.
Adopting Brooklin would add unnecessary complexity to the current EFD architecture.

.. _Waifair use case: https://tech.wayfair.com/2020/06/scaling-kafka-mirroring-pipelines-at-wayfair/

Uber uReplicator
----------------

We could build on Uber's `uReplicator`_, which solves some of MirrorMaker problems.
However, uReplicator uses Apache Helix to provide some of the features already present in Kafka Connect, like a REST API, dynamic configuration changes, cluster management, coordination, etc.
We choose to give preference to solutions that use native components of Apache Kafka.
That said, uReplicator was also a major source of inspiration for the MirrorMaker 2 design.

.. _uReplicator: https://github.com/uber/uReplicator

Mirus
-----

`Mirus`_ is developed at Salesforce.
It is essentially a custom Kafka Connect Source Connector specialized for reading data from multiple Kafka clusters.
Mirus also includes a monitor thread that looks for task and connector failures with optional auto-restarts, and custom JMX metrics for monitoring.
You can read more about Mirus in this `blog post <https://engineering.salesforce.com/open-sourcing-mirus-3ec2c8a38537>`_.

.. _Mirus: https://github.com/salesforce/mirus

Mirus is able to replicate new topics partitions created in the source cluster to the destination cluster, however it cannot replicate `other topic configurations`_.

.. _other topic configurations: https://github.com/salesforce/mirus/issues/20


MirrorMaker 2
-------------

`KIP-382`_ proposed MirrorMaker 2 as a replacement for MirrorMaker with a new multi-cluster, cross-data-center replication engine based on the Kafka Connect framework, recognizing the need for a native replication solution for Apache Kafka.

.. _KIP-382: https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0

Both Confluent Replicator and MirrorMaker2 leverage the Kafka Connect framework and share many design principles.
One  feature that is key for **full replication** is to preserve identical topic configuration between source and destination cluster.
For example, if new partitions are added on to the source cluster, that should be automatically detected and replicated to the destination cluster.

MirrorMaker 2 introduces the concept of remote topics, which are replicated topics referencing a source topic via a naming convention.
Any partitions in a remote topic are remote partitions and refer to the same partitions in the source topic.

Because MirrorMaker 2 is the native replication solution for Apache Kafka and is built on the Kafka Connect framework it is the best alternative to replace Confluent Replicator for the EFD replication service.

A toy example
=============

In this section, we create a toy example to illustrate how MirrorMaker 2 works.
Our goal is to replicate the topic ``foo`` from a source kafka cluster to a destination kafka cluster.

Figure 1 shows the setup of our toy example.


.. figure:: /_static/toy_example.svg
   :name: Toy example setup.

   Toy example setup showing the source and destination kafka clusters.


Download the `docker-compose.yaml`_ file to create the services for the toy example:

.. code-block:: bash

  $ docker-compose up -d

Kafka Connect runs on the destination cluster and the MirrorMaker 2 plugins are already installed in the kafka connect docker image.
Next we briefly describe each of the MirrorMaker 2 connectors and use the ``kafkaconnect`` tool to configure them.

.. _docker-compose.yaml: https://raw.githubusercontent.com/lsst-sqre/kafka-connect-manager/master/connectors/mirrormaker2/docker-compose.yaml


Configuring the connectors
--------------------------

MirrorHeartbeat
^^^^^^^^^^^^^^^

The MirrorHeartbeat connector emits heartbeats in the source cluster.
Heartbeats are internal topics replicated to the destination cluster to demonstrate connectivity between the kafka clusters.
A heartbeat topic indicates that MirrorMaker 2 can replicate records for the configured topics.
By default, MirrorMaker 2 always replicate heartbeats.
In more complex topologies, heartbeats can be used to auto discover replication flows, report on latency, etc.

In our toy example, you can run ``kafkaconnect`` using:

.. prompt:: bash $,# auto

  $docker-compose run --entrypoint /bin/bash kafkaconnect

And from inside the container upload the connector configuration:

.. prompt:: bash $,# auto

  $cat <<EOF > heartbeat-connector.json
  {
    "name": "heartbeat",
    "connector.class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "source.cluster.alias": "src",
    "target.cluster.alias": "destn",
    "source.cluster.bootstrap.servers": "broker-src:29093",
    "topics": "foo"
  }
  EOF
  $kafkaconnect upload --name heartbeat heartbeat-connector.json

The ``kafkaconnect upload`` command will validate and upload the connector configuration to Kafka Connect.
You can check if the connector is running using:

.. prompt:: bash $,# auto

  $kafkaconnect status heartbeat

MirrorCheckpoint
^^^^^^^^^^^^^^^^

In addition to topic replication, MirrorMaker 2 also replicates the state of consumer groups.
The MirrorCheckpoint connector periodically emit checkpoints in the destination cluster, containing offsets for each consumer group in the source cluster.
The connector will periodically query the source cluster for all committed offsets, and keep track of offsets for each topic being replicated to make sure they are in sync.

.. prompt:: bash $,# auto

  $cat <<EOF > checkpoint-connector.json
  {
    "connector.class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "name": "checkpoint",
    "source.cluster.alias": "src",
    "source.cluster.bootstrap.servers": "broker-src:29093",
    "target.cluster.alias": "destn",
    "target.cluster.bootstrap.servers": "broker-destn:29092",
    "topics": "foo"
  }
  EOF
  $kafkaconnect upload --name checkpoint checkpoint-connector.json

MirrorSource
^^^^^^^^^^^^

The MirrorSource connector has a pair of consumers and producers to replicate records, and a pair of AdminClients to replicate topic configuration.

In our toy example, the destination cluster pulls records and topic configuration changes for the ``foo`` topic from the source cluster using the MirrorSource connector. In a more complex setup, for example involving active/active replication, both source and destination clusters can run the MirrorSource connector.

The MirrorSource connector can be configured to replicate specific topics via a whitelist or a regex and run multiple replication tasks. In our toy example, we are running only one replication task:

.. prompt:: bash $,# auto

  $cat <<EOF > mirror-source-connector.json
  {
    "checkpoints.topic.replication.factor": 1,
    "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "heartbeats.topic.replication.factor": 1,
    "name": "mirror-source",
    "offset-syncs.topic.replication.factor": 1,
    "refresh.topics.interval.seconds": 60,
    "replication.factor": 1,
    "source.cluster.alias": "src",
    "source.cluster.bootstrap.servers": "broker-src:29093",
    "sync.topic.acls.enabled": false,
    "sync.topic.configs.enabled": true,
    "sync.topic.configs.interval.seconds": 60,
    "target.cluster.alias": "destn",
    "target.cluster.bootstrap.servers": "broker-destn:29092",
    "tasks.max": "1",
    "topics": "foo",
    "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
  }
  EOF
  $kafkaconnect upload --name mirror-source mirror-source-connector.json

See KIP-382 for a list of `configuration properties`_ for MirrorSource.\ [#configuration-properties]_
Notice that we explicitly disable the synchronization of topic ACLs.
We also set the replication factors for the remote and internal topics to match the number of brokers in each cluster in our toy example.

.. _configuration properties: https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0#KIP382:MirrorMaker2.0-ConnectorConfigurationProperties

At this point, you can see the initialization of the three connectors from the Kafka Connect logs:

.. prompt:: bash $,# auto

  $docker-compose logs connect-destn

In particular, the ``heartbeats`` internal topic is created on the destination cluster, and the ``mm2-offset-syncs.destn.internal`` topic is created in the source cluster to keep track of the remote topic offsets.

.. [#configuration-properties] Because MirrorMaker 2 is a relatively new project, there is no much documentation available yet besides KIP-382. Notice that the list of configuration properties presented in the table is not complete and some of the default values do not correspond to the actual default values in the code.

Creating a topic in the source cluster
--------------------------------------

Lets create the topic ``foo`` with one partition in the source cluster:

.. prompt:: bash $,# auto

  $docker-compose exec broker-src kafka-topics --bootstrap-server broker-src:29093 --create --topic foo --partitions 1 --replication-factor 1

.. note::

  By default ``refresh.topics.interval.seconds`` and ``sync.topic.configs.interval.seconds`` are set to 600 seconds. In our toy example, we changed both to 60 seconds to refresh topics and sync topic configurations more frequently to the destination cluster.

The remote topic is then replicated to the destination cluster:

.. prompt:: bash $,# auto

  $docker-compose exec broker-destn kafka-topics --bootstrap-server broker-destn:29092 --list

Notice that, by default, the remote topic name is prefixed by the source cluster alias ``src.foo``.

Producing Avro records
----------------------

You can use the ``kafka-avro-console-producer`` tool to produce Avro records to the ``foo`` topic in the source cluster.
``kafka-avro-console-producer`` can also register the Avro schema for the ``foo`` topic with the Schema Registry.
In the example, we create a simple schema for the ``foo`` topic with the fields ``baz`` and ``bar``.

.. prompt:: bash $,# auto

  $docker-compose exec schema-registry-src kafka-avro-console-producer --bootstrap-server broker-src:29093 --topic foo --property value.schema='{"type": "record", "name": "foo", "fields": [{"name": "bar", "type": "string"}, {"name": "baz", "type": "float"}]}'

You can paste a few records and then finish with ``Ctrl+D``.

.. prompt:: bash $,# auto

  {"bar": "John Doe", "baz": 1}
  {"bar": "John Doe", "baz": 1}
  {"bar": "John Doe", "baz": 1}

From the Kafka Connect logs you can see MirrorSource pulling records from the ``foo`` topic and committing offsets for its consumer.


Finally, you can verify that a change in the original topic configuration is in fact replicated to the remote topic. For example, this command will add another partition to the ``foo`` topic:

.. prompt:: bash $,# auto

  $docker-compose exec broker-src kafka-topics --bootstrap-server broker-src:29093 --alter --topic foo --partitions 2

and the new partition is added to the remote topic after 60 seconds:

.. prompt:: bash $,# auto

  $docker-compose exec broker-destn kafka-topics --bootstrap-server broker-destn:29092 --describe --topic src.foo

.. _schema-migration:

Migrating Avro schemas
----------------------

Avro schemas are needed at the destination cluster to deserialize the Avro messages before writing to  InfluxDB or to other storage formats.

To replicate schemas, we need to replicate the ``_schemas`` internal topic to the destination cluster.
That is done by explicitly adding this topic to the ``topics`` configuration property in the MirrorHeartbeat, MirrorCheckpoint and MirrorSource connector configurations.

One problem, however, is that, by default, remote topics are renamed, and Kafka Connect will fail trying to find a subject name that corresponds to the default remote topic name.

Confluent Replicator solves this problem with schema translation (see :ref:`confluent-replicator`).
That feature is missing in MirrorMaker 2.

A possible solution in MirrorMaker 2 is to set ``"source.cluster.alias": ""`` and ``"replication.policy.separator": ""``.
This way  because topics and subject names are preserved, no schema translation is required. Aslo the name of the `_schemas` topic is preserved no specific configuration is required for the Schema Registry at the destination cluster.

Once you apply these configurations, you can use the ``kafka-avro-console-consumer`` tool to consume records from the remote topic ``foo`` using the Schema Registry at the destination cluster to deserialize the messages:

.. prompt:: bash $,# auto

  $docker-compose exec schema-registry-destn kafka-avro-console-consumer --topic foo --from-beginning --bootstrap-server  broker-destn:29092 --property schema.registry.url=http://schema-registry-destn:8084


Design considerations
=====================

In this section we discuss some design considerations for the EFD replication service.

- The retention period for the Summit Kafka cluster is planned to be 72h, or one tenth of the 30 days Summit InfluxDB retention period. With this retention period we can recover data from the Summit Kafka cluster after a downtime in the EFD replication service at LDF or a downtime at the active InfluxDB/Chronograf instance at the Summit.

- We have observed an end-to-end latency of ~1 second for the EFD replication service. As a result, the standby InfluxDB/Chronograf instance at LDF could be used as failover for the active InfluxDB/Chronograf instance at the Summit.

- Telemetry, Commands, and Event Logs records are published at different data rates, and Telemetry records have the highest throughput. While Telemetry is the most important source of information for the LDF EFD, Commads and Event Logs have proved to be useful for interpreting Telemetry data and debugging the system. We might consider a separate replication flow for Telemetry if needed.

- When a remote topic is consumed, for example by the InfluxDB Sink connector, the corresponding Avro schema must be available at the Schema Registry at LDF. Currently, there is no mechanism to ensure that schemas are replicated before their corresponding topics. If a deserialization error occurs at the destination cluster, the MirrorSource connector needs to be restarted.

- We have observed that a Kafka Connect cluster with 3 pods running 10 MirrorSource connector tasks can handle the current throughput from the Summit EFD. We can scale up the the EFD replication service by adding more pods in the Kafka Connect cluster and by increasing the number of MirrorSource connector tasks to handle a larger number of topics and an increasing throughput.

- A good practice is to have a separate Kafka Connect deployment for the Mirrormaker 2, to isolate its connectors from other connectors running in the cluster. We are not doing that at the moment.

- We plan on monitoring MirrorMaker 2 metrics and using Kapacitor to detect interruptions in the EFD replication service.

- We did not discuss in this document a mechanism to automatically migrate the Chronograf dashboards or kapacitor alert rules from the Summit to LDF. Currently, this is done manually.

Conclusion
==========

Based on this investigation, MirrorMaker 2 is the best open source alternative to Confluent Replicator for the EFD replication service.
It has been tested and is currently operational in the active/standby scenario to replicate EFD topics and schemas from the Summit EFD to the LDF EFD.

MirrorMaker 2 is a relatively new project.
One feature that is still missing is schema translation.
We overcame that by preserving the topic names in the Summit and LDF Kafka clusters.
However, schema translation is necessary if we decide to replicate EFD data from multiple origins to the LDF EFD (e.g. base, or Tucson Teststand), or if we implement the active/active replication scenario.

We discussed some design considerations for the EFD replication service. In particular, we observed an end-to-end latency of ~1 second between the Kafka clusters at the Summit and LDF with the current number of topics and throughput. We also showed that we can increase the number of pods in the Kafka Connect cluster and the number of MirrorSource connector tasks to handle a larger number of topics and an increasing throughput.

.. _confluent-replicator:

Appendix A - Confluent Replicator
=================================

We have tried Confluent Replicator, initially, as a solution for the EFD replication service.

In the active/standby replication scenario, the Replicator source connector runs on the destination cluster and consumes topics from the source cluster.
Replicated topics are prefixed with ``summit.{topic}`` to indicate their origin, the Summit EFD, in this case.

.. _active/standby replication scenario: https://docs.confluent.io/current/multi-dc-replicator/index.html#multi-dc

Schema migration follows the `continuous migration model`_.
Confluent Replicator continuously migrate schemas from the source cluster to the destination cluster.
`Schema translation`_ ensures that subjects are renamed following the topic rename strategy when migrated to the destination cluster.

.. _continuous migration model: https://docs.confluent.io/current/schema-registry/installation/migrate.html#schemaregistry-migrate

.. _Schema translation: https://docs.confluent.io/current/tutorials/examples/replicator-schema-translation/docs/index.html

Here's an example of a configuration for the Confluent Replicator that includes topic replication and schema migration with schema translation:

.. code-block:: bash

  {
    "connector.class": "io.confluent.connect.replicator.ReplicatorSourceConnector",
    "dest.kafka.bootstrap.servers": "'${DEST_BROKER}'",
    "header.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
    "key.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
    "src.kafka.bootstrap.servers": "'${SRC_BROKER}'",
    "schema.subject.translator.class": "io.confluent.connect.replicator.schemas.DefaultSubjectTranslator",
    "schema.registry.topic": "_schemas",
    "schema.registry.url": "'${DEST_SCHEMA_REGISTRY}'",
    "src.consumer.group.id": "replicator",
    "tasks.max": "1",
    "topic.whitelist": "_schemas,foo",
    "topic.rename.format": "summit.${topic}",
    "value.converter": "io.confluent.connect.replicator.util.ByteArrayConverter"
  }

.. figure:: /_static/confluent-replicator.svg
   :name: Set up for testing Confluent Replicator.

   Topic replication and schema migration with Confluent Replicator.

Note that Kafka Connect ``bootstrap.servers`` configuration must include the URL of the destination Kafka cluster and that the destination Schema Registry must be configured in IMPORT mode.
To initialize the destination Schema Registry in IMPORT mode, firs set ``mode.mutability=True``.
The Schema Registry at the destination cluster must be empty when doing that. See `schema migration configuration`_ for details.

.. _schema migration configuration: https://docs.confluent.io/current/schema-registry/installation/migrate.html#id1

Confluent recommendation is to deploy the Replicator source connector at the destination cluster (remote consuming).
In some scenarios involving a VPN remote consuming is not possible, and we deployed the Confluent Replicator at the source cluster (remote producing).


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
