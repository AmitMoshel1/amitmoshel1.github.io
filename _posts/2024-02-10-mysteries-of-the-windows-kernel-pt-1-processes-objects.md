---
layout: post
title: Mysteries of the Windows Kernel Pt.1 - Processes & Objects
date: 2024-02-10 22:14 +0200
categories: [mysteries-of-the-windows-kernel]
tags: [Reverse-Engineering, Security-Research, Kernel-Mode, Windows-Internals, WinDbg, IDA-Pro, Processes]
---

Hello everyone,

I’ve decided to start a series of Windows Internals articles which, at the beginning, will not directly relate to security. After covering some important Windows Internals topics, I’ll start connecting the material to aspects related to security. During the course, I’ll demonstrate practical aspects using **WinDbg** (kernel debugging) and **Process Explorer**.

## Prerequisites

Before starting, I highly recommend having some basic user-mode understanding of Windows Internals, including:

- Processes, threads, subsystem DLLs, and virtual address space.
- Basic knowledge of user-mode debugging with WinDbg.
- Basic knowledge of C/C++ (especially structures).

## Windows User Mode and Kernel Mode Flow of Execution

Before discussing Windows processes, let’s introduce the user-mode and kernel-mode flow of execution:

![img-description](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ctlcv_pdBA78bidKRQKrGQ.png)

[Architecture of Windows NT](https://www.wikiwand.com/en/Architecture_of_Windows_NT)

From the kernel-mode perspective, the main component of interest is the **Executive Subsystem**. This subsystem (located in `C:\Windows\System32\ntoskrnl.exe`) is the main layer of the kernel, and it comprises several managers, such as:

- **Memory Manager**
- **Process Manager**
- **I/O Manager**
- **Power Manager**
- **Object Manager**

Each manager is responsible for managing different important operations within the Windows kernel and operates through a set of kernel-mode functions.

### Function Prefixes

To identify which manager a function belongs to, the function names are prefixed as follows:

- **Ob** — Object Manager function.
- **Ps** — Process Manager function.
- **Io** — I/O Manager function.
- **Mm** — Memory Manager function.

### Kernel Objects

The kernel defines its main components as **Kernel Objects**. The best definition I found for a kernel object is a **"runtime instance of a static structure."** Examples of kernel objects include:

- **Processes**
- **Threads**
- **Mutexes**
- **Semaphores**
- **Jobs**
- **Events**
- **Registry Keys**
- **Files**

To track these components at runtime, the kernel uses structures called **Kernel Objects**. Most kernel objects have two types of structures:

- **Executive Subsystem structure:** Prefixed with `E` (e.g., `EPROCESS` for processes).
- **Kernel Component structure:** A smaller, lower-level structure (e.g., `KPROCESS` for processes).

### User Mode Access to Kernel Objects

Kernel objects cannot be directly accessed from user mode. Instead, they are accessed via **handles**, which allow interaction with kernel objects from user mode.

## The "Process" Object

A process mainly consists of the following components:

- **Virtual Address Space**
- **Access Token**
- **Threads**
- **Handle Table**
- **Virtual Address Descriptors (VADs)**

### Virtual Address Space

Every process has its own virtual address space, which includes:

- The PE image of the executed binary.
- DLLs required for correct execution.
- Structures such as **PEB** (Process Environment Block) and **TEB** (Thread Environment Block).

The size of the address space varies between architectures:

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*qp-DrbJ0_V9mJ8zobafFLQ.png)

- **32-bit architecture:**
  - Each process has 2 GB of user-mode address space (4 GB if the `LARGEADDRESSAWARE` attribute is enabled).
  - Kernel-Mode has 2 GB of kernel space.
- **64-bit architecture:**
  - Each process has 128 TB of user-mode address space.
  - Kernel-Mode has 128 TB of kernel space.

