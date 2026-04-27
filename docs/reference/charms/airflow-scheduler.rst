Charmed Airflow Scheduler
=========================

``airflow-scheduler-k8s`` is a Kubernetes charm for the Apache Airflow scheduler
component. The scheduler is the heart of Airflow's task orchestration: it continuously monitors
all DAGs and tasks, triggers task instances when their dependencies are met, and submits them to
the configured executor for execution.

Core responsibilities
---------------------

* Parsing DAG files and building the task dependency graph.
* Monitoring DAG schedules and triggering new DAG runs at the appropriate time or in response
  to external triggers.
* Evaluating task dependencies and marking tasks as ready to run.
* Submitting ready task instances to the executor (e.g. the Kubernetes Executor).
* Performing DAG run and task instance health checks, detecting and handling failures.

Project and community
---------------------

Charmed Airflow is a member of the Ubuntu family. It is an open source project that warmly
welcomes community contributions, suggestions, fixes, and constructive feedback.

* `Code of conduct`_
* `Join the Discourse community forum`_
* `Contribute on GitHub`_
