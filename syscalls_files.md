# System Calls: Files

Welcome to the final set of system calls! We've already covered process-related system calls like `fork()`, `exit()`, and `wait()`. Now it's time to look at the system calls that let user programs interact with the file system.

If you've been following along, you've seen how xv6's file system works from the bottom up: the disk driver reads and writes blocks, the buffer cache speeds things up, the logging layer ensures crash safety, block allocation manages free space, inodes store file metadata, and directories map names to inodes. We've also seen how pathnames get resolved and how file descriptors provide an abstraction for user programs.

All of that infrastructure exists to support the system calls we're about to look at. These are the functions that user programs actually call when they want to open files, read and write data, create directories, and navigate the file system. They live in sysfile.c, and they tie together everything we've learned about the file system.

Let's dive in!

## Helper Functions

Before we get to the actual system calls, let's look at a couple helper functions that many of them use.

### Getting a File Descriptor: argfd()

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

This helper extracts a file descriptor argument from a system call and looks it up in the process's `ofile` table. Remember, each process has an array of file pointers (`ofile[NOFILE]`) that maps FD numbers to `struct file` pointers in the global file table.

The function:
1. Uses `argint()` to get the FD number from the system call arguments
2. Validates that it's in range (0 to NOFILE-1) and points to an open file
3. Optionally returns both the FD number and the file pointer through the output parameters

If anything goes wrong (invalid argument, FD out of range, or FD not open), it returns -1.

This is used by almost every file system call since they nearly all take a file descriptor as their first argument.

### Allocating a File Descriptor: fdalloc()

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

This helper allocates a free file descriptor slot in the current process's `ofile` table. It just searches for the first unused entry (where `ofile[fd] == 0`) and stores the file pointer there, then returns the FD number.

If all NOFILE slots are in use, it returns -1 to indicate failure.

An important note: FDs 0, 1, and 2 have no special meaning in the kernel -- they're just regular file descriptors. By convention, user programs treat 0 as stdin, 1 as stdout, and 2 as stderr, but that's purely a user-space convention. The shell sets these up when starting a program, not the kernel.

## Simple File Descriptor Operations

Let's start with the simpler system calls that just manipulate file descriptors.

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

The `dup()` system call creates a copy of a file descriptor. Both the old and new FDs point to the same `struct file`, so they share the same file offset and open mode.

Here's what happens:
1. Get the file structure with `argfd()`
2. Allocate a new FD with `fdalloc()`
3. Increment the file's reference count with `filedup()`
4. Return the new FD number

This is crucial for I/O redirection in the shell. For example, to redirect stdout to a file:
1. Open the file, getting FD 3 (say)
2. Close FD 1 (stdout)
3. Call `dup(3)`, which allocates FD 1 pointing to the same file
4. Close FD 3

Now stdout (FD 1) points to the file!

### Reading from a File: sys_read()

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
```

Super straightforward! The `read()` system call takes three arguments:
- File descriptor
- Buffer pointer
- Number of bytes to read

We extract all three, validate them, and pass them to `fileread()` (from file.c), which we covered in an earlier post. That function handles the actual reading, whether it's from a regular file, pipe, or device.

The return value is the number of bytes actually read, or -1 on error.

### Writing to a File: sys_write()

```c
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

The `write()` system call is the mirror of `read()`. Same arguments, same pattern, but it calls `filewrite()` instead. Like `read()`, it returns the number of bytes written or -1 on error.

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

The `close()` system call closes a file descriptor. Simple:
1. Get the FD and file structure
2. Clear the `ofile` entry (mark the FD as unused)
3. Call `fileclose()` to decrement the reference count and potentially free the file structure

Remember, the file might still be open by other file descriptors (after `dup()`) or other processes (after `fork()`), so `fileclose()` only frees the file structure when its reference count reaches zero.

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

The `fstat()` system call fills in a `stat` structure with information about a file. The `stat` structure (defined in stat.h) contains:
- File type (regular file, directory, or device)
- Device number
- Inode number
- Number of hard links
- File size in bytes

This is useful when you need to check file metadata without reading the entire file. For example, `ls` uses `fstat()` to get file sizes and types.

Again, just a simple wrapper around `filestat()`.

## Link Management

Hard links are one of the coolest features of Unix file systems. Multiple directory entries can point to the same inode, so the same file can have multiple names. Let's see how the system calls for managing links work.

### Creating a Hard Link: sys_link()

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

