# User Space: C Library

When you write a user program in a typical Unix system, you have the entire C standard library at your disposal -- functions like `printf`, `malloc`, `strcpy`, and hundreds of others. But xv6 is a teaching operating system that's designed to be simple and understandable, so it doesn't include the full standard C library. Instead, it provides a minimal C library that implements just the essential functions that user programs need.

This minimal library is split across three files: ulib.c (string and utility functions), printf.c (formatted output), and umalloc.c (dynamic memory allocation). Together, these files provide enough functionality for xv6's user programs to work, while keeping the implementation small enough to understand completely.

One interesting thing to note: xv6 user programs are statically linked against these library files. There's no shared library support, no libc.so -- each user program contains its own copy of the library code. This is simpler but means every program has a larger binary size. For a teaching OS with small programs, this trade-off is perfectly reasonable.

## ulib.c

Let's start with ulib.c, which contains string manipulation functions, memory operations, and a few utility functions.

```c
#include "types.h"
#include "stat.h"
#include "fcntl.h"
#include "user.h"
#include "x86.h"
```

Standard includes for user programs. The `user.h` header declares the system call wrappers and library functions, while `x86.h` contains x86-specific assembly routines.

### String Functions

#### strcpy()

```c
char*
strcpy(char *s, const char *t)
{
  char *os;

  os = s;
  while((*s++ = *t++) != 0)
    ;
  return os;
}
```

This is the classic string copy function. It copies characters from `t` to `s` until it hits a null terminator. The function returns the original destination pointer (`os`), which is a common convention that allows chaining function calls.

The implementation is textbook K&R C: the loop condition both copies a character and checks if it's the null terminator, all in one expression. This is idiomatic C but can be confusing if you're not used to it. It's equivalent to:
```c
while(1) {
  *s = *t;
  if(*t == 0) break;
  s++;
  t++;
}
```

#### strcmp()

```c
int
strcmp(const char *p, const char *q)
{
  while(*p && *p == *q)
    p++, q++;
  return (uchar)*p - (uchar)*q;
}
```

String comparison. This function returns 0 if the strings are equal, a negative number if `p` comes before `q` lexicographically, or a positive number if `p` comes after `q`.

The loop advances both pointers as long as the characters match and we haven't hit a null terminator. Once we exit the loop, we return the difference between the two characters. The cast to `uchar` ensures we treat the characters as unsigned, which gives us consistent ordering.

Why does this work? If the strings are equal, both `*p` and `*q` will be null terminators (0), so the difference is 0. If they differ, the difference tells us which string comes first.

#### strlen()

```c
uint
strlen(const char *s)
{
  int n;

  for(n = 0; s[n]; n++)
    ;
  return n;
}
```

String length. This counts characters until it hits a null terminator. Simple and straightforward -- just iterate and count.

Note that the return type is `uint` (unsigned int) rather than `size_t` like the standard C library uses. xv6 keeps things simple and just uses basic integer types.

#### strchr()

```c
char*
strchr(const char *s, char c)
{
  for(; *s; s++)
    if(*s == c)
      return (char*)s;
  return 0;
}
```

Find a character in a string. This function scans through the string looking for the character `c`, returning a pointer to the first occurrence, or null if it's not found.

The implementation is straightforward: iterate through the string, and return as soon as we find a match. If we reach the end (null terminator) without finding it, return 0 (null).

### Memory Functions

#### memset()

```c
void*
memset(void *dst, int c, uint n)
{
  stosb(dst, c, n);
  return dst;
}
```

Set a region of memory to a specific byte value. This is commonly used to zero out structures or arrays.

The interesting part is that xv6 doesn't implement this in C -- it delegates to `stosb()`, which is defined in x86.h as inline assembly:

```c
static inline void
stosb(void *addr, int data, int cnt)
{
  asm volatile("cld; rep stosb" :
               "=D" (addr), "=c" (cnt) :
               "0" (addr), "1" (cnt), "a" (data) :
               "memory", "cc");
}
```

This uses the x86 `rep stosb` instruction, which is highly optimized for filling memory. It's much faster than a C loop. The instruction repeats `stosb` (store string byte) `cnt` times, storing the byte in register `al` (the low byte of `eax`) to memory starting at address in `edi`.

