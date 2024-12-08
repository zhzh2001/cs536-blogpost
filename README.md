# Background and Problem Overview

Background: The emergency of user-space runtime systems and
kernel-bypass:

-   Kernel-bypass libraries including DPDK and SDPK move IO handling
    into user space.

-   The accelerator demands of the new-era of computation directs modern
    CPUs to provide functionalities that user space applications can
    directly leverage, such as the `UMONITOR` and `UMWAIT` instructions.

With the rise of language-level concurrency and kernel bypass
interfaces, user-level runtimes are increasingly responsible for tasks
that once belonged to the OS kernel such as preemption, synchronization,
and scheduling (CPU and IO). User-level concurrency such as goroutines
(Go), coroutines (C++), and user level threads, allow data center
applications to scale to millions of concurrent
requests and support microsecond
timescales. They do so by
avoiding the high cost of kernel threads and shift scheduling to
user-level runtimes. Similarly, kernel-bypass interfaces such as
DPDK avoid OS overheads and shift IO scheduling
to user level, enabling high performance I/O at microsecond timescales.

However, user-space runtime systems lack a key support --- fine-grained
interrupts. Interrupts deliver low-latency asynchronous notifications
from devices, cores (IPIs), and timers. Asynchronous notification
**minimizes latency**, as it immediately changes the control flow on the
receiving core, and **maximizes efficiency**, because it doesn't require
the receiver to waste cycles polling for notifications.

Without interrupt support, runtime systems must choose between slow OS
kernel interfaces or inefficient memory polling - a limitation that
critically impacts **preemption**. While preemption is essential for
preventing head-of-line blocking in datacenter workloads and safely
running untrusted code in serverless platforms, many user-level runtimes
either don't support it or limit its frequency (e.g., Go's 10ms
intervals due to expensive kernel-based signals). Beyond preemption,
popular **kernel bypass interfaces** including DPDK and SPDK must rely
on polling for microsecond-scale notifications, wasting CPU cycles and
energy as cores busy-spin. The growing interest in specialized
**accelerators** (e.g., DSA) will make
this even more challenging.

The new **user interrupts** offers a promising starting point for
bringing interrupt support to user space, which allow interrupts are
directly delivered into user space without OS kernel involved. However,
they still face significant performance and functionality limitations:

-   Our measurements show that an interrupt incurs an average cost of
    720 cycles on the receiver side, significantly constraining the
    granularity at which applications running on the receiver side can
    be preempted.

-   Sending an interrupt is also expensive at 383 cycles, forcing
    user-space runtimes to dedicate an entire core for sending
    preemption interrupts (cite Libpreemptible).

-   User interrupts are limited to thread-to-thread communication,
    excluding hardware devices. This prevents kernel bypass libraries
    like DPDK and SPDK from utilizing interrupts for I/O notifications.

Fortunately, these limitations on performance and functionality could be
addressed by a set of processor extensions.

Based on the processor extensions, we plan to develop a user-space
runtime system leveraging the extended user interrupts:

-   The runtime system will be preemptive and support fine-grained
    preemption, enabled by the expected low receiver cost of extended
    user interrupts.

-   It will utilize the proposed kernel-bypass timer for improved
    efficiency.

-   The runtime system leverages DPDK and supports receiving interrupts
    from the Network Interface Controller (NIC) for incoming packet
    notifications, eliminating the need for polling.

-   It will include a suite of preemptive synchronization primitives.
    Compared to existing primitives, these preemptive primitives will
    enable precise control over runnable threads.

## We want to talk about the workload now

# Related Work

Optimizing network and I/O workloads at microsecond timescales is a
critical challenge for modern data centers, where achieving low latency
and high throughput is essential for supporting latency-sensitive
applications. Prior works have explored user-space scheduling frameworks
and hardware-software co-design to address these challenges,
particularly focusing on mitigating interference and improving CPU
efficiency.

