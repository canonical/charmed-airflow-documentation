.. _deploy-airflow-tutorial:

Tutorial: Deploy Airflow with Juju
==================================

In this tutorial you will deploy a fully functional `Apache Airflow`_ cluster on
Kubernetes using `Juju`_ charms. By the end you will have a running Airflow
instance with an API Server UI you can access from your browser.

What you'll need
----------------

* Ubuntu 24.04 (or later).
* A machine with at least a **4-core CPU**, **8 GB RAM**, and **30 GB** of free
  disk space.
* A K8s cluster (v1.32+) with a Juju controller bootstrapped on it.

  See `Set up your deployment
  <https://documentation.ubuntu.com/juju/3.6/howto/manage-your-juju-deployment/set-up-your-juju-deployment-local-testing-and-development/>`_
  for a step-by-step guide.

What you'll do
--------------

#. Install and configure dependencies.
#. Deploy the Airflow charms and their database.
#. Integrate the charms.
#. Access the Airflow API Server UI.
#. Tear down the deployment.

Install and configure dependencies
-----------------------------------

Install k8s and Juju via concierge
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Install concierge snap if you haven't already:

.. code-block:: bash

   sudo snap install concierge --classic
   sudo -E concierge prepare -p k8s
   mkdir -p ~/.kube
   sudo k8s kubectl config view --raw > ~/.kube/config

Create a Juju model
~~~~~~~~~~~~~~~~~~~

Create a dedicated model for the Airflow deployment:

.. code-block:: bash

   juju add-model airflow

.. _deploy-charms:

Deploy the charms
-----------------

The Charmed Airflow solution consists of several charms that integrate.
This section walks you through deploying each one and integrating them.

Deploy PostgreSQL and PgBouncer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Airflow requires a PostgreSQL database to store metadata. Deploy PostgreSQL
and PgBouncer (connection pooler for PostgreSQL) first:

.. code-block:: bash

   juju deploy postgresql-k8s --channel=14/stable --trust
   juju deploy pgbouncer-k8s --channel=1/stable --trust

.. note::

   **PgBouncer is optional.** PgBouncer is a connection pooler, reducing
   the number of direct connections to PostgreSQL. It is recommended for
   production workloads but not required. If you skip PgBouncer, integrate the
   Airflow Coordinator charm directly with PostgreSQL instead of PgBouncer in the
   integration step below.

Deploy the Airflow Coordinator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Airflow Coordinator is the central configuration hub. It provisions required PostgreSQL database
schemas and generates and distributes the Airflow configuration.

.. code-block:: bash

   juju deploy airflow-coordinator-k8s --channel=3.1/edge

Deploy the core Airflow components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Deploy the four core Airflow workload charms:

.. code-block:: bash

   juju deploy airflow-api-server-k8s --channel=3.1/edge
   juju deploy airflow-scheduler-k8s --channel=3.1/edge
   juju deploy airflow-dag-processor-k8s --channel=3.1/edge
   juju deploy airflow-triggerer-k8s --channel=3.1/edge

These charms map to the Airflow components:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Charm
     - Purpose
   * - ``airflow-api-server-k8s``
     - Serves the Airflow dashboard UI.
   * - ``airflow-scheduler-k8s``
     - Schedules and triggers DAG runs.
   * - ``airflow-dag-processor-k8s``
     - Parses DAG files and updates the metadata database.
   * - ``airflow-triggerer-k8s``
     - Handles deferred (asynchronous) tasks.

.. _integrate-charms:

Integrate the charms
--------------------

Juju *integrations* (also called *relations*) connect the charms so they can
exchange configuration, endpoints, and credentials automatically.

If you deployed the PgBouncer charm, integrate it with the PostgreSQL charm first:

.. code-block:: bash

   juju integrate pgbouncer-k8s:backend-database postgresql-k8s:database

Integrate the Airflow Coordinator with the database
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
If you deployed PgBouncer (recommended):

.. code-block:: bash

   juju integrate airflow-coordinator-k8s:postgres pgbouncer-k8s:database

Or, if you chose to skip PgBouncer:

.. code-block:: bash

   juju integrate airflow-coordinator-k8s:postgres postgresql-k8s:database

Integrate the API server with the Airflow Coordinator charm
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The API server integrates with the Airflow Coordinator to share its host and port 
information. This allows the Airflow Coordinator to include these details in the centralized 
``airflow.cfg``, which it then distributes across the entire cluster.

