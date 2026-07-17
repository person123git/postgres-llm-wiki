---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-17T16:13:56Z
---

# How wal_sender_timeout Is Used and What It Impacts in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [GUC definition, v12 apply scope, and live reload](#guc-definition-v12-apply-scope-and-live-reload)
  - [Activation and call boundaries](#activation-and-call-boundaries)
  - [State and what counts as a reply](#state-and-what-counts-as-a-reply)
  - [Scheduling, keepalives, and termination](#scheduling-keepalives-and-termination)
  - [Physical replication and streaming clients](#physical-replication-and-streaming-clients)
  - [Logical replication impact](#logical-replication-impact)
  - [Operational impacts](#operational-impacts)
  - [What it does not do](#what-it-does-not-do)
  - [Build, generated artifacts, and extension boundaries](#build-generated-artifacts-and-extension-boundaries)
  - [Tests and review reproduction](#tests-and-review-reproduction)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, give a comprehensive explanation of how wal_sender_timeout is used and what it impacts. Does it impact logical replication?

Prompt note: the original request had typos and grammar issues. The user approved correcting the prompt before filing this page.

## Answer

Yes. `wal_sender_timeout` affects logical replication in PostgreSQL 12. It runs in the **publisher-side WAL sender process** after a client starts `START_REPLICATION ... LOGICAL`. The logical sender uses the same `WalSndLoop()` reply tracking, keepalive, and disconnect code as a physical sender.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1058-L1141) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2275)

The setting measures **receiver silence**, not replication lag and not an absence of new WAL. Each valid receiver message resets a sender-local clock even when the reported write, flush, and apply LSNs do not advance. If no such message arrives before the current timeout deadline, the sender logs `terminating walsender process due to replication timeout`, closes the connection without sending that error to the receiver, and exits.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1596-L1699) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148)

### Short answer

| Question | PostgreSQL 12 answer |
| --- | --- |
| Default and disable value | `60 * 1000` milliseconds, documented as 60 seconds; `0` disables timeout-driven keepalives and termination.[guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665) [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737) |
| Apply scope | `PGC_USERSET`: no restart; `SET` is session-scoped and `SET LOCAL` is transaction-scoped. A configuration-file change is reloadable and reaches an existing sender unless a higher-priority startup-client or session value overrides it.[guc.h#GucContext](../../../raw/postgres-12/src/include/utils/guc.h#L50-L77) [guc.h#GucSource](../../../raw/postgres-12/src/include/utils/guc.h#L79-L121) [guc.c#source-priority](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6916-L6941) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2166-L2182) |
| When is it active? | Only while `WalSndLoop()` is streaming after physical or logical `START_REPLICATION`; it is not a general lifetime limit for a replication connection.[walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1518-L1576) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2166) |
| What resets it? | A valid `CopyData` status (`r`) or hot-standby-feedback (`h`) message, or `CopyDone`. The receiver-provided timestamp and LSN progress are not the timeout clock.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1601-L1698) [walsender.c#ProcessStandbyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1701-L1730) |
| When is a reply requested? | Normally at half the timeout if no earlier heartbeat is outstanding. Logical WAL waits and shutdown have additional keepalive branches, so “always exactly at half-time” is not correct.[walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2068-L2111) [walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1358-L1370) [walsender.c#WalSndDone](../../../raw/postgres-12/src/backend/replication/walsender.c#L2883-L2922) |
| Does it affect built-in logical subscriptions? | Yes. It covers the publisher stream used by the main apply worker and any table-synchronization worker that enters logical catch-up streaming.[worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1769) [tablesync.c#LogicalRepSyncTableStart](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L884-L958) |
| Is it the subscriber receive timeout? | No. `wal_receiver_timeout`, `wal_receiver_status_interval`, and `wal_retrieve_retry_interval` are subscriber-side controls and also apply to logical workers.[worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1289-L1341) [config.sgml#logical-subscriber-settings](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4297-L4310) |

### GUC definition, v12 apply scope, and live reload

The v12 GUC table defines `wal_sender_timeout` as `PGC_USERSET` with millisecond units, a `60 * 1000` default, a range from `0` through `INT_MAX`, and no check, assign, or show hook. `guc.c` reaches the process-global variable through the ordinary `extern` declaration in `walsender.h`; there is no generated GUC definition for this setting.[guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665) [walsender.h#wal_sender_timeout](../../../raw/postgres-12/src/include/replication/walsender.h#L29-L42) [walsender.c:119](../../../raw/postgres-12/src/backend/replication/walsender.c#L119-L124)

`PGC_USERSET` means no restart is required. Any user can set a session value with `SET`; `SET LOCAL` lasts only through the current transaction. `START_REPLICATION` is forbidden inside a transaction block, so a transaction-local value cannot carry into streaming; useful per-connection choices are a session value established before logical streaming or a startup `options=-c ...` value.[guc.h#GucContext](../../../raw/postgres-12/src/include/utils/guc.h#L50-L77) [ref/set.sgml#SET-scope](../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L39-L58) [walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1537-L1576) A value in `postgresql.conf` requires a reload to reach existing processes. User-context file changes affect an existing session only when no session-local value is established, and the GUC source ordering also gives startup-client and session values higher priority than file values.[catalogs.sgml#context-user](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10601-L10610) [guc.h#GucSource](../../../raw/postgres-12/src/include/utils/guc.h#L79-L121) [guc.c#source-priority](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6916-L6941)

An active sender processes `SIGHUP` in the main loop, while blocked writing logical output, and while waiting for WAL. It calls `ProcessConfigFile(PGC_SIGHUP)` in all three places. Because every deadline is recomputed as `last_reply_timestamp + current wal_sender_timeout`, lowering a file-sourced value below the already-silent interval can terminate the connection at the next check; increasing it moves that deadline later, and changing it to `0` disables the check.[walsender.c#WalSndWriteData](../../../raw/postgres-12/src/backend/replication/walsender.c#L1212-L1256) [walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1314-L1329) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2166-L2182) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2148)

PostgreSQL 12 specifically changed this GUC from cluster-wide-only to per-connection settable. The v12 release note identifies commit `db361db2f` and the new per-connection behavior.[release-12.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L4519-L4534) A physical standby can pass a connection value through `primary_conninfo`, using libpq's `options=-c ...` startup parameter. The v12 high-availability documentation gives `options='-c wal_sender_timeout=5000'` as its example.[high-availability.sgml#primary_conninfo-options](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L708-L742) [libpq.sgml#options](../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L1153-L1167)

The same mechanism is available to a logical subscription. `CREATE SUBSCRIPTION ... CONNECTION` stores a libpq connection string; the apply worker passes it to `walrcv_connect()`, and `libpqwalreceiver` expands it while adding `replication=database`. Therefore an `options=-c wal_sender_timeout=...` connection parameter sets the value in the publisher-side WAL sender for that subscription connection.[create_subscription.sgml#connection](../../../raw/postgres-12/doc/src/sgml/ref/create_subscription.sgml#L72-L80) [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1684-L1745) [libpqwalreceiver.c#libpqrcv_connect](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L116-L163)

### Activation and call boundaries

A WAL sender is a backend-like process dedicated to one replication connection. Before streaming, it accepts replication commands rather than applying this setting as a connection-wide idle timeout.[walsender.c#overview](../../../raw/postgres-12/src/backend/replication/walsender.c#L3-L25) `exec_replication_command()` parses the command and dispatches a physical `StartReplicationCmd` to `StartReplication()` and a logical one to `StartLogicalReplication()`.[walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1468-L1594) The generated replication parser constructs the same `StartReplicationCmd` node for both grammar forms; the node carries the replication kind, slot, timeline, start LSN, and logical options.[repl_gram.y#start_replication](../../../raw/postgres-12/src/backend/replication/repl_gram.y#L295-L324) [replnodes.h#StartReplicationCmd](../../../raw/postgres-12/src/include/nodes/replnodes.h#L75-L87)

The physical path enters copy-both mode, initializes `sentPtr` and synchronous-replication state, and calls `WalSndLoop(XLogSendPhysical)`. The logical path acquires the logical slot, creates a decoding context with WAL-sender read/write/progress callbacks, enters copy-both mode, initializes positions, and calls `WalSndLoop(XLogSendLogical)`.[walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L533-L755) [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1058-L1141) `WalSndLoop()` then initializes `last_reply_timestamp`, which is the point at which timeout processing becomes active.[walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2166)

The boundary excludes several operations that also use replication connections:

- `BASE_BACKUP` dispatches to `SendBaseBackup()` and never enters `WalSndLoop()`. With `pg_basebackup -X stream`, only the **separate background WAL-streaming connection** is covered; the main base-backup copy is not.[walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1518-L1548) [pg_basebackup.c#StartLogStreamer](../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L460-L566)
- Logical `CREATE_REPLICATION_SLOT` explicitly sets `last_reply_timestamp = 0` before `DecodingContextFindStartpoint()`, because that phase cannot yet accept feedback or send keepalives.[walsender.c#CreateReplicationSlot](../../../raw/postgres-12/src/backend/replication/walsender.c#L941-L957)
- A logical table-synchronization worker first creates a temporary slot and performs the initial `COPY ... TO STDOUT` through ordinary `walrcv_exec()` calls. The timeout applies only if that worker later leaves `LogicalRepSyncTableStart()` and enters the normal logical catch-up stream in `ApplyWorkerMain()`.[tablesync.c#copy_table](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L745-L797) [tablesync.c#LogicalRepSyncTableStart](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L884-L958) [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1690-L1766)
- SQL logical-decoding functions such as `pg_logical_slot_get_changes()` use a local WAL reader and tuplestore writer. They do not enter the replication-protocol `WalSndLoop()`, so `wal_sender_timeout` does not limit those SQL calls.[logicalfuncs.c#pg_logical_slot_get_changes_guts](../../../raw/postgres-12/src/backend/replication/logical/logicalfuncs.c#L125-L168) [logicalfuncs.c#CreateDecodingContext](../../../raw/postgres-12/src/backend/replication/logical/logicalfuncs.c#L238-L317) [logicalfuncs.c#SQL-entry-points](../../../raw/postgres-12/src/backend/replication/logical/logicalfuncs.c#L370-L404)

### State and what counts as a reply

The timeout state is process-local, not shared. `last_processing` records when the sender last entered `ProcessRepliesIfAny()`. `last_reply_timestamp` records the last such call that saw a valid message. `waiting_for_ping_response` records whether a heartbeat is already outstanding.[walsender.c#reply-state](../../../raw/postgres-12/src/backend/replication/walsender.c#L159-L174)

`ProcessRepliesIfAny()` drains all immediately available frontend messages. A `CopyData` message counts after `ProcessStandbyMessage()` accepts its payload; `CopyDone` also counts and starts orderly copy shutdown. `Terminate` exits immediately. EOF, a message after `CopyDone`, an invalid frontend type, or an invalid inner standby type exits through a communication or protocol error and does not become a successful reply.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1596-L1699) [walsender.c#ProcessStandbyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1701-L1730)

The two accepted `CopyData` payload types are:

- `r`, a standby status update. It carries write, flush, apply, client-send-time, and reply-request fields. The sender updates shared positions and lag, may answer the receiver's ping, may release synchronous-replication waiters, and may advance a physical or logical replication slot.[walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1764-L1868)
- `h`, hot-standby feedback. It also counts for timeout purposes. Its handler updates the shared monitoring `replyTime` and may update physical xmin horizons, but it does not update write, flush, or apply positions.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1651-L1698) [walsender.c#ProcessStandbyHSFeedbackMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1949-L2066)

After at least one valid message, the sender sets `last_reply_timestamp = last_processing`; it does **not** use the timestamp embedded by the receiver. Consequently, unchanged LSNs still keep the connection alive, and `wal_sender_timeout` cannot enforce forward progress.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1601-L1608) [walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1691-L1698)

`WalSnd` is a separate shared-memory structure used by monitoring and synchronous replication. It stores PID, sender state, sent/write/flush/apply LSNs, lag values, latch, synchronous priority, and `replyTime`; `WalSndCtlData` owns the cluster-wide sender array and synchronous wait queues.[walsender_private.h#WalSnd](../../../raw/postgres-12/src/include/replication/walsender_private.h#L31-L83) [walsender_private.h#WalSndCtlData](../../../raw/postgres-12/src/include/replication/walsender_private.h#L87-L112) `replyTime` is the receiver-supplied send time exposed as `pg_stat_replication.reply_time`, not the process-local timeout clock. It therefore is not an exact countdown to disconnection.[walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1783-L1789) [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1835-L1853) [monitoring.sgml#reply_time](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1969-L1983)

### Scheduling, keepalives, and termination

The normal idle algorithm is:

1. `WalSndLoop()` processes replies, invokes the physical or logical send callback, flushes output, checks the full timeout, then considers sending a keepalive.[walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2166-L2242)
2. If no heartbeat is outstanding, `WalSndComputeSleeptime()` schedules the next wakeup at `last_reply_timestamp + wal_sender_timeout / 2`; otherwise it schedules the full deadline.[walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111)
3. At the half-time point, `WalSndKeepaliveIfNecessary()` sends a `k` keepalive with `requestReply = 1`, marks a response outstanding, and tries to flush it. The protocol tells the client to reply as soon as possible to avoid disconnection.[walsender.c#WalSndKeepalive](../../../raw/postgres-12/src/backend/replication/walsender.c#L3400-L3419) [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454) [protocol.sgml#primary-keepalive](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2144-L2196)
4. At the full deadline, `WalSndCheckTimeOut()` logs at `COMMERROR` and calls `WalSndShutdown()`. `WalSndShutdown()` suppresses further protocol output and exits, so the receiver observes a closed connection rather than the server's timeout text.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2148) [walsender.c#WalSndShutdown](../../../raw/postgres-12/src/backend/replication/walsender.c#L324-L339)

There are two important exceptions to the simplified “ping exactly at half-time” description. While logical decoding waits for more flushed WAL, `WalSndWaitForWal()` can send an earlier non-requesting keepalive containing the current position and still set `waiting_for_ping_response = true`; that outstanding flag suppresses the normal half-time requesting keepalive. During final server shutdown, `WalSndDone()` can instead request a reply immediately when no heartbeat is already outstanding.[walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1358-L1370) [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3433-L3449) [walsender.c#WalSndDone](../../../raw/postgres-12/src/backend/replication/walsender.c#L2883-L2922)

Logical output also has a fast and slow network-write path. `WalSndWriteData()` uses the fast path only while it is earlier than the half-time point and no output remains buffered; otherwise it processes replies, checks timeout, sends keepalives, and waits for socket readiness. This keeps timeout work reachable while a large decoded transaction emits many messages.[walsender.c#WalSndWriteData](../../../raw/postgres-12/src/backend/replication/walsender.c#L1171-L1260)

The check deliberately uses `last_processing`, a timestamp captured before the sender performs more work, rather than taking a fresh timestamp inside `WalSndCheckTimeOut()`. This avoids charging a server-side stall directly to the receiver. The source also records the remaining edge: after a long stall, the eventual keepalive can leave the receiver almost no time to answer before the next check.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2123) Timeout enforcement is cooperative rather than an independent timer interrupt: the checks occur in `WalSndLoop()`, `WalSndWriteData()`, and `WalSndWaitForWal()`, so code blocked elsewhere delays the check.[walsender.c#WalSndWriteData](../../../raw/postgres-12/src/backend/replication/walsender.c#L1212-L1238) [walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1394-L1418) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2238-L2273)

Socket EOF, protocol violations, and output-flush failures can close the same sender independently of `wal_sender_timeout`. When server logging captures `COMMERROR`, the timeout's specific log text distinguishes this path from those other replication disconnects.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1611-L1636) [walsender.c#WalSndShutdown-callers](../../../raw/postgres-12/src/backend/replication/walsender.c#L1195-L1207) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2193-L2206)

### Physical replication and streaming clients

Physical streaming reaches the timeout through `StartReplication()` and `WalSndLoop(XLogSendPhysical)`, whether the sender is a primary or a cascading standby.[walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L533-L755) `XLogSendPhysical()` reads safely flushed WAL, sends `w` messages, advances `sentPtr`, and publishes that position in `MyWalSnd`.[walsender.c#XLogSendPhysical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2538-L2806)

A PostgreSQL physical walreceiver normally prevents sender timeout. It sends periodic `r` status updates, sends progress after receiving data, requests a sender reply at half of its separate `wal_receiver_timeout`, and sends a forced status reply when a sender keepalive has `replyRequested = true`.[walreceiver.c#receive-loop](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L384-L451) [walreceiver.c#receiver-timeout](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L501-L549) [walreceiver.c#keepalive-reply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L857-L877) Even when `wal_receiver_status_interval = 0` suppresses ordinary reports, a forced keepalive response still sends.[walreceiver.c#XLogWalRcvSendReply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L1045-L1115)

`pg_receivewal` and the `-X stream` background connection of `pg_basebackup` use the shared `ReceiveXlogStream()` client path. Both default to a 10-second status interval; the shared keepalive handler replies immediately when requested.[pg_receivewal.c#defaults](../../../raw/postgres-12/src/bin/pg_basebackup/pg_receivewal.c#L32-L47) [pg_basebackup.c#defaults](../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L85-L105) [receivelog.c#START_REPLICATION](../../../raw/postgres-12/src/bin/pg_basebackup/receivelog.c#L557-L580) [receivelog.c#ProcessKeepaliveMsg](../../../raw/postgres-12/src/bin/pg_basebackup/receivelog.c#L977-L1031) A timeout can therefore affect standbys, cascading receivers, `pg_receivewal`, and the WAL-streaming side connection of `pg_basebackup`; it does not time out the main `BASE_BACKUP` copy.[pg_basebackup.c#LogStreamerMain](../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L470-L525) [pg_basebackup.c#StartLogStreamer](../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L528-L566)

### Logical replication impact

The main logical apply worker reads the subscription, connects with `replication=database`, builds options containing its slot, protocol version, and publication list, and calls `walrcv_startstreaming()`.[worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1766) `libpqwalreceiver` constructs `START_REPLICATION SLOT "..." LOGICAL ...`, expects copy-both mode, and thereby dispatches the publisher into `StartLogicalReplication()` and `WalSndLoop(XLogSendLogical)`.[libpqwalreceiver.c#libpqrcv_startstreaming](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L359-L449) [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1058-L1141)

The subscriber sends the messages that keep that publisher connection alive. `LogicalRepApplyLoop()` processes publisher `w` data and `k` keepalives, forces feedback when a keepalive requests it, and also calls `send_feedback()` after draining input and on timeout wakeups.[worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1122-L1249) [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1265-L1341) `send_feedback()` emits an `r` status update containing receive, flush, apply, send-time, and reply-request fields. `wal_receiver_status_interval` controls ordinary periodic reports, while a forced call bypasses a disabled interval.[worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1345-L1435)

The subscriber-side settings remain independent:

- `wal_receiver_timeout` terminates a logical worker that receives nothing from the publisher and requests a publisher reply at half its own interval.
- `wal_receiver_status_interval` controls ordinary feedback frequency.
- `wal_retrieve_retry_interval` limits how frequently the launcher starts a missing apply worker and how frequently a failed table-synchronization worker is relaunched.

All three are reloadable `PGC_SIGHUP` standby settings in v12, not publisher-side substitutes for `wal_sender_timeout`.[guc.c#wal_receiver_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2107-L2126) [guc.c#wal_retrieve_retry_interval](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2911-L2922) [launcher.c#ApplyLauncherMain](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L995-L1066) [tablesync.c#process_syncing_tables](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L312-L321) [tablesync.c:506](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L506-L517)

`pg_recvlogical` is affected for the same reason. It starts `START_REPLICATION SLOT ... LOGICAL`, sends `r` feedback on its status interval, responds to requesting keepalives, and contains an explicit forced-feedback path because otherwise `wal_sender_timeout` can kill its connection.[pg_recvlogical.c#StreamLogicalLog](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L202-L312) [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164) [pg_recvlogical.c#keepalive-reply](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L442-L487)

Custom logical protocol clients and custom output plugins have the same transport boundary. `StartLogicalReplication()` places the output plugin behind WAL-sender read/write/progress callbacks, while timeout handling remains in the core sender loops around those callbacks.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1086-L1129) [walsender.c#XLogSendLogical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2808-L2880) Core timeout handling does not change which changes a plugin emits; it requires the protocol client consuming those changes to continue sending valid replies.

### Operational impacts

1. **Failure detection and WAL-sender capacity.** A smaller value detects a silent client or broken path sooner; a larger value tolerates longer stalls. The documentation recommends choosing by network location and latency.[config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3721-L3737) Timeout exit clears the sender's `WalSnd` slot, so it also releases one of the finite `max_wal_senders` process slots. The `max_wal_senders` documentation explicitly cites timeout as the mechanism that eventually clears an orphaned connection after an abrupt client disconnection.[config.sgml#max_wal_senders](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3592-L3618) [walsender.c#WalSndKill](../../../raw/postgres-12/src/backend/replication/walsender.c#L2332-L2352)

2. **Reconnect churn and recovery delay.** A physical standby notices that streaming ended, cycles through its WAL sources, and retries after `wal_retrieve_retry_interval` when no source provides WAL.[xlog.c#WaitForWALToBecomeAvailable](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11748-L11757) [xlog.c#stream-retry](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11878-L11921) A logical worker exits when its publisher stream fails; its exit callback clears the worker slot and wakes the launcher, which starts missing workers no more often than `wal_retrieve_retry_interval`.[launcher.c#logicalrep_worker_onexit](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L716-L725) [launcher.c#ApplyLauncherMain](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L995-L1066)

3. **Monitoring and diagnosis.** `pg_stat_replication` has one row per active sender and exposes sender state, LSNs, lag, sync state, and the receiver-supplied `reply_time`.[system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782) [walsender.c#pg_stat_get_wal_senders](../../../raw/postgres-12/src/backend/replication/walsender.c#L3226-L3398) Timeout exit sets the `WalSnd` PID to zero, so the row disappears; when emitted, the specific server log message records the direct cause.[walsender.c#WalSndKill](../../../raw/postgres-12/src/backend/replication/walsender.c#L2336-L2352) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2137-L2148) Neither `last_reply_timestamp` nor the computed deadline appears in the view.[walsender_private.h#WalSnd](../../../raw/postgres-12/src/include/replication/walsender_private.h#L40-L83)

4. **Synchronous replication.** An `r` reply updates write, flush, and apply positions, then calls `SyncRepReleaseWaiters()` for a non-cascading sender. That function wakes commit queues only when enough eligible synchronous standbys have reached the required position.[walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1835-L1868) [syncrep.c#SyncRepReleaseWaiters](../../../raw/postgres-12/src/backend/replication/syncrep.c#L411-L514) Timeout shutdown itself does not call `SyncRepReleaseWaiters()`. If the timed-out connection was the only required synchronous standby, waiting commits can remain blocked until another eligible standby reports enough progress, the synchronous requirement is removed, or the waiting backend is canceled or terminated.[syncrep.c#SyncRepWaitForLSN](../../../raw/postgres-12/src/backend/replication/syncrep.c#L130-L310) [syncrep.c#SyncRepUpdateSyncStandbysDefined](../../../raw/postgres-12/src/backend/replication/syncrep.c#L1067-L1107) A logical subscription can be that synchronous standby, using its subscription name by default or a connection `application_name` override.[logical-replication.sgml#subscription-sync-standby](../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L187-L200)

5. **Replication slots, WAL retention, and row-removal horizons.** Timeout calls `proc_exit()`. The shared-memory exit chain invokes the registered `ProcKill()` callback, which releases an acquired slot and cleans up session-temporary slots; `WalSndKill()` separately marks the sender entry unused.[walsender.c#WalSndShutdown](../../../raw/postgres-12/src/backend/replication/walsender.c#L324-L339) [ipc.c#proc_exit](../../../raw/postgres-12/src/backend/storage/ipc/ipc.c#L90-L107) [ipc.c#shmem_exit](../../../raw/postgres-12/src/backend/storage/ipc/ipc.c#L193-L240) [proc.c#ProcKill-registration](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L458-L463) [proc.c#ProcKill](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L811-L851) A persistent slot is only marked inactive and preserves its required resources; a temporary slot is dropped during process cleanup.[slot.c#ReplicationSlotRelease](../../../raw/postgres-12/src/backend/replication/slot.c#L417-L473) [slot.c#ReplicationSlotCleanup](../../../raw/postgres-12/src/backend/replication/slot.c#L475-L511)

   Status replies advance a physical slot's `restart_lsn` or a logical slot's `confirmed_flush` and, when eligible, its `restart_lsn` and catalog xmin.[walsender.c#PhysicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/walsender.c#L1732-L1762) [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1859-L1868) [logical.c#LogicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/logical/logical.c#L1000-L1084) Once disconnected, no new feedback advances those positions. `ReplicationSlotsComputeRequiredLSN()` considers all allocated slots, including inactive persistent ones, and v12 has no built-in slot-WAL size cap; a disconnected persistent slot can therefore retain increasing WAL.[slot.c#ReplicationSlotsComputeRequiredLSN](../../../raw/postgres-12/src/backend/replication/slot.c#L744-L776) [high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L908-L938) Without a slot, the disconnected standby instead depends on retained `wal_keep_segments` or archive WAL and can find that required WAL has been removed before reconnect.[config.sgml#wal_keep_segments](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3643-L3675)

6. **Wakeups and protocol traffic.** When no earlier heartbeat is outstanding, a lower value schedules the requested keepalive earlier and a higher value later. This can change sender wakeup frequency and reply traffic on an otherwise idle stream.[walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111) [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454)

### What it does not do

- It does not time out ordinary SQL, idle replication-command connections, the main `BASE_BACKUP` stream, the initial logical table `COPY`, or SQL logical-decoding functions. Its checks are confined to the replication streaming loops described above.[walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1518-L1576) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2275)
- It does not control how long a receiver waits for publisher data. Physical walreceivers and logical workers use the separate `wal_receiver_timeout` path.[walreceiver.c#receiver-timeout](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L501-L549) [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1289-L1341)
- It does not enforce LSN progress, cap replication lag, or cap disk space. Any valid reply resets it, while persistent slot retention follows `restart_lsn` and confirmation progress.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1691-L1698) [slot.c#ReplicationSlotsComputeRequiredLSN](../../../raw/postgres-12/src/backend/replication/slot.c#L744-L776)
- It does not delete a persistent slot when it disconnects the sender. It does cause a session-temporary slot to be dropped as part of process cleanup.[slot.c#ReplicationSlotRelease](../../../raw/postgres-12/src/backend/replication/slot.c#L417-L473) [slot.c#ReplicationSlotCleanup](../../../raw/postgres-12/src/backend/replication/slot.c#L475-L511)
- Core timeout handling does not alter logical output-plugin filtering or message contents. The timeout surrounds the decoding callbacks at the replication transport layer.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1086-L1129) [walsender.c#XLogSendLogical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2808-L2880)
- It is not a strict wall-clock deadline while arbitrary server code is stalled. The sender checks it cooperatively and deliberately discounts server-side delay through `last_processing`.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148)

### Build, generated artifacts, and extension boundaries

The timeout implementation is core backend code. The replication makefile builds `walsender.o`, `slot.o`, and `syncrep.o` into the backend. It also builds the generated replication grammar and scanner used to recognize `START_REPLICATION`; those generated files select the physical or logical call path but do not define the timeout.[replication/Makefile#OBJS](../../../raw/postgres-12/src/backend/replication/Makefile#L11-L31) [repl_gram.y#start_replication](../../../raw/postgres-12/src/backend/replication/repl_gram.y#L295-L324)

The setting itself has no catalog row or generated header in v12: `guc.c` points through the ordinary `walsender.h` declaration to the process variable.[guc.c#includes](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L60-L76) [guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665) The adjacent monitoring surface does have generated-catalog input: `pg_proc.dat` declares `pg_stat_get_wal_senders()`, and `system_views.sql` builds `pg_stat_replication` on that function.[pg_proc.dat#pg_stat_get_wal_senders](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5130-L5138) [system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782)

Built-in logical replication crosses two shared-module boundaries. The worker dynamically loads `libpqwalreceiver`, and subscriptions use the `pgoutput` output plugin; their makefiles build both as shared libraries.[worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1620-L1629) [libpqwalreceiver/Makefile#shared-library](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/Makefile#L11-L26) [pgoutput/Makefile#shared-library](../../../raw/postgres-12/src/backend/replication/pgoutput/Makefile#L11-L21) Neither module owns the timeout. A whole-checkout symbol search found no `wal_sender_timeout` reference under `contrib/`; a third-party output plugin inherits the core transport behavior only when a client consumes it through `START_REPLICATION ... LOGICAL`.

### Tests and review reproduction

The pinned checkout has no regression, isolation, or TAP test under `src/test` that names `wal_sender_timeout`, sets it, and asserts the timeout log or disconnect. The closest protocol test checks `SHOW` of a different user-settable GUC on physical and database-connected replication sessions.[001_stream_rep.pl#replication-GUC-SHOW](../../../raw/postgres-12/src/test/recovery/t/001_stream_rep.pl#L135-L164) Logical table-sync and synchronous-replication tests exercise adjacent worker and `pg_stat_replication` paths, but not this timeout.[004_sync.pl#setup](../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L1-L18) [007_sync_rep.pl#pg_stat_replication](../../../raw/postgres-12/src/test/recovery/t/007_sync_rep.pl#L1-L45)

Client source provides direct defensive coverage: `pg_recvlogical` forces otherwise-superfluous status feedback specifically because `wal_sender_timeout` would kill the connection.[pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164)

For this review, an isolated server built from the exact pin was started under `.wiki-runtime/`. A physical `pg_receivewal` stream used a persistent slot. After the client was paused, reloading `wal_sender_timeout` from 60 seconds to 1 second changed the active sender, produced the exact timeout log, removed its `pg_stat_replication` row, and left the persistent slot present, inactive, and retaining a valid `restart_lsn`. The server was then stopped. Those observations match the live-reload, shutdown, sender-slot, and persistent-slot source paths.[walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2166-L2182) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2148) [proc.c#ProcKill](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L811-L851) [slot.c#ReplicationSlotRelease](../../../raw/postgres-12/src/backend/replication/slot.c#L417-L473)

## Context Reviewed

- PostgreSQL version: 12, exact `REL_12_2` pin `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Required navigation: `wiki/versions.md`, `wiki/index.md`, the latest 20 `wiki/log.md` entries via the project venv, and `wiki/v12/index.md`.
- Core sender path: `walsender.c`, `walsender.h`, `walsender_private.h`, `guc.c`, replication grammar/nodes, sender startup, reply parsing, timeout/keepalive scheduling, physical/logical send callbacks, shutdown callbacks, shared sender state, synchronous replication, and replication slots.
- Receiver/client path: physical `walreceiver.c`, logical `worker.c`/`launcher.c`/`tablesync.c`, `libpqwalreceiver`, `pg_receivewal`, `pg_basebackup`, and `pg_recvlogical`.
- Boundaries: base backup, logical slot creation, initial table copy, SQL logical decoding, persistent/temporary slots, synchronous commits, monitoring, generated parser/catalog inputs, shared modules, output plugins, and all `contrib/` references.
- History: same-checkout ancestry for timeout-related changes, including server-stall accounting, large logical transactions, slot-creation disablement, busy-sender keepalives, and the v12 per-connection GUC change; only current-pin behavior is stated as fact.
- Tests: exact-symbol and timeout-message searches across `src/test`, adjacent recovery/subscription tests, and the isolated exact-pin smoke test described above.

## Evidence Map

| Claim | Primary evidence |
| --- | --- |
| The GUC is user-context, millisecond-based, defaults to 60 seconds, and accepts `0` to disable. | [guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665), [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737) |
| v12 made it per-connection settable; active file-sourced senders process reloads. | [release-12.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L4519-L4534), [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2166-L2182) |
| Physical and logical `START_REPLICATION` both enter `WalSndLoop()`. | [walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L533-L755), [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1058-L1141) |
| Valid protocol messages, not LSN advancement or receiver timestamps, reset the timeout. | [walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1596-L1699), [walsender.c#ProcessStandbyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1701-L1730) |
| Normal requested keepalive is at half-time, with separate logical-wait and shutdown branches; termination is at the full deadline. | [walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111), [walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1358-L1370), [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2148) |
| Built-in subscriptions and streaming table-sync catch-up are publisher-side timeout clients. | [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1769), [tablesync.c#LogicalRepSyncTableStart](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L884-L958) |
| Physical and logical receivers send periodic and forced replies; their receive timeout is separate. | [walreceiver.c#receiver-timeout](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L501-L549), [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1289-L1341), [worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1345-L1435) |
| Timeout can remove a required synchronous sender without releasing its commit waiters. | [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1835-L1868), [syncrep.c#SyncRepWaitForLSN](../../../raw/postgres-12/src/backend/replication/syncrep.c#L130-L310) |
| Exit preserves an inactive persistent slot, drops a temporary slot, and can therefore increase WAL retained by the persistent slot. | [proc.c#ProcKill](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L811-L851), [slot.c#ReplicationSlotRelease](../../../raw/postgres-12/src/backend/replication/slot.c#L417-L473), [slot.c#ReplicationSlotCleanup](../../../raw/postgres-12/src/backend/replication/slot.c#L475-L511), [slot.c#ReplicationSlotsComputeRequiredLSN](../../../raw/postgres-12/src/backend/replication/slot.c#L744-L776) |
| No direct upstream timeout test exists at the pin; `pg_recvlogical` explicitly defends against it. | [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164), [001_stream_rep.pl#replication-GUC-SHOW](../../../raw/postgres-12/src/test/recovery/t/001_stream_rep.pl#L135-L164) |

## Open Questions

No runtime behavior remains unresolved at the pinned commit. One source comment is stale: the variable declaration calls `wal_sender_timeout` the “maximum time to send one WAL data message,” while the executable path measures time since a receiver reply and the user documentation describes inactive replication connections. This page follows the implementation and user documentation.[walsender.c:119](../../../raw/postgres-12/src/backend/replication/walsender.c#L119-L124) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148) [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737)

The remaining verification gap is upstream test coverage: the checkout has no dedicated test that drives `wal_sender_timeout` to expiration or checks its live-reload deadline change.

## Source References

- [guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665)
- [guc.c#wal_receiver_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2107-L2126)
- [guc.c#wal_retrieve_retry_interval](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2911-L2922)
- [guc.h#GucContext](../../../raw/postgres-12/src/include/utils/guc.h#L50-L77)
- [guc.h#GucSource](../../../raw/postgres-12/src/include/utils/guc.h#L79-L121)
- [guc.c#source-priority](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6916-L6941)
- [catalogs.sgml#context-user](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10601-L10610)
- [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737)
- [release-12.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L4519-L4534)
- [high-availability.sgml#primary_conninfo-options](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L708-L742)
- [high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L908-L938)
- [logical-replication.sgml#subscription](../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L163-L200)
- [protocol.sgml#primary-keepalive](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2144-L2196)
- [protocol.sgml#standby-status-update](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2200-L2285)
- [walsender.h#wal_sender_timeout](../../../raw/postgres-12/src/include/replication/walsender.h#L29-L42)
- [walsender_private.h#WalSnd](../../../raw/postgres-12/src/include/replication/walsender_private.h#L31-L83)
- [walsender_private.h#WalSndCtlData](../../../raw/postgres-12/src/include/replication/walsender_private.h#L87-L112)
- [walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L533-L755)
- [walsender.c#CreateReplicationSlot](../../../raw/postgres-12/src/backend/replication/walsender.c#L941-L957)
- [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1058-L1141)
- [walsender.c#WalSndWriteData](../../../raw/postgres-12/src/backend/replication/walsender.c#L1171-L1260)
- [walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1293-L1423)
- [walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1596-L1699)
- [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1764-L1868)
- [walsender.c#ProcessStandbyHSFeedbackMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1949-L2066)
- [walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2068-L2111)
- [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148)
- [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2275)
- [walsender.c#WalSndKill](../../../raw/postgres-12/src/backend/replication/walsender.c#L2332-L2352)
- [walsender.c#XLogSendPhysical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2538-L2806)
- [walsender.c#XLogSendLogical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2808-L2880)
- [walsender.c#pg_stat_get_wal_senders](../../../raw/postgres-12/src/backend/replication/walsender.c#L3226-L3398)
- [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454)
- [walreceiver.c#receive-loop](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L384-L451)
- [walreceiver.c#receiver-timeout](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L501-L549)
- [walreceiver.c#XLogWalRcvSendReply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L1045-L1115)
- [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1118-L1343)
- [worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1345-L1435)
- [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1769)
- [tablesync.c#LogicalRepSyncTableStart](../../../raw/postgres-12/src/backend/replication/logical/tablesync.c#L804-L958)
- [launcher.c#logicalrep_worker_onexit](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L716-L725)
- [launcher.c#ApplyLauncherMain](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L995-L1066)
- [libpqwalreceiver.c#libpqrcv_connect](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L116-L163)
- [libpqwalreceiver.c#libpqrcv_startstreaming](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L359-L449)
- [logicalfuncs.c#pg_logical_slot_get_changes_guts](../../../raw/postgres-12/src/backend/replication/logical/logicalfuncs.c#L125-L368)
- [logical.c#LogicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/logical/logical.c#L1000-L1084)
- [slot.c#ReplicationSlotRelease](../../../raw/postgres-12/src/backend/replication/slot.c#L417-L473)
- [slot.c#ReplicationSlotCleanup](../../../raw/postgres-12/src/backend/replication/slot.c#L475-L511)
- [slot.c#ReplicationSlotsComputeRequiredLSN](../../../raw/postgres-12/src/backend/replication/slot.c#L744-L776)
- [ipc.c#proc_exit](../../../raw/postgres-12/src/backend/storage/ipc/ipc.c#L90-L107)
- [proc.c#ProcKill-registration](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L458-L463)
- [proc.c#ProcKill](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L811-L851)
- [syncrep.c#SyncRepWaitForLSN](../../../raw/postgres-12/src/backend/replication/syncrep.c#L130-L310)
- [syncrep.c#SyncRepReleaseWaiters](../../../raw/postgres-12/src/backend/replication/syncrep.c#L411-L514)
- [system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782)
- [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164)
- [receivelog.c#ProcessKeepaliveMsg](../../../raw/postgres-12/src/bin/pg_basebackup/receivelog.c#L977-L1031)
- [001_stream_rep.pl#replication-GUC-SHOW](../../../raw/postgres-12/src/test/recovery/t/001_stream_rep.pl#L135-L164)
- [004_sync.pl#setup](../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L1-L18)
- [007_sync_rep.pl#pg_stat_replication](../../../raw/postgres-12/src/test/recovery/t/007_sync_rep.pl#L1-L45)

## Navigation

- [PostgreSQL 12 index](../index.md)
- [PostgreSQL 12 codebase navigation guide](../codebase-navigation-guide.md)
- [Wiki index](../../index.md)
- [Version manifest](../../versions.md)