Fried et al. proposed Caladan, a system that extends Shenango's IOKernel
framework to optimize CPU core allocation for microsecond-scale
workloads. Unlike traditional approaches, Caladan separates
packet processing from the IOKernel, enabling direct packet delivery to
application cores without involving the IOKernel for routine operations.
This separation reduces overheads associated with packet copying and NIC
flow table updates, resulting in improved latency and CPU efficiency. A
key feature of Caladan is its congestion detection mechanism, which
dynamically reallocates CPU cores by monitoring queue backlogs. This
mechanism minimizes head-of-line blocking and ensures balanced resource
utilization. However, scalability remains a challenge, as the IOKernel
must manage a growing number of fast-changing application queues,
potentially becoming a bottleneck in large-scale deployments.

Gupta introduced Hermes, a system that focuses on balancing CPU
efficiency and responsiveness in high-performance networking
environments. Hermes leverages the Ensō NIC interface, which
introduces a streaming abstraction that decouples the control plane
(notifications) from the data plane (packet data). By decoupling these
planes, Hermes enables schedulers to efficiently detect congestion and
make informed resource allocation decisions without the overhead of
frequent data copying or NIC flow table updates. Hermes's ability to
analyze notification patterns from the control plane allows for
fine-grained congestion control and optimized core allocation. This
approach significantly improves CPU utilization while maintaining low
tail latency, making it well-suited for workloads that require both
efficiency and responsiveness.

Both works offer complementary approaches to addressing the challenges
of network and I/O workloads. While Caladan emphasizes minimizing
interference through efficient core allocation and separation of data
and control paths, Hermes leverages hardware-level abstractions to
streamline communication and enable responsive scheduling. Together,
these works provide a foundation for designing systems that achieve
low-latency, high-throughput performance in modern data centers.

# Characterization of User Interrupts

xUI builds on the existing UIPI support in Intel processors. Here, we
provide the first in-depth analysis of this feature. This also provides
context for our discussion of xUI in
section 4.

## Overview of User Interrupts

User interprocessor interrupts (UIPI) was recently introduced by Intel
in their Sapphire Rapids processors.[^1] UIPI allows user-level code
(kernel threads) to send and receive IPIs, avoiding the context-switch
overheads of signals.

There were several challenges to enabling user-level IPIs on the
existing hardware:

**Adding Access Control:** Access control is necessary to prevent
threads from sending IPIs to other non-consenting threads. UIPI
delegates this task to the kernel by introducing a hardware interface
(using MSRs and data-structures). The is a table managed by the kernel
that controls which threads a process can send to. Each entry consists
of a tuple $\langle UPID, vector\rangle$, that specifies a destination
(explained below). The kernel allows authorized processes to set up
these mappings with system calls.

**Routing interrupts at User Level:** The interrupt system is
responsible for routing messages from sources (devices, cores, timers)
to destinations. The destinations are cores (addressed by APICID), and
the messages are (on x86) 8-bit values---vectors---that uniquely
identifies the cause of the interrupt. This system is by design quite
static, APICIDs are typically assigned to cores at startup time, and
rarely change.

Thus, to enable user level interrupts, there are several challenges: (1)
there is no notion of addressing threads---which can come and go and
migrate between cores. (2) the x86 vector space is quite small (256) per
core (APICID), and some are used for other purposes---so sharing this
space with user IPI's would be quite limiting. (3) cores are always
there, threads are not; when an interrupt arrives, the destination
thread may be context switched out.

To solve this, UIPI gives threads their own orthogonal virtual name
space (thread ids) and vector space (a 6-bit user vector (UV)). However,
this virtual name space is not globally routable, thus devices cannot
address threads -- we discuss solutions for this with xUI in
(§ 4.3).

**Sending and Receiving interrupts:** When a core sends an interrupt, it
looks up the UPID for the destination and modifies it in memory---then
sends an IPI. What it modifies is the Posted Interrupt Requests (PIR)
field of the UPID, a 64-bit field, with one bit for each vector. With
one UPID per thread, the receiver core can track which threads require a
notification based on the information provided in the UPID.

