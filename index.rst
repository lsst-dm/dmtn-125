:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

Introduction
============

From September 2018 to March 2019, LSST Data Management (DM) and Google Cloud conducted a proof of concept engagement to investigate the utility, applicability, and performance of Google Cloud services with regard to various DM needs.

`DMTN-078 <https://DMTN-078.lsst.io/>`_ laid out several potentially fruitful areas of collaboration and was the organizing skeleton for the engagement.
These areas related to `Qserv`_, the LSST parallel distributed database for large-scale catalog data products; Prompt Processing, the service that receives images and metadata in near real-time from the observatory and produces alerts for the community; and the `LSST Science Platform`_ (LSP), the environment for scientists to retrieve, process, and analyze LSST data products close to where they live.

.. _Qserv: https://ldm-135.lsst.io/
.. _LSST Science Platform: https://ldm-542.lsst.io/

This document describes what was accomplished in each area, including measurements obtained, and presents some conclusions about how the engagement went overall.

Qserv
=====

Goals
-----

The goals for this proof of concept, in decreasing order of priority, were to:

* Deploy Qserv under `Kubernetes`_ and `Google Kubernetes Engine`_ (GKE).
* Test performance in the cloud environment versus dedicated hardware.
* Try different price/performance points for Qserv storage.
* Compare Qserv with a cloud-native database service.
* Investigate next-to-database processing with cloud-native database storage.

.. _Kubernetes: https://kubernetes.io
.. _Google Kubernetes Engine: https://cloud.google.com/kubernetes-engine/

Qserv deployment had been manual prior to this; demonstrating that it could be deployed under Kubernetes, the deployment platform for the LSP and many other LSST DM services would be an advance.
Qserv is designed to maximize efficiency in an on-premises, dedicated hardware environment.
As more computing moves to the cloud, it is important to see how Qserv might perform in that environment, where constraints differ, and how cloud-native services would perform with the same catalogs, queries, and processing.

Results
-------

Deployment
^^^^^^^^^^

Qserv was successfully deployed on GKE as a `StatefulSet`_ with `persistent volumes`_.
GKE worked well as a testbed to improve the Kubernetes configuration for all deployment environments.
Qserv shards its data into chunks which are then distributed across multiple nodes.
Some queries access only a single chunk; others can query across all chunks in parallel.
The same number of nodes (30) and chunking scheme was initially used on GKE as had been used for tests on the LSST Prototype Data Access Center (PDAC) hardware at NCSA as reported in `DMTR-52 <https://dmtr-52.lsst.io/>`_.
Later, the Qserv replication service, which had not been previously been deployed under Kubernetes, was also successfully integrated, and that service was used to elastically redistribute chunks when the cluster was expanded from 30 nodes to 40.

.. _StatefulSet: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
.. _persistent volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Performance
^^^^^^^^^^^

Performance was tested using Google Cloud's `Persistent Disk`_ block storage.
This is a relatively expensive permanent store on network-attached disk.

.. _Persistent Disk: https://cloud.google.com/persistent-disk/

Queries were executed on three separate systems: the PDAC at NCSA (from DMTR-52), Qserv on Google Cloud, and the `Google BigQuery`_ database service.
Results are given in `Document-31100 <https://ls.st/Document-31100>`_.

.. _Google BigQuery: https://cloud.google.com/bigquery/

Query times for "point" queries limited to a single chunk and sometimes indexed were generally faster in the cloud than on the PDAC hardware.
But "scan" queries that processed large amounts of data were slower.
There is one data point that was not included in the above document: a ``COUNT(*)`` query on the ``wise_00.allwise_p3as_psd`` table took 1141.1 seconds in a "typical" run on the cloud, as opposed to the reported maximum of 1825.9 seconds.
This suggests that anecdotal but unrecorded measurements of approximately 20% degradation in query times for larger queries are reasonable.

Comparisons with other types of cloud storage were not done.
Temporary node-attached storage, while perhaps higher performance, would be unsuitable for a production deployment as there would be insufficient space and permanence.
Google `Cloud Storage`_ object store would be cheaper but would require major code modifications within Qserv.

