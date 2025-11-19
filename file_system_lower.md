# File System: Lower Layers

Okay, we've made it through most of the kernel! We've seen how xv6 boots up, manages memory with paging, creates and schedules processes, handles traps and system calls, and even interacts with the disk driver. Now it's time to put all that together and see how xv6 actually organizes data on the disk in a way that makes sense: the file system.

The file system is probably one of the most complex parts of any OS, and xv6's implementation is no exception. But don't worry -- the xv6 authors broke it down into nice, clean layers that build on top of each other. We're gonna tackle the lower layers first: the buffer cache, the logging layer for crash recovery, and the on-disk layout. Later, we'll see how inodes, directories, and pathnames work on top of these.

Quick note before we start: I'm assuming you've read the OSTEP chapters on file systems (chapters 39 through 44). If you haven't, go do that now! Seriously, the xv6 file system will make way more sense if you understand the general concepts first.

## The Big Picture

The xv6 file system is organized in six layers, stacked like a beautiful layer cake:

1. **Disk driver** (ide.c): We already covered this! It talks to the actual disk hardware.
2. **Buffer cache** (bio.c): Caches disk blocks in memory and synchronizes access to them.
3. **Logging layer** (log.c): Provides atomic transactions so crashes don't leave the file system in an inconsistent state.
4. **Inode layer** (fs.c, part 1): Manages unnamed files with inodes and block allocation.
5. **Directory layer** (fs.c, part 2): Implements directories as special inodes.
6. **Pathname layer** (fs.c, part 3): Handles path parsing like "/usr/rtm/xv6/fs.c".

This post will cover layers 2-4 (the "lower layers"), focusing on the parts in bio.c, log.c, and the block allocation / inode portions of fs.c. The next post will handle the upper layers with directories and pathnames.

Let's start from the bottom and work our way up!

## The Buffer Cache (bio.c)

Remember how we said that accessing the disk is super slow compared to accessing memory? Like, orders of magnitude slower? The buffer cache solves this problem by keeping copies of recently-used disk blocks in memory. That way, if the kernel needs a block it recently read, it doesn't have to go all the way to the slow disk again.

But that's not the only thing the buffer cache does! It also acts as a synchronization point for disk blocks. The cache makes sure that (1) only one copy of each disk block exists in memory at any time, and (2) only one kernel thread can use that copy at a time. This prevents all sorts of nasty race conditions where two processes might try to modify the same disk block simultaneously.

Let's start by looking at the data structures, starting with buf.h:

```c
struct buf {
  int flags;
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev;
  struct buf *next;
  struct buf *qnext;  // disk queue
  uchar data[BSIZE];
};
#define B_VALID 0x2  // buffer has been read from disk
#define B_DIRTY 0x4  // buffer needs to be written to disk
```

Each `buf` represents one disk block held in memory. Let's go through the fields:
- `flags`: Contains two bits we care about: `B_VALID` means the block's data has been read from disk and is up to date, and `B_DIRTY` means the block has been modified and needs to be written back to disk eventually.
- `dev` and `blockno`: Together these identify which disk block this buffer holds.
- `lock`: A sleep-lock (we'll cover sleep-locks in detail later, but for now just know it's a lock that lets a process sleep while waiting for it instead of spinning).
- `refcnt`: A reference count tracking how many pointers to this buffer exist. The buffer cache won't recycle a buffer while its refcnt is non-zero.
- `prev` and `next`: These link all the buffers together in a doubly-linked list, ordered by how recently they were used (most recent first). This is an LRU (Least Recently Used) cache!
- `qnext`: Used by the disk driver to queue up disk requests; we already saw this when we covered ide.c.
- `data`: The actual contents of the disk block, all BSIZE bytes of it (that's 512 bytes, defined in param.h).

Now let's look at the buffer cache structure itself:

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];
  
  // Linked list of all buffers, through prev/next.
  // head.next is most recently used.
  struct buf head;
} bcache;
```

The `bcache` structure contains:
- A spin-lock to protect access to the cache.
- An array of NBUF buffers (that's 30, defined in param.h).
- A dummy head node for the linked list of buffers. The list is circular, with the most recently used buffer at `bcache.head.next` and the least recently used at `bcache.head.prev`.

You might notice that NBUF is pretty small -- only 30 buffers! That means xv6 can only cache 30 * 512 = 15 KB of disk data at a time. For a modern computer with gigabytes of RAM, that's tiny. But remember, it's 1995, and xv6 is meant to be simple and educational, not high-performance.

### Initializing the Buffer Cache

The first function in bio.c is `binit()`, which gets called during boot from main():

```c
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

