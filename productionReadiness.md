# Teleport Production Readiness Test Plan

> **Note to Claude:** This document is a structured input for building an executable command set. For each test below, gather the real values for the placeholders listed in that section's "Variables to Gather" checklist (ask the user for any values not already known/available), then substitute them into the commands to produce a ready-to-run script or step list. Do not run destructive or load-generating commands without explicit user confirmation.

## Overview

**Document Purpose:** This test plan provides a structured validation framework for confirming Teleport readiness across critical access modes, security primitives, and operational failover scenarios. Each section covers a specific validation area and can be used jointly or independently with others as needed.

**Audience:** Infrastructure, security, and DevOps engineers validating Teleport pre-production deployment. Assumes working knowledge of Kubernetes, SSH, PKI, and Teleport architecture.

**How to Use This Plan:**
1. Review the section corresponding to your deployment scenario and need.
2. Follow the steps to execute each test case.
3. Validate against the stated success criteria.
4. Mark each test PASS or FAIL and note any deviations.

**Test Areas Covered:**
- SSH Access & Load Testing Validation
- Kubernetes Access & Load Testing Validation
- Database Access & Load Testing Validation
- Desktop Access & Load Testing Validation
- Web UI Access & Load Testing Validation
- CA Rotation Validation
- HSM PKI Validation
- Audit & Session Recording Validation
- Auth Heartbeat Stress & Load Testing Validation
- Failover Behavior (Single Region) Validation

> Only SSH, Kubernetes, Database (Postgres), Desktop, and Web UI Access areas are detailed in this document (see sections below). The remaining areas (CA Rotation, HSM PKI, Audit & Session Recording, Auth Heartbeat Stress & Load Testing, Failover Behavior) were listed in the source overview but no test steps were provided.

**Special Instructions (Backend Pressure Testing):** If you need to simulate thousands of nodes, use Docker Swarm to scale node replicas:
1. Install Docker
2. Run `docker swarm init`
3. Use the appropriate command to spin up Teleport node replicas at scale

**Assessment Tracking Fields:**
- Assessment Start Date: _____
- Assessment End Date: _____
- Approval Parties: _____
- Test Owner(s): _____

---

## 1. SSH Node Access

### Variables to Gather
- `<avg_#_of_user_connections>` — expected average concurrent SSH connections/rate
- `<host_user>` — OS login user on the target host
- `<label_key>` / `<label_value>` — Teleport resource label selecting target node(s)
- `<cmd>` — command to execute over the SSH session
- `<teleport_user>` — Teleport username to audit against
- `<time_window>` — time range for audit queries

### Test 1.1 — Backend/Session Reliability Under Average Load
**Purpose:** Ensure backend/session reliability under average load.

**Steps:**
```
tsh bench ssh --rate=<avg_#_of_user_connections> --duration=10m <host_user>@<label_key>=<label_value> <cmd>
```
Compare `Requests originated` and `Requests failed` in the report output.

**Success Criteria:** >99% session success rate.
**Validation:** Confirm success rate is within acceptable range.

### Test 1.2 — User Session Responsiveness Under Average Load
**Purpose:** Validate user session responsiveness under average load.

**Steps:**
```
tsh bench ssh --rate=<avg_#_of_user_connections> --duration=10m <host_user>@<label_key>=<label_value> <cmd>
```
Inspect P50 and P95 latency values from the report output.

**Success Criteria:**
1. P50 session latency <1s
2. P95 session latency <2s

**Validation:**
1. Confirm latencies are within acceptable range.
2. Calculate requests impacted by each latency percentile: `x = <total_requests> × (1 - 0.<latency_percentile>)`

### Test 1.3 — Resource Cleanup After Sessions
**Purpose:** Ensure proper resource cleanup after sessions.

**Steps:**
1. Record baseline `go_goroutines` and `process_open_fds` metrics.
2. Run:
```
tsh bench ssh --rate=<avg_#_of_user_connections> --duration=10m <host_user>@<label_key>=<label_value> <cmd>
```
3. Monitor metrics during, and 1–2 mins after, the test.
4. Compare with baseline.

**Success Criteria:**
1. No goroutine leaks
2. No file descriptor leaks

**Validation:**
1. Goroutine count should return to baseline ±10% post-test with no long-term growth trend.
2. FD count should remain stable within ±10% with no consistent increase over time.

### Test 1.4 — Auditable Access Trail
**Purpose:** Validate auditable access trail.

**Steps:**
1. Run:
```
tsh bench ssh --rate=<avg_#_of_user_connections> --duration=10m <host_user>@<label_key>=<label_value> <cmd>
```
2. Compare `Requests originated` and `Requests failed` to get total number of successful requests.
3. Run:
```
tctl audit query exec "SELECT COUNT(*) AS start_count FROM session_start WHERE login = '<host_user>' AND user = '<teleport_user>' AND initial_command = ARRAY['<cmd>'] AND time >= '<time_window>';"
```
4. Compare successful requests vs. actual `session_start` count.

