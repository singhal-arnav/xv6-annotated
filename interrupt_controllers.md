# Devices: Interrupt Controllers

We've now seen how xv6 discovers the CPUs and I/O APIC using the MP Specification data structures, and we've looked at how traps work to get us from user mode into the kernel. But there's still a missing piece: how do interrupts from devices actually reach the CPUs in the first place? And how does xv6 configure the hardware to route those interrupts to the right places?

That's what this post is about: the Local APIC (LAPIC) and I/O APIC interrupt controllers. Every CPU has its own LAPIC that handles interrupts destined for that specific processor, and the system has one I/O APIC that receives interrupts from external devices and routes them to the appropriate LAPIC. These are the hardware components that make it possible for devices like the keyboard, disk, and timer to notify the CPU when they need attention.

## APIC Architecture

Before we dive into the code, let's talk about the overall architecture. Modern x86 multiprocessor systems use what's called the Advanced Programmable Interrupt Controller (APIC) system. This replaced the older 8259 Programmable Interrupt Controller (PIC) that was used in early PC systems.

The APIC system has two main components:

1. **Local APIC (LAPIC)**: Each CPU has its own LAPIC, which is responsible for:
   - Receiving interrupts from the I/O APIC
   - Handling inter-processor interrupts (IPIs) for CPU-to-CPU communication
   - Providing a local timer for each CPU
   - Managing interrupt priority and masking

2. **I/O APIC (IOAPIC)**: The system has one (or sometimes more) I/O APIC, which:
   - Receives interrupt signals from external devices
   - Routes those interrupts to specific LAPICs based on configuration
   - Supports flexible interrupt routing (any device to any CPU)

The LAPICs and I/O APIC communicate over a dedicated interrupt bus. This architecture is much more flexible than the old PIC system, which could only route interrupts to a single CPU and had limited programmability.

Both the LAPIC and I/O APIC are controlled through memory-mapped I/O (MMIO), meaning we access their registers by reading and writing to specific physical memory addresses. This is different from port I/O, which we've seen used for devices like the serial port.

## lapic.c

Let's start with the Local APIC code. Each CPU needs to initialize and interact with its own LAPIC.

```c
#include "types.h"
#include "defs.h"
#include "date.h"
#include "memlayout.h"
#include "traps.h"
#include "mmu.h"
#include "x86.h"
#include "proc.h"
```

Standard includes. We'll need `traps.h` for the interrupt vector numbers and `proc.h` because we'll be working with per-CPU data.

### LAPIC Register Definitions

```c
// Local APIC registers, divided by 4 for use as uint[] indices.
#define ID      (0x0020/4)   // ID
#define VER     (0x0030/4)   // Version
#define TPR     (0x0080/4)   // Task Priority
#define EOI     (0x00B0/4)   // EOI
#define SVR     (0x00F0/4)   // Spurious Interrupt Vector
  #define ENABLE     0x00000100   // Unit Enable
#define ESR     (0x0280/4)   // Error Status
#define ICRLO   (0x0300/4)   // Interrupt Command
  #define INIT       0x00000500   // INIT/RESET
  #define STARTUP    0x00000600   // Startup IPI
  #define DELIVS     0x00001000   // Delivery status
  #define ASSERT     0x00004000   // Assert interrupt (vs deassert)
  #define DEASSERT   0x00000000
  #define LEVEL      0x00008000   // Level triggered
  #define BCAST      0x00080000   // Send to all APICs, including self.
  #define BUSY       0x00001000
  #define FIXED      0x00000000
#define ICRHI   (0x0310/4)   // Interrupt Command [63:32]
#define TIMER   (0x0320/4)   // Local Vector Table 0 (TIMER)
  #define X1         0x0000000B   // divide counts by 1
  #define PERIODIC   0x00020000   // Periodic
#define PCINT   (0x0340/4)   // Performance Counter LVT
#define LINT0   (0x0350/4)   // Local Vector Table 1 (LINT0)
#define LINT1   (0x0360/4)   // Local Vector Table 2 (LINT1)
#define ERROR   (0x0370/4)   // Local Vector Table 3 (ERROR)
  #define MASKED     0x00010000   // Interrupt masked
#define TICR    (0x0380/4)   // Timer Initial Count
#define TCCR    (0x0390/4)   // Timer Current Count
#define TDCR    (0x03E0/4)   // Timer Divide Configuration

volatile uint *lapic;  // Initialized in mp.c
```

