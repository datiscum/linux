# Scheduler stability backport

## Background

This branch contains scheduler fixes backported from the Linux 7.1 development
cycle to Linux 6.18.

The affected production server uses one AMD EPYC 7643 48-Core Processor with
48 physical CPU cores and 96 hardware threads.

Diagnosing this failure took approximately six months. Before the complete
scheduler patch set was installed, the server normally remained operational
for only about three days before the next crash, lockup, or complete scheduler
stall.

The failure was particularly difficult to reproduce because the server usually
locked up while the system was under relatively low load, not during the period
of highest CPU utilization.

In one representative test, the server sustained a system load above 100 for
approximately three hours. The workload completed successfully, but shortly
after the test ended and the system returned to low load, the server locked
up.

The machine could therefore survive hours of extreme load and then fail almost
immediately during or after the transition back to low load.

## Observed failure signatures

The failures produced different stack traces and symptoms:

* a direct divide-by-zero exception in `avg_vruntime()`
* soft lockups
* hard lockups
* RCU stalls and RCU-preempt starvation
* runnable tasks no longer receiving CPU time
* blocked or stalled kernel workers
* complete system freezes without a useful final panic
* secondary traces in apparently unrelated kernel subsystems

The different final traces were one reason the investigation took so long.

The collected evidence is consistent with weighted-vruntime accounting having
overflowed or become inconsistent before at least some of these failures.
Invalid scheduler accounting can lead to incorrect scheduling decisions and
may starve tasks, kernel workers, or RCU callbacks.

A final stack trace may therefore identify the subsystem that first detected
the resulting lack of progress rather than the earlier scheduler-accounting
failure. This provides a plausible explanation for the different visible
failure signatures, although not every possible lockup or RCU stall is
necessarily caused by this accounting problem.

## The decisive stack trace

The most important trace was the first one that pointed directly into the
CFS/EEVDF scheduler.

It identified the division in `avg_vruntime()` as the immediate crash site.
The relevant upstream Linux 7.1 form of the function was:

```c
u64 avg_vruntime(struct cfs_rq *cfs_rq)
{
        struct sched_entity *curr = cfs_rq->curr;
        long weight = cfs_rq->sum_weight;
        s64 delta = 0;

        if (curr && !curr->on_rq)
                curr = NULL;

        if (weight) {
                s64 runtime = cfs_rq->sum_w_vruntime;

                if (curr) {
                        unsigned long w =
                                avg_vruntime_weight(cfs_rq,
                                                   curr->load.weight);

                        runtime += entity_key(cfs_rq, curr) * w;
                        weight += w;
                }

                /* sign flips effective floor / ceiling */
                if (runtime < 0)
                        runtime -= (weight - 1);

                delta = div64_long(runtime, weight);
        } else if (curr) {
                /*
                 * When there is but one element, it is the average.
                 */
                delta = curr->vruntime - cfs_rq->zero_vruntime;
        }

        update_zero_vruntime(cfs_rq, delta);

        return cfs_rq->zero_vruntime;
}
```

This trace was decisive because it was the first failure that did not merely
report a secondary soft lockup, hard lockup, or RCU stall. It showed an
arithmetic failure directly inside the scheduler.

The initial `if (weight)` test checks only the value copied from
`cfs_rq->sum_weight`. When a current entity is present, its adjusted weight is
added afterwards. The resulting signed `long` value is not checked again
before it is passed to `div64_long()`.

If that addition or earlier accounting overflow leaves the effective weight
at zero, the division faults. A negative result does not itself cause a
divide-by-zero exception, but it is equally invalid for this weighted-average
calculation and indicates wrapped or inconsistent accounting.

The same type of accounting failure does not necessarily reach the division
immediately. Incorrect vruntime or eligibility values may instead prevent
runnable entities from being selected. This provides a plausible explanation
for failures that appeared as task starvation, RCU stalls, or a completely
frozen server without an immediate divide-by-zero exception.

## Upstream Linux 7.1 backports

The patch series contains three scheduler fixes from the Linux 7.1 development
cycle.

### 1. Increase weight bits for avg_vruntime

Commit:

```text
4823725d9d1d9cc5b36647e0cb8ff616cad6536f
```

Subject:

```text
sched/fair: Increase weight bits for avg_vruntime
```

