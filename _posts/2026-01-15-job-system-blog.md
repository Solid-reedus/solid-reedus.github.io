---
title: "Building a Fast Job System with ECS Integration"
date: 2026-01-16
description: "How I designed and implemented a multithreaded job system that scales with large workloads and integrates cleanly with ECS."
---


# Building a Fast Job System with ECS Integration

## Introduction

Over the last two decades, game engines have evolved from largely single-threaded designs into heavily multithreaded systems that leverage modern multi-core CPUs. While this shift enables significantly higher performance, it also introduces complexity in scheduling, synchronization, and debugging.

For this project, I set out to build a **job system** capable of handling large amounts of work efficiently, while integrating cleanly with an **Entity Component System (ECS)**. To showcase its capabilities, I built a **Total War–inspired simulation** where thousands of entities are updated in parallel.

In this blog post, I will cover:
- Why job systems are essential for modern engines
- How ECS fits naturally into data-parallel workloads
- The architecture and implementation of my job system
- Performance results, trade-offs, and lessons learned

---

## Context & Background


<!-- might change it back to old "Why Is This Problem Important?"? -->
### Why Is This Problem Important?

CPU clock speeds have largely plateaued, but the number of cores continues to increase. To fully utilize modern hardware, software must be **parallel by design**.

Games—especially simulations like RTS titles—often update thousands of entities every frame. Even with an ECS, a purely sequential update loop quickly becomes a bottleneck. To scale effectively, workloads must be split and scheduled across multiple cores.

A job system solves this by:
- Breaking work into small, schedulable units
- Dynamically balancing load across worker threads
- Allowing dependencies without serializing the entire frame

This blog is aimed at fellow students interested in:
- Engine and tools programming  
- Multithreading and performance  
- ECS-based data-oriented design  

---

## What Is a Job System?

A job system is a concurrency framework that schedules small units of work (jobs) across a pool of worker threads. Instead of dedicating threads to specific systems, jobs are dynamically distributed, improving utilization and scalability.

The execution model is **cooperative**: when a worker finishes its own work, it helps complete remaining jobs rather than going idle.

> *A metaphor for this cooperative execution model:*  
> ![apes together strong - cooperative threads metaphor](/media/apes_together_strong.jpg)

While cooperation enables impressive gains, it also introduces complexity in synchronization, correctness, and debugging—topics that strongly influenced the system’s design.

---

## What I Built

The result of this project is a job system designed to handle both **many small tasks** and **large data-parallel workloads**, while remaining explicit and predictable to use.

The system supports:
- **Task parallelization** via lightweight jobs
- **Explicit dependency management** for ordered execution
- **Data-parallel ECS processing** using chunked views
- **Cooperative busy waiting** to keep workers productive

To validate the system, I built a **Total War–style simulation** with thousands of units executing formation logic and behavior in parallel. This created a mixed workload with uneven execution times—an effective stress test for scheduling and load balancing.

---








## Implementation

### Architecture Overview

The job system is designed to keep all CPU cores busy without requiring the programmer to manually manage threads. Work is broken into small **jobs**, which are dynamically scheduled across a fixed pool of worker threads.

![thread pool](/media/thread_pools.png)
<!-- Source: https://jenkov.com/tutorials/java-concurrency/thread-pools.html -->

Each worker thread owns its **own job queue**, reducing contention compared to a single global queue. Workers primarily consume jobs from their local queue, which keeps common-case execution fast.

![work stealing](/media/work_stealing_diagram.png)
<!-- Source: https://actor-framework.readthedocs.io/en/stable/core/Scheduler.html -->

When a worker runs out of work, it attempts to **steal jobs** from another worker’s queue. This balances uneven workloads and prevents CPU cores from sitting idle.

An **atomic job counter** tracks outstanding work, and explicit synchronization points such as `WaitAll()` allow the user to define when parallel execution must converge. This keeps synchronization predictable and avoids hidden stalls.


---

### Job Representation

Each job is an explicit unit of work containing:
- A function to execute
- A dependency counter
- A list of dependent jobs
- A completion flag

This design avoids global locks and enables efficient dependency resolution.

---

### Job Creation and Submission

Jobs are created, linked, and submitted explicitly:

```cpp
auto job1 = Engine.JobSystem().CreateJob([]() 
{
    DoWorkA();
});

auto job2 = Engine.JobSystem().CreateJob([]() 
{
    DoWorkB();
});

Engine.JobSystem().AddDependency(job1, job2);

Engine.JobSystem().Submit(job1);
Engine.JobSystem().Submit(job2);
Engine.JobSystem().WaitAll();
```

This approach is intentionally explicit:

* The system does not infer dependencies
* Synchronization points are clearly defined
* Execution order is predictable

This mirrors real-world engine APIs and avoids hidden stalls.

---

### ECS Integration with `ParallelFor`

#### A Short ECS Refresher

An Entity Component System organizes data by components rather than object hierarchies. Entities are identifiers, components are data, and systems operate over views of components. This layout is ideal for **data-parallel processing**.