Why use assembly? For commonly-used operations like `memset`, the performance difference can be significant. And since xv6 is tied to x86 anyway, there's no portability concern.

#### memmove()

```c
void*
memmove(void *vdst, const void *vsrc, int n)
{
  char *dst;
  const char *src;

  dst = vdst;
  src = vsrc;
  while(n-- > 0)
    *dst++ = *src++;
  return vdst;
}
```

Copy a region of memory from source to destination. Unlike `memcpy`, `memmove` is supposed to handle overlapping regions correctly, but xv6's implementation doesn't actually do this properly!

The standard `memmove` checks if the regions overlap and copies backwards if necessary to avoid overwriting source data before it's copied. But xv6's version always copies forwards, so it doesn't handle all overlap cases correctly.

For example, if `dst` is within `src`'s range and comes after `src`, copying forwards will overwrite parts of the source before they're copied. The proper implementation would detect this and copy backwards instead.

This is one of those cases where xv6 takes a shortcut for simplicity. In practice, xv6's code is careful not to call `memmove` with problematic overlapping regions, so it works out fine.

### Utility Functions

#### gets()

```c
char*
gets(char *buf, int max)
{
  int i, cc;
  char c;

  for(i=0; i+1 < max; ){
    cc = read(0, &c, 1);
    if(cc < 1)
      break;
    buf[i++] = c;
    if(c == '\n' || c == '\r')
      break;
  }
  buf[i] = '\0';
  return buf;
}
```

Read a line from standard input (file descriptor 0). This function reads one character at a time until it hits a newline, carriage return, end-of-file, or the buffer fills up. It always null-terminates the result.

The `i+1 < max` condition ensures we always leave room for the null terminator. We stop reading when we get a newline or carriage return, or when `read()` returns less than 1 (indicating EOF or an error).

This is a simple but inefficient implementation -- reading one character at a time means we make a system call for every character. A more efficient version would buffer the input and read larger chunks. But for a teaching OS, simplicity wins over efficiency.

#### stat()

```c
int
stat(const char *n, struct stat *st)
{
  int fd;
  int r;

  fd = open(n, O_RDONLY);
  if(fd < 0)
    return -1;
  r = fstat(fd, st);
  close(fd);
  return r;
}
```

Get file status information. This is a convenience wrapper around the `fstat` system call. Since `fstat` requires a file descriptor, we open the file, call `fstat`, and then close it.

This is a common pattern in Unix: some system calls work on file descriptors, and we provide library wrappers that work on pathnames for convenience. The kernel provides the low-level interface (`fstat`), and the library provides the more convenient interface (`stat`).

#### atoi()

```c
int
atoi(const char *s)
{
  int n;

  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
```

Convert a string to an integer. This is the classic atoi (ASCII to integer) function.

The implementation is simple: start with 0, and for each digit character, multiply the accumulated value by 10 and add the digit value. We get the digit value by subtracting '0' from the character (since '0' through '9' are consecutive in ASCII).

Note that this implementation doesn't handle negative numbers, leading/trailing whitespace, or error checking. It just stops when it hits a non-digit character. This is another example of xv6 keeping things minimal -- the shell and other programs don't need fancy number parsing.

## printf.c

Now let's look at printf.c, which implements formatted output for user programs.

```c
#include "types.h"
#include "stat.h"
#include "user.h"
```

### Helper Functions

#### putc()

```c
static void
putc(int fd, char c)
{
  write(fd, &c, 1);
}
```

Write a single character to a file descriptor. This is just a wrapper around the `write` system call for convenience. It's declared `static` because it's only used within this file.

#### printint()

```c
static void
printint(int fd, int xx, int base, int sgn)
{
  static char digits[] = "0123456789ABCDEF";
  char buf[16];
  int i, neg;
  uint x;

  neg = 0;
  if(sgn && xx < 0){
    neg = 1;
    x = -xx;
  } else {
    x = xx;
  }

  i = 0;
  do{
    buf[i++] = digits[x % base];
  }while((x /= base) != 0);
  if(neg)
    buf[i++] = '-';

  while(--i >= 0)
    putc(fd, buf[i]);
}
```

Print an integer in a given base (10 for decimal, 16 for hexadecimal). This is the workhorse function for printing numbers.

