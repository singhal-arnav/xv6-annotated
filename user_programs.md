# User Space: Programs (optional)

We've made it! We've climbed all the way up from the boot loader, through the kernel, past the system calls, and now we're finally at the top of the stack: user space programs. These are the actual executables that users interact with when they boot up xv6 and type commands at the shell prompt.

If you've been following along from the beginning, you've seen how xv6 virtualizes the CPU and memory to give each process its own isolated environment, how it schedules processes to share the processor, how it handles traps and system calls to mediate access to hardware, and how it manages files on disk. All of that kernel complexity exists for one reason: to make it easy to write simple user programs that don't have to worry about any of those details.

The user programs in xv6 are refreshingly simple compared to all the kernel code we've been wading through. They're just regular C programs that use the system call interface we've already seen. But they're worth looking at for a few reasons: (1) they show you how to actually use the system calls, (2) they demonstrate some classic Unix design patterns, and (3) if you're gonna do any xv6 projects, you'll probably be writing programs just like these.

## How User Programs Get Built

Before we dive into the code, let's talk about how these programs get compiled and added to xv6's file system. If you crack open the Makefile, you'll see a variable called `UPROGS` that lists all the user programs:

```makefile
UPROGS=\
  _cat\
  _echo\
  _forktest\
  _grep\
  _init\
  _kill\
  _ln\
  _ls\
  _mkdir\
  _rm\
  _sh\
  _stressfs\
  _usertests\
  _wc\
  _zombie\
```

Each program gets compiled separately (unlike the kernel, which gets compiled as one big unit). The Makefile has an implicit rule that compiles each `.c` file into a `.o` object file, then links it with `ulib.o` and `usys.o` to create an executable. The `ulib.o` file contains the C library functions we saw earlier (like `printf`, `malloc`, `strcpy`, etc.), and `usys.o` contains the system call wrapper functions that user programs use to invoke system calls.

Then there's the `fs.img` target, which creates the file system image:

```makefile
fs.img: mkfs README $(UPROGS)
  ./mkfs fs.img README $(UPROGS)
```

This runs the `mkfs` utility (make file system), which creates a file system image called `fs.img` and populates it with the `README` file and all the user programs. When you boot xv6, this file system image gets loaded onto a virtual disk, so when you type `ls` at the shell prompt, you're seeing the files that `mkfs` put there.

If you want to add your own program to xv6 (which you probably will for any xv6 projects), you need to do two things: (1) add your program to the `UPROGS` list, and (2) add your `.c` file to the `EXTRA` list so it gets included in the distribution. Then rebuild xv6, and your program will show up in the file system.

## The Anatomy of a User Program

Every user program in xv6 follows the same basic structure. Let's look at the simplest one, `echo.c`, to see the pattern:

```c
#include "types.h"
#include "stat.h"
#include "user.h"

int
main(int argc, char *argv[])
{
  int i;

  for(i = 1; i < argc; i++){
    printf(1, "%s%s", argv[i], (i+1 < argc) ? " " : "\n");
  }
  exit();
}
```

First, we include three headers:
- `types.h` defines the basic types used in xv6 (`int`, `uint`, `short`, `ushort`, etc.)
- `stat.h` defines the `struct stat` used by file system system calls
- `user.h` declares all the system call wrappers and standard library functions available to user programs

Every program has a `main()` function that takes `argc` and `argv` as arguments, just like you'd expect from any C program. The shell will pass command-line arguments through these parameters when it execs the program.

Finally, and this is important: every program must call `exit()` at the end (or when it wants to terminate). If you forget to call `exit()`, the program will just keep executing whatever random instructions happen to be in memory after `main()` returns, which will probably cause a page fault and crash. Modern C runtimes handle this for you automatically, but xv6 doesn't, so you have to remember to call `exit()` yourself.

The `printf()` call here is interesting: the first argument is a file descriptor. In xv6's `printf()`, you always specify where you want the output to go. File descriptor 1 is standard output (stdout), so this prints to the console. If you wanted to print an error message, you'd use file descriptor 2 (stderr) instead.

## File Utilities

Most of the user programs in xv6 are simple Unix utilities for working with files. Let's look at a few of the most important ones.

### cat