The `link()` system call creates a new name for an existing file. This is more complex than the previous calls, so let's trace through it carefully:

1. **Extract arguments**: Get the old and new path names
2. **Start transaction**: Call `begin_op()` since we're modifying the disk
3. **Look up old file**: Use `namei()` to get the inode for the existing file
4. **Check it's not a directory**: xv6 doesn't allow hard links to directories because they could create cycles in the directory tree, making cleanup extremely complicated. (Imagine trying to delete a directory that's its own parent!)
5. **Increment link count**: Bump `ip->nlink` and write it to disk with `iupdate()`
6. **Get parent directory**: Use `nameiparent()` to get the parent directory of the new path
7. **Validate same device**: Both files must be on the same device -- you can't hard link across file systems
8. **Add directory entry**: Use `dirlink()` to add the new name to the directory
9. **Clean up and commit**: Unlock everything and call `end_op()` to commit the transaction

The error path (the `bad:` label) is important: if anything goes wrong after we've incremented `nlink`, we need to undo that increment before returning.

**How it works**: Hard links work by having multiple directory entries point to the same inode. The inode's `nlink` field tracks how many links exist. When it reaches 0 (all names deleted), the inode and its data blocks get freed.

Example usage: `link("foo.txt", "bar.txt")` creates a new name `bar.txt` for the file `foo.txt`. They're literally the same file -- same inode, same data, same everything. If you write to one, the changes appear in the other immediately.

### Unlinking (Deleting) a File: sys_unlink()

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

The `unlink()` system call removes a name from the directory tree. This is the opposite of `link()`, and it's what implements file deletion. It's more complex because we have to handle directories specially.

Let's trace through it:

1. **Get parent directory**: Use `nameiparent()` to find the directory containing the file to delete
2. **Start transaction**: Call `begin_op()`
3. **Validate name**: Check that we're not trying to unlink "." or ".." (that would be catastrophic!)
4. **Look up file**: Use `dirlookup()` to find the file and get its offset in the directory
5. **Sanity check**: Verify that `nlink >= 1` (if not, something's seriously broken)
6. **Check if directory**: If this is a directory, make sure it's empty with `isdirempty()`. You can't delete non-empty directories!
7. **Remove directory entry**: Zero out the directory entry by writing a zeroed `dirent` structure
8. **Update parent link count**: If we just deleted a directory, decrement the parent's link count (because the deleted directory had a ".." entry pointing to the parent)
9. **Decrement link count**: Reduce the inode's `nlink` and update it on disk
10. **Clean up and commit**: Unlock everything and end the transaction

**Critical insight**: We decrement `nlink` but we DON'T delete the inode or free its data blocks here. That happens later in `iput()` (from the inode layer). Remember, `iput()` checks if `nlink == 0` and `ref == 1`, and if so, it calls `itrunc()` to free the data blocks and marks the inode as free.

This lazy deletion is a classic Unix behavior: if a file is still open (reference count > 1) when it's unlinked, the file continues to exist until all file descriptors are closed. This is super useful! For example, you can:
1. Open a temporary file
2. Unlink it immediately (so it doesn't clutter the directory)
3. Keep using it as a scratch space
4. When you close it, it automatically gets deleted

The helper function for checking if a directory is empty:

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

We start at offset `2*sizeof(de)` to skip the "." and ".." entries (which are always present in directories), then check if any entry has a non-zero inode number.

## Creating Files and Directories

Several system calls need to create new inodes: `open()` with O_CREATE, `mkdir()`, and `mknod()` (for device files). They all use a common helper function.

### Helper: create()

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

This is the workhorse function for creating new files, directories, and device nodes. Here's what it does:

1. **Get parent directory**: Use `nameiparent()` to find where to create the new entry
2. **Check if exists**: Use `dirlookup()` to see if the name already exists. If it does:
   - For regular files, return the existing inode (this lets `open()` with O_CREATE work when the file exists)
   - For directories/devices, that's an error
3. **Allocate inode**: Use `ialloc()` to get a new inode
4. **Initialize inode**: Set type, major/minor (for devices), and `nlink = 1`, then write to disk
5. **Special handling for directories**: 
   - Increment parent's link count (for the ".." entry we're about to add)
   - Create "." entry pointing to itself
   - Create ".." entry pointing to parent
   - Note: We DON'T increment `nlink` for the "." entry! Otherwise we'd have circular reference counts and could never free directories.
6. **Add to parent**: Use `dirlink()` to add the new name to the parent directory
7. **Return locked inode**: The caller gets a locked inode ready to use

### Opening a File: sys_open()

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

The `open()` system call is one of the most important file system calls. It takes a path and a mode, and returns a file descriptor you can use for reading and writing.

The mode contains flags like:
- `O_RDONLY` (0): Open for reading only
- `O_WRONLY` (1): Open for writing only  
- `O_RDWR` (2): Open for reading and writing
- `O_CREATE` (0x200): Create the file if it doesn't exist

Here's what happens:

1. **Extract arguments**: Get the path and mode
2. **Start transaction**: We might modify the disk
3. **Create or open**: 
   - If `O_CREATE` is set, use `create()` to create the file or return the existing inode
   - Otherwise, use `namei()` to look up the file
   - Special check: Directories can only be opened read-only (you can't write directly to a directory's data!)
4. **Allocate resources**: Get a file structure with `filealloc()` and a file descriptor with `fdalloc()`
5. **Initialize file structure**: Set type to `FD_INODE`, point to the inode, set offset to 0, and set readable/writable flags based on mode
6. **Clean up**: Unlock the inode and end the transaction (important: we don't keep holding the inode lock after returning to user space!)
7. **Return FD**: Give the user their shiny new file descriptor

**Why unlock before returning?** Once we return to user space, the program might not call `close()` for a long time. We don't want to hold the inode lock or keep the transaction open for that long. The file structure holds a reference to the inode (via the reference count), so the inode won't disappear even though we unlocked it.

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

The `mkdir()` system call creates a new directory. It's beautifully simple: just call `create()` with type `T_DIR`. The `create()` function handles all the details of initializing the directory with "." and ".." entries.

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

The `mknod()` system call creates a device file. Device files don't contain any data on disk -- instead, reads and writes are handled by the device driver identified by the major/minor numbers.

For example, the console device (major=1, minor=1) routes reads and writes to the console driver. When you read from `/dev/console`, the file system sees it's a device file and calls `devsw[major].read()` instead of reading from disk.

Again, we just use `create()` with type `T_DEV` and pass along the major/minor numbers. Easy!

## Navigation System Calls

These system calls help programs navigate the directory hierarchy.

### Changing Directory: sys_chdir()

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

1. Look up the path with `namei()`
2. Check that it's a directory
3. Release the old cwd (dropping its reference count with `iput()`)
4. Set the process's `cwd` field to the new inode

Remember that relative paths (paths not starting with '/') are resolved starting from `myproc()->cwd`, so changing this field changes where relative paths are resolved from.

**Note**: xv6 doesn't have a `getcwd()` system call! The kernel only tracks the inode of the current directory, not its full pathname. If you wanted to print the current directory's path, you'd have to walk up the tree following ".." entries and reconstructing the path -- xv6 leaves that to user programs. (Some Unix systems added `getcwd()` later, but classic Unix worked this way.)

## Process Management: sys_exec()

The `exec()` system call isn't technically a "file system" call, but it lives in sysfile.c because it needs to read executable files from the file system.

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

The `exec()` system call replaces the current process's memory with a new program loaded from an executable file. The arguments are:
- `path`: Path to the executable
- `argv`: Array of argument strings (like `char *argv[]` in main)

Here's what happens:

1. **Extract path**: Get the executable's pathname
2. **Extract argv array**: The user passes a pointer to an array of string pointers (`char **`). We need to copy all those pointers and the strings they point to into kernel space:
   - Use `fetchint()` to read each pointer from the array
   - Use `fetchstr()` to copy each string into kernel space
   - Stop when we hit a NULL pointer
3. **Call exec()**: Pass everything to `exec()` (from exec.c, which we covered in the processes post)

The `exec()` function loads the ELF binary, sets up the user stack with the arguments, and starts executing the new program. If it succeeds, it never returns -- the process is now running the new program! If it fails, it returns -1 and the current process continues.

This is how the shell runs programs: it calls `fork()` to create a child process, then the child calls `exec()` to transform into the program you want to run.

## Inter-Process Communication: sys_pipe()

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

The `pipe()` system call creates a one-way communication channel. It returns two file descriptors:
- `fd[0]`: The read end
- `fd[1]`: The write end

Data written to the write end can be read from the read end.

Here's what happens:

1. **Extract argument**: Get a pointer to the output array (where we'll store the two FDs)
2. **Allocate pipe**: Call `pipealloc()` (from pipe.c) which creates a pipe and returns two file structures
3. **Allocate FDs**: Get file descriptors for both ends
4. **Return FDs**: Store them in the output array

Pipes are used for inter-process communication, typically between parent and child after `fork()`. The classic example is shell pipelines like `ls | grep foo`:
1. Shell creates a pipe
2. Forks two processes
3. First process closes stdout (FD 1), dups the write end to FD 1, then execs `ls`
4. Second process closes stdin (FD 0), dups the read end to FD 0, then execs `grep`
5. Now `ls`'s output goes into the pipe, and `grep` reads from the pipe!

We'll cover pipes in more detail in the next post when we look at how the shell uses them.

## Putting It All Together: A Complete Example

Let's trace through a complete example to see how everything works together. Suppose a user program runs:

```c
int fd = open("/tmp/foo.txt", O_CREATE | O_WRONLY);
write(fd, "hello\n", 6);
close(fd);
```

Here's the journey:

### Step 1: open("/tmp/foo.txt", O_CREATE | O_WRONLY)

1. Trap into kernel, system call handler calls `sys_open()`
2. `sys_open()` starts a transaction with `begin_op()`
3. Since O_CREATE is set, it calls `create("/tmp/foo.txt", T_FILE, 0, 0)`
4. `create()` calls `nameiparent("/tmp/foo.txt", name)`, which:
   - Starts at root (inode 1, since path starts with '/')
   - Looks up "tmp" with `dirlookup()`, gets its inode
   - Returns the inode for `/tmp` and sets `name` to `"foo.txt"`
5. `create()` locks `/tmp` and checks if `foo.txt` exists with `dirlookup()` -- let's say it doesn't
6. `create()` allocates a new inode with `ialloc()` (from the inode layer)
7. Sets the inode's type to T_FILE, nlink to 1, and calls `iupdate()` to write it to the log
8. Calls `dirlink()` to add an entry to `/tmp` mapping "foo.txt" to the new inode number
9. Returns the locked inode for `foo.txt`
10. Back in `sys_open()`, we allocate a file structure with `filealloc()` and a file descriptor with `fdalloc()` -- say we get FD 3
11. Unlock the inode, end the transaction with `end_op()` (which commits all changes to disk)
12. Fill in the file structure: type=FD_INODE, ip=inode pointer, off=0, readable=0, writable=1
13. Return FD 3 to user space

### Step 2: write(3, "hello\n", 6)

1. Trap into kernel, system call handler calls `sys_write()`
2. `sys_write()` extracts FD 3, validates it, gets the file structure
3. Calls `filewrite(f, "hello\n", 6)` (from file.c)
4. `filewrite()` starts a transaction with `begin_op()`
5. Calls `writei(f->ip, "hello\n", f->off, 6)` (from the inode layer)
6. `writei()` calls `bmap()` to get or allocate disk blocks for the write
7. Since this is a new file, `bmap()` calls `balloc()` to allocate a new data block
8. `balloc()` searches the bitmap, finds a free block, marks it used with `log_write()`
9. `writei()` calls `bread()` to read the block into the buffer cache
10. Copies "hello\n" into the buffer
11. Calls `log_write()` to log the change
12. Updates the inode's size to 6 bytes and calls `iupdate()` to log that
13. Updates `f->off` to 6 (advances the file offset)
14. `filewrite()` ends the transaction with `end_op()`
15. `end_op()` calls `commit()` which writes the log header (commit point!), copies logged blocks to their final locations, and clears the log
16. Returns 6 (bytes written) to user space

### Step 3: close(3)

1. Trap into kernel, system call handler calls `sys_close()`
2. Validates FD 3, gets the file structure
3. Clears `myproc()->ofile[3]` (marks FD as unused)
4. Calls `fileclose(f)` to decrement the file's reference count
5. Since ref reaches 0 (no more FDs point to this file), `fileclose()` calls `iput()`
6. `iput()` decrements the inode's in-memory reference count
7. Since nlink is still 1 (file not deleted), the inode stays allocated
8. Returns 0 to user space

And we're done! The file exists on disk with the contents "hello\n".

## Common Patterns and Design Principles

Looking at all these system calls together, several patterns emerge:

### 1. Transactions Everywhere

Almost every system call that modifies the file system is wrapped in `begin_op()` / `end_op()`. This ensures crash safety -- either the entire operation completes or none of it does. You never end up with half-written data or inconsistent metadata.

The logging layer makes this possible by recording all changes in a log before applying them. If we crash mid-operation, the log gets cleared on reboot and the changes are lost. But once `end_op()` commits the log, all changes are guaranteed to make it to disk.

### 2. Careful Lock Ordering

The code carefully locks inodes in a consistent order to avoid deadlocks. For example, in `sys_link()`, we lock the source inode before locking the parent directory of the new link. This ordering is maintained throughout the code.

The `namex()` function (which implements path traversal) is particularly careful: it locks the current directory before looking up the next one, then unlocks it before locking the next. This prevents deadlocks when multiple processes traverse paths in opposite directions.

### 3. Reference Counting

Both file structures and inodes use reference counting to track when they can be freed:
- File structures have a `ref` count tracking how many FDs point to them
- Inodes have both an in-memory `ref` count (in the inode cache) and an on-disk `nlink` count (tracking directory entries)

The pattern is always: increment when you get a reference, decrement when you're done, only free when the count reaches zero.

### 4. Two-Level Naming

File descriptors provide an extra layer of indirection between user programs and kernel data structures:
- User programs use small integers (FDs)
- FDs map to file structures (via `ofile[]`)
- File structures point to inodes

This indirection enables powerful features:
- Multiple FDs can share the same file structure (after `dup()`)
- Multiple processes can share the same FD space (after `fork()`)
- The same inode can have multiple names (hard links)

### 5. Lazy Deletion

When you `unlink()` a file, the inode and data blocks aren't immediately freed. They're only freed when:
- `nlink` reaches 0 (all names deleted), AND
- `ref` reaches 1 (no more in-memory references)

This happens in `iput()`, not in `sys_unlink()`.

This lazy approach allows powerful patterns like:
```c
int fd = open("tempfile", O_CREATE);
unlink("tempfile");  // Remove name, but file still exists
// ... use fd ...
close(fd);  // Now the file really gets deleted
```

### 6. Validation Then Action

System calls follow a consistent pattern:
1. Extract and validate all arguments
2. Check preconditions (file exists, permissions OK, etc.)
3. Acquire resources (locks, file structures, FDs)
4. Perform the operation
5. Release resources
6. Return result

If any step fails, we clean up what we've done and return an error. This keeps error handling straightforward.

### 7. Separation of Concerns

Each layer has a clear responsibility:
- **System calls** (sysfile.c): Argument extraction, high-level logic, transaction management
- **File layer** (file.c): File descriptor management, dispatching reads/writes
- **Pathname layer** (fs.c): Path traversal, directory operations
- **Inode layer** (fs.c): Inode management, reading/writing file data
- **Block layer** (bio.c, log.c): Buffer cache, logging, block allocation

No layer knows about layers above it. This makes the code easier to understand and modify.

## Error Handling

One thing you might notice: error handling in xv6 is pretty simple. When something goes wrong, we usually just return -1. There's no sophisticated error reporting (no errno, no error messages beyond console panics).

This is partly because xv6 is a teaching OS, but it also reflects the Unix philosophy: keep it simple. If an operation fails, clean up and return an error. The caller can decide what to do about it.

The exception is `panic()`, which is used for "should never happen" conditions that indicate a serious kernel bug (like failing to write a log entry). These immediately halt the system since the kernel's internal state might be corrupted.

## What We've Learned

We've covered all the file-related system calls in xv6:

**Basic I/O**:
- `dup()`: Duplicate a file descriptor
- `read()`: Read from a file
- `write()`: Write to a file
- `close()`: Close a file descriptor
- `fstat()`: Get file metadata

**Link Management**:
- `link()`: Create a hard link
- `unlink()`: Remove a name (delete a file)

**File Creation**:
- `open()`: Open or create a file
- `mkdir()`: Create a directory
- `mknod()`: Create a device file

**Navigation**:
- `chdir()`: Change current directory

**Other**:
- `exec()`: Execute a program
- `pipe()`: Create a communication channel

These system calls provide a clean, simple interface that hides all the complexity we've seen in the lower layers: buffer caching, logging, block allocation, inode management, directory traversal, pathname resolution, and so on.

User programs don't need to know about any of that -- they just call `open()`, `read()`, `write()`, and `close()`, and the kernel takes care of everything else. That's the power of abstraction!

Now that we've covered all the kernel infrastructure -- from boot to traps, from memory management to file systems -- it's time to see how it all comes to life. In the next post, we'll watch xv6 boot up its very first user process and see how user space actually gets started. See you there!