**Success Criteria:** 100% audit log/SSH session overlap.
**Validation:** Every started session must have a matching ended event with a consistent session ID.

---

## 2. Kubernetes Exec Access

### Variables to Gather
- `<avg_#_of_user_execs>` — expected average concurrent kubectl exec rate
- `<ns>` — kube-namespace
- `<teleport_kube_cluster_name>` — Teleport-registered Kubernetes cluster name
- `<pod_name>` — target pod
- `<exec_cmd>` — command to exec in the pod
- `<cmd>` — command value for audit filtering (used in audit query; should match `<exec_cmd>`)
- `<teleport_user>` — Teleport username to audit against
- `<time_window>` — time range for audit queries

### Test 2.1 — Backend/Exec Reliability Under Average Load
**Purpose:** Ensure backend/exec reliability under average load.

**Steps:**
```
tsh bench kube exec --rate=<avg_#_of_user_execs> --duration=10m --kube-namespace=<ns> <teleport_kube_cluster_name> <pod_name> "<exec_cmd>"
```
Compare `Requests originated` and `Requests failed` in report output.

**Success Criteria:** >99% kubectl exec success rate.
**Validation:** Confirm success rate is within acceptable range.

### Test 2.2 — User Session Responsiveness Under Average Load
**Purpose:** Validate user session responsiveness under average load.

**Steps:**
```
tsh bench kube exec --rate=<avg_#_of_user_execs> --duration=10m --kube-namespace=<ns> <teleport_kube_cluster_name> <pod_name> "<exec_cmd>"
```
Inspect P50 and P95 latency values from report output.

**Success Criteria:**
1. P50 exec latency <1s
2. P95 exec latency <2s

**Validation:**
1. Confirm latencies are within acceptable range.
2. Calculate requests impacted by each latency percentile: `x = <total_requests> × (1 - 0.<latency_percentile>)`

### Test 2.3 — Cluster-Side Exec Routing Performance Under Average Load
**Purpose:** Validate cluster-side exec routing performance under average load.

**Steps:**
1. Run:
```
tsh bench kube exec --rate=<avg_#_of_user_execs> --duration=10m --kube-namespace=<ns> <teleport_kube_cluster_name> <pod_name> "<exec_cmd>"
```
2. Monitor metrics during, and 1–2 mins after, the test.
3. Review `teleport_kubernetes_client_request_duration_seconds`, `teleport_kubernetes_client_in_flight_requests`, and `teleport_kubernetes_client_requests_total` metrics.

**Success Criteria:**
1. No request queuing saturation
2. No request duration spikes

**Validation:**
1. `in_flight_requests` should remain low or transient.
2. `request_duration_seconds` histogram should remain within expected latency bounds.

### Test 2.4 — Auditable Access Trail
**Purpose:** Validate auditable access trail.

**Steps:**
1. Run:
```
tsh bench kube exec --rate=<avg_#_of_user_execs> --duration=10m --kube-namespace=<ns> <teleport_kube_cluster_name> <pod_name> "<exec_cmd>"
```
2. Record number of successful requests from bench output.
3. Run:
```
tctl audit query exec "SELECT COUNT(*) FROM exec WHERE user = '<teleport_user>' AND command = '<cmd>' AND time >= '<time_window>';"
```
4. Compare number of successful exec requests vs. actual exec count to verify audit logging completeness.

**Success Criteria:** 100% audit log/exec session overlap.
**Validation:** All successful exec requests must be logged as exec events.

---

## 3. Postgres DB Access

### Variables to Gather
- `<teleport-db-service-name>` — Teleport-registered database service name
- `<db>` — database name
- `<teleport-db-user>` — Teleport database user
- `<avg_#_of_queries_per_user>` — expected average query rate per user
- `<teleport_user>` — Teleport username to audit against
- `<time_window>` — time range for audit queries

### Test 3.1 — Backend/Query Reliability Under Average Load
**Purpose:** Ensure backend/query reliability under average load.

**Steps:**
```
tsh bench postgres <teleport-db-service-name> --db-name=<db> --user=<teleport-db-user> --rate=<avg_#_of_queries_per_user> --duration=10m
```
Compare `Requests originated` and `Requests failed` in bench output.

**Success Criteria:** >99% query success rate.
**Validation:** Confirm success rate is within acceptable range.

### Test 3.2 — User Session Performance Under Average Load
**Purpose:** Validate user session performance under average load.

