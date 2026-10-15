.. _integrate-provider-configurator:

Integrate with the Airflow Provider Configurator
=================================================

The Airflow Provider Configurator charm (``airflow-provider-configurator``) lets you
configure Airflow providers — connection parameters, credentials, and feature flags —
from a Git repository and a Juju secret, instead of editing ``airflow.cfg`` by hand. It
reads non-sensitive configuration from an ``.ini`` file in a Git repository and
sensitive values from a Juju user secret, then shares the combined configuration with
``airflow-coordinator-k8s`` for merging into the deployed Airflow configuration.

Prerequisites
-------------

* A working Charmed Airflow deployment with ``airflow-coordinator-k8s`` already
  deployed and active.
* A Git repository containing an ``.ini`` file with your non-sensitive provider
  configuration.

----

Deploy the charms
-----------------

Deploy the Provider Configurator charm:

.. code-block:: bash

   juju deploy airflow-provider-configurator

.. note::

   Only one ``airflow-provider-configurator`` application is supported per Charmed
   Airflow deployment.

The charm accepts a single Git integrator application, so unless your provider
configuration lives in the same repository as your DAGs, deploy a dedicated one and
configure it with the repository holding the ``.ini`` file. See
:doc:`Integrate with Git Integrator </how-to/integrate/git-integrator>` for the
configuration options:

.. code-block:: bash

   juju deploy git-integrator provider-config-git --channel 1.0/stable

----

Prepare your provider configuration
-----------------------------------

The charm reads provider configuration from two sources:

* **Non-sensitive configuration** lives in an ``.ini`` file in your Git repository. Each
  section maps to an Airflow configuration section, and each option to a setting within
  it. For example:

  .. code-block:: ini

     [smtp]
     smtp_host = smtp.example.com
     smtp_port = 587
     smtp_starttls = True

* **Sensitive configuration** (passwords, tokens, keys) is supplied through a Juju user
  secret, so it never has to be committed to the repository. The secret holds a JSON
  string under the key ``airflow-provider-configurations``, mapping a provider name to a
  nested section/option map:

  .. code-block:: json

     {"smtp-provider": {"smtp": {"smtp_password": "my-secret-password"}}}

  The provider-name nesting (``smtp-provider`` above) is for your own authoring
  convenience; the charm flattens it and keeps the section/option/value structure.

----

Configure the charm
-------------------

Point the charm at the ``.ini`` file in your Git repository. The path is relative to the
root of the repository, or to the subdirectory advertised by the Git integrator if one
is set. This option is **required**:

.. code-block:: bash

   juju config airflow-provider-configurator \
     airflow_provider_configurations_file_path=<path/to/config.ini>

To supply sensitive values, create a Juju secret, grant it to the charm, and reference
it:

.. code-block:: bash

   juju add-secret airflow-provider-secret \
     airflow-provider-configurations='{"smtp-provider": {"smtp": {"smtp_password": "my-secret-password"}}}'

   juju grant-secret airflow-provider-secret airflow-provider-configurator

   juju config airflow-provider-configurator \
     airflow_provider_configurations_secret=secret:<secret-id>

.. note::

   Replace ``<secret-id>`` with the secret ID returned by ``juju add-secret``. You can
   find it again with ``juju list-secrets``. The secret key is hyphenated
   (``airflow-provider-configurations``) because Juju does not allow underscores in
   secret keys.

Optionally, change how often the charm polls the Git repository for changes (the default
is ``60s``):

.. code-block:: bash

   juju config airflow-provider-configurator airflow_provider_configurations_sync_period=120s

----

Integrate with the Charmed Airflow components
---------------------------------------------

1. Integrate with the Git integrator so the charm can fetch your configuration
   repository:

   .. code-block:: bash

      juju integrate airflow-provider-configurator:remote-airflow-provider-configurations \
        provider-config-git:git

2. Integrate with the coordinator so the provider configuration is merged into the
   deployed Airflow configuration:

   .. code-block:: bash

      juju integrate airflow-provider-configurator:airflow-provider-configuration \
        airflow-coordinator-k8s:airflow-provider-configuration

Wait for all units to return to ``active/idle``:

.. code-block:: bash

   juju status --watch 5s

----

Verify the configuration
------------------------

The Provider Configurator charm stays ``blocked`` until it can publish a configuration.
Check its status message with ``juju status`` if it does not reach ``active/idle``. The
most common causes are a missing ``airflow_provider_configurations_file_path``, no Git
integrator relation, a file that cannot be found at the configured path, and a Juju
secret that cannot be read.

Once the charm is ``active``, inspect its logs to confirm what was published:

.. code-block:: bash

   juju debug-log --include airflow-provider-configurator

.. note::

   Some provider configuration is dropped rather than applied, without blocking the
   deployment: settings that providers are never allowed to set, such as those
   belonging to Airflow's core machinery, and settings the coordinator already manages
   itself. The unit that drops a setting reports this in its status message and names
   each dropped setting in its logs.

----

Force an immediate sync
-----------------------

By default the charm republishes provider configuration on the configured poll interval.
To apply a change immediately — for example after updating the ``.ini`` file in Git — run
the ``sync-now`` action:

.. code-block:: bash

   juju run airflow-provider-configurator/0 sync-now

----

Remove the integration
----------------------

.. code-block:: bash

   juju remove-relation airflow-provider-configurator:airflow-provider-configuration \
     airflow-coordinator-k8s:airflow-provider-configuration