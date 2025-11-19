# Devices: Serial Port and Console Drivers

One of the first things you notice when you boot up xv6 is text appearing on your screen. You can type commands, see output, and interact with the system -- all through the console. But how does text get from the kernel to your screen, and from your keyboard to the kernel? That's what we're going to explore in this post.

The console in xv6 is actually a combination of two different output devices and two different input devices. For output, xv6 writes to both the serial port (UART) and the CGA text display. For input, xv6 reads from both the serial port and the keyboard. This design allows xv6 to work whether you're running it on real hardware with a monitor and keyboard, or in an emulator like QEMU that connects the serial port to your terminal.

The code is split across two files: uart.c handles the serial port hardware, while console.c provides the higher-level console interface that coordinates between all the input/output devices.

## Serial Ports and UARTs

Before we dive into the code, let's talk about what a serial port actually is. A UART (Universal Asynchronous Receiver/Transmitter) is a hardware device that converts parallel data (bytes) into a serial bit stream for transmission, and vice versa for reception. This is how computers communicate over serial cables -- one bit at a time, transmitted sequentially.

The original IBM PC had serial ports at specific I/O port addresses. The first serial port, COM1, is at I/O port 0x3F8. The UART has multiple registers at consecutive port addresses that control different aspects of its operation: data transmission/reception, interrupt enable, line control, etc.

In modern systems, you might not have a physical serial port, but emulators like QEMU provide virtual serial ports that can be connected to your terminal. This is why when you run xv6 in QEMU with the `-serial mon:stdio` flag, the serial port output appears in your terminal window.

## uart.c

Let's start with the simpler of the two files: uart.c, which handles the serial port hardware.

```c
#include "types.h"
#include "defs.h"
#include "param.h"
#include "traps.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "fs.h"
#include "file.h"
#include "mmu.h"
#include "proc.h"
#include "x86.h"
```

Standard includes. We need a bunch of these because the console interacts with many parts of the system.

### UART Register Definitions

```c
#define COM1    0x3f8

static int uart;    // is there a uart?
```

COM1 is the I/O port address for the first serial port. The `uart` variable is a flag that indicates whether we successfully initialized the UART. If there's no UART present (or if initialization failed), this will be 0, and the UART functions will do nothing.

The UART has several registers at consecutive port addresses starting from COM1:
- COM1+0: Data register (for reading/writing bytes)
- COM1+1: Interrupt Enable Register
- COM1+2: Interrupt Identification and FIFO control
- COM1+3: Line Control Register
- COM1+4: Modem Control Register  
- COM1+5: Line Status Register

### uartinit()

```c
void
uartinit(void)
{
  char *p;

  // Turn off the FIFO
  outb(COM1+2, 0);
```

This function initializes the UART during kernel startup. It's called from `consoleinit()` in console.c.

First, we turn off the FIFO (First In First Out buffer). The UART has a built-in buffer that can hold multiple bytes, but xv6 disables it for simplicity. This means the UART will interrupt immediately when each character arrives or is transmitted, rather than batching multiple characters.

```c
  // 9600 baud, 8 data bits, 1 stop bit, parity off.
  outb(COM1+3, 0x80);    // Unlock divisor
  outb(COM1+0, 115200/9600);
  outb(COM1+1, 0);
  outb(COM1+3, 0x03);    // Lock divisor, 8 data bits.
  outb(COM1+4, 0);
  outb(COM1+1, 0x01);    // Enable receive interrupts.
```

Now we configure the UART's transmission parameters. The baud rate (bits per second) is set by writing a divisor to the divisor latch registers. The formula is: divisor = 115200 / desired_baud_rate. For 9600 baud, that's 115200/9600 = 12.

To access the divisor latch registers, we first need to set bit 7 (the DLAB bit) in the Line Control Register (COM1+3). This "unlocks" the divisor by making COM1+0 and COM1+1 become the divisor latch registers instead of the normal data and interrupt enable registers. We write the divisor (12) to these registers, then clear bit 7 to "lock" the divisor and return COM1+0 and COM1+1 to their normal functions.

The value 0x03 written to COM1+3 configures 8 data bits and 1 stop bit, with no parity. This is a standard serial port configuration.