The algorithm is standard: repeatedly divide by the base, taking the remainder as each digit. The digits are generated right-to-left (least significant first), so we store them in a buffer and then output them in reverse order.

The `sgn` parameter indicates whether to treat the number as signed. If it's signed and negative, we make it positive, set a flag, and add a minus sign at the end (which will appear at the beginning when we output in reverse).

The `digits` array maps digit values (0-15) to their character representations. For base 10, we only use '0' through '9'. For base 16, we use '0' through 'F'.

Why 16 bytes for the buffer? In the worst case (32-bit integer in base 10), we need at most 10 digits, plus 1 for the minus sign, plus 1 for safety. 16 is a nice round number that's definitely enough.

### printf()

```c
void
printf(int fd, const char *fmt, ...)
{
  char *s;
  int c, i, state;
  uint *ap;

  state = 0;
  ap = (uint*)(void*)(&fmt + 1);
```

The main printf function. Unlike standard C printf, xv6's version takes a file descriptor as the first argument, so you can print to any file, not just stdout.

The `...` indicates a variadic function -- it can take a variable number of arguments. The `ap` (argument pointer) points to the first argument after `fmt`. In C's calling convention, arguments are pushed on the stack in reverse order, so `&fmt + 1` points to the next argument.

The `state` variable tracks whether we're in the middle of processing a format specifier. It's 0 for normal characters, and '%' when we've just seen a percent sign.

```c
  for(i = 0; fmt[i]; i++){
    c = fmt[i] & 0xff;
    if(state == 0){
      if(c == '%'){
        state = '%';
      } else {
        putc(fd, c);
      }
    } else if(state == '%'){
      if(c == 'd'){
        printint(fd, *ap, 10, 1);
        ap++;
      } else if(c == 'x' || c == 'p'){
        printint(fd, *ap, 16, 0);
        ap++;
      } else if(c == 's'){
        s = (char*)*ap;
        ap++;
        if(s == 0)
          s = "(null)";
        while(*s != 0){
          putc(fd, *s);
          s++;
        }
      } else if(c == 'c'){
        putc(fd, *ap);
        ap++;
      } else if(c == '%'){
        putc(fd, c);
      } else {
        // Unknown % sequence.  Print it to draw attention.
        putc(fd, '%');
        putc(fd, c);
      }
      state = 0;
    }
  }
}
```

The main loop processes the format string character by character. When we're in state 0 (normal), we output characters as-is unless we see a '%', which switches us to state '%'. When we're in state '%', we look at the next character to determine the format:

- `%d`: Signed decimal integer
- `%x` or `%p`: Hexadecimal integer (unsigned). Both are treated identically; `%p` is conventionally used for pointers.
- `%s`: String
- `%c`: Single character
- `%%`: Literal percent sign

For integer and character formats, we consume the next argument from `ap` and advance the pointer. For strings, we output characters until we hit the null terminator. If the string pointer is null, we output "(null)" instead of crashing.

If we see an unknown format specifier, we output it literally (with the '%') to draw attention to the bug. This is better than silently ignoring it or crashing.

This is a much simpler printf than the standard C library version. It doesn't support:
- Field width specifications (like `%10d`)
- Precision (like `%.2f`)
- Floating point (`%f`)
- Long integers (`%ld`)
- Zero-padding (`%08x`)
- Left/right justification
- Dynamic width/precision (`%*d`)

But for a teaching OS, this simple version is perfectly adequate. The xv6 programs don't need fancy formatting.

## umalloc.c

Finally, let's look at umalloc.c, which implements dynamic memory allocation (malloc and free).

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "param.h"

// Memory allocator by Kernighan and Ritchie,
// The C programming Language, 2nd ed.  Section 8.7.
```

The comment tells us this is the classic K&R malloc implementation from "The C Programming Language". This is the same algorithm that many Unix systems used for years. It's simple, reasonably efficient, and well-understood.

### Data Structures

```c
typedef long Align;

union header {
  struct {
    union header *ptr;
    uint size;
  } s;
  Align x;
};

typedef union header Header;

