# Devices: Multiprocessing

Up until now, we've mostly been thinking about xv6 as if it's running on a single CPU. But one of the cool things about xv6 is that it's designed to run on a multiprocessor system -- that is, a computer with multiple CPUs that can all execute code simultaneously. This opens up a world of possibilities for parallelism and better performance, but it also introduces a whole new set of challenges when it comes to coordination and synchronization between CPUs.

Before we dive into the interrupt controllers (like the LAPIC and IOAPIC) that handle interrupts on each CPU, we need to understand how xv6 even figures out how many CPUs are available and gets them all up and running in the first place. That's what this post is all about: the multiprocessor (MP) initialization code in mp.c.

## Multiprocessor Basics

In a multiprocessor x86 system, there's typically one bootstrap processor (BSP) that starts up first when the machine boots, and then zero or more application processors (APs) that need to be explicitly started by the BSP. The BSP is the CPU that runs the boot loader and starts the kernel; the APs sit idle until they're told to wake up and start executing code.

x86 multiprocessor systems use a standard called the MultiProcessor Specification (or MP Specification) to describe the hardware configuration. This specification defines data structures in memory that tell the operating system how many CPUs there are, what their IDs are, and where the I/O APIC (the I/O interrupt controller) is located. The code in mp.c is responsible for finding and parsing these data structures during kernel initialization.

If you're wondering why we need all this complexity, remember that each CPU needs its own local interrupt controller (LAPIC) to handle interrupts, and the operating system needs to know the memory addresses of these controllers. Plus, we need to know how many CPUs there are so we can set up per-CPU data structures and make sure the scheduler can assign processes to all available CPUs.

## mp.h

Before we look at the code in mp.c, let's take a quick look at the header file mp.h, which defines the data structures we'll be working with. These data structures are defined by the MP Specification, so they have very specific layouts that match what the hardware provides.

The main structures we care about are:
- `struct mp`: The MP floating pointer structure, which points to the MP configuration table
- `struct mpconf`: The MP configuration table header
- `struct mpproc`: An entry describing a CPU
- `struct mpioapic`: An entry describing an I/O APIC

The MP floating pointer structure is how we find the rest of the MP configuration information. It's located in a specific area of memory and contains a magic signature that we can search for.

## mp.c

Now let's go through the code in mp.c. This file is all about searching memory for the MP data structures and extracting the information we need.

```c
#include "types.h"
#include "defs.h"
#include "param.h"
#include "memlayout.h"
#include "mp.h"
#include "x86.h"
#include "mmu.h"
#include "proc.h"
```

Standard include files. We'll need proc.h because we're setting up the global array of CPU structures.

```c
struct cpu cpus[NCPU];
int ncpu;
uchar ioapicid;
```

Here we define some global variables:
- `cpus[]`: An array of `struct cpu` for each processor in the system. NCPU is defined in param.h as 8, which is the maximum number of CPUs that xv6 supports.
- `ncpu`: The actual number of CPUs found in the system
- `ioapicid`: The ID of the I/O APIC

### sum()

```c
static uchar
sum(uchar *addr, int len)
{
  int i, sum;
  
  sum = 0;
  for(i = 0; i < len; i++)
    sum += addr[i];
  return sum;
}
```

This is a simple helper function that adds up all the bytes in a region of memory. The MP Specification uses checksums to verify that structures haven't been corrupted, so we'll use this function to validate the data structures we find. If the sum of all bytes in a structure equals zero (modulo 256), then the checksum is valid.

### mpsearch()

```c
static struct mp*
mpsearch(void)
{
  uchar *bda;
  uint p;
  struct mp *mp;
```

This function searches memory for the MP floating pointer structure. According to the MP Specification, this structure can be in one of three places:
1. In the first kilobyte of the Extended BIOS Data Area (EBDA)
2. In the last kilobyte of system base memory
3. In the BIOS ROM between 0xF0000 and 0xFFFFF