//PAGEBREAK!
  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```

This function initializes the lock protecting the cache, then builds the circular doubly-linked list. It starts with the head pointing to itself, then loops through all NBUF buffers in the array and inserts each one at the front of the list. The end result is that all the buffers are linked together, with `bcache.head.next` pointing to the last buffer in the array and `bcache.head.prev` pointing to the first one.

Note the `//PAGEBREAK!` comment in there -- this is a marker for the xv6 book's PDF generator to insert a page break. You'll see these scattered throughout the code.

### Getting a Buffer: bget()

Now for the most important function in the buffer cache: `bget()`. This function looks up a block in the cache, returning it if it's there, or allocating a new buffer for it if it's not. Either way, it returns a locked buffer.

```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached; recycle an unused buffer.
  // Even if refcnt==0, B_DIRTY indicates a buffer is in use
  // because log.c has modified it but not yet committed it.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0 && (b->flags & B_DIRTY) == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->flags = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

Let's walk through this carefully:

First, we grab the buffer cache lock. This is a spin-lock, so we'll busy-wait if another process is currently modifying the cache.

Then we search through the linked list to see if the block is already cached. If we find a buffer with matching dev and blockno, we increment its refcnt, release the cache lock, acquire the buffer's sleep-lock, and return it. Notice that we release the cache lock *before* acquiring the buffer's sleep-lock -- this is important! We don't want to hold the cache lock while sleeping, because that would prevent other processes from accessing the cache.

Why do we need both locks? The cache lock protects the structure of the cache itself (the linked list), while each buffer's lock protects the contents of that specific buffer. We use a spin-lock for the cache because operations on the cache structure are quick, but we use a sleep-lock for each buffer because operations on buffer contents might take a long time (like reading from or writing to disk).

If the block isn't in the cache, we need to allocate a buffer for it. We scan through the list backwards (from least recently used to most recently used) looking for a buffer with refcnt == 0 and the B_DIRTY bit clear. The refcnt == 0 means no one is currently using this buffer. The B_DIRTY check is subtle: even if refcnt is 0, if B_DIRTY is set, it means the logging layer has modified this buffer but hasn't yet written it to disk, so we can't recycle it yet. We'll understand this better when we look at log.c.

When we find a buffer we can recycle, we set its dev and blockno to the new block, clear its flags (it's not valid yet since we haven't read it from disk), set refcnt to 1, release the cache lock, acquire the buffer's sleep-lock, and return it.

If we can't find any buffer to recycle (all have non-zero refcnt or are dirty), we panic. In a real OS, we might try to write some dirty buffers to disk or increase the cache size, but xv6 keeps it simple and just gives up.

One more subtle point: notice that the function returns with the buffer's sleep-lock held! The caller is responsible for eventually releasing this lock with `brelse()`. This ensures that only one process can use the buffer at a time.

### Reading a Block: bread()

Now that we have `bget()` to get us a buffer, reading a block is easy:

```c
// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if((b->flags & B_VALID) == 0) {
    iderw(b);
  }
  return b;
}
```

We call `bget()` to get a buffer for the block. If the B_VALID flag isn't set, it means this buffer was just recycled and doesn't contain the actual block data yet, so we call `iderw()` (from ide.c, which we covered earlier) to read the block from disk. The disk driver will set B_VALID when it's done. Finally, we return the locked buffer.

Notice that if the block was already in the cache (B_VALID was set), we didn't need to touch the disk at all! That's the whole point of the cache.

### Writing a Block: bwrite()

```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  b->flags |= B_DIRTY;
  iderw(b);
}
```

This function writes a buffer to disk. First it checks that the caller holds the buffer's lock (this is a safety check). Then it sets the B_DIRTY flag and calls `iderw()` to write the buffer to disk.

Wait, why do we need B_DIRTY if we're writing to disk right now? Great question! It's because of the logging layer (which we'll cover soon). When the logging layer is active, `bwrite()` won't actually write directly to the final disk location. Instead, it'll write to the log. The logging layer will clear the B_DIRTY flag when it eventually writes the buffer to its final location.

### Releasing a Buffer: brelse()

When a process is done using a buffer, it calls `brelse()` to release it:

```c
// Release a locked buffer.
// Move to the head of the MRU list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

First we check that the caller holds the buffer's lock, then we release it.

