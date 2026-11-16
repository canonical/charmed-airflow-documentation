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

Events to be logged
-------------------

Each event below names the component responsible for emitting it, or explains why
it is not emitted by the charm-managed layer. Runtime-owned events must be
instrumented in Airflow or the identity provider before they can be considered
covered by this product.

The status column describes current **charm-layer** coverage. A red cross means
the event is not emitted by a charm; it does not necessarily mean the event is
inapplicable to the Airflow product.

.. list-table:: Event coverage overview
   :header-rows: 1
   :widths: 12 28 60

   * - Category
     - Event
     - Charm-layer coverage / owner
   * - [AUTHN]
     - Successful Login
     - ❌ Kratos validates user credentials; the Identity Platform owns the login event.
   * - [AUTHN]
     - Failed Login
     - ❌ Kratos rejects user credentials; the Identity Platform owns the failure event.
   * - [AUTHN]
     - Account Lockout
     - ❌ Not implemented by Airflow charms. Identity Platform owns lockout policy.
   * - [AUTHN]
     - Password Changed
     - ❌ Kratos/Identity Platform owns user password changes. Machine signing-key/client-secret rotations use [AUTHZ] administrative events instead.
   * - [AUTHN]
     - Token Created
     - ❌ Hydra owns OAuth access-token issuance. Airflow API JWTs, if issued by Airflow, belong to its API auth manager. Signing-key creation is not token creation.
   * - [AUTHN]
     - Token Revoked
     - ❌ Hydra owns OAuth token revocation; Airflow's API auth manager owns any Airflow-issued API JWTs.
   * - [AUTHN]
     - Token Reused
     - ❌ Hydra checks OAuth tokens; the component validating a rejected, revoked, or replayed token should emit this event.
   * - [AUTHN]
     - Token Deleted
     - ❌ Hydra owns OAuth token records; Airflow's API auth manager owns any Airflow-issued API JWT records.
   * - [AUTHZ]
     - Unauthorized Access Attempt
     - ❌ Airflow's request authorization layer evaluates and denies access.
   * - [AUTHZ]
     - Administrative Activity
     - ✅ Partial: coordinator logs observed security-policy and credential-revision changes. Airflow UI/API admin actions require runtime instrumentation.
   * - [SYS]
     - System Startup
     - ✅ Workload charms observe Pebble stopped-to-running transitions.
   * - [SYS]
     - System Shutdown
     - ✅ Workload charms log successful charm-requested stops of running services.
   * - [SYS]
     - System Restart
     - ✅ Workload charms log successful charm-requested restarts.
   * - [SYS]
     - System Crash
     - ✅ Limited: logged when a failed Pebble operation is followed by Pebble `ERROR` status.
   * - [SYS]
     - System Monitoring Disabled
     - ❌ Charms expose no operation that disables security monitoring.
   * - [USER]
     - User Created
     - ❌ Not applicable to the charm-managed layer. Airflow FAB/IdP owns creation if OAuth registration is enabled.
   * - [USER]
     - User Updated
     - ❌ Not applicable to the charm-managed layer. Airflow/IdP owns user updates if supported.

Authentication-related events [AUTHN]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **Successful Login** (``authn_login_success``): Kratos/Identity Platform emits this
  when it successfully validates a user's credentials. Airflow charms do not observe
  individual login requests.
* **Failed Login** (``authn_login_fail``): Kratos/Identity Platform emits this when
  credential validation fails. Airflow charms do not observe individual failures.
* **Account Lockout** (``authn_login_lock``): lockout is not implemented by Airflow
  charms, and Kratos support has not been confirmed. Identity Platform owns this
  control; if it is supported and enabled, the Identity Platform should emit the
  event. Confirm the deployment's lockout capability with that team before marking
  the event not applicable to the overall product.
* **Password Changed** (``authn_password_change``): Kratos/Identity Platform emits
  this when a user password changes. Machine signing-key and client-secret rotations
  are logged instead as coordinator ``authz_admin`` actions, not as user password
  changes.
