Logging and monitoring
======================

Charmed Airflow emits structured JSON security events from its Juju charms. These events are written to the normal charm log stream. This page describes the events currently implemented and their operational limits; it does not imply that every event in the Canonical SSDLC vocabulary is available.

Log schema
-----------

Each security event is a single-line JSON object with these fields:

.. code-block:: json

   {
     "owasp_event": {
       "datetime": "2026-11-13T16:04:45.317+00:00",
       "type": "security",
       "appid": "airflow.api-server",
       "event": "sys_restart:juju-charm",
       "level": "WARN",
       "actor": "juju-charm",
       "target": "airflow",
       "action": "restart",
       "outcome": "success",
       "component": "api-server",
       "description": "Airflow api-server: Service restarted by the charm.",
       "data": {"trigger": "configuration_changed"}
     }
   }

The logger emits the OWASP event under the ``owasp_event`` key in an OTLP-compatible
JSON record. ``datetime`` is UTC with an explicit offset. ``appid`` identifies the
product component, while ``event`` contains the event type and schema parameters.
The core context fields include ``actor``, ``action``, ``target``, and ``outcome``;
the ``data`` object carries supplementary product-specific context. Secret values,
API keys, JWT signing keys, OAuth client secrets, and bearer tokens are never logged.

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

Where remaining events belong
------------------------------

Charm hooks do not observe individual user requests, so request-level events belong
at the component that makes the authentication or authorization decision:

* ``authn_login_success`` and ``authn_login_fail`` belong in the Airflow API/auth
  manager for Airflow-validated credentials. If Hydra or another identity provider
  performs the authentication, emit the event there instead.
* Token creation, revocation, reuse, and deletion belong in Airflow or Hydra,
  depending on which service issues, validates, or manages the token. The Airflow
  ``secret-key`` and ``jwt-secret`` are signing keys, not access tokens; their
  observed rotation is logged by the coordinator as ``authz_admin``.
* ``authz_fail`` and administrative actions performed through the Airflow UI or API
  belong in the Airflow authorization layer. The coordinator emits ``authz_admin``
  for the configuration and credential changes it can observe during reconciliation.
* Password changes and account lockouts belong in the identity provider when it
  owns those controls. Airflow deployments that enable FAB OAuth user registration
  should also audit user lifecycle events in Airflow or the identity provider.

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