These defines map out the LAPIC's registers. The LAPIC is accessed through memory-mapped I/O starting at a base address (which we discovered in mp.c and stored in the global `lapic` pointer). Each register is at a specific offset from that base address.

The offsets are divided by 4 because we're treating the LAPIC as an array of 32-bit unsigned integers. So instead of saying `*(lapic + 0x0020)` to access the ID register, we can say `lapic[ID]` where `ID` is `0x0020/4`.

There are also a bunch of bit flags defined for the various fields within these registers. For example, the Spurious Interrupt Vector Register (SVR) has an ENABLE bit at position 8 that turns the LAPIC on or off.

A few important registers to know about:
- **EOI (End Of Interrupt)**: We write to this to tell the LAPIC we're done handling an interrupt
- **ICR (Interrupt Command Register)**: Used to send inter-processor interrupts (split into low and high 32-bit parts)
- **TIMER**: Configures the local timer
- **TPR (Task Priority Register)**: Controls which interrupts are allowed through based on priority

### lapicw()

```c
static void
lapicw(int index, int value)
{
  lapic[index] = value;
  lapic[ID];  // wait for write to finish, by reading
}
```

This is a simple helper function to write to a LAPIC register. It takes an index (like `TIMER` or `EOI`) and a value to write.

The interesting part is the second line: after writing to the LAPIC, we immediately read from it (specifically from the ID register, but any register would work). Why? LAPIC writes are posted, meaning the CPU can continue executing before the write actually reaches the LAPIC hardware. Reading from the LAPIC forces the CPU to wait until all previous writes have completed. This is a memory barrier technique to ensure ordering.

This matters when we need to guarantee that a write has taken effect before continuing. For example, when sending an inter-processor interrupt, we want to make sure the interrupt command is fully written before we check whether it's been delivered.

### lapicinit()

```c
void
lapicinit(void)
{
  if(!lapic)
    return;
```

This function initializes the LAPIC for the current CPU. It's called once per CPU during kernel initialization. The first thing we do is check if `lapic` is set -- if not, we're not on a multiprocessor system and there's no LAPIC to initialize, so we just return.

```c
  // Enable local APIC; set spurious interrupt vector.
  lapicw(SVR, ENABLE | (T_IRQ0 + IRQ_SPURIOUS));
```

The first step is to enable the LAPIC by setting the ENABLE bit in the Spurious Interrupt Vector Register. We also configure what vector number to use for spurious interrupts. A spurious interrupt is a hardware quirk where the CPU might receive an interrupt signal that doesn't correspond to any real interrupt source. By giving it a specific vector number, we can handle it gracefully in our trap handler.

`T_IRQ0` is 32, the start of hardware interrupt vectors in our IDT, and `IRQ_SPURIOUS` is 31, so spurious interrupts will be vector 63.

```c
  // The timer repeatedly counts down at bus frequency
  // from lapic[TICR] and then issues an interrupt.
  // If xv6 cared more about precise timekeeping,
  // TICR would be calibrated using an external time source.
  lapicw(TDCR, X1);
  lapicw(TIMER, PERIODIC | (T_IRQ0 + IRQ_TIMER));
  lapicw(TICR, 10000000);
```

Now we set up the local timer, which is built into each LAPIC. The timer counts down from an initial value and generates an interrupt when it reaches zero. This is what drives xv6's preemptive scheduling -- each CPU gets timer interrupts at regular intervals.

First, we set the Timer Divide Configuration Register (TDCR) to X1, meaning divide by 1 (no division). The timer counts at the bus frequency divided by this value.

