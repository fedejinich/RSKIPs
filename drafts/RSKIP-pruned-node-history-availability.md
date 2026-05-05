---
rskip: pull_request_number_here
title: Pruned Node History Availability
created: 2026-05-05
author: TBD
purpose: Sca, Usa
layer: Net
complexity: 3
status: Draft
description: Define history-retention profiles, segment pruning, p2p availability advertisement, and JSON-RPC semantics for interoperable Rootstock pruned full nodes.
---

| RSKIP | pull_request_number_here |
| :------------ |:-------------|
| **Title** | Pruned Node History Availability |
| **Created** | 05-MAY-2026 |
| **Author** | TBD |
| **Purpose** | Sca, Usa |
| **Layer** | Net |
| **Complexity** | 3 |
| **Status** | Draft |

## Abstract

This RSKIP specifies how Rootstock clients can implement interoperable pruned full nodes. A pruned full node fully validates and follows the best chain from its local synchronization point, but only retains historical data for explicit block ranges and data segments. Archive nodes remain valid and continue to retain all historical data.

The proposal defines:

* node history profiles;
* segment-based pruning modes;
* the minimum history a pruned node must keep to keep validating new Rootstock blocks;
* a p2p history-availability capability for advertising retained ranges;
* JSON-RPC behavior when requested data has been pruned;
* sync, reorg, import/export, and testing requirements.

This proposal does not change block validity, transaction validity, merged-mining rules, bridge rules, or any other consensus rule. It is a network and node-interface standard. If a future proposal mandates network-wide historical retention limits, changes block/header formats, or changes consensus validation rules, that future proposal must be specified separately.

## Motivation

Rootstock nodes currently need large local storage when they retain the full historical chain, receipts, logs, transaction indexes, and historical state data. Reducing this storage requirement lowers the cost of running validating nodes and makes multi-client implementations easier to operate.

Pruning cannot be treated as a local database optimization only. Once a node may not have old blocks, receipts, logs, transaction lookup entries, or historical state, peers and RPC users need a precise way to know which data is available. Without a common contract:

* peers may request data that a node cannot serve and treat timeouts as peer failures;
* RPC clients cannot distinguish "not found" from "known range pruned";
* snapshot/checkpoint sync cannot safely select peers that have the required historical window;
* archive, pruned, and minimal nodes cannot be mixed predictably in the same network.

Ethereum clients have converged on segment-oriented pruning. For example, Reth defaults to archive mode, exposes pruned/full profiles, and allows independent pruning of segments such as transaction lookup, receipts, account history, storage history, sender recovery, and bodies. Ethereum's EIP-4444 also motivates bounding p2p historical data and highlights that checkpoint/snapshot sync and historical-data preservation must be specified explicitly.

Rootstock needs a Rootstock-specific version of this mechanism because block validation and execution have Rootstock-specific historical dependencies, including the `BLOCKHASH` opcode window, REMASC reward processing, uncles/siblings, and bridge-related indexes.

## Definitions

An **archive node** is a node that retains all canonical historical blocks and all locally indexed historical data that it claims to serve.

A **pruned full node** is a node that fully validates new blocks from its local synchronization point and retains only a configured subset of historical data. A pruned full node is not an archive node.

A **minimal node** is a pruned full node that retains only the data required to keep validating and executing new blocks plus any explicitly configured operational data. It SHOULD NOT advertise archive-like RPC availability.

A **segment** is a class of data that can be retained or pruned independently. Examples include headers, bodies, receipts, logs, transaction lookup, historical state, and bridge indexes.

A **history floor** is the lowest block number for which a node claims a segment is available. Segment floors can differ.

A **history ceiling** is the highest block number for which a node claims a segment is available. For a synced node, this is normally the best block number or a recent safe block number, depending on the segment.

A **retention distance** is a moving window measured backward from the canonical head. A segment with retention distance `N` keeps data for blocks `head - N` through `head`, inclusive, subject to reorg protection.