```c
  bda = (uchar *) P2V(0x400);
  if((p = ((bda[0x0F]<<8)| bda[0x0E]) << 4)){
    if((mp = mpsearch1(P2V(p), 1024)))
      return mp;
  } else {
    p = ((bda[0x14]<<8)|bda[0x13])*1024;
    if((mp = mpsearch1(P2V(p-1024), 1024)))
      return mp;
  }
```

The BIOS Data Area (BDA) is located at physical address 0x400 and contains various hardware configuration information. Bytes 0x0E and 0x0F contain the segment address of the EBDA. We shift left by 4 because segment addresses in real mode are shifted left by 4 bits to get the physical address.

If we find an EBDA address, we search the first 1024 bytes of it. Otherwise, bytes 0x13 and 0x14 tell us the size of base memory in kilobytes, and we search the last 1024 bytes of that.

Note the use of P2V() to convert physical addresses to kernel virtual addresses, since we're running with paging enabled.

```c
  return mpsearch1(P2V(0xF0000), 0x10000);
}
```

If we still haven't found the MP floating pointer structure, we search the BIOS ROM area between 0xF0000 and 0xFFFFF (64 KB).

### mpsearch1()

```c
static struct mp*
mpsearch1(uint a, int len)
{
  uchar *e, *p, *addr;

  addr = P2V(a);
  e = addr+len;
  for(p = addr; p < e; p += sizeof(struct mp))
    if(memcmp(p, "_MP_", 4) == 0 && sum(p, sizeof(struct mp)) == 0)
      return (struct mp*)p;
  return 0;
}
```

This helper function does the actual searching. It scans through the memory region looking for the "_MP_" signature (the magic number for the MP floating pointer structure). The MP Specification requires that this structure is aligned on a 16-byte boundary, but since `sizeof(struct mp)` is 16, incrementing by that amount automatically handles the alignment.

When we find a potential match, we verify the checksum using our `sum()` function. If the checksum is valid (sums to zero), we return a pointer to the structure; otherwise, we keep searching.

### mpconfig()

```c
static struct mpconf*
mpconfig(struct mp **pmp)
{
  struct mpconf *conf;
  struct mp *mp;

  if((mp = mpsearch()) == 0 || mp->physaddr == 0)
    return 0;
```

