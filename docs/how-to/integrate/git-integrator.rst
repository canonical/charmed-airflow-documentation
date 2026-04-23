.. _integrate-git-integrator:

Integrate with Git Integrator
==============================

`Git Integrator`_ is a Juju charm for sharing Git repository connection details
with consumer charms, enabling Charmed Airflow to load DAGs directly from a Git
repository.

Prerequisites
-------------

A running Charmed Airflow deployment. If you haven't deployed it yet, follow
the :doc:`tutorial </tutorial>` first.

Deploy the Git Integrator charm
--------------------------------

.. code-block:: bash

   juju deploy git-integrator --channel 1.0/stable

Configure the charm
--------------------

Public repository
~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   juju config git-integrator \
     repository_url=https://github.com/my-org/my-dags.git \
     path=dags \
     tracking_ref=main

Private repository — HTTPS credentials
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a `Juju user secret`_ containing your personal access token. The
``juju add-secret`` command prints the secret URI — note it down:

.. code-block:: bash

   juju add-secret my-pat credentials-personal-access-token=<token>

Grant access and configure the charm:

.. code-block:: bash

   juju grant-secret my-pat git-integrator

   juju config git-integrator \
     repository_url=https://github.com/my-org/my-dags.git \
     authentication_method=credentials \
     credentials_username=<username> \
     credentials_personal_access_token_secret=<secret-uri>

Private repository — SSH
~~~~~~~~~~~~~~~~~~~~~~~~~

Create a Juju user secret containing your SSH private key. The
``juju add-secret`` command prints the secret URI — note it down:

.. code-block:: bash

   juju add-secret my-ssh-key ssh-private-key="$(cat ~/.ssh/id_ed25519)"

Grant access and configure the charm:

.. code-block:: bash

   juju grant-secret my-ssh-key git-integrator

   juju config git-integrator \
     repository_url=git@github.com:my-org/my-dags.git \
     authentication_method=ssh \
     ssh_private_key_secret=<secret-uri>

Optionally, enable strict host key checking:

.. code-block:: bash

   juju config git-integrator ssh_strict_host_key_checking=true

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
