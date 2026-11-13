Logging and monitoring
======================

Charmed Airflow emits structured JSON security events from its Juju charms. These events are written to the normal charm log stream. This page describes the events currently implemented and their operational limits; it does not imply that every event in the Canonical SSDLC vocabulary is available.

Log schema
-----------

Each security event is a single-line JSON object with these fields:

.. code-block:: json

   {
     "datetime": "2026-11-13T16:04:45.317+00:00",
     "appid": "airflow.api-server",
     "event": "sys_restart:juju-charm",
     "level": "WARN",
     "actor": "juju-charm",
     "target": "airflow",
     "description": "Airflow api-server: Service restarted by the charm.",
     "data": {"trigger": "configuration_changed"}
   }

``datetime`` is UTC with an explicit offset. The ``appid`` identifies the product component. ``event`` contains the event type and any schema parameters; event-specific context is carried in ``data``. Secret values, API keys, JWT signing keys, OAuth client secrets, and bearer tokens are not logged.

Event inventory
---------------

System events are emitted by the API server, scheduler, DAG processor, and triggerer charms.
Their app IDs are ``airflow.api-server``, ``airflow.scheduler``,
``airflow.dag-processor``, and ``airflow.triggerer``.

* ``sys_startup:pebble`` (``WARN``): reconcile observes a stopped service become
  running after Pebble applies the charm's service plan.
* ``sys_shutdown:juju-charm`` (``WARN``): the charm successfully stops a running
  service because its coordinator relation is missing.
* ``sys_restart:juju-charm`` (``WARN``): the charm successfully restarts a service
  after configuration changes.
* ``sys_crash:PebbleServiceError`` (``WARN``): a failed Pebble operation is
  followed by Pebble reporting the service as ``ERROR``.

The coordinator app ID is ``airflow.coordinator``. After successful configuration
publication, it emits ``authz_admin`` for changed IdP role mappings,
user-registration policy, or OAuth activation. It also emits ``authz_admin`` actions
``rotate_signing_key`` and ``update_client_secret`` when an existing Juju secret
revision changes. These events have ``actor: unattributed``; correlate them with
Juju audit logs for human attribution. First reconciliation establishes an audit
baseline and does not emit a change event.

Not yet emitted
---------------

The charm hooks do not observe individual Airflow authentication or authorization decisions. ``authn_login_success``, ``authn_login_fail``, token issuance/revocation/reuse, and ``authz_fail`` require instrumentation in the Airflow authentication/authorization runtime or Hydra, where those decisions occur.

Event-category applicability
----------------------------

* ``[USER]`` is **Not Applicable to the charm-managed layer**. The coordinator,
  API server charm, scheduler, DAG processor, and triggerer charms do not implement
  or expose user-account creation and modification operations; their responsibility
  is deploying services and reconciling configuration. Consequently, these charms
  do not emit ``user_created`` or ``user_updated``.
* This is not a blanket claim that user lifecycle events are inapplicable to every
  Charmed Airflow deployment. OAuth user registration can be enabled in the Airflow
  FAB configuration. When enabled, Airflow or its identity provider owns account
  creation and any subsequent user management; those events must be audited at
  that runtime or identity-provider boundary.
* ``[AUTHN]`` is applicable to the Airflow product. The charm layer does not validate
  individual API/JWT or OAuth login attempts, so login and token lifecycle events
  must be emitted by Airflow or Hydra.
* ``[AUTHZ]`` is applicable to the Airflow product. The coordinator emits
  ``authz_admin`` for observed security-policy and credential-rotation changes;
  request-level denials (``authz_fail``) and user-initiated administrative actions
  require instrumentation in the Airflow runtime.

Operational boundaries
----------------------

``[SYS]`` events describe transitions observed or initiated by a charm through Pebble. They are not a complete record of Kubernetes pod eviction, node failure, OOM termination, or every process restart. 

The charm log stream is not automatically a SIEM destination. Operators must
configure log forwarding and JSON field extraction for their environment. The
implementation repository contains experimental `Sigma <https://sigmahq.io/>`_
rules; validate and tune converted rules for the target SIEM before enabling
alerts. See the `Sigma consumer guide
<https://github.com/canonical/airflow-core-operators/tree/track/3.1/docs/security/sigma>`_.