**Steps:**
```
tsh bench postgres <teleport-db-service-name> --db-name=<db> --user=<teleport-db-user> --rate=<avg_#_of_queries_per_user> --duration=10m
```
Inspect P50 and P95 latency values from command report output.

**Success Criteria:**
1. P50 query latency <1s
2. P95 query latency <2s

**Validation:**
1. Confirm latencies are within acceptable range.
2. Calculate requests impacted by each latency percentile: `x = <total_requests> × (1 - 0.<latency_percentile>)`

### Test 3.3 — Teleport Proxy Stability Under DB Access Load
**Purpose:** Validate Teleport proxy stability under DB access load.

**Steps:**
1. Run:
```
tsh bench postgres <teleport-db-service-name> --db-name=<db> --user=<teleport-db-user> --rate=<avg_#_of_queries_per_user> --duration=10m
```
2. Monitor `teleport_db_initialized_connections_total` to track session volume.
3. Monitor `teleport_db_active_connections_total` for concurrency.
4. Monitor `teleport_db_errors_total` for proxy error spikes.

**Success Criteria:**
1. Active connections return to baseline ±10% post-test
2. Error count remains 0 or unchanged

**Validation:**
1. Confirm connection teardown behaves as expected under load.
2. Confirm no synthetic proxy errors occurred.

### Test 3.4 — Auditable Access Trail
**Purpose:** Validate auditable access trail.

**Steps:**
1. Run:
```
tsh bench postgres <teleport-db-service-name> --db-name=<db> --user=<teleport-db-user> --rate=<avg_#_of_queries_per_user> --duration=10m
```
2. Record number of successful requests from bench output.
3. Run:
```
tctl audit query exec "SELECT COUNT(*) FROM db_session_query WHERE user = '<teleport_user>' AND time >= '<time_window>';"
```
4. Compare number of successful DB query requests vs. actual `db_session_query` count to verify audit logging completeness.

**Success Criteria:** 100% audit log/DB session overlap.
**Validation:** All successful DB query requests must be logged as `db_session_query` events.

---

## 4. Desktop Access

### Variables to Gather
- `<teleport_user>` — Teleport username to audit against
- `<time_window>` — time range for audit queries
- Idle timeout value for `client_idle_timeout` test (e.g., `2m`)
- List of connected desktops (for spot-checking if too numerous to test exhaustively)

### Test 4.1 — Backend/Session Reliability Under Average Load
**Purpose:** Ensure backend/session reliability under average load.

**Steps:**
1. Initiate multiple RDP sessions via Web UI to each connected desktop.
   - Note: if the number of connected desktops is too large to feasibly test every one, spot-check the most used/critical instances only.

**Success Criteria:** 100% RDP session success rate.
**Validation:** Confirm all tested desktops are reachable and connect successfully even when multiple users are connected simultaneously.

### Test 4.2 — User Session Responsiveness Under Average Load
**Purpose:** Validate user session responsiveness under average load.

**Steps:**
1. Initiate multiple RDP sessions via Web UI to each connected desktop.
2. Note time to desktop load and responsiveness of input/display.

**Success Criteria:**
1. Desktop renders in <2s
2. User input is responsive

**Validation:** Ensure user experience remains fast and fluid under concurrent load.

### Test 4.3 — Clipboard Sharing Functionality (if enabled)
**Purpose:** Validate clipboard sharing functionality.

**Steps:**
1. Copy/paste content between the local machine and the RDP session.
2. Run:
```
tctl audit query exec "SELECT COUNT(*) FROM windows_desktop_clipboard WHERE user = '<teleport_user>' AND time >= '<time_window>';"
```

**Success Criteria:**
1. Clipboard sharing works
2. Clipboard sharing generates `windows_desktop_clipboard` events

**Validation:** Confirm feature is functional and properly audited for session observability/compliance.

### Test 4.4 — Directory Sharing Functionality (if enabled)
**Purpose:** Validate directory sharing functionality.

**Steps:**
1. Share a folder from the browser during the session.
2. Upload or download a file via the shared folder.
3. Run:
```
tctl audit query exec "SELECT COUNT(*) FROM windows_desktop_directory WHERE user = '<teleport_user>' AND time >= '<time_window>';"
```

**Success Criteria:**
1. Directory sharing works
2. Directory sharing generates `windows_desktop_directory` events

**Validation:** Confirm file transfer capability works correctly and actions are tracked in audit logs.

### Test 4.5 — Idle Timeout Enforcement
**Purpose:** Validate idle timeout enforcement.

**Steps:**
1. Configure `client_idle_timeout` (e.g., `2m`).
2. Connect and remain idle.
3. Observe the session after the timeout passes.

