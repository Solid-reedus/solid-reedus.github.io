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
- the perforamnce gain it can add to your engine and other big applications that use a lot of performance.
- the synergy between job systems and ECS
- the architecture of a job system and the bits and pieces its made up of
- the gains and result of a job system had on my engine.

<br>

---

## Context & Background

### Why Is This Problem Important?

Over the past decade, CPU clock speeds have largely stopped increasing, while the number of available cores on consumer hardware has steadily grown. This means that modern performance gains no longer come from faster single-threaded code, but from using **multiple cores effectively**.

For engine programmers, this changes the problem entirely. Writing highly optimized sequential code is no longer enough. To take advantage of modern hardware, systems must be designed with **parallel execution in mind from the start**, rather than retrofitted later.

Games with simulation-heavy workloads are particularly affected. Updating large numbers of entities every frame quickly becomes a bottleneck, even when using data-oriented techniques like ECS. Movement logic, AI decisions, formation updates, and combat behavior all add up, and running them one after another does not scale.

A job system addresses this by breaking work into smaller units that can be executed concurrently. Instead of thinking in terms of “systems running one after another,” the engine can think in terms of **work that is allowed to happen at the same time**, with synchronization only where it is actually needed.

This blog is written for fellow students who are interested in:
- Engine and tools programming
- Multithreading and performance
- ECS-based, data-oriented design

The goal is not to present a production-ready solution, but to show how these ideas come together in practice, what problems arise, and how they can be approached.


<br>

---

## What Is a Job System?

A job system is a concurrency framework that schedules small units of work, called *jobs*, across a pool of worker threads. Instead of assigning long-running systems to dedicated threads, work is split into smaller pieces and distributed dynamically. This allows the CPU to stay busy and scale with the number of available cores.

A central idea behind most job systems is that threads should only be idle when there is truly no work left to do. In my implementation, the execution model is cooperative. When a worker finishes its own jobs, it actively looks for other work to help with instead of immediately sleeping or yielding.

> *A simple metaphor for this cooperative execution model:*  
> ![apes together strong - threads working together metaphor](/media/apes_together_strong.jpg)

This approach can produce large performance gains, especially for workloads that can be split into many independent tasks. However, it also introduces complexity. Synchronization becomes harder, debugging gets more difficult, and careless design can easily lead to stalls or unpredictable behavior.

Because of this, a major design goal of my job system was **explicitness**. Parallel work, dependencies, and synchronization points should be visible in the code, not hidden behind abstractions. This makes the system slightly more demanding to use, but much easier to reason about and debug.


<br>

---

## What I Built

The result of this project is a job system designed to handle both **many small tasks** and **large data-parallel workloads**, while remaining explicit and predictable to use in an engine context.

The system supports:
- Task parallelization through lightweight jobs
- Explicit dependency management, avoiding hidden synchronization
- Data-parallel ECS processing using chunked iteration
- Cooperative busy waiting to keep workers productive
- Work stealing to improve load balancing under uneven workloads

To validate the design, I built a Total War–style simulation with thousands of units executing formation logic, movement, and behavior updates in parallel. This created a deliberately mixed workload: some jobs are small and cheap, others are heavier and more expensive.

This kind of workload is challenging for many schedulers and makes a good stress test. It exposes problems in load balancing, scheduling fairness, synchronization overhead, and frame-time stability. Rather than optimizing for an ideal case, the system was tested under conditions that resemble real gameplay scenarios.


<br>

---

### Architecture Overview

There are several common ways to approach the design of a job system. At a high level, these include:
- Dedicated threads per system
- Thread pools
- Fiber-based schedulers

For this project, I chose a **thread pool–based design**. It sits in a practical middle ground: powerful enough to demonstrate real performance gains, but not so complex that the system becomes difficult to understand or debug.

#### What Is a Thread Pool?

A thread pool is a fixed collection of worker threads managed by the job system. Instead of creating and destroying threads dynamically, the system feeds work to these workers as jobs become available.

A typical thread pool consists of:
- Worker threads
- A representation of jobs to execute
- One or more queues used to distribute work

The diagram below shows a simplified visualization of a thread pool:

![thread pool](/media/thread_pools.png)  
<sup>Source: https://jenkov.com/tutorials/java-concurrency/thread-pools.html</sup>

In my implementation, each worker owns a **local job queue** rather than relying on a single global queue. This design enables **work stealing**. When a worker runs out of local work, it attempts to steal jobs from another worker’s queue.

Work stealing helps reduce idle time and improves performance under uneven workloads, which are common in real simulations where not all jobs take the same amount of time to execute.

![work stealing](/media/work_stealing_diagram.png)  
<sub>Source: https://actor-framework.readthedocs.io/en/stable/core/Scheduler.html</sub>

