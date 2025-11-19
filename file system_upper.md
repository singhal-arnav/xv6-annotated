# File System: Upper Layers

Welcome back! In the last post, we covered the lower layers of the xv6 file system: the buffer cache, logging, block allocation, and inodes. Those layers gave us everything we need to read and write blocks of data, but they didn't give us anything that looks like the file systems we're used to -- no directories, no pathnames like `/usr/bin/ls`, no way to actually create or delete files.

That's what we're tackling in this post: the upper layers of the file system. We'll see how directories work (spoiler: they're just special files!), how path names get resolved into inodes, and how all the file system calls actually work: `open()`, `link()`, `unlink()`, `mkdir()`, and more.

This is where everything comes together. We'll be looking at code in fs.c (the directory and pathname functions) and sysfile.c (the system call implementations). Let's dive in!

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
- `inum`: The inode number of the file this entry refers to.
- `name`: The file name, up to DIRSIZ (14) characters.

A directory's data blocks contain a sequence of these dirent structures, one after another. If `inum` is 0, that entry is empty and available for use. Otherwise, `name` is the name of a file, and `inum` is its inode number.

This is a key insight: directory entries don't contain file metadata or data -- they just map names to inode numbers. All the actual file information lives in the inode itself. This is what makes hard links possible: multiple directory entries can point to the same inode!

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

This compares two file names, but only up to DIRSIZ characters. It's just a wrapper around `strncmp()` (which we've seen before in the string functions). We use this instead of `strcmp()` because directory entry names aren't always null-terminated -- if a name is exactly 14 characters long, there's no room for the null terminator!

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

First, we check that `dp` is actually a directory inode. If not, panic.

Then we loop through the directory's contents, one dirent at a time. For each entry:
- Read the dirent using `readi()`. We read directly into the `de` structure.
- If `de.inum` is 0, this entry is empty, so skip it.
- If the name matches what we're looking for, we found it! Save the offset in `*poff` (if the caller wants it), grab the inode number, and return the inode via `iget()`.

If we get through the whole directory without finding the name, return 0 (NULL).

Notice that we return an *unlocked* inode via `iget()`. This is important for avoiding deadlocks. Consider what happens if someone looks up "." (the current directory): we're already holding a lock on `dp`, so if we tried to lock the inode before returning, we'd try to lock `dp` again and deadlock! The caller is responsible for locking the returned inode if they need to use it.

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

First, we check if the name already exists in the directory. If it does, that's an error -- you can't have two files with the same name in the same directory. We return -1 to indicate failure.

Then we search for an empty directory entry (one with inum == 0). We loop through the directory until we find one, or until we reach the end of the directory.

Once we've found a slot (or reached the end), we create a new dirent: copy the name with `strncpy()` (which pads with nulls if the name is shorter than DIRSIZ), set the inode number, and write it to the directory with `writei()`.

A subtle point: if we reach the end of the directory (`off == dp->size`), `writei()` will automatically extend the directory by allocating a new block (remember, `writei()` calls `bmap()` which allocates blocks as needed). So directories can grow dynamically as files are added!

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

1. Skip any leading slashes.
2. If we've reached the end of the string, return 0.
3. Find the next slash (or end of string) -- that's the end of this path component.
4. Copy the component into `name`. If it's longer than DIRSIZ, truncate it. If it's shorter, null-terminate it.
5. Skip any trailing slashes.
6. Return a pointer to the remaining path.

