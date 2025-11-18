# Devices: Keyboard Driver (optional)

We've seen how the serial port and console handle input and output, but there's one more input device we need to talk about: the keyboard. Thankfully, the keyboard driver is one of the simplest device drivers in xv6, so this'll be a quick one. If you've made it through the console and serial port posts, the keyboard will be a breeze.

The keyboard driver lives in two files: kbd.h (which is mostly just data tables) and kbd.c (which has two tiny functions). Together they handle reading input from the PS/2 keyboard controller, which is the standard keyboard interface on x86 machines (well, it was standard back in our 1995-era world, anyway).

## How the Keyboard Works

When you press a key on your keyboard, the keyboard controller (an 8042 chip on x86 systems, or an emulated one in QEMU) generates a hardware interrupt. The controller has two I/O ports that xv6 uses to communicate with it:

- **Port 0x64**: The status port, which we can read to see if there's data waiting for us
- **Port 0x60**: The data port, where we can read the actual scan code that the keyboard sent

Each key press generates a scan code (a number identifying which physical key was pressed), and each key release generates another scan code (the same number, but with the high bit set). So if you press and hold the 'a' key for a second then release it, the keyboard will send three scan codes: one for the initial press, a bunch more for the repeated key (because of key repeat), and one final scan code when you release it.

The job of the keyboard driver is to:
1. Read those scan codes from port 0x60
2. Figure out which key was pressed or released
3. Keep track of modifier keys (Shift, Ctrl, Alt, Caps Lock, etc.)
4. Convert the scan code plus modifiers into an ASCII character
5. Pass that character to the console code

## kbd.h: Scan Code Translation Tables

Let's start with kbd.h, which defines the constants and lookup tables the driver needs. First up, we have the port addresses and some bit flags:

```c
#define KBSTATP         0x64    // kbd controller status port(I)
#define KBS_DIB         0x01    // kbd data in buffer
#define KBDATAP         0x60    // kbd data port(I)
```

`KBSTATP` (0x64) is the status port; when we read from it, we can check if the `KBS_DIB` (keyboard data in buffer) bit is set to see if there's a scan code waiting for us. `KBDATAP` (0x60) is where we read the actual scan code from.

Next, we have a bunch of bit flags for the modifier keys:

```c
#define NO              0

#define SHIFT           (1<<0)
#define CTL             (1<<1)
#define ALT             (1<<2)

#define CAPSLOCK        (1<<3)
#define NUMLOCK         (1<<4)
#define SCROLLLOCK      (1<<5)

#define E0ESC           (1<<6)
```

These are used as bit flags in a variable that keeps track of which modifiers are currently active. The `E0ESC` flag is special: some extended keys (like arrow keys) send a two-byte sequence starting with 0xE0, so we use this flag to remember that we just saw an 0xE0 byte.

There are also some special key codes for keys that don't have ASCII equivalents:

```c
#define KEY_HOME        0xE0
#define KEY_END         0xE1
#define KEY_UP          0xE2
#define KEY_DN          0xE3
#define KEY_LF          0xE4
#define KEY_RT          0xE5
#define KEY_PGUP        0xE6
#define KEY_PGDN        0xE7
#define KEY_INS         0xE8
#define KEY_DEL         0xE9
```

Then there's a handy macro for Control key combinations:

```c
#define C(x) (x - '@')
```

So `C('A')` gives you Control-A (which is ASCII value 1). This works because '@' is ASCII 64, and 'A' is ASCII 65, so 'A' - '@' = 1.

Now comes the tedious part: lookup tables to convert scan codes to characters. There are separate tables for:

- `shiftcode[]`: Which scan codes represent Shift, Ctrl, and Alt keys
- `togglecode[]`: Which scan codes represent Caps Lock, Num Lock, and Scroll Lock
- `normalmap[]`: What character each scan code produces normally
- `shiftmap[]`: What character each scan code produces with Shift held
- `ctlmap[]`: What character each scan code produces with Ctrl held

These tables are initialized using C's designated initializer syntax, which lets us set specific array indices. For example:

```c
static uchar shiftcode[256] =
{
  [0x1D] CTL,
  [0x2A] SHIFT,
  [0x36] SHIFT,
  [0x38] ALT,
  [0x9D] CTL,
  [0xB8] ALT
};
```

Scan code 0x1D is the left Control key, 0x2A is the left Shift key, 0x36 is the right Shift key, and so on. Any scan codes not explicitly listed default to zero.

The `normalmap[]`, `shiftmap[]`, and `ctlmap[]` tables are huge (256 entries each), mapping every possible scan code to its corresponding character. For example, scan code 0x02 is the '1' key, which produces '1' normally, '!' with Shift, and NO (0) with Ctrl. I won't paste all 256 entries here, but you can look them up in kbd.h if you're curious about a specific key.

## kbd.c: Reading the Keyboard

Now let's look at the actual driver code. There are only two functions: `kbdgetc()` reads a character from the keyboard (if one is available), and `kbdintr()` handles keyboard interrupts.

### kbdgetc()

This function reads from the keyboard controller and returns either an ASCII character (if a key was pressed), 0 (if a modifier key or key release occurred), or -1 (if no data is available).

First, we need some static variables to keep track of the modifier state across function calls:

```c
int
kbdgetc(void)
{
  static uint shift;
  static uchar *charcode[4] = {
    normalmap, shiftmap, ctlmap, ctlmap
  };
  uint st, data, c;
```

The `shift` variable is a bit field tracking which modifiers are currently pressed (using the bit flags we saw earlier). The `charcode[]` array is a clever trick: it's an array of pointers to the different character maps, indexed by the shift state. If no modifiers are pressed, we use `normalmap` (index 0); if Shift is pressed, we use `shiftmap` (index 1); if Ctrl is pressed (bit 1 set), we use `ctlmap` (index 2 or 3).

Now we check if there's data available:

```c
  st = inb(KBSTATP);
  if((st & KBS_DIB) == 0)
    return -1;
  data = inb(KBDATAP);
```

We read the status port and check the `KBS_DIB` bit. If it's not set, there's no data waiting, so we return -1. Otherwise, we read the scan code from the data port.

Next, we handle the special 0xE0 escape code:

```c
  if(data == 0xE0){
    shift |= E0ESC;
    return 0;
  } else if(data & 0x80){
    // Key released
    data = (shift & E0ESC ? data : data & 0x7F);
    shift &= ~(shiftcode[data] | E0ESC);
    return 0;
  } else if(shift & E0ESC){
    // Last character was an E0 escape; or with 0x80
    data |= 0x80;
    shift &= ~E0ESC;
  }
```

If we see 0xE0, we set the `E0ESC` flag and return 0 (not an actual character). If the high bit is set (data & 0x80), it's a key release, so we clear the corresponding modifier flag and return 0. If the `E0ESC` flag is set from a previous call, we OR the scan code with 0x80 to convert it to the extended key range and clear the `E0ESC` flag.

Now we update the modifier state and look up the character:

```c
  shift |= shiftcode[data];
  shift ^= togglecode[data];
  c = charcode[shift & (CTL | SHIFT)][data];
```

We use `shiftcode[data]` to see if this scan code is a modifier key (Shift, Ctrl, or Alt) and update the `shift` variable. We XOR with `togglecode[data]` to handle toggle keys like Caps Lock (pressing it once turns it on, pressing it again turns it off). Then we look up the character in the appropriate character map.

Finally, we handle Caps Lock for letter keys:

```c
  if(shift & CAPSLOCK){
    if('a' <= c && c <= 'z')
      c += 'A' - 'a';
    else if('A' <= c && c <= 'Z')
      c += 'a' - 'A';
  }
  return c;
}
```

If Caps Lock is on, we swap the case of letter keys (lowercase becomes uppercase and vice versa). Then we return the final character.

### kbdintr()

The keyboard interrupt handler is refreshingly simple:

```c
void
kbdintr(void)
{
  consoleintr(kbdgetc);
}
```

That's it! When a keyboard interrupt occurs, we just call `consoleintr()` (which we saw in the console driver post) and pass it a pointer to `kbdgetc()`. The console code will call `kbdgetc()` to read characters from the keyboard until there are no more available, then process them (handling backspace, Control-C, etc.) and add them to the console's input buffer.

This design is elegant: the keyboard driver doesn't need to know anything about what the console does with the characters. It just provides a simple interface (`kbdgetc()`) for reading characters, and the console driver uses that interface the exact same way it uses `uartgetc()` for the serial port.

## Initialization

Where does the keyboard interrupt handler get registered? Back in trap.c, when we set up the IDT, we saw this line:

```c
ioapicenable(IRQ_KBD, 0);
```

That enables keyboard interrupts on CPU 0. The interrupt vector (trap number 32 + IRQ_KBD, which is trap number 33) is already set up to call the appropriate handler, which will eventually call `kbdintr()`.

## Summary

The keyboard driver is one of the simplest device drivers in xv6:

- **kbd.h** defines I/O port addresses, bit flags for modifiers, and lookup tables to translate scan codes to ASCII characters
- **kbdgetc()** reads scan codes from the keyboard controller, tracks modifier key state, and returns ASCII characters
- **kbdintr()** handles keyboard interrupts by calling `consoleintr(kbdgetc)`
- The console driver processes the characters and adds them to the input buffer

The beauty of this design is its simplicity. The keyboard driver doesn't need to deal with complex buffering, line editing, or special key handling—it just converts hardware scan codes to ASCII characters and lets the console driver handle the rest. This separation of concerns makes the code easier to understand and maintain.

If you've followed along with the other device driver posts, you'll notice a pattern: each device (serial port, console, keyboard) provides a simple `getc()` function for reading input, and the console code uses all of them through a common interface. This is good software engineering: a simple, consistent interface that hides the messy hardware details.

Now we've covered all the input devices in xv6. The keyboard driver completes our tour of how xv6 handles user input, from the hardware level (reading I/O ports and handling interrupts) up to the system call level (reading from file descriptors). Next up, we'll start looking at how xv6 manages persistent storage on disk!
