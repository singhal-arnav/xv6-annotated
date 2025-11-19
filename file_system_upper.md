# File System: Upper Layers

Welcome back! In the last post, we covered the lower layers of the xv6 file system: the buffer cache, logging, block allocation, and inodes. Those layers gave us everything we need to read and write blocks of data, but they didn't give us anything that looks like the file systems we're used to -- no directories, no pathnames like `/usr/bin/ls`, no way to actually navigate the file system tree.

That's what we're tackling in this post: the upper layers of the file system. We'll see how directories work (spoiler: they're just special files!), how path names get resolved into inodes, and how file descriptors provide a convenient abstraction for user programs.

We've already covered the system call implementations in the previous post, so here we'll focus on the infrastructure that makes those system calls possible: the directory operations, pathname resolution, and file descriptor management. This is where everything comes together!

## Directories (fs.c, part 3)

In Unix, directories are just special files. Like regular files, directories are represented by inodes, but their type is T_DIR instead of T_FILE, and their contents follow a specific format: they contain a sequence of directory entries.

### Directory Entries

Each directory entry is a `dirent` structure, defined in fs.h:

```c
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

Simple! Each entry contains:
- `inum`: The inode number of the file this entry refers to
- `name`: The file name, up to DIRSIZ (14) characters

A directory's data blocks contain a sequence of these dirent structures, one after another. If `inum` is 0, that entry is empty and available for use. Otherwise, `name` is the name of a file, and `inum` is its inode number.

**This is a key insight**: directory entries don't contain file metadata or data -- they just map names to inode numbers. All the actual file information lives in the inode itself. This is what makes hard links possible: multiple directory entries can point to the same inode!

Another subtle point: file names in xv6 are limited to 14 characters. That's pretty restrictive by modern standards (most systems allow 255 characters or more), but remember it's 1995 and xv6 is designed to be simple. If you need longer names, you'd have to modify DIRSIZ and reformat the disk.

### Comparing Names: namecmp()

Before we look at the directory functions, here's a tiny helper:

```c
int
namecmp(const char *s, const char *t)
{
  return strncmp(s, t, DIRSIZ);
}
```

This compares two file names, but only up to DIRSIZ characters. It's just a wrapper around `strncmp()`. We use this instead of `strcmp()` because directory entry names aren't always null-terminated -- if a name is exactly 14 characters long, there's no room for the null terminator!

### Looking Up a Name: dirlookup()

Now for the fun stuff. The `dirlookup()` function searches a directory for a specific name:

```c
// Look for a directory entry in a directory.
// If found, set *poff to byte offset of entry.
struct inode*
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de;

  if(dp->type != T_DIR)
    panic("dirlookup not DIR");

  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlookup read");
    if(de.inum == 0)
      continue;
    if(namecmp(name, de.name) == 0){
      // entry matches path element
      if(poff)
        *poff = off;
      inum = de.inum;
      return iget(dp->dev, inum);
    }
  }

  return 0;
}
```

Let's walk through this:

First, we check that `dp` is actually a directory inode. If not, panic -- this is a programming error.

Then we loop through the directory's contents, one dirent at a time:
- Read the dirent using `readi()` (from the inode layer). We read directly into the `de` structure.
- If `de.inum` is 0, this entry is empty, so skip it.
- If the name matches what we're looking for, we found it! Save the offset in `*poff` (if the caller wants it), grab the inode number, and return the inode via `iget()`.

If we get through the whole directory without finding the name, return 0 (NULL).

**Important**: Notice that we return an *unlocked* inode via `iget()`. This is crucial for avoiding deadlocks. Consider what happens if someone looks up "." (the current directory): we're already holding a lock on `dp`, so if we tried to lock the inode before returning, we'd try to lock `dp` again and deadlock!

The caller is responsible for locking the returned inode if they need to use it. This design gives the caller flexibility and prevents lock ordering issues.

Also notice that `dirlookup()` only searches a single directory -- it doesn't handle paths with slashes like "usr/bin". We'll see path handling shortly.

### Adding an Entry: dirlink()

The opposite of looking up a name is adding one:

```c
// Write a new directory entry (name, inum) into the directory dp.
int
dirlink(struct inode *dp, char *name, uint inum)
{
  int off;
  struct dirent de;
  struct inode *ip;

  // Check that name is not present.
  if((ip = dirlookup(dp, name, 0)) != 0){
    iput(ip);
    return -1;
  }

  // Look for an empty dirent.
  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlink read");
    if(de.inum == 0)
      break;
  }

  strncpy(de.name, name, DIRSIZ);
  de.inum = inum;
  if(writei(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
    panic("dirlink");

  return 0;
}
```

First, we check if the name already exists in the directory. If it does, that's an error -- you can't have two files with the same name in the same directory. We call `iput()` to release the inode reference and return -1 to indicate failure.

Then we search for an empty directory entry (one with inum == 0). We loop through the directory until we find one, or until we reach the end of the directory.

Once we've found a slot (or reached the end), we create a new dirent: copy the name with `strncpy()` (which pads with nulls if the name is shorter than DIRSIZ), set the inode number, and write it to the directory with `writei()`.

**A subtle point**: If we reach the end of the directory (`off == dp->size`), `writei()` will automatically extend the directory by allocating a new block. Remember from the inode layer, `writei()` calls `bmap()` which allocates blocks as needed. So directories can grow dynamically as files are added!

And that's directories! They're surprisingly simple: just files containing a sequence of (name, inode number) pairs.

## Pathnames (fs.c, part 4)

Okay, now we can search a single directory for a name. But what about hierarchical paths like `/usr/bin/ls`? That's what the pathname layer handles.

### Parsing Path Elements: skipelem()

First, we need a function to parse paths. The `skipelem()` function extracts one component from a path:

```c
// Copy the next path element from path into name.
// Return a pointer to the element following the copied one.
// The returned path has no leading slashes,
// so the caller can check *path=='\0' to see if the name is the last one.
// If no name to remove, return 0.
//
// Examples:
//   skipelem("a/bb/c", name) = "bb/c", setting name = "a"
//   skipelem("///a//bb", name) = "bb", setting name = "a"
//   skipelem("a", name) = "", setting name = "a"
//   skipelem("", name) = skipelem("////", name) = 0
//
static char*
skipelem(char *path, char *name)
{
  char *s;
  int len;

  while(*path == '/')
    path++;
  if(*path == 0)
    return 0;
  s = path;
  while(*path != '/' && *path != 0)
    path++;
  len = path - s;
  if(len >= DIRSIZ)
    memmove(name, s, DIRSIZ);
  else {
    memmove(name, s, len);
    name[len] = 0;
  }
  while(*path == '/')
    path++;
  return path;
}
```

This function is a bit gnarly, but the comment explains it nicely. Here's what it does:

1. Skip any leading slashes
2. If we've reached the end of the string, return 0
3. Find the next slash (or end of string) -- that's the end of this path component
4. Copy the component into `name`. If it's longer than DIRSIZ, truncate it. If it's shorter, null-terminate it.
5. Skip any trailing slashes
6. Return a pointer to the remaining path

The examples in the comment are super helpful -- look at those to understand the behavior. Notice how it handles multiple consecutive slashes (they're treated as one), empty paths, and paths with or without trailing slashes.

**One tricky bit**: If the path component is >= DIRSIZ characters, we truncate it without null-terminating. This matches the directory entry format where exactly-DIRSIZ-length names don't have null terminators.

### Looking Up a Path: namex()

Now we can put it all together. The `namex()` function resolves a complete pathname to an inode:

```c
// Look up and return the inode for a path name.
// If parent != 0, return the inode for the parent and copy the final
// path element into name, which must have room for DIRSIZ bytes.
// Must be called inside a transaction since it calls iput().
static struct inode*
namex(char *path, int nameiparent, char *name)
{
  struct inode *ip, *next;

  if(*path == '/')
    ip = iget(ROOTDEV, ROOTINO);
  else
    ip = idup(myproc()->cwd);

  while((path = skipelem(path, name)) != 0){
    ilock(ip);
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    if(nameiparent && *path == '\0'){
      // Stop one level early.
      iunlock(ip);
      return ip;
    }
    if((next = dirlookup(ip, name, 0)) == 0){
      iunlockput(ip);
      return 0;
    }
    iunlockput(ip);
    ip = next;
  }
  if(nameiparent){
    iput(ip);
    return 0;
  }
  return ip;
}
```

This is the heart of the pathname layer. Let's trace through it carefully:

**Step 1: Determine starting point**
- If the path starts with `/`, it's an absolute path, so we start at the root directory (inode number ROOTINO on device ROOTDEV)
- Otherwise, it's a relative path, so we start at the current working directory (`myproc()->cwd`)
- We use `idup()` for the cwd to increment its reference count

**Step 2: Traverse the path**

Then we loop, extracting one path element at a time with `skipelem()` and following it:

1. Lock the current inode
2. Check that it's a directory. If not, fail -- you can't traverse through a regular file!
3. If `nameiparent` is set and this is the last path element (`*path == '\0'`), stop here and return the *parent* directory. This is used by system calls that need to operate on the parent directory, like when creating or deleting files.
4. Look up the current path element in the directory with `dirlookup()`
5. If the lookup fails (name doesn't exist), return 0
6. Unlock and release the current inode, and move to the next one

**Step 3: Return result**

When we run out of path elements, `skipelem()` returns 0 and the loop ends:
- If `nameiparent` was set but we never hit the "stop one level early" case, that means the path was empty, so return 0
- Otherwise, return the final inode

**Important design decisions**:

1. **Sleep-locks**: Notice that `namex()` might sleep (it calls `ilock()`, which uses a sleep-lock). That's why the comment says it must be called inside a transaction -- we don't want to hold any spin-locks when we sleep!

2. **Lock ordering**: We always lock the *current* directory before looking up the next one, then unlock it before locking the next. This prevents deadlocks that could occur if two processes tried to traverse paths in opposite directions.

3. **Reference counting**: We carefully manage inode references with `idup()`, `iput()`, `iunlockput()`. This ensures inodes don't get freed while we're using them, but also get freed when we're done.

### Two Variants: namei() and nameiparent()

The `namex()` function is used by two wrapper functions:

```c
struct inode*
namei(char *path)
{
  char name[DIRSIZ];
  return namex(path, 0, name);
}

struct inode*
nameiparent(char *path, char *name)
{
  return namex(path, 1, name);
}
```

**`namei()`** returns the inode for a path. It's what you'd use to open an existing file. For example, `namei("/usr/bin/ls")` returns the inode for the `ls` file.

**`nameiparent()`** returns the inode for the *parent directory* and copies the final path element into `name`. It's what you'd use when creating or deleting a file -- you need to modify the parent directory's contents. For example, `nameiparent("/usr/bin/ls", name)` returns the inode for the `/usr/bin` directory and sets `name` to `"ls"`.

These two functions provide a clean interface that hides the complexity of path traversal. System calls just call `namei()` or `nameiparent()` depending on what they need, and the pathname layer handles all the details.

And that's the pathname layer! It's elegant how it builds on top of the directory layer: `skipelem()` parses paths, `dirlookup()` searches directories, and `namex()` combines them to traverse the file system tree.

## File Descriptors (file.c)

Now let's look at the file descriptor layer, which provides the abstraction that user programs actually interact with. In Unix, a file descriptor (FD) is an integer that represents an open file. When you call `open()`, you get back a file descriptor, and you use that FD in subsequent calls to `read()`, `write()`, and `close()`.

File descriptors provide a layer of indirection between user programs and the actual file system. They let the kernel track which files each process has open, manage the current read/write position, and handle different types of "files" (regular files, directories, pipes, devices) uniformly.

### The File Structure

Each open file is represented by a `file` structure (from file.h):

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe;
  struct inode *ip;
  uint off;
};
```

Fields:
- `type`: What kind of file this is: a pipe, an inode (regular file or directory), or unused
- `ref`: Reference count. Multiple file descriptors can point to the same `file` structure (after `dup()` or `fork()`)
- `readable` and `writable`: Whether this file is open for reading and/or writing
- `pipe`: If this is a pipe, a pointer to the pipe structure
- `ip`: If this is an inode-based file, a pointer to the inode
- `off`: The current read/write offset in the file

The kernel maintains a global table of all open files:

```c
struct {
  struct spinlock lock;
  struct file file[NFILE];
} ftable;
```

NFILE is 100, so xv6 can have at most 100 open files system-wide. That's pretty limited, but remember, simplicity!

Each process also has its own per-process table of open file descriptors in its proc structure:

```c
struct proc {
  ...
  struct file *ofile[NOFILE];  // Open files
  ...
};
```

NOFILE is 16, so each process can have at most 16 open files. The `ofile` array maps file descriptor numbers (0, 1, 2, ...) to pointers to `file` structures in the global ftable.

**This two-level structure is important**: Multiple processes can have file descriptors pointing to the same `file` structure (after `fork()`), and multiple file descriptors (in the same or different processes) can point to the same `file` structure (after `dup()`). This enables powerful features like shared file offsets and I/O redirection.

### File Table Operations

We briefly saw the file table operations in an earlier post, but let's review them since they're crucial for understanding how file descriptors work:

```c
// Allocate a file structure.
struct file*
filealloc(void)
{
  struct file *f;

  acquire(&ftable.lock);
  for(f = ftable.file; f < ftable.file + NFILE; f++){
    if(f->ref == 0){
      f->ref = 1;
      release(&ftable.lock);
      return f;
    }
  }
  release(&ftable.lock);
  return 0;
}
```

`filealloc()` finds an unused entry in the global file table (one with `ref == 0`), sets its reference count to 1, and returns it. If all entries are in use, it returns 0.

```c
// Increment ref count for file f.
struct file*
filedup(struct file *f)
{
  acquire(&ftable.lock);
  if(f->ref < 1)
    panic("filedup");
  f->ref++;
  release(&ftable.lock);
  return f;
}
```

`filedup()` increments a file's reference count. This is used by `dup()` and `fork()` to share file structures.

```c
// Close file f.  (Decrement ref count, close when reaches 0.)
void
fileclose(struct file *f)
{
  struct file ff;

  acquire(&ftable.lock);
  if(f->ref < 1)
    panic("fileclose");
  if(--f->ref > 0){
    release(&ftable.lock);
    return;
  }
  ff = *f;
  f->ref = 0;
  f->type = FD_NONE;
  release(&ftable.lock);

  if(ff.type == FD_PIPE)
    pipeclose(ff.pipe, ff.writable);
  else if(ff.type == FD_INODE){
    begin_op();
    iput(ff.ip);
    end_op();
  }
}
```

`fileclose()` decrements the reference count. When it reaches 0, it actually closes the file:
- For pipes, call `pipeclose()` to clean up the pipe
- For inodes, call `iput()` to release the inode reference (which might trigger file deletion if this was the last reference to an unlinked file)

Notice the clever pattern: we copy the file structure to a local variable, mark the entry as free, then release the lock before doing the actual cleanup. This minimizes the time we hold the spin-lock.

### Reading and Writing

The `fileread()` and `filewrite()` functions handle reading and writing through file descriptors:

```c
// Read from file f.
int
fileread(struct file *f, char *addr, int n)
{
  int r;

  if(f->readable == 0)
    return -1;
  if(f->type == FD_PIPE)
    return piperead(f->pipe, addr, n);
  if(f->type == FD_INODE){
    ilock(f->ip);
    if((r = readi(f->ip, addr, f->off, n)) > 0)
      f->off += r;
    iunlock(f->ip);
    return r;
  }
  panic("fileread");
}
```

First, check that the file is open for reading. Then dispatch based on type:
- For pipes, call `piperead()`
- For inodes, lock the inode, call `readi()` to read the data, update the file offset, and unlock

Notice how the file offset is managed automatically -- user programs don't have to track it themselves.

```c
// Write to file f.
int
filewrite(struct file *f, char *addr, int n)
{
  int r;

  if(f->writable == 0)
    return -1;
  if(f->type == FD_PIPE)
    return pipewrite(f->pipe, addr, n);
  if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * 512;
    int i = 0;
    while(i < n){
      int n1 = n - i;
      if(n1 > max)
        n1 = max;

      begin_op();
      ilock(f->ip);
      if ((r = writei(f->ip, addr + i, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op();

      if(r < 0)
        break;
      if(r != n1)
        panic("short filewrite");
      i += r;
    }
    return i == n ? n : -1;
  }
  panic("filewrite");
}
```

Writing is more complex because of transactions. The logging layer has a maximum transaction size (MAXOPBLOCKS), so we can't write too much data in one transaction. The calculation determines how many bytes we can safely write per transaction, then we loop, writing chunks and committing each one.

For each chunk:
1. Start a transaction with `begin_op()`
2. Lock the inode
3. Call `writei()` to write the data
4. Update the file offset
5. Unlock the inode
6. Commit the transaction with `end_op()`

This ensures each chunk is written atomically, even if it's part of a larger write.

### Getting File Metadata

```c
// Get metadata about file f.
int
filestat(struct file *f, struct stat *st)
{
  if(f->type == FD_INODE){
    ilock(f->ip);
    stati(f->ip, st);
    iunlock(f->ip);
    return 0;
  }
  return -1;
}
```

`filestat()` fills in a `stat` structure with metadata about a file. It only works for inode-based files (not pipes). It just calls `stati()` from the inode layer to fill in the structure.

## How It All Fits Together

Let's trace through a complete file system operation to see how all these layers work together. We'll use a simple example: reading from a file.

Suppose a user program calls:
```c
char buf[100];
read(fd, buf, 100);
```

Here's the journey (we covered the system call part in the previous post, but let's see the whole picture):

