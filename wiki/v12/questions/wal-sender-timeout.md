---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How wal_sender_timeout Is Used and What It Impacts in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Scope, defaults, and apply scope](#scope-defaults-and-apply-scope)
  - [Where timeout tracking starts](#where-timeout-tracking-starts)
  - [What counts as a reply](#what-counts-as-a-reply)
  - [Keepalive and termination algorithm](#keepalive-and-termination-algorithm)
  - [Physical streaming path](#physical-streaming-path)
  - [Does it impact logical replication](#does-it-impact-logical-replication)
  - [Operational impacts](#operational-impacts)
  - [What it does not do](#what-it-does-not-do)
  - [Test coverage in the pinned checkout](#test-coverage-in-the-pinned-checkout)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, give a comprehensive explanation of how wal_sender_timeout is used and what it impacts. Does it impact logical replication?

Prompt note: the original request had typos and grammar issues. The user approved correcting the prompt before filing this page.

## Answer

Yes. In PostgreSQL 12, `wal_sender_timeout` is a **sender-side WAL sender timeout**. It is checked by the WAL sender process while it is streaming after `START_REPLICATION`; if the sender has not received any reply from the receiving side for the configured interval, the sender logs `terminating walsender process due to replication timeout` and shuts down that replication connection.[walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2265) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148)

It **does impact logical replication** on the publisher side. Built-in logical subscriptions start a replication connection with `START_REPLICATION SLOT ... LOGICAL`; the publisher dispatches that command to `StartLogicalReplication()`, which runs the same `WalSndLoop()` timeout machinery used by physical streaming.[worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1766) [libpqwalreceiver.c#libpqrcv_startstreaming](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L360-L448) [walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1468-L1546) [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129)

### Short answer

| Question | PostgreSQL 12 answer |
| --- | --- |
| What is it? | A GUC that makes the sending server terminate inactive replication connections after the configured interval.[config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737) |
| Default and disable value? | The built-in default is `60 * 1000` ms, documented as 60 seconds; `0` disables the timeout mechanism.[guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2664) [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3721-L3728) |
| Apply scope? | The GUC is `PGC_USERSET`, so it does not require restart; v12 documents `user` context settings as settable in config or within a session, with any user allowed to set a session-local value.[guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2664) [catalogs.sgml#context-user](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10601-L10610) |
| What does it measure? | Time since the WAL sender last processed a receiver reply during streaming; the code stores that time in `last_reply_timestamp`.[walsender.c#reply-timestamps](../../../raw/postgres-12/src/backend/replication/walsender.c#L164-L174) [walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1601-L1698) |
| What happens before it fires? | At half the timeout with no reply, the sender sends a keepalive with `requestReply = true`; at the full timeout, it shuts down the walsender connection.[walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454) [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148) |
| Does it affect logical replication? | Yes, for the publisher-side WAL sender used by logical subscriptions and logical decoding clients.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129) [worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1350-L1427) [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164) |
| Is it the subscriber-side receive timeout? | No. Logical apply workers also use `wal_receiver_timeout`, `wal_receiver_status_interval`, and `wal_retrieve_retry_interval`; those are subscriber-side settings, separate from the publisher-side `wal_sender_timeout`.[worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1122-L1341) [config.sgml#logical-subscriber-settings](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4297-L4310) |

### Scope, defaults, and apply scope

`wal_sender_timeout` lives in the replication-sending GUC group. The docs define the replication section as controlling built-in streaming replication, with servers acting as senders, receivers, or both under cascading replication; the sending-server subsection applies to servers that send replication data.[config.sgml#replication-section](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3563-L3577) [config.sgml#sending-servers](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3579-L3589)

The GUC entry is `PGC_USERSET`, has millisecond units, points at the global `wal_sender_timeout` variable, defaults to `60 * 1000`, accepts `0` through `INT_MAX`, and has no custom check, assign, or show hook.[guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2664) The walsender source initializes the same global variable to `60 * 1000` and declares it as a user-settable walsender parameter.[walsender.c#user-settable-parameters](../../../raw/postgres-12/src/backend/replication/walsender.c#L118-L124) [walsender.h#user-settable-parameters](../../../raw/postgres-12/src/include/replication/walsender.h#L29-L38)

The docs say the parameter terminates replication connections inactive longer than the configured interval, helps the sending server detect a standby crash or network outage, treats unitless values as milliseconds, defaults to 60 seconds, and uses `0` to disable the timeout mechanism.[config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737) The sample config places it under sending servers and documents `60s` with `0 disables`.[postgresql.conf.sample#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/postgresql.conf.sample#L282-L290)

Because it is `PGC_USERSET`, it is a session/transaction-scope GUC for `SET` purposes and does not require restart.[guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2664) [catalogs.sgml#context-user](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10601-L10610) In practice, a replication receiver can set a per-connection sender-side value through libpq startup options: v12's physical-standby documentation shows `primary_conninfo` passing `options='-c wal_sender_timeout=5000'`, and libpq documents the `options` connection parameter as command-line options sent to the server at connection start.[high-availability.sgml#primary_conninfo-options](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L736-L740) [libpq.sgml#libpq-connect-options](../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L1153-L1172)

For logical subscriptions, the same setting must be applied to the publisher-side replication connection if a per-subscription value is desired: `CREATE SUBSCRIPTION ... CONNECTION` stores a libpq connection string to the publisher, the apply worker passes that string into `walrcv_connect()`, and the libpq walreceiver uses `PQconnectStartParams(..., expand_dbname = true)` with `replication = database` for logical replication.[create_subscription.sgml#connection](../../../raw/postgres-12/doc/src/sgml/ref/create_subscription.sgml#L72-L80) [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1686-L1766) [libpqwalreceiver.c#libpqrcv_connect](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L121-L163)

### Where timeout tracking starts

The timeout is not a general SQL-session timeout. The replication-command dispatcher parses replication commands with the replication grammar, then routes `T_StartReplicationCmd` to `StartReplication()` for physical replication or `StartLogicalReplication()` for logical replication.[walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1468-L1546) The grammar builds `StartReplicationCmd` nodes for physical `START_REPLICATION` and logical `START_REPLICATION SLOT ... LOGICAL`, and `StartReplicationCmd` carries the replication kind, slot name, timeline, start LSN, and options.[repl_gram.y#start-replication](../../../raw/postgres-12/src/backend/replication/repl_gram.y#L295-L323) [replnodes.h#StartReplicationCmd](../../../raw/postgres-12/src/include/nodes/replnodes.h#L20-L24) [replnodes.h#StartReplicationCmd-fields](../../../raw/postgres-12/src/include/nodes/replnodes.h#L79-L87)

Timeout tracking becomes active when the sender enters `WalSndLoop()`. `WalSndLoop()` sets `last_reply_timestamp = GetCurrentTimestamp()` and clears `waiting_for_ping_response` before the streaming loop starts.[walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2166) Physical replication reaches this loop through `StartReplication()` after the sender enters copy-both mode, initializes `sentPtr`, updates `MyWalSnd->sentPtr`, initializes synchronous-replication config, and calls `WalSndLoop(XLogSendPhysical)`.[walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L647-L698) Logical replication reaches the same loop through `StartLogicalReplication()` after the sender acquires the logical slot, builds a logical decoding context with walsender callbacks, enters copy-both mode, initializes `sentPtr` from the slot's `confirmed_flush`, and calls `WalSndLoop(XLogSendLogical)`.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129)

The code explicitly disables the timeout during one non-streaming logical-slot creation subpath. While `CREATE_REPLICATION_SLOT ... LOGICAL` builds the initial snapshot, `CreateReplicationSlot()` sets `last_reply_timestamp = 0` because it is not yet accepting feedback or sending keepalives and might otherwise be killed while waiting for WAL.[walsender.c#CreateReplicationSlot-timeout-disabled](../../../raw/postgres-12/src/backend/replication/walsender.c#L947-L954) More generally, `WalSndCheckTimeOut()` returns immediately when `last_reply_timestamp <= 0`, and `WalSndKeepaliveIfNecessary()` returns when `wal_sender_timeout <= 0` or `last_reply_timestamp <= 0`.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2148) [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454)

### What counts as a reply

The protocol runs `START_REPLICATION` in copy-both mode, where both sides can send `CopyData` messages until either side sends `CopyDone`.[protocol.sgml#copy-both](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L1208-L1221) PostgreSQL's replication protocol sends WAL data and primary keepalive messages from server to client as `CopyData` payloads, and lets the receiving process send replies back to the sender at any time as `CopyData` payloads.[protocol.sgml#xlogdata](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2063-L2143) [protocol.sgml#primary-keepalive](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2143-L2190) [protocol.sgml#standby-status-update](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2200-L2287)

`ProcessRepliesIfAny()` is the timeout clock's input path. It records `last_processing = GetCurrentTimestamp()`, reads all immediately available frontend messages, handles standby replies wrapped in `CopyData`, treats `CopyDone` as a received message, and if it received at least one message sets `last_reply_timestamp = last_processing` and clears `waiting_for_ping_response`.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1601-L1698)

Inside `CopyData`, the first byte selects a standby message type. `ProcessStandbyMessage()` accepts `r` for a standby status update and `h` for hot-standby feedback; any other message type is a protocol violation that exits the process.[walsender.c#ProcessStandbyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1702-L1730) A status update carries write, flush, apply, reply-time, and reply-request fields; the sender uses it to update `MyWalSnd->write`, `flush`, `apply`, lag times, and `replyTime`, to optionally answer a receiver ping, to release synchronous-replication waiters when this is not a cascading walsender, and to advance the replication slot's confirmed position when a slot is active.[walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1764-L1868)

Hot-standby feedback also counts as a received `CopyData` reply because `ProcessRepliesIfAny()` marks the message as received before the inner message type is interpreted.[walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1651-L1698) Its handler updates `MyWalSnd->replyTime` and, for physical replication, may set xmin horizons through the walsender or physical replication slot.[walsender.c#ProcessStandbyHSFeedbackMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1949-L2066)

The key shared-memory data structure is `WalSnd`. It stores the walsender PID, state, sent LSN, receiver write/flush/apply LSNs, measured write/flush/apply lag, the latch pointer, synchronous-standby priority, and `replyTime`; `WalSndCtlData` stores the cluster-wide array of `WalSnd` entries and synchronous-replication queues.[walsender_private.h#WalSnd](../../../raw/postgres-12/src/include/replication/walsender_private.h#L22-L83) [walsender_private.h#WalSndCtlData](../../../raw/postgres-12/src/include/replication/walsender_private.h#L87-L112)

### Keepalive and termination algorithm

`WalSndComputeSleeptime()` controls how long the streaming loops sleep. Its default sleep is 10 seconds, but when `wal_sender_timeout > 0` and `last_reply_timestamp > 0`, it computes the next wakeup at the full timeout; if no ping is currently outstanding, it instead wakes at half the timeout so `WalSndKeepaliveIfNecessary()` can request a reply.[walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111)

`WalSndKeepaliveIfNecessary()` sends a primary keepalive once half of `wal_sender_timeout` has elapsed without a reply. The keepalive carries the current `sentPtr`, the server timestamp, and a byte set to `1` when it requests an immediate reply; after sending it, the sender sets `waiting_for_ping_response = true` and attempts to flush pending output.[walsender.c#WalSndKeepalive](../../../raw/postgres-12/src/backend/replication/walsender.c#L3400-L3419) [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454) The protocol defines that keepalive reply-request byte as `1` meaning the client should reply as soon as possible to avoid a timeout disconnect.[protocol.sgml#primary-keepalive](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2143-L2190)

`WalSndCheckTimeOut()` computes `timeout = last_reply_timestamp + wal_sender_timeout`. If timeouts are active and `last_processing >= timeout`, it logs a communication error and calls `WalSndShutdown()`; the comment says the sender does not send that error to the standby because timeout expiration usually means a communication problem.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148) The check uses `last_processing`, not a new timestamp taken inside the function, to avoid counting server-side stalls against the client; the code comment also notes that a long server-side stall can still force the client to reply almost immediately after the eventual keepalive.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2124)

The timeout is checked in all relevant streaming waits. The main walsender loop calls `ProcessRepliesIfAny()`, sends more data if possible, flushes output, calls `WalSndCheckTimeOut()`, sends keepalives if needed, and sleeps with `WL_SOCKET_READABLE`, `WL_TIMEOUT`, and sometimes `WL_SOCKET_WRITEABLE`.[walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2265) When a logical sender is blocked flushing output, `WalSndWriteData()` processes replies, checks timeouts, sends keepalives, computes timeout-aware sleeps, and waits for socket readability/writability.[walsender.c#WalSndWriteData](../../../raw/postgres-12/src/backend/replication/walsender.c#L1172-L1260) When a sender is waiting for more WAL, `WalSndWaitForWal()` processes replies, checks timeouts, sends keepalives, computes timeout-aware sleeps, and waits for latch, socket readability, timeout, and possibly socket writability.[walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1294-L1423)

### Physical streaming path

Physical streaming starts in `StartReplication()`. The sender rejects logical slots in the physical path, chooses the timeline and flush pointer, enters copy-both mode, sets `sentPtr` from the requested start point, updates `MyWalSnd->sentPtr`, initializes synchronous-replication config, and calls `WalSndLoop(XLogSendPhysical)`.[walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L540-L698)

`XLogSendPhysical()` determines how far WAL can safely be sent from the current timeline, records a lag-tracking sample, constructs an `XLogData` message, reads WAL into the output buffer, fills the send timestamp, sends it as `CopyData`, advances `sentPtr`, and updates `MyWalSnd->sentPtr`.[walsender.c#XLogSendPhysical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2540-L2805)

The physical walreceiver sends status replies back to satisfy the sender's timeout. It sends a reply after receiving data, sends forced replies requested by recovery, checks its own `wal_receiver_timeout`, requests a reply from the sender at half its receiver timeout, and calls `XLogWalRcvSendReply()`.[walreceiver.c#receive-loop](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L430-L559) `XLogWalRcvSendReply()` can skip non-forced status reports when `wal_receiver_status_interval <= 0`; otherwise it sends an `r` message with write, flush, apply, current timestamp, and the receiver's own reply-request byte.[walreceiver.c#XLogWalRcvSendReply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L1048-L1115) When the physical walreceiver receives a primary keepalive with `replyRequested = true`, it immediately sends a forced reply.[walreceiver.c#keepalive-reply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L828-L882)

### Does it impact logical replication

Yes. Built-in logical replication uses the same publisher-side walsender timeout after it starts streaming. The logical apply worker loads `libpqwalreceiver`, reads the subscription, connects to the publisher, creates logical streaming options with `options.logical = true`, the subscription slot name, protocol version, and publication list, then calls `walrcv_startstreaming()` and runs `LogicalRepApplyLoop()`.[worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1766)

The libpq walreceiver builds exactly the logical replication command that the publisher dispatches into `StartLogicalReplication()`: it begins with `START_REPLICATION`, adds `SLOT "..."`, appends `LOGICAL`, adds the start LSN, and adds logical protocol options including `proto_version` and `publication_names`; it expects `PGRES_COPY_BOTH` for successful streaming.[libpqwalreceiver.c#libpqrcv_startstreaming](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L360-L448) On the publisher, `StartLogicalReplication()` acquires the slot, creates a logical decoding context with walsender write/progress callbacks, enters copy-both mode, initializes `sentPtr` from the slot's `confirmed_flush`, and calls `WalSndLoop(XLogSendLogical)`.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129)

The logical apply worker sends the replies that reset the publisher's `last_reply_timestamp`. In `LogicalRepApplyLoop()`, `w` messages advance `last_received` and dispatch changes, `k` keepalive messages parse `reply_requested`, and `send_feedback(last_received, reply_requested, false)` replies immediately when the publisher requested it.[worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1122-L1225) On timeout-driven wakeups, the logical worker checks the subscriber-side `wal_receiver_timeout`, sets `requestReply` at half that timeout, and calls `send_feedback(last_received, requestReply, requestReply)`.[worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1295-L1341) The `send_feedback()` function constructs an `r` standby status update with receive/write, flush, apply, send time, and reply-request fields, then sends it with `walrcv_send()` over the publisher connection.[worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1350-L1427) [libpqwalreceiver.c#libpqrcv_send](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L783-L795)

`wal_sender_timeout` also affects standalone logical decoding clients that use the replication protocol. `pg_recvlogical` starts `START_REPLICATION SLOT "..." LOGICAL ...`, sends `r` feedback, and its feedback code explicitly forces timeout-driven feedback because otherwise `wal_sender_timeout` will kill the client connection.[pg_recvlogical.c#StreamLogicalLog](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L203-L312) [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164) `pg_recvlogical` also parses `k` keepalives and flushes/sends feedback when the sender requests a reply.[pg_recvlogical.c#keepalive-reply](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L442-L487)

Do not confuse the publisher-side `wal_sender_timeout` with subscriber-side logical worker settings. The v12 docs say the subscriber-side logical replication settings control subscriber behavior and that `wal_receiver_timeout`, `wal_receiver_status_interval`, and `wal_retrieve_retry_interval` affect logical replication workers.[config.sgml#logical-subscriber-settings](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4297-L4310) The GUC table defines `wal_receiver_status_interval` as `PGC_SIGHUP` in seconds and `wal_receiver_timeout` as `PGC_SIGHUP` in milliseconds, both under replication standby settings.[guc.c#wal_receiver_timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2107-L2126) The logical launcher retries missing apply workers no more often than `wal_retrieve_retry_interval`, which is why a sender-side timeout normally appears as a broken stream followed by a later worker reconnect attempt rather than as an automatic restart of the same background worker.[launcher.c#ApplyLauncherMain](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L1000-L1066) [launcher.c#worker-exit-cleanup](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L650-L724)

### Operational impacts

1. **Failure detection for replication connections.** A smaller value makes the sender detect a dead receiver or broken network sooner; a larger value tolerates longer network stalls. The docs describe this exact tradeoff for clusters spread across geographic locations.[config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3721-L3737)

2. **Replication stream churn.** When the timeout fires, the sender shuts down the walsender process for that connection.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2148) Walsender exit cleanup releases the active replication slot if one is held and marks the `WalSnd` entry inactive by setting its PID to `0`.[walsender.c#WalSndErrorCleanup](../../../raw/postgres-12/src/backend/replication/walsender.c#L300-L315) [walsender.c#WalSndKill](../../../raw/postgres-12/src/backend/replication/walsender.c#L2336-L2352)

3. **Monitoring.** `pg_stat_replication` is one row per WAL sender process and exposes sent, write, flush, replay LSNs, lag fields, sync state, and `reply_time`.[system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782) [monitoring.sgml#pg_stat_replication-view](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1771-L1982) Those fields are populated from `WalSnd` shared memory by `pg_stat_get_wal_senders()`, and the sender updates write/flush/apply/lag/reply-time fields when it processes standby status replies.[walsender.c#pg_stat_get_wal_senders](../../../raw/postgres-12/src/backend/replication/walsender.c#L3226-L3388) [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1835-L1854) If the timeout terminates the sender, its `WalSnd` PID is cleared and the `pg_stat_replication` row disappears because the view joins activity rows to active `pg_stat_get_wal_senders()` rows.[walsender.c#WalSndKill](../../../raw/postgres-12/src/backend/replication/walsender.c#L2336-L2352) [system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782)

4. **Synchronous replication.** A status reply can release synchronous-commit waiters because `ProcessStandbyReplyMessage()` calls `SyncRepReleaseWaiters()` for non-cascading walsenders after it updates write/flush/apply positions.[walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1835-L1858) `SyncRepReleaseWaiters()` uses the current write, flush, and apply positions from synchronous standbys to wake wait queues for `remote_write`, flush, and apply levels.[syncrep.c#SyncRepReleaseWaiters](../../../raw/postgres-12/src/backend/replication/syncrep.c#L418-L514) Logical subscriptions can be synchronous standbys in v12; the docs say a logical replication subscription can be a standby for synchronous replication and uses the subscription name as the default standby name.[logical-replication.sgml#subscription-sync-standby](../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L187-L200)

5. **Replication slot advancement and WAL retention.** For a slot-backed connection, each standby status reply with a valid flush LSN advances either the logical slot's confirmed location or the physical slot's received location.[walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1859-L1868) `LogicalConfirmReceivedLocation()` writes the logical slot's `confirmed_flush` and may advance `restart_lsn`, persist slot state, and recompute required WAL.[logical.c#LogicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/logical/logical.c#L1000-L1084) `ReplicationSlotsComputeRequiredLSN()` computes the oldest `restart_lsn` across active slots and informs xlog of the required minimum LSN.[slot.c#ReplicationSlotsComputeRequiredLSN](../../../raw/postgres-12/src/backend/replication/slot.c#L745-L775) The docs state that replication slots prevent the master from removing WAL segments until all standbys have received them, and in v12 there is no built-in bound on the disk space requirement for replication slots.[high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930) Therefore a timeout does not itself delete a persistent slot; it stops feedback until the receiver reconnects, so a slot-backed lagging receiver can cause WAL retention while disconnected.[walsender.c#WalSndErrorCleanup](../../../raw/postgres-12/src/backend/replication/walsender.c#L300-L315) [high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930)

6. **Timeout-related wakeups and network traffic.** A lower timeout makes the sender wake and request replies earlier because the keepalive threshold is half the configured timeout; a higher timeout delays those keepalives and the final disconnect.[walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111) [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454)

### What it does not do

`wal_sender_timeout` does **not** timeout ordinary SQL queries. The timeout functions are in `walsender.c`, are reached by `WalSndLoop()`, and the replication dispatcher enters that loop only for `START_REPLICATION` physical or logical streaming.[walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1468-L1546) [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2265)

It does **not** control how long the receiver waits for data from the sender. Physical walreceiver and logical apply workers use `wal_receiver_timeout` for that direction, and the docs explicitly list `wal_receiver_timeout`, `wal_receiver_status_interval`, and `wal_retrieve_retry_interval` as affecting logical replication workers.[walreceiver.c#receive-loop](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L501-L548) [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1295-L1341) [config.sgml#logical-subscriber-settings](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4297-L4310)

It does **not** by itself bound replication lag or slot disk usage. It only disconnects a replication connection that stops replying; slot retention is governed by slot `restart_lsn`/confirmation progress and by the fact that v12 replication slots retain needed WAL without a built-in size cap.[walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148) [logical.c#LogicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/logical/logical.c#L1000-L1084) [high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930)

It does **not** change logical decoding output-plugin behavior. The publisher's logical path creates a decoding context with walsender callbacks and then streams through `WalSndLoop(XLogSendLogical)`; timeout handling is outside the plugin's message content and applies to the replication protocol connection around it.[walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1093-L1129) [walsender.c#XLogSendLogical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2809-L2880) The built-in subscription slot creation path uses the `pgoutput` output plugin, while standalone logical clients such as `pg_recvlogical` can use logical replication protocol streaming and still must send feedback to avoid `wal_sender_timeout`.[libpqwalreceiver.c#libpqrcv_create_slot](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L804-L856) [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164)

It is not active during the initial logical slot snapshot build inside `CREATE_REPLICATION_SLOT ... LOGICAL`, because that code sets `last_reply_timestamp = 0` before `DecodingContextFindStartpoint()`.[walsender.c#CreateReplicationSlot-timeout-disabled](../../../raw/postgres-12/src/backend/replication/walsender.c#L947-L954)

### Test coverage in the pinned checkout

I did not find a dedicated `wal_sender_timeout` regression, TAP, or isolation test under `src/test` in the pinned PostgreSQL 12 checkout during this review. The source-visible coverage closest to this behavior is client-side: `pg_recvlogical` has an explicit comment and code path forcing feedback when needed because otherwise `wal_sender_timeout` will kill the logical decoding client connection.[pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164) Related subscriber-side tests set other retry/receiver knobs, such as `wal_retrieve_retry_interval`, but that is not direct `wal_sender_timeout` coverage.[004_sync.pl#wal_retrieve_retry_interval](../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L15-L18)

## Context Reviewed

- PostgreSQL version: 12 (`REL_12_STABLE`) pinned at `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Required wiki navigation read before drafting: `wiki/versions.md`, `wiki/index.md`, recent `wiki/log.md` entries via `scripts/recent_log --limit 20`, and `wiki/v12/index.md`.
- Primary source files: `src/backend/replication/walsender.c`, `src/include/replication/walsender.h`, `src/include/replication/walsender_private.h`, `src/backend/utils/misc/guc.c`, `src/backend/replication/walreceiver.c`, `src/backend/replication/logical/worker.c`, `src/backend/replication/logical/launcher.c`, `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c`, `src/backend/replication/logical/logical.c`, `src/backend/replication/slot.c`, `src/backend/replication/syncrep.c`, `src/backend/replication/repl_gram.y`, and `src/include/nodes/replnodes.h`.
- Supporting docs and client code: `doc/src/sgml/config.sgml`, `doc/src/sgml/high-availability.sgml`, `doc/src/sgml/logical-replication.sgml`, `doc/src/sgml/protocol.sgml`, `doc/src/sgml/monitoring.sgml`, `doc/src/sgml/libpq.sgml`, `doc/src/sgml/ref/create_subscription.sgml`, `src/backend/catalog/system_views.sql`, and `src/bin/pg_basebackup/pg_recvlogical.c`.
- Test-surface search: `src/test` for `wal_sender_timeout`, and related logical subscription tests for receiver/retry settings.

## Evidence Map

| Claim | Source evidence |
| --- | --- |
| `wal_sender_timeout` is a user-context millisecond GUC with 60s default and `0` disable behavior. | [guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2664), [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737), [catalogs.sgml#context-user](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10601-L10610) |
| Timeout tracking starts when `WalSndLoop()` begins streaming and records `last_reply_timestamp`. | [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2166) |
| Physical and logical `START_REPLICATION` both enter `WalSndLoop()`. | [walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L647-L698), [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129) |
| Replies reset the timeout clock; `r` status updates also update LSN/lag/sync/slot state. | [walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1601-L1698), [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1764-L1868) |
| The sender requests a reply at half timeout and terminates at full timeout. | [walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111), [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454), [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148) |
| Built-in logical subscriptions are affected because the apply worker starts `START_REPLICATION SLOT ... LOGICAL` and the publisher uses `StartLogicalReplication()`. | [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1766), [libpqwalreceiver.c#libpqrcv_startstreaming](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L360-L448), [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129) |
| Logical workers send `r` feedback and respond to publisher keepalive requests. | [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1122-L1341), [worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1350-L1427) |
| Subscriber-side receive timeouts are separate settings that affect logical workers. | [guc.c#wal_receiver_timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2107-L2126), [config.sgml#logical-subscriber-settings](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4297-L4310) |
| Timeout can affect monitoring, synchronous replication, and slot WAL retention indirectly through connection termination and missing feedback. | [system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782), [syncrep.c#SyncRepReleaseWaiters](../../../raw/postgres-12/src/backend/replication/syncrep.c#L418-L514), [logical.c#LogicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/logical/logical.c#L1000-L1084), [high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930) |
| There is no dedicated `src/test` coverage found for this exact GUC in the pinned checkout; client code in `pg_recvlogical` directly guards against being killed by it. | [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164), [004_sync.pl#wal_retrieve_retry_interval](../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L15-L18) |

## Open Questions

None for the source behavior described above at the pinned PostgreSQL 12 commit. The only gap is test coverage: this review found no dedicated server-side test that sets `wal_sender_timeout` and asserts the timeout path.

## Source References

- [guc.c#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2664)
- [guc.c#wal_receiver_timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2107-L2126)
- [catalogs.sgml#context-user](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10601-L10610)
- [config.sgml#replication-section](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3563-L3577)
- [config.sgml#sending-servers](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3579-L3589)
- [config.sgml#wal_sender_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3714-L3737)
- [config.sgml#logical-subscriber-settings](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4297-L4310)
- [postgresql.conf.sample#wal_sender_timeout](../../../raw/postgres-12/src/backend/utils/misc/postgresql.conf.sample#L282-L290)
- [walsender.h#user-settable-parameters](../../../raw/postgres-12/src/include/replication/walsender.h#L29-L38)
- [walsender_private.h#WalSnd](../../../raw/postgres-12/src/include/replication/walsender_private.h#L22-L83)
- [walsender_private.h#WalSndCtlData](../../../raw/postgres-12/src/include/replication/walsender_private.h#L87-L112)
- [walsender.c#user-settable-parameters](../../../raw/postgres-12/src/backend/replication/walsender.c#L118-L124)
- [walsender.c#reply-timestamps](../../../raw/postgres-12/src/backend/replication/walsender.c#L164-L174)
- [walsender.c#WalSndErrorCleanup](../../../raw/postgres-12/src/backend/replication/walsender.c#L300-L315)
- [walsender.c#StartReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L540-L698)
- [walsender.c#CreateReplicationSlot-timeout-disabled](../../../raw/postgres-12/src/backend/replication/walsender.c#L947-L954)
- [walsender.c#StartLogicalReplication](../../../raw/postgres-12/src/backend/replication/walsender.c#L1063-L1129)
- [walsender.c#WalSndWriteData](../../../raw/postgres-12/src/backend/replication/walsender.c#L1172-L1260)
- [walsender.c#WalSndWaitForWal](../../../raw/postgres-12/src/backend/replication/walsender.c#L1294-L1423)
- [walsender.c#exec_replication_command](../../../raw/postgres-12/src/backend/replication/walsender.c#L1468-L1546)
- [walsender.c#ProcessRepliesIfAny](../../../raw/postgres-12/src/backend/replication/walsender.c#L1601-L1698)
- [walsender.c#ProcessStandbyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1702-L1730)
- [walsender.c#ProcessStandbyReplyMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1764-L1868)
- [walsender.c#ProcessStandbyHSFeedbackMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1949-L2066)
- [walsender.c#WalSndComputeSleeptime](../../../raw/postgres-12/src/backend/replication/walsender.c#L2076-L2111)
- [walsender.c#WalSndCheckTimeOut](../../../raw/postgres-12/src/backend/replication/walsender.c#L2113-L2148)
- [walsender.c#WalSndLoop](../../../raw/postgres-12/src/backend/replication/walsender.c#L2151-L2265)
- [walsender.c#WalSndKill](../../../raw/postgres-12/src/backend/replication/walsender.c#L2336-L2352)
- [walsender.c#XLogSendPhysical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2540-L2805)
- [walsender.c#XLogSendLogical](../../../raw/postgres-12/src/backend/replication/walsender.c#L2809-L2880)
- [walsender.c#pg_stat_get_wal_senders](../../../raw/postgres-12/src/backend/replication/walsender.c#L3226-L3388)
- [walsender.c#WalSndKeepalive](../../../raw/postgres-12/src/backend/replication/walsender.c#L3400-L3419)
- [walsender.c#WalSndKeepaliveIfNecessary](../../../raw/postgres-12/src/backend/replication/walsender.c#L3421-L3454)
- [walreceiver.c#receive-loop](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L430-L559)
- [walreceiver.c#keepalive-reply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L828-L882)
- [walreceiver.c#XLogWalRcvSendReply](../../../raw/postgres-12/src/backend/replication/walreceiver.c#L1048-L1115)
- [worker.c#LogicalRepApplyLoop](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1122-L1341)
- [worker.c#send_feedback](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1350-L1427)
- [worker.c#ApplyWorkerMain](../../../raw/postgres-12/src/backend/replication/logical/worker.c#L1596-L1766)
- [launcher.c#logicalrep_worker_launch](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L399-L456)
- [launcher.c#worker-exit-cleanup](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L650-L724)
- [launcher.c#ApplyLauncherMain](../../../raw/postgres-12/src/backend/replication/logical/launcher.c#L1000-L1066)
- [libpqwalreceiver.c#libpqrcv_connect](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L121-L163)
- [libpqwalreceiver.c#libpqrcv_startstreaming](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L360-L448)
- [libpqwalreceiver.c#libpqrcv_send](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L783-L795)
- [libpqwalreceiver.c#libpqrcv_create_slot](../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L804-L856)
- [logical.c#LogicalConfirmReceivedLocation](../../../raw/postgres-12/src/backend/replication/logical/logical.c#L1000-L1084)
- [slot.c#ReplicationSlotsComputeRequiredLSN](../../../raw/postgres-12/src/backend/replication/slot.c#L745-L775)
- [syncrep.c#SyncRepReleaseWaiters](../../../raw/postgres-12/src/backend/replication/syncrep.c#L418-L514)
- [repl_gram.y#start-replication](../../../raw/postgres-12/src/backend/replication/repl_gram.y#L295-L323)
- [replnodes.h#StartReplicationCmd](../../../raw/postgres-12/src/include/nodes/replnodes.h#L20-L24)
- [replnodes.h#StartReplicationCmd-fields](../../../raw/postgres-12/src/include/nodes/replnodes.h#L79-L87)
- [system_views.sql#pg_stat_replication](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L782)
- [high-availability.sgml#primary_conninfo-options](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L736-L740)
- [high-availability.sgml#replication-slots](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930)
- [logical-replication.sgml#subscription](../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L163-L200)
- [logical-replication.sgml#subscription-sync-standby](../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L187-L200)
- [protocol.sgml#copy-both](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L1208-L1221)
- [protocol.sgml#xlogdata](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2063-L2143)
- [protocol.sgml#primary-keepalive](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2143-L2190)
- [protocol.sgml#standby-status-update](../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2200-L2287)
- [monitoring.sgml#pg_stat_replication-view](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1771-L1982)
- [libpq.sgml#libpq-connect-options](../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L1153-L1172)
- [create_subscription.sgml#connection](../../../raw/postgres-12/doc/src/sgml/ref/create_subscription.sgml#L72-L80)
- [pg_recvlogical.c#sendFeedback](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L112-L164)
- [pg_recvlogical.c#StreamLogicalLog](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L203-L312)
- [pg_recvlogical.c#keepalive-reply](../../../raw/postgres-12/src/bin/pg_basebackup/pg_recvlogical.c#L442-L487)
- [004_sync.pl#wal_retrieve_retry_interval](../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L15-L18)

## Navigation

- [PostgreSQL 12 index](../index.md)
- [PostgreSQL 12 codebase navigation guide](../codebase-navigation-guide.md)
- [Wiki index](../../index.md)
- [Version manifest](../../versions.md)