Next, we grab the cache lock and decrement the buffer's refcnt. If refcnt drops to zero (meaning no one is using this buffer anymore), we move it to the front of the LRU list. This involves some classic doubly-linked list pointer manipulation:
1. Remove the buffer from its current position: make its neighbors point to each other.
2. Insert it at the front: make it point to the current front, make the head point to it.

Why do we move it to the front? Because this buffer was just used, so it's now the *most* recently used! By keeping the list ordered this way, `bget()` can easily find the least recently used buffer when it needs to recycle one (it's always at `bcache.head.prev`).

Finally, we release the cache lock.

And that's the buffer cache! Pretty straightforward: it's just an LRU cache with locks for synchronization. The buffer cache sits between the file system code and the disk driver, making disk accesses much faster and preventing race conditions.

## The Logging Layer (log.c)

Okay, now for one of the coolest parts of the file system: the logging layer. This layer provides *crash recovery* for the file system.

Here's the problem: most file system operations require modifying multiple disk blocks. For example, deleting a file might require (1) removing its directory entry, (2) updating the file's inode to mark it as free, and (3) updating the bitmap to mark its data blocks as free. If the system crashes or loses power in the middle of these updates, the file system could be left in an inconsistent state -- maybe the directory entry is gone but the inode is still marked as allocated, causing a "zombie" file that wastes disk space forever.

The logging layer solves this with *transactions*. File system code can wrap multiple disk writes into a single transaction, and the logging layer guarantees that either *all* of the writes happen, or *none* of them do. If the system crashes in the middle of a transaction, on reboot the file system will either replay the entire transaction or skip it entirely, but it'll never leave things half-done.

This is called a write-ahead log or journal. The idea is simple: instead of writing changes directly to their final locations on disk, we first write them to a special log area. Once we've safely written all the changes to the log, we write a commit record. Then we copy the changes from the log to their final locations. If we crash before the commit record, on reboot we'll see an incomplete transaction in the log and just ignore it. If we crash after the commit record, we'll replay the entire transaction by copying from the log.

### The Log Structure

Let's look at the data structures in log.c:

```c
// Contents of the header block, used for both the
// in-memory header and the on-disk header
struct logheader {
  int n;
  int block[LOGSIZE];
};

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing?
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
struct log log;
```

The `logheader` contains:
- `n`: The number of blocks currently in the log.
- `block`: An array of block numbers that are logged. LOGSIZE is 30, defined in param.h.

The `log` structure contains:
- `lock`: Protects the log structure.
- `start`: The block number where the log starts on disk.
- `size`: The total size of the log (in blocks).
- `outstanding`: A counter of how many file system system calls are currently executing. We can only commit when this reaches zero.
- `committing`: A flag indicating whether a commit is currently in progress.
- `dev`: The device number.
- `lh`: The in-memory copy of the log header.

On disk, the log looks like this:
```
[ header block ][ logged block 1 ][ logged block 2 ][ ... ]
```

The header block contains the logheader structure, and the logged blocks contain copies of the disk blocks being modified.

### Initializing the Log

```c
void
initlog(int dev)
{
  if (sizeof(struct logheader) >= BSIZE)
    panic("initlog: too big logheader");
  
  struct superblock sb;
  initlock(&log.lock, "log");
  readsb(dev, &sb);
  log.start = sb.logstart;
  log.size = sb.nlog;
  log.dev = dev;
  recover_from_log();
}
```

