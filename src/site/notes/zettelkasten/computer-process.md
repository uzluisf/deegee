---
{"dg-publish":true,"permalink":"/zettelkasten/computer-process/"}
---


Informally, a **computer process** is defined as a **running program**. However the program itself, be it a shell program or a binary, simply sits in memory, thus it's a lifeless thing that contains instructions and static data. It isn't until the operating system takes the bytes representing the program and gets them running that the program becomes useful.

More often than not, a computer may be seemingly running tens or hundreds of processes at the same time, even despite the fact the computer only has a few physical CPUs. Thus the [[zettelkasten/operating-system\|operating system]] creates the illusion of virtually infinite number of CPUs by [[zettelkasten/computer-virtualization\|virtualizing]] the available physical CPUs. In order other words, the OS takes a process, runs it for some time, then stop it, and run another process, and it does this back and forth until a process has terminated. This basic technique of allowing users to run as many concurrent processes as they'd like is known as [[zettelkasten/time-sharing\|time sharing]].

To implement virtualization of the CPU, the OS needs:
* **Low-level machinery** or **mechanisms**, which are low-level methods or protocols that implement a needed piece of functionality. For example, a [[zettelkasten/context-switch\|context switch]] is an example of a mechanism that allows the OS to stop running one program and start running another on a CPU. 
    * Think of a mechanism as providing the answer to a *how* question, e.g., *how* does an operating system perform a [[zettelkasten/context-switch\|context switch]]? 
* **High-level intelligence** that takes the form of **policies** which are algorithms for making some kind of decision within the OS. For example, questions like "given a number of programs that can run on a CPU, which program should the CPU run?" are answered by a [[zettelkasten/scheduling-policy\|scheduling policy]]. 
    * Think a policy as providing the answer to a *which* question, e.g., *which* process should the operating system run right now?

---
* [[zettelkasten/computer-process-abstracted\|computer-process-abstracted]]
* [[zettelkasten/high-level-process-api\|high-level-process-api]]
* [[zettelkasten/process-states\|process-states]]
* [[zettelkasten/data-structures-in-process\|data-structures-in-process]]