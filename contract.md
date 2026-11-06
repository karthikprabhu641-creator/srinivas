# Shared Implementation Contract

**Project:** Single-process embedded key-value database  
**Language standard:** C++17  
**Purpose:** Give all four developers and their IDE agents one consistent implementation target.

> **Status: PROPOSED — TEAM APPROVAL REQUIRED.** This document is a starting contract, not proof that the design is already implemented or tested. The team must explicitly agree on the decisions marked **TEAM DECISION** before modules are developed against them.

---

## 1. Non-negotiable invariants

Every implementation and pull request must preserve these invariants:

1. **Acknowledged-write durability:** Once a mutating operation reports success (or emits its `ACK`), it must survive a process crash, subject to the event's stated storage assumptions.
2. **Atomic batch:** After recovery, a batch is entirely visible or entirely absent. No partial batch may be applied.
3. **No torn values:** Recovery must never expose a partially written key or value.
4. **Safe recovery:** Recovery must be deterministic and safe to repeat. A crash during recovery must not make a previously valid database unrecoverable.
5. **Single-process scope:** The database supports concurrent threads in one process. Multi-process writers are out of scope unless the team explicitly adds and tests process locking.
6. **Compaction equivalence:** Compaction must preserve the exact logical key-value state.
7. **Honest errors:** If durability cannot be established (including disk-full or I/O errors), the operation must not report success.

These invariants take priority over throughput.

## 2. Scope and required operations

The database stores byte strings as keys and values. The CLI may accept text arguments, but the storage layer should not rely on null-terminated strings.

Required logical operations:

- `open(path, max_bytes?)`
- `put(key, value)`
- `get(key)`
- `delete(key)`
- `batch(operations)`
- `close()`

Required supporting operations:

- `compact()`
- `dump()`
- `check()`

The implementation must target at least 1,000,000 keys and batches of up to 10,000 operations. Performance and recovery targets must be measured on the final test environment; do not claim they are met without recorded tests.

## 3. Proposed module ownership

| Owner | Module | Responsibility |
|---|---|---|
| Member 1 | Storage engine | File abstraction, record encoding/decoding, in-memory index, PUT/GET/DELETE integration |
| Member 2 | Durability and recovery | Commit protocol, durable append, log scan, batch atomicity, recovery tests |
| Member 3 | Concurrency and compaction | Locking, thread safety, compaction, maximum file size and disk-full behavior |
| Member 4 | CLI and torture tool | Command parsing, judge-facing CLI, independent ACK oracle, crash/restart testing |
| Team lead | Integration | Resolve interfaces, review PRs, run integrated tests, maintain architecture and final demo |

Ownership does not mean isolated design. Storage and recovery must agree on the exact record format and commit protocol. Concurrency and compaction must use the same locking and durability rules as the storage engine.

## 4. Common API proposal

Use one shared public header, for example `include/kvdb/database.hpp`. The signatures below are illustrative; agree on the final signatures before implementation.

```cpp
enum class Status {
    Ok,
    NotFound,
    InvalidArgument,
    AlreadyOpen,
    NotOpen,
    Corruption,
    NoSpace,
    IoError,
    InternalError
};

struct Operation {
    enum class Type { Put, Delete };
    Type type;
    std::string key;
    std::string value; // ignored for Delete
};

struct Result {
    Status status;
    std::string value; // populated for successful Get
};

class Database {
public:
    Status open(const std::string& path,
                std::uint64_t max_bytes = 0);

    Status put(const std::string& key,
               const std::string& value);

    Result get(const std::string& key);

    Status remove(const std::string& key);

    Status batch(const std::vector<Operation>& operations);

    Status compact();

    Status close();
};
```

**TEAM DECISION:** Finalize exact types, error reporting, maximum key/value sizes, and whether `get()` returns a copied value or another safe representation. Do not return references/pointers whose lifetime is invalidated by concurrent writes.

### API semantics

- Empty key and empty value behavior must be explicitly decided and tested.
- `get()` returns `NotFound` for an absent key; an empty stored value is not the same as a missing key.
- `delete()` of a missing key should have one documented behavior (recommended: idempotent success).
- Invalid arguments must fail without modifying logical state.
- A successful mutating call means the operation is durable according to Section 7.
- `close()` must prevent new operations, wait for or reject in-flight operations according to one documented policy, and release resources.

