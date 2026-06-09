Integrate with the Canonical Identity Platform
=============================================================

The `Canonical Identity Platform`__ is a charmed solution designed to integrate with external Identity and Access Management (IAM) systems, while also offering the option for a fully built-in IAM deployment. Within the platform, `Hydra`__ acts as the OpenID Connect (OIDC) server, providing an ``oauth`` relation interface.

The Canonical Identity Platform can either serve as an identity broker to federate external identity providers, or support native identity management via Ory Kratos.

Prerequisites
---------------------------------------------

A running Charmed Airflow deployment. If you haven't deployed it yet, follow
the :doc:`tutorial </tutorial>` first.

Deploy the Canonical Identity Platform
---------------------------------------

Deploy the Canonical Identity Platform and configure your preferred identity provider. For detailed instructions on both supported integration methods, refer to the ``Set up the identity provider`` section of the `Canonical Identity Platform tutorial`__.

Integrate with Airflow Coordinator
-------------------------------------

Add the required integration between the ``hydra`` and ``airflow-coordinator-k8s`` charms:

.. code-block:: bash
    juju switch airflow
    juju consume <hydra_offer_url> hydra
    juju integrate hydra:oauth airflow-coordinator-k8s:ouath

This integration will enable authentication with the identity providers in the deployed Canonical Identity Platform.

Configure Authorization
---------------------------

Configure any of the following options to map identity provider groups to their predefined Airflow roles:

- ``idp_groups_for_admin``: Groups mapped to the Airflow ``Admin`` role
- ``idp_groups_for_op``: Groups mapped to the Airflow ``Op`` role
- ``idp_groups_for_user``: Groups mapped to the Airflow ``User`` role
- ``idp_groups_for_viewer``: Groups mapped to the Airflow ``Viewer`` role
- ``idp_groups_for_public``: Groups mapped to the Airflow ``Public`` role

.. code-block:: bash
    juju config airflow-coordinator-k8s idp_groups_for_admin="managers,product_managers"
    juju config airflow-coordinator-k8s idp_groups_for_user="data_engineers"

Additionally, configure ``enable_user_registration`` to toggle user self registration within Airflow:

.. code-block:: bash
    juju config airflow-coordinator-k8s enable_user_registration=true
