Integrate with Git Integrator
==============================

`Git Integrator`_ is a Juju charm that distributes Git repository connection details
to connected charms. Integrating it with Charmed Airflow lets the cluster pull DAGs
directly from a Git repository.

Prerequisites
-------------

* Charmed Airflow deployed and healthy. See :doc:`/tutorial` for instructions.
* A Git repository containing your DAG files.
* For private repositories: a `personal access token`_ (HTTPS) or an SSH key pair.

Deploy Git Integrator
---------------------

.. code-block:: bash

   juju deploy git-integrator --channel 1.0/stable

Configure Git Integrator
-------------------------

All configuration is set with :code:`juju config`. The only required option is
:code:`repository_url`. The optional :code:`path` and :code:`tracking_ref` options
narrow the scope to a sub-directory and a specific branch or tag respectively.

Public repository
~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   juju config git-integrator \
     repository_url=https://github.com/my-org/my-dags.git \
     path=dags \
     tracking_ref=main

Private repository — HTTPS credentials
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Store the personal access token in a `Juju user secret`_, then pass its URI to
the charm:

.. code-block:: bash

   # Create the secret
   juju add-secret my-pat credentials-personal-access-token=<token>

   # Grant the secret to the charm
   juju grant-secret my-pat git-integrator

   # Retrieve the secret URI
   SECRET_URI=$(juju show-secret my-pat --format=json \
     | python3 -c "import sys,json; print(list(json.load(sys.stdin).keys())[0])")

   # Configure the charm
   juju config git-integrator \
     repository_url=https://github.com/my-org/my-dags.git \
     authentication_method=credentials \
     credentials_username=<username> \
     credentials_personal_access_token_secret="$SECRET_URI"

Private repository — SSH
~~~~~~~~~~~~~~~~~~~~~~~~~

Store the private key in a Juju user secret, then pass its URI to the charm:

.. code-block:: bash

   # Create the secret (value must be the raw PEM content)
   juju add-secret my-ssh-key ssh-private-key="$(cat ~/.ssh/id_ed25519)"

   # Grant the secret to the charm
   juju grant-secret my-ssh-key git-integrator

   # Retrieve the secret URI
   SECRET_URI=$(juju show-secret my-ssh-key --format=json \
     | python3 -c "import sys,json; print(list(json.load(sys.stdin).keys())[0])")

   # Configure the charm
   juju config git-integrator \
     repository_url=git@github.com:my-org/my-dags.git \
     authentication_method=ssh \
     ssh_private_key_secret="$SECRET_URI"

To enforce strict host key checking, also set:

.. code-block:: bash

   juju config git-integrator ssh_strict_host_key_checking=true

Integrate with Charmed Airflow
-------------------------------

Once the charm is active, create the relation:

.. code-block:: bash

   juju integrate git-integrator:git airflow-coordinator-k8s:git

Wait for all units to re-settle:

.. code-block:: bash

   juju status --watch 5s

All units should return to ``active/idle``.

Remove the integration
-----------------------

To stop syncing DAGs from the Git repository, remove the relation:

.. code-block:: bash

   juju remove-relation git-integrator:git airflow-coordinator-k8s:git

.. LINKS
.. _Git Integrator: https://charmhub.io/git-integrator
.. _personal access token: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
.. _Juju user secret: https://documentation.ubuntu.com/juju/en/latest/reference/secret/

