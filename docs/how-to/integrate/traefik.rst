.. _integrate-traefik:

Integrate with Traefik
======================

`Traefik K8s`_ provides reverse proxy and load balancer functionalities,
and when integrated with the Charmed Airflow API server,
it gives you an external URL that can be used for reaching the Airflow API server and UI over the internet.

This guide covers:

* Deploying Traefik and connecting it to the API server.
* Enable TLS termination at ingress.

Prerequisites
-------------

* A running Charmed Airflow deployment. If you haven't deployed it yet, follow
  the :doc:`Tutorial </tutorial/index>` or :doc:`Deploy with Terraform </how-to/deploy-with-terraform>` first.

* A Kubernetes cluster with a load balancer configured. Refer to your kubernetes distribution's documentation to enable
  load balancer support (for example, the `Canonical K8s load balancer <https://documentation.ubuntu.com/canonical-kubernetes/latest/snap/howto/networking/default-loadbalancer/>`_).


Deploy and integrate Traefik
-----------------------------

Deploy the ``traefik-k8s`` charm with trust (it needs Kubernetes RBAC
permissions to manage load balancer resources):

.. code-block:: bash

   juju deploy traefik-k8s --channel=latest/stable --trust

Wait for it to become active using ``juju status``.

Then integrate it with the Airflow API server charm:

.. code-block:: bash

   juju integrate airflow-api-server-k8s:ingress traefik-k8s:ingress

After a few seconds, all units should return to ``active/idle``. Traefik
assigns a URL to the Airflow API server, which you can retrieve with:

.. code-block:: bash

   juju run traefik-k8s/0 show-proxied-endpoints

The output looks like:

.. code-block:: json

   {
     "proxied-endpoints": "{\"airflow-api-server-k8s\": {\"url\": \"http://<LoadBalancerIP>/airflow-airflow-api-server-k8s\"}}"
   }

The inner ``url`` value is the external URL for the Airflow API Server UI.

.. note::

   Replace ``<LoadBalancerIP>`` with the load balancer IP assigned to your Traefik instance.
   Note that while this guide uses the default routing mode, both ``subdomain`` and ``path`` based routing are supported.


See `Traefik K8s | Configurations`_ for all
available Traefik configuration options.

Enable TLS termination at ingress
--------------------------------------

For HTTPS access, integrate Charmed Traefik with a certificate provider charm.
Traefik handles TLS termination — the API server itself always
communicates over plain HTTP inside the cluster. Please refer to `Security with X.509 TLS certificates`_
to understand the different certificate use cases and choose the solution that best fits yours.

Deploy the certificates provider charm of your choice:

.. code-block:: bash

   juju deploy <tls-certificate-provider>

Integrate it with Charmed Traefik or Traefik Charm:

.. code-block:: bash

   juju integrate traefik-k8s <tls-certificate-provider>

Once ready, Traefik will serve HTTPS. Retrieve the new URL:

.. code-block:: bash

   juju run traefik-k8s/0 show-proxied-endpoints

The URL will now start with ``https://``:

.. code-block:: json

   {
     "proxied-endpoints": "{\"airflow-api-server-k8s\": {\"url\": \"https://<Load Balancer IP>/airflow-airflow-api-server-k8s\"}}"
   }


Further reading
----------------

* `Traefik K8s`_
* `Traefik K8s | Configurations`_
* `Ingress`_
* `Traefik documentation`_