static Header base;
static Header *freep;
```

The memory allocator maintains a circular linked list of free blocks. Each block has a header that contains:
- `ptr`: Pointer to the next free block
- `size`: Size of this block in units of `sizeof(Header)`

The union is a clever trick to ensure proper alignment. The `Align x` member (which is a `long`, typically 4 or 8 bytes) forces the entire union to be aligned on a long boundary. This ensures that any memory allocated will be properly aligned for any data type.

The `base` is a dummy header that serves as the starting point of the free list. It's never actually used to store data; it just simplifies the list manipulation code by ensuring the list is never empty.

The `freep` pointer always points to the free list entry where we last allocated or freed memory. This implements a "next-fit" strategy -- we start searching for free blocks from where we left off last time, which can be more efficient than always starting from the beginning.

### free()

```c
void
free(void *ap)
{
  Header *bp, *p;

  bp = (Header*)ap - 1;
```

To free a block, we first back up to the header. Remember that when we allocate memory, we return a pointer to the memory after the header, so users don't accidentally corrupt the header. To free, we need to get back to the header, which is one `Header` before the pointer we were given.

```c
  for(p = freep; !(bp > p && bp < p->s.ptr); p = p->s.ptr)
    if(p >= p->s.ptr && (bp > p || bp < p->s.ptr))
      break;
```

Now we need to find where to insert this block into the free list. The free list is kept sorted by address, which makes it easier to coalesce adjacent free blocks.

We scan through the free list looking for the spot where `bp` belongs -- specifically, we want to find two adjacent entries `p` and `p->s.ptr` such that `bp`'s address falls between them.

The second condition handles the wraparound case: if `p >= p->s.ptr`, we're at the end of the list (it wraps around to the beginning). In this case, we want to insert `bp` if it's either after `p` (at the end) or before `p->s.ptr` (at the beginning).

```c
  if(bp + bp->s.size == p->s.ptr){
    bp->s.size += p->s.ptr->s.size;
    bp->s.ptr = p->s.ptr->s.ptr;
  } else
    bp->s.ptr = p->s.ptr;
```

Now we try to coalesce with the next block. If the block we're freeing (`bp`) is immediately adjacent to the next free block (`p->s.ptr`), we merge them into a single larger block. We do this by adding the sizes and skipping over the next block's header.

The address arithmetic `bp + bp->s.size` works because `bp` is a `Header*`, so adding `bp->s.size` advances by that many `Header`-sized units.

```c
  if(p + p->s.size == bp){
    p->s.size += bp->s.size;
    p->s.ptr = bp->s.ptr;
  } else
    p->s.ptr = bp;
  freep = p;
}
```

Now we try to coalesce with the previous block. If the previous free block (`p`) is immediately adjacent to the block we're freeing, we merge them similarly.

After coalescing (or not), we link the block into the free list and update `freep` to point to it. This means the next allocation will start searching from here.

Coalescing is important to avoid fragmentation. Without it, we'd end up with lots of small free blocks that can't be used for larger allocations, even though there's plenty of free memory in total.

### morecore()

```c
static Header*
morecore(uint nu)
{
  char *p;
  Header *hp;

  if(nu < 4096)
    nu = 4096;
  p = sbrk(nu * sizeof(Header));
  if(p == (char*)-1)
    return 0;
```

When we run out of free memory, we need to ask the kernel for more. The `sbrk` system call expands the process's heap by the given number of bytes. It returns a pointer to the start of the new memory, or -1 on failure.

We allocate at least 4096 units (which is typically 16-32 KB) at a time. This reduces the number of system calls and provides a reasonable chunk of memory to work with. Allocating in large chunks is more efficient than allocating small amounts many times.

```c
  hp = (Header*)p;
  hp->s.size = nu;
  free((void*)(hp + 1));
  return freep;
}
```

Here's a clever trick: instead of directly adding the new memory to the free list, we fake-allocate it and then immediately free it. This lets us reuse the `free()` function's logic for inserting into the free list and potentially coalescing with adjacent blocks.

We set up a header at the start of the new memory, set its size, and then call `free()` with a pointer to the memory after the header (just like a real allocated block). The `free()` function will properly insert it into the free list and handle any coalescing.

### malloc()

```c
void*
malloc(uint nbytes)
{
  Header *p, *prevp;
  uint nunits;

  nunits = (nbytes + sizeof(Header) - 1)/sizeof(Header) + 1;
```

To allocate memory, we first calculate how many units we need. We add `sizeof(Header)` to the requested size (for the header itself), round up to the nearest `Header` size, and add 1 to ensure we always have at least one unit.

The rounding up is important for alignment: we always allocate in multiples of `Header` size, which ensures proper alignment for any data type.

```c
  if((prevp = freep) == 0){
    base.s.ptr = freep = prevp = &base;
    base.s.size = 0;
  }
```

If this is the first allocation ever, we need to initialize the free list. We set up the `base` dummy header to point to itself, creating a circular list with one entry. From now on, `freep` will always point to a valid entry.

```c
  for(p = prevp->s.ptr; ; prevp = p, p = p->s.ptr){
    if(p->s.size >= nunits){
      if(p->s.size == nunits)
        prevp->s.ptr = p->s.ptr;
      else {
        p->s.size -= nunits;
        p += p->s.size;
        p->s.size = nunits;
      }
      freep = prevp;
      return (void*)(p + 1);
    }
```

Now we search the free list for a block that's large enough. We use a "first-fit" strategy: we take the first block that's big enough.

If we find a block that's exactly the right size, we unlink it from the free list and return it. If it's larger, we split it: we shrink the free block by `nunits`, and carve out a new block of size `nunits` at the end of the free block. This lets us allocate from the end while keeping the free block at the beginning with its header intact.

The pointer arithmetic `p + p->s.size` gives us a pointer to the end of the (now smaller) free block, which is where we'll put the header for the allocated block.

We return `p + 1`, which points to the memory after the header -- this is what the user will use. We also update `freep` to point to `prevp`, so the next allocation will start searching from here.

```c
    if(p == freep)
      if((p = morecore(nunits)) == 0)
        return 0;
  }
}
```

If we've gone all the way around the circular list without finding a large enough block, we need more memory from the kernel. We call `morecore()`, which will expand the heap and add the new memory to the free list.

If `morecore()` fails (maybe we're out of memory), we return 0 (null pointer) to indicate failure.

After `morecore()` succeeds, the loop continues, and we'll find the newly-added block on the next iteration.

## How It All Works Together

Let's trace through what happens when a user program calls `printf`:

1. The program calls `printf(1, "Hello %d\n", 42)`
2. `printf` processes the format string, outputting characters one at a time
3. When it sees `%d`, it calls `printint(1, 42, 10, 1)`
4. `printint` converts 42 to the string "42" in a buffer, then outputs each character
5. Each `putc` call invokes the `write` system call
6. The kernel's `sys_write` function routes this to the console driver
7. The console driver outputs the characters to both the serial port and CGA display
8. We see "Hello 42" on our screen!

For a malloc/free sequence:

1. Program calls `malloc(100)` to allocate 100 bytes
2. `malloc` calculates it needs about 7 units (each unit is typically 16 bytes)
3. It searches the free list for a block large enough
4. If found, it carves out the needed space and returns a pointer
5. The program uses the memory
6. Eventually, the program calls `free()` with the pointer
7. `free` backs up to the header, finds the right spot in the free list, and inserts it
8. It tries to coalesce with adjacent free blocks to reduce fragmentation

## Summary

xv6's C library is minimal but functional. It provides:

1. **String functions** (strcpy, strcmp, strlen, strchr) - basic string manipulation
2. **Memory functions** (memset, memmove) - memory operations, using x86 assembly for performance
3. **Utility functions** (gets, stat, atoi) - convenience wrappers and basic parsing
4. **Formatted output** (printf) - simple but adequate formatting for debugging and user interaction
5. **Dynamic allocation** (malloc, free) - the classic K&R algorithm with free list and coalescing

Key design decisions:

- **Simplicity over features**: xv6's library implements only what's needed, not the full C standard library
- **Static linking**: Each program contains its own copy of the library code
- **Classic algorithms**: The K&R malloc is well-understood and reasonably efficient
- **x86-specific optimizations**: Using assembly for common operations like memset
- **Minimal error handling**: Functions assume valid inputs and don't do extensive checking

The library is about 300 lines of code total (compared to tens of thousands in glibc), yet it's enough for xv6's shell, utilities, and test programs to work. This demonstrates the principle that a lot can be done with very little code, as long as you're willing to make trade-offs and focus on the essential functionality.

For a student learning operating systems, this C library is perfect: it's small enough to understand completely, yet realistic enough to show how user-space libraries work in practice. And if you ever wondered how malloc actually works under the hood, now you know!