Then we configure the timer's entry in the Local Vector Table (LVT). We set it to PERIODIC mode (so it automatically reloads and keeps counting) and specify that timer interrupts should use vector `T_IRQ0 + IRQ_TIMER`, which is vector 32 (IRQ_TIMER is 0).

Finally, we set the Timer Initial Count Register (TICR) to 10,000,000. The timer will count down from this value to 0, then generate an interrupt and reload. This gives us timer interrupts at roughly 100 Hz on typical hardware, though the exact frequency depends on the bus speed.

```c
  // Disable logical interrupt lines.
  lapicw(LINT0, MASKED);
  lapicw(LINT1, MASKED);
```

The LAPIC has two Local Interrupt pins (LINT0 and LINT1) that can receive interrupt signals directly. These are legacy features from the old PIC days. We mask them (disable them) because xv6 uses the I/O APIC for external interrupts instead.

```c
  // Disable performance counter overflow interrupts
  // on machines that provide that interrupt entry.
  if(((lapic[VER]>>16) & 0xFF) >= 4)
    lapicw(PCINT, MASKED);
```

Newer LAPICs (version 4 and up) have a performance counter that can generate interrupts. We check the version number in the Version Register and disable performance counter interrupts if they're available. xv6 doesn't use performance counters.

```c
  // Map error interrupt to IRQ_ERROR.
  lapicw(ERROR, T_IRQ0 + IRQ_ERROR);
```

The LAPIC can generate error interrupts if something goes wrong (like an illegal register access). We configure these to use vector `T_IRQ0 + IRQ_ERROR`, which is 32 + 19 = 51.

```c
  // Clear error status register (requires back-to-back writes).
  lapicw(ESR, 0);
  lapicw(ESR, 0);
```

We clear any existing errors in the Error Status Register. The LAPIC specification requires two consecutive writes to clear the error status -- this is just a hardware quirk. The first write triggers a load of current error state into the ESR, and the second write clears it.

```c
  // Ack any outstanding interrupts.
  lapicw(EOI, 0);
```

If there are any pending interrupts from before we initialized the LAPIC, we acknowledge them by writing to the End Of Interrupt register. This clears them out so we start with a clean slate.

```c
  // Send an Init Level De-Assert to synchronise arbitration ID's.
  lapicw(ICRHI, 0);
  lapicw(ICRLO, BCAST | INIT | LEVEL);
  while(lapic[ICRLO] & DELIVS)
    ;
```

This is sending a special broadcast interrupt to synchronize all the LAPICs' arbitration IDs. In older APIC implementations, LAPICs used arbitration IDs to resolve conflicts when multiple CPUs tried to send interrupts simultaneously. Modern systems don't really need this, but it's part of the initialization sequence.

We write to both halves of the Interrupt Command Register (ICR). ICRHI specifies the destination (0 means broadcast to all), and ICRLO specifies the interrupt type and delivery mode. We send a LEVEL-triggered INIT interrupt in BCAST (broadcast) mode.

Then we wait for the delivery status bit (DELIVS) to clear, which indicates the interrupt has been sent.

```c
  // Enable interrupts on the APIC (but not on the processor).
  lapicw(TPR, 0);
}
```

Finally, we set the Task Priority Register to 0, which is the lowest priority. This means the LAPIC will accept all interrupts. Note that this doesn't enable interrupts on the processor itself -- that's controlled by the IF flag in the EFLAGS register. This just tells the LAPIC not to block any interrupts based on priority.

### lapicid()

```c
int
lapicid(void)
{
  if (!lapic)
    return 0;
  return lapic[ID] >> 24;
}
```

This function returns the APIC ID of the current CPU. Each LAPIC has a unique ID that we discovered during MP initialization. The ID is stored in the top 8 bits of the ID register, so we shift right by 24 to extract it.

If there's no LAPIC (uniprocessor system), we return 0.

### lapiceoi()

```c
void
lapiceoi(void)
{
  if(lapic)
    lapicw(EOI, 0);
}
```

This function acknowledges an interrupt by writing to the End Of Interrupt register. It's called at the end of every interrupt handler in trap.c. Writing any value to EOI tells the LAPIC that we're done handling the current interrupt and it can deliver the next one.