## 5. Proposed on-disk format

Use one append-only binary log file. Do not write native C++ structs directly to disk: compiler padding, alignment, and endianness can vary. Encode and decode each field explicitly.

### 5.1 File header

Proposed fixed header:

| Field | Size | Purpose |
|---|---:|---|
| Magic | 8 bytes | Identifies this database format |
| Format version | 4 bytes | Allows future format changes |
| Header length | 4 bytes | Validates header size |
| Reserved | 16 bytes | Must be zero in version 1 |

All integer fields use a single agreed byte order (recommended: little-endian). Header creation and validation must be tested.

### 5.2 Record frame

Proposed frame layout:

| Field | Size | Purpose |
|---|---:|---|
| Frame magic | 4 bytes | Detects frame boundaries |
| Format version | 2 bytes | Record-format version |
| Record type | 1 byte | `BEGIN`, `PUT`, `DELETE`, `COMMIT` |
| Flags | 1 byte | Reserved; zero in version 1 |
| Transaction ID | 8 bytes | Groups records into one transaction |
| Key length | 4 bytes | Byte length of key |
| Value length | 8 bytes | Byte length of value |
| Payload | variable | Key bytes followed by value bytes |
| Checksum | 4 bytes | Detects torn/corrupt frames |

The checksum algorithm and exact coverage (recommended: all frame header fields plus payload, excluding the checksum field) must be chosen once and used identically by writer, recovery, checker, and torture tests.

Record rules:

- `BEGIN`: no key/value payload.
- `PUT`: key and value payload.
- `DELETE`: key payload; value length is zero.
- `COMMIT`: no key/value payload.
- All lengths are validated against configured limits before allocation.
- Unknown versions/types, impossible lengths, and checksum failures must not be interpreted as valid data.
- Never trust a length read from disk before checking it against file size and configured limits.

**TEAM DECISION:** Confirm checksum algorithm, maximum frame size, integer byte order, and whether a file header checksum is also required.

## 6. Transaction model and atomic batches

Treat each mutating operation as a transaction. A single PUT or DELETE is a one-operation transaction. A batch is one transaction containing all its operations.

Proposed logical sequence:

```text
BEGIN(txid)
  PUT/DELETE(txid, ...)
  PUT/DELETE(txid, ...)
  ...
COMMIT(txid)
```

A valid transaction is applied to the in-memory index only after recovery has established that it contains:

1. A valid `BEGIN` for the transaction ID.
2. The expected valid operation frames.
3. One valid `COMMIT` for that same transaction ID.
4. No invalid or incomplete frame within the transaction.

The implementation must define how it distinguishes the end of a transaction and rejects duplicate, nested, or out-of-order transaction IDs. A simple initial implementation may permit only one in-flight transaction in the append log at a time.

For a batch:

- Validate the entire batch before writing.
- Append the complete transaction.
- Make the commit durable before returning success.
- Apply all batch changes to the in-memory index as one protected state transition.
- If the process crashes before a valid durable commit, recovery must not apply any operation from that batch.
- If the process crashes after success was acknowledged, recovery must apply the whole batch.

**Important:** Do not update the visible index operation-by-operation while a batch is still being committed.

## 7. Durability and write ordering

For POSIX-like systems, the intended sequence for a mutating transaction is:

1. Acquire the required exclusive database lock.
2. Validate arguments, limits, and capacity.
3. Encode all transaction frames.
4. Append `BEGIN`, operation frames, and `COMMIT`.
5. Check every write result and handle short writes.
6. Call `fsync()` on the database file descriptor and check its return value.
7. Only after successful synchronization, publish the transaction to the in-memory index.
8. Release the lock and report success / emit `ACK`.

`std::ofstream::flush()` is not a substitute for OS-level durable synchronization. The implementation should use a file descriptor and a carefully checked write loop (or another reviewed approach) for the durable log.

If any write or `fsync()` fails:

- Do not acknowledge success.
- Do not publish a partially written transaction to the in-memory index.
- Mark the database as needing recovery or enter a failed state if the append position is uncertain.
- On the next open, recovery decides whether the transaction has a valid commit.
- Never attempt an unsafe in-place truncation unless its crash behavior is understood and tested.

