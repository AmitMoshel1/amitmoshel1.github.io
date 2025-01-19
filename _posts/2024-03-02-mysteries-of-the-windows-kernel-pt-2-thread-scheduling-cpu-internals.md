---
layout: post
title: Mysteries of the Windows Kernel Pt.2 - Thread Scheduling & CPU Internals
date: 2024-03-02 22:14 +0200
---

# Understanding Threads and CPU Scheduling

In this article, we’ll explore what a thread is, its components, how threads relate to the CPU, and how they are scheduled when running on CPU cores.

---

## What is a Thread?

A **thread** is a unit of execution that operates under a certain process. It is considered a **Dispatcher Kernel Object** (also known as a **"Waitable Object"**, which we’ll explore in depth later).

A thread primarily consists of the following components:
- **Access Token**: Can be inherited from the process or impersonate another user's token.

- **Priority**: Includes **Base Priority** and **Dynamic Priority** (explained later).

- **Context**: Also explained later.

- **State**: Could be one of the following - **Waiting**, **Running**, **Ready**, **Standby**.

- **Affinity**

To understand each of these components more in depth, let’s understand the main components of the CPU:

![img-description](https://miro.medium.com/v2/resize:fit:720/format:webp/1*pOKS4s67jByRsoA_zYg3ww.png)

[CPU Core vs Threads](https://www.temok.com/blog/cores-vs-threads/)

In the image above, the right architecture represents the concept of **Hyper-Threading**.

---

## Understanding CPU Components

The CPU resides on a **socket**, a component within the motherboard. The CPU is primarily composed of **cores**, each of which is a unit of execution. Initially, each core was a single execution unit. However, with advancements in technology, Intel developed **Hyper-Threading**.

**“Hyper Threading”** is a technology that was developed by intel and allows a single core to be divided into 2 execution units, each called a **“Logical Processor”** (also known as **“Hardware Threads”**). In the image above, the architecture shown on the right displays the idea of **“Hyper Threading”**.

If we open **Task Manager** and navigate to the **Performance** tab, the **CPU** section shows graphs for each Logical Processor.

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*CVoJRWUZYJiTzn490AaJzQ.png)

 At the bottom, it lists:
- **Number of Sockets** (e.g., 1 in this case)
- **Number of CPU cores** (e.g., 4 cores)
- **Number of Logical Processors** (e.g., 8 Logical Processors, indicating Hyper-Threading support)

The more Logical Processors available, the more threads can be in the **Running** state simultaneously.

---

## Thread States

Each thread has 3 primary states:
1. **Running**: The thread is actively executing within a Logical Processor.

2. **Ready**: A state where a thread is ready to be executed within one of the **“Logical Processors”**, but at this point and time it’s not in a high priority or the **“Quantum”** of the other thread that has the same priority didn’t finish yet. I’ll explain what a **“Quantum”** is soon.

3. **Waiting**: The thread is waiting for a signal from a **Dispatcher Kernel Object** (or **Waitable Object**). A **“Dispatcher Kernel Object”**, also called a **“Waitable Object”** is an object in which a thread can be in a “Waiting State” for until a signal receives from another object. For example, a thread can halt it’s execution and be in a **“Waiting”** state until another **“thread”** (which is a **“Waitable Kernel Object”)** signals. The meaning of a signal changes between different kernel objects

---

## Thread Prioritization

The **Kernel Scheduler** in Windows prioritizes threads using 2 key components:
- **Base Priority**: Determined by the process's Priority Class (modifiable via the `SetPriorityClass()` WinAPI function).
  
- **Dynamic Priority**: Adjusted dynamically at runtime.

### Priority Classes and Base Priority
- **Priority 0**: Reserved for the **Zero Page Thread**.
- **Low**: Priority **4**
- **Below Normal**: Priority **6**
- **Normal**: Priority **8**
- **Above Normal**: Priority **10**
- **High**: Priority **13**
- **Real Time**: Priority **24**

### Dynamic Priority Adjustments
- **Idle**: Priority set to 1.
- **Lowest**: 2 levels below the Base Priority.
- **Below Normal**: 1 level below the Base Priority.
- **Normal**: Matches the Base Priority.
- **Above Normal**: 1 level above the Base Priority.
- **Highest**: 2 levels above the Base Priority.
- **Time Critical**: Raised to Priority 31 (highest).

Using tools like **Process Explorer**, we can view very interesting information, such as:
- **Thread’s state** (Waiting, Running, Ready)
- **Wait Reason** — Relevant for threads that are in the “Waiting” state, it gives a brief description for what the thread is in a “Waiting” state for.
- **TID** — Thread ID.
- **CPU Time** — Time of execution within the CPU.
- **Base Pri** — The **“Base Priority”** of the thread.
- **Dyn Pri** — The **“Dynamic Priority”** of the thread.


![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*ZB6-r3Tvexu_NqDBHpvljQ.png)

---

## Single-Core Scheduling

On single-core systems, threads in the **Ready** state are organized in a **Ready Queue** and prioritized. The highest-priority thread gets execution time.

But what happens if there are multiple threads in the same priority and wants to execute? But there is only 1 processor available on the CPU?
Here’s where things get interesting.

### Managing Equal-Priority Threads
In a case where there are multiple threads that have the same priority on a single-core CPU, then each thread receives a slice of execution time, called a **Quantum**. On client machines, the Quantum is typically **31ms** (2 slices of 15.625ms).

### Reasons for Context Switching
Threads stop execution and undergo a **Context Switch** in 3 scenarios:
1. The thread’s Quantum expires.
   
2. A higher-priority thread becomes Ready.

3. The thread enters a Waiting state (e.g., waiting for a signal).
   

---

## Viewing Thread Information in WinDbg

Using the command **`!process 0 7 lsass.exe`** in WinDbg reveals detailed information about threads in the `lsass.exe` process:

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*0P7WHWU5Lev17UFcqeMHQA.png)

 Key details include:
- **THREAD ffff9c8b090b4080**: Address of the thread's **ETHREAD** structure.

- **WAIT**: Current state (e.g., Waiting) and the object it waits on.

- **Context Switch Count**: Number of context switches since thread creation.

- **Priority**: Current **Dynamic Priority**.

- **Base Priority**: The thread's **Base Priority**.

- **UserTime**: Execution time in **User Mode**.

- **KernelTime**: Execution time in **Kernel Mode**.


---

## Multiprocessor Scheduling

### Affinity
Each process has an attribute called **Affinity** mask. **Affinity** is a 64-bit mask value defining the **Logical Processors** the process can execute on:
- **Soft Affinity**: Default, allowing execution on **all** **Logical Processors**.
  
- **Hard Affinity**: Restricts execution to **specific** **Logical Processors** (rarely used).

To view the **“Affinity”** of a process within **“Process Explorer”**, open **“Process Explorer”**, right click on some process and click on **“Set Affinity”**:

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*ekWehoPopBJhWfBkFzaqig.png)

We can see that we have a list of a total of **64 “Logical Processors”** (only 19 are marked and relevant in this case because this was captured on a system that has 19 LPs (**“Logical Processors”**)), and we can see that the process is allowed to run on all of the LPs available on the system, which means it’s **Affinity** is a **“Soft Affinity”**.

For a system that has more than 64 processors, each **64 LPs** on the system are divided into groups and to express that, there is another attribute appended in the **“Affinity”** which is called **“Processor Group”**. For example, on a system that has **200 LPs**, the first **64 LPs** are related to **“Processor Group 0”**, the second **64 LPs** are related to **“Processor Group 1"**, the third **64 LPs** are related to **“Processor Group 3”**, and the remaining LPs are related to **“Processor Group 4”**.

To interact with Affinity, we can use the WINAPI functions such as:
- `SetThreadAffinityMask()`
- `SetProcessAffintyMask()`
- `GetProcessAffinityMask()`

### Ideal Processor

Another attribute which is important for multiprocessor thread scheduling is **“Ideal Processor”**.

**“Ideal Processor”** is a field that each thread has and it refers to a processor in which the thread’s execution would be ideal. The concept of **ideal processor** is one of the criterias that the scheduler takes into consideration when performing **multiprocessor scheduling**.

---

## CPU Caching

To optimize execution and reduce RAM interactions, CPUs utilize 3 cache levels:
1. **L1 Cache**
2. **L2 Cache**
3. **L3 Cache**

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*nW3odKhNNkU7PZpgEv7_Ug.png)
[L1 L2 L3 Caches](https://www.hardwaretimes.com/difference-between-l1-l2-and-l3-cache-how-does-cpu-cache-work/)

The idea behind these caches is to try to minimize interaction with the RAM during execution in order to optimize and increase execution speed at processing time.

- **L1 Cache** — The smallest and fastest cache that each core has. It’s divided into 2 types of caching slots within it: **Instruction Cache** (**I Cache**), **Data Cache** (**D Cache)**

- **L2 Cache** — A bigger cache but slower, every core has 1 **L2 cache**. This means that if a core supports **“Hyper-Threading”** and has 2 LPs within, the 2 LPs within the core will share the same **L2 cache**.

- **L3 Cache** — The biggest and slowest cache. There is only 1 L3 Cache within the CPU and it’s being shared by all of the cores within the CPU.

Before getting to the idea of “Multiprocessor scheduling” I have to mention that the multiprocessor scheduling scheme is a base scheme and doesn’t include the idea of **“Asymmetric Multiprocessing”**, which may change the scheduling scheme and make it a bit more complicated.

The idea of **“Asymmetric Multiprocessing”** is that not all the cores within the CPU are being treated equally and some of the processors might be used for a specific purposes.

After having that out of the way, let’s see how multiprocessor scheduling works on **“Symmetric Multiprocessing”**.

First, the **“scheduler”** will first check if there is any **“Logical Processor”** available on the system. If there is, the following checks will be performed by order:
- The first check will be if the **“Logical Processor”** in which the thread was previously running on is available, the reason for that is because there might be some cache related to the thread. If so, the **“scheduler”** will assign the **“Logical Processor”** to the thread.

- The second check is if the **“Ideal Processor”** that is assigned to the thread is available. If so, the **“scheduler”** will assign the **“Logical Processor”** to the thread.

- If both of the checks weren’t available, the **“scheduler”** will find the first available **“Logical Processor”** and run on it.

If there isn’t any **“Logical Processor”** that’s available on the CPU, the “scheduler” will check to see if there is any **“Logical Processor”** that runs a thread with a lower priority, if so, a **“Context Switch”** will occur, and the **“Logical Processor”** will be assigned to the thread. If not, the thread will continue to reside in a **“Ready Queue”** and wait until one of the scheduling conditions will match.

in the next article I’ll touch more about the “Memory Manager” component of the kernel, interaction between **“Virtual Memory”** and **“Physical Memory”** and how the “Memory Manager” manages memory within the system.
