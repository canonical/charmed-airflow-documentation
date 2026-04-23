.. _integrate-s3-integrator:

Integrate with S3 Integrator
==============================

`S3 Integrator`_ is a Juju charm for supplying S3 storage credentials to
consumer charms. Integrating it with Charmed Airflow allows your deployment to
load DAGs from an S3-compatible bucket.

Prerequisites
-------------

A running Charmed Airflow deployment. If you haven't deployed it yet, follow
the :doc:`tutorial </tutorial>` first.

Deploy the S3 Integrator charm
-------------------------------

.. code-block:: bash

   juju deploy s3-integrator --channel 1/stable

Configure the charm
--------------------

Set the bucket and, for non-AWS endpoints, the S3 service URL:

.. code-block:: bash

   juju config s3-integrator \
     bucket=my-dags-bucket \
     path=/dags

For a non-AWS S3-compatible service (for example, MinIO), also set the
endpoint and region:

.. code-block:: bash

   juju config s3-integrator \
     endpoint=https://minio.example.com:9000 \
     region=us-east-1

Set credentials via the ``sync-s3-credentials`` action:

.. code-block:: bash

   juju run s3-integrator/leader sync-s3-credentials \
     access-key=<access-key> \
     secret-key=<secret-key>

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