### Layer 1: System Call (sysfile.c)

1. Trap into kernel, system call handler calls `sys_read()`
2. `sys_read()` extracts the FD, buffer pointer, and length
3. Looks up the FD in `myproc()->ofile[fd]` to get the file structure
4. Calls `fileread(f, buf, 100)`

### Layer 2: File Descriptor (file.c)

5. `fileread()` checks that the file is open for reading
6. Checks the file type -- let's say it's FD_INODE
7. Locks the inode with `ilock()`
8. Calls `readi(f->ip, buf, f->off, 100)` to read from offset `f->off`
9. Updates `f->off` by the number of bytes read
10. Unlocks the inode

### Layer 3: Inode (fs.c)

11. `readi()` loops through the file, reading one block at a time
12. For each block, it calls `bmap()` to translate the file block number to a disk block number
13. Calls `bread()` to read the block into the buffer cache
14. Copies data from the buffer to the user's buffer
15. Releases the buffer with `brelse()`

### Layer 4: Buffer Cache (bio.c)

16. `bread()` checks if the block is already in the cache with `bget()`
17. If not, `bget()` finds a free buffer (or evicts an old one)
18. Reads the block from disk with `iderw()`
19. Returns the buffer

### Layer 5: Disk Driver (ide.c)

20. `iderw()` adds the request to the disk queue
21. Starts the disk if it's idle
22. Waits for the disk interrupt
23. When the interrupt arrives, the disk driver marks the request as done
24. `iderw()` returns