This function initializes the log during boot. It reads the superblock (we'll see what that is soon) to find out where the log is located on disk (`logstart`) and how big it is (`nlog`). Then it calls `recover_from_log()` to replay any committed transactions that might have been interrupted by a crash.

### Crash Recovery

```c
static void
recover_from_log(void)
{
  read_head();
  install_trans(); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```

During boot, we check if there are any committed transactions that need to be replayed. `read_head()` reads the log header from disk into memory. If there are any logged blocks (`log.lh.n > 0`), then `install_trans()` copies them from the log to their final disk locations. Finally, we set `log.lh.n` to 0 and write the header back to disk, clearing the log.

```c
// Read the log header from disk into the in-memory log header
static void
read_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *lh = (struct logheader *) (buf->data);
  int i;
  log.lh.n = lh->n;
  for (i = 0; i < log.lh.n; i++) {
    log.lh.block[i] = lh->block[i];
  }
  brelse(buf);
}
```

This reads the header block and copies its contents into the in-memory `log.lh`. Note the clever C here: we cast `buf->data` to a `struct logheader *`, which lets us treat the raw bytes in the buffer as a logheader structure. This works because the logheader structure layout matches the on-disk format.

```c
// Copy committed blocks from log to their home location
static void
install_trans(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    brelse(lbuf);
    brelse(dbuf);
  }
}
```

This function copies each logged block to its final destination. For each block number in the log header, we read the logged copy from the log area and the destination block, copy the data, write it to disk, and release both buffers.

Notice that we call `bwrite()` here, which will go through the IDE driver directly (not through the log) since we're called during recovery before the log is ready.

```c
// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}
```

This writes the in-memory log header to disk. This is the commit point -- once this returns, the transaction is committed and will be replayed if we crash.

### Beginning a Transaction

Every file system system call should start with `begin_op()` and end with `end_op()`. This tells the logging layer when a transaction starts and ends.

```c
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

This function waits until it's safe to start a new file system operation. It loops until two conditions are met:

1. No commit is currently in progress (`log.committing == 0`). If a commit is happening, we sleep until it finishes.

2. There's enough space in the log for this operation. The expression `log.lh.n + (log.outstanding+1)*MAXOPBLOCKS` estimates how many log blocks will be needed:
   - `log.lh.n`: Blocks already in the log from earlier operations
   - `(log.outstanding+1)`: The number of operations that will be in flight (including this new one)
   - `*MAXOPBLOCKS`: Each operation might write up to MAXOPBLOCKS blocks (that's 10, defined in param.h)

If there might not be enough space, we sleep and wait for a commit to clear the log.

Once both conditions are met, we increment `log.outstanding` to record that one more operation is in progress, and we return.

### Logging a Write

When file system code wants to write to a block, it calls `log_write()` instead of `bwrite()`:

```c
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache with B_DIRTY.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n)
    log.lh.n++;
  b->flags |= B_DIRTY; // prevent eviction
  release(&log.lock);
}
```

This function records that a block needs to be logged, but doesn't actually write anything to disk yet. Here's what it does:

First, it checks that we're not exceeding the log size and that we're inside a transaction.

Then it searches through the log header to see if we've already logged this block number in the current transaction. If we have, we don't need to log it again -- this is called log absorption. For example, if we write to the same block twice in one transaction, we only need one log entry for it.

If this is a new block, we add its block number to the log header and increment `log.lh.n`.

Finally, we set the B_DIRTY flag on the buffer. This is crucial: it tells the buffer cache not to recycle this buffer until the transaction commits. Remember in `bget()` when we checked for `(b->flags & B_DIRTY) == 0`? This is why.

### Ending a Transaction

When a file system operation is done, it calls `end_op()`:

```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

We decrement `log.outstanding`. If this was the last outstanding operation (outstanding reaches 0), we set `do_commit` and set `log.committing` to prevent new operations from starting. Otherwise, we wake up any operations that might be waiting in `begin_op()` -- now that we've finished, there might be more space available.

If we need to commit, we call `commit()` without holding the lock (since commit might sleep).

### Committing a Transaction

```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```

This is where the magic happens! Here's the commit process:

1. `write_log()`: Copy all the modified blocks from the buffer cache into the log on disk.
2. `write_head()`: Write the log header to disk with the updated count of logged blocks. This is the commit point! Once this write completes, the transaction is committed and will survive a crash.
3. `install_trans()`: Copy the logged blocks to their final destinations on disk.
4. Clear the log by setting `log.lh.n` to 0 and writing the header again.

```c
// Copy modified blocks from cache to log.
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to); // write the log
    brelse(from);
    brelse(to);
  }
}
```

For each block in the transaction, read the buffer cache version and the corresponding log block, copy the data, and write to disk.

That's the logging layer! It might seem complicated, but the key insight is simple: write to the log first, commit, then write to the final locations. This way, if we crash, we can always recover by either replaying the log (if it was committed) or ignoring it (if it wasn't).

## Disk Layout and Block Allocation (fs.c, part 1)

Now we've got the infrastructure in place -- the buffer cache for fast access and the log for crash recovery. Time to see how xv6 actually organizes data on the disk!

First, let's look at the on-disk layout. The disk is divided into several regions:

```
[ boot block | super block | log | inodes | free bitmap | data blocks ]
```

- **Block 0**: The boot block, which contains the boot loader code. We don't need this after boot.
- **Block 1**: The superblock, which contains metadata about the file system.
- **Blocks 2 to 2+log.size**: The log area, for crash recovery.
- **Next blocks**: Inodes, with multiple inodes per block.
- **Next blocks**: A bitmap tracking which data blocks are free.
- **Remaining blocks**: Data blocks, which hold file contents.

### The Superblock

The superblock (defined in fs.h) contains metadata about the file system:

```c
struct superblock {
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};
```

This structure tells us where everything is located on the disk. The file system reads this during initialization:

```c
// Read the super block.
void
readsb(int dev, struct superblock *sb)
{
  struct buf *bp;
  
  bp = bread(dev, 1);
  memmove(sb, bp->data, sizeof(*sb));
  brelse(bp);
}
```

Simple: read block 1, copy its data into the provided superblock structure. Notice the use of `memmove()` again to copy the raw bytes into our structure.

### Free Block Bitmap

xv6 tracks which data blocks are free using a bitmap. Each bit represents one data block: 0 means free, 1 means in use. Let's look at some helper macros from fs.h:

```c
#define BPB           (BSIZE*8)  // Bits per block
#define BBLOCK(b, ninodes) (b/BPB + (ninodes)/IPB + 3)
```

`BPB` is the number of bits in one block (512 * 8 = 4096 bits). `BBLOCK` calculates which bitmap block contains the bit for data block `b`. The formula `(ninodes)/IPB + 3` skips past the boot block, superblock, and inode blocks to find the start of the bitmap.

### Allocating a Block: balloc()

```c
// Allocate a zeroed disk block.
static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;
  struct superblock sb;

  bp = 0;
  readsb(dev, &sb);
  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb.ninodes));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}
```

This function allocates a free block by searching through the bitmap. Let's trace through it:

First, we read the superblock to find out how many blocks exist (`sb.size`).

Then we loop through the bitmap, one bitmap block at a time. For each bitmap block:
- Read the bitmap block with `bread()`.
- Loop through each bit in the block.
- Calculate a bitmask `m` for the current bit using `1 << (bi % 8)`. This creates a value with a single 1 bit in the right position.
- Check if the bit is 0 (free) by ANDing with the mask: `bp->data[bi/8] & m`. The `bi/8` finds which byte contains our bit.
- If we found a free block, mark it as in use with `bp->data[bi/8] |= m`, write the bitmap block back to disk with `log_write()` (not `bwrite()`!), and return the block number.
- Before returning, call `bzero()` to clear the newly allocated block.

If we get through all the blocks without finding a free one, panic.

The bit manipulation here is classic C: `bi/8` gives us the byte index, `bi % 8` gives us the bit position within that byte, and we use bitwise AND (`&`) to test bits and bitwise OR (`|`) to set bits.

### Zeroing a Block: bzero()

```c
// Zero a block.
static void
bzero(int dev, int bno)
{
  struct buf *bp;
  
  bp = bread(dev, bno);
  memset(bp->data, 0, BSIZE);
  log_write(bp);
  brelse(bp);
}
```

Simple: read the block, set all its bytes to 0 with `memset()`, log the write, and release the buffer.

### Freeing a Block: bfree()

```c
// Free a disk block.
static void
bfree(int dev, uint b)
{
  struct buf *bp;
  struct superblock sb;
  int bi, m;

  readsb(dev, &sb);
  bp = bread(dev, BBLOCK(b, sb.ninodes));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic("freeing free block");
  bp->data[bi/8] &= ~m;
  log_write(bp);
  brelse(bp);
}
```

This is the opposite of `balloc()`. We calculate which bitmap block and which bit corresponds to block `b`, check that it's actually in use (otherwise panic), then clear the bit with `bp->data[bi/8] &= ~m`. The `~m` flips all the bits in the mask, so we have 1s everywhere except the position we want to clear, and ANDing with that clears just that one bit.

## Inodes (fs.c, part 2)

Now for the heart of the file system: inodes. An inode (short for "index node") represents a file. It contains metadata about the file (like its type, size, and number of links to it) and pointers to the data blocks that hold the file's contents.

### On-Disk Inode Structure

Here's the on-disk inode structure from fs.h:

```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEV only)
  short minor;          // Minor device number (T_DEV only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

Let's go through the fields:
- `type`: The file type. Can be 0 (free inode), `T_DIR` (directory), `T_FILE` (regular file), or `T_DEV` (device).
- `major` and `minor`: For device files, these identify which device. Otherwise unused.
- `nlink`: The number of directory entries that point to this inode. When this reaches 0, the inode and its data blocks can be freed.
- `size`: The file size in bytes.
- `addrs`: An array of block numbers. The first NDIRECT entries (that's 12, defined in fs.h) are direct pointers to data blocks. The last entry is a pointer to an indirect block, which itself contains 128 more block numbers (since each block is 512 bytes and each block number is 4 bytes).

This means a file can have up to 12 + 128 = 140 data blocks, for a maximum file size of 140 * 512 = 71,680 bytes, or about 70 KB. That's tiny by modern standards, but remember xv6 is meant to be simple!

Some more useful constants from fs.h:

```c
#define IPB           (BSIZE / sizeof(struct dinode))  // Inodes per block
#define IBLOCK(i, ninodes)  ((i) / IPB + 2)
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)
```

`IPB` is how many inodes fit in one block (512 / 64 = 8). `IBLOCK` calculates which block contains inode `i`. `NINDIRECT` is how many block numbers fit in an indirect block (512 / 4 = 128). `MAXFILE` is the maximum number of blocks per file (140).

### In-Memory Inode Structure

The kernel keeps an in-memory cache of inodes, similar to the buffer cache:

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock;
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

The first few fields are for managing the cache:
- `dev` and `inum`: Together these identify which inode this is.
- `ref`: A reference count. The inode cache won't recycle an inode while ref > 0.
- `lock`: A sleep-lock protecting the inode.
- `valid`: Whether the inode's data has been read from disk yet.

The rest of the fields are copies of the on-disk inode.

The inode cache itself:

```c
struct {
  struct spinlock lock;
  struct inode inode[NINODE];
} icache;
```

Just like the buffer cache: a lock and an array of inodes. NINODE is 50, defined in param.h.

### Initializing the Inode Cache

```c
void
iinit(int dev)
{
  int i = 0;
  
  initlock(&icache.lock, "icache");
  for(i = 0; i < NINODE; i++) {
    initsleeplock(&icache.inode[i].lock, "inode");
  }

  readsb(dev, &sb);
  cprintf("sb: size %d nblocks %d ninodes %d nlog %d logstart %d\
 inodestart %d bmap start %d\n", sb.size, sb.nblocks,
          sb.ninodes, sb.nlog, sb.logstart, sb.inodestart,
          sb.bmapstart);
}
```

Initialize the lock, initialize the sleep-lock for each inode, read the superblock, and print some debug info about the file system layout. The `\` at the end of the `cprintf()` line is a line continuation -- it lets us break a long line across multiple lines in the source code.

### Getting an Inode: iget()

```c
// Find the inode with number inum on device dev
// and return the in-memory copy. Does not lock
// the inode and does not read it from disk.
static struct inode*
iget(uint dev, uint inum)
{
  struct inode *ip, *empty;

  acquire(&icache.lock);

  // Is the inode already cached?
  empty = 0;
  for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++){
    if(ip->ref > 0 && ip->dev == dev && ip->inum == inum){
      ip->ref++;
      release(&icache.lock);
      return ip;
    }
    if(empty == 0 && ip->ref == 0)    // Remember empty slot.
      empty = ip;
  }

  // Recycle an inode cache entry.
  if(empty == 0)
    panic("iget: no inodes");

  ip = empty;
  ip->dev = dev;
  ip->inum = inum;
  ip->ref = 1;
  ip->valid = 0;
  release(&icache.lock);

  return ip;
}
```

This is very similar to `bget()` from the buffer cache. We search through the inode cache to see if the inode is already there. If so, increment its reference count and return it.

While searching, we also remember the first free slot (ref == 0) we see.

If the inode isn't in the cache, we use the free slot we found, set its dev and inum, set ref to 1, mark it as not valid yet (we haven't read it from disk), and return it.

Note that `iget()` doesn't lock the inode or read it from disk! That's the caller's responsibility.

### Locking and Reading an Inode: ilock()

```c
// Lock the given inode.
// Reads the inode from disk if necessary.
void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic("ilock");

  acquiresleep(&ip->lock);

  if(ip->valid == 0){
    bp = bread(ip->dev, IBLOCK(ip->inum, sb.ninodes));
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```

First, we acquire the inode's sleep-lock.

Then, if the inode hasn't been read from disk yet (valid == 0), we read it. We use `IBLOCK()` to find which block contains this inode, then calculate the offset within that block with `ip->inum % IPB`. The expression `(struct dinode*)bp->data + ip->inum%IPB` is pointer arithmetic: we cast the buffer data to a `struct dinode*`, then add an offset to get to the right inode within the block.

We copy all the fields from the on-disk inode to the in-memory one, then set valid = 1.

### Unlocking an Inode: iunlock()

```c
// Unlock the given inode.
void
iunlock(struct inode *ip)
{
  if(ip == 0 || !holdingsleep(&ip->lock) || ip->ref < 1)
    panic("iunlock");

  releasesleep(&ip->lock);
}
```

Simple: just release the inode's sleep-lock, with some safety checks.

### Dropping a Reference: iput()

```c
// Drop a reference to an in-memory inode.
// If that was the last reference, the inode cache entry can
// be recycled.
// If that was the last reference and the inode has no links
// to it, free the inode (and its content) on disk.
// All calls to iput() must be inside a transaction in
// case it has to free the inode.
void
iput(struct inode *ip)
{
  acquiresleep(&ip->lock);
  if(ip->valid && ip->nlink == 0){
    acquire(&icache.lock);
    int r = ip->ref;
    release(&icache.lock);
    if(r == 1){
      // inode has no links and no other references: truncate and free.
      itrunc(ip);
      ip->type = 0;
      iupdate(ip);
      ip->valid = 0;
    }
  }
  releasesleep(&ip->lock);

  acquire(&icache.lock);
  ip->ref--;
  release(&icache.lock);
}
```

This function drops a reference to an inode. It's a bit tricky because it has to handle the case where the inode should be deleted.

First, we lock the inode and check if it's valid and has nlink == 0 (no directory entries point to it). If both are true, and this is the last reference (ref == 1), then we should free the inode and its data blocks. We call `itrunc()` to free all the data blocks, set type to 0 to mark the inode as free, call `iupdate()` to write the changes to disk, and set valid to 0.

Finally, we decrement the reference count.

The locking here is subtle. We have to check ref while holding the icache lock, then release it before calling itrunc() (which might sleep). This creates a small race condition window, but it's okay because we still hold the inode lock, which prevents anyone else from using the inode.

### Updating an Inode on Disk: iupdate()

```c
// Copy a modified in-memory inode to disk.
// Must be called after every change to an ip->xxx field
// that lives on disk.
// Caller must hold ip->lock.
void
iupdate(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  bp = bread(ip->dev, IBLOCK(ip->inum, sb.ninodes));
  dip = (struct dinode*)bp->data + ip->inum%IPB;
  dip->type = ip->type;
  dip->major = ip->major;
  dip->minor = ip->minor;
  dip->nlink = ip->nlink;
  dip->size = ip->size;
  memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
  log_write(bp);
  brelse(bp);
}
```

The opposite of reading an inode: we read the block containing the inode, find the right offset, copy all the fields from the in-memory inode to the on-disk one, and log the write.

### Getting a Block Number: bmap()

Now for one of the most important functions in the file system: `bmap()`. This function takes an inode and a logical block number within the file, and returns the actual disk block number. It also allocates new blocks if necessary.

```c
// Inode content
//
// The content (data) associated with each inode is stored
// in blocks on the disk. The first NDIRECT block numbers
// are listed in ip->addrs[].  The next NINDIRECT blocks are
// listed in block ip->addrs[NDIRECT].

// Return the disk block address of the bn'th block in inode ip.
// If there is no such block, bmap allocates one.
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

Let's trace through this carefully:

If `bn` (the logical block number) is less than NDIRECT (12), it's a direct block. We check if `ip->addrs[bn]` is 0 (not allocated yet). If it is, we allocate a new block with `balloc()` and store the result in `ip->addrs[bn]`. Then we return the block number.

If `bn` is 12 or higher, we subtract NDIRECT to get the index within the indirect block, then check that it's within range (less than NINDIRECT).

For indirect blocks, we first need to make sure the indirect block itself is allocated. We check `ip->addrs[NDIRECT]` and allocate it if necessary.

Then we read the indirect block with `bread()`. We cast its data to `uint*` so we can treat it as an array of block numbers. We check if `a[bn]` is 0, and if so, allocate a new block and store it in the array, logging the write.

Finally, we return the block number.

Notice that `bmap()` might modify the inode (by allocating new blocks and updating `ip->addrs`), but it doesn't call `iupdate()`. That's the caller's responsibility!

### Truncating a File: itrunc()

```c
// Truncate inode (discard contents).
// Only called when the inode has no links
// to it (no directory entries referring to it)
// and has no in-memory reference to it (is
// not an open file or current directory).
static void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

This function frees all the data blocks associated with an inode. It's called when a file is deleted.

First, we loop through all the direct blocks and free each one with `bfree()`.

Then, if there's an indirect block, we read it, loop through all its entries, and free each data block. Finally, we free the indirect block itself.

We set the inode's size to 0 and call `iupdate()` to write the changes to disk.

### Allocating an Inode: ialloc()

```c
// Allocate an inode on device dev.
// Mark it as allocated by  giving it type type.
// Returns an unlocked but allocated and referenced inode.
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;

  for(inum = 1; inum < sb.ninodes; inum++){
    bp = bread(dev, IBLOCK(inum, sb.ninodes));
    dip = (struct dinode*)bp->data + inum%IPB;
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp);
      return iget(dev, inum);
    }
    brelse(bp);
  }
  panic("ialloc: no inodes");
}
```

This function allocates a new inode by scanning through all the on-disk inodes looking for one with type == 0 (free).

For each inode number from 1 to ninodes (we skip inode 0 -- it's reserved), we read the block containing that inode, check if it's free, and if so, zero it out, set its type, log the write, and return a pointer to the in-memory inode via `iget()`.

Notice that we skip inode 0. By convention, inode number 0 is never used, so a block number or inode number of 0 can safely mean "not allocated yet."

### Reading and Writing Inodes: readi() and writei()

Finally, let's look at the functions that actually read and write file data:

```c
// Read data from inode.
// Caller must hold ip->lock.
int
readi(struct inode *ip, char *dst, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(ip->type == T_DEV){
    if(ip->major < 0 || ip->major >= NDEV || !devsw[ip->major].read)
      return -1;
    return devsw[ip->major].read(ip, dst, n);
  }

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > ip->size)
    n = ip->size - off;

  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    memmove(dst, bp->data + off%BSIZE, m);
    brelse(bp);
  }
  return n;
}
```

This function reads `n` bytes from an inode starting at offset `off`, copying them to `dst`.

If the inode is a device file (type == T_DEV), we call the device's read function from the device switch table `devsw`. We covered this back in the traps post.

Otherwise, we check that the offset and length are valid, adjusting `n` if necessary to not read past the end of the file.

Then we loop, reading one block at a time. For each iteration:
- Use `bmap()` to find the disk block number for the block containing offset `off`.
- Calculate how many bytes to copy from this block: either the remaining bytes we need (`n - tot`) or the bytes left in this block (`BSIZE - off%BSIZE`), whichever is smaller.
- Copy the bytes from the buffer to the destination with `memmove()`. The source is `bp->data + off%BSIZE` (the offset within the block).
- Release the buffer.

The loop variables update on each iteration: `tot` tracks total bytes copied, `off` advances to the next position in the file, and `dst` advances in the destination buffer.

```c
// Write data to inode.
// Caller must hold ip->lock.
int
writei(struct inode *ip, char *src, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(ip->type == T_DEV){
    if(ip->major < 0 || ip->major >= NDEV || !devsw[ip->major].write)
      return -1;
    return devsw[ip->major].write(ip, src, n);
  }

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    memmove(bp->data + off%BSIZE, src, m);
    log_write(bp);
    brelse(bp);
  }

  if(n > 0 && off > ip->size){
    ip->size = off;
    iupdate(ip);
  }
  return n;
}
```

This is very similar to `readi()`, but in reverse. We handle device files, check bounds (making sure we don't exceed MAXFILE * BSIZE), then loop through blocks.

For each block:
- Use `bmap()` to get (or allocate!) the disk block.
- Calculate how many bytes to write.
- Copy from `src` to `bp->data + off%BSIZE`.
- Log the write (not just `bwrite()`!).
- Release the buffer.

After the loop, if we extended the file (off > ip->size), we update the inode's size and call `iupdate()` to write it to disk.

Notice that `bmap()` will automatically allocate new blocks if we're writing past the current end of the file. Pretty neat!

## Wrapping Up

Whew! That was a lot, but we've covered all the lower layers of the xv6 file system:

1. **Buffer cache** (bio.c): Caches disk blocks in memory and synchronizes access to them with locks. Uses an LRU eviction policy.

2. **Logging layer** (log.c): Provides crash recovery through write-ahead logging. File system operations are wrapped in transactions that are atomic -- either all writes happen or none do.

3. **Block allocation** (fs.c): Manages free space using a bitmap. `balloc()` and `bfree()` allocate and free data blocks.

4. **Inodes** (fs.c): Represent files with metadata and block pointers. Support both direct and indirect blocks for a maximum file size of about 70 KB. Functions like `ialloc()`, `ilock()`, `iget()`, `iput()`, `bmap()`, `readi()`, and `writei()` manage inodes.

These layers build on each other: the logging layer uses the buffer cache, and the inode layer uses both logging and block allocation. It's a beautiful example of layered system design!

In the next post, we'll cover the upper layers: directories, pathnames, and the system calls that let user programs create, open, read, and write files. We'll also see how the file system gets used during boot to load and run the first user process. See you there!