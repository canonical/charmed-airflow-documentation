.. _deploy-with-terraform:

Deploy with Terraform
=====================

This guide shows how to deploy Charmed Airflow using `Terraform`_ and the
`Juju Terraform Provider`_.
Terraform automates the deployment of all charms and their integrations in a
single, reproducible plan.

Prerequisites
-------------

* A Juju controller (v3.1+) bootstrapped on a Kubernetes cluster.
  See the :doc:`tutorial </tutorial>` for setup instructions.
* `Terraform CLI`_ (v1.12+).


Clone the Charmed Airflow Solutions repository
----------------------------------------------

Clone the ``charmed-airflow-solutions`` repository that contains the Terraform
module:

.. code-block:: bash

   git clone https://github.com/canonical/charmed-airflow-solutions.git
   cd charmed-airflow-solutions/modules/charmed-airflow

Deploy with the Local Executor (default)
----------------------------------------

The default deployment uses Airflow's **Local Executor**, where the Airflow Scheduler
runs tasks in local subprocesses.

Before deploying, generate a `Fernet key`_ and store it as a Juju secret.
Airflow uses this key to encrypt sensitive data in its metadata database.

.. code-block:: bash

   juju add-secret fernet-key-secret \
     fernet-key="$(python3 -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')"

.. note::

   Note the secret ID returned by ``juju add-secret``. You can also find it
   with ``juju list-secrets``.

Create a ``terraform.tfvars`` file, passing the Fernet key secret via the
coordinator's config:

.. code-block:: hcl

   model_uuid = "<your-model-uuid>"

   airflow_coordinator = {
     config = {
       fernet_key_secret = "secret:<secret-id>"
     }
   }

   postgresql = {
     profile = "testing"
   }

.. important::

   You must also grant the secret to the coordinator application after
   deployment:

   .. code-block:: bash

      juju grant-secret fernet-key-secret airflow-coordinator

.. note::

   Set ``profile = "production"`` for production workloads. The ``testing``
   profile uses fewer resources and is intended for development.

Initialise and apply:

.. code-block:: bash

   terraform init
   terraform apply --var-file="terraform.tfvars"

This deploys all seven application charms (PostgreSQL, PgBouncer, Airflow Coordinator, Airflow API
Server, Airflow Scheduler, Airflow DAG Processor, Airflow Triggerer) and integrates the applications
automatically.

Monitor the deployment using ``juju status`` and wait until all units are ``active/idle``:


Deploy with the Kubernetes Executor (optional)
----------------------------------------------

To use the **Kubernetes Executor** (where tasks are run in short-lived individual Kubernetes pods),
set the ``executor`` variable and provide the executor configuration.

Create a ``deploy_with_kubernetes_executor.tfvars`` file:

.. code-block:: hcl

   model_uuid = "<your-model-uuid>"

   airflow_coordinator = {
     config = {
       fernet_key_secret = "secret:<secret-id>"
     }
   }

   postgresql = {
     profile = "testing"
   }

   executor = "kubernetes"

   airflow_kubernetes_executor = {
     config = {
       base_image = "ubuntu/airflow:3.1-24.04_edge"
       namespace  = "airflow-executor-workers"
     }
   }

.. important::

   The ``namespace`` specified in ``airflow_kubernetes_executor.config`` must
   already exist in the Kubernetes cluster. Use ``kubectl`` to create the namespace.


Then apply:

.. code-block:: bash

   terraform init
   terraform apply -var-file="deploy_with_kubernetes_executor.tfvars"

Customise charm parameters
--------------------------

Each charm accepts an object with optional overrides. For example, to change
the Airflow Coordinator's channel and the PostgreSQL unit count:

.. code-block:: hcl

   model_uuid = "<your-model-uuid>"

   airflow_coordinator = {
     channel = "3.1/stable"
   }

   postgresql = {
     units   = 3
     profile = "production"
   }

All configurable parameters follow this pattern:

.. code-block:: text

   charm_variable = {
     app_name = "custom-name"       # optional
     channel  = "3.1/edge"          # optional
     units    = 1                   # optional
     config   = {}                  # optional, map(string)
     revision = null                # optional, pin a specific revision
   }

See the
`Charmed Airflow Solutions`_
for the full variable reference.

Access the Airflow UI
---------------------

After deployment completes, access the web UI using any of the methods
described in the :doc:`tutorial </tutorial>`.

Tear down
---------

To remove the deployment:

.. code-block:: bash

   terraform destroy -var-file="terraform.tfvars"

This removes all charmed applications and integrations from the model.

Next steps
----------

* Explore the `Charmed Airflow Solutions`_ repository for more information.
* :doc:`Integrate with Traefik </how-to/integrate/traefik>` for external HTTP/HTTPS
  access.
* :doc:`Integrate with Git Integrator </how-to/integrate/git-integrator>` to sync
  DAGs from a Git repository.