**TEAM DECISION:** Confirm the target OS. The `fsync()` examples assume POSIX. If the judging environment is Windows, select and document the equivalent durable-flush API. Also decide whether parent-directory synchronization is needed for initial file creation and rename operations.

## 8. Recovery procedure

On `open()`:

1. Open the database file using the agreed flags and acquire the process/thread ownership policy.
2. Validate the file header and version.
3. Scan frames from the first record to the end of file.
4. Validate frame magic, version, lengths, transaction ordering, and checksum.
5. Buffer records for the current transaction; do not publish uncommitted operations.
6. When a valid matching `COMMIT` is found, apply that transaction to the in-memory index.
7. Ignore a trailing incomplete/uncommitted transaction, according to the agreed append-tail policy.
8. If corruption occurs in the middle of the file (rather than a clearly incomplete final tail), fail `open()` or enter an explicitly documented corruption mode. Do not silently discard later records.
9. Position the append cursor safely for future writes.
10. Return from `open()` only when the index is ready.

Recovery must be idempotent: running it repeatedly on the same file must produce the same logical state.

**TEAM DECISION:** Define precisely which final-tail conditions are recoverable (for example, truncated final frame or a complete transaction without a commit) and which conditions indicate mid-file corruption. Add tests for each.

## 9. Concurrency and locking

Initial recommended model:

- One database-wide `std::shared_mutex`.
- Readers (`get`) take a shared lock.
- Mutations (`put`, `delete`, `batch`) take an exclusive lock.
- Compaction and close take an exclusive lock.
- The log append position and index updates are protected by the same exclusive transaction boundary.

If the team targets only C++17, `std::shared_mutex` is available. Avoid lock-free structures or fine-grained locking in the first implementation.

Requirements:

- No data race on the index, file descriptor, append offset, transaction ID, or database state.
- No lock is held while waiting for user input or performing unrelated CLI work.
- Document lock acquisition order if more than one lock is introduced.
- The single-process requirement does not automatically protect against two independent processes opening the same file. Either reject/lock out a second process or explicitly document that multi-process access is unsupported.

## 10. Compaction

Compaction rewrites only the current live key-value state into a fresh database file.

Proposed safe replacement sequence:

1. Acquire the exclusive database lock and stop concurrent mutations.
2. Create a uniquely named temporary file in the same directory.
3. Write a valid file header and one committed transaction (or a sequence of committed transactions) representing the live state.
4. Check all writes and synchronize the temporary file (`fsync`).
5. Atomically rename the temporary file over the database file, using the platform's supported atomic replacement semantics.
6. Synchronize the containing directory where required to make the rename durable.
7. Reopen/use the replacement file descriptor and update the append offset.
8. Release the lock.

The exact replacement strategy must account for open file descriptors and platform differences. A crash at any stage must leave either the old valid database or the new valid database; never delete the only valid copy first.

**TEAM DECISION:** Confirm temporary-file naming, rename semantics, directory synchronization, and file-descriptor swap behavior on the judging OS. Test process termination at each compaction stage.

## 11. Maximum file size and disk-full behavior

- Accept an optional maximum database size at open/create time.
- Define whether the limit includes the file header and all log/compaction overhead.
- Before appending, calculate the maximum possible encoded transaction size and check the configured limit.
- Still handle real OS write failures: pre-checking free space or configured size does not guarantee a later write will succeed.
- On `ENOSPC` or equivalent, do not acknowledge the operation.
- Do not corrupt or overwrite earlier committed records.
- Compaction may be needed to reclaim space, but compaction itself requires temporary space. Document and test what happens when insufficient temporary space exists.

**TEAM DECISION:** Define the exact error code/output and whether the database remains usable after a capacity error.

## 12. CLI contract

Proposed commands:

```text
kvdb open <path> [--max-bytes N]
kvdb put <path> <key> <value>
kvdb get <path> <key>
kvdb del <path> <key>
kvdb batch <path> <file>
kvdb dump <path>
kvdb check <path>
kvdb serve <path>
```

The exact command behavior must be consistent and documented in `README.md`.

- Exit codes must distinguish success from failure.
- `get` must distinguish missing keys from empty values.
- Mutating commands report success only after durable commit.
- `serve` prints `ACK <op-id>` only after durability is established.
- Batch parser must reject malformed input before writing anything.
- `dump` should have deterministic output suitable for comparison.
- `check` should report corruption without silently modifying the file.