.. _Cloud Storage: https://cloud.google.com/storage/

The results for BigQuery show significant speedups for queries that retrieve a limited number of columns, as expected due to BigQuery's columnar organization.
Spherical geometry primitives were able to be adapted for use in astronomical queries.
Proper data organization, in particular clustering the BigQuery tables by spatial index, along with the use of a spatial restriction primitive led to substantial improvements in query time for a near-neighbor query.
Retrieval of individual objects was relatively slow, however, due to BigQuery's startup time and lack of indexing.
It seems clear that it is possible, with some work on `ADQL`_ (Astronomical Data Query Language) translation and possibly creation of auxiliary tables, for BigQuery to handle the largest-scale catalog queries.

.. _ADQL: http://www.ivoa.net/documents/latest/ADQL.html

Possible alternatives to BigQuery include `BigTable`_, a non-relational database that might have more difficulties with ADQL translation and joins, and `Spanner`_, a SQL relational database that might be too featureful and expensive for LSST's needs.
Neither was investigated during the engagement.

.. _BigTable: https://cloud.google.com/bigtable/
.. _Spanner: https://cloud.google.com/spanner/

`Cloud Dataproc`_, a hosted `Apache Spark`_ and `Hadoop`_ data processing service, was not tested, but there is no reason to believe that it will not work as advertised against data stored in BigQuery.

.. _Apache Spark: http://spark.apache.org/
.. _Hadoop: http://hadoop.apache.org/
.. _Cloud Dataproc: https://cloud.google.com/dataproc/


Prompt Processing
=================

Goals
-----

The goals for this proof of concept, in decreasing order of priority, were to:

* Transmit images from LSST facilities to Google Cloud.
* Deploy the `Prompt Processing Database`_ (PPDB, now known as the Alert Processing Database or APDB) in the cloud.
* Test PPDB performance in the cloud environment, on larger rented machines than available as dedicated hardware.
* Attempt to use a different technology for PPDB.
* Investigate sharding the PPDB across multiple machines.
* Execute Alert Distribution and Filtering on Google Cloud with simulated alert clients for large-scale throughput tests.
* Execute Alert Production on Google Cloud to understand the cloud's suitability as a test platform.
* Investigate workflow/orchestration services for executing Alert Production.

.. _Prompt Processing Database: https://dmtn-113.lsst.io/

Since Prompt Processing runs in near-realtime, getting data rapidly into the cloud environment is a necessity if processing is to occur there.
The PPDB is a major performance bottleneck, as it needs to handle a significant query and update load for each visit processed.
Alert Distribution and Alert Production are reasonably well-understood; the cloud offers opportunities for testing at scale and possibly simplifying execution.

Results
-------

Data Transfer
^^^^^^^^^^^^^