The `cat` program reads files and writes their contents to standard output. It's one of the most useful utilities in Unix, and the implementation is dead simple:

```c
char buf[512];

void
cat(int fd)
{
  int n;

  while((n = read(fd, buf, sizeof(buf))) > 0) {
    if (write(1, buf, n) != n) {
      printf(1, "cat: write error\n");
      exit();
    }
  }
  if(n < 0){
    printf(1, "cat: read error\n");
    exit();
  }
}

int
main(int argc, char *argv[])
{
  int fd, i;

  if(argc <= 1){
    cat(0);
    exit();
  }

  for(i = 1; i < argc; i++){
    if((fd = open(argv[i], 0)) < 0){
      printf(1, "cat: cannot open %s\n", argv[i]);
      exit();
    }
    cat(fd);
    close(fd);
  }
  exit();
}
```

The `cat()` helper function takes a file descriptor and does all the work: it reads up to 512 bytes at a time from the file into a buffer, then writes that buffer to stdout (file descriptor 1). It keeps going until `read()` returns 0 (end of file) or a negative number (error).

The `main()` function handles the command-line arguments. If no arguments are given, it reads from stdin (file descriptor 0); this lets you use `cat` in a pipeline like `echo hello | cat`. Otherwise, it opens each file specified on the command line, cats it, and closes it.

This demonstrates a classic Unix pattern: write simple tools that can work on either files or standard input, so they can be composed together with pipes.

### wc

The `wc` (word count) program counts lines, words, and characters in a file. It's a bit more complex than `cat` because it has to parse the input, but it's still straightforward:

```c
char buf[512];

int
wc(int fd, char *name)
{
  int i, n;
  int l, w, c, inword;

  l = w = c = 0;
  inword = 0;
  while((n = read(fd, buf, sizeof(buf))) > 0){
    for(i=0; i<n; i++){
      c++;
      if(buf[i] == '\n')
        l++;
      if(strchr(" \r\t\n\v", buf[i]))
        inword = 0;
      else if(!inword){
        w++;
        inword = 1;
      }
    }
  }
  if(n < 0){
    printf(1, "wc: read error\n");
    exit();
  }
  printf(1, "%d %d %d %s\n", l, w, c, name);
  return 0;
}

int
main(int argc, char *argv[])
{
  int fd, i;

  if(argc <= 1){
    wc(0, "");
    exit();
  }

  for(i = 1; i < argc; i++){
    if((fd = open(argv[i], 0)) < 0){
      printf(1, "wc: cannot open %s\n", argv[i]);
      exit();
    }
    wc(fd, argv[i]);
    close(fd);
  }
  exit();
}
```

The `wc()` function reads the file in chunks, then iterates through each character. It counts newlines for lines (`l`), uses a simple state machine to count words (`w` and `inword`), and counts every character (`c`). A word is defined as any sequence of non-whitespace characters, so the `inword` flag tracks whether we're currently inside a word or not.

Again, `main()` follows the same pattern as `cat`: if no arguments are given, read from stdin; otherwise, process each file specified on the command line.

### grep

The `grep` program searches for lines matching a pattern. xv6's implementation is much simpler than GNU grep (no regular expressions, no fancy options), but it demonstrates the basic idea:

```c
char buf[1024];
int match(char*, char*);

void
grep(char *pattern, int fd)
{
  int n, m;
  char *p, *q;

  m = 0;
  while((n = read(fd, buf+m, sizeof(buf)-m-1)) > 0){
    m += n;
    buf[m] = '\0';
    p = buf;
    while((q = strchr(p, '\n')) != 0){
      *q = 0;
      if(match(pattern, p)){
        *q = '\n';
        write(1, p, q+1 - p);
      }
      p = q+1;
    }
    if(m > 0){
      m -= p - buf;
      memmove(buf, p, m);
    }
  }
}

int
main(int argc, char *argv[])
{
  int fd, i;
  char *pattern;

  if(argc <= 1){
    printf(2, "usage: grep pattern [file ...]\n");
    exit();
  }
  pattern = argv[1];

  if(argc <= 2){
    grep(pattern, 0);
    exit();
  }

  for(i = 2; i < argc; i++){
    if((fd = open(argv[i], 0)) < 0){
      printf(1, "grep: cannot open %s\n", argv[i]);
      exit();
    }
    grep(pattern, fd);
    close(fd);
  }
  exit();
}
```