With normal hardware interrupts, the receiver is always the kernel, and
thus always available. However, with user interrupts, the receiving
thread may have been context switched out. Thus, there are two delivery
paths, the ---where hardware can directly invoke the the user level
interrupt handler, and the where the kernel handles the interrupt if the
thread being addressed is not currently running. To determine which path
to take, the hardware examines the ON bit on the UPID for the current
task. If it receives an interrupt, but this has not been set, it knows
to take the slow path. On the slow path, when the kernel resumes the
thread an interrupt was destined for, it will repost the captured UIPI
as a self UIPI through the local APIC.

Finally, a receiver thread might migrate from one physical core to
another. Therefore, the sender needs to determine the correct physical
core to which it should send the UPID. To solve this, UIPI incorporates
a field in the UPID that represents the APIC ID of the core where the
receiving thread runs. When a migration occurs, the OS updates this
field. The sender process, for each UIPI that it sends, consults the
UPID of the receiver to identify the correct physical destination.

## Details of Sending and Receiving a UIPI.

![**Architecture view of UIPI:** *UIPI is largely implemented in
microcode. The sender communicates the UIPI vector through shared memory
(UPID). Steps described in
§ 3.2*](figures/fig_uipi_end_to_end.pdf)

Before an interrupt is sent, the application sets up a route for the
interrupt using system calls. allocates the UPID and
takes a pointer to the user-level interrupt handler,
`register_sender(...)` allocates an entry in the UITT. The sender can
then use to post an interrupt.
Figure 1
illustrates the end-to-end process of sending and receiving a posted
UIPI:

() The sending core looks up the receiver's UPID in the UITT, updates
the UPID's (PIR) to reflect the vector being sent, and sets it's
outstanding interrupt (ON) bit, indicating an interrupt is pending. (),
The sending core reads the APICID and vector of the receiving core from
the UPID, and writes it to interrupt command register (ICR). This causes
the local APIC to send an interrupt to the receiving core. () The IOAPIC
sends the interrupt message over the system bus to the local APIC on the
receiver. The local APIC notifies the core by raising an interrupt
signal line. () The receiving core issues a full pipeline
flush and starts executing a microcode procedure called *notification
processing* that reads the user interrupt vector from the UPID of the
current thread, saves it in a dedicated register (UIRR), and clears the
outstanding notification bit in the current threads UPID. () The core
executes the *user interrupt delivery* microcode procedure. It pushes
the stack pointer, PC, and user vector onto the stack. It clears the
user interrupt flag (UIF) disabling interrupt delivery for the handler.
It updates the UIRR to show the interrupt has been processed. And
finally, it jumps to the user-level interrupt handler specified in the
register. () The handler executes. () The handler executes a , popping
the stack pointer and PC from the stack, and sets the user interrupt
flag (UIF), re-enabling interrupt delivery, and resuming normal control
flow on the receiver.

## UIPI Performance

This table summarizes the key performance metrics of
UIPI. The setup for measuring these is described in
§ 5.

| End-to-End Latency | Receiver Cost | `SENUIPI` | `CLUI` | `STUI` |
|---------------------|---------------|-----------|--------|--------|
| 1360 cycles | 720 cycles | 383 cycles | 2 cycles | 32 cycles |

# Extensions to User Interrupts

The design of xUI is inspired by our own frustration with the
limitations of existing user-level notification support in software
dataplanes and demanding serverless settings. Thus, we aim to design
extensions that would support a diverse set of uses, and also be
practical to implement in existing hardware. xUI attempts to meet the
following design goals:

-   faster is better, notification is a basic building block for IO,
    synchronization, etc. xUI aims to offer performance par with shared
    memory notification to eliminate the need for polling.

-   xUI's should be adoptable in existing high performance processors,
    thus a small bill of materials, minimal changes to ISA, and not
    imposing a tax on any other features are mandatory. As we developed
    xUI in the x86-64 context, it builds on UIPI.

-   xUI should work with existing software, and enable simpler solutions
    than current user-level notification.

-   xUI should support all forms of notification offered by the
    interrupt system including timers, device interrupts, IPIs.

In the following subsections, we will discuss these extensions and
provide a brief description of their microarchitectural designs.

