# RISC-V SBI specification

## Table of Contents
1. [Introduction](#Introduction)
2. [SBI List](#sbi-list)
3. [Types](#Types)
4. [Description](#Description)
	* [Timer](#Timer)
	* [Inter Processor Interrupt (IPI)](#IPI)
	* [Memory Model](#memory-model)
	* [Console](#Console)
	* [Shutdown](#Shutdown)
5. [Conclusion](#Conclusion)


## Copyright and license information

This RISC-V Supervisor Binary Interface (SBI) specification document is

 &copy; 2018 Atish Patra <atish.patra@wdc.com>
 &copy; 2018 Andrew Waterman <aswaterman@gmail.com>
 &copy; 2018 Palmer Dabbelt <palmer@dabbelt.com>

It is licensed under the Creative Commons Attribution 4.0 International
License (CC-BY 4.0).  The full license text is available at
https://creativecommons.org/licenses/by/4.0/.

## Introduction<a name="Introduction" />

Supervisor Binary Interface (SBI) describes the instructions that can be executed
together with a set of SBI calls out to the supervisor execution environment (SEE)
on a given platform. Several instructions that might normally be handled by the
supervisor operating system (OS) directly are handled via SBI calls to the SEE in
RISC-V. This provides a cleaner interface for the supervisor OS. Currently, SBI
is implemented by the Berkeley Boot Loader (BBL). The next section
describes different SBI calls defined for RISC-V.

## SBI List<a name="sbi-list" />
There are mainly 5 SBI calls defined in the specification as per
the table below.

| Type          | Function          | Function ID |
|:--------------|:------------------|-------------|
| Timer         | sbi_set_timer     |0		  |
| IPI           | sbi_clear_ipi<br>sbi_send_ipi  | 3<br>4|
| Memory Model| sbi_remote_fence_i<br>sbi_remote_sfence_vma<br>sbi_remote_sfence_vma_asid | 5<br>6<br>7 |
|	Console				| sbi_console_putchar <br> sbi_console_getchar | 1<br>2 |
| Shutdown         |	sbi_shutdown | 8 |

## Types<a name="Types" />
Most types are defined through standards such as C99's `stdbool.h`, `stdint.h`, etc.
SBI-specific types are defined here.

## Description<a name="Description" />
Each SBI call adheres to a specific calling convention that defines how data is
provided as input to the call or returned as output from the call.
The SBI calling convention matches the
syscall calling convention (which in turn is very similar to the function calling
convention). Every SBI call uses the *ecall* instruction to make a request to the SEE
environment. When executed in S-mode, it generates an environment-call-from-S-mode
exception and performs no other operation. The required arguments are passed
through registers *a0-a2* and the SBI call type is passed via register *a7*. Once
the machine mode receives the trap, it identifies the type of SBI call from the *a7*
register and performs the required operation. Any unsupported SBI call should return error code *-38*
(which is the value of -ENOSYS in Linux) to indicate the supervisor mode that it is not supported.
All SBI calls should be assumed to clobber a0 i.e. any return value will be passed
through register *a0*. Individual SBI call signatures and their purpose are described next.

### Timer<a name="Timer" />
```C
void sbi_set_timer(uint64_t stime_value)
```
Programs the clock for next event after *stime_value* time. This function also
clears the pending timer interrupt bit.

If the supervisor wishes to clear the timer interrupt without scheduling the next
timer event, it can either request a timer interrupt infinitely far into the
future (i.e., (uint64_t)-1), or it can instead mask the timer interrupt by
clearing sie.STIE.

### IPI<a name="IPI" />
```C
void sbi_send_ipi(const unsigned long *hart_mask)
```
Send an inter-processor interrupt to all the harts defined in hart_mask.
Interprocessor interrupts manifest at the receiving harts as Supervisor Software
Interrupts.

hart_mask is a physical address that points to a bit-vector of harts. The bit
vector is represented as a sequence of unsigned longs whose length equals the
number of harts in the system divided by the number of bits in an unsigned long,
rounded up to the next integer.

If hart_mask is NULL, an IPI is sent to every hart in the system.

```C
void sbi_clear_ipi(void)
```
Clears the pending IPIs if any. The IPI is cleared only in the hart for which
this SBI call is invoked.

### Memory Model<a name="memory-model" />
```C
void sbi_remote_fence_i(const unsigned long *hart_mask)
```
Instruct remote harts to execute the FENCE.I instruction.
N.B. hart_mask is as described in sbi_send_ipi.
```C
void sbi_remote_sfence_vma(const unsigned long *hart_mask,
                           unsigned long start,
                           unsigned long size)
```
//TODO

```C
void sbi_remote_sfence_vma_asid(const unsigned long *hart_mask,
                                unsigned long start,
                                unsigned long size,
                                unsigned long asid)
```
//TODO

### Console<a name="Console" />

```C
int sbi_console_getchar(void)
```
Read a byte from the debug console; returns the byte on success, or -1 for failure.
Note: This is the only SBI call that has a non-void return type.

```C
void sbi_console_putchar(int ch)
```
Write data present in *ch* to the debug console.

Unlike `sbi_console_getchar`, this SBI call **will block** if there
remain any pending characters to be transmitted or if the receiving terminal
is not yet ready to receive the byte. However, if the console doesn't exist
at all, then the character is thrown away.

### Shutdown<a name="Shutdown" />
```C
void sbi_shutdown(void)
```
Puts all the harts in shut down state from the supervisor point of view. This SBI
call doesn't return.

## Conclusion<a name="Conclusion" />
The objective of defining the SBI at this point is to prevent any further changes
unless absolutely necessary. Thus, we can say that it is mostly a permanent
interface. This also allows OS developers to develop supervisor code without
worrying about modifying their code in the future due to changes in the SBI.
