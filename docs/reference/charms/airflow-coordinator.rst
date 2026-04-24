Charmed Airflow Coordinator
===========================

``airflow-coordinator-k8s`` is a Kubernetes charm that acts as the central configuration
and integration hub of the Charmed Airflow solution. It does not run an Airflow process itself;
instead, it manages the shared infrastructure that all other Airflow component charms depend on:
the metadata database, and the merged Airflow configuration.

All core Airflow component charms — the API server, scheduler, triggerer, and DAG processor —
integrate with the coordinator to receive the configuration they need to operate. Executor
charms such as ``airflow-kubernetes-executor-k8s`` also integrate with the coordinator to
deliver executor-specific configuration back into the deployment.


Core responsibilities
---------------------

* Merging Airflow configuration from all integrated executor charms and distributing the
  combined configuration to all component charms via the ``airflow-config`` relation.
* Acting as the single source of truth for the Airflow configuration across the deployment.

Project and community
---------------------

Charmed Airflow is a member of the Ubuntu family. It is an open source project that warmly
welcomes community contributions, suggestions, fixes, and constructive feedback.

* `Code of conduct`_
* `Join the Discourse community forum`_
* `Contribute on GitHub`_