Then everything unwinds back up the stack, and the data ends up in the user's buffer!

## Common Patterns

Looking at all this code together, several design patterns emerge:

### 1. Layered Abstraction

Each layer provides a clean interface and hides implementation details:
- **Pathname layer**: Converts paths to inodes
- **Directory layer**: Maps names to inode numbers within a directory
- **File descriptor layer**: Provides a convenient integer-based interface for user programs
- **Inode layer**: Manages file metadata and data blocks
- **Buffer cache**: Caches disk blocks in memory
- **Disk driver**: Actually reads and writes the hardware

No layer knows about the layers above it. This makes the code modular and easy to understand.

### 2. Reference Counting

Reference counts are everywhere:
- File structures track how many FDs point to them
- Inodes have both in-memory references (for the inode cache) and on-disk references (nlink, for hard links)
- Buffers track how many users are accessing them

The pattern is always: increment when you get a reference, decrement when you're done, only free when the count reaches zero.

### 3. Lock Ordering

To avoid deadlocks, locks must be acquired in a consistent order. The general hierarchy is:
1. ftable.lock (file table lock)
2. inode locks
3. bcache.lock (buffer cache lock)

Within each category, we're careful about ordering. For example, `namex()` always locks parent before child when traversing directories.

### 4. Sleep-Locks vs Spin-Locks