## Tracked Interrupts

Tracked interrupts are equivalent to normal interrupts from a
programmers perspective, they just improve performance. To achieve this,
we employ a different approach for dealing with the challenges of
speculative execution. There are two well known approaches for handling
interrupts. Flushing is the common approach on modern out of order
processors, it flushes the pipeline immediately when an interrupt is
received. This throws away in flight work, which can be substantial in
modern out-of-order processors and is becoming more expensive each year
as instruction windows grow. Draining retires in-flight instruction
before delivering the interrupt. While this does not waste work, it adds
latency, and similar to flushing, becomes more expensive as instructions
windows grow.

Tracking avoids both these compromises by exploiting the observation
that interrupts are inherently asynchronous, and thus the processor has
some flexibility as to where in the instruction stream the interrupt
handling code issues. Whether flushing or draining, current processors
treat the precise point in the instruction stream where the interrupt
signal was detected as meaningful, i.e., as a synchronous event, and
issue the interrupt handling micro-ops after that instruction. Tracking
exploits the fact that this "where" doesn't actually matter, thus it can
instead issue the interrupt handling micro-ops at the place where they
achieve the best possible latency that does not sacrifice throughput.

At a high level, our tracking approach works by resteering the fetch
control flow to the micro-ops that are responsible for handling an
interrupt ---referred to as interrupt notification processing--- as soon
as an interrupt is received and accepted by the APIC. The approach then
tracks whether these micro-ops are committed or not, ensuring that: 1)
they are not lost if the processors flushes parts of the ROB on
mispredicition, and 2) we can accurately identify and save the last
program instruction (return address). This method guarantees that the
interrupt is eventually delivered, even if the current branch is
misspeculated, without needlessly throwing away useful work.

## The Kernel Bypass (KB) Timer

The Kernel bypass timer (KB_timer) offers a per-thread timer, similar to
the local APIC timer. As with the APIC timer, it is not intended for
direct use by applications developers. Instead it offers a low level
primitive through which *user-level runtimes* can implement software
timers for tasks like preemption, periodic polling, timeouts, etc.

From user level, the timer is used through two new instructions: and .
is a 64-bit unsigned integer, and the mode is a one bit flag, indicating
if the timer is a periodic or a one-shot timer. If the is periodic,
cycles is interpreted as a period in cycles. If the is one-shot, cycles
is interpreted as a deadline, again, in keeping with the traditional
APIC design that makes it simple to specify the next deadline when
implementing multiple software timers.

There is one KB_timer per physical core, which is multiplexed among
multiple threads by the operating system. The KB_timer has its own
physical timer, rather than simply re-using the local APIC timer, as
that is already in use by the kernel. Similar to modern local APIC
timers, the timer uses the system clock to offer high resolution.

KB_timer interrupts are delivered by invoking the user-level interrupt
handler, and passing it the vector that was assigned by the kernel.
KB_timer interrupt delivery is very inexpensive (cycles), as it can skip
the microcode steps related to UPID's and routing ()
(§ 3.2) and invoke the microcode directly.

Unlike IPIs, there is no special slow path support, i.e., it is not
intended to fire when either the kernel, or any thread other than the
owner is executing. If the timer reaches its target in kernel mode, it
will trap. If it is in user mode, it will deliver an interrupt to
whatever thread is currently executing. Consequently, it is up to the
kernel to manage the timer state.

## Interrupt Forwarding: Routing Device Interrupts

We want to be able to get the benefits of xUI for devices, however, UIPI
has it's own delivery scheme using UPIDs that the rest of the interrupt
systems knows nothing about. It only knows how to route interrupts to
APICIDs, which are generally assigned to physical cores at boot time.

Interrupt forwarding solves this by extending the local APIC, allowing
it to forward interrupts destined for the core directly to the current
thread. If the destination thread is not running, the local APIC will
tell the OS to handle the notification, similar to UIPI. To implement
this, Interrupt Forwarding lets the kernel to setup mappings between
per-core (APICID) vectors, and user level vectors. Thus, when an
interrupt arrives on a mapped vector, the interrupt is forwarded, rather
like port forwarding in a network.

