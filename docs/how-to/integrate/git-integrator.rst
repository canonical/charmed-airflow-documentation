.. _integrate-git-integrator:

Integrate with Git Integrator
==============================

`Git Integrator`_ is a Juju charm is designed to manage and share Git repository details and credentials with consumer charms. By integrating Git Integrator to Charmed Airflow, you provide the connection details necessary to define a Git DAG bundle, enabling Airflow to automatically load and sync DAGs directly from you remote repository.


Prerequisites
-------------

A running Charmed Airflow deployment. If you haven't deployed it yet, follow the :doc:`tutorial </tutorial>` first.

Deploy the Git Integrator charm
--------------------------------

You can deploy multiple instances of the ``git-integrator`` charm, one per repository, and integrate each with Charmed Airflow independently.

.. code-block:: bash

   juju deploy git-integrator --channel 1.0/stable

Configure the charm
--------------------

For full configuration options, including public and private repository authentication (HTTPS credentials and SSH), see the `Git Integrator`_ documentation on Charmhub.

Integrate with Charmed Airflow
-------------------------------

.. code-block:: bash

   juju integrate git-integrator:git airflow-coordinator-k8s:git

Wait for all units to return to ``active/idle``:

.. code-block:: bash

   juju status --watch 5s

Remove the integration
-----------------------

.. code-block:: bash

   juju remove-relation git-integrator:git airflow-coordinator-k8s:git
