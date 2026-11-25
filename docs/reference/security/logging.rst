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
       "target": "airflow-api-server",
       "description": "Service restarted by the charm.",
       "data": {"trigger": "configuration_changed"}
     }
   }

The logger emits the OWASP event under the ``owasp_event`` key in an OTLP-compatible
JSON record. ``datetime`` is UTC with an explicit offset. ``appid`` identifies the
product component, while ``event`` contains the event type and schema parameters.
``actor`` and ``target`` provide context, and ``data`` carries supplementary
product-specific metadata. The action is encoded in the ``event`` identifier;
severity is represented by ``level`` rather than a separate outcome field. Secret
values, API keys, JWT signing keys, OAuth client secrets, and bearer tokens are
never logged.

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
     - ❌ Kratos validates user credentials; the Identity Platform owns the login event; emission is not verified here.
   * - [AUTHN]
     - Failed Login
     - ❌ Kratos rejects user credentials; the Identity Platform owns the failure event; emission is not verified here.
   * - [AUTHN]
     - Account Lockout
     - ❌ Not available: neither the Airflow charms nor Kratos implement account lockout.
   * - [AUTHN]
     - Password Changed
     - ❌ Kratos/Identity Platform owns user password changes; emission is not verified here. Machine signing-key, fernet-key, and client-secret rotations use [AUTHZ] administrative events instead.
   * - [AUTHN]
     - Token Created
     - ❌ Hydra owns OAuth access-token issuance; emission is not verified here. Airflow API JWTs, if issued by Airflow, belong to its API auth manager. Signing-key creation is not token creation.
   * - [AUTHN]
     - Token Revoked
     - ❌ Hydra owns OAuth token revocation; emission is not verified here. Airflow's API auth manager owns any Airflow-issued API JWTs.
   * - [AUTHN]
     - Token Reused
     - ❌ Hydra checks OAuth tokens; the component validating a rejected, revoked, or replayed token should emit this event; emission is not verified here.
   * - [AUTHN]
     - Token Deleted
     - ❌ Hydra owns OAuth token records; emission is not verified here. Airflow's API auth manager owns any Airflow-issued API JWT records.
   * - [AUTHZ]
     - Unauthorized Access Attempt
     - ❌ Airflow's authorization layer owns request-level denials; charm hooks do not evaluate per-request permissions.
   * - [AUTHZ]
     - Administrative Activity
     - ✅ Coordinator logs observed security-policy changes and rotations of the API signing keys, the fernet key, and the Hydra client secret; Provider Configurator logs successfully published configuration changes. Airflow UI/API admin actions require runtime instrumentation.
   * - [SYS]
     - System Startup
     - ✅ Core workload charms and Provider Configurator observe Pebble stopped-to-running transitions.
   * - [SYS]
     - System Shutdown
     - ✅ Core workload charms and Provider Configurator log successful charm-requested stops of running services.
   * - [SYS]
     - System Restart
     - ✅ Core workload charms and Provider Configurator log successful charm-requested restarts.
   * - [SYS]
     - System Crash
     - ✅ Limited: logged for a failed Pebble operation followed by Pebble ``ERROR`` status.
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

* **Successful Login** (``authn_login_success``): Kratos/Identity Platform is responsible
  for this event because it validates user credentials. Airflow charms do not observe
  individual login requests, so the charm layer does not emit this event. Emission by
  Kratos in this deployment is not verified.
* **Failed Login** (``authn_login_fail``): Kratos/Identity Platform is responsible for
  this event when credential validation fails. Airflow charms do not observe individual
  failures. Emission by Kratos in this deployment is not verified.
* **Account Lockout** (``authn_login_lock``): **Not available** in the current
  deployment; neither the Airflow charms nor Kratos implement account lockout.
* **Password Changed** (``authn_password_change``): Kratos/Identity Platform is
  responsible for this event when a user password changes. Machine signing-key,
  fernet-key, and client-secret rotations are logged instead as coordinator
  ``authz_admin`` actions, not as user password changes. Emission by Kratos in this
  deployment is not verified.
* **Token Created** (``authn_token_created``): Hydra is responsible for this event when
  it issues an OAuth access token. If Airflow's API auth manager separately issues API
  JWTs, it owns those events. The coordinator's ``secret-key`` and ``jwt-secret`` are
  signing keys, not access tokens, so their creation does not trigger this event.
  Emission by Hydra in this deployment is not verified.
* **Token Revoked** (``authn_token_revoked``): Hydra is responsible for this event when
  it revokes an OAuth token it manages; Airflow's API auth manager owns revocation
  events for any Airflow-issued API JWTs. Emission by Hydra in this deployment is not
  verified.