**Success Criteria:** Confirm idle timeout sessions expire and disconnect after the configured interval.
**Validation:** Confirm idle session management works and no zombie sessions/wasted resources are left hanging around.

### Test 4.6 — Auditable Access Trail
**Purpose:** Validate auditable access trail.

**Steps:**
1. Initiate multiple RDP sessions via Web UI to each connected desktop.
2. Run:
```
tctl audit query exec "SELECT COUNT(*) FROM windows_desktop_session_start WHERE user = '<teleport_user>' AND time >= '<time_window>';"
```
3. Run the equivalent query for session end events.
4. Compare with expected session count.

**Success Criteria:**
1. 100% audit/RDP session overlap
2. 100% parity between windows desktop session start and session end events

**Validation:** All successful RDP sessions must be logged as auditable start/stop events.

### Test 4.7 — RDP Session Recording Integrity
**Purpose:** Validate RDP session recording integrity.

**Steps:**
1. Initiate an RDP session via Web UI to any connected desktop.
2. Close the RDP session and wait a few minutes (for backend session syncing).
3. Replay using:
```
tsh play <session-id>
```
or via the Web UI.

**Success Criteria:** Session is recorded and viewable without corruption.
**Validation:** Confirm playback is complete, smooth, and matches the timeline of the original session.

---

## 5. WebUI Access

### Variables to Gather
- `<subcommand>` — one of: `sessions`, `access-requests`, `audit-log`, `nodes`, `databases`, `kubernetes`, `desktops`, `recordings`, `auth-connectivity`
- `<avg_user_web_sessions_per_sec>` — expected average Web UI session rate
- `<username>` — must be valid in Teleport RBAC for the resource type being tested (e.g., included in a role's `logins` if testing SSH sessions)
- `<cmd>` — command to run in the simulated session
- `<teleport_user>` — Teleport username to audit against
- `<login_method>` — SSO login method used
- `<time_window>` — time range for audit queries

### Special Requirements for `tsh bench web`
`tsh bench web` simulates a user logging into Teleport via the Web UI and accessing connected resources. Requirements:
1. Log into Teleport via `tsh` using a Teleport **local user** (not SSO).
2. Ensure the local user has access to all resources being tested with `<subcommand>` (e.g., SSH nodes if using `sessions`).
3. The `<username>` passed to the command must be valid within the RBAC configuration for that resource type (e.g., if using `sessions` and `root` as `<username>`, the role being tested must include `root` in `logins`).
4. Re-authenticate with the local user's credentials when prompted once the test starts.

For a quick baseline test, use the `sessions` subcommand:
```
tsh bench web sessions --ttl=1 --duration=10m --rate=10 root ls
```

### Test 5.1 — Backend Reliability Under Average Load
**Purpose:** Ensure backend reliability under average load.

**Steps:**
```
tsh bench web <subcommand> --ttl=1 --duration=10m --rate=<avg_user_web_sessions_per_sec> <username> <cmd>
```
Observe errors/timeouts in bench output.

**Success Criteria:** >99% success rate.
**Validation:** Confirm success rate is within acceptable range under load.

### Test 5.2 — Web UI Responsiveness Under Backend Load
**Purpose:** Validate Web UI responsiveness under backend load.

**Steps:**
1. Run:
```
tsh bench web <subcommand> --ttl=1m --duration=10m --rate=<avg_user_web_sessions_per_sec> <username> <cmd>
```
2. While the bench command is running:
   - Log in via Web UI.
   - Navigate across UI tabs (Audit, Nodes, Desktops, Apps).
   - Attempt to connect to node/desktop sessions.
3. Record any UI delays, rendering issues, or failed connections.

**Success Criteria:** Web UI remains responsive and all core workflows functional under load.
**Validation:** Confirm that backend stress does not degrade end-user Web UI experience.

### Test 5.3 — TTL-Based Session Expiry via Web UI
**Purpose:** Validate TTL-based session expiry via Web UI.

**Steps:**
1. Set session TTL in the user role to `1m`.
2. Log in via Web UI and connect to a session.
3. Observe auto-disconnect after the TTL expires.

**Success Criteria:** TTL-enforced session ends after the expected interval.
**Validation:** Check the `session.end` event timestamp to confirm it aligns with TTL expiry.

### Test 5.4 — Audit Event Generation from Web UI Activity
**Purpose:** Validate audit event generation from Web UI activity.

**Steps:**
1. Log in to the Web UI via SSO.
2. Run:
```
tctl audit query exec "SELECT COUNT(*) FROM user_login WHERE user = '<teleport_user>' AND method = '<login_method>' AND success = true AND time >= '<time_window>';"
```

**Success Criteria:** 100% audit event coverage for Web UI login events.
**Validation:** Confirm that Web UI SSO login is fully functioning and trackable.