## Specification

### Node profiles

Clients SHOULD expose the following profiles:

* `archive`: retain all canonical historical data and historical indexes supported by the implementation.
* `full`: retain the execution-safety window and a configurable recent RPC window. This is a pruned validating node profile.
* `minimal`: retain only the execution-safety window and data needed for local operation.
* `custom`: retain data according to segment-specific settings.

The profile name is not a consensus property. A node's actual advertised segment availability is authoritative.

### Segment pruning model

Clients SHOULD support segment-oriented retention. At minimum, implementations SHOULD model the following segments, even if some are aliases in a particular database layout:

| Segment id | Name | Description |
| --- | --- | --- |
| 0 | `headers` | Canonical block headers and uncle headers required for serving block/header queries. |
| 1 | `bodies` | Block transactions, uncle lists, and body-level data. |
| 2 | `receipts` | Transaction receipts by block and by transaction. |
| 3 | `transaction_lookup` | Transaction hash to block location indexes. |
| 4 | `logs` | Expanded log data and log indexes. |
| 5 | `logs_bloom` | Bloom or acceleration data for log filtering. |
| 6 | `state_roots` | Historical block-to-state-root mapping. |
| 7 | `account_history` | Account history indexes. |
| 8 | `storage_history` | Storage-slot history indexes. |
| 9 | `trie_nodes_by_root` | Historical trie nodes needed to reconstruct state at old roots. |
| 10 | `bridge_state` | Bridge/PowPeg state snapshots or indexes retained outside current state. |
| 11 | `bridge_events` | Bridge/PowPeg event indexes and projections. |
| 12 | `sender_recovery` | Derived sender data if stored separately. |

Unknown segment ids MUST be ignored by receivers.

Each segment MUST be configured as one of:

* `archive`: retain all data for the segment.
* `distance(N)`: retain a moving window of at least `N` blocks.
* `before(B)`: prune data before block `B` and retain data from `B` onward.
* `drop`: do not retain this segment except where needed for consensus execution or local safety.

Implementations MAY expose additional segment modes, but their p2p advertisement MUST be expressible as concrete available ranges.

### Execution-safety window

A pruned full node MUST retain enough recent data to validate and execute new blocks without relying on historical RPC or archive peers during normal operation.

For Rootstock, the execution-safety window MUST be at least:

```
max(BLOCKHASH_WINDOW, REMASC_MATURITY(network))
```

where `BLOCKHASH_WINDOW` is 256 blocks and `REMASC_MATURITY(network)` is the network-specific REMASC maturity. As of the RSKj master branch checked on 2026-05-05, `REMASC_MATURITY(mainnet)` and `REMASC_MATURITY(fallbackmain)` are 4000, `REMASC_MATURITY(testnet)` and `REMASC_MATURITY(testnet2)` are 50, and `REMASC_MATURITY(regtest)` is 10.

The execution-safety window must include the canonical headers, canonical block bodies, and uncle/sibling headers required by REMASC and block validation. A node MAY retain a larger window for reorg safety, mining, bridge indexing, local snapshots, or operator policy.

A node MUST NOT advertise itself as a pruned full node if it has pruned below the execution-safety window and cannot validate the next block without external history.

### Reorg protection

Pruning MUST be delayed by a reorg protection depth. A node MUST retain all segments needed to unwind and re-execute any block above its pruning floor. If a chain reorganization requires data below a segment floor, the node MUST stop, resynchronize from a valid checkpoint, or obtain the missing data from an archive source before continuing.

A node MUST NOT silently continue execution with missing historical state, missing parent data, missing uncle data, or missing receipts needed by its local indexes.

### P2P history availability capability

This RSKIP defines an optional RLPx capability named `rskh`, version `1`.

Peers MUST NOT send `rskh` messages unless both peers negotiated the `rskh/1` capability. The existing `rsk` status message MUST NOT be extended with additional fields by this RSKIP.