This function finds the MP configuration table. It starts by calling `mpsearch()` to find the MP floating pointer structure. The `physaddr` field in the MP floating pointer tells us where the actual configuration table is located. If it's zero, there's no configuration table (some systems use a default configuration instead, but xv6 doesn't support that).

```c
  conf = (struct mpconf*) P2V((uint) mp->physaddr);
  if(memcmp(conf, "PCMP", 4) != 0)
    return 0;
  if(conf->version != 1 && conf->version != 4)
    return 0;
  if(sum((uchar*)conf, conf->length) != 0)
    return 0;
  *pmp = mp;
  return conf;
}
```

We verify that the configuration table has the correct signature ("PCMP"), that it's version 1 or 4 (the only versions xv6 supports), and that its checksum is valid. The `length` field tells us how many bytes to include in the checksum calculation.

If everything checks out, we store the MP floating pointer in `*pmp` and return the configuration table pointer.

###mpinit()

```c
void
mpinit(void)
{
  uchar *p, *e;
  int ismp;
  struct mp *mp;
  struct mpconf *conf;
  struct mpproc *proc;
  struct mpioapic *ioapic;
```

This is the main initialization function, called from main() early in the kernel boot process. It discovers all the CPUs and I/O APICs in the system.

```c
  if((conf = mpconfig(&mp)) == 0)
    panic("Expect to run on an SMP");
  ismp = 1;
  lapic = (uint*)conf->lapicaddr;
```

First, we try to find the MP configuration table. If we can't find it, we panic -- xv6 requires a multiprocessor system. (In practice, even single-processor systems usually have MP tables these days.)

We set the global `ismp` flag to 1 to indicate that we're running on a multiprocessor system. Then we extract the physical address of the local APIC from the configuration table and store it in the global `lapic` variable (which is declared in lapic.c).

```c
  for(p=(uchar*)(conf+1), e=(uchar*)conf+conf->length; p<e; ){
    switch(*p){
    case MPPROC:
      proc = (struct mpproc*)p;
      if(ncpu < NCPU) {
        cpus[ncpu].apicid = proc->apicid;
        ncpu++;
      }
      p += sizeof(struct mpproc);
      continue;
```

After the configuration table header comes a variable-length list of entries describing the system's CPUs, buses, I/O APICs, and interrupt routing. Each entry starts with a type byte that tells us what kind of entry it is.

We loop through these entries, starting right after the header (`conf+1`) and continuing until we reach the end of the table (`conf+conf->length`).

For processor entries (MPPROC), we extract the APIC ID and store it in the next available slot in the `cpus[]` array. Each CPU has a unique APIC ID that's used to address its local APIC. We keep track of how many CPUs we've found in `ncpu`.

Note that we check `ncpu < NCPU` to make sure we don't overflow the array. If there are more CPUs than xv6 can handle, we just ignore the extras.

```c
    case MPIOAPIC:
      ioapic = (struct mpioapic*)p;
      ioapicid = ioapic->apicno;
      p += sizeof(struct mpioapic);
      continue;
```

For I/O APIC entries, we just save the APIC number. In practice, most systems have only one I/O APIC, so xv6's simple approach of just saving the last one it finds usually works fine.

```c
    case MPBUS:
    case MPIOINTR:
    case MPLINTR:
      p += 8;
      continue;
    default:
      ismp = 0;
      break;
    }
  }
```

For bus, I/O interrupt, and local interrupt entries, we don't need any information from them, so we just skip over them. Each of these entry types is 8 bytes long.

If we encounter an entry type we don't recognize, we set `ismp` to 0 and break out of the loop. This is a defensive programming practice -- if we see something unexpected, we assume the configuration table might be corrupt and stop parsing it.

```c
  if(!ismp)
    panic("Didn't find a suitable machine");

  if(mp->imcrp){
    // Bochs doesn't support IMCR, so this doesn't run on Bochs.
    // But it would on real hardware.
    outb(0x22, 0x70);   // Select IMCR
    outb(0x23, inb(0x23) | 1);  // Mask external interrupts.
  }
}
```

After processing all the entries, we check if `ismp` is still 1. If not, something went wrong and we panic.

Finally, if the MP floating pointer structure indicates that there's an Interrupt Mode Configuration Register (IMCR), we need to configure it to disable the legacy PIC (Programmable Interrupt Controller) and route interrupts through the APIC instead. This is done through port I/O.

The IMCR is a legacy compatibility feature that allows switching between PIC mode and APIC mode. Setting bit 0 of register 0x70 (accessed through ports 0x22 and 0x23) tells the hardware to use the APIC. Note that Bochs (an x86 emulator) doesn't support the IMCR, but real hardware often does.

## Summary

The code in mp.c is responsible for the early stages of multiprocessor initialization. It searches memory for the MP Specification data structures, extracts information about the CPUs and I/O APIC, and sets up the global `cpus[]` array that the rest of the kernel will use.

The actual startup of the application processors happens later, in a different file (entryother.S and main.c). At this point in the boot process, only the bootstrap processor is running. The AP startup code will use the information we've gathered here to wake up the other CPUs and get them executing kernel code.

There are a few things worth noting about this code:
1. It's very dependent on x86-specific hardware and the MP Specification. Modern systems use ACPI instead, but xv6 sticks with the older MP standard for simplicity.
2. The code is fairly defensive, checking signatures and checksums, but it still makes some assumptions (like panicking if it can't find MP tables).
3. xv6's approach is simple -- it just finds the CPUs and I/O APIC and ignores most of the other information in the MP tables. A production operating system would need to parse more of the interrupt routing information.

The multiprocessor support in xv6 is one of the features that makes it more realistic than many teaching operating systems. Real-world systems need to deal with multiple CPUs, and xv6 provides a good example of how to do the basic initialization. Just keep in mind that this is the simple version -- production systems have to deal with much more complexity, especially on modern hardware with features like hyperthreading and non-uniform memory access (NUMA).