This patch changes the weighted-vruntime representation so that
`avg_vruntime()` can track the full scheduler weight range. It introduces
`avg_vruntime_weight()`, `sum_shift`, and the optional `PARANOID_AVG`
accounting checks.

The upstream commit explains that measurements from kernel builds and
`hackbench` runs used approximately 45 bits for the relevant deltas. Keeping
more weight bits reduces numerical artifacts during reweighting and related
scheduler operations.

Original discussion:

* https://patch.msgid.link/20260219080624.942813440%40infradead.org

Mainline sources:

* Commit:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4823725d9d1d9cc5b36647e0cb8ff616cad6536f
* Patch:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/patch/?id=4823725d9d1d9cc5b36647e0cb8ff616cad6536f

### 2. Avoid overflow in enqueue_entity

Commit:

```text
556146ce5e9476db234134c46ddf0e154ca17028
```

Subject:

```text
sched/fair: Avoid overflow in enqueue_entity()
```

The upstream commit describes a reproducer that triggered the problem after
approximately one hour on a machine with 256 CPUs:

```bash
stress-ng --yield=32 -t 10000000s &

while true; do
        perf bench sched messaging -p -t -l 100000 -g 16
done
```

The upstream report does not describe the physical CPU topology of that
machine. It states only that the machine exposed 256 CPUs to the kernel.

The captured values showed an overflowing multiplication between a very large
negative `entity_key` and the entity weight.

The original upstream commit misspells the diagnostic label as
`__enqeue_entity`; it refers to the actual kernel function
`__enqueue_entity()`.

The diagnostic label is normalized below:

```text
__enqueue_entity:
    entity_key(-141245081754)
    weight(90891264)
    overflow_mul(5608800059305154560)
    vlag(57498)
    delayed?(0)

cfs_rq:
    zero_vruntime(3809707759657809)
    sum_w_vruntime(0)
    sum_weight(0)
    nr_queued(1)

cfs_rq->curr:
    entity_key(0)
    vruntime(3809707759657809)
    deadline(3809723966988476)
    weight(37)
```

The fix moves `zero_vruntime` towards a newly enqueued heavy entity when that
entity is heavier than the existing runqueue load. This keeps the factors in
the weighted-vruntime multiplication within a safer range and avoids the
reported overflow.

Original discussion:

* https://patch.msgid.link/20260407120052.GG3738010%40noisy.programming.kicks-ass.net

Mainline sources:

* Commit:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=556146ce5e9476db234134c46ddf0e154ca17028
* Patch:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/patch/?id=556146ce5e9476db234134c46ddf0e154ca17028

### 3. Use a 128-bit eligibility calculation

Original TIP commit:

```text
416dd8fc352b6027210b42de5889ee280e4bc40c
```

Final Linux mainline commit:

```text
b6eee96843e8d088200f01b035da98e72067c5fe
```

Subject:

```text
sched/fair: Fix overflow in vruntime_eligible()
```

The first scheduler backport was incomplete because it contained
`4823725d9d1d` and `556146ce5e94`, but not this final eligibility fix.

Before this fix, the eligibility comparison multiplied the signed vruntime key
by the runqueue load using a native-width intermediate:

```c
return avg >=
       vruntime_op(vruntime, "-", cfs_rq->zero_vruntime) * load;
```

The difference calculated by `vruntime_op()` is represented by `key` in the
corrected implementation.

The upstream analysis describes a worst case involving a very light cgroup
entity with a weight of 2 and another entity with a weight of
`100 * NICE_0_LOAD`.

The resulting comparison is effectively:

```text
avg >= puny.key * load
```

The complete product contains approximately:

```text
(slice + TICK_NSEC) * NICE_0_LOAD * NICE_0_LOAD * 100
```

On a 64-bit system, the mathematically correct result may require more than
64 bits and overflow `s64`.

An overflow can change the resulting value or sign and flip the eligibility
result. An entity that should be eligible may consequently appear ineligible.

If all otherwise suitable entities appear ineligible, `pick_eevdf()` may fail
to select a runnable entity. This can result in task starvation and scheduler
failure even when no immediate divide-by-zero exception occurs.

The final implementation uses a 128-bit intermediate on 64-bit architectures
that support it:

```c
static int vruntime_eligible(struct cfs_rq *cfs_rq, u64 vruntime)
{
        struct sched_entity *curr = cfs_rq->curr;
        s64 key, avg = cfs_rq->sum_w_vruntime;
        long load = cfs_rq->sum_weight;

        if (curr && curr->on_rq) {
                unsigned long weight =
                        avg_vruntime_weight(cfs_rq, curr->load.weight);

                avg += entity_key(cfs_rq, curr) * weight;
                load += weight;
        }

        key = vruntime_op(vruntime, "-", cfs_rq->zero_vruntime);

#ifdef CONFIG_64BIT
#ifdef CONFIG_ARCH_SUPPORTS_INT128
        /*
         * This often results in simpler code than
         * __builtin_mul_overflow().
         */
        return avg >= (__int128)key * load;
#else
        s64 rhs;

        /*
         * On overflow, the sign of key tells us the correct answer:
         * a large positive key means vruntime >> V, so not eligible;
         * a large negative key means vruntime << V, so eligible.
         */
        if (check_mul_overflow(key, load, &rhs))
                return key <= 0;

        return avg >= rhs;
#endif
#else
        return avg >= key * load;
#endif
}
```

This is not a general conversion of the scheduler to 128-bit arithmetic. It
specifically protects the multiplication in `vruntime_eligible()`, whose
mathematically correct result may not fit in a native signed 64-bit
intermediate.

The upstream discussion explains that several architectures already calculate
a 128-bit product internally when checking multiplication overflow. Using
`__int128` directly can therefore produce simpler code. Architectures without
`CONFIG_ARCH_SUPPORTS_INT128` retain the explicit
`check_mul_overflow()` fallback.

The fix was merged during the Linux 7.1 development cycle and was present by
Linux v7.1-rc3.

`416dd8fc` was the original TIP `sched/urgent` commit announced on
May 5, 2026. During subsequent TIP integration, the final mainline version
received commit ID `b6eee968`.

Original discussion:

* https://patch.msgid.link/20260505103155.GN3102924%40noisy.programming.kicks-ass.net

Original TIP announcement:

* https://www.spinics.net/lists/kernel/msg6186951.html

Final mainline sources:

* Commit:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b6eee96843e8d088200f01b035da98e72067c5fe
* Patch:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/patch/?id=b6eee96843e8d088200f01b035da98e72067c5fe

## Local divide-by-zero protection

The upstream overflow fixes prevent known overflow paths in the scheduler
accounting and eligibility calculations. This branch additionally contains a
defensive check directly before the dangerous division in `avg_vruntime()`.

The complete relevant function after applying the local guard is:

```c
u64 avg_vruntime(struct cfs_rq *cfs_rq)
{
        struct sched_entity *curr = cfs_rq->curr;
        long weight = cfs_rq->sum_weight;
        s64 delta = 0;

        if (curr && !curr->on_rq)
                curr = NULL;

        if (weight) {
                s64 runtime = cfs_rq->sum_w_vruntime;

                if (curr) {
                        unsigned long w =
                                avg_vruntime_weight(cfs_rq,
                                                   curr->load.weight);

                        runtime += entity_key(cfs_rq, curr) * w;
                        weight += w;
                }

                /*
                 * This must never happen. If it does, the cfs_rq
                 * accounting is inconsistent or the weight addition
                 * wrapped non-positive.
                 *
                 * Do not divide by 1 here: that could turn a corrupted
                 * runtime into a huge vruntime jump. Fall back to the
                 * only sane local reference point.
                 */
                if (unlikely(weight <= 0)) {
                        delta = curr ? entity_key(cfs_rq, curr) : 0;
                } else {
                        /* sign flips effective floor / ceiling */
                        if (runtime < 0)
                                runtime -= (weight - 1);

                        delta = div64_long(runtime, weight);
                }
        } else if (curr) {
                /*
                 * When there is but one element, it is the average.
                 */
                delta = curr->vruntime - cfs_rq->zero_vruntime;
        }

        update_zero_vruntime(cfs_rq, delta);

        return cfs_rq->zero_vruntime;
}
```

The condition deliberately checks `weight <= 0`, not only
`weight == 0`.

`weight` is a signed `long`. A wrapped or inconsistent value can therefore
become negative as well as zero.

Zero would cause the observed divide-by-zero exception. A negative value would
not itself cause a divide-by-zero exception, but it is equally invalid for this
weighted-average calculation and must not be passed to the normal division
path.