The `grep()` function reads the input into a buffer, then splits it into lines by looking for newline characters with `strchr()`. For each line, it calls `match()` to check if the pattern appears anywhere in the line; if so, it prints the line to stdout.

The clever bit here is the buffer management: after processing complete lines, it copies any remaining partial line to the beginning of the buffer before reading more data. This ensures we don't lose any data or accidentally split a line in the middle.

The `match()` function (not shown here, but it's in the same file) does a simple substring search. It's not as powerful as real regular expressions, but it's good enough for basic searching.

## Directory Utilities

Unix systems organize files in a hierarchical directory structure, so xv6 includes a few utilities for navigating and manipulating directories.

### ls

The `ls` program lists the contents of a directory. It's more complex than the file utilities because it has to read directory entries and stat each file:

```c
char*
fmtname(char *path)
{
  static char buf[DIRSIZ+1];
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;

  // Return blank-padded name.
  if(strlen(p) >= DIRSIZ)
    return p;
  memmove(buf, p, strlen(p));
  memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));
  return buf;
}

void
ls(char *path)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){
    printf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    printf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_FILE:
    printf(1, "%s %d %d %d\n", fmtname(path), st.type, st.ino, st.size);
    break;

  case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf(1, "ls: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf(1, "ls: cannot stat %s\n", buf);
        continue;
      }
      printf(1, "%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  close(fd);
}

int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    ls(".");
    exit();
  }
  for(i=1; i<argc; i++)
    ls(argv[i]);
  exit();
}
```

The `ls()` function first opens the path and stats it to see if it's a file or a directory. If it's a file, it just prints the file's information. If it's a directory, things get more interesting:

1. It reads directory entries one at a time. Each `struct dirent` contains an inode number and a name.
2. For each entry, it constructs the full path by appending the entry name to the directory path.
3. It stats that path to get the file's metadata (type, inode number, size).
4. It prints the formatted information.

The `fmtname()` helper function formats the file name to be right-padded with spaces to a fixed width (DIRSIZ characters), which makes the output line up nicely in columns.

### mkdir and rm

These utilities are much simpler than `ls`:

```c
// mkdir.c
int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    printf(2, "Usage: mkdir files...\n");
    exit();
  }

  for(i = 1; i < argc; i++){
    if(mkdir(argv[i]) < 0){
      printf(2, "mkdir: %s failed to create\n", argv[i]);
      break;
    }
  }

  exit();
}
```

```c
// rm.c
int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    printf(2, "Usage: rm files...\n");
    exit();
  }

  for(i = 1; i < argc; i++){
    if(unlink(argv[i]) < 0){
      printf(2, "rm: %s failed to delete\n", argv[i]);
      break;
    }
  }

  exit();
}
```

These are basically just thin wrappers around the `mkdir()` and `unlink()` system calls. They take a list of file names on the command line and create/delete each one. Simple, but effective.

### ln

The `ln` program creates links (hard links, specifically). It's another thin wrapper around a system call:

```c
int
main(int argc, char *argv[])
{
  if(argc != 3){
    printf(2, "Usage: ln old new\n");
    exit();
  }
  if(link(argv[1], argv[2]) < 0)
    printf(2, "link %s %s: failed\n", argv[1], argv[2]);
  exit();
}
```

The `link()` system call creates a new directory entry that points to the same inode as an existing file. So after running `ln foo bar`, both `foo` and `bar` refer to the same file on disk. Changes to one are immediately visible through the other, because they're literally the same file.

## Process Utilities

Unix is all about processes, so xv6 includes a utility for killing them:

### kill

```c
int
main(int argc, char **argv)
{
  int i;

  if(argc < 2){
    printf(2, "usage: kill pid...\n");
    exit();
  }
  for(i=1; i<argc; i++)
    kill(atoi(argv[i]));
  exit();
}
```

This is as simple as it gets: it takes process IDs as arguments, converts them from strings to integers with `atoi()`, and calls the `kill()` system call on each one. The `kill()` system call marks the process as killed; the next time that process enters or exits the kernel (via a system call, trap, or interrupt), it'll be terminated.

Note that xv6 doesn't have signals like real Unix systems do, so `kill()` is much simpler here—it's essentially a "forcefully terminate this process" command with no nuance.

## Test Programs

xv6 includes a few programs that are used for testing rather than as user-facing utilities:

### zombie

```c
int
main(void)
{
  if(fork() > 0)
    sleep(5);  // Let child exit before parent.
  exit();
}
```

This program demonstrates zombie processes. The child process exits immediately, but the parent sleeps for 5 seconds before exiting. During that time, the child is a zombie: it's terminated, but its parent hasn't called `wait()` yet, so the kernel keeps its process slot around. When you run this program, you can press Ctrl-P during those 5 seconds to see the zombie process in the process list.

### forktest

This program tests the fork implementation by creating a deeply nested tree of processes. It's useful for stress-testing the process management code:

```c
void
forktest(void)
{
  int n, pid;

  printf(1, "fork test\n");

  for(n=0; n<N; n++){
    pid = fork();
    if(pid < 0)
      break;
    if(pid == 0)
      exit();
  }

  if(n == N){
    printf(1, "fork claimed to work N times!\n", N);
    exit();
  }

  for(; n > 0; n--){
    if(wait() < 0){
      printf(1, "wait stopped early\n");
      exit();
    }
  }

  if(wait() != -1){
    printf(1, "wait got too many\n");
    exit();
  }

  printf(1, "fork test OK\n");
}
```

The test forks N times (where N is a large number like 1000), with each child immediately exiting. Then it waits for all N children to exit. This tests that fork can handle many processes and that wait correctly reports when all children have exited.

### stressfs

This program stress-tests the file system by creating many files, writing to them, and reading them back:

```c
int
main(int argc, char *argv[])
{
  int fd, i;
  char path[] = "stressfs0";
  char data[512];

  printf(1, "stressfs starting\n");
  memset(data, 'a', sizeof(data));

  for(i = 0; i < 4; i++)
    if(fork() > 0)
      break;

  printf(1, "write %d\n", i);

  path[8] += i;
  fd = open(path, O_CREATE | O_RDWR);
  for(i = 0; i < 20; i++)
    //    printf(fd, "%d\n", i);
    write(fd, data, sizeof(data));
  close(fd);

  printf(1, "read\n");

  fd = open(path, O_RDONLY);
  for (i = 0; i < 20; i++)
    read(fd, data, sizeof(data));
  close(fd);

  wait();

  exit();
}
```

It forks 4 child processes, and each one creates a different file (`stressfs0`, `stressfs1`, etc.), writes to it, and reads it back. This tests concurrent file system operations and helps catch bugs in the file system implementation.

### usertests

This is a comprehensive test suite that exercises many different parts of the system. It's a large file with many test functions, each testing a different feature (pipes, file descriptors, fork, exec, memory, etc.). If you're hacking on xv6, you should run `usertests` to make sure you haven't broken anything.

## What We Learned

User space programs in xv6 are simple, but they demonstrate important Unix design principles:

1. **Small, composable tools**: Each program does one thing well. You can combine them with pipes to accomplish complex tasks.

2. **Everything is a file**: Programs use the same system calls (open, read, write, close) to work with files, pipes, and devices. The kernel abstracts away the differences.

3. **Standard input/output/error**: Programs can work with files or standard input, making them flexible and composable. Error messages go to stderr so they don't pollute the output.

4. **Simple error handling**: When something goes wrong, print an error message and exit. Don't try to be clever about recovering from errors.

5. **Command-line arguments**: The shell passes arguments to programs through argc/argv, making it easy to parameterize behavior.

These programs also show you the pattern for writing your own xv6 programs: include the right headers, write a main function that processes command-line arguments, use system calls to interact with the kernel, and remember to call exit() at the end.

If you're doing xv6 projects, you'll probably write programs that look a lot like these. The good news is that once you've seen one or two, you've seen the pattern. The kernel does all the hard work; user programs just have to make the right system calls in the right order.

And with that, we've completed our tour of xv6, from the boot loader all the way up to user space programs. You've seen how a real operating system works from the ground up: how it boots, how it manages memory and processes, how it schedules work, how it handles interrupts and system calls, how it abstracts hardware, and how it presents a simple interface to user programs. Not bad for an OS that fits in a few thousand lines of code!
