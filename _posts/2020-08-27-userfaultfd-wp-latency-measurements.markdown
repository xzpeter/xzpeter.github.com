---
title: Userfaultfd-wp Latency Measurements
layout: page
---

# Overview

Userfaultfd-wp (or in short, *uffd-wp*) was just merged into Linux 5.8.  This
article tries to analyze the overhead of uffd-wp on servicing page requests,
and compare it with *mprotect()*.

Note that this test only covers the message delivery path for a single threaded
application.  No extra features are compared and considered.

# Environments

The measurement will be based on Linux 5.9 (with [patch][patch-cow-fix] applied
to fix the breakage of uffd-wp for the time when the article is written, since
5.9-rc1 still has uffd-wp broken).

To make things simple, all mitigations are turned off for the tests.
Meanwhile, timestamp is always collected by *rdtsc* instructions.

Host CPU information:

    Architecture:                    x86_64
    CPU op-mode(s):                  32-bit, 64-bit
    Byte Order:                      Little Endian
    Address sizes:                   39 bits physical, 48 bits virtual
    CPU(s):                          8
    On-line CPU(s) list:             0-7
    Thread(s) per core:              2
    Core(s) per socket:              4
    Socket(s):                       1
    NUMA node(s):                    1
    Vendor ID:                       GenuineIntel
    CPU family:                      6
    Model:                           142
    Model name:                      Intel(R) Core(TM) i7-8665U CPU @ 1.90GHz
    Stepping:                        12
    CPU MHz:                         1900.281
    BogoMIPS:                        4199.88
    Virtualization:                  VT-x
    L1d cache:                       128 KiB
    L1i cache:                       128 KiB
    L2 cache:                        1 MiB
    L3 cache:                        8 MiB
    NUMA node0 CPU(s):               0-7

# How To Measure

No matter which method we use (uffd-wp, or mprotect()), the write protection
and page fault resolving procedures are similar:

  1. A memory write to a write-protected page, triggers #PF, a message or
     signal is sent.
  2. Receiving of the message.
  3. Resolving of the page fault by unprotect the page.
  4. The memory access continues.
  
In the measurement, for each of the memory accesses like this, we take a
timestamp right before each step, then we can do some simple maths on roughly
how long time used for each step.

The [test program][test-new] that is used to do the measurement in this article
is based on a program that was written by Aditya [here][test-origin].  In the
new program, we restricted the write pattern to sequential writes only so that
we can take notes on each of the page fault request in an array easily.  Also
that's required for the extra *uffd-wp-sigbus* comparison test.

To run the test with specific protection modes, we can use:

    ./uffd <test-mode>
    
To run with userfaultfd-wp, we can use:

    ./uffd uffd-wp
    
While to run with mprotect()+SEGV, we can use:

    ./uffd mprotect
    
# Test Results

## mprotect() + SEGV

mprotect() + SEGV is the old method for write protecting pages.  The page fault
is delivered as a *SIGSEGV* message. A signal handler is responsible for
resolving the page fault by calling the syscall *mprotect()* again with the
write permission guaranteed.  One thing to mention is that, both the page fault
and fault resolving is done in a single thread in the test program, so no
thread switching is needed.  However there'll still be context switches between
the user/kernel for either signal delivery, or page fault resolving (the 2nd
call to mprotect()).

Although it is signal based, the single-thread performance is actually quite
well, with an average overhead of ***1.92us*** per page request.  Here:

  * From write happens, to receiving SIGSEGV: ***0.74us***
  * Page fault resolving took (mprotect()): ***0.36us***
  * Then until the write continues: ***0.81us***

## uffd-wp

uffd-wp is the new way to do similar thing.  Instead of using signals, it uses
*uffd_msg* to deliver fault messages, so that the other thread can poll the
userfaultfd handle and receive the message by reading the handle.  Also it
naturally supports threadings, so that the fault resolving thread is different
from the worker thread.  From that point of view, extra switching overhead is
required for simple write protection cases (like the [test program][test-new]
that we'll be using).

The thread-based solution should have brought extra overhead to uffd-wp when
there's only one single worker thread and one single servicing thread, with an
average of ***4.74us*** per page request.  Here:

  * From write happens, to receiving the uffd_msg: ***2.43us***
  * Page fault resolving took (UFFDIO_WRITEPROTECT): ***1.16us***
  * Then until the write continues: ***1.15us***
  
We can see that each step took longer than the mprotect() case.  Note that here
"until the write continues" is not strict in our case because after the ioctl
UFFDIO_WRITEPROTECT completes, logically the other worker thread can run in
parallel with the uffd servicing thread.  However we can ignore the difference
for now (however from the real data, we can see some very rare negative values,
probably because the worker thread is scheduled and ran faster, so the ioctl
UFFDIO_WRITEPROTECT returned even after the worker thread continued running).

The wild guess of the performance degradation on uffd-wp is that we have had
two threads so there're extra scheduling overhead.  Also I believe there can be
more cache hit for mprotect() too when all operations are happening in the same
thread, so the context is kept for the whole process while uffd-wp does not.

To validate our thoughts, I added an extra test to use uffd-wp but keep it in a
single thread (so avoid context switches).  It's done by leveraging the
userfaultfd feature UFFD_FEATURE_SIGBUS so that instead of sending a uffd_msg,
we'll send a SIGBUS when write protection happens.

## uffd-wp + SIGBUS

To run with userfaultfd-wp SIGBUS mode, we can use:

    ./uffd uffd-wp

With the same program.  A total average of overhead is ***1.85us***, and for
each steps:

  * From write happens, to receiving the SIGBUS: ***0.77us***
  * Page fault resolving took (UFFDIO_WRITEPROTECT): ***0.27us***
  * Then until the write continues: ***0.81us***

With single thread context, uffd-wp with SIGBUS performed even slightly better
than mprotect() (seems UFFDIO_WRITEPROTECT is the major part of the win, since
for the other two steps it performs similar to mprotect() case).  This kind of
verified the previous idea that the context switching should have brought extra
overhead to uffd-wp.

# Conclusion

Here's a summary of all the measurements (raw data can be found [here][data]):

|----------------------+---------------+---------------+----------+--------|
| Test Mode            | Receiving Msg | #PF Resolving | Complete | Total  |
|----------------------+---------------+---------------+----------+--------|
| mprotect() + SIGSEGV | 0.74us        | 0.36us        | 0.81us   | 1.92us |
| uffd-wp              | 2.43us        | 1.16us        | 1.15us   | 4.74us |
| uffd-wp + SIGBUS     | 0.77us        | 0.27us        | 0.81us   | 1.85us |
|----------------------+---------------+---------------+----------+--------|

Although userfaultfd-wp should have brought significantly new features for old
mprotect() scenarios, it probably does not mean that uffd-wp will always be
faster, especially for a single threaded application and when the workload is
low.

Some more multi-thread measurements could be done in the future to further
compare between uffd-wp with mprotect().

[patch-cow-fix]: https://lore.kernel.org/lkml/20200821234958.7896-1-peterx@redhat.com/
[test-origin]: https://github.com/adityamandaleeka/userfaultfd-wp-writetracking/blob/main/uffd.cpp
[test-new]: https://github.com/xzpeter/userfaultfd-wp-writetracking/blob/peter/uffd.cpp
[data]: /misc/uffd-mprotect-delay.ods
