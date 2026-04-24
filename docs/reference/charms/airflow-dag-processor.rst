Charmed Airflow DAG Processor
=============================

``airflow-dag-processor-k8s`` is a Kubernetes charm for the Apache Airflow standalone DAG
processor component. In a typical Airflow deployment, the scheduler is responsible for both
parsing DAG files and scheduling tasks. The standalone DAG processor separates these concerns:
it takes over responsibility for parsing and serialising DAG files into the metadata database,
allowing the scheduler to focus exclusively on scheduling.

Within the Charmed Airflow solution, the DAG processor component becomes valuable at scale,
when the volume of DAG files would otherwise slow down the scheduler's scheduling loop.
It integrates with ``airflow-coordinator-k8s`` for database access and shared
Airflow configuration.

Core responsibilities
---------------------

* Continuously scanning the configured DAG folder for new or modified DAG files.
* Parsing DAG files in separate sub-processes and serialising the resulting DAG objects into the
  Airflow metadata database.
* Detecting and reporting import errors in DAG files.
* Offloading DAG parsing work from the scheduler, improving scheduling throughput at scale.

Project and community
---------------------

Charmed Airflow is a member of the Ubuntu family. It is an open source project that warmly
welcomes community contributions, suggestions, fixes, and constructive feedback.

* `Code of conduct <https://ubuntu.com/community/code-of-conduct>`_
* `Join the Discourse community forum <https://discourse.charmhub.io>`_
* `Contribute on GitHub <https://github.com/canonical/airflow-core-operators>`_
