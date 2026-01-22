- Feature Name: eevdf_scheduler
- Start Date: 2024-11-11
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Implement the [EEVDF scheduler](https://www.kernel.org/doc/html/next//scheduler/sched-eevdf.html)
algorithm for the Redox kernel.

# Motivation
[motivation]: #motivation

It was [recently raised](https://matrix.to/#/!lJMcRpoJcrWcQwuxoV:matrix.org/$j1XwiHyk7K7GlpZ22qbee5UZeylcH4WswiTCKzEBVmI?via=matrix.org&via=tchncs.de&via=t2bot.io)
that the lack of a better scheduler is one of the biggest pain points for Redox's performance.

Redox's kernel currently implements a pure round robin solution, meaning that critical
tasks compete for resources with mundane tasks without any distinct priority.

There's an [issue](https://gitlab.redox-os.org/redox-os/kernel/-/issues/156) dedicated
to this, in which the former Linux scheduler, the Completely Fair Scheduler, or CFS for
short, is mentioned. However, since 2007 it has received numerous patches and still
relies on "often-fragile heuristics", as pointed out in [this article](https://lwn.net/Articles/925371/).

Furthermore, EEVDF has recently received an [important patch](https://lwn.net/ml/linux-kernel/20240405102754.435410987@infradead.org/)
to deal with sleeping tasks properly. Even then, the overall simplicity of the algorithm remains
mostly untouched.

# Detailed design
[design]: #detailed-design

Let's first cover some basics. "EEVDF" stands for "Earliest Eligible Virtual Deadline First".
In a setting where processes compete for CPU time, a "virtual deadline" (VD) is a relative
measure of urgency. The lower the VD of an eligible process, the sooner it should be granted
its share of the resource. And an eligible process is simply one that can continue its doing,
i.e., it's not blocked, sleeping etc.

Note: the design proposed in this document is a composition of ideas from
* [The current scheduling algorithm](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/context/switch.rs?ref_type=heads)
* [The original EEVDF paper](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564)
* [A high level description](https://lwn.net/Articles/969062/) of Peter Zijlstra's work
* My own thought process

## Quantifying virtual deadlines

VD is relative because it's only sensible for comparison purposes. In the end, we just want
more crucial processes to be picked up more often. But doing so in a fair manner is fundamentally
a heuristic-based problem, since acquiring preempive knowledge of the complete unfolding of a
process and how it will affect the global set of processes is not practical from the kernel's
perspective - not to say virtually impossible. That said, let's start with the basic behaviors
we want to encode.

1. Processes should be rewarded for austerity: when switching contexts, the new process is given
a maximum slice of CPU time it can use. But the penalty should be proportional to the actual CPU
time the process consumes, as it can yield earlier.

2. The more important a task is, the less it should be penalized: "more important" is a deliberately
vague description. But we need to capture some notion of importance otherwise processes with greater
potential to unblock others (e.g. performing syscalls) won't receive any special treatment when
competing for CPU time.

3. Processes spawned recently shouldn't preempt already existing ones, unless explicitly specified
otherwise.

Now we can be more concrete. If we consider VD to be some numerical quantity, such as a `float`,
"penalizing" a process means giving it a higher VD value. So let's define

* `q` as the **q**uantum of maximum CPU time processes are granted in their turn,
* `t_p` as the actual CPU **t**ime the process `p` consumed (`t_p <= q`) and
* `w_p` as the **w**eight of the process `p`, capturing its importance as mentioned above.

The incrementing operation for `vd_p` (VD of the process `p`) can be simply

```
vd_p += t_p / w_p
```

That shows how we can update the VD of a process. To pick an initial VD for a process, we can choose
the highest VD among all processes plus some small `ε`. This is enough to make a process that was just
spawned be picked last, following chronological order of spawn times and thus being fair to already
existing processes.

However, by demand of higher powers, a process can be requested to run earlier by specifying a `0 <= α <= 1`,
which should place the process near the `α`-percentile:

`vd_p := MIN_VD + α*(MAX_VD - MIN_VD) - ε`.

We use `ε` here just to guarantee that when `α = 0`, the process will actually come out as the immediate
next one to be picked up.

### Avoiding overflowing VDs

By the incremental equation `vd_p += t_p / w_p`, note that VDs can only grow. To avoid overflowing values,
we multiply the increment by some small `0 < K << 1`.

```
vd_p += K * t_p / w_p
```

Additionally, if necessary, the kernel can have a task triggered on demand to slide all VDs so that the
smallest VD is zero again.

## Querying the process with the lowest VD

At this point we aren't left with many options to choose the data structure to hold the set of
contexts. We need a min-[heap](https://en.wikipedia.org/wiki/Heap_(data_structure)) whose node's
keys are their VDs. Then popping the process with the lowest VD will be done in `O(log n)`,
where `n` is the number of processes in the heap.

From now on, let's refer to this set of contexts as the *process **q**ueue*, `Q`.

## Not all processes are runnable

Just because a process has the lowest VD it doesn't necessarily mean it's runnable. For instance, a
process may be sleeping or simply blocked due to arbitrary reasons.

So we need to keep popping processes from `Q` until we find one that can run. Sleeping processes, in
particular, may be able to wake up if their wake time is lower than or equal to the current CPU clock
**t**ime `T`, which makes them eligible to run right away.

However, we need to accumulate processes as we pop them so we can insert them back after the one process
that can run is found. So here goes a first attempt:

```
schedule Q :=
  non_runnable := ∅ -- accumulates non-runnable processes
  while let some p := Q.pop
    if p.can_run
      Q.drain non_runnable
      return some p
    else
      if let some wake_time := p.wake_time
        -- blocked by sleep
        if wake_time <= T
          -- p can wake up and run
          p.wake_time = none
          Q.drain non_runnable
          return some p
        else
          -- sleeping but can't wake up
          non_runnable.insert p
      else
        -- blocked by some other reason
        non_runnable.insert p
  -- no return so far so no process is runnable
  Q.drain non_runnable
  none
```

But what would happen if a process that's sleeping, potentially for a long time, has the lowest VD?
It would keep being popped and reintroduced to `Q` over and over until it can wake up. This overhead
can be avoided for tasks with known wake up times.

### Dealing with sleeping processes

The idea is straightforward: dequeue sleeping processes if they can't wake up yet.

We can store such **s**leeping processes on a min-heap `S`, but this one should use the processes'
wake time as the comparing key. Then, when it's time to schedule a process to run next, we move
processes that can wake up from `S` back to `Q`.

Let's scratch the previous attempt and consider the following as the proposed algorithm:

```
schedule Q S :=
  -- take sleeping processes that can wake up from S back to Q
  while let some p := S.head
    -- S is not empty
    if p.wake_time.unwrap <= T -- all processes in S have wake time set
      -- the head process can wake up
      Q.insert S.pop
    else
      -- no other process will be ready to wake up
      break

  -- and now we do almost the same thing as before
  non_runnable := ∅ -- accumulates non-runnable processes that aren't sleeping
  while let some p := Q.pop
    if p.can_run
      Q.drain non_runnable
      return some p
    else
      if let some wake_time := p.wake_time
        -- blocked by sleep
        if wake_time <= T
          -- p can wake up and run
          p.wake_time = none
          Q.drain non_runnable
          return some p
        else
          -- sleeping but can't wake up... move to S!
          S.insert p
      else
        -- blocked by some other reason
        non_runnable.insert p
  -- no return so far so no process is runnable
  Q.drain non_runnable
  none
```

## The idle process

The idle process **must** be the last option to be picked because, according to Jeremy,

> If the idle process is picked when another process could potentially run, the CPU will
sleep waiting for interrupts instead of doing useful work.

To account for this, we make the idle process always have lower priority when compared with
other processes, as if it had an infinite VD. This is enough to push it to the end of the
queue.

## Validating if the algorithm is functioning correctly

This can be done by debug logging and then parsing the logs to assert that everything is
working as expected. The assertions would need to, somehow, capture the scheduling algorithm.
So in principle it's equivalent to having a second implementation that must match with what
the kernel is actually doing.

## Pinning processes to a core

There can be a third set of processes that are **p**inned to cores, `P`, with quick lookup
properties. When it's time to switch, check whether the current process is in `P` or not.
If it is, don't touch it.

For extra efficiency, a process in `P` should not be kept in `Q` because it doesn't make sense
to keep on popping a process that's already running on some core.

## Reserved cores

A set of reserved cores can be maintained and the scheduler needs to check whether cores are
available when computing the `can_run` function from the algorithm presented above.

## Ideas for heuristics to be tried later

These are not part of the algorithm per se, but can be experimented to see if the overall
performance improves:

* Attempt to distribute tasks over efficiency cores vs performance cores
* Decrease swapping by linking/chaining tasks that share resources
* Preempt driver/scheme/service tasks, followed by their originators
* Set the VD of processes marked to be terminated to zero, pulling them to the top of `Q`

# Drawbacks
[drawbacks]: #drawbacks

* Code complexity will increase, but it's expected as the current scheduler is deemed
too simple
* There isn't as much time backing off the performance of EEVDF in the wild, since it's
a reasonably new addition to the Linux kernel

# Alternatives
[alternatives]: #alternatives

CFS has been mentioned as a valid alternative, but I don't think there's much of
a drawback for not doing it (now) because this is not a one-way door. In fact,
if/when we arrive at a point where it's possible to objectively measure performances
of different schedulers, there's no reason why we shouldn't implement both (and
even more) algorithms for comparison purposes.

More alternatives (open to collaboration!)
* Minix
* Plan 9

# Unresolved questions
[unresolved]: #unresolved-questions

There's basically one unresolved question, which I believe more seasoned Redox developers
should be more capable of answering: *how to define the weights for processes?*

## How to define the weights for processes?

I believe more seasoned Redox developers should be more capable of answering this question. But
I don't think we need to take too many turns for a first version, as this is also something that
can be tuned later once we have a method to measure the performance impact of changes on the
scheduler. Also, just by not punishing processes that yield early and processes that sleep, the
proposed algorithm should already have better performance than what's currently implemented, even
if all weights are set to `1`.

## Measuring performance

This probably deserves a whole separate RFC. Ideally, we'd be able to stress the system in a way
that's representative of real scenarios in which Redox would go through.
