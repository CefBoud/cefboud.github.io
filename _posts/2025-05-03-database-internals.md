---
title: "Taking a Look at Database Disk, Memory, and Concurrency Management"
author: cef
date: 2025-05-03
categories: [Technical Writing, Open Source]
tags: [Databases, Transactions, Book]
render_with_liquid: false
description: Buffer Pools, transactions, concurrency management and all that good stuff.
---


## Intro
Buffer pools, transactions, and concurrency management—these terms sound familiar, yet remain somewhat elusive. It's easy to work with transactions every day and still feel like they're a bit out of reach. Peeking behind the curtain is the only real way to demystify them. That's exactly what I set out to do in this article.

Right now, I'm having a great time reading *Database Design and Implementation*, which offers a magnificent exploration of databases. The book also walks you through building a simple database, SimpleDB, in Java. I'm following along by implementing it in Go: [CefDB](https://github.com/CefBoud/CefDB).

What follows are my notes from the book along with my implementation, sprinkled with a few extra insights along the way.

### Disk and File management

Disks under the hood consist of blocks, which are essentially arrays of bytes. Each block has a fixed size, typically a power of 2, such as 1024, 4096, or 8192 bytes.

It is possible to access these blocks directly using a program, such as a `dd`, although elevated permissions are required.

```shell
# Read the 100th 512-byte block of the device /dev/sda
root@sandbox:/home/ec2-user# dd if=/dev/sda  bs=512 skip=100 count=1
pp p?@P??i??P??8?|h?ppp?P??+??P??s.h?pp
p?P??+??P???s.h?pp?
                   p?P??+??P?ks.h?pp?p?P??+??P?۾s.h?pp?p?P??+??P?f's.h?pp?p?P??+??P???s.h?pp??P??+??P??}s.h?1+0 records in
1+0 records out
512 bytes copied, 6.0645e-05 s, 8.4 MB/s
```
`dd` can be used to back up a partition or create an image of a disk and save it to another storage device, like this:
`dd if=/dev/sda of=~/backup/sdadisk.img`.
This image can then be restored by simply swapping the `if` and `of` arguments:
`dd if=~/backup/sdadisk.img of=/dev/sda`.

Operating systems provide an additional method for reading and writing data to disks through the file system. Rather than working with raw blocks, the file system introduces an abstraction layer that organizes data into files and directories. When a client reads bytes from a file, they specify a starting position, without having to worry about the underlying block structure. The file system handles the conversion to and from the physical blocks behind the scenes.

```go
// read 3 bytes from `file.txt` starting a position 10
	file, _ := os.Open("file.txt")
	defer file.Close()
	file.Seek(int64(10), 0)
  bytes := make([]byte, 3)
	file.Read(bytes)
```

We can access disk bytes either at the block level or the file level.

Block-level access is very flexible and provides the database engine with complete control, allowing it to bypass file system limitations like file size restrictions. However, this approach comes with its own challenges. Different types of storage have varying low-level details, and while block-level access is powerful, the file system offers a convenient, portable abstraction. Additionally, the file system has cool features like efficient block management and metadata handling.

On the other hand, the database can rely exclusively on file-level access, but this has its own set of challenges. The database must manage data carefully in both memory and on disk in a very specific ways and the file-level lacks this flexibility. Relying on fixed blocks is very handy when managing access across different mediums.

To strike a balance, databases often use file-level access but rely on logical blocks within the file. Each file consists of a sequence of fixed-size blocks, and access to data is managed through this logical block interface.

To access the 3rd logical block, we can do the following:

```go
BLOCK_SIZE = 1024
file.Seek(BLOCK_SIZE*2, 0) // seek to the start of third block
bytes := make([]byte, BLOCK_SIZE)
file.Read(bytes) // read BLOCK_SIZE bytes 
```


When this block of disk data reaches memory, it is mapped to a page which is simply a block-sized area of memory.

Reading or writing data to disk always goes through memory. We read a block into a page, modify it, them we write it back.


The block and file management is quite simple to implement. We simply have a map of opening file that provide logical block operations:

```go
type BlockId struct {
	Filename string
	Blknum   int
}

type FileMgr struct {
	dbDirectory string
	blockSize   int
	isNew       bool
	openFiles   map[string]*os.File
	sync.Mutex
}

func (fm *FileMgr) getFile(filename string) (*os.File, error) {
	if file, ok := fm.openFiles[filename]; ok {
		return file, nil
	}

	filePath := filepath.Join(fm.dbDirectory, filename)
	file, err := os.OpenFile(filePath, os.O_RDWR|os.O_CREATE, 0666)
	if err != nil {
		return nil, fmt.Errorf("opening file '%v': %w", filePath, err)
	}
	fm.openFiles[filename] = file
	return file, nil
}

// Read reads a block from the specified BlockId into the Page.
func (fm *FileMgr) Read(blk *BlockId, p *Page) error {
	fm.Lock()
	defer fm.Unlock()

	file, err := fm.getFile(blk.Filename)
	if err != nil {
		return fmt.Errorf("getting file '%v': %w", blk.Filename, err)
	}

	offset := int64(blk.Blknum) * int64(fm.blockSize)
	_, err = file.Seek(offset, io.SeekStart)
	if err != nil {
		return fmt.Errorf("seeking to offset %d in file '%v': %w", offset, blk.Filename, err)
	}

	_, err = io.ReadFull(file, p.Contents())
	if err != nil {
		return fmt.Errorf("reading block %v from file '%v': %v", blk, blk.Filename, err)
	}

	return nil
}

// Write writes the contents of the Page to the specified BlockId.
func (fm *FileMgr) Write(blk *BlockId, p *Page) error {
	fm.Lock()
	defer fm.Unlock()

	file, err := fm.getFile(blk.Filename)
	if err != nil {
		return fmt.Errorf("getting file '%v': %v", blk.Filename, err)
	}

	offset := int64(blk.Blknum) * int64(fm.blockSize)
	_, err = file.Seek(offset, io.SeekStart)
	if err != nil {
		return fmt.Errorf("seeking to offset %d in file '%v': %w", offset, blk.Filename, err)
	}

	_, err = file.Write(p.Contents())
	if err != nil {
		return fmt.Errorf("writing block %v to file '%v': %w", blk, blk.Filename, err)
	}

	// Ensure the write is persisted to disk
	if err := file.Sync(); err != nil {
		return fmt.Errorf("syncing file '%v': %w", blk.Filename, err)
	}

	return nil
}

// Append appends a new block to the specified file and returns the BlockId of the new block.
func (fm *FileMgr) Append(filename string) (*BlockId, error) {
	fm.Lock()
	defer fm.Unlock()

	file, err := fm.getFile(filename)
	if err != nil {
		return nil, fmt.Errorf("getting file '%v': %w", filename, err)
	}

	lastOffset, err := file.Seek(0, io.SeekEnd)
	if err != nil {
		return nil, fmt.Errorf("failed seeking to end of file '%v': %v", filename, err)
	}

	newBlockId := int(int(lastOffset) / fm.blockSize)
	// Create a buffer of zeros for the new block
	b := make([]byte, fm.blockSize)
	_, err = file.Write(b)
	if err != nil {
		return nil, fmt.Errorf("appending block to file '%v': %v", filename, err)
	}

	// Ensure the write is persisted to disk
	if err := file.Sync(); err != nil {
		return nil, fmt.Errorf("syncing file '%v': %v", filename, err)
	}

	return &BlockId{Filename: filename, Blknum: newBlockId}, nil
}

```

### Disk vs Memory
So DBs store their files in disks. Two big categories of disk compete: 
 - Hard Disk Drive: physical, high latency, cheap
 - Solid State Drive: lower latency, more expensive but getting cheaper. 

 SATA is a serial protocol used by HDD and SSD. NVMe is a new protocol used by SSD and offer better latency and speed.
 


| **Feature**                  | **SATA**                                   | **NVMe**                                       | **RAM**                                   |
|------------------------------|--------------------------------------------|-----------------------------------------------|-------------------------------------------|
| **Max Speed**                 | Up to 600 MB/s                             | Up to 7,000 MB/s (for PCIe Gen 4)              | Up to 50,000 MB/s (DDR5)                  |
| **Latency**                   | ~100,000 nanoseconds (ns)                  | ~50,000 nanoseconds (ns)                       | ~50 nanoseconds (ns)                      |
| **Power Efficiency**          | Higher power consumption                   | Lower power consumption (especially with M.2 form factor) | Low power consumption (especially in modern DDR5) |
| **Cost**                      | Generally cheaper                          | More expensive due to high performance         | More expensive (depends on capacity and speed) |


NVMe still pales in comparison to RAM. It has 1000 times lower latency and about 10 times better throughput. This highlights a fundamental principle when building a DB engine: minimize disk accesses and rely on memory as much as possible.

### Memory Management: Buffer Pools and WAL


A DB engine uses memory pages to make disks block available. Each memory pages (an array of bytes) corresponds to a disk block. Writing to disk means writing to a memory page first then flushing that page to disk. Reading from disk is the same thing. We read a file into a page (again, byte array). One of the fundamental principle in DB engines is to minimize disk access. Given how fast memory is compared to disk (), we want to only flush to disk when necessary.

#### The Log
When a user writes data to a database table, behind the scenes, it's a write to a logical block within one of the database files. So, a client interacting with the database is essentially reading and writing to files—more specifically, to logical blocks that correspond to database tables and records within the file.

When a user makes a change to a block, the database needs to track that change in case it needs to be reversed. Each change operation is recorded in a log as a log record. The advantage of using a log is that once we know the change has been persisted there, we don't need to immediately flush the modified pages to disk.

For example, if a client modifies 100 pages (blocks) in memory, and all these changes are written to the log. Flushing the log to disk is sufficient to ensure the durability of the changes even after a system crash. This is because once the log has been flushed from memory to disk, we don't need to flush all 100 modified blocks. Flushing just the log block is enough. During crash recovery, replaying the log and applying each change restores the database to its pre-crash state.

While data blocks still need to be flushed, relying on the log allows us to batch multiple flushes together, reducing the number of disk accesses and improving performance.

#### Buffer Pool
So, the user data is held in memory pages. The DB engine allocates a fixed set of pages called the **buffer pool** (fancy!), managed by the **Buffer Manager** (BM).

The buffer pool is critical; it's actually the bulk of the memory used by the DB. It should naturally fit the machine's RAM. When a client wants to read data from disk, it asks the BM for it, and the BM **pins** the block (a chunk of the file) into the page. As long as the client is using that block, the page is pinned. When the client finishes, the BM can **unpin** the page, thus allowing another block to be loaded.

The DB's memory management is this dance between disk blocks, memory pages, and clients. The song and rhythm are governed by the previously mentioned principle: minimize disk access because it's painfully slow.

The BM doesn't necessarily have to remove a block from a page when a client finishes, unless that page needs to be reused. This means that if another client needs that block, we can save ourselves a disk access since the data is already in memory. This also means that when we need to replace one of multiple recently unpinned pages to make room for a new block, the choice is very important: replacing a block that will be needed in the near future, instead of one that won't be needed for a long time, proves to be wasteful because it requires us to make an additional disk read.

Techniques exist to pick the replacement:
* Naive (naively pick the first available page)
* FIFO (replace the oldest)
* LRU (replace the Least Recently Used)

Other techniques exist, but the idea is to minimize disk operations and keep relevant soon-to-be-needed data in memory.

In case the DB is heavily used and there are no available memory pages, a client will timeout waiting for a page and it can retry later.

Let's take a look at some code.

```go
// Buffer manages a page of data in memory, associated with a disk block.
type Buffer struct {
	fm       *file.FileMgr
	lm       *log.LogMgr
	contents *file.Page
	blk      *file.BlockId
	pins     int
	txnum    int
	lsn      int // lsn of the most recent related log record
	sync.Mutex
}

// Pin increases the buffer's pin count.
func (b *Buffer) Pin() {
	b.Lock()
	defer b.Unlock()
	b.pins++
}

// Unpin decreases the buffer's pin count.
func (b *Buffer) Unpin() {
	b.Lock()
	defer b.Unlock()
	b.pins--
}

// AssignToBlock reads the contents of the specified block into the buffer.
// If the buffer was dirty, its previous contents are first written to disk.
func (b *Buffer) AssignToBlock(blk *file.BlockId) error {
	b.Lock()
	defer b.Unlock()
	b.Flush()
	b.blk = blk
	err := b.fm.Read(b.blk, b.contents)
	if err != nil {
		return err
	}
	b.pins = 0
	return nil
}

// Flush writes the buffer to its disk block if it is dirty.
func (b *Buffer) Flush() error {
	if b.txnum >= 0 {
		// flush log record first
		err := b.lm.Flush(b.lsn)
		if err != nil {
			return err
		}
		// flush page
		err = b.fm.Write(b.blk, b.contents)
		if err != nil {
			return err
		}
		b.txnum = -1
	}
	return nil
}


type BufferMgr struct {
	bufferpool   []*Buffer
	numAvailable int
	mu           *lock.CASMutex
}


func (bm *BufferMgr) tryPin(blk *file.BlockId) *Buffer {
	b := bm.findExistingBuffer(blk)
	if b == nil {
		b = bm.chooseUnpinnedBuffer()
		if b == nil {
			return nil
		}
		b.AssignToBlock(blk)
	}

	if !b.IsPinned() {
		bm.numAvailable--
	}
	b.Pin()
	return b
}

func (bm *BufferMgr) findExistingBuffer(blk *file.BlockId) *Buffer {
	for _, b := range bm.bufferpool {
		if b.blk != nil && *b.blk == *blk {
			return b
		}
	}
	return nil
}

func (bm *BufferMgr) chooseUnpinnedBuffer() *Buffer {
	// naive buffer search, return first unpinned.
	// Alternatives: LRU, FIFO ...
	for _, b := range bm.bufferpool {
		if !b.IsPinned() {
			return b
		}
	}
	return nil
}

```

As we can see, a Buffer is simply a memory page (byte array) with some extra information: the number of clients currently using it (pin), the last log sequence number (LSN) of its last change (last log record), and, most importantly, the block whose data it's currently holding.

What about the Buffer Pool? It's simply an array of buffers. When I hear "Buffer Pool," I always picture something fancy and complex, but it's really just a group of buffers which are really just byte arrays.

The methods `tryPin`, `findExistingBuffer`, and `chooseUnpinnedBuffer` do exactly what their names suggest. Their implementations are extremely straightforward, but I believe they convey the essence of what they're meant to do.

### Concurrency and Transactions

Transactions are, in my opinion, one of the most fascinating aspects of databases. They are the reason databases can handle concurrent requests without the risk of data inconsistencies.

Let's understand the problem. A canonical example is an e-commerce product table with inventory information. We have one item left. Two users log in at almost the same time and try to buy the same item. It would be problematic if both thought they got it.

If we translate this into terms of blocks and buffers, the two users are both interested in the same disk block that holds the item record, and this block is requested and moved into a memory buffer. It is pinned twice by the two users. If they both want to write to that buffer, that would be trouble.

Let's illustrate that using SQL.

```go
sqlClient.Execute("SELECT count FROM ITEMS WHERE item_id = ?", itemID).Scan(&available)
sqlClient.Execute("UPDATE ITEMS SET count = ? WHERE item_id = ?", available-1, itemID)

```

If both clients run the `SELECT` and then stop, they both think `count == 1`. Then they will both decrease the count to 0, and the count will be updated to 0 twice.

How do we ensure that doesn't happen? Transactions!

If User A wants to decrease the available item count, they should start a transaction and **LOCK** that item. Once locked, if User B tries to do the same, it will attempt to acquire the lock before doing so. If the lock is already taken, User B won't be able to proceed.

That's the magic behind locks. At its heart, that's how databases ensure that different users/clients don't step on each other's toes.

We can't, of course, talk about transactions without mentioning ACID. Not the substance, but rather the four properties that transactions guarantee. Transactions allow multiple operations happening simultaneously (concurrently) to act as if each were happening on its own. The properties are:

* **Atomicity**: All steps of the transaction are either committed or none are.
* **Consistency**: The transaction results in a valid and consistent state (e.g., not an inventory of -1).
* **Isolation**: Each transaction runs as if it were the only one executing.
* **Durability**: Once committed, a transaction's changes are permanent and persisted to disk.


So how do transactions and locks achieve these properties ? Log records and locks.

#### Log Records
Each time a transaction-related operation occurs, it is written to the log:

* When a transaction starts.
* When a transaction ends, either by commit (persisting all changes) or rollback (reverting all changes).
* An update record is written for each change made to disk.


Here is an example:
```
START 0
UPDATE 0 BLOCK 1 OFFSET 50 1 2
START 1
UPDATE 1 BLOCK 2 OFFSET 100 oldString newString
COMMIT 0
ROLLBACK 1
```

Start transaction 0, update block 1 at offset 50 and replace value 1 with 2.
Start transaction 1, then replace 'oldString' with 'newString' in block 2 at offset 100.

The level of granularity used in our example is the value, meaning changes made by a transaction are expressed as values (integer, string) in a block at certain offsets. Other levels of granularity are possible. We can express changes at different levels:

* row/record
* block

Each log record update will contain the old and new values of the updated item. Larger granularity results in fewer log updates, but those updates would be larger.

Using our log, we can roll back transactions by simply reverting their updates. We can also recover from a crash by reverting all unfinished transactions (those that are neither committed nor rolled back).

The recovery manager is responsible for this. Upon startup, it reads the log starting from the end and reverts all unfinished transactions `UNDO`. This ensures that our DB state contains only committed data.

However, there are some caveats. If data blocks are flushed to disk with log records when a transaction commits, then recovery would only involve reverting unfinished transactions. But we've discussed the ability to delay flushing buffers and flush only the log to avoid making too many disk requests. In that case, recovery would require applying updates from committed transactions that have unflushed data blocks. This is called `UNDO-REDO`

As DB operations accumulate, the log can grow quite large, potentially exceeding the size of the data files. To address this, we can create checkpoints, which are a type of log record indicating that all previous log records can be discarded. 

A checkpoint is created when we ensure that all completed transactions have been flushed to disk and that no updates from incomplete transactions remain. This last condition can be relaxed if we track unfinished transactions in the checkpoint, allowing us to account for them during recovery.

The core idea behind checkpoints is to treat the database at the checkpoint moment as a consistent state, meaning there is no need for recovery from earlier points in the log.

#### Concurrency Management

The holy grail of DB concurrency is **serializability**. This means its transactions run as if they were executed serially—one after the other—even though they are running concurrently.

```
Tx1: Write(B1), Write(B2)
Tx2: Write(B3), Write(B4)

W1(B1), W2(B3), W2(B4), W1(B2)

Tx3: Write(B5), Read(B6)
Tx4: Write(B5), Write(B6)

W3(B5), W4(B5), W4(B6), R3(B6)
```

Tx1 and Tx2 are interleaved and are running concurrently, but given that they are changing different blocks, their execution is serializable and produces the same result as if they were run serially. This is a serializable schedule. It is equivalent to running Tx1 then Tx2 or Tx2 then Tx1 in complete isolation.

Tx3 and Tx4, on the other hand, do not look so good. Tx3 writes to B5 and is then interrupted by Tx4, which writes to both B5 and B6. Tx3 reads B6, which was changed by Tx4, and this data differs from the value of B6 when Tx3 started. This is not serializable.

A serializable run of transactions is considered correct regardless of the order of transactions. The job of the concurrency manager is to ensure that. As we explained earlier, the magic behind that is **locking**.

For each block or unit of concurrency (it could be a row), there is a **read** or **shared** lock (Slock), and there is a **write** or **exclusive** lock (Xlock). To guarantee a serializable schedule, the algorithm is elegantly simple:

* Before reading a block, acquire a shared lock.
* Before updating a block, acquire an exclusive lock.
* Release all locks on commit or rollback.

That's it. These 3 fabulous rules ensure that transactions run according to a serializable schedule.

Why does it work? Well, we can express it differently: you can't read a block if someone is modifying it, and you can't modify a block if someone is reading it. Note that a block can be read (shared for reading) by multiple transactions because that's harmless. The harm arises from reading modified data or modifying data that is being read by a different transaction. 

One way to think of it is that a running transaction must have the blocks/rows it is interacting with unchanged for the duration of its execution, as if it's the only transaction running and using that part of the DB.

How do we implement that? Using a global lock table. Whenever a Tx wants access to a block, it requests its lock. Slock for reading and Xlock for modifying.

```go

// global lock map storing blockId => *sync.RWMutex{}
// very important to store pointers as mutex must not be copied
var globalLockMap sync.Map

type ConcurrencyMgr struct {
	CurrentLocks map[file.BlockId]int
}

func (cm *ConcurrencyMgr) SLock(blk *file.BlockId) bool {
	// if we already have a lock (S or X), we return
	if _, ok := cm.CurrentLocks[*blk]; ok {
		return ok
	}

	v, _ := globalLockMap.LoadOrStore(*blk, lock.NewCASMutex())
	l := v.(*lock.CASMutex)
	ok := l.RTryLockWithTimeout(MAX_TIME)
	if ok {
		cm.CurrentLocks[*blk] = S_LOCK
	}
	return ok
}

func (cm *ConcurrencyMgr) XLock(blk *file.BlockId) bool {

	if v, ok := cm.CurrentLocks[*blk]; ok {
		// if we already have a X_LOCK, we return early
		if v == X_LOCK {
			return ok
		}
		// we have a S_LOCK, we release it to upgrade to an X_LOCK
		cm.SUnlock(blk)
	}

	v, _ := globalLockMap.LoadOrStore(*blk, lock.NewCASMutex())
	l := v.(*lock.CASMutex)
	ok := l.TryLockWithTimeout(MAX_TIME)
	if ok {
		cm.CurrentLocks[*blk] = X_LOCK
	}
	return ok
}


// Transaction provides transaction management for clients,
// ensuring that all transactions are serializable and recoverable.
type Transaction struct {
	recoveryMgr *RecoveryMgr
	concurMgr   *ConcurrencyMgr
	bm          *buffer.BufferMgr
	fm          *file.FileMgr
	txnum       int
	mybuffers   map[file.BlockId]*buffer.Buffer
}

func (tx *Transaction) GetInt(blk *file.BlockId, offset int) (int, error) {
	ok := tx.concurMgr.SLock(blk)
	if !ok {
		return 0, fmt.Errorf("unable to acquire Slock for %v", blk)
	}

	buff := tx.mybuffers[*blk]
	return buff.Contents().GetInt(offset), nil
}

func (tx *Transaction) SetInt(blk *file.BlockId, offset int, val int) error {
	ok := tx.concurMgr.XLock(blk)
  
	if !ok {
		return fmt.Errorf("unable to acquire Xlock for %v", blk)
  }
  // ...
	buff.Contents().SetInt(offset, val)
	return nil
}
```

So there a `globalLockMap` and each transaction has a `concurrencyManager` that interacts with this global.

Notice how the transaction request an Slock before reading (`GetInt`) and a Xlock before writing (`SetInt`).

Before calling it a day, I want to talk about *phantoms* (cool name, huh?).

Consider the following example, where we have a budget and want to distribute it evenly among our employees.

```go
sqlClient.Execute("SELECT count(*) FROM EMPLOYEE").Scan(&numEmployees)
sqlClient.Execute("UPDATE EMPLOYEE_BUDGET SET amount = amount + ? ", totalBudget/float64(numEmployees))
```

What happens if a new employee is added to the database between the `SELECT` and the `UPDATE`? Well, we'll end up with an employee who is budget-less!
It's a contrived example, granted, but it illustrates the concept of *phantoms* which occur when the number of records or blocks can change during a transaction. 
To prevent this, an additional lock can be employed to prevent new records or blocks from being appended while the transaction is in progress, thus defending against phantoms.

## Conclusion
Thanks for getting this far! I hope you learned a thing or two.

You can find me on Twitter/X at [@moncef_abboud](https://x.com/moncef_abboud).  