For more details, refer to [Windows Kernel Programming - Second Edition](https://leanpub.com/windowskernelprogrammingsecondedition).

### Access Token

The **Access Token** is used by the Operating System to determine the process’s level of privilege and access to Windows resources. Every process has a main access token, which threads inherit during execution. Threads may use a different access token via **Token Impersonation**.

### Threads

Threads are the units of execution within a process. A process must have at least one thread to execute correctly; otherwise, it will terminate.

### Virtual Address Descriptors (VADs)

The **Memory Manager** maintains a hierarchical tree of **Virtual Address Descriptors (VADs)** to track virtual address space ranges allocated to a process. VADs store information such as:

- **Protection** (Execute, Read, Write)
- **Allocated address ranges**
- **State** (**Committed**/**Reserved**)
- **Memory type**

## Viewing Processes in Kernel Debugging with WinDbg

### Setting Up Local Kernel Debugging

If you don’t have the **Local Kernel Debugging** feature set up, follow the instructions in the [official documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-local-kernel-debugging-of-a-single-computer-manually).

1. Open WinDbg.
2. Navigate to **File** → **Attach To Kernel** → Press **OK**.

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*CXq59Tva8D6j3vV2NFKtKQ.png)

You’ll see a command line with the `lkd` prompt, indicating **local kernel debugging mode**.

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*sPfCVhP8w9GTPpIcLGpFmg.png)

### Displaying Processes

Unlike user mode debugging, in kernel mode debugging we’re debugging the whole system kernel space and not a specific process.

Let’s start by getting a general information about the whole processes running in the system, and to do that, we’ll execute the command:

```cmd
!process 0 0
```

![img-description](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*4X78X4gV5d-NU5tiB20fYQ.png)

This command is an extension command, which displays general information about all of the running processes within the system.

* The first `0` displays information about all running processes.

* The second `0` controls the level of detail (ranges from 0 to 7).

#### Example: Viewing `lsass.exe`

Let’s take the `lsass.exe` process as an example. The output might include:

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*nh2JxvJs7WO56JkE091Yxg.png)

- **`PROCESS ffffaf874da020c0`** — The `EPROCESS` kernel address for `lsass.exe`.

- **`SessionId`** — The session in which the process runs (e.g., Session 0).

- **`Cid`** — Client ID (Process ID).

- **`Peb`** — A link to the user-mode PEB structure.

- **`ParentCid`** — Parent Process ID.

- **`Dirbase`** — Used by the Memory Manager for address translation.

- **`ObjectTable`** — Address of the process’s handle table (`_HANDLE_TABLE` structure).

- **`HandleCount`** — Total number of handles to kernel objects.
  
- **`Image`** — The PE image the process runs.

To get more detailed information, increase the detail level to 7:

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*u5GtyNJtED12PAGmlEEm6Q.png)

Now we can see the process’ threads information.
Let’s take the first thread and break it down:
- **`THREAD`** -  This the **“ETHREAD”** structure of the thread object. The **“ETHREAD”** structure will be discuessed in the next article.

- **`Cid`** — **Client ID,** in this case it holds 2 values separated by a dot (“.”) The first value **“0468”** is the hexadecimal representation of the **PID** in which the thread is under. The second value **“047c”** is the hexadecimal representation of the **TID** (**Thread ID**) of the thread that is under that process.
  
- **`Teb`** — the **user mode** address for the **“Thread Environment Block”** of a thread.
- **`WAIT`** — it means that the thread is in a **“Waiting”** state for a Semaphore object + the address of the semaphore object within kernel space. I’ll explain about thread’s states more in depth in the next article.
- **`UserTime`** — Amount of thread’s execution Time within **“User Mode”**.
- **`KernelTime`** — Amount of thread’s execution time within **“Kernel Mode”**.
  
- **`Priority 9`** — The **“Dynamic Priority”** of a thread, will be explained in the next article.
 
- **`Base Priority`** — The **“Base Priority”** of a thread, will also be explained in the next article.
  
- **`Base`** — The **kernel thread’s** **stack base**.

- **`Limit`**  — The **kernel thread’s** **stack limit**.

Another interesting command is:

```cmd
.process /r /p ffffaf874da020c0; !process ffffaf874da020c0 7
```

This command switches the debugger’s context to the specified process, reloads user-mode symbols (`/r`), and displays the full call stack (kernel and user mode).

### Viewing Raw EPROCESS Structure

The **`!process`** is an extension command which prettifies the information shown in the **`_EPROCESS`** structure within the executive subsystem.

If we want to view the raw structure of the **`_EPROCESS`** of the **`lsass.exe`**, we’ll execute the following command:
“dt nt!_eprocess ffffaf874da020c0“

```cmd
dt nt!_eprocess ffffaf874da020c0
```

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*orUSeH6zONBIH3R__2ZjhA.png)

This structure is much bigger than shown in the image and holds all of the information required for the process to execute properly within the system.

#### Key Fields in `_EPROCESS`

- **`ActiveProcessLinks`** — Doubly-linked list connecting all `EPROCESS` structures in the system. The head resides in `PsActiveProcessHead`.
- **`Protection`** — Process protection level (e.g., **Protected Process Light**).
- **`ObjectTable`** — Address of the process’s **handle table**.
- **`PEB`** — The Process Environment Block (user-mode structure).

To view the `PEB` structure, switch the debugger’s context:

```cmd
dt nt!_eprocess ffffaf874da020c0 Peb
.process /r /p ffffaf874da020c0; !peb 0x<PEB_Address>
```
![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*LUG3OEVjLBov_FaWSOLl9Q.png)

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*D4DRrWSRklSxyXO_z-YdDg.png)
---

Another very helpful thing to know is that if we have an address to some type of object but we’re not sure what type of kernel object it is, we can execute the **`!object`** extension command and specify the address to the kernel object.

Let’s try to take the **`EPROCESS`** address of the **`lsass.exe`** process we’re looking at and execute the **`!object`** command against that:

```cmd
“!object ffffaf874da020c0”
```
![img-description](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*_Rpsqe1PR3cXBhmj1-L5ng.png)

We can see that every process has an address to the **object type**, which is an address to an **`_OBJECT_TYPE`** structure.

Viewing the **`_OBJECT_TYPE`** structure of a **`Process`** object type by executing the command:

```cmd
“dt nt!_object_type ffffaf872f8dcbc0”
```

![img-description](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*7j22IQ-U1RWLWVODxNrFIw.png)

Let’s break down some of the important fields within this structure:
- **`TypeList`** - Doubly-linked list connecting all objects of this type.
  
- **`Name`** - Name of the object type, stored as a “UNICODE_STRING” structure.
  
- **`DefaultObject`** - Pointer to a default object of this type.

- **`Index`** - Index or identifier for the object type.
  
- **`TotalNumberOfObjects`** Total number of objects of this type currently in the system.

- **`TotalNumberOfHandles`** - Total number of handles to objects of this type currently in the system.

- **`HighWaterNumberOfObjects`** - Maximum number of objects of this type within the entire system.

- **`HighWaterNumberOfHandles`** - Maximum number of handles to objects of this type within the
entire system.

- **`Key`**: Unique identifier for this object type.

- **`CallbackList`** - Doubly linked List of callbacks registered for this object type. Callbacks are routines (functions) that are registered for a specific events that can occur during the object’s lifetime. For example, a callback routine can be registered to an event such process creation. This means that a certain routine can be executed when a process is being created.

We can also see that there is another address of **“objectHeader”**, which is also an address to an **`_OBJECT_HEADER`** structure. To view the **`_OBJECT_HEADER`** structure of that object we’ll execute the following command:

```cmd
dt nt!_OBJECT_HEADER ffffaf874da02090
```

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*SW3QRwyQAYXSc93LWTOQMQ.png)


Let’s explain the main fields within it:
- **`PointerCount`** — the amount of references to objects this process object has. Objects within kernel mode can be accessed through a pointer references to objects or handles. In user mode the only way to access kernel objects is through handles.
  
- **`HandleCount`** — The amount of handles to other objects this object has.
  
- **`SecurityDescriptor`** — Holds a pointer to the **security descriptor** associated with this kernel object. The **security descriptor** contains information such as the object’s owner, group, **`DACL`** (**Discretionary Access Control List**), and **`SACL`** (**System Access Control Lis**t).
 
We can view this information by using the **`!sd`** command and changing the last **`0xf`** flag of the pointer to **0**:

```cmd
!sd 0xffff8e89`1c0fe9a0
```

![img-description](https://miro.medium.com/v2/resize:fit:828/format:webp/1*XtxpKnLWId9T_0ubcNPL4w.png)


It’s important to note that the **`!object`** command works not only on **“Process”** objects but on every other **“Kernel Object”** available on the system.

---

That would be it for part 1 of the series of articles. Obviously this topic is much much larger to summarize in one article.
In the next part of the series I’ll touch more in depth about threads within the kernel and how the relation between threads and CPUs come into play within the windows.