We write 0 to COM1+4 (Modem Control Register), which keeps all the modem control signals inactive. Then we write 0x01 to COM1+1 to enable receive interrupts -- this means the UART will interrupt the CPU whenever it receives a byte.

```c
  // If status is 0xFF, no serial port.
  if(inb(COM1+5) == 0xFF)
    return;
  uart = 1;
```

We read the Line Status Register (COM1+5) to check if there's actually a UART present. If we get back 0xFF, that means there's no device at this I/O port address, so we return without enabling the UART. Otherwise, we set the `uart` flag to 1 to indicate that the UART is available.

```c
  // Acknowledge pre-existing interrupt conditions;
  // enable interrupts.
  inb(COM1+2);
  inb(COM1+0);
  ioapicenable(IRQ_COM1, 0);
```

We read from the Interrupt Identification Register (COM1+2) and the Data Register (COM1+0) to clear any pending interrupts from before we initialized the UART. Then we call `ioapicenable()` to route COM1 interrupts (IRQ 4) to CPU 0.

```c
  // Announce that we're here.
  for(p="xv6...\n"; *p; p++)
    uartputc(*p);
}
```

Finally, we print a startup message to the serial port. This is the "xv6..." you see when xv6 boots. It's a nice way to verify that the UART is working.

### uartputc()

```c
void
uartputc(int c)
{
  int i;

  if(!uart)
    return;
```

This function writes a character to the serial port. If the UART isn't initialized, we just return immediately.

```c
  for(i = 0; i < 128 && !(inb(COM1+5) & 0x20); i++)
    microdelay(10);
  outb(COM1+0, c);
}
```

Before we can write a character, we need to wait for the UART to be ready. Bit 5 of the Line Status Register (COM1+5) is the "transmit holding register empty" bit -- it's set to 1 when the UART is ready to accept another character.

We poll this bit, waiting for it to become 1. We'll wait for up to 128 iterations with a 10 microsecond delay between each check. This gives the UART time to transmit the previous character. Once the UART is ready (or we've timed out), we write the character to the Data Register (COM1+0).

Note that `microdelay()` is actually a no-op in xv6, as we discussed in the LAPIC section. The delay is only useful on real hardware; emulators like QEMU are slow enough that the polling loop provides sufficient delay.

### uartgetc()

```c
static int
uartgetc(void)
{
  if(!uart)
    return -1;
  if(!(inb(COM1+5) & 0x01))
    return -1;
  return inb(COM1+0);
}
```

This function reads a character from the serial port, or returns -1 if there's no character available. It's declared `static` because it's only used within uart.c as a helper function passed to `consoleintr()`.

Bit 0 of the Line Status Register indicates "data ready" -- it's set to 1 when the UART has received a character. If this bit is clear, there's no character available, so we return -1. Otherwise, we read and return the character from the Data Register.

### uartintr()

```c
void
uartintr(void)
{
  consoleintr(uartgetc);
}
```

This is the UART interrupt handler, called from trap.c when IRQ 4 (COM1) fires. It simply calls `consoleintr()` with a pointer to `uartgetc()`. The `consoleintr()` function will repeatedly call `uartgetc()` to read all available characters from the UART and process them.

This is a nice design pattern: the device-specific code (uart.c) handles the low-level hardware details, while the generic console code (console.c) handles the higher-level processing like line editing and echoing characters back to the user.

## console.c

Now let's look at console.c, which provides the high-level console interface and coordinates between all the I/O devices.

```c
#include "types.h"
#include "defs.h"
#include "param.h"
#include "traps.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "fs.h"
#include "file.h"
#include "memlayout.h"
#include "mmu.h"
#include "proc.h"
#include "x86.h"
```

Same includes as uart.c. The console needs access to file structures, process structures, and trap numbers.

### Global Variables and Structures

```c
static void consputc(int);

static int panicked = 0;

static struct {
  struct spinlock lock;
  int locking;
} cons;
```