The first message sent by a peer that negotiated `rskh/1` SHOULD be `HistoryStatus`.

#### `HistoryStatus`

```
HistoryStatus = [
  version,
  chainId,
  genesisHash,
  bestBlockNumber,
  bestBlockHash,
  profile,
  segments
]

SegmentAvailability = [
  segmentId,
  mode,
  lowestAvailableBlock,
  highestAvailableBlock,
  flags
]
```

Field definitions:

* `version`: unsigned integer. MUST be `1`.
* `chainId`: Rootstock chain id.
* `genesisHash`: hash of the genesis block for the advertised chain.
* `bestBlockNumber`: peer's current best block number.
* `bestBlockHash`: peer's current best block hash.
* `profile`: unsigned integer. `0 = archive`, `1 = full`, `2 = minimal`, `255 = custom`.
* `segments`: list of `SegmentAvailability`.
* `segmentId`: unsigned integer from the segment table above.
* `mode`: unsigned integer. `0 = unavailable`, `1 = archive`, `2 = range`, `3 = current-only`, `255 = implementation-specific`.
* `lowestAvailableBlock`: lowest block number served for this segment.
* `highestAvailableBlock`: highest block number served for this segment.
* `flags`: bitfield reserved for future use. Unknown flags MUST be ignored.

A node that advertises `mode = archive` for a segment MUST serve that segment from genesis through `highestAvailableBlock`, subject to normal fork choice and canonicality rules.

A node that advertises `mode = range` for a segment MUST serve that segment for every canonical block in `[lowestAvailableBlock, highestAvailableBlock]`, unless the data is absent because the request is for a non-canonical block or an unknown hash.

A node SHOULD update or resend `HistoryStatus` when a segment floor advances.

#### Requests outside advertised ranges

A peer SHOULD NOT request data from another peer outside the advertised range for the corresponding segment.

If a peer that negotiated `rskh/1` requests data outside an advertised range using the existing `rsk` subprotocol, the responder SHOULD send `HistoryUnavailable` on `rskh/1` if it can map the request to a segment and request id. The responder SHOULD NOT penalize the requester unless the requester repeatedly ignores advertised ranges.

```
HistoryUnavailable = [
  requestProtocol,
  requestCode,
  requestId,
  segmentId,
  requestedBlock,
  lowestAvailableBlock,
  highestAvailableBlock
]
```

`requestProtocol` is an ASCII byte string such as `rsk`. `requestCode` is the message code in that protocol. `requestId` is the request id when the original message has one, or `0` otherwise.

Old peers that do not negotiate `rskh/1` are unaffected. They may still receive no response for unavailable history, matching current behavior.

### Peer selection

A syncing node SHOULD prefer peers that advertise the segments required by the selected sync strategy.

Genesis full sync requires peers that can serve headers and bodies from genesis, or out-of-band historical data import.

Checkpoint or snapshot sync requires peers that can serve:

* the checkpoint/snapshot state;
* the validation headers and blocks needed to verify the checkpoint according to the client's trust model;
* all data from the checkpoint forward to the current tip;
* the execution-safety window at the resulting head.

The checkpoint trust model is out of scope for this RSKIP. Clients MUST document whether a checkpoint is trusted, weakly trusted, or fully verified from genesis or from another local trust anchor.

### JSON-RPC availability

Clients MUST expose a machine-readable way for local users and applications to inspect history availability. This RSKIP defines:

```
rsk_getHistoryStatus() -> {
  "profile": string,
  "bestBlockNumber": quantity,
  "bestBlockHash": data32,
  "segments": [
    {
      "name": string,
      "mode": string,
      "lowestAvailableBlock": quantity,
      "highestAvailableBlock": quantity
    }
  ]
}
```

For JSON-RPC methods that explicitly identify a block number, block hash, or block range, clients MUST NOT silently return a successful empty result when the requested block or range is outside an advertised local segment floor and the client can determine that the miss is caused by pruning.

