.. _integrate-s3-integrator:

Integrate with S3 Integrator
==============================

`S3 Integrator`_ is a Juju charm used to manage and provide S3-compatible storage credentials to other applications. By relating S3 integrator to Charmed Airflow, you securely share the access details required to define an S3 DAG bundle, allowing your deployment to load and sync DAGs directly from an S3 bucket.

Prerequisites
-------------

A running Charmed Airflow deployment. If you haven't deployed it yet, follow
the :doc:`tutorial </tutorial>` first.

Deploy the S3 Integrator charm
-------------------------------

.. code-block:: bash

   juju deploy s3-integrator

.. note::
   For detailed information on available channels, tracks, and the latest stable releases, please refer to the `S3 Integrator`_ documentation on Charmhub.

Configure the charm
--------------------

For full configuration options, including bucket setup, credentials, and
non-AWS endpoints, see the `S3 Integrator`_ documentation on Charmhub.

Integrate with Charmed Airflow
-------------------------------

.. code-block:: bash

   juju integrate s3-integrator:s3-credentials airflow-coordinator-k8s:s3

Wait for all units to return to ``active/idle``:

.. code-block:: bash

   juju status --watch 5s

.. note::

   You can integrate multiple ``s3-integrator`` applications with the
   coordinator charm — one per bucket.

Remove the integration
-----------------------

.. code-block:: bash

   juju remove-relation s3-integrator:s3-credentials airflow-coordinator-k8s:s3
