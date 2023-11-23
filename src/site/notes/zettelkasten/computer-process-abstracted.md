---
{"dg-publish":true,"permalink":"/zettelkasten/computer-process-abstracted/"}
---


```ad-tldr
* The **process** is the major OS abstraction of a running program. At any point in time, the process can be described by its state: the contents of memory in its address space, the contents of CPU registers, and information about I/O.
* The process API consists of calls that programs can make related to processes, such as process creation and destruction.
* Processes exist in one of many different process states, including running, ready to run, and blocked.
* When the OS creates a process, it loads the proggram from memory, allocates memory for both the stack and the heap, and does some initialization tasks such as opening the default three file descriptors, i.e., stdin, stdout, and stderr.
* A **process list** contains information about all processes in the system. Each entry is found in a **process control block** (PCB), which is a data structure that contains information about a specific process.
```

More abstractly, we define a **process** as running program with all the different pieces of the system it accesses or affects while it's executing. Henceforth, a program is simply the instructions, sitting in memory, that can potentially do something while a process is those instructions running.

A process can be in different states, which can be represented as a [[zettelkasten/state-machine\|state machine]], i.e., what a process can read or update when it's running. Some components of machine state that comprises a process are:
* **[[zettelkasten/computing-memory\|Memory]]**, where the program's instructions and the data the process reads and writes lie. Thus the memory that the process can address, called its [[zettelkasten/address-space\|address space]], is part of the process.
* **[[zettelkasten/compute-register\|Registers]]**, which are the locations in memory that many instructions explicitly read or update. Some special registers that are part of this machine state are:
	* **program counter** (PC), which tells us which instruction of the program will execute next.
	* **stack pointer** and **frame pointer**, which are used to manage the stack for function parameters, local variables, and return addresses.
* **[[zettelkasten/disk\|disk]]**, since processes usually access [[zettelkasten/persistent-storage-device\|persistent storage devices]]. Such [[zettelkasten/input-and-output\|IO]] information might include a list of the files the process currently has open.