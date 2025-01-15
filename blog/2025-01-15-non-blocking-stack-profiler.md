# SDB: a Non-Blocking Ruby Stack Profiler

## Introduction

Several Ruby stack profilers already exist, but most of them depend on the Ruby Global VM Lock (GVL), which blocks applications while scanning the Ruby stack. Based on my testing, a Ruby stack profiler can increase request latency by up to 10% when sampling at a 1ms interval. Rbspy's async mode is an exception, as it scans the Ruby stack from a separate process. However, this approach can lead to errors[1] because it accesses complex Ruby data structures without acquiring the GVL.

In this article, I will introduce how SDB scans Ruby stacks without the GVL while maintaining safety and delivering the results we want. It allows us to scan the Ruby stack without increasing application delay. I believe this implementation is interesting as it demonstrates the benefits of releasing the GVL in a performance-sensitive library.


## Scanning Stacks Without GVL

The Ruby GVL ensures the integrity of Ruby data but impacts concurrency.

It seems that we must hold the GVL to access Ruby data. However, when we look closely, if the data itself can guarantee atomic, it doesn’t need the GVL’s protection. For example, it is safe to read 64-bit aligned data since its updates are atomic — there are no partial updates[2][3]. An ISeq’s address is an example of such data. If we know where an ISeq’s address is stored, it is safe to read it without holding the GVL.

The Ruby stack contains ISeq data in an array of `rb_control_frame_struct` objects, allocated when Ruby creates a thread. It is safe to read it as long as Ruby has not reclaimed the stack. Without the GVL, we might encounter empty or outdated data (due to lack of memory barriers), but we will not read invalid data (e.g., addresses from a different ISeq).

To ensure we stop scanning a thread's stack once it has been reclaimed, we can use meta-programming to hook into Ruby's thread lifecycle. The following example demonstrates how to wrap thread creation and deletion:

```ruby
module ThreadInitializePatch
  def initialize(*args, &block)
    old_block = block

    block = ->() do
      Sdb.thread_created(Thread.current)
      result = old_block.call(*args)
      Sdb.thread_deleted(Thread.current)
      result
    end

    super(&block)
  end
end

Thread.prepend(ThreadInitializePatch)
```

Before Ruby reclaims a thread, it calls `Sdb.thread_deleted`, which removes the thread from SDB's scanning list. This ensures safe stack scanning without the GVL.

## Symbolization

The data we collected are ISeq addresses, which are not readable to humans.SDB uses eBPF to capture ISeq creation events during compilation or loading (e.g., via bootsnap). Then, we can use an offline program to translate these addresses into human-readable symbols.
Ruby's memory compaction can move ISeq objects to new addresses. To handle this, we can use eBPF to capture memory compaction events too. This feature is still in progress, as Ruby's memory compaction is not enabled by default and is rarely used in production applications.

## Ensuring Fully Correctness

Data races may occur if the Ruby VM updates the stack while SDB is reading it. However, resolving this issue completely is not the primary goal of a stack profiler, as minor inaccuracies do not typically affect its usage in identifying performance bottlenecks.

Similar issues exist in other profilers like rbspy[4], py-spy[5], and async-profiler[6]. The key question is whether the profiler can identify slow paths in the application.  Data races can only occur if the stack is updated faster than the scanner can read it, which typically happens with extremely fast functions that have minimal impact on overall performance.

To manage this, we can use a generation number for optimistic concurrency control. When a stack is pushed, we increment the generation number by one and check this number before and after scanning the stack.[7]

## Summary

This article introduces how SDB scans the Ruby stack without relying on the GVL. By not blocking the application, it enables faster stack scanning and more accurate latency measurements, making a true always-on stack profiler possible.

## References

1. [Should I pause a Ruby process to collect its stack?
](https://jvns.ca/blog/2018/01/15/should-i-pause-a-ruby-process-to-collect-its-stack/#what-happens-if-i-don-t-pause-the-ruby-process-i-m-profiling)
2. https://pdos.csail.mit.edu/6.1810/2024/lec/l-rcu.txt
3. [Ruby Memory Model](https://docs.google.com/document/d/1pVzU8w_QF44YzUCCab990Q_WZOdhpKolCIHaiXG-sPw/edit?tab=t.0)
4. https://github.com/rbspy/rbspy
5. https://github.com/benfred/py-spy
6. https://github.com/async-profiler/async-profiler