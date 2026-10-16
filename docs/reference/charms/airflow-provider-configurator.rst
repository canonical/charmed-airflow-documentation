Charmed Airflow Provider Configurator
=====================================

``airflow-provider-configurator`` is a `Kubernetes charm`_ that lets Charmed Airflow
operators configure Airflow providers — connection parameters, credentials, and feature
flags — without manually editing ``airflow.cfg``.

The charm reads non-sensitive provider configuration from an ``.ini`` file in a Git
repository (kept in sync through a ``git-sync`` sidecar) and sensitive values from a
`Juju user secret`_, so credentials never have to live in the repository. It combines
the two sources and shares the result with ``airflow-coordinator-k8s`` over the
``airflow-provider-configuration`` relation, where it is merged into the configuration
distributed to the rest of the deployment.

Core responsibilities
---------------------

* Continuously syncing non-sensitive provider configuration from a Git repository, on a
  configurable poll interval, via a ``git-sync`` sidecar container.
* Reading sensitive provider configuration (such as passwords and tokens) from a Juju
  user secret, keeping it out of the Git repository.
* Combining the two sources and sharing the resulting provider configuration with
  ``airflow-coordinator-k8s`` for merging into ``airflow.cfg``.
* Dropping any provider configuration that belongs to a reserved section, that is, one
  that providers are never allowed to set: ``api``, ``api_auth``, ``core``,
  ``dag_processor``, ``database``, ``default``, ``fab.auth_backends``,
  ``kubernetes_executor``, and ``webserver``. This check is applied to both the ``.ini``
  file and the Juju user secret. Dropped settings are reported in the unit status
  message and logged, and do not block the unit. The coordinator applies a further check
  of its own, dropping provider configuration that collides with the configuration it
  already manages.

Only one ``airflow-provider-configurator`` application is supported per Charmed Airflow
deployment, and it accepts a single Git integrator application.

Project and community
---------------------

Charmed Airflow is a member of the Ubuntu family. It is an open source project that warmly
welcomes community contributions, suggestions, fixes, and constructive feedback.

* `Code of conduct`_
* `Join the Discourse community forum`_
* `Contribute on GitHub`_