To setup a mapping, a thread registers through the OS to receive
interrupts from a device, and is assigned a notification vector.
Interrupts are delivered as normal through interrupt handler. However,
similar to the KB_timer, the UPID is never touched, thus fast path
interrupt delivery (where the receiving thread is running) is faster and
free of contention for shared memory.

Interrupt forwarding extends the local APIC with two new 256-bit
registers, where each bit corresponds to a vector. , indicates which
interrupts to forward on the current core. , indicates which interrupts
should be forwarded to the currently running thread.

Our interrupt forwarding scheme, while efficient and minimally
intrusive, is constrained by the limited vector space of the underlying
core as this must be shared by threads on the host. This limitation
restricts the number of device/user pairs that can be supported
simultaneously. One could imagine adding a new field to the message
format of the interrupt system, or to repurpose unused bits in the
existing message format (e.g. the clusterID) to avoid this limitation.

# Experimental Setup

## Hardware Platform

We run all experiments on an Intel Xeon Gold 5420+ processor with 28
Sapphire Rapids cores, using Intel's fork of Linux v6.0.0 with their
UIPI patches, The processor operates at 2 GHz with Turbo
Boost, frequency scaling, and c-states disabled to ensure controlled
measurements. For measurement, we use both the Linux perf
and Agner tools to read the performance counters. We calculate
the difference between the number of committed micro-ops and decoded
instructions to estimate the number of flushed micro-ops, as there is no
direct performance counter for flushed micro-ops.

## gem5 Simulation

Our simulation is based on gem5 (version 23), and extends the
out-of-order CPU model. We configure the baseline architecture to model
an Intel Sapphire Rapids processor such as the Intel Xeon Gold 5420+,
using the parameters shown in
Table 1. As kernel support is required, we run gem5
in full system mode, and again use Intel's fork of the v6.0.0 Linux
kernel. We implemented UIPI support in gem5, using the
results of our characterization study
to provide accurate costs for flushing, notification and delivery, and
finally added support for xUI on top of this. To support our I/O
notification experiments
(§ 5.3), we ported the
gem5-dpdk device model from ARM to x86, and added support
for delivering I/O completions using xUI.

| Parameter | Value | Parameter | Value |
|-----------|-------|-----------|-------|
| Frequency | 2.0 GHz | I cache | 32 KB, 8 way |
| Fetch Width | 6 $\mu$ops | D cache | 32 KB, 8 way |
| Issue Width | 10 $\mu$ops | Retire Width | 10 $\mu$ops |
| Squash Width | 10 $\mu$ops | Decode Width | 6 $\mu$ops |
| SQ Size | 72 entries | IQ | 168 entries |
| LQ Size | 128 entries | Functional Units | Int ALU(6), Mult(2), FPALU/Mult(3) |
| ROB Size | 384 entries | | |

**Table 1: Architecture Detail for the Baseline x86 Core**

One interesting discovery was that gem5's model of interrupt performance
is quite different from real hardware. For example, it drains the
pipeline instead of flushing it, and a fixed 13[^2] cycles were
artificially added after each drain. To remedy this, we plan to upstream
our improved interrupt model to gem5.

## IO Notification: L3Fwd & Simulated Accelerators

**DPDK-Based L3 Forwarding.** We used a methodology similar to previous
work to measure the inefficiencies associated with spin
polling with the use of DPDK. We used DPDK's Layer 3 Forwarding (L3Fwd)
application, with the the Longest Prefix Match (LPM) algorithm, and a
routing table containing 16,000 entries. For these experiments, we used
64-byte IPv4 UDP packets.

Similar to gem5-dpdk, we use a dedicated external device as
an open-loop packet generator. We modify gem5-dpdk's packet generator to
use an exponential distribution for inter-arrival times of a fixed
inter-arrival time to model the burstiness of real network
traffic.

## System Configuration

We model a system with 1, 2, 4, or 8 different NICs, each with its own
receive queue. The packets are sent back to the same NIC in the 1 NIC
case, similar to.