.. code-block:: bash

   juju integrate airflow-coordinator-k8s:airflow-api-server \
     airflow-api-server-k8s:airflow-api-server

Integrate all core charms with the Airflow Coordinator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Every core charm receives its Airflow configuration from the Airflow Coordinator
through the ``airflow-coordinator`` integration:

.. code-block:: bash

   juju integrate airflow-coordinator-k8s:airflow-coordinator \
     airflow-api-server-k8s:airflow-coordinator

   juju integrate airflow-coordinator-k8s:airflow-coordinator \
     airflow-scheduler-k8s:airflow-coordinator

   juju integrate airflow-coordinator-k8s:airflow-coordinator \
     airflow-dag-processor-k8s:airflow-coordinator

   juju integrate airflow-coordinator-k8s:airflow-coordinator \
     airflow-triggerer-k8s:airflow-coordinator

Wait for the deployment
~~~~~~~~~~~~~~~~~~~~~~~

Monitor the status of your deployment:

.. code-block:: bash

   juju status --watch 5s

Wait until all applications and units show ``active/idle``. This can take
several minutes while the coordinator runs the initial database migration and
distributes the configuration.

.. code-block:: text

   Model    Controller     Cloud/Region  Version  SLA          Timestamp
   airflow  concierge-k8s  k8s           3.6.21   unsupported  17:32:31-04:00
   
   App                        Version  Status  Scale  Charm                      Channel    Rev  Address         Exposed  Message
   airflow-api-server-k8s              active      1  airflow-api-server-k8s     3.1/edge     9  10.152.183.39   no       
   airflow-coordinator-k8s             active      1  airflow-coordinator-k8s    3.1/edge    24  10.152.183.188  no       
   airflow-dag-processor-k8s           active      1  airflow-dag-processor-k8s  3.1/edge     7  10.152.183.227  no       
   airflow-scheduler-k8s               active      1  airflow-scheduler-k8s      3.1/edge    10  10.152.183.98   no       
   airflow-triggerer-k8s               active      1  airflow-triggerer-k8s      3.1/edge     7  10.152.183.190  no       
   pgbouncer-k8s              1.21.0   active      1  pgbouncer-k8s              1/stable   520  10.152.183.121  no       
   postgresql-k8s             14.20    active      1  postgresql-k8s             14/stable  774  10.152.183.69   no       
   
   Unit                          Workload  Agent  Address     Ports  Message
   airflow-api-server-k8s/0*     active    idle   10.1.0.242         
   airflow-coordinator-k8s/0*    active    idle   10.1.0.107         
   airflow-dag-processor-k8s/0*  active    idle   10.1.0.40          
   airflow-scheduler-k8s/0*      active    idle   10.1.0.43          
   airflow-triggerer-k8s/0*      active    idle   10.1.0.115         
   pgbouncer-k8s/0*              active    idle   10.1.0.189         
   postgresql-k8s/0*             active    idle   10.1.0.2           Primary

Verify the cluster health
~~~~~~~~~~~~~~~~~~~~~~~~~

You can check the health of all Airflow components through the REST API:

.. code-block:: bash

   curl -s http://<pod-ip>:8080/api/v2/monitor/health | jq

Expected output:

.. code-block:: json

   {
     "metadatabase": {
       "status": "healthy"
     },
     "scheduler": {
       "status": "healthy",
       "latest_scheduler_heartbeat": "...timestamp..."
     },
     "triggerer": {
       "status": "healthy",
       "latest_triggerer_heartbeat": "...timestamp..."
     },
     "dag_processor": {
       "status": "healthy",
       "latest_dag_processor_heartbeat": "...timestamp..."
     }
   }

.. _access-ui:

Access the Airflow UI
---------------------

The Airflow web UI is served by the API server charm on port **8080**. There
are several ways to reach it depending on your environment.

Option A: Port-forward from your local machine or remote VM
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can forward traffic from your local machine to the API server pod. By default, this maps the pod's port **8080** to your local port **8080**. 

* **Local clusters:** Standard port-forwarding allows access via ``http://localhost:8080``.
* **Remote/VM clusters (e.g., Multipass):** To reach the UI from your host machine, you must bind the command to all network interfaces using the ``--address 0.0.0.0`` flag. 
You can then access the UI using the VM's IP address.