We have a few global variables:
- `panicked`: Set to 1 when the kernel panics, which disables normal console output
- `cons`: A structure containing a spinlock for synchronizing console access, and a `locking` flag that indicates whether locking is enabled (it's disabled during early boot and after a panic)

### Helper Functions for Output

#### printint()

```c
static void
printint(int xx, int base, int sign)
{
  static char digits[] = "0123456789abcdef";
  char buf[16];
  int i;
  uint x;

  if(sign && (sign = xx < 0))
    x = -xx;
  else
    x = xx;

  i = 0;
  do{
    buf[i++] = digits[x % base];
  }while((x /= base) != 0);

  if(sign)
    buf[i++] = '-';

  while(--i >= 0)
    consputc(buf[i]);
}
```

This helper function converts an integer to a string in a given base (10 for decimal, 16 for hexadecimal) and outputs it character by character. The algorithm builds the string in reverse order in a buffer, then outputs it from right to left (which is left to right in the final output).

The `sign` parameter indicates whether to treat the number as signed. If it's negative, we make it positive, set a flag, and add a minus sign at the end (which will appear at the beginning in the output).

#### cprintf()

```c
void
cprintf(char *fmt, ...)
{
  int i, c, locking;
  uint *argp;
  char *s;

  locking = cons.locking;
  if(locking)
    acquire(&cons.lock);

  if (fmt == 0)
    panic("null fmt");

  argp = (uint*)(void*)(&fmt + 1);
```

This is xv6's version of printf. It's a variadic function that takes a format string and a variable number of arguments. The first thing we do is acquire the console lock if locking is enabled. This prevents multiple CPUs from trying to print at the same time and getting garbled output.

The `argp` pointer points to the first argument after `fmt`. In C's calling convention, arguments are pushed onto the stack in reverse order, so `&fmt + 1` points to the next argument.

```c
  for(i = 0; (c = fmt[i] & 0xff) != 0; i++){
    if(c != '%'){
      consputc(c);
      continue;
    }
    c = fmt[++i] & 0xff;
    if(c == 0)
      break;
    switch(c){
    case 'd':
      printint(*argp++, 10, 1);
      break;
    case 'x':
    case 'p':
      printint(*argp++, 16, 0);
      break;
    case 's':
      if((s = (char*)*argp++) == 0)
        s = "(null)";
      for(; *s; s++)
        consputc(*s);
      break;
    case '%':
      consputc('%');
      break;
    default:
      // Print unknown % sequence to draw attention.
      consputc('%');
      consputc(c);
      break;
    }
  }
```

We loop through the format string, looking for format specifiers (marked by %). When we find one, we look at the next character to determine the type:
- `%d`: Signed decimal integer
- `%x` or `%p`: Hexadecimal integer (unsigned)
- `%s`: String
- `%%`: Literal percent sign

For each specifier, we consume the next argument from `argp` and output it appropriately. If we see an unknown specifier, we print it literally to draw attention to the error.

This is a much simpler printf than the standard C library version -- it doesn't support field widths, precision, or many other format options. But it's good enough for kernel debugging output.

```c
  if(locking)
    release(&cons.lock);
}
```

Finally, we release the console lock.

#### panic()

```c
void
panic(char *s)
{
  int i;
  uint pcs[10];

  cli();
  cons.locking = 0;
  cprintf("cpu%d: panic: ", cpuid());
  cprintf(s);
  cprintf("\n");
  getcallerpcs(&s, pcs);
  for(i=0; i<10; i++)
    cprintf(" %p", pcs[i]);
  panicked = 1; // freeze other CPU
  for(;;)
    ;
}
```

The `panic()` function is called when the kernel encounters an unrecoverable error. It disables interrupts, turns off console locking (so we can print the panic message even if we're already holding the lock), prints an error message with a stack backtrace, sets the `panicked` flag, and then loops forever.

The `getcallerpcs()` function (implemented elsewhere) walks the stack to find the return addresses of the calling functions, giving us a backtrace. This is invaluable for debugging kernel crashes.

Once a CPU panics, it never leaves this function. Other CPUs will see the `panicked` flag and also halt. The only way to recover is to reboot the system.

### CGA Text Display

Now let's look at the code that writes to the CGA (Color Graphics Adapter) text display.

```c
#define BACKSPACE 0x100
#define CRTPORT 0x3d4
static ushort *crt = (ushort*)P2V(0xb8000);  // CGA memory
```

The CGA text display has a framebuffer at physical address 0xB8000. We convert this to a kernel virtual address using P2V(). The framebuffer is an array of 16-bit values, where each value represents one character on the screen. The low 8 bits are the ASCII character code, and the high 8 bits are the attribute (color and style).

The display is 80 columns by 25 rows, for a total of 2000 characters. The CRTPORT (CRT Controller Port) at 0x3D4 is used to control the hardware cursor position.

#### cgaputc()

```c
static void
cgaputc(int c)
{
  int pos;

  // Cursor position: col + 80*row.
  outb(CRTPORT, 14);
  pos = inb(CRTPORT+1) << 8;
  outb(CRTPORT, 15);
  pos |= inb(CRTPORT+1);
```

This function writes a character to the CGA display. First, we read the current cursor position from the CRT controller. The cursor position is stored in two registers (14 and 15), which together form a 16-bit value. Register 14 contains the high 8 bits, and register 15 contains the low 8 bits.

The CRT controller uses an indirect addressing scheme: we write the register number to CRTPORT, then read or write the register value at CRTPORT+1.

```c
  if(c == '\n')
    pos += 80 - pos%80;
  else if(c == BACKSPACE){
    if(pos > 0) --pos;
  } else
    crt[pos++] = (c&0xff) | 0x0700;  // black on white
```

Now we handle the character:
- `\n` (newline): Move to the start of the next line. We calculate this as the next multiple of 80.
- BACKSPACE: Move back one position (unless we're already at position 0).
- Any other character: Write it to the framebuffer at the current position, with attribute 0x0700 (black text on white background), and advance the position.

Wait, black on white? Actually, the comment is backwards -- 0x0700 means gray (white) text on black background, not black on white. Attribute bits 0-3 are the foreground color, and bits 4-7 are the background color. 0x07 in the low nibble is light gray, and 0x00 in the high nibble is black background.

```c
  if(pos < 0 || pos > 25*80)
    panic("pos under/overflow");

  if((pos/80) >= 24){  // Scroll up.
    memmove(crt, crt+80, sizeof(crt[0])*23*80);
    pos -= 80;
    memset(crt+pos, 0, sizeof(crt[0])*(24*80 - pos));
  }
```

We check for cursor position overflow as a sanity check. Then, if we've gone past row 24 (the last row, since rows are 0-indexed and we have rows 0-24), we need to scroll the display.

Scrolling is simple: we copy rows 1-23 to rows 0-22, effectively discarding row 0 and moving everything up by one row. Then we clear the last row and move the cursor up by 80 positions (one row).

```c
  outb(CRTPORT, 14);
  outb(CRTPORT+1, pos>>8);
  outb(CRTPORT, 15);
  outb(CRTPORT+1, pos);
  crt[pos] = ' ' | 0x0700;
}
```

Finally, we update the hardware cursor position by writing to CRT controller registers 14 and 15, and we draw the cursor by writing a space character at the cursor position.

### Console Output

#### consputc()

```c
void
consputc(int c)
{
  if(panicked){
    cli();
    for(;;)
      ;
  }

  if(c == BACKSPACE){
    uartputc('\b'); uartputc(' '); uartputc('\b');
  } else
    uartputc(c);
  cgaputc(c);
}
```

This is the main console output function. It writes a character to both the serial port and the CGA display, implementing the "console as a combination of devices" concept we mentioned earlier.

If the kernel has panicked, we disable interrupts and loop forever. This prevents any further output from interfering with the panic message.

For backspace, we send a special sequence to the UART: backspace, space, backspace. This moves the cursor back, overwrites the character with a space (erasing it), and moves back again. This is necessary because the UART doesn't have a framebuffer -- it's a stream of characters.

For all other characters, we just send them to both devices.

### Console Input

Now let's look at how console input works. This is more complex because we need to handle line editing (backspace, control-U to kill the line, etc.) and we need to buffer input until a complete line is entered.

```c
#define INPUT_BUF 128
struct {
  char buf[INPUT_BUF];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} input;

#define C(x)  ((x)-'@')  // Control-x
```

The input buffer is a circular buffer that holds characters typed by the user. We maintain three indices:
- `r`: Read index -- where `consoleread()` will read from next
- `w`: Write index -- where the next complete line will end
- `e`: Edit index -- where the user is currently typing

The difference between `e` and `w` is important: as the user types, characters go at `e`, but they're not available for reading until the user presses Enter. When Enter is pressed, `w` is updated to equal `e`, making the entire line available to be read.

The `C(x)` macro converts a letter to its corresponding control character. Control characters are ASCII values 1-26, corresponding to Ctrl-A through Ctrl-Z. The ASCII value is the letter minus '@' (since '@' is ASCII 64, and 'A' is ASCII 65, so 'A' - '@' = 1).

#### consoleintr()

```c
void
consoleintr(int (*getc)(void))
{
  int c, doprocdump = 0;

  acquire(&cons.lock);
  while((c = getc()) >= 0){
```

This function is called from interrupt handlers (like `uartintr()` and the keyboard interrupt handler) to process input characters. The `getc` function pointer is device-specific: it might be `uartgetc()` for serial port input or `kbdgetc()` for keyboard input.

We acquire the console lock to protect the input buffer, then loop calling `getc()` until it returns -1 (no more characters available).

```c
    switch(c){
    case C('P'):  // Process listing.
      // procdump() locks cons.lock indirectly; invoke later
      doprocdump = 1;
      break;
    case C('U'):  // Kill line.
      while(input.e != input.w &&
            input.buf[(input.e-1) % INPUT_BUF] != '\n'){
        input.e--;
        consputc(BACKSPACE);
      }
      break;
    case C('H'): case '\x7f':  // Backspace
      if(input.e != input.w){
        input.e--;
        consputc(BACKSPACE);
      }
      break;
```

We handle several special keys:
- Ctrl-P: Print a process listing. We defer this until after we release the lock because `procdump()` needs to acquire the lock itself.
- Ctrl-U: Kill (erase) the entire line. We back up the edit index to the start of the line, outputting backspaces to erase the characters on screen.
- Ctrl-H or Delete (0x7F): Backspace one character. We back up the edit index by one and output a backspace.

Note that we only allow backspacing within the current line (we check `input.e != input.w` to make sure we don't back up past the start of the line).

```c
    default:
      if(c != 0 && input.e-input.r < INPUT_BUF){
        c = (c == '\r') ? '\n' : c;
        input.buf[input.e++ % INPUT_BUF] = c;
        consputc(c);
        if(c == '\n' || c == C('D') || input.e == input.r+INPUT_BUF){
          input.w = input.e;
          wakeup(&input.r);
        }
      }
      break;
    }
  }
  release(&cons.lock);
  if(doprocdump) {
    procdump();  // now call procdump() wo. cons.lock held
  }
}
```

For normal characters, we add them to the input buffer (if there's space), echo them back to the user by calling `consputc()`, and advance the edit index.

We convert carriage return ('\r') to newline ('\n') for consistency.

If we've received a newline, Ctrl-D (end-of-file), or filled the buffer, we update the write index to equal the edit index, making the line available to be read. Then we wake up any processes that are blocked waiting for input in `consoleread()`.

After processing all characters, we release the lock. If Ctrl-P was pressed, we call `procdump()` now that we've released the lock.

#### consoleread()

```c
int
consoleread(struct inode *ip, char *dst, int n)
{
  uint target;
  int c;

  iunlock(ip);
  target = n;
  acquire(&cons.lock);
  while(n > 0){
    while(input.r == input.w){
      if(myproc()->killed){
        release(&cons.lock);
        ilock(ip);
        return -1;
      }
      sleep(&input.r, &cons.lock);
    }
    c = input.buf[input.r++ % INPUT_BUF];
    if(c == C('D')){  // EOF
      if(n < target){
        // Save ^D for next time, to make sure
        // caller gets a 0-byte result.
        input.r--;
      }
      break;
    }
    *dst++ = c;
    --n;
    if(c == '\n')
      break;
  }
  release(&cons.lock);
  ilock(ip);

  return target - n;
}
```

This function is called when a process reads from the console (file descriptor 0). It sleeps until a complete line is available, then copies characters from the input buffer to the user's buffer.

We unlock the inode before acquiring the console lock to avoid potential deadlocks (the inode lock and console lock need to be acquired in a consistent order).

We loop, sleeping on `input.r` while no input is available (`input.r == input.w`). When we wake up, we copy characters from the input buffer to the destination, stopping at a newline or when we've read `n` bytes.

The EOF handling (Ctrl-D) is a bit tricky. If Ctrl-D is the first character we read, we return 0 bytes (EOF). But if we've already read some characters, we push the Ctrl-D back (by decrementing `input.r`) and return what we've read so far, so the Ctrl-D will be seen as EOF on the next read.

#### consolewrite()

```c
int
consolewrite(struct inode *ip, char *buf, int n)
{
  int i;

  iunlock(ip);
  acquire(&cons.lock);
  for(i = 0; i < n; i++)
    consputc(buf[i] & 0xff);
  release(&cons.lock);
  ilock(ip);

  return n;
}
```

This function is called when a process writes to the console (file descriptor 1 or 2). It's much simpler than reading: we just output each character to the console.

We acquire the console lock to prevent interleaved output from multiple processes.

### Console Initialization

#### consoleinit()

```c
void
consoleinit(void)
{
  initlock(&cons.lock, "console");

  devsw[CONSOLE].write = consolewrite;
  devsw[CONSOLE].read = consoleread;
  cons.locking = 1;

  ioapicenable(IRQ_KBD, 0);
}
```

This function initializes the console during kernel startup. It's called from `main()`.

First, we initialize the console lock. Then we set up the device switch table (`devsw[]`) to point to our console read and write functions. The device switch is how the file system knows which functions to call when someone reads or writes to a device file.

We enable console locking (it was disabled during early boot when only one CPU was running), and we enable keyboard interrupts by calling `ioapicenable()` for IRQ 1.

Note that `uartinit()` is called separately before `consoleinit()`, since we want to be able to print to the UART as early as possible during boot.

## How It All Works Together

Let's trace through what happens when you type a line of text and press Enter:

1. You press a key, which generates a keyboard interrupt (IRQ 1) or serial port interrupt (IRQ 4)
2. The interrupt handler calls `consoleintr()` with a device-specific `getc()` function
3. `consoleintr()` reads the character and adds it to the input buffer, echoing it back via `consputc()`
4. When you press Enter, `consoleintr()` updates `input.w` and wakes up any processes waiting in `consoleread()`
5. The waiting process wakes up, copies the line from the input buffer, and returns to user space
6. When the process writes output, it calls `consolewrite()`, which calls `consputc()` for each character
7. `consputc()` writes to both the UART (via `uartputc()`) and the CGA display (via `cgaputc()`)

The console code is a great example of how device drivers work in a real operating system:
- Multiple devices (UART, CGA, keyboard) are combined into a single logical device (the console)
- Interrupt-driven I/O allows the CPU to do other work while waiting for slow devices
- Buffering and line editing improve the user experience
- Careful locking prevents race conditions when multiple CPUs access the console simultaneously

## Summary

The serial port and console drivers in xv6 demonstrate several important operating system concepts:

1. **Device abstraction**: The console combines multiple physical devices into a single logical device
2. **Interrupt-driven I/O**: Characters are processed as they arrive via interrupts, not by polling
3. **Buffering**: Input is buffered in a circular buffer until a complete line is ready
4. **Synchronization**: Spinlocks protect shared data structures from concurrent access
5. **Sleep/wakeup**: Processes sleep when no input is available and are woken when input arrives

The code is fairly simple compared to a production operating system's console driver, but it implements all the essential features: output to multiple devices, input buffering with line editing, and proper synchronization for multiprocessor systems.

One interesting design choice is that xv6 always echoes input characters back to the user. Most Unix systems have a "canonical mode" where the terminal driver handles line editing and echoing, and a "raw mode" where the application sees characters immediately without echoing. xv6 only implements canonical mode, which is sufficient for simple command-line interaction but wouldn't work well for programs like text editors that need character-by-character input.

The UART code is also quite simple -- a production driver would handle more error conditions, support different baud rates and data formats, implement flow control, and possibly use DMA for higher performance. But for a teaching operating system, xv6's simple polled approach works fine and is much easier to understand.
