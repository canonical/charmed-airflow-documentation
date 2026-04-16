Integrate with the Charmed Airflow Kubernetes Executor
=============================================================

The Airflow Kubernetes executor charm (``airflow-kubernetes-executor-k8s``) enables a Charmed Airflow
deployment to run DAG tasks as individual Pods in a Kubernetes cluster. This provides strong
workload isolation, elastic scaling, and fine-grained resource control per task. This guide walks
you through deploying and configuring the Kubernetes executor charm, integrating it with your existing
Charmed Airflow solution, and verifying that tasks are being scheduled as Kubernetes Pods.

**Prerequisites:** A working Charmed Airflow deployment with ``airflow-coordinator-k8s`` already
deployed and active.

----

Deploy the Kubernetes Executor charm
---------------------------------------------

You can deploy the charm using either ``juju deploy`` directly or a Terraform module.

Option A: Deploy with Juju
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The charm requires Juju trust to create and manage Kubernetes resources (ConfigMaps and Secrets)
in the cluster on behalf of Airflow:

.. code-block:: bash

   juju deploy airflow-kubernetes-executor-k8s --trust

Option B: Deploy with the Terraform module
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you manage your infrastructure with Terraform, use the module provided in the
`airflow-kubernetes-executor-k8s-operator repository
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

----

Configure the charm
-----------------------------

Before the charm becomes active, you must supply the required configuration options. Set them
using ``juju config``:

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

The executor charm communicates with ``airflow-coordinator-k8s`` over two integration endpoints.
Both are required for the executor to function:

.. code-block:: bash

   juju integrate airflow-kubernetes-executor-k8s:airflow-config airflow-coordinator-k8s
   juju integrate airflow-kubernetes-executor-k8s:airflow-executor-config airflow-coordinator-k8s

The ``airflow-config`` relation shares Airflow connection details and credentials with the
executor. The ``airflow-executor-config`` relation delivers the executor-specific pod template
and Kubernetes configuration back to the coordinator, which Airflow uses when scheduling tasks.

Wait for the units to settle. You can check status with:

.. code-block:: bash

   juju status --watch 5s

All units should reach ``active/idle`` before proceeding.

Because the Airflow scheduler process needs to talk to the Kubernetes API, the charm has to be trusted as well:

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

In your DAG, use the ``executor_config`` parameter with a ``k8s.V1Pod`` specification:

.. code-block:: python

   from kubernetes.client import models as k8s
   from airflow.decorators import task

   @task(
       executor_config={
           "pod_override": k8s.V1Pod(
               spec=k8s.V1PodSpec(
                   containers=[
                       k8s.V1Container(
                           name="base",
                           resources=k8s.V1ResourceRequirements(
                               requests={"cpu": "500m", "memory": "512Mi"},
                               limits={"cpu": "1", "memory": "1Gi"},
                           ),
                       )
                   ]
               )
           )
       }
   )
   def my_resource_intensive_task():
       ...

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