For the exact command syntax and advanced configuration flags, refer to the official `kubectl port-forward documentation <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/>`_.

Option B: Use the Kubernetes service directly
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If your machine can reach the Kubernetes cluster network directly (for example,
when running on the same host as the cluster), query the pod IP:

.. code-block:: bash

   kubectl get pod -n airflow airflow-api-server-k8s-0 \
     -o jsonpath='{.status.podIP}'

Then open ``http://<pod-ip>:8080`` in your browser.

.. image:: images/airflow-login-ui.png
   :alt: Airflow API Server UI Login
   :align: center

Retrieve the credentials from the API server pod:

.. code-block:: bash

   kubectl exec -it airflow-api-server-k8s-0 -c airflow-api-server \
     -n airflow -- cat /opt/airflow/simple_auth_manager_passwords.json.generated

Alternatively, search the API server logs:

.. code-block:: bash

   kubectl logs airflow-api-server-k8s-0 -c airflow-api-server -n airflow | grep -i password

.. image:: images/airflow-ui.png
   :alt: Airflow API Server UI
   :align: center

Verify the cluster health
~~~~~~~~~~~~~~~~~~~~~~~~~

With port-forwarding active, confirm all Airflow components are healthy:

.. code-block:: bash

   curl -s http://localhost:8080/api/v2/monitor/health | jq

Expected output:

.. code-block:: json

   {
     "metadatabase": {
       "status": "healthy"
     },
     "scheduler": {
       "status": "healthy",
       "latest_scheduler_heartbeat": "2026-04-16T..."
     },
     "triggerer": {
       "status": "healthy",
       "latest_triggerer_heartbeat": "2026-04-16T..."
     },
     "dag_processor": {
       "status": "healthy",
       "latest_dag_processor_heartbeat": "2026-04-16T..."
     }
   }

.. _understand-architecture:

Understanding how the charms work together
------------------------------------------

The following diagram shows how the charms interact:

.. code-block:: text

                 ┌──────────────┐
                 │ PostgreSQL   │
                 └──────┬───────┘
                        │ database
                 ┌──────┴───────┐
                 │  PgBouncer   │  (optional connection pooler)
                 └──────┬───────┘
                        │ postgres
   ┌────────────────────┴──────────────────────┐
   │   Airflow Coordinator                     │
   │  • Provisions PostgreSQL db schema        │
   │  • Unifies and distributes ``airflow.cfg``│
   └──┬────────┬───────────┬─────────┬─────────┘
      │        │           │         │  airflow-coordinator
   ┌──┴───┐ ┌──┴──────┐ ┌──┴──────┐ ┌┴────────┐
   │ API  │ │Scheduler│ │ DAG     │ │Triggerer│
   │Server│ └─────────┘ │Processor│ └─────────┘
   └──────┘             └─────────┘ 

**Airflow Coordinator** is the brain of the deployment:

* On startup, it connects to PostgreSQL and runs ``airflow db migrate`` to
  initialise the metadata database.
* It renders a unified ``airflow.cfg`` from its Jinja2 template and pushes it
  to every core charm via the ``airflow-coordinator`` integration.

**Core charms** (API server, scheduler, DAG processor, triggerer) all follow
the same pattern:

* They wait for the Airflow Coordinator integration.
* Once they receive the configuration, they write ``/opt/airflow/airflow.cfg``
  and start their respective Airflow service.
* If the Airflow Coordinator integration is removed, they stop the service and enter
  a blocked state.

**API server** has an additional integration with the Airflow Coordinator
(``airflow-api-server``) through which it sends its hostname and port. The
Airflow Coordinator uses the provided hostname and port to set ``base_url`` 
in the Airflow configuration.

.. _tear-down:

Tear down the deployment
------------------------

To remove everything deployed in this tutorial:

.. code-block:: bash

   juju destroy-model airflow --destroy-storage --no-prompt

Next steps
----------

* :doc:`Deploy with Terraform </how-to/deploy-with-terraform>` for an automated,
  reproducible deployment.
* :doc:`Integrate with Traefik </how-to/integrate/traefik>` for external HTTP/HTTPS
  access.
* :doc:`Integrate with Git Integrator </how-to/integrate/git-integrator>` to sync
  DAGs from a Git repository.