The fallback does not divide by one. Dividing a potentially corrupted
`runtime` value by one could turn the accounting failure into a very large
vruntime jump and cause further task starvation.

Instead, the fallback uses the current entity's local scheduler key:

```c
delta = curr ? entity_key(cfs_rq, curr) : 0;
```

The function then continues through its normal exit path:

```c
update_zero_vruntime(cfs_rq, delta);
return cfs_rq->zero_vruntime;
```

This preserves the normal `zero_vruntime` update while preventing the
divide-by-zero panic and avoiding an uncontrolled vruntime jump.

The local check is a defensive last-resort guard. It does not replace the
upstream overflow fixes and cannot repair an accounting state that has already
become inconsistent.

## Runtime workarounds did not fix the problem

The following scheduler-feature changes were tested during the six-month
investigation:

```bash
echo PARANOID_AVG > /sys/kernel/debug/sched/features
echo NO_DELAY_DEQUEUE > /sys/kernel/debug/sched/features
echo NO_DELAY_ZERO > /sys/kernel/debug/sched/features
```

These settings did not make the server stable.

`PARANOID_AVG` added diagnostic accounting checks and rebuilding behavior.
`NO_DELAY_DEQUEUE` and `NO_DELAY_ZERO` changed delayed-dequeue and zero-lag
behavior.

They were useful diagnostic experiments, but they did not replace the missing
overflow fixes, did not provide the 128-bit eligibility comparison, and did
not guarantee a valid divisor in `avg_vruntime()`.

The crashes continued until the complete scheduler patch series and the local
`weight <= 0` protection were installed.

## Proxy-execution donor protection

The branch additionally contains a local protection for the active
proxy-execution donor in `try_steal_cookie()`.

The original core-scheduling check excluded only `src->core_pick` and
`src->curr`:

```c
if (p == src->core_pick || p == src->curr)
        goto next;
```

When proxy execution is active, `rq->donor` is also part of the active
scheduling context. Moving that task to another runqueue while the source
runqueue still uses it as its donor is unsafe.

The relevant observed scheduler path was:

```text
sched_core_balance()
  -> steal_cookie_task()
     -> try_steal_cookie()
        -> double_rq_lock()
           -> raw_spin_rq_lock()
```

The local check therefore also protects `src->donor`:

```c
if (p == src->core_pick || p == src->curr || p == src->donor)
        goto next;
```

## Observed result

Before the complete scheduler fixes, the production server normally ran for
only about three days before the next crash or lockup.

The server usually locked up while the system was under relatively low load,
not during the period of highest CPU utilization.

In one representative test, the server sustained a system load above 100 for
approximately three hours. The workload completed successfully, but shortly
after the test ended and the system returned to low load, the server locked
up.

Finding the common cause required approximately six months of repeated
testing, stack-trace analysis, incomplete fixes, and production failures.

With the three Linux 7.1 scheduler backports, the local defensive
`avg_vruntime()` check, and the proxy-execution donor protection installed,
the affected AMD EPYC 7643 system has operated stably without the previously
observed scheduler divide-by-zero panics, scheduler starvation, soft or hard
lockups, and RCU stalls.

This is the observed result on this specific 48-core/96-thread production
server. It is not a claim that the patches fix every possible cause of a
kernel lockup or RCU stall.

## Intel Xe backports and runtime power requirement

This branch contains selected Intel Xe and TTM stability backports, including
Battlemage D3Cold handling and fixes for execution recovery, GuC handling,
page tables, dma-buf lifetime, WOPCM sizing, and GT resume failures.

The tested graphics card is an Intel Arc(TM) Pro B60 Graphics based on the
Battlemage BMG G21 GPU. It is assigned to a KVM virtual machine through VFIO.

If the device fails, the virtual machine enters the paused state and must be
forcibly powered off.

Even with the Xe backports applied, the PCI runtime power policy must be
forced to `on`:

```bash
echo on > /sys/bus/pci/devices/0000:04:00.0/power/control
```

This command must be executed after the Xe driver has been loaded and
initialized. Otherwise, the Xe driver sets the runtime power policy back to
`auto`.

Keeping the policy set to `on` prevents runtime suspension and avoids the
observed device failure. This requirement applies to the tested Intel
Arc(TM) Pro B60 Graphics (BMG G21) at PCI address `0000:04:00.0`.
