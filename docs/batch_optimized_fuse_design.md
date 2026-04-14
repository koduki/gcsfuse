# GCS Batch-Optimized FUSE (Draft Design)

> Status: Draft proposal (not implemented in current Cloud Storage FUSE releases).

## 1. Overview

This document proposes a **batch-optimized FUSE mode for Google Cloud Storage (GCS)**
for AI/ML and ETL workloads that prioritize throughput and completion semantics over
low-latency, full POSIX behavior.

The core concept is:

```text
File API -> Cache / Buffer -> GCS Objects
```

## 2. Target Scope

### In scope

- AI/ML training pipelines (checkpoints, datasets)
- ETL and data pipelines
- Distributed batch processing

### Out of scope

- Real-time processing
- Low-latency interactive workloads
- Strict POSIX conformance workloads
- Concurrent mutation of the same file from multiple nodes

## 3. Core Value Propositions

### 3.1 High-throughput single-file I/O

- Prioritize maximizing throughput for large files.
- Lean on parallel I/O and buffering.

### 3.2 Atomic publish

- Data remains invisible while in-progress.
- Final artifact becomes visible atomically at publish time.

### 3.3 Explicit completion semantics

- Completion is represented by publish state.
- Avoid separate marker files (for example `_SUCCESS`) for job chaining.

## 4. Data Model

### 4.1 Lifecycle state machine

```text
STAGING -> FLUSHED -> COMPOSED -> PUBLISHED
```

| State     | Meaning |
|-----------|---------|
| STAGING   | Buffering locally and writing part objects |
| FLUSHED   | All bytes uploaded to GCS part objects |
| COMPOSED  | Part objects composed into a single object |
| PUBLISHED | Object made externally visible |

### 4.2 Namespace layout

```text
/bucket/
  staging/
    job123/
      parts/
      file.bin
  ready/
    job123/
      file.bin
```

## 5. Write Path

1. `open`
2. `append` to local memory/disk buffer
3. `chunk` partitioning
4. parallel upload of part objects
5. `flush`
6. `compose`
7. `publish`

### 5.1 Append

- Buffer to RAM and/or local disk.

### 5.2 Part upload

- Upload chunk-aligned part objects.
- Support parallel workers.
- Use resumable behavior when possible.

### 5.3 Compose

- Build one finalized object from multiple parts.

### 5.4 Publish

- Atomically switch visibility from staging to ready namespace (for example,
  with rename semantics on hierarchical namespace-enabled buckets).

## 6. Read Path

### Characteristics

- Sequential read optimization
- Parallel range read
- Prefetch
- Local cache utilization

### Behavior

- Reads return quickly from cache/buffer when possible.
- Background asynchronous reads prefetch upcoming ranges.

## 7. Cache and Buffering

### Read cache

- Local SSD / RAM backed file cache

### Write buffering

- Write buffer and chunking
- Spill-to-disk when memory pressure is high

## 8. Consistency Model

### Principles

- In-progress data is not externally visible.
- Only published data is externally visible.

### Guarantees

- Post-publish readers see complete objects only.
- Partial/unpublished artifacts are hidden from consumers of the ready namespace.

## 9. API / Operation Model

### Internal operations

- `flush`
- `compose`
- `publish`

### External interface options

- Automatic flush/commit on close
- Explicit CLI commit/publish
- Directory-scoped publish for batch handoff

## 10. Performance Goals

### Optimized for

- Throughput of large single-file transfers

### Not optimized for

- Small I/O latency
- Heavy `fsync`-driven workflows

## 11. Failure Tolerance

### Mechanisms

- Retry logic
- Resumable upload support
- Part-level recovery

### Incomplete work handling

- Unpublished work remains in staging namespace
- Safe to retry from staging state

## 12. Constraints

- No concurrent mutation of one file
- No strict POSIX guarantee
- Not intended for real-time serving workloads

## 13. Expected Benefits

- Faster batch I/O pipelines
- Less data copy overhead
- Safer producer/consumer job handoff
- Better utilization of GCS object semantics for batch workflows

## 14. Implementation Notes for gcsfuse

This proposal can be implemented incrementally in gcsfuse as an optional mode:

1. **Config surface**:
   - add an opt-in `batch_mode` feature gate;
   - add tunables for chunk size, parallel upload workers, and publish mode.
2. **Write pipeline enhancements**:
   - stage chunk metadata;
   - upload chunks concurrently;
   - compose finalized object;
   - publish by atomic namespace transition.
3. **Visibility contract**:
   - mount path should expose only published namespace for consumers by default.
4. **Crash recovery**:
   - replay manifest state from staging and resume unfinished upload/compose.

Because this is a draft design, behavior of current released builds should remain
unchanged unless the new mode is explicitly enabled.
