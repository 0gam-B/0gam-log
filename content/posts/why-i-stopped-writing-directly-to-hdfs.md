+++
date = '2026-04-29T08:58:49+09:00'
draft = false
title = 'Why I stopped writing directly to HDFS: rebuilding our Kafka ingestion as a Connect sink'
tags = ['hdfs', 'kafka', 'kafka-connect', 'data-engineering', 'scala']
categories = ['data-engineering']
summary = 'How a legacy Kafka consumer doing direct HDFS appends caused years of data loss, rebalance storms, and 10-hour reloads — and what replaced it.'
+++

HDFS was designed around a "write once, read many" model.

Append exists. It works. It's been stable for years. But it has never been what HDFS is *for* — and using it the way I did, at the rate I did, was always going to break something eventually.

This is the story of what broke, and what I replaced it with.

## The legacy: a Kafka consumer that wrote straight to HDFS

The original ingestion was a hand-rolled Kafka consumer. For every record it received, it would:

1. Open an HDFS file (acquire a lease)
2. Append the line
3. Wait for the replication ACK chain across three datanodes (the default replication factor)
4. Close the file
5. Commit the offset
6. Repeat

Per record. Across hundreds of partitions. All day, every day.

It worked, in the sense that data ended up in HDFS most of the time. But "most of the time" is the part that gets you.

## What "most of the time" actually looked like

A few patterns kept showing up:

**Cluster slowdowns froze the entire pipeline.** Any HDFS-side delay — a slow datanode, a heavy Spark job hitting the cluster, a network blip — turned into consumer timeouts. The consumer couldn't ACK fast enough, Kafka decided it was dead, and rebalancing kicked in. Then the new owner of the partition tried to do the same thing, hit the same wall, and the cycle repeated.

**HDFS outages caused data loss.** Not "potential" loss — actual. When the cluster went down hard, in-flight appends were lost, offsets had already moved, and there was no clean way to recover. Almost every time the cluster blinked, data consumers came back asking for reloads.

**Reloads were brutal.** The largest pipeline normally finished its daily delivery between 11 AM and noon. After an incident, the same data wouldn't be ready until 8 PM — a 10-hour reload window, every time. People planned their day around it.

**Silent kick-outs.** Sometimes a consumer job would just... stop. No crash, no alert, no exception worth noticing. Just no progress on offsets. I'd find out from a user asking why their data hadn't arrived. I eventually built internal monitoring to catch this — but the underlying issue was the architecture, not the visibility.

The common thread: **a slow or unhealthy storage layer dragged the entire ingestion layer down with it.** They were one system, pretending to be two.

## Why direct HDFS append was the root cause

It's tempting to blame the cluster, or Kafka, or the network. The honest answer is that I was using HDFS in a way it was never built for.

HDFS append is real, but it's expensive. Every append involves:

- A **lease** on the file (only one writer at a time, with timeout-based reclamation)
- A **replication pipeline** to the configured replicas, with ACKs at each hop
- A **lease and block recovery** dance if the writer or any datanode dies mid-write

This is fine for occasional, large writes. It breaks down when you do it per Kafka record, across hundreds of partitions, all day long. You're spending the cluster's time and attention on tracking writes that average a few KB.

None of this is a bug. HDFS append has been stable for years, and the configuration that once gated it is no longer needed. The API works exactly as advertised.

The mismatch wasn't with the API — it was with how I was using it. HDFS is built for **write-once, read-many** workloads — large files that get written once and scanned by analytics jobs many times. I was using it as a real-time, append-heavy log store. Every record I wrote was going against what the system was built to do.

**Lesson, in retrospect:** "the API exists" and "the system is designed for this" are different statements. When your access pattern fights the storage engine's design, you'll lose — slowly at first, then all at once.

## The redesign: stage locally, upload asynchronously

The fix is conceptually simple — separate the ingestion layer from the storage layer.

```
Before:
  [Kafka] ──► [Consumer] ──► [HDFS]
                              (direct append, blocking)

After:
  [Kafka] ──► [SinkTask] ──► [in-mem buffer per topic-partition]
                                       │
                                       ▼
                             [Disruptor ring buffer]
                                       │
                                       ▼
                                [local stage file]
                                       │
                              (async, threshold-based)
                                       ▼
                                     [HDFS]
```

The new pipeline is a Kafka **Connect** sink (Scala), not a hand-rolled consumer. Records pass through three buffers before they ever hit HDFS:

1. **In-memory buffer**, per topic-partition, filled by `put()`
2. **LMAX Disruptor ring buffer**, which separates `put()` from disk I/O
3. **Local stage file** on the task's local disk, where records actually land

Only then, on a schedule, completed stage files get uploaded to HDFS as whole files. HDFS sees one large `create + write + close` instead of millions of `append` calls.

## Why Disruptor and not a `BlockingQueue`

The ring buffer choice mattered. I wanted three things:

- **Lock-free** behavior on the multi-threaded `put()` path — no contention with a global lock
- A clean handoff: `put()` adds to the buffer and returns; a background thread drains and writes
- **Backpressure** when the disk consumer falls behind, without dropping records

LMAX Disruptor gives all three. The ring buffer is fixed-size, so when the consumer can't keep up, producers naturally slow down at `next()` instead of either blocking on a lock or eating memory until OOM. Pair it with a blocking wait strategy on the consumer side and you get clean backpressure with low CPU when idle. It's the kind of library where the second time you use it, you understand why financial systems standardized on it.