Instead, clients SHOULD return a JSON-RPC error with implementation-defined code in the server-error range and an error object whose `data.reason` is `history_pruned`:

```
{
  "code": -32001,
  "message": "requested history has been pruned",
  "data": {
    "reason": "history_pruned",
    "segment": "receipts",
    "requestedBlock": "0x1234",
    "lowestAvailableBlock": "0x2000",
    "highestAvailableBlock": "0x3000"
  }
}
```

For hash-only methods where the node cannot distinguish an invalid hash from a valid hash whose index has been pruned, clients MAY return `null` as they do for not-found data. Clients SHOULD document this ambiguity in `rsk_getHistoryStatus`.

The following RPC behavior is recommended:

| Method family | Required segment(s) | Outside retained range |
| --- | --- | --- |
| `eth_getBlockByNumber`, `eth_getBlockByHash`, block transaction count, uncle queries | `headers`, `bodies` | error if explicit block/range is known pruned; `null` if unknown hash ambiguity remains |
| `eth_getTransactionByHash` | `transaction_lookup`, `bodies` | `null` if hash-only ambiguity remains; error if transaction location is known but body was pruned |
| `eth_getTransactionReceipt` | `transaction_lookup`, `receipts` | `null` if hash-only ambiguity remains; error if transaction location is known but receipt was pruned |
| `eth_getLogs` | `logs`, optionally `logs_bloom` | error if range intersects a pruned interval |
| `eth_call`, `eth_getBalance`, `eth_getCode`, `eth_getStorageAt` at historical block | `trie_nodes_by_root`, `state_roots` | error if state at requested block was pruned |
| same state methods at `latest` or equivalent | current state | allowed if current state is available |

### Import and export

Clients SHOULD support importing and exporting historical data by segment. A node MAY become more capable after importing a segment, but it MUST update its local history status before advertising the new range.

Historical data imported out of band MUST be verified against canonical block hashes, receipt roots, state roots, and other relevant commitments before it is served to peers or RPC users.

### Configuration

The exact configuration format is client-specific. The following TOML shape is recommended:

```toml
[history]
profile = "full"
prune_interval = 5
reorg_protection_depth = 256

[history.segments]
headers = "archive"
bodies = { distance = 10000 }
receipts = { distance = 10000 }
transaction_lookup = { distance = 10000 }
logs = { distance = 10000 }
logs_bloom = { distance = 10000 }
state_roots = { distance = 10000 }
trie_nodes_by_root = { distance = 10000 }
bridge_state = { distance = 10000 }
bridge_events = { distance = 10000 }
sender_recovery = "drop"
account_history = { distance = 10000 }
storage_history = { distance = 10000 }
```

Clients SHOULD refuse configurations that prune below the execution-safety window unless the node is explicitly configured as a non-validating archival-data appliance or other non-full-node mode.

## Rationale

This RSKIP uses segment-based pruning instead of a single global block range because different APIs and sync flows require different data. A node may retain headers but prune receipts; retain receipts but prune transaction lookup; or retain current state while pruning historical trie nodes.

The existing Rootstock `StatusMessage` is intentionally not extended. Current clients have historically accepted two-field and four-field status payloads. Adding more fields risks disconnects or decode failures in clients that validate the exact shape. A separate `rskh/1` capability provides explicit opt-in versioning and keeps old peers compatible.

The proposal does not require a hard fork because it does not change consensus validity. It standardizes how nodes describe and expose local data availability. A node that prunes too aggressively may fail to validate or serve data, but that is a local conformance failure, not a chain rule change.

The execution-safety window is Rootstock-specific. Ethereum-style retention windows are not sufficient by themselves because Rootstock execution currently depends on recent block hashes and REMASC historical data. Implementations can retain more than the minimum, and most production nodes should do so.

## Backward compatibility

This RSKIP is backward compatible with existing consensus rules and existing p2p peers.

