---
date: 2026-96-11
modified: 2026-06-23T12:43:35+02:00
---
# The Persistence Guarantee
Every persistent database makes a fundamental promise: **once a transaction is committed, the changes it made will survive** a process crash, an OS failure, or a machine restart — as long as the storage medium itself is intact. Databases such as MySQL, PostgreSQL, and DynamoDB all uphold this guarantee, and the technique they universally rely on to do so is **write-ahead logging (WAL)**.

# What Actually Happens on Commit
When a transaction commits in a relational database, the changes must be **durably written to non-volatile storage**. Consider a simple `UPDATE` on a single row. At the storage level, that change touches multiple structures: the page in the B+ tree where the row lives, and any index pages that cover the affected column. Each of these is a separate block on disk, so a single logical write fans out into multiple physical disk I/Os. Making matters more complicated, a disk write is not a single atomic operation. Data moves through four layers before it reaches stable storage:
1. **In-memory flush buffer** — the write is first staged in RAM.
2. **OS buffer cache** — the operating system (Linux, Windows, etc.) maintains a block cache; the data passes through it on the way down.
3. **Disk cache** — most physical drives have their own on-board cache for frequently accessed blocks.
4. **Non-volatile storage** — the actual write to the SSD or magnetic disk happens last.

By default, a write call returns as soon as the data is handed off to the OS buffer cache, not when it has reached stable storage. To bypass the intermediate caches and guarantee durability, a file must be opened in synchronous (`O_SYNC`) mode or an explicit `fsync` call must be issued, forcing the OS to flush all pending writes down to the physical medium before returning.

> **What is `fsync`?** `fsync` is a system call (an instruction from a program to the operating system) that says: "I need you to flush everything you have buffered for this file all the way down to the physical disk right now, and only tell me when that is done." Without it, the OS decides on its own when to flush, which is faster but not safe for durability.

# The Core Idea of Write-Ahead Logging
Write-ahead logging addresses the tension between durability and performance with one principle: **before modifying any data file or index, record the change in a separate log file that is opened in synchronous mode.**

When a transaction commits, the database does not immediately flush the affected B+ tree pages and index blocks to disk. Instead, it appends the change — typically the SQL statement or a logical description of the operation — to the WAL file first. Only after that log entry is safely on disk is the commit acknowledged to the client. The actual data-file updates can then happen lazily, in the background, at whatever pace suits the system.

This works because of a key difference in how the writes are done:
- Writing to the WAL is a **sequential append**: you always add new entries at the end of the same file, one after another. This is very fast on both spinning disks and SSDs.
- Updating data files requires **random writes**: jumping to different, scattered locations on disk to update the B+ tree page here, the index page there. This is much slower because each jump takes time.

# Advantages of Write-Ahead Logging
**Performance through deferred writes.** Because the WAL captures what changed, the in-memory representations of data pages (the buffer pool) can absorb many commits without being flushed to disk after each one.

> **What is the buffer pool?** The buffer pool is the main in-memory cache of the database. When the database reads a page from disk (a B+ tree page, an index page, etc.), it keeps a copy in RAM inside the buffer pool. Future reads of the same page are served from RAM, which is much faster. When a transaction modifies a page, the change is made to the in-memory copy first — this modified copy is called a "dirty page". The buffer pool holds dirty pages until they are eventually written to disk.

The database periodically drains the buffer pool to disk asynchronously. Most databases expose this as a configurable flush frequency — for example, `innodb_flush_log_at_trx_commit` in MySQL's InnoDB engine allows you to choose between flushing on every commit (full durability) or batching flushes at, say, one-second intervals (higher throughput with a small durability window). The WAL is what makes the batched option safe: even if the buffer pool has not been flushed, the log file has a durable record of every committed transaction.

**Crash recovery.** If the database crashes with committed transactions in the buffer pool but not yet written to the data files, recovery is straightforward: replay the WAL from the last known checkpoint. Each log entry describes a change that should have been applied; replaying them brings the data files back into the state they would have been in had the crash not occurred.

**Reduced disk I/O.** By consolidating many logical writes into sequential log appends and deferring the scattered random writes to data files, WAL dramatically reduces the total number of disk write operations. This is one of the primary reasons well-tuned databases can sustain high transaction rates.

**Point-in-time recovery.** Because the WAL is an ordered, timestamped record of every mutation applied to the database, it is possible to reconstruct the database's state at any past moment. Starting from a base snapshot and replaying log entries up to a chosen point produces an exact image of the database as it existed at that time. This is invaluable for automation testing (restoring a known-good dataset from before a bug introduced inconsistent data) and for disaster recovery scenarios where you need to roll forward to the moment just before data corruption occurred.

# Protecting Log Entries with CRC
Although writing to the WAL is simpler than writing to data files, it is still not atomic at the hardware level. A 512-byte write to a spinning disk, or a multi-sector write to an SSD, can be interrupted mid-way by a power failure, leaving a partial record on disk. A corrupted log entry that is mistaken for a valid one during crash recovery would silently apply an incomplete change, producing data inconsistency.

To guard against this, every WAL entry is protected by a **cyclic redundancy check (CRC)** checksum.

> **What is a CRC checksum?** A checksum is a small number computed from a block of data using a mathematical formula. If even a single byte in the data changes (due to corruption or partial writes), the checksum value will be different. It works like a fingerprint: you compute it before writing, store it alongside the data, and then recompute it when reading. If the two values do not match, you know the data was corrupted.

Before appending an entry, the database computes the CRC of the entry's content, writes the CRC first, and then writes the entry body. During recovery, the CRC is verified before the entry is replayed. If the checksum does not match — indicating the entry was partially written or otherwise corrupted — the database stops replay at that point and reports an error, rather than silently applying a corrupt change.

# WAL File Structure
A WAL is an **append-only file** — entries are only ever added to the end, never modified or deleted in place. In practice, the WAL is not a single monolithic file but a sequence of **segment files**, each capped at a fixed size (16 MB in PostgreSQL).

> **Why use segment files instead of one big file?** Splitting the WAL into fixed-size segments makes management easier. Once a segment is no longer needed for recovery (because all its changes are already written to the data files and a checkpoint has passed), it can be safely deleted or archived without touching the rest of the WAL. It also avoids having a single file that grows indefinitely.

Within each segment, data is organized into **pages** of 8 KB — matching the block size that the OS uses when performing disk I/O — so every read or write to the WAL moves a whole page at a time.

Within this structure, each log entry is assigned a **Log Sequence Number (LSN)**. The LSN is not a simple integer counter; it encodes the **byte offset** of the entry within the log file.

> **What does "byte offset" mean here?** A byte offset is simply the position of the entry from the beginning of the file, measured in bytes. For example, an LSN might say "this entry starts at byte 4,096 of segment file number 3." This is more useful than a plain counter because the database can jump directly to that exact position in the file without reading everything before it.

This design choice has a practical benefit: given an LSN, the database can seek directly to that position in the segment file without scanning from the beginning. The database maintains a record of the LSN up to which log entries have been applied to the data files. On the next recovery or checkpoint pass, it resumes from exactly that offset, with no ambiguity about where to start.

Regarding what is logged: rather than recording the new state of every row affected by an operation, WAL typically logs the **statement** (or a compact logical representation of it). For an `UPDATE` that modifies a million rows, this means appending one compact log entry rather than a million row images. During replay, the statement is re-executed against the data files. This dramatically reduces the volume of data written to the WAL for large bulk operations.