We present results for two different configurations that resemble a
particular load on two different Intel accelerators (as estimated by our
own measurements):

-   $2~\mu{}s$: latency of copying one 16 KB buffer using
    DSA, or latency of copying a batch of 8 buffers of
    64-2048 bytes

-   $20~\mu{}s$: latency of copying one 1 MB buffer using
    DSA or the latency of encrypting/decrypting 32 KB
    using QAT

We introduce variability in modeling the response time of accelerators
for different workload sizes to account for both inaccuracies in
estimating accelerator time (due to factors such as cold-start time,
DVFS, etc.) and resource contention (e.g., by other processes accessing
the accelerator or contention on the bus). We vary this unpredictability
to cover a wide range of scenarios. For the above points, we model
response time using a truncated normal distribution with parameters
$(\text{min}, \mu) = (1\mu{}s, 2\mu{}s)$ and $(10\mu{}s, 20\mu{}s)$.

## Evaluated Approaches

In our evaluation, we compare the following approaches:

-   *Polling*: no interrupts, poll continuously until an event completes

-   *UIPI SW Timer*: Intel's UIPIs, sent from a dedicated timer core

-   *UIPI SW Timer + Tracking*: tracked user interrupts sent from a
    dedicated timer core

-   *UI KB Timer + Tracking*: tracked user interrupts sent by per-core
    kernel-bypass timers

-   *Tracked Device UIs*: tracked user interrupts sent from a device

# Evaluation

## IO notification in l3fwd

![**xUI's CPU Utilization with DPDK-l3fwd application:** *Tracked Device
UIs can free up CPU cycles that polling would waste.*
](figures/fig-l3-forwarding.pdf)

In this experiment, we compare the performance of polling with xUI's
tracked interrupts from devices, using the DPDK l3fwd application. We
observe that the throughput of both approaches is nearly identical, with
tracking's maximum throughput falling short of polling by 0.08%. With
Device UIs, the interrupt handler polls the network queue again before
returning; at high loads the handler never returns, yielding the same
throughput as with polling. Tracking achieves a 95th percentile tail
latency within +2%, -8%, and +65% of polling's latency for 1, 4, and 8
NICs, respectively. As shown in
Figure 2, with polling, all CPU cycles are spent
either handling packets ("Networking Cycles") or polling the network
stack. In contrast, device interrupts, polling is unnecessary. Thus,
there are extra CPU cycles ("Rescued Cycles") that could be used for
other application-level work when the core is not busy handling packets
or interrupts. For example, with 1 queue and 40% load, tracking recovers
45% of the CPU cycles that polling would waste.

## Preemption for RocksDB in Aspen

Preemption can mitigate HOL blocking, reducing tail latency in workloads
with high dispersion. We evaluated these benefits for RocksDB in Aspen,
using three configurations: UIPI SW
Timer, UI KB Timer + Tracking, and Non-preemptive (baseline). with a
preemption quantum of 5 .

Figure 3
shows the tail latency for GET and SCAN requests as we vary the offered
load. Without preemption, the tail latency for GET requests is hundreds
of microseconds, even under very low load. In contrast, with preemption
via UIPIs, Aspen is able to maintain low tail latency for GET requests
(at most a few hundred microseconds) up to throughputs of over 100,000
requests per second. The addition of KB Timers and Tracking further
increases throughput by 10%, due to the lower overhead of receiving UIs.
In both cases, preemption comes at the cost of slightly elevated tail
latencies for SCAN requests at high load.

![**Throughput of RocksDB:** *KB timers and tracked interrupts can
achieve 10% higher throughput for GET requests than UIPI.*
](figures/fig-rocksdb.pdf)

[^1]: User interrupt support is slated to appear in both server and
    consumer class chips (Sapphire Rapids, Sierra Forest, Grand Ridge,
    Arrow Lake, and Lunar Lake) in the coming
    year.

[^2]: We don't know that this is artificial, it might be high but the
    commit stage has to communicate with the fetch stage