To coordinate execution across workers, the system tracks outstanding work using an **atomic job counter**. Explicit synchronization points, such as `WaitAll()`, allow the user to define when parallel execution must converge back to a known state.

This approach makes synchronization visible and predictable. Instead of hidden barriers or implicit waits, the programmer can clearly see where parallel execution ends, which makes both performance behavior and debugging easier to reason about.


---


### Job Representation

At the lowest level, a job represents a **single unit of executable work** along with the metadata required to schedule and synchronize it safely.

In my implementation, a job consists of:

```cpp
    struct Job
    {
        Job(const std::function<void()>& p_func) : func(p_func) {}

        std::function<void()> func;

        std::atomic<int> dependencies{0};
        std::vector<std::shared_ptr<Job>> dependents;

        std::atomic<bool> finished{false};
    };
```


Each job contains:

- A **callable function** representing the work to execute  
- An **atomic dependency counter** indicating how many prerequisite jobs must finish before this job becomes runnable  
- A list of **dependent jobs** that should be notified when this job completes  
- A **completion flag**, used for synchronization and debugging

This structure keeps jobs small and self-contained. Instead of relying on a global dependency graph or centralized scheduler state, dependency resolution happens locally by decrementing counters when jobs complete.

The design deliberately avoids heavy locking and global coordination. While this limits some forms of dependency expression, it keeps scheduling overhead low and makes the system easier to reason about under load.


---

### Job Creation and Submission

Jobs are created, linked, and submitted **explicitly** by the user. Rather than inferring execution order or attempting to automatically detect dependencies, the job system requires these relationships to be stated directly.

A typical usage flow looks like this:

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

In this setup, one job is declared as depending on the completion of another. The dependency is enforced through the job’s dependency counter, ensuring the correct execution order without blocking worker threads unnecessarily.

This approach is intentionally explicit:

- The job system does **not** infer dependencies
- Synchronization points such as `WaitAll()` are clearly visible
- Execution order and synchronization costs are predictable

While this places more responsibility on the programmer, it avoids hidden synchronization and implicit barriers. For an engine-level tool, this explicitness helps make performance behavior easier to reason about and debug.


---

### ECS Integration with `ParallelFor`

#### ECS in a Nutshell

An Entity Component System (ECS) separates **data** from **behavior**. Entities are identifiers, components are plain data, and systems operate over views of components. This layout leads to contiguous memory access and predictable iteration patterns, making ECS well suited for data-parallel execution.

In practice, ECS already organizes data in a way that a job system can take advantage of.

---

To integrate ECS workloads into the job system, I implemented a `ParallelFor` abstraction. This function takes an ECS view, splits it into chunks, and schedules each chunk as a separate job.

At a high level, `ParallelFor` performs the following steps:

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

        auto job = CreateJob([&, begin, end]() 
        {
            for (int j = begin; j < end; ++j)
            {
                func(view, entities[j]);
            }
        });

        Submit(job);
    }

    WaitAll();
}
```

From the user’s perspective, the change is minimal.

**Before:**  
```cpp
for (auto [entity, foo] : fooView.each())
{
    foo.val++;
}
```

**After:**  
```cpp
jobSystem.ParallelFor(fooView, [](auto& view, auto entity)
{
    auto& foo = view.get<Foo>(entity);
    foo.val++;
});
```

The structure of the ECS update remains familiar. The main difference is that iteration is now executed in parallel, with chunking and scheduling handled by the job system.

This design choice was intentional. By keeping the API close to standard ECS iteration patterns, the job system can be introduced gradually without requiring large refactors or deep multithreading knowledge from the user.


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

To evaluate the job system under realistic conditions, I built a **Total War–style simulation** featuring thousands to over a million entities executing mixed workloads. These workloads included movement updates, formation logic, and behavior processing all with uneven execution times.

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

Designing a job system involves a series of trade-offs, and this project is no exception.

One major choice was favoring **explicit dependency management** over automatic safety. The system does not try to infer which jobs are safe to run in parallel. Instead, the user must define dependencies manually. This increases responsibility for the programmer, but avoids hidden synchronization and unpredictable stalls.

Another trade-off is the use of **explicit synchronization boundaries**. Calls like `WaitAll()` must be placed deliberately. This can be error-prone if misused, but it makes execution order and performance costs visible, which is valuable in engine-level code.

Finally, the system intentionally avoids heavy abstraction. While higher-level abstractions can make APIs easier to use, they often hide costs and make profiling more difficult. In this project, clarity and predictability were prioritized over maximum convenience.

These choices make the system less forgiving, but they align well with the goal of understanding and controlling performance characteristics.


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