Sleep-locks (used for inodes and buffers) allow the holder to sleep. Spin-locks (used for ftable and bcache) require the holder to keep running. We never acquire a spin-lock while holding a sleep-lock, because sleeping while holding a spin-lock would freeze other CPUs!

### 5. Error Propagation

Errors propagate up the stack via return values (0 or NULL for failure, positive values for success). Each layer checks return values and cleans up appropriately before passing the error up.

## Wrapping Up

We've covered the entire upper half of the file system:

1. **Directories**: Special files containing (name, inode number) pairs
   - `dirlookup()`: Search for a name in a directory
   - `dirlink()`: Add a name to a directory
   - `namecmp()`: Compare file names

2. **Pathnames**: Traversing the directory tree
   - `skipelem()`: Parse one component from a path
   - `namex()`: Resolve a complete pathname
   - `namei()` and `nameiparent()`: Convenient wrappers

3. **File descriptors**: The abstraction user programs interact with
   - `file` structure: Tracks open files
   - `filealloc()`, `filedup()`, `fileclose()`: Managing file table entries
   - `fileread()`, `filewrite()`: Dispatching I/O operations
   - `filestat()`: Getting file metadata

Combined with the lower layers from the previous post (buffer cache, logging, block allocation, and inodes) and the system calls from the post before this, we now understand the complete xv6 file system!

The xv6 file system is a beautiful example of layered design. Each layer is relatively simple on its own, but together they create a robust, crash-safe system that can store and retrieve data reliably even in the face of power failures and crashes.

In the next post, we'll look at how user programs actually use this file system by examining the system calls that operate on files. We'll see how `open()`, `read()`, `write()`, `link()`, `unlink()`, and all the other file system calls tie everything together. See you there!