Old peers that do not support `rskh/1` continue to use the existing `rsk` capability. New nodes MUST NOT send `rskh/1` messages to old peers.

Existing archive nodes remain valid. Existing full-sync-from-genesis workflows remain valid when connected to archive peers or when historical data is imported out of band.

RPC applications that assume every endpoint is archive-backed may need to inspect `rsk_getHistoryStatus` or handle `history_pruned` errors.

## Implementation notes

RSKj currently has snapshot sync and state garbage collection mechanisms, but those mechanisms do not constitute a complete pruned full node contract. RSKj would need public history/pruning configuration, physical pruning across historical segments, p2p `rskh/1` support, range-aware peer selection, explicit RPC errors, and tests for archive/pruned interop.

The current RSKj p2p status message advertises the best block when the block store begins at genesis, but falls back to genesis status when the local block store minimum is not zero. This behavior is not a substitute for advertising retained history ranges.

`rsk_rs` already has storage-level pruning types and metadata, but at the time of this draft its pruning path records the last pruned block without physically deleting all segment data. A conforming implementation would need physical segment pruning, public profile/config support, p2p `rskh/1`, RPC range semantics, and reorg/checkpoint handling.

## Test cases

Implementations SHOULD include at least the following tests:

1. `archive` node advertises each supported segment from genesis.
2. `full` pruned node advances segment floors as the head advances.
3. Two peers without `rskh/1` interoperate exactly as before.
4. A peer with `rskh/1` does not request headers, bodies, receipts, or logs outside advertised ranges.
5. A `rskh/1` peer that receives an out-of-range request returns `HistoryUnavailable` when possible.
6. `eth_getLogs` over a range below the logs floor returns `history_pruned`.
7. Historical `eth_call` below the historical-state floor returns `history_pruned`.
8. Hash-only transaction lookup remains `null` when the node cannot distinguish invalid hash from pruned lookup.
9. A reorg within the reorg protection depth succeeds.
10. A reorg below the local pruning floor forces resync, checkpoint rollback, or safe shutdown.
11. A node cannot start in pruned full mode with a retained window below the execution-safety window.
12. Imported historical data is verified before being advertised or served.

## Security considerations

Pruning reduces storage but increases dependence on archive nodes, snapshots, checkpoints, or out-of-band historical data sources. The network should preserve independently operated archive nodes and historical-data distribution mechanisms.

A malicious peer can advertise ranges that it does not serve. Clients SHOULD score peers for repeated violations of advertised availability, but MUST NOT penalize peers for refusing data outside their advertised ranges.

Checkpoint and snapshot sync may introduce trust assumptions. Clients MUST make those assumptions explicit to operators and MUST verify every imported block, receipt, and state commitment that can be verified from local trust anchors.

RPC users may otherwise mistake pruned data for absent data. This RSKIP requires explicit `history_pruned` errors for requests where pruning can be identified.

## Open questions

1. Should `rskh/1` define only availability advertisement, or should it also define new range-aware block/header/body/receipt request messages instead of relying on the existing `rsk` request messages?
2. Should Rootstock standardize a default `full` retention distance, or should each client choose a default above the execution-safety minimum?
3. Should the recommended RPC error code be fixed to `-32001`, or should only `data.reason = history_pruned` be normative?
4. Should bridge/federator operational indexes be mandatory segments, or implementation-specific extensions?
5. Should Rootstock specify a standard checkpoint object for pruned bootstrap in this RSKIP or in a separate RSKIP?

## References

[1] RSKIP64: Garbage Collector for State Pruning, https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP64.md

[2] RSKIP85: Improvements to REMASC contract, https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP85.md

[3] EIP-4444: Bound Historical Data in Execution Clients, https://eips.ethereum.org/EIPS/eip-4444

[4] Reth pruning documentation, https://reth.rs/run/faq/pruning/

[5] Reth configuration documentation, https://reth.rs/run/configuration/#the-prune-section

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