**TEAM DECISION:** Finalize batch file syntax, quoting/escaping, stdout/stderr format, and exact exit codes.

## 13. Torture-testing contract

The torture tool must be independent of the database's in-memory index.

Minimum behavior:

1. Start the database process.
2. Generate deterministic random operations from a user-provided seed.
3. Run concurrent writers/readers as supported.
4. Persist every acknowledged operation in a separate oracle file before treating it as an expected durable result.
5. Randomly terminate the database process (for example, SIGKILL on POSIX).
6. Restart the database and allow recovery to finish.
7. Dump or query the recovered database.
8. Compare recovered state against the oracle.
9. Report the seed, iteration, operation history, ACK history, and first mismatch.
10. Support targeted modes for large values, batches, recovery, compaction, and disk-full behavior.

The default run should target at least 500 iterations if time permits and should be reproducible with the same seed.

Critical oracle rule: record an operation as expected durable only after receiving the database's success/ACK response. The oracle must not share the database's internal files or implementation logic.

## 14. Pull request and integration rules

Every PR must:

- Target `main` from the assigned feature branch.
- State the module and behavior changed.
- Include tests and exact commands used to run them.
- State known limitations and any contract changes.
- Avoid unrelated formatting or module rewrites.
- Pass the shared build and relevant tests.

Contract changes require discussion and approval from the team lead and affected module owners before implementation. If a change affects the binary format, write ordering, commit protocol, recovery, or locking, update this document and notify all four developers.

## 15. Acceptance checklist

### Storage engine
- [ ] Binary encoding/decoding is explicit and bounds-checked.
- [ ] PUT/GET/DELETE work after reopen.
- [ ] Empty and large values are handled as documented.
- [ ] Index rebuild matches the log.

### Durability and recovery
- [ ] Success is returned only after durable commit.
- [ ] Short writes and synchronization failures are handled.
- [ ] Incomplete final transaction is not partially applied.
- [ ] Acknowledged transactions survive forced termination.
- [ ] Batch recovery is all-or-nothing.
- [ ] Repeated recovery produces the same state.

### Concurrency and compaction
- [ ] Concurrent operations are race-free under the chosen lock model.
- [ ] Compaction preserves logical state.
- [ ] Crash during compaction leaves an old or new valid database.
- [ ] Maximum-size and disk-full paths do not acknowledge failed writes.

### CLI and torture tool
- [ ] Required commands and exit behavior are documented.
- [ ] `serve` ACKs only after durable commit.
- [ ] Oracle is independent and persists acknowledged operations.
- [ ] Seeded crash/restart test is reproducible.
- [ ] Targeted test modes exist or are explicitly listed as incomplete.

## 16. Required architecture evidence

`ARCHITECTURE.md` must explain, in the team's own words:

- Architecture diagram and module boundaries.
- Exact on-disk layout.
- Write ordering and the point at which success is acknowledged.
- Recovery algorithm and handling of incomplete/corrupt tails.
- Locking strategy.
- Compaction replacement sequence.
- At least 32 crash/failure scenarios with expected recovered state and reasoning.

`BREAKING.md` must record crash points, the ordering argument, known weaknesses, failed attempts, and next improvements.

`PROMPTS.md` must preserve the AI prompts chronologically and unedited, with tool and approximate time, as required by the event rules.

---

## 17. Team sign-off before coding

Each owner should confirm:

- [ ] I have read this contract.
- [ ] I understand the success/ACK durability rule.
- [ ] I understand that a batch must recover entirely or not at all.
- [ ] My module uses the agreed binary format and public interfaces.
- [ ] I will not silently change shared formats or protocols.
- [ ] I will add tests and document limitations in my PR.

### Open decisions to resolve in the first design meeting

1. Target operating system and durable-flush API.
2. Checksum algorithm and exact frame checksum coverage.
3. Exact integer encoding and maximum key/value/frame sizes.
4. Transaction ordering rules and transaction-ID policy.
5. Final-tail recovery versus mid-file corruption policy.
6. Process-exclusive file locking policy.
7. Compaction rename and directory-sync semantics.
8. Batch input syntax, CLI output, and exit codes.
9. Configured max-file-size accounting and disk-full recovery behavior.

**Do not treat the contract as finalized until these decisions are recorded and all module owners have reviewed it.**
