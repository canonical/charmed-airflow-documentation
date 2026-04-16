Integrate with the Charmed Airflow Kubernetes Executor
=============================================================

The Airflow Kubernetes executor charm (``airflow-kubernetes-executor-k8s``) enables a Charmed Airflow
deployment to run DAG tasks as individual Pods in a Kubernetes cluster. This guide walks
you through deploying and configuring the Airflow Kubernetes executor charm, integrating it with your existing
Charmed Airflow solution, and verifying that tasks are being scheduled as Kubernetes Pods.

Prerequisites
---------------------------------------------

A working Charmed Airflow deployment with ``airflow-coordinator-k8s`` already
deployed and active.

----

Deploy the Kubernetes Executor charm
---------------------------------------------

You can deploy the charm using either ``juju deploy`` directly or a Terraform module.

Option A: Deploy with Juju
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Deploy with `--trust` as the charm creates and manages Kubernetes resources:

.. code-block:: bash

   juju deploy airflow-kubernetes-executor-k8s --trust

Option B: Deploy with the Terraform module
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you manage your infrastructure with Terraform, use the module provided `here 
<https://github.com/canonical/airflow-kubernetes-executor-k8s-operator/tree/track/3.1/terraform>`_.

Add the module to your Terraform configuration:

.. code-block:: hcl

   module "airflow_kubernetes_executor" {
     source = "git::https://github.com/canonical/airflow-kubernetes-executor-k8s-operator//terraform?ref=track/3.1"

     # Pass required configuration as module variables.
     # Refer to the module's variables.tf for all available inputs.
   }

Then apply your changes:

.. code-block:: bash

   terraform init
   terraform apply

.. note::
   If deploying with the `Charmed Airflow solution module <https://github.com/canonical/charmed-airflow-solutions/tree/track/3.1`_, you just need to set the ``executor``
   input variable to ``kubernetes`` and add the required configurations.

----

Configure the charm
-----------------------------

Supply the required configuration options.

.. code-block:: bash

   juju config airflow-kubernetes-executor-k8s \
     base_image=<your-airflow-oci-image> \
     namespace=<target-kubernetes-namespace>

Optionally, customise the base name for worker Pods:

.. code-block:: bash

   juju config airflow-kubernetes-executor-k8s pod_name=my-airflow-worker

The full list of configuration options can be found in the `charm configuration page <https://charmhub.io/airflow-kubernetes-executor-k8s/configurations>`_.

----

Integrate with the Charmed Airflow components
------------------------------------------------

1. Add the required integrations with the ``airflow-coordinator-k8s`` charm:

.. code-block:: bash

   juju integrate airflow-kubernetes-executor-k8s:airflow-config airflow-coordinator-k8s
   juju integrate airflow-kubernetes-executor-k8s:airflow-executor-config airflow-coordinator-k8s

All units should reach ``active/idle`` before proceeding.

2. The Airflow scheduler charm has to be trusted as well as its process interacts with the Kubernetes API:

.. code-block:: bash

   juju trust airflow-scheduler-k8s --scope=cluster

----

Verify workers are running
------------------------------------

Once the integrations are in place, Airflow will schedule each task in your DAGs as an individual
Kubernetes Pod. To observe this in action:

1. **Trigger a DAG run** via the Airflow UI.

2. **Watch for worker Pods** appearing in the configured namespace if you have access to the cluster:

   .. code-block:: bash

      kubectl get pods -n <target-kubernetes-namespace> --watch

   You should see Pods named after the ``pod_name`` configuration value (e.g. ``airflow-worker-*``)
   being created as tasks are queued, and terminating once they complete.

3. **Inspect a worker Pod** for live logs during execution:

   .. code-block:: bash

      kubectl logs -n <target-kubernetes-namespace> <pod-name> -f

4. **Check task logs in the Airflow UI.** Logs are streamed from the Pod during execution and
   visible in the task log view.

.. note::
   Task logs are currently ephemeral — once a worker Pod terminates, its logs are no longer
   accessible from the cluster. Remote logging support is on the roadmap and will be enabled in a
   future release to provide a durable logging solution.

----

Customising worker Pods with ``pod_override``
----------------------------------------------

Pod customisation can be done through Airflow's ``pod_override``
feature. This lets you override resource requests, environment variables,
tolerations, and any other Kubernetes Pod spec field on a per-task basis.

For the full reference on ``pod_override``, see the `upstream Airflow Kubernetes Executor
documentation
<https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html#pod-override>`_.

----

Reference
----------

- `airflow-kubernetes-executor-k8s-operator on GitHub <https://github.com/canonical/airflow-kubernetes-executor-k8s-operator>`_
- `Terraform module <https://github.com/canonical/airflow-kubernetes-executor-k8s-operator/tree/track/3.1/terraform>`_
- `Airflow Kubernetes Executor — upstream documentation <https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html>`_
- `Pod override reference <https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html#pod-override>`_