---

To integrate ECS workloads, I implemented a `ParallelFor` abstraction that splits an ECS view into jobs:

```cpp
template <typename View, typename Func>
void ParallelFor(View& view, Func&& func)
{
    std::vector<typename View::entity_type> entities;
    for (auto e : view) entities.push_back(e);

    const int total = static_cast<int>(entities.size());
    const int workers = static_cast<int>(m_workers.size());
    const int chunkSize = std::max(1, total / (workers * 2));

    for (int i = 0; i < total; i += chunkSize)
    {
        int begin = i;
        int end = std::min(i + chunkSize, total);

        auto job = CreateJob([&, begin, end]() {
            for (int j = begin; j < end; ++j)
                func(view, entities[j]);
        });

        Submit(job);
    }

    WaitAll();
}
```

**Before (synchronous ECS):**

```cpp
for (auto [entity, foo] : fooView.each())
{
    foo.val++;
}
```

**After (parallel ECS):**

```cpp
jobSystem.ParallelFor(fooView, [](auto& view, auto entity)
{
    auto& foo = view.get<Foo>(entity);
    foo.val++;
});
```

This enables parallel entity processing with minimal changes to existing ECS code.

---

### Synchronization Strategy

Synchronization is one of the hardest parts of any multithreaded system. Excessive locking can destroy performance, while insufficient synchronization leads to subtle and difficult-to-debug race conditions.

Rather than relying on traditional mutex-heavy designs, this job system uses a combination of:

- **Atomic counters** to track job completion and dependencies
- **Cooperative busy waiting** instead of blocking sleeps
- **Explicit synchronization points** via `WaitAll()`

The key idea is that workers should *help finish the frame*, not wait for it.

When a worker runs out of local work, it does not immediately sleep or yield the CPU. Instead, it actively searches for remaining jobs—either from its own queue or by stealing from others. This cooperative behavior significantly reduces idle time, especially in frames with uneven workloads.

Explicit synchronization is a deliberate design choice. Rather than hiding synchronization behind implicit barriers, the user must clearly define where parallel execution converges. While this places more responsibility on the programmer, it makes execution order and performance costs predictable—an important property in engine-level code.

In practice, this approach trades some ease of use for:
- Lower contention
- Fewer context switches
- More consistent frame times under load


---

## Results & Analysis

### Demo & Stress Testing

To evaluate the job system under realistic conditions, I built a **Total War–style simulation** featuring thousands to over a million entities executing mixed workloads. These workloads included movement updates, formation logic, and behavior processing—all with uneven execution times.

This setup was intentionally chosen to stress:
- Load balancing across worker threads
- Dependency resolution under pressure
- Scalability as entity count increases

[insert demo video here]

During profiling, several important behaviors emerged.

First, **frame times remained stable** as entity counts increased. While total CPU usage rose—as expected—the cost per entity decreased due to improved parallel efficiency.

Second, the system achieved **up to a 16× speedup** compared to equivalent synchronous execution. Interestingly, the largest gains appeared at higher workloads, where workers remained busy without frequently synchronizing.

Profiling also directly influenced design decisions:
- An early global job queue caused noticeable contention under stress → replaced with per-worker queues
- Very small chunk sizes increased scheduling overhead → chunk sizes were tuned empirically
- Larger batches improved cache locality and reduced atomic pressure

These observations reinforced a recurring theme: in multithreaded systems, *more work can sometimes mean better performance*, as overheads are amortized and workers remain productive.


---

### Trade-offs

Design choices included:

* Explicit dependency management instead of automatic safety
* Manual synchronization boundaries
* Minimal abstraction to avoid hidden costs

These favor performance and predictability over convenience.

---

## Conclusion & Reflection

### What I Learned

This project deepened my understanding of:

* Job scheduling and synchronization
* Data-oriented design
* Profiling-driven optimization

One key insight was that **larger workloads often perform better**, as workers remain busy without frequent synchronization.

---

### Future Work

With more time, I would explore:

* Priority-based job scheduling
* More advanced debug visualization
* Lock-free allocators for job data
* Data parallelism outside ECS contexts

---

### Further Reading

[this is still imcomplete]

* *Game Engine Architecture* — Jason Gregory
* *Parallel and High Performance Computing* — Hager & Wellein
* EnTT documentation
* GDC talks on job systems and engine architecture



---

### Links
[still need to add them]

<!--
* Code repository: *(link here)*
* Demo videos: *(link here)*
-->

---

## Final Thoughts

This project is not a replacement for production engine job systems, but rather a **stepping stone**—a way to explore how scalable multithreaded tools are designed and used.

Building this system gave me a much clearer understanding of how modern engines structure work and why parallelism is as much a design challenge as a technical one. Simply adding threads is rarely enough; shaping workloads and defining clear synchronization points matters far more.

While limited in scope, this project provided a strong foundation for thinking about performance, engine architecture, and multithreaded systems going forward.

