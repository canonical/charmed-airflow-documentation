Charmed Airflow Kubernetes Executor
=====================================

``airflow-kubernetes-executor-k8s`` is a Kubernetes charm that enables a Charmed Airflow
deployment to run DAG task instances as individual Pods in a Kubernetes cluster. It supports
configuring the Airflow Kubernetes Executor, providing strong workload isolation, elastic capacity, and
per-task resource customisation.

The charm manages the Kubernetes resources required by worker Pods — a ConfigMap and a Secret —
and shares the resulting executor configuration and pod template with
``airflow-coordinator-k8s`` via the ``airflow-executor-config`` relation.

Core responsibilities
---------------------

* Creating and maintaining the Kubernetes ConfigMap and Secret required by Airflow worker Pods that contain
  the shared Airflow configuration.
* Building the pod template that Airflow uses when launching worker Pods, incorporating the
  ``base_image`` and ``namespace`` configuration.
* Sharing the executor type, pod template, and supplementary Airflow configuration with the
  coordinator charm.
  deployment.

Project and community
---------------------

Charmed Airflow is a member of the Ubuntu family. It is an open source project that warmly
welcomes community contributions, suggestions, fixes, and constructive feedback.

* `Code of conduct`_
* `Join the Discourse community forum`_
* `Contribute on GitHub`_