The examples in the comment are super helpful -- look at those to understand the behavior. Notice how it handles multiple consecutive slashes (they're treated as one), empty paths, and paths with or without trailing slashes.

One tricky bit: if the path component is >= DIRSIZ characters, we truncate it without null-terminating. This matches the directory entry format where exactly-DIRSIZ-length names don't have null terminators.

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

First, we need to know where to start. If the path starts with `/`, it's an absolute path, so we start at the root directory (inode number ROOTINO on device ROOTDEV). Otherwise, it's a relative path, so we start at the current working directory (`myproc()->cwd`). We use `idup()` to increment the inode's reference count.

Then we loop, extracting one path element at a time with `skipelem()` and following it:

1. Lock the current inode.
2. Check that it's a directory. If not, fail -- you can't traverse through a regular file!
3. If `nameiparent` is set and this is the last path element (`*path == '\0'`), stop here and return the *parent* directory. This is used by functions that need to operate on the parent directory, like when creating or deleting files.
4. Look up the current path element in the directory with `dirlookup()`.
5. If the lookup fails (name doesn't exist), return 0.
6. Unlock and release the current inode, and move to the next one.

When we run out of path elements, `skipelem()` returns 0 and the loop ends. If `nameiparent` was set but we never hit the "stop one level early" case, that means the path was empty, so we return 0. Otherwise, we return the final inode.

A subtle point: notice that `namex()` might sleep (it calls `ilock()`, which uses a sleep-lock). That's why the comment says it must be called inside a transaction -- we don't want to hold any spin-locks when we sleep!

Also notice the careful lock ordering: we always lock the *current* directory before looking up the next one, then unlock it before locking the next. This prevents deadlocks that could occur if two processes tried to traverse paths in opposite directions.

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

`namei()` returns the inode for a path. It's what you'd use to open an existing file.

`nameiparent()` returns the inode for the *parent directory* and copies the final path element into `name`. It's what you'd use when creating or deleting a file -- you need to modify the parent directory's contents.

For example:
- `namei("/usr/bin/ls")` returns the inode for the `ls` file.
- `nameiparent("/usr/bin/ls", name)` returns the inode for the `/usr/bin` directory and sets `name` to `"ls"`.

And that's the pathname layer! It's elegant how it builds on top of the directory layer: `skipelem()` parses paths, `dirlookup()` searches directories, and `namex()` combines them to traverse the file system tree.

## File Descriptors (file.c)

Before we get to the actual system calls, we need to understand file descriptors. In Unix, a file descriptor (FD) is an integer that represents an open file. When you call `open()`, you get back a file descriptor, and you use that FD in subsequent calls to `read()`, `write()`, and `close()`.

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
- `type`: What kind of file this is: a pipe, an inode (regular file or directory), or unused.
- `ref`: Reference count. Multiple file descriptors can point to the same `file` structure (after `dup()` or `fork()`).
- `readable` and `writable`: Whether this file is open for reading and/or writing.
- `pipe`: If this is a pipe, a pointer to the pipe structure.
- `ip`: If this is an inode-based file, a pointer to the inode.
- `off`: The current read/write offset in the file.

The kernel maintains a global table of all open files:

```c
struct {
  struct spinlock lock;
  struct file file[NFILE];
} ftable;
```

NFILE is 100, so xv6 can have at most 100 open files system-wide. That's pretty limited, but again, simplicity!

Each process also has its own per-process table of open file descriptors in its proc structure:

```c
struct proc {
  ...
  struct file *ofile[NOFILE];  // Open files
  ...
};
```

NOFILE is 16, so each process can have at most 16 open files. The `ofile` array maps file descriptor numbers (0, 1, 2, ...) to pointers to `file` structures in the global ftable.

This two-level structure is important: multiple processes can have file descriptors pointing to the same `file` structure (after `fork()`), and multiple file descriptors (in the same or different processes) can point to the same `file` structure (after `dup()`).

We covered the file descriptor functions back in the system calls post, but let me give you a quick refresher since we'll be using them:

- `filealloc()`: Allocates a free entry in ftable.
- `filedup()`: Increments a file's reference count.
- `fileclose()`: Decrements the reference count and closes the file if it reaches 0.
- `fileread()` and `filewrite()`: Read and write files.
- `filestat()`: Gets file metadata.

## File System System Calls (sysfile.c)

Finally! Time to look at the actual system calls that user programs use to interact with the file system. All of these live in sysfile.c.

### Helper Functions

First, a couple helper functions:

```c
// Fetch the nth word-sized system call argument as a file descriptor
// and return both the descriptor and the corresponding struct file.
static int
argfd(int n, int *pfd, struct file **pf)
{
  int fd;
  struct file *f;

  if(argint(n, &fd) < 0)
    return -1;
  if(fd < 0 || fd >= NOFILE || (f=myproc()->ofile[fd]) == 0)
    return -1;
  if(pfd)
    *pfd = fd;
  if(pf)
    *pf = f;
  return 0;
}
```

This helper extracts a file descriptor argument from a system call and looks it up in the process's ofile table. It's used by many of the file system calls.

```c
// Allocate a file descriptor for the given file.
// Takes over file reference from caller on success.
static int
fdalloc(struct file *f)
{
  int fd;
  struct proc *curproc = myproc();

  for(fd = 0; fd < NOFILE; fd++){
    if(curproc->ofile[fd] == 0){
      curproc->ofile[fd] = f;
      return fd;
    }
  }
  return -1;
}
```

This allocates a free file descriptor in the current process's ofile table. It searches for the first unused slot and returns its index.

Notice that FDs 0, 1, and 2 have no special meaning in the kernel -- they're just regular file descriptors. By convention, user programs use 0 for stdin, 1 for stdout, and 2 for stderr, but that's just convention. The shell sets those up when it starts a program.

### Duplicating a File Descriptor: sys_dup()

```c
int
sys_dup(void)
{
  struct file *f;
  int fd;

  if(argfd(0, 0, &f) < 0)
    return -1;
  if((fd=fdalloc(f)) < 0)
    return -1;
  filedup(f);
  return fd;
}
```

The `dup()` system call creates a copy of a file descriptor. Both the old and new FDs point to the same `file` structure, so they share the same file offset.

We get the file structure with `argfd()`, allocate a new FD with `fdalloc()`, increment the file's reference count with `filedup()`, and return the new FD.

This is used for I/O redirection in the shell. For example, to redirect stdout to a file, the shell might:
1. Open the file, getting FD 3 (say).
2. Close FD 1 (stdout).
3. Call `dup(3)`, which allocates FD 1 pointing to the same file.
4. Close FD 3.

Now FD 1 (stdout) points to the file instead of the console!

### Reading and Writing: sys_read() and sys_write()

```c
int
sys_read(void)
{
  struct file *f;
  int n;
  char *p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argptr(1, &p, n) < 0)
    return -1;
  return fileread(f, p, n);
}

int
sys_write(void)
{
  struct file *f;
  int n;
  char *p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argptr(1, &p, n) < 0)
    return -1;
  return filewrite(f, p, n);
}
```

These are straightforward wrappers around `fileread()` and `filewrite()`, which we covered back in the system calls post. They extract the FD, buffer pointer, and length from the system call arguments and pass them to the appropriate file function.

### Closing a File: sys_close()

```c
int
sys_close(void)
{
  int fd;
  struct file *f;

  if(argfd(0, &fd, &f) < 0)
    return -1;
  myproc()->ofile[fd] = 0;
  fileclose(f);
  return 0;
}
```

Close is also simple: get the FD and file structure, clear the ofile entry, and call `fileclose()` to decrement the reference count and potentially free the file structure.

### Getting File Status: sys_fstat()

```c
int
sys_fstat(void)
{
  struct file *f;
  struct stat *st;

  if(argfd(0, 0, &f) < 0 || argptr(1, (void*)&st, sizeof(*st)) < 0)
    return -1;
  return filestat(f, st);
}
```

The `fstat()` system call fills in a `stat` structure with information about a file (its type, size, inode number, etc.). Again, just a wrapper around `filestat()`.

### Creating a Hard Link: sys_link()

Now for the interesting stuff! The `link()` system call creates a new name for an existing file (a hard link):

```c
// Create the path new as a link to the same inode as old.
int
sys_link(void)
{
  char name[DIRSIZ], *new, *old;
  struct inode *dp, *ip;

  if(argstr(0, &old) < 0 || argstr(1, &new) < 0)
    return -1;

  begin_op();
  if((ip = namei(old)) == 0){
    end_op();
    return -1;
  }

  ilock(ip);
  if(ip->type == T_DIR){
    iunlockput(ip);
    end_op();
    return -1;
  }

  ip->nlink++;
  iupdate(ip);
  iunlock(ip);

  if((dp = nameiparent(new, name)) == 0)
    goto bad;
  ilock(dp);
  if(dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0){
    iunlockput(dp);
    goto bad;
  }
  iunlockput(dp);
  iput(ip);

  end_op();

  return 0;

bad:
  ilock(ip);
  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);
  end_op();
  return -1;
}
```

Here's what happens:

1. Extract the old and new path names from the arguments.
2. Start a transaction with `begin_op()` -- we're going to modify the disk.
3. Look up the old path with `namei()` to get the inode.
4. Check that it's not a directory. xv6 doesn't allow hard links to directories because they could create cycles in the directory tree, making cleanup on deletion very complicated.
5. Increment the inode's link count (`nlink`) and write it to disk with `iupdate()`.
6. Get the parent directory of the new path with `nameiparent()`.
7. Check that both files are on the same device (you can't hard link across file systems!).
8. Add the new name to the directory with `dirlink()`.
9. Clean up and end the transaction.

If anything goes wrong after we increment nlink (the `goto bad` case), we need to undo that increment before returning.

The key insight: hard links work by having multiple directory entries point to the same inode. The inode's `nlink` field tracks how many links exist, and when it reaches 0, the inode and its data can be freed.

### Unlinking (Deleting) a File: sys_unlink()

The opposite of `link()` is `unlink()`, which removes a name from the directory tree:

```c
int
sys_unlink(void)
{
  struct inode *ip, *dp;
  struct dirent de;
  char name[DIRSIZ], *path;
  uint off;

  if(argstr(0, &path) < 0)
    return -1;

  begin_op();
  if((dp = nameiparent(path, name)) == 0){
    end_op();
    return -1;
  }

  ilock(dp);

  // Cannot unlink "." or "..".
  if(namecmp(name, ".") == 0 || namecmp(name, "..") == 0)
    goto bad;

  if((ip = dirlookup(dp, name, &off)) == 0)
    goto bad;
  ilock(ip);

  if(ip->nlink < 1)
    panic("unlink: nlink < 1");
  if(ip->type == T_DIR && !isdirempty(ip)){
    iunlockput(ip);
    goto bad;
  }

  memset(&de, 0, sizeof(de));
  if(writei(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
    panic("unlink: writei");
  if(ip->type == T_DIR){
    dp->nlink--;
    iupdate(dp);
  }
  iunlockput(dp);

  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);

  end_op();

  return 0;

bad:
  iunlockput(dp);
  end_op();
  return -1;
}
```

This is more complex. Let's trace through it:

1. Get the parent directory and the final name.
2. Start a transaction.
3. Lock the parent directory.
4. Check that we're not trying to unlink "." or ".." (that would be bad!).
5. Look up the name in the directory to get the inode and the offset of the directory entry.
6. Lock the inode.
7. Check that nlink is at least 1 (otherwise something is seriously wrong).
8. If this is a directory, check that it's empty (via `isdirempty()`). You can't delete a non-empty directory!
9. Zero out the directory entry by writing a zeroed dirent to the directory. This removes the name from the directory.
10. If this was a directory, decrement the parent's link count (because directories have a ".." entry pointing back to the parent).
11. Unlock the parent directory.
12. Decrement the inode's link count and update it.
13. Unlock the inode.
14. End the transaction.

The crucial bit: we decrement `nlink`, but we don't actually delete the inode or free its data blocks here. That happens later in `iput()` (from the lower layers post). Remember, `iput()` checks if nlink == 0 and ref == 1, and if so, it calls `itrunc()` to free the data blocks and marks the inode as free.

This lazy deletion is important: if a file is still open (ref > 1) when it's unlinked, the file continues to exist until all file descriptors are closed. This is a classic Unix behavior -- you can delete a file that's currently being read or written!

### Helper: Checking if a Directory is Empty

```c
// Is the directory dp empty except for "." and ".." ?
static int
isdirempty(struct inode *dp)
{
  int off;
  struct dirent de;

  for(off = 2*sizeof(de); off < dp->size; off += sizeof(de)){
    if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
      panic("isdirempty: readi");
    if(de.inum != 0)
      return 0;
  }
  return 1;
}
```

This checks if a directory is empty (except for "." and ".." which are always present). We start at offset `2*sizeof(de)` to skip the first two entries (which must be "." and ".."), then check if any entry has a non-zero inum.

### Creating Files: create()

Many system calls need to create new inodes: `open()` with O_CREATE, `mkdir()`, and `mknod()` (for device files). They all use a common helper function:

```c
static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  if((dp = nameiparent(path, name)) == 0)
    return 0;
  ilock(dp);

  if((ip = dirlookup(dp, name, 0)) != 0){
    iunlockput(dp);
    ilock(ip);
    if(type == T_FILE && ip->type == T_FILE)
      return ip;
    iunlockput(ip);
    return 0;
  }

  if((ip = ialloc(dp->dev, type)) == 0)
    panic("create: ialloc");

  ilock(ip);
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  if(type == T_DIR){  // Create . and .. entries.
    dp->nlink++;  // for ".."
    iupdate(dp);
    // No ip->nlink++ for ".": avoid cyclic ref count.
    if(dirlink(ip, ".", ip->inum) < 0 || dirlink(ip, "..", dp->inum) < 0)
      panic("create dots");
  }

  if(dirlink(dp, name, ip->inum) < 0)
    panic("create: dirlink");

  iunlockput(dp);

  return ip;
}
```

This is quite involved:

1. Get the parent directory and final name.
2. Lock the parent directory.
3. Check if the name already exists. If it does and we're creating a regular file and it's already a regular file, that's okay -- just return the existing inode (this is used by `open()` with O_CREATE when the file already exists). Otherwise, it's an error.
4. Allocate a new inode with `ialloc()`.
5. Lock it and fill in the metadata (type, major/minor for devices, nlink = 1).
6. If we're creating a directory, create the "." and ".." entries. Also increment the parent's link count because the new directory's ".." points to the parent.
7. Add the new name to the parent directory with `dirlink()`.
8. Unlock the parent directory.
9. Return the locked inode.

Notice the comment about not incrementing `nlink` for the "." entry. If we did, directories would have reference count cycles (the directory points to itself via "."), which would make it impossible to ever free directories. Instead, we just ignore the "." link when counting.

### Opening a File: sys_open()

Now for one of the most important system calls: `open()`:

```c
int
sys_open(void)
{
  char *path;
  int fd, omode;
  struct file *f;
  struct inode *ip;

  if(argstr(0, &path) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlock(ip);
  end_op();

  f->type = FD_INODE;
  f->ip = ip;
  f->off = 0;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);
  return fd;
}
```

Let's walk through this:

1. Extract the path and mode from the arguments. The mode contains flags like O_CREATE, O_RDONLY, O_WRONLY, O_RDWR, etc.
2. Start a transaction.
3. If O_CREATE is set, use `create()` to create the file if it doesn't exist, or return the existing inode if it does.
4. Otherwise, look up the path with `namei()`. If it's a directory and we're trying to open it for writing, that's an error -- directories can only be opened read-only.
5. Allocate a file structure with `filealloc()` and a file descriptor with `fdalloc()`. If either fails, clean up and return an error.
6. Unlock the inode and end the transaction.
7. Fill in the file structure: set type to FD_INODE, point to the inode, set offset to 0, and set the readable/writable flags based on the mode.
8. Return the file descriptor.

A subtle point: why do we unlock the inode and end the transaction before returning? Because once we return to user space, the user program might not call `close()` for a long time, and we don't want to hold the inode lock or keep the transaction open for that long. The file structure holds a reference to the inode (via the reference count we incremented in `ialloc()` or `iget()`), so the inode won't go away even though we unlocked it.

### Creating a Directory: sys_mkdir()

```c
int
sys_mkdir(void)
{
  char *path;
  struct inode *ip;

  begin_op();
  if(argstr(0, &path) < 0 || (ip = create(path, T_DIR, 0, 0)) == 0){
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();
  return 0;
}
```

Creating a directory is super simple: just call `create()` with type T_DIR. The `create()` function handles all the details of creating the "." and ".." entries.

### Creating a Device File: sys_mknod()

```c
int
sys_mknod(void)
{
  struct inode *ip;
  char *path;
  int major, minor;

  begin_op();
  if((argstr(0, &path)) < 0 ||
     argint(1, &major) < 0 ||
     argint(2, &minor) < 0 ||
     (ip = create(path, T_DEV, major, minor)) == 0){
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();
  return 0;
}
```

Device files are similar: call `create()` with type T_DEV and pass the major/minor numbers. Device files don't contain any data -- instead, reads and writes are handled by the device driver identified by the major/minor numbers.

We covered device files back in the system calls post when we looked at `fileread()` and `filewrite()`. When you read or write a device file, those functions check the type and call the appropriate device driver instead of reading from disk.

### Changing the Current Directory: sys_chdir()

```c
int
sys_chdir(void)
{
  char *path;
  struct inode *ip;
  struct proc *curproc = myproc();
  
  begin_op();
  if(argstr(0, &path) < 0 || (ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);
  if(ip->type != T_DIR){
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlock(ip);
  iput(curproc->cwd);
  end_op();
  curproc->cwd = ip;
  return 0;
}
```

The `chdir()` system call changes the current working directory. Simple:

1. Look up the path with `namei()`.
2. Check that it's a directory.
3. Release the old cwd (dropping its reference count).
4. Set the process's cwd to the new inode.

Remember from earlier that relative paths start from `myproc()->cwd`, so changing this field changes where relative paths are resolved from.

### Getting the Current Directory: sys_getcwd()

Wait, there's no `sys_getcwd()` in xv6! The kernel doesn't track the *name* of the current directory, only its inode. If you want to print the current directory's path, you'd have to walk up the tree following ".." entries and reconstructing the path -- xv6 leaves that to user programs.

Actually, looking at the xv6 source, there's no `getcwd()` system call at all. Some Unix systems added it later, but classic Unix worked this way: the kernel just tracks the inode, and if you want the path, you figure it out yourself by traversing the directory tree.

### Executing a Program: sys_exec()

Finally, let's look at `sys_exec()`, which loads a new program into the current process:

```c
int
sys_exec(void)
{
  char *path, *argv[MAXARG];
  int i;
  uint uargv, uarg;

  if(argstr(0, &path) < 0 || argint(1, (int*)&uargv) < 0){
    return -1;
  }
  memset(argv, 0, sizeof(argv));
  for(i=0;; i++){
    if(i >= NELEM(argv))
      return -1;
    if(fetchint(uargv+4*i, (int*)&uarg) < 0)
      return -1;
    if(uarg == 0){
      argv[i] = 0;
      break;
    }
    if(fetchstr(uarg, &argv[i]) < 0)
      return -1;
  }
  return exec(path, argv);
}
```

The `exec()` system call replaces the current process's memory with a new program. The arguments are:
- `path`: The path to the executable file.
- `argv`: An array of argument strings (like `char *argv[]` in a C main function).

First, we extract the path and the pointer to the argv array from user space.

Then we build a kernel copy of the argv array. The user passes a pointer to an array of pointers (a `char **`), and we need to copy all of those pointers and the strings they point to into kernel space. We use `fetchint()` to read each pointer and `fetchstr()` to copy each string. When we hit a NULL pointer, we're done.

Finally, we call the `exec()` function (from exec.c), which we covered way back in the processes post. That function loads the ELF binary, sets up the user stack with the arguments, and starts executing the new program.

If `exec()` succeeds, it never returns -- the process is now running the new program! If it fails, it returns -1 and the current process continues.

### Piping: sys_pipe()

The last file system system call is `pipe()`, which creates a pipe:

```c
int
sys_pipe(void)
{
  int *fd;
  struct file *rf, *wf;
  int fd0, fd1;

  if(argptr(0, (void*)&fd, 2*sizeof(fd[0])) < 0)
    return -1;
  if(pipealloc(&rf, &wf) < 0)
    return -1;
  fd0 = -1;
  if((fd0 = fdalloc(rf)) < 0 || (fd1 = fdalloc(wf)) < 0){
    if(fd0 >= 0)
      myproc()->ofile[fd0] = 0;
    fileclose(rf);
    fileclose(wf);
    return -1;
  }
  fd[0] = fd0;
  fd[1] = fd1;
  return 0;
}
```

A pipe is a one-way communication channel. The `pipe()` system call creates a pipe and returns two file descriptors: one for reading and one for writing. Data written to the write end can be read from the read end.

Here's what happens:

1. Extract the pointer to the output array (where we'll store the two FDs).
2. Allocate a pipe with `pipealloc()` (from pipe.c, which we'll cover in the concurrency posts). This returns two file structures: one for reading and one for writing.
3. Allocate file descriptors for both.
4. Store the FDs in the output array.
5. Return 0 for success.

Pipes are used for inter-process communication, typically between a parent and child process after `fork()`. The classic example is shell pipelines like `ls | grep foo`, where the shell creates a pipe, forks two processes, connects one's stdout to the write end and the other's stdin to the read end, and execs the two programs.

We'll cover pipes in more detail when we get to concurrency and the shell!

## Putting It All Together

Okay, let's trace through a complete example to see how everything fits together. Suppose a user program calls:

```c
int fd = open("/tmp/foo.txt", O_CREATE | O_WRONLY);
write(fd, "hello", 5);
close(fd);
```

Here's what happens:

1. **`open("/tmp/foo.txt", O_CREATE | O_WRONLY)`**:
   - The system call handler calls `sys_open()`.
   - `sys_open()` starts a transaction with `begin_op()`.
   - Since O_CREATE is set, it calls `create("/tmp/foo.txt", T_FILE, 0, 0)`.
   - `create()` calls `nameiparent("/tmp/foo.txt", name)`, which:
     - Starts at root (inode 1).
     - Looks up "tmp" with `dirlookup()`, finding its inode.
     - Returns the inode for `/tmp` and sets `name` to `"foo.txt"`.
   - `create()` locks the `/tmp` inode and calls `dirlookup()` to check if `foo.txt` exists. Let's say it doesn't.
   - `create()` calls `ialloc()` to allocate a new inode.
   - It sets the inode's type to T_FILE, nlink to 1, and calls `iupdate()` to write it to disk (actually, to the log).
   - It calls `dirlink()` to add an entry to `/tmp` mapping "foo.txt" to the new inode number.
   - It unlocks `/tmp` and returns the locked inode for `foo.txt`.
   - Back in `sys_open()`, we allocate a file structure with `filealloc()` and a file descriptor with `fdalloc()`.
   - We unlock the inode, end the transaction, fill in the file structure (type, ip, offset, readable/writable), and return the FD (say, 3).

2. **`write(fd, "hello", 5)`**:
   - The system call handler calls `sys_write()`.
   - `sys_write()` calls `filewrite()` (from file.c).
   - `filewrite()` checks the type (FD_INODE), starts a transaction, and calls `writei()` multiple times, writing chunks of up to BSIZE bytes at a time.
   - `writei()` (from the lower layers) calls `bmap()` to get or allocate disk blocks, then calls `bread()` to read each block, copies the data, and calls `log_write()` to log the change.
   - `bmap()` calls `balloc()` to allocate a new data block (since this is a new file).
   - `balloc()` searches the free block bitmap, finds a free block, marks it as used with `log_write()`, and zeroes the block with `bzero()`.
   - After writing all the data, `writei()` updates the inode's size and calls `iupdate()` to log that change.
   - `filewrite()` ends the transaction.
   - When the transaction ends, `commit()` writes all the logged blocks to the log, writes the log header (the commit point!), copies the blocks to their final locations, and clears the log.

3. **`close(fd)`**:
   - The system call handler calls `sys_close()`.
   - `sys_close()` clears `ofile[3]` and calls `fileclose()`.
   - `fileclose()` decrements the file's reference count. If it reaches 0 (no more FDs point to this file), it calls `iput()`.
   - `iput()` decrements the inode's reference count in the inode cache. If it reaches 0 and nlink is 0 (file was deleted), it would call `itrunc()` and free the inode, but in our case nlink is still 1, so the inode stays allocated.

And that's it! The file now exists on disk with the contents "hello".

The beautiful thing is how all the layers work together:
- The buffer cache ensures we don't read the same block twice.
- The logging layer ensures that even if we crash in the middle of writing, the file system stays consistent.
- The block allocator finds free space.
- The inode layer manages file metadata and data blocks.
- The directory layer maps names to inodes.
- The pathname layer traverses the directory tree.
- The file descriptor layer provides a convenient abstraction for user programs.

Each layer builds on the ones below it, creating a robust, crash-safe file system from relatively simple components.

## File System Initialization

One last thing: how does the file system get set up in the first place? During boot, `main()` (in main.c) calls `forkret()` for the first process, which calls `iinit()` and `initlog()`. Later, when the first user process starts (we'll cover this in detail in a future post), it checks if the root directory exists. If not, it creates it:

Actually, let me check the xv6 code for this... The root directory setup happens in `mkfs.c`, which is a separate program that creates the initial file system image. It creates inode 1 as the root directory with "." and ".." entries pointing to itself. Then when xv6 boots, the root directory already exists on disk.

The kernel itself doesn't create the root directory -- it's built into the file system image by the mkfs tool during compilation. Pretty clever! This avoids the chicken-and-egg problem of how to create the first directory when you need directories to create directories.

## Common Patterns and Idioms

Before we wrap up, let's highlight some common patterns you'll see in file system code:

**1. Transactions**: Almost every file system operation is wrapped in `begin_op()` / `end_op()`. This ensures crash safety.

**2. Lock ordering**: The code carefully locks inodes in a consistent order to avoid deadlocks. For example, when creating a link, we lock the source inode before locking the parent directory.

**3. Reference counting**: Both the buffer cache and inode cache use reference counts to track when entries can be recycled. The pattern is always: increment refcnt when you get a reference, decrement when you're done, and only recycle when refcnt reaches 0.

**4. Two-level naming**: File descriptors provide indirection between user programs and kernel data structures (file structures, which in turn point to inodes). This lets multiple FDs share the same file structure (after `dup()`) and multiple processes share the same FD space (after `fork()`).

**5. Lazy deletion**: When you unlink a file, the inode and data blocks aren't immediately freed. They're only freed when the last reference goes away (in `iput()`). This lets you delete files that are still open.

**6. Separation of concerns**: Each layer has a clear responsibility and doesn't know about the layers above or below (except through well-defined interfaces). The buffer cache doesn't know about inodes, inodes don't know about directories, directories don't know about pathnames, etc.

These patterns are common in operating system design, and you'll see them again and again as we explore the rest of xv6.

## Wrapping Up

Whew, that was a long one! We've covered the entire upper half of the file system:

1. **Directories**: Special files containing (name, inode number) pairs.
2. **Pathnames**: The `namex()`, `namei()`, and `nameiparent()` functions that traverse the directory tree.
3. **File descriptors**: The abstraction that user programs use to access files.
4. **System calls**: All the file system calls in sysfile.c: `open()`, `close()`, `read()`, `write()`, `link()`, `unlink()`, `mkdir()`, `mknod()`, `chdir()`, `exec()`, and `pipe()`.

Combined with the lower layers from the previous post (buffer cache, logging, block allocation, and inodes), we now understand the complete xv6 file system!

The xv6 file system is a beautiful example of layered design. Each layer is relatively simple on its own, but together they create a robust, crash-safe system that can store and retrieve data reliably even in the face of power failures and crashes.

In upcoming posts, we'll see how this file system gets used when we boot the first user process, and we'll dive into the shell to see how it uses these system calls to provide the command-line interface we know and love. See you there!