If there's no LAPIC, there's nothing to acknowledge, so we just skip it.

### microdelay()

```c
void
microdelay(int us)
{
}
```

This function is supposed to delay for a given number of microseconds. On real hardware, you'd implement this using the LAPIC timer or another timing source. But xv6 runs in emulators like QEMU and Bochs, which already run much slower than real hardware, so an empty function is fine. The delays in `lapicstartap()` below will still work because the emulator is slow enough.

### lapicstartap()

```c
#define CMOS_PORT    0x70
#define CMOS_RETURN  0x71

void
lapicstartap(uchar apicid, uint addr)
{
  int i;
  ushort *wrv;
```

This function starts up an application processor (AP) using the LAPIC's inter-processor interrupt (IPI) mechanism. It's called by the bootstrap processor during kernel initialization to wake up the other CPUs.

The arguments are the APIC ID of the CPU to start and the physical address where that CPU should start executing code.

```c
  // "The BSP must initialize CMOS shutdown code to 0AH
  // and the warm reset vector (DWORD based at 40:67) to point at
  // the AP startup code prior to the [universal startup algorithm]."
  outb(CMOS_PORT, 0xF);  // offset 0xF is shutdown code
  outb(CMOS_PORT+1, 0x0A);
  wrv = (ushort*)P2V((0x40<<4 | 0x67));  // Warm reset vector
  wrv[0] = 0;
  wrv[1] = addr >> 4;
```

The comment quotes from the MultiProcessor Specification, which describes the "universal startup algorithm" for waking up APs. There's some legacy PC architecture involved here.

The CMOS is a small battery-backed RAM that stores system configuration and the real-time clock. We write 0x0A to offset 0xF, which is the shutdown code. This tells the BIOS what kind of reset is happening.

The "warm reset vector" is a location in low memory (physical address 0x40:0x67, which is 0x467 in linear address) where the BIOS looks for the address to jump to on a warm reset. We need to set this to point to our AP startup code. The address is stored as a real-mode segment:offset pair, so we convert our physical address accordingly.

```c
  // "Universal startup algorithm."
  // Send INIT (level-triggered) interrupt to reset other CPU.
  lapicw(ICRHI, apicid<<24);
  lapicw(ICRLO, INIT | LEVEL | ASSERT);
  microdelay(200);
  lapicw(ICRLO, INIT | LEVEL);
  microdelay(100);    // should be 10ms, but too slow in Bochs!
```

Now we send an INIT IPI to reset the target CPU. We write the APIC ID to the high part of the ICR (shifted to bits 24-31), then send an INIT interrupt by writing to the low part.

The INIT interrupt is level-triggered, so we first assert it (set the signal high), wait a bit, then deassert it (set it low), and wait again. This resets the target CPU to a known state.

The spec says to wait 10ms after deasserting, but the comment notes that's too slow in Bochs, so xv6 only waits 100 microseconds. Again, the microdelay function is a no-op anyway.

```c
  // Send startup IPI (twice!) to enter code.
  // Regular hardware is supposed to only accept a STARTUP
  // when it is in the halted state due to an INIT.  So the second
  // should be ignored, but it is part of the official Intel algorithm.
  // Bochs complains about the second one.  Too bad for Bochs.
  for(i = 0; i < 2; i++){
    lapicw(ICRHI, apicid<<24);
    lapicw(ICRLO, STARTUP | (addr>>12));
    microdelay(200);
  }
}
```

Finally, we send two STARTUP IPIs (SIPIs). These tell the target CPU to start executing at the specified address. The address in the SIPI is divided by 4096 (shifted right by 12) because it's specified in 4 KB page units.

The Intel spec says to send the SIPI twice because the first one might get lost on some hardware. Real hardware should ignore the second one if the first succeeded, but Bochs complains about it -- hence the apologetic comment.

After this function completes, the AP should start executing the code at `addr`, which in xv6 is the `entryother` code.

### cmostime()