## Two thresholds: switching vs. uploading

The trickiest part of the design was deciding *when* to roll a stage file and *when* to ship it. Mixing those two decisions is a common mistake. I separated them.

**Switching** rolls the active stage file:

- When a file reaches **250 MB**, or
- When the configured switching interval elapses

The 250 MB cap is on purpose. Our HDFS cluster uses a 256 MB block size, and I wanted a single uploaded file to fit cleanly into one block — no spill, no fragmentation. After switching, the in-memory queue is redirected to a new file (or to an existing under-cap file in the same partition, if one is around).

**Uploading** ships a closed (or eligible) stage file to HDFS:

- When the file goes over **250 MB** (always uploaded immediately), or
- When its configured upload interval elapses (1 / 5 / 10 / 30 minutes, per pipeline)

Two knobs let each pipeline tune its own latency-vs-efficiency tradeoff. A high-volume pipeline can keep its upload interval long because the size threshold will trigger first anyway — files fill up fast, and 250 MB chunks land in HDFS regularly. A low-volume pipeline that needs near-real-time visibility can pick a 1-minute interval and accept smaller files.

The result: **size is the main trigger for big pipelines, time is the main trigger for small ones**, and both pipelines use the same code.

## The small-file problem, and what I did about it

Anyone who's run a Hadoop cluster knows the punchline already: frequent uploads create small files, small files create NameNode pressure, and NameNode pressure ruins your day.

The redesign would have just traded one problem for another if I hadn't dealt with this. So I built a separate **merge-and-compress scheduler** (Quartz-based) that runs alongside the sink:

- **Merge** targets files smaller than 200 MB, in recently-uploaded directories (uploaded in the last hour, scoped to the last 30 days). It rewrites them into 250 MB chunks. Same block-alignment logic as the sink itself.
- **Compress** runs on data older than 30 days, going back five years. Output formats are `parquet.snappy` for columnar workloads and `tsv.gzip` for legacy text consumers.

The scheduler is a separate concern from the sink — the sink doesn't know or care that merging exists. That separation matters: when the cluster is under load, I can pause compaction without touching ingestion.

The merge and compress thresholds aren't fixed across datasets, though. Some teams asked for longer grace periods — *"don't touch the last 90 days,"* or *"leave the last 5 days alone."* I honor those, and not because of storage cost or CPU cost. It's because **merging and compressing rewrite files**. Old file goes away, new file takes its place. If a user's query happens to list the directory and then read the files a moment later, a file that was there during `LIST` can be gone by the time `READ` runs. The query fails with a "file not found," even though the data is right there under a new name.

So the rule isn't "compress everything older than 30 days." It's "compress everything older than 30 days, *unless* this dataset's users have asked for a longer quiet window." Active reading periods stay untouched. Compaction only runs over data that no one is actively scanning.

## What the new design gave us

Concretely, since the rewrite:

- **HDFS slowdowns no longer affect ingestion.** Records keep flowing into stage files. If HDFS is unavailable, uploads queue up locally and resume when the cluster recovers. Kafka Connect calls `preCommit()` regularly before offset commits, and I use that hook to retry uploads of any remaining staged files.
- **No more rebalance storms.** Because tasks aren't blocked on HDFS, they don't time out, they don't get kicked, and partitions don't shuffle.
- **Zero data loss or duplication, for four years.** This is the number that surprised me the most. The old system had reload requests on most cluster incidents; the new one has had none.
- **Cluster maintenance is invisible to users.** HDFS upgrades, NameNode failovers, datanode rolls — ingestion just keeps writing to local stage. Whatever uploads have to wait, wait.

The architecture isn't new. "Stage and ship asynchronously" is the same pattern Flume, Logstash, and dozens of internal tools have used for years. I just hadn't applied it to this pipeline.

## Lessons learned

1. **Read the design, not just the API.** HDFS append works. It's also the wrong way to use HDFS. A working API is not the same as the right tool — and "write once, read many" is a design statement, not a slogan.
2. **Coupling shows up at runtime, not just in code.** My consumer and HDFS were technically separate processes, but whenever HDFS slowed down, the consumer slowed down with it. One slow link froze everything downstream.
3. **Local disk is your friend.** Treating the task's local disk as a buffer — not as scratch space, but as a deliberate decoupling layer — fixes a class of problems that no amount of consumer tuning will.
4. **Separate "when to roll" from "when to ship."** Mixing these into one knob makes either small files or high latency unavoidable.
5. **Solve the problems your fix creates.** Async upload causes small files. Plan for the merger before you ship the sink, not after.
6. **Monitoring covers the symptom; architecture removes it.** Internal alerts were useful, but they existed because the underlying system needed watching. The redesign made most of what they watched stop happening.
7. **Rewriting files is never atomic from the reader's view.** Merge and compress jobs replace files, and a query that listed the directory a moment ago will fail when its `READ` finds a different file there. The fix isn't smarter retries — it's leaving recent, frequently-read data alone until the read traffic moves on.

---

If you're running a hand-rolled Kafka-to-HDFS path today, the question worth asking is not "is it working?" — it probably is, most of the time. The question is what happens during the next storage incident, and how long the reload window is afterward.

Mine used to be 10 hours. It is, now, however long the upload queue takes to drain.