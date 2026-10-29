# DR Playbook: Individual CN(s) Unable to Publish Blocks

> **Summary:** One or more Consensus Nodes (CNs) cannot publish blocks to the Block Node.
> Other CNs are still publishing; the network is still processing transactions. As long as
> 2/3 of CNs are still publishing, the network continues - you have time to diagnose carefully.
> Identify the EndOfStream code from BN logs or metrics, classify the failure, and resolve
> at the CN or BN-network layer. Most cases resolve in under 30 minutes without a BN restart.

|                      |                                                                                                        |
|----------------------|--------------------------------------------------------------------------------------------------------|
| **Playbook ID**      | PB-05                                                                                                  |
| **Severity**         | SEV-2 (partial publishing loss - network still processing transactions)                                |
| **Primary executor** | Council Node Operator + BN DevOps                                                                      |
| **Escalate to**      | Hashgraph Engineering if `INVALID_REQUEST` persists after CN reconnect or `PERSISTENCE_FAILED` appears |

---

## Is this your doc?

Check these without running any commands:

- A CN operator reports their CN is being rejected or keeps disconnecting from the BN
- `publisher_open_connections` on the BN is lower than the expected number of active CN publishers
- `publisher_stream_errors` metric is rising on the BN
- No SEV-1 alert is firing - the network appears operational (confirmed in Phase 2)

**Not your doc if:**

- All CNs are halted and transactions are stopped - all CNs are affected, start with PB-04
- The BN process is down or the host is unreachable - see PB-02
- The BN is returning block verification errors (`BAD_BLOCK_PROOF`, `MISSING_VERIFICATION_DATA`,
  etc.) - see PB-06
- The issue affects all publishers simultaneously - this indicates a BN-side problem; start
  with PB-04 or PB-06 depending on symptoms

---

## Before starting

> **Open the incident bridge immediately. Assign:**
> - **Incident commander** - owns resolution, makes go/no-go calls on each phase
> - **Communication lead** - owns public status page and stakeholder updates (not the person
> executing steps)
>
> Every phase below is executed by the incident commander and designated operators.
> Communication lead posts updates externally while technical work proceeds in parallel.

**These commands assume a Kubernetes deployment** using the Hiero Block Node Helm chart. For
bare-metal or VM deployments, substitute `journalctl -u block-node-server` for `kubectl logs`
and use the host IP directly instead of port-forwarding.

**Coordinate with the CN operator early.** Categories B and C require action on the CN side.
Get the CN operator on the incident bridge before starting Phase 3 so they are ready to act
when needed.

> **Before starting Phase 1:**
> - Do not restart the BN to fix a single-CN issue - a BN restart disconnects all publishers
> and drops in-progress blocks
> - Do not assume the issue is BN-side without first checking CN-side connectivity and logs

**Prerequisite for Category B (serverStatus check):** `grpcurl` and the bn-proto bundle are
required for the block status check in B1. Set this up before an incident - see
[PB-06 Before Starting](./06-bn-data-corruption.md#before-starting) for download steps.

**Set these shell variables before running any commands:**

```bash
NAMESPACE=block-node          # Kubernetes namespace
RELEASE=<helm-release-name>   # Helm release name (check: helm list -n block-node)
BN_HOST=<bn-host>             # BN hostname or IP (see pre-incident config record)
BN_GRPC_PORT=40840            # BN gRPC port (BNCE default)
BN_STATUS_PORT=40982          # BN status port (BNCE default)
MN_HOST=<mn-host>             # Mirror Node host (see pre-incident config record)
```

## Phase 1 - Diagnose

> If the CN operator has already told you the EndOfStream code, you can start at Step 5 -
> but still run Steps 1 and 4 first to capture baseline metrics before proceeding.

**Get the pod name** (Kubernetes deployments):

```bash
kubectl get pods -n $NAMESPACE -l app.kubernetes.io/instance=$RELEASE
```

| # |                                     Action                                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Done |
|---|--------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| 1 | Check how many publisher connections are currently open on the BN.             | `curl -s http://$BN_HOST:16007/metrics \| grep -E "publisher_open_connections\|publisher_active_handlers\|publisher_passive_handlers"`. Note the values: `publisher_open_connections` = _______ / `publisher_active_handlers` = _______ / `publisher_passive_handlers` = _______. Expected in a healthy BNCE environment: `publisher_open_connections` = 7 (one per CN). If the count is lower, one or more CNs have disconnected. If the count is 0, all publishers are gone - this is PB-04 territory.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | ☐    |
| 2 | Identify which CN(s) are affected and the EndOfStream code the BN sent them.   | `kubectl logs -n $NAMESPACE $RELEASE-block-node-server-0 -c block-node-server \| grep -iE "EndStream\|TOO_FAR_BEHIND\|DUPLICATE_BLOCK\|PERSISTENCE_FAILED\|INVALID_REQUEST\|TIMEOUT\|Behind\|publisher.*error\|handler.*behind"`<br><br>**Look for:** Log format varies by code:<br>- `"Handler N received EndStream with TIMEOUT"` (INFO) or `"...TOO_FAR_BEHIND"` (DEBUG) - CN terminated the stream<br>- `"Handler N is sending DUPLICATE_BLOCK(N)"` (DEBUG) - BN rejected duplicate block<br>- `"Handler N ending with code PERSISTENCE_FAILED"` (DEBUG) - BN storage failure<br>- `"Handler N is sending Behind(N)"` (DEBUG) - BehindPublisher, not an EndOfStream<br>**Record:** handler _______ / code _______<br><br>**Log level caveat:** CN-terminated `TIMEOUT` and `ERROR` streams log at INFO (always visible). `TOO_FAR_BEHIND`, `DUPLICATE_BLOCK`, `BehindPublisher`, `PERSISTENCE_FAILED`, and BN-initiated stall timeouts all log at DEBUG - invisible at default INFO log level.<br><br>**If grep returns nothing:** rely on metrics from Step 4, or add `logging.level.org.hiero.block.node.stream.publisher=DEBUG` to the BN config and restart to enable debug logging. | ☐    |
| 3 | Check whether the affected CN(s) can reach BN port 40840 at the network level. | Run from the **CN host**: `nc -zv $BN_HOST $BN_GRPC_PORT`. Expected output: `Connection to <host> 40840 port [tcp/*] succeeded!`. If the command times out or returns a connection refused error, the issue is network/firewall (Category A). If it succeeds, the network is open and the issue is at the application layer (Categories B, C, or D).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | ☐    |
| 4 | Check publisher-level error and stall metrics on the BN.                       | `curl -s http://$BN_HOST:16007/metrics \| grep -E "publisher_stream_errors\|publisher_stall_timeouts_sent\|publisher_block_node_behind_sent\|publisher_block_endofstream_sent\|publisher_disconnected_oversize"`. Note all non-zero values: `publisher_stream_errors_total` = _______ / `publisher_stall_timeouts_sent_total` = _______ / `publisher_block_node_behind_sent_total` = _______ / `publisher_block_endofstream_sent_total` = _______. A rising `publisher_block_node_behind_sent_total` means the BN is actively telling CNs they are behind. A rising `publisher_stall_timeouts_sent_total` means CNs are stalling mid-block. A rising `publisher_disconnected_oversize_total` means one or more CNs sent messages exceeding the BN's hard size limit - coordinate with the CN team on message size configuration.                                                                                                                                                                                                                                                                                                                                                             | ☐    |

**Step 5 - Classify the failure:**

|                                    Signal from Steps 2-4                                     | Category |              Typical time to recover              |               Escalate immediately if...               |
|----------------------------------------------------------------------------------------------|----------|---------------------------------------------------|--------------------------------------------------------|
| CN cannot reach BN port 40840 (Step 3 fails)                                                 | **A**    | 15-60 minutes (network/firewall fix)              | Fix applied but CN still cannot connect                |
| BN log: `"sending Behind(N)"` for the CN's handler (requires DEBUG log level)                | **B**    | 15-30 minutes (CN reconnects at correct block)    | CN operator cannot reconnect or block range is missing |
| Metric `publisher_block_node_behind_sent_total` rising (no DEBUG logs needed)                | **B**    | 15-30 minutes (CN reconnects at correct block)    | CN operator cannot reconnect or block range is missing |
| BN log: `"received EndStream with TIMEOUT"` (INFO) or `"is sending DUPLICATE_BLOCK"` (DEBUG) | **C**    | 5-15 minutes (usually self-resolves on reconnect) | Repeats more than 3 times without clearing             |
| BN log: `"ending with code INVALID_REQUEST"` (DEBUG)                                         | **C**    | Varies - CN software issue                        | Persists after CN reconnect - escalate to Engineering  |
| BN log: `"ending with code PERSISTENCE_FAILED"` (DEBUG); all handlers drop simultaneously    | **D**    | Pivot to PB-06                                    | Immediately - this is a BN storage failure             |

Category: _______

> Continue to Phase 2 before starting Phase 3 - log preservation in Phase 2 Step 2 is
> critical if a restart may follow.

---

## Phase 2 - Buy Time

| # |                                        Action                                        |                                                                                                                                                                         Notes                                                                                                                                                                         | Done |
|---|--------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| 1 | Confirm the network is still processing transactions despite the CN publish failure. | Ask the CN operator or check the Mirror Node REST API: `curl -s http://$MN_HOST:<MN_REST_PORT>/api/v1/transactions?limit=1&order=desc` (BNCE REST port: 8000). A recent transaction timestamp confirms transactions are flowing. As long as 2/3 of CNs are still publishing to the BN, the network continues. If transactions have halted: see PB-04. | ☐    |
| 2 | Preserve BN logs before any restart.                                                 | `kubectl logs -n $NAMESPACE $RELEASE-block-node-server-0 -c block-node-server > /tmp/bn_publisher_incident_$(date +%Y%m%d_%H%M%S).log`. Share this log with Engineering when escalating.                                                                                                                                                              | ☐    |
| 3 | Contact the CN operator and get them on the incident bridge if not already present.  | Provide them with: the handler ID from Step 2, the EndOfStream code, and the block number from `"sending Behind(N)"` if present. They will need this for Category B recovery.                                                                                                                                                                         | ☐    |

---

## Phase 3 - Recover

Identify your failure category from Phase 1 Step 5 and jump to that section.

---

### Category A - Network or firewall blocking CN-to-BN connectivity

The CN cannot reach BN gRPC port 40840. The connection is being blocked at the network layer -
either a firewall rule on the BN side, a firewall rule on the CN side, a routing issue, or a
DNS resolution failure.

| #  |                                      Action                                      |                                                                                                                                                                                                                                                                                  Notes                                                                                                                                                                                                                                                                                   | Done |
|----|----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| A1 | Verify DNS resolution for the BN hostname from the CN host.                      | Run from the CN host: `dig $BN_HOST` or `nslookup $BN_HOST`. The response should resolve to the BN's IP. If resolution fails: check DNS configuration on the CN host and whether the BN hostname is in the expected zone.                                                                                                                                                                                                                                                                                                                                                | ☐    |
| A2 | Check nftables rules on the BN host for the publisher port.                      | Run on the BN host directly (not inside the pod - nftables are host-level, not container-level): `ssh $BN_HOST "nft list ruleset \| grep 40840"`. Port 40840 should be open for inbound connections from CN IP ranges. If missing: the firewall rule was lost (possibly after a host reboot or chart upgrade) - re-apply the nftables rules from the [BN deployment runbook](https://github.com/hashgraph/block-node-infrastructure/blob/main/deployments/block-node-cutover/bnce_deploy_block_nodes.sh). **Do not delete existing rules** - only add the missing entry. | ☐    |
| A3 | Check whether the CN host's outbound firewall is blocking gRPC to BN port 40840. | Run from the CN host: `iptables -L OUTPUT -n -v \| grep 40840` or the equivalent for the CN's firewall. Outbound to BN port 40840 should be unrestricted. If blocked: coordinate with the CN operator to open the rule.                                                                                                                                                                                                                                                                                                                                                  | ☐    |
| A4 | After fixing the network rule, confirm the CN reconnects.                        | `watch -n 5 'curl -s http://$BN_HOST:16007/metrics \| grep publisher_open_connections'`. The value should increase back to the expected count within 1-2 minutes of the fix. If it does not: check whether the CN's block streaming service restarted its publisher (the CN may need a manual publisher reconnect from the CN operator). Go to Phase 4.                                                                                                                                                                                                                  | ☐    |

---

### Category B - CN is too far behind (BehindPublisher response)

The BN sent a `BehindPublisher` response to the CN, indicating the CN is trying to publish
a block the BN's active buffer no longer holds. The response includes the block number the BN
is currently at. The CN must reconnect and resume publishing from that block number. This is
a CN-side operation.

| #  |                                     Action                                      |                                                                                                                                                                                                                                                                                                                                                                            Notes                                                                                                                                                                                                                                                                                                                                                                             | Done |
|----|---------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| B1 | Confirm the block number the BN reported as its current position.               | From the log captured in Phase 1 Step 2: the `"sending Behind(N)"` message includes the block number N the BN is at. Also confirm via serverStatus (requires `grpcurl` + bn-proto bundle; see **Prerequisite for Category B** in the Before Starting section above): `grpcurl -plaintext -import-path ~/bn-proto -proto block-node/api/node_service.proto -d '{}' $BN_HOST:$BN_STATUS_PORT org.hiero.block.api.BlockNodeService/serverStatus \| jq '.lastAvailableBlock'`. Record the block number: _______.                                                                                                                                                                                                                                                 | ☐    |
| B2 | Provide the CN operator with the target block number and ask them to reconnect. | Your role here is coordination - provide the block number and confirm reconnect on the BN metrics (B3); the CN operator executes the restart. The CN operator should restart block streaming on their CN to resume from block N (the BN's current position) rather than where the CN left off (they will know their deployment's restart procedure). On reconnect, the BN will send a `SkipBlock` response for blocks it already has, allowing the CN to fast-forward to the live tip.                                                                                                                                                                                                                                                                       | ☐    |
| B3 | Confirm the CN publisher reconnects and resumes at the correct block.           | Watch `publisher_open_connections` recover: `watch -n 5 'curl -s http://$BN_HOST:16007/metrics \| grep publisher_open_connections'`. The value should increase. To confirm via logs (requires DEBUG level): `kubectl logs -n $NAMESPACE $RELEASE-block-node-server-0 -c block-node-server \| grep "Added new handler"` - the BN logs `"Added new handler N"` at DEBUG when a publisher connects. Then watch ACKs: `watch -n 5 'curl -s http://$BN_HOST:16007/metrics \| grep publisher_blocks_ack_sent'`. The counter should increase as the CN streams forward. If the CN keeps getting `BehindPublisher` after reconnect: the CN's block storage may be significantly behind - escalate to Hashgraph Engineering with the block number gap. Go to Phase 4. | ☐    |

---

### Category C - Transient disconnect (DUPLICATE_BLOCK, TIMEOUT, INVALID_REQUEST, ERROR)

`DUPLICATE_BLOCK` means the CN sent a block the BN already received from another CN.
`TIMEOUT` means the CN stalled mid-block and the BN closed the session. Both are handled by
the CN reconnecting; the BN will send `SkipBlock` for duplicates so the CN can fast-forward.
`INVALID_REQUEST` indicates a malformed stream - usually a CN software bug. `ERROR` is a
BN-side processing error for this specific handler - reconnect; if all handlers drop
simultaneously, pivot to Category D.

| #  |                                           Action                                            |                                                                                                                                                                                                                                                                                                                           Notes                                                                                                                                                                                                                                                                                                                           | Done |
|----|---------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| C1 | Check whether the CN has already reconnected automatically.                                 | `watch -n 5 'curl -s http://$BN_HOST:16007/metrics \| grep publisher_open_connections'`. If the count has recovered to the expected value within 2-3 minutes, the CN reconnected on its own. Monitor `publisher_stream_errors_total` and `publisher_stall_timeouts_sent_total` for continued errors. If no further errors: go to Phase 4.                                                                                                                                                                                                                                                                                                                 | ☐    |
| C2 | If the CN has not reconnected after 5 minutes, ask the CN operator to manually reconnect.   | The CN operator should restart block streaming on their CN (they will know their CN deployment's restart procedure). The CN will re-establish the gRPC connection to BN port 40840 and the BN will send `SkipBlock` for any blocks it already holds, allowing the CN to resume at the live tip.                                                                                                                                                                                                                                                                                                                                                           | ☐    |
| C3 | If the failure code was `INVALID_REQUEST` and it recurs after reconnect: escalate.          | `INVALID_REQUEST` means the CN sent a malformed `PublishBlockStream` request that the BN rejected - either the block exceeded the per-block byte ceiling (see C4) or the request was structurally invalid. If the CN reconnects and publishes successfully afterwards, no further action is needed. If `INVALID_REQUEST` repeats on reconnect: the CN is producing a consistently invalid stream. **Escalate to Hashgraph Engineering** with the log excerpt from Phase 1 Step 2 and the CN's software version. Do not restart the BN.                                                                                                                    | ☐    |
| C4 | If `publisher_disconnected_oversize_total` is rising: the CN is sending oversized messages. | The BN enforces a per-block byte ceiling controlled by `server.maxMessageSizeBytes` (default: 131,072,000 bytes, ~125 MB). If a single block's cumulative bytes exceed this ceiling, the BN sends `INVALID_REQUEST` and disconnects the publisher. Check the current limit: `kubectl exec -n $NAMESPACE $RELEASE-block-node-server-0 -- env \| grep -i "SERVER_MAX_MESSAGE_SIZE"`. Check the BN log for `"disconnecting publisher, block N exceeds the cumulative byte ceiling"`. Coordinate with the CN operator on their block item sizes. If the limit needs adjustment: update `server.maxMessageSizeBytes` in the Helm values and upgrade the chart. | ☐    |

---

### Category D - BN persistence failure (PERSISTENCE_FAILED)

`PERSISTENCE_FAILED` means the BN could not write the block to storage. This triggers a BN-wide
broadcast that ends ALL publisher connections simultaneously - never a single-CN event. This is
a BN-side problem - pivot to PB-06 immediately.

| #  |                                 Action                                 |                                                                                                                                                                                                                                                              Notes                                                                                                                                                                                                                                                              | Done |
|----|------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| D1 | Confirm all publishers are affected (expected for PERSISTENCE_FAILED). | `curl -s http://$BN_HOST:16007/metrics \| grep publisher_open_connections`. When the BN encounters a persistence failure it ends ALL publisher connections simultaneously via an internal broadcast - this is always a BN-wide event, not a single-CN event. If all connections dropped simultaneously and `publisher_stream_errors_total` is rising: **Pivot to [PB-06](./06-bn-data-corruption.md)** (Block Node Data Corruption or Block Verification Errors). Bring the log preserved in Phase 2 Step 2 into that incident. | ☐    |
| D2 | Preserve evidence before anything is restarted.                        | `kubectl logs -n $NAMESPACE $RELEASE-block-node-server-0 -c block-node-server \| grep -iE "persist\|storage\|PERSISTENCE_FAILED\|ERROR" > /tmp/bn_persistence_$(date +%Y%m%d_%H%M%S).log`. Share with Hashgraph Engineering alongside the [PB-06](./06-bn-data-corruption.md) handoff. **Do not restart the BN until Engineering advises.**                                                                                                                                                                                     | ☐    |

---

## Phase 4 - Verify and Close

> Publishing is restored when the affected CN(s) are reconnected and ACKs are flowing again.
> Confirm across all key signals before closing the incident.

| # |                               Action                                |                                                                                                                                                                Notes                                                                                                                                                                | Done |
|---|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| 1 | Confirm `publisher_open_connections` is back to the expected count. | `watch -n 10 'curl -s http://$BN_HOST:16007/metrics \| grep publisher_open_connections'`. Should equal the number of active CNs publishing to this BN (7 in a full BNCE environment). Any continued shortfall: return to Phase 1 and reclassify the remaining disconnected CN(s).                                                   | ☐    |
| 2 | Confirm `publisher_blocks_ack_sent` is increasing.                  | `watch -n 10 'curl -s http://$BN_HOST:16007/metrics \| grep publisher_blocks_ack_sent'`. The counter should increment every few seconds as blocks are acknowledged. If it is frozen while `publisher_open_connections` is healthy: check BN verification - if verification is failing, the BN may not be ACKing. See PB-06.         | ☐    |
| 3 | Confirm `publisher_stream_errors` has stopped increasing.           | `curl -s http://$BN_HOST:16007/metrics \| grep publisher_stream_errors_total`. Value should be frozen. Any continued increase: return to Phase 1.                                                                                                                                                                                   | ☐    |
| 4 | Confirm BN block range is advancing.                                | `grpcurl -plaintext -import-path ~/bn-proto -proto block-node/api/node_service.proto -d '{}' $BN_HOST:$BN_STATUS_PORT org.hiero.block.api.BlockNodeService/serverStatus`. `lastAvailableBlock` should increase every ~2 s. If the BN block range is stalled while publishers are connected: check verification metrics - see PB-06. | ☐    |
| 5 | Update the status page and close the incident bridge.               | Confirm with the CN operator that their CN is streaming normally before closing.                                                                                                                                                                                                                                                    | ☐    |

---

## Reference

### Key metrics

|               Metric               |                                    Alert condition                                     |
|------------------------------------|----------------------------------------------------------------------------------------|
| `publisher_open_connections`       | Page if below expected CN count for more than 2 minutes                                |
| `publisher_stream_errors`          | Alert if increasing (indicates publisher-layer rejection)                              |
| `publisher_stall_timeouts_sent`    | Alert if sustained (indicates CNs are stalling mid-block)                              |
| `publisher_block_node_behind_sent` | Alert on any non-zero value (BN is telling CNs they are behind)                        |
| `publisher_blocks_ack_sent`        | Monitor for flatness while open connections are healthy (indicates a block-flow stall) |
| `publisher_disconnected_oversize`  | Alert on any non-zero value (CN message exceeds BN hard size limit)                    |

The raw `/metrics` endpoint at port 16007 exposes metric names without any prefix - use
`publisher_<name>` in `curl | grep` commands. Prometheus adds the `blocknode_` prefix and
`_total` suffix at scrape time. In PromQL, use `blocknode_<metric_name>_total` for counters;
gauges do not get `_total`.

### EndOfStream code quick-reference

|        Code sent by BN         | Category |                                             What it means                                             |
|--------------------------------|----------|-------------------------------------------------------------------------------------------------------|
| `BehindPublisher` (not an EOS) | B        | CN is behind; BN sends the current block number for CN to resume from                                 |
| `DUPLICATE_BLOCK`              | C        | CN sent a block the BN already received - reconnect and skip forward                                  |
| `TIMEOUT`                      | C        | CN stalled mid-block; BN closed the session - reconnect                                               |
| `INVALID_REQUEST`              | C        | Malformed publish request - CN software issue if it recurs                                            |
| `SUCCESS`                      | -        | Normal graceful shutdown of the stream (not a failure)                                                |
| `PERSISTENCE_FAILED`           | D        | BN could not write block to storage - pivot to PB-06                                                  |
| `ERROR`                        | C        | BN processing error for this handler - reconnect; if all handlers drop simultaneously, pivot to PB-06 |
| `BAD_BLOCK_PROOF`              | -        | Block proof failed BN verification - this is a verification issue, see PB-06                          |

### Pre-incident configuration record

Fill this in before you need it.

- BN gRPC port (`BN_GRPC_PORT`): _______ *(BNCE default: 40840)*
- BN status port (`BN_STATUS_PORT`): _______ *(BNCE default: 40982)*
- BN metrics port: _______ *(BNCE default: 16007)*
- BN hostnames: _______ *(BNCE: s01-s03.blk.bnce.dal.lat.ope.eng.hashgraph.io)*
- CN hostnames: _______ *(BNCE: s01-s07.csn.bnce.dal.lat.ope.eng.hashgraph.io)*
- Mirror Node host (`MN_HOST`): _______ *(BNCE: s01.aux.bnce.dal.lat.ope.eng.hashgraph.io)*
- Expected `publisher_open_connections` count (number of CNs publishing to this BN): _______
- Helm chart version currently deployed: _______
- `server.maxMessageSizeBytes` current value: _______ *(default: 131,072,000 bytes)*
- On-call CN operator contact: _______
- On-call Hashgraph Engineering contact: _______

### Post-incident checklist

After the incident is closed: archive logs, run a post-mortem within 48 hours, and address:

|                         Finding                          |                                         Remediation                                          |
|----------------------------------------------------------|----------------------------------------------------------------------------------------------|
| CN fell behind due to restart without state preservation | Ensure CN restarts resume streaming from the last ACKed block, not from genesis              |
| Firewall rule lost after BN host reboot or upgrade       | Add firewall rule persistence to the BN deployment runbook; verify rules after every upgrade |
| CN stall timeouts were frequent before the disconnect    | Investigate CN block production latency; check network throughput between CN and BN          |
| `publisher_disconnected_oversize` triggered disconnect   | Check `server.maxMessageSizeBytes`; coordinate CN block item sizes with the CN team          |
| `INVALID_REQUEST` pointed to CN software bug             | Ensure CN software version is compatible with the current BN; escalate to CN maintainers     |

---

*Playbook version: 1.0 DRAFT*
*Closes: [#3787](https://github.com/hiero-ledger/hiero-block-node/issues/3787)*

[Back to DR Playbook Index](./README.md)