* **Token Reused** (``authn_token_reuse``): Hydra is responsible for this event when its
  token checks detect a revoked, expired, or replayed OAuth token. Airflow's API auth
  manager owns equivalent events for tokens it validates. Charm hooks do not validate
  request tokens. Emission by Hydra in this deployment is not verified.
* **Token Deleted** (``authn_token_delete``): Hydra is responsible for this event when an
  OAuth token record it owns is deleted. Airflow's API auth manager owns deletion events
  for any Airflow-issued token records. Juju signing-secret revisions are not token
  records. Emission by Hydra in this deployment is not verified.

Authorization-related events [AUTHZ]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **Unauthorized Access Attempt** (``authz_fail``): emitted by Airflow's
  authorization layer when an authenticated request is denied. Charm hooks do not
  evaluate per-request roles or permissions, so request-level ``authz_fail`` is not
  currently emitted by Airflow charms. When the Provider Configurator's Layer 2
  denylist drops a prohibited provider setting, it is logged as invalid input
  rather than an access attempt: ``input_validation_discrete_fail:<config-key>,unattributed``
  at ``WARN``. Repeated identical denied-key sets are deduplicated.
* **Administrative Activity** (``authz_admin``): the coordinator emits these
  actions:

  * ``update_security_config``: a security-related charm option changed (user
    registration, IdP group-to-role mappings, or OAuth authorization). Only the names
    of the changed options are logged.
  * ``rotate_signing_key``: the revision of the coordinator-owned API signing-key
    secret changed.
  * ``rotate_fernet_key``: the ``fernet_key_secret`` option now points to a
    different secret (``observed_via: charm_config``), or a new revision of the
    configured secret was received (``observed_via: juju_secret_changed``).
  * ``update_client_secret``: the Hydra OAuth client ID or client secret published
    on the ``oauth`` relation changed (``observed_via: oauth_relation_data``), or a
    new revision of that secret was received (``observed_via: juju_secret_changed``).

  The fernet key and Hydra client secrets are owned by the user and by Hydra, so
  the coordinator cannot read their revision numbers; it records only their secret
  IDs and reacts to Juju ``secret-changed`` notifications. These events mean the
  coordinator observed the change. They do not confirm that every Airflow component
  has started using the new value; that is shown by each component's ``sys_restart``
  event.

  Actions performed by an administrator through the Airflow UI or API must be
  emitted by the Airflow runtime. The Provider Configurator emits this when changed
  provider configuration is successfully published. Charm-side events use
  ``actor: unattributed`` and point to Juju or repository audit logs because the
  hook cannot identify the human initiator.

System-level events [SYS]
~~~~~~~~~~~~~~~~~~~~~~~~~

These events are emitted by the API server, scheduler, DAG processor, and triggerer
charms, with app IDs ``airflow.api-server``, ``airflow.scheduler``,
``airflow.dag-processor``, and ``airflow.triggerer`` respectively. The Provider
Configurator emits lifecycle events for its Pebble-managed ``git-sync`` service
with app ID ``airflow.provider-configurator``.

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

Locating events in Juju logs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Security events are read directly from the charm log stream with ``juju debug-log``. Each
record is a single line containing the ``owasp_event`` JSON object, so operators can
filter for the key and pretty-print the matches with ``jq``:

.. code-block:: bash

   # Replay all buffered logs and show only security events
   juju debug-log --replay --no-tail | grep -o '{"owasp_event".*}' | jq .

The ``grep -o`` step strips the Juju log prefix so that only the JSON object is passed
to ``jq``. To narrow the output to a specific component or event type, filter on the
``appid`` and ``event`` fields:

.. code-block:: bash

   # Show only API server restarts
   juju debug-log --replay --no-tail \
     | grep -o '{"owasp_event".*}' \
     | jq 'select(.owasp_event.appid == "airflow.api-server"
                  and (.owasp_event.event | startswith("sys_restart")))'

Omit ``--replay --no-tail`` to follow new events as they are emitted. These commands
verify that events are being produced; they are not a substitute for forwarding the
logs to a durable, searchable store.

The charm log stream is not automatically a SIEM destination. Operators must
configure log forwarding and JSON field extraction for their environment. The
implementation repository contains experimental `Sigma <https://sigmahq.io/>`_
rules; validate and tune converted rules for the target SIEM before enabling
alerts. See the `Sigma consumer guide
<https://github.com/canonical/airflow-core-operators/tree/track/3.1/docs/security/sigma>`_.
