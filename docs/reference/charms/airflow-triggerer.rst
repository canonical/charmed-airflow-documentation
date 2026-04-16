Charmed Airflow Triggerer
=========================

``airflow-triggerer-k8s`` is a Kubernetes charm for the Apache Airflow triggerer
component. The triggerer adds support for *deferrable operators* — a
pattern that allows task instances to suspend themselves and release their worker slot while
waiting for an asynchronous event (e.g. a file arriving in object storage, a sensor condition,
or an external API state change).

Within the Charmed Airflow solution, the triggerer runs as a separate process alongside the
scheduler and integrates with ``airflow-coordinator-k8s`` for shared configuration and database
access.

Core responsibilities
---------------------

* Running an asyncio event loop that manages active triggers registered by deferrable operators.
* Monitoring triggers for completion conditions and resuming the associated task instances when
  their condition is met.
* Freeing executor capacity by allowing deferred tasks to yield their worker slot while waiting.
* Integrating with the coordinator charm for database connectivity and Airflow configuration.

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
