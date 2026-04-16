Charmed Airflow API Server
==========================

``airflow-api-server-k8s`` is a `Kubernetes charm`_ for the `Apache Airflow`_ REST API server
component. It exposes Airflow's stable REST API, which is used by external clients, the Airflow
UI, and other components to query and manage DAG runs, tasks, connections, variables, and other
Airflow objects.

Within the Charmed Airflow solution, the API server acts as the primary HTTP entry point for
the deployment. It is integrated with the ``airflow-coordinator-k8s`` charm, which manages
shared configuration and the database backend.

Core responsibilities
---------------------

* Serving the `Airflow REST API`_ for programmatic access to the Airflow platform.
* Exposing the Airflow web UI for browser-based interaction.
* Authenticating and authorising API requests.
* Integrating with the coordinator charm for database connectivity, shared Fernet keys, and
  Airflow configuration.

Project and community
---------------------

Charmed Airflow is a member of the Ubuntu family. It is an open source project that warmly
welcomes community contributions, suggestions, fixes, and constructive feedback.

* `Code of conduct`_
* `Join the Discourse community forum`_
* `Contribute on GitHub`_

.. _Code of conduct: https://ubuntu.com/community/code-of-conduct
.. _Join the Discourse community forum: https://discourse.charmhub.io
.. _Contribute on GitHub: https://github.com/canonical/airflow-core-operators