```c
// Additional helper function to read time from CMOS RTC
// (not shown here for brevity, but exists in the full code)
```

The full lapic.c also includes a `cmostime()` function that reads the current date and time from the CMOS real-time clock. We won't go through it in detail since it's mostly just tedious port I/O to read the RTC registers, but it's used by the `date` system call.

## ioapic.c

Now let's look at the I/O APIC code, which handles routing external device interrupts to the CPUs.

```c
#include "types.h"
#include "defs.h"
#include "traps.h"

#define IOAPIC  0xFEC00000   // Default physical address of IO APIC
```

The I/O APIC lives at a fixed physical address, 0xFEC00000. This is a standard address defined by the hardware.

### Register Definitions

```c
#define REG_ID     0x00  // Register index: ID
#define REG_VER    0x01  // Register index: version
#define REG_TABLE  0x10  // Redirection table base

// The redirection table starts at REG_TABLE and uses
// two registers to configure each interrupt.
// The first (low) register in a pair contains configuration bits.
// The second (high) register contains a bitmask telling which
// CPUs can serve that interrupt.
#define INT_DISABLED   0x00010000  // Interrupt disabled
#define INT_LEVEL      0x00008000  // Level-triggered (vs edge-)
#define INT_ACTIVELOW  0x00002000  // Active low (vs high)
#define INT_LOGICAL    0x00000800  // Destination is CPU id (vs APIC ID)

volatile struct ioapic *ioapic;
```

These defines map out the I/O APIC's registers. Unlike the LAPIC, the I/O APIC is accessed through an indirect addressing scheme: you write the register index to one location, then read or write the data at another location.

The redirection table is the key data structure. It's a table where each entry configures how one external interrupt (IRQ) should be routed. Each entry is 64 bits wide (two 32-bit registers). The low register contains configuration bits like whether the interrupt is level or edge-triggered, active high or low, enabled or disabled, etc. The high register specifies which CPU(s) should receive the interrupt.

### I/O APIC Structure

```c
struct ioapic {
  uint reg;
  uint pad[3];
  uint data;
};
```

This structure represents the memory-mapped layout of the I/O APIC. To access a register, you write its index to the `reg` field, then read or write the actual data in the `data` field. The `pad` array represents reserved space between these two fields in the hardware.

### ioapicread() and ioapicwrite()

```c
static uint
ioapicread(int reg)
{
  ioapic->reg = reg;
  return ioapic->data;
}

static void
ioapicwrite(int reg, uint data)
{
  ioapic->reg = reg;
  ioapic->data = data;
}
```

These helper functions implement the indirect addressing scheme. To read a register, we write its index to `ioapic->reg`, then read the value from `ioapic->data`. To write a register, we write the index to `ioapic->reg`, then write the value to `ioapic->data`.

### ioapicinit()

```c
void
ioapicinit(void)
{
  int i, id, maxintr;

  ioapic = (volatile struct ioapic*)IOAPIC;
  maxintr = (ioapicread(REG_VER) >> 16) & 0xFF;
  id = ioapicread(REG_ID) >> 24;
  if(id != ioapicid)
    cprintf("ioapicinit: id isn't equal to ioapicid; not a MP\n");
```

This function initializes the I/O APIC. First, we set up the `ioapic` pointer to point to the I/O APIC's memory-mapped address (converted to a kernel virtual address).

We read the Version Register to find out how many interrupt entries the I/O APIC supports. The maximum number of redirectable interrupts is stored in bits 16-23 of the version register. Most I/O APICs support 24 interrupts (IRQ 0-23).

We also read the I/O APIC's ID and verify it matches what we found during MP initialization. If it doesn't match, something is wrong -- maybe we're not actually on a multiprocessor system.

```c
  // Mark all interrupts edge-triggered, active high, disabled,
  // and not routed to any CPUs.
  for(i = 0; i <= maxintr; i++){
    ioapicwrite(REG_TABLE+2*i, INT_DISABLED | (T_IRQ0 + i));
    ioapicwrite(REG_TABLE+2*i+1, 0);
  }
}
```