* **Token Created** (``authn_token_created``): Hydra emits this when it issues an
  OAuth access token. If Airflow's API auth manager separately issues API JWTs, it
  owns those events. The coordinator's ``secret-key`` and ``jwt-secret`` are signing
  keys, not access tokens, so their creation does not trigger this event.
* **Token Revoked** (``authn_token_revoked``): Hydra emits this when it revokes an
  OAuth token it manages; Airflow's API auth manager owns revocation events for any
  Airflow-issued API JWTs.
* **Token Reused** (``authn_token_reuse``): Hydra emits this when its token checks
  detect a revoked, expired, or replayed OAuth token. Airflow's API auth manager
  owns equivalent events for tokens it validates. Charm hooks do not validate
  request tokens.
* **Token Deleted** (``authn_token_delete``): Hydra emits this when an OAuth token
  record it owns is deleted. Airflow's API auth manager owns deletion events for any
  Airflow-issued token records. Juju signing-secret revisions are not token records.

Authorization-related events [AUTHZ]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **Unauthorized Access Attempt** (``authz_fail``): emitted by Airflow's
  authorization layer when an authenticated request is denied. Charm hooks do not
  evaluate per-request roles or permissions, so this event is not currently emitted
  by the charms.
* **Administrative Activity** (``authz_admin``): the coordinator emits this for
  observed security-policy changes and signing-key/client-secret revisions. Actions
  performed by an administrator through the Airflow UI or API must be emitted by
  the Airflow runtime. Coordinator events use ``actor: unattributed`` and include
  a pointer to Juju audit logs because the hook cannot identify the human initiator.

System-level events [SYS]
~~~~~~~~~~~~~~~~~~~~~~~~~

These events are emitted by the API server, scheduler, DAG processor, and triggerer
charms, with app IDs ``airflow.api-server``, ``airflow.scheduler``,
``airflow.dag-processor``, and ``airflow.triggerer`` respectively.

* **System Startup** (``sys_startup``): emitted when reconcile observes a Pebble
  service transition from stopped to running after Pebble applies the charm's
  service plan.
* **System Shutdown** (``sys_shutdown``): emitted when a charm successfully stops a
  previously running service because its coordinator relation is missing.
* **System Restart** (``sys_restart``): emitted after a charm successfully restarts
  its service due to a configuration change.
* **System Crash** (``sys_crash``): emitted only when a failed Pebble operation is
  followed by Pebble reporting the service as ``ERROR``. It does not cover every
  process crash or Kubernetes eviction.
* **System Monitoring Disabled** (``sys_monitor_disabled``): not emitted. The charms
  do not expose an operation that disables a security monitor or telemetry sensor.
  The coordinator disables the unused workload health check for its database-only
  migration container; that is not treated as disabling security monitoring.

User-related events [USER]
~~~~~~~~~~~~~~~~~~~~~~~~~~

* **User Created** (``user_created``): **Not Applicable to the charm-managed layer**.
  The coordinator and workload charms do not implement or expose account creation.
  If OAuth user registration is enabled, Airflow FAB or the identity provider owns
  registration and must emit the event.
* **User Updated** (``user_updated``): **Not Applicable to the charm-managed layer**.
  The charms do not modify user accounts. If user updates are supported by the
  configured Airflow auth manager or identity provider, that runtime must emit the
  event.

Operational boundaries
----------------------

``[SYS]`` events describe transitions observed or initiated by a charm through Pebble. They are not a complete record of Kubernetes pod eviction, node failure, OOM termination, or every process restart. 

The charm log stream is not automatically a SIEM destination. Operators must
configure log forwarding and JSON field extraction for their environment. The
implementation repository contains experimental `Sigma <https://sigmahq.io/>`_
rules; validate and tune converted rules for the target SIEM before enabling
alerts. See the `Sigma consumer guide
<https://github.com/canonical/airflow-core-operators/tree/track/3.1/docs/security/sigma>`_.