Images were transmitted from a data transfer node in the AURA machine room in La Serena to a US storage bucket within Google Cloud Storage.
The configuration of the node, networks, and the data are given in `IT-991 <https://ls.st/IT-991>`_ and `DM-18125 <https;//ls.st/DM-18125>`_; the latter contains the measurements obtained.
The fastest network link available from La Serena to Santiago (where peering with Google's own network occurred) was a 10 Gbit/sec link.
As a result, the data to be transferred was scaled down appropriately.
Nevertheless, the results are not fully representative of the performance of the 100 Gbit/sec link that will be available for LSST Operations as there may be downstream bottlenecks, effects from multiple parallel transfer nodes, problems from large bandwidth-delay products, etc.

The Google `gsutil`_ tool was used to perform the copy.

.. _gsutil: https://cloud.google.com/storage/docs/gsutil

Simple regression over the 4 measured data points gives a large startup time of 5.6 sec, even with data in memory.
The transfer bandwidth derived from the regression (1500 Mbits/sec) is quite reasonable given the lack of tuning.
Overall, the results indicate that the Santiago-to-Google Cloud networking can handle large transfers, although it is yet to be proven that ten times the scale could be handled on a production basis.
Substantial further work would likely be required to reduce the transfer latency, where the goal is under 2 sec, if this were to be used as the primary channel for Prompt Processing image transfer.

PPDB
^^^^

The PPDB, in a PostgreSQL implementation, was successfully deployed on the cloud on a single large compute instance.
Its performance was tested using existing client code.
The results are documented in `DMTN-113 <https://dmtn-113.lsst.io/>`_.
On the cloud, it was possible to execute a more realistic scenario than on previous development hardware.
The client code could run on a separate machine from the database, and the database itself could run on a larger server (64 vCPUs versus 56 hyperthreads shared with clients; 10 TB SSD versus 2 TB NVMe + 3 TB SATA SSD and 7.3 TB RAID), althoough it should be noted that the cloud storage involved, though SSD, was still accessed over the network, potentially constraining bandwidth and I/O operations per second.

The performance was found to be roughly comparable with Oracle RAC, somewhat worse for writes/inserts.
With the larger machine size, it was possible to extend the PostgreSQL results to ~2 months of visits versus ~2 weeks on the previous hardware.

Alert Distribution and Production
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These goals were not attempted.
Existing tests were deemed to be sufficient to show the Alert Distribution scaling.
The pipeline code was not in a readily-deployable state for Alert Produciton testing.


LSST Science Platform
=====================

Goals
-----

The goals for this proof of concept, in decreasing order of priority, were to:

* Determine how to deploy Jupyter notebook pods into the cloud from a JupyterHub at NCSA.
* Determine how authentication and authorization can span environments.
* Determine how user files and user databases can be shared between the environments.
* Determine whether LSST data products need to be resident in the cloud or can be retrieved on demand.

The LSST Science Platform is already deployed on Kubernetes and instances have been running in the cloud since its inception.
The primary concern, as a result, is whether the cloud can be combined in a hybrid architecture with on-premises resources in the LSST Data Access Center.

Results
-------

Unfortunately, none of the goals could be accomplished, as insufficient expert staff time was available to research the problems or develop solutions.
Instead, a moderate scaling test (31 unique users for 2 hours) was run to generate data that could be used to better estimate costs for future cloud deployments of the LSP.
Some results are presented in `DM-17298 <https://ls.st/DM-17298>`_.

In particular, charges for the day of the test amounted to:
* $62.90 for compute
* $31.62 for memory
* $6.86 for storage (SSD + persistent disk)
* $1.62 for Cloud SQL
* $0.55 for egress (inter-zone and Americas)


Meta-Results
============

Kubernetes
----------

The engagement increased the level of comfort and familiarity with Kubernetes within the LSST team.
This is critical as it is serving as the primary deployment platform for many services.
In addition, developers became comfortable with GKE.
It offers a relatively simple, performant, and elastic implementation that is useful for test deployments.
The deployment of Qserv on Kubernetes was moved forward.

Cloud
-----

The engagement demonstrated the usefulness of rented machines for testing.
It educated Google staff as to LSST requirements, improving their ability to suggest appropriate services to meet those requirements.
Data was obtained that should enable more appropriate costing and consulting in the future.

Google Engineering
------------------

The ambitious goals for the engagement had been set with the hope that Google engineering talent could be significantly leveraged.
In the end, Ross Thomson did a large part of the Qserv and BigQuery testing after initial efforts by the LSST team.
Robinson Maureira assisted ably with Kubernetes and Google Cloud administration.
The Google staff members were responsive and met regularly.
But LSST was unable to frame problems in such a way that Google could drive the answers.
Instead, many issues ended up having to be resolved by LSST staff.

LSST Management
---------------

The ability to achieve the proof of concept goals turned out to be highly dependent on the availability of LSST staff time because of the nature of the problems that needed to be solved.
Setting the goals from above in order to address the greatest risks and unknowns in the overall LSST DM architecture proved to be somewhat unsuccessful.
Since those goals were often not directly relevant to immediate team milestones, the team managers (T/CAMs) tended not to allocate sufficient staff time.
Where time was allocated, it was used most effectively when management and staff were co-located.
Note that almost all significant progress occurred with people that the engagement manager (Vaikunth Thukral) could talk to on a face-to-face basis every week.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