Now we initialize the redirection table. We loop through all the interrupt entries and disable them all. For each interrupt `i`, we write to two registers:

1. `REG_TABLE+2*i`: The low 32 bits of the entry. We set it to disabled (`INT_DISABLED`) and specify the vector number as `T_IRQ0 + i`. This means IRQ 0 will use vector 32, IRQ 1 will use vector 33, and so on.

2. `REG_TABLE+2*i+1`: The high 32 bits of the entry. We set it to 0, meaning don't route it to any CPU.

By default, interrupts are edge-triggered and active-high (these are the defaults when those bits are 0). Edge-triggered means the interrupt fires on a rising edge of the signal; level-triggered means it fires as long as the signal is high.

### ioapicenable()

```c
void
ioapicenable(int irq, int cpunum)
{
  // Mark interrupt edge-triggered, active high,
  // enabled, and routed to the given cpunum,
  // which happens to be that cpu's APIC ID.
  ioapicwrite(REG_TABLE+2*irq, T_IRQ0 + irq);
  ioapicwrite(REG_TABLE+2*irq+1, cpunum << 24);
}
```

This function enables a specific IRQ and routes it to a specific CPU. It's called by device drivers during initialization. For example, the disk driver calls `ioapicenable(IRQ_IDE, ncpu-1)` to route disk interrupts to the last CPU.

We write to the two registers for the given IRQ:
1. The low register: We set the vector number to `T_IRQ0 + irq` and leave out the `INT_DISABLED` flag, which enables the interrupt.
2. The high register: We write the CPU number (which is also the APIC ID) in bits 24-31, specifying which CPU should receive this interrupt.

Note that we're using physical destination mode (because we didn't set `INT_LOGICAL`), so the CPU number must match the CPU's APIC ID.

## How It All Fits Together

Let's trace through what happens when a device interrupt occurs:

1. A device (like the disk or keyboard) signals an interrupt on one of its IRQ lines
2. The I/O APIC receives the interrupt signal
3. The I/O APIC looks up the IRQ in its redirection table to find the configuration
4. If the interrupt is enabled, the I/O APIC sends an interrupt message to the appropriate LAPIC over the interrupt bus
5. The LAPIC receives the interrupt and checks if it should accept it (based on priority and masking)
6. If accepted, the LAPIC delivers the interrupt to its CPU by asserting the interrupt line
7. The CPU looks up the vector number in the IDT and jumps to the appropriate handler
8. The handler does its work, then writes to the LAPIC's EOI register to signal completion
9. The LAPIC can now deliver the next interrupt

The LAPIC timer works a bit differently since it's internal to the LAPIC. It just counts down and generates an interrupt locally without involving the I/O APIC.

## Summary

The LAPIC and I/O APIC are the hardware components that make interrupts work on multiprocessor x86 systems. Each CPU has a LAPIC that receives and manages interrupts for that CPU, and the I/O APIC routes external device interrupts to the appropriate LAPIC.

Key takeaways:
- The LAPIC and I/O APIC are controlled through memory-mapped I/O
- Each CPU must initialize its own LAPIC during kernel startup
- The LAPIC provides a local timer that drives preemptive scheduling
- The I/O APIC's redirection table configures how external interrupts are routed
- Inter-processor interrupts (IPIs) are sent through the LAPIC's ICR
- Every interrupt handler must acknowledge the interrupt by writing to the LAPIC's EOI register

The code in lapic.c and ioapic.c is fairly low-level and hardware-specific, but it provides the foundation for all interrupt handling in xv6. Without these interrupt controllers properly configured, devices wouldn't be able to notify the kernel when they need attention, and the kernel wouldn't be able to preempt processes or communicate between CPUs.

One thing worth noting: this is all specific to x86 multiprocessor systems. Modern systems have moved beyond this architecture -- for example, x86-64 systems use the x2APIC, and ARM systems use entirely different interrupt controllers like the GIC. But the basic concepts (local interrupt controllers per CPU, routing external interrupts to CPUs, inter-processor interrupts) remain the same.
