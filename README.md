# 4190.307 Operating Systems (Spring 2020)
# Project #5: Memory sharing across fork()
### Due: 11:59PM (Sunday), May 24

## Introduction

When the ``fork()`` system call is invoked, ``xv6`` simply copies all the memory used by the parent process to the child process. In this project, you will implement efficient memory sharing between the parent and child process. The goal of this project is to understand the paging hardware of the RISC-V processor and how the operating system manages virtual-to-physical address mapping to improve memory efficiency.

## Background

### RISC-V paging hardware

The RISC-V processor implements a standard paging with the page size of 4KiB. Especially, ``xv6`` runs on Sv39 RISC-V, which uses 39-bit virtual addresses with three-level page tables. When the paging is turned on in the supervisor mode, the ``satp`` register points to the physical address of the root page table. A page fault occurs when a process attemps to access a page with an invalid PTE (page table entry) or with an invalid access permission. 

 When a trap is taken into the supervisor mode, the ``scause`` register is written with a code indicating the event that caused the trap as shown in the following table.

|  Exception code | Description                     |
|:---------------:|:--------------------------------| 
| 0               | Instruction address misaligned  |
| 1               | Instruction access fault        |
| 2               | Illegal instruction             |
| 3               | Breakpoint                      |
| 4               | _Reserved_                      |
| 5               | Load access fault               |
| 6               | AMO address misaligned          |
| 7               | Store/AMO access fault          |
| 8               | Environment call (syscall)      |
| 9 - 11          | _Reserved_                      |
| 12              | __Instruction page fault__      |
| 13              | __Load page fault__             |
| 14              | _Reserved_                      |
| 15              | __Store page fault__        |
| >= 16           | _Reserved_                      |

Please pay attention to the following three events in the above table: Instruction page fault (12), Load page fault (13) and Store page fault (15). These events indicate the page faults caused by an instruction fetch, a load instruction, or a store instruction, respectively. On a page fault, RISC-V also provides the information on the _virtual_ address that caused the fault using the ``stval`` register. Currently, no page faults occur in ``xv6`` because all the required code and data are resident in the physical memory. However, you will need to handle these page faults in this project. Note that the value of the ``scause`` or ``stval`` registers can be read by calling the ``r_scause()`` or ``r_stval()`` function in ``xv6``, respectively. For more information on the RISC-V paging hardware, please refer to the [RISC-V Privileged Architecture Specification](http://csl.snu.ac.kr/courses/4190.307/2020-1/riscv-privileged-v1.10.pdf).


### Initializing process address space during ``exec()``

The ``exec()`` system call is used to replace the current process address space with a new process image from a file stored in the file system. All the executable files in ``xv6`` follow the standard [__Executable and Linkable File (ELF)__](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) format. An ELF binary consists of an ELF header followed by a sequence of program section headers. Each program header marked with ``LOAD`` indicates the section of the application that must be loaded into memory. The following example shows the program headers in the ``ls`` (``./user/ls.c``) program in ``xv6``. 

```
$ riscv64-unknown-elf-readelf -l ./user/_ls

Elf file type is EXEC (Executable file)
Entry point 0x27a
There is 1 program header, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000078 0x0000000000000000 0x0000000000000000
                 0x0000000000000a91 0x0000000000000ac0  RWE    0x8

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .sbss .bss
```

You can see that ``xv6`` programs use only one program section header where all the code (``.text``), read-only data (``.rodata``), and uninitialized data (``.sbss`` and ``.bss``) are stored. For more details, you are kindly invited to read Section 3.8 in the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2020-1/book-riscv-rev0.pdf).

Modern operating systems use separate program headers for code and data so that they can be mapped into different pages. This allows us to share code pages among the processes generated from the same executable file. This can be done by specifying the ``--no-omagic`` option (instead of ``-N``) in the linker flags (``LDFLAGS``). The following shows the program headers when you build the same ``ls`` program with this linker option:

```
$ riscv64-unknown-elf-ld -z max-page-size=4096 --no-omagic -e main -Ttext 0 -o user/_ls user/ls.o user/ulib.o user/usys.o user/printf.o user/umalloc.o
$ riscv64-unknown-elf-readelf -l ./user/_ls

Elf file type is EXEC (Executable file)
Entry point 0x27a
There are 2 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000001000 0x0000000000000000 0x0000000000000000
                 0x0000000000000a91 0x0000000000000a91  R E    0x1000
  LOAD           0x0000000000001a98 0x0000000000001a98 0x0000000000001a98
                 0x0000000000000000 0x0000000000000028  RW     0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata
   01     .sbss .bss
```

Now you can see that the executable file has two program headers, one for code and read-only data (``.text`` and ``.rodata``), and the other for data (``.sbss`` and ``.bss``). Also, the flag of the first segment is marked with ``RE`` (Readable and Executable), while that of the second segment is set to ``RW`` (Readable and Writable). During ``exec()``, we can load the content of the first segment into code pages with setting its permission to Read-only, and that of the second segment into another data pages with the Read/Write permission. Sounds complicated? Don't worry, we have already made the necessary changes in the skeleton code for you (see ``exec()`` and ``loadseg()`` at ``./kernel/exec.c``).

### What happens to memory during ``fork()`` in ``xv6``

The ``fork()`` system call creates a new (child) process by duplicating the calling (parent) process. The child process and the parent process run in separate address spaces. At the time of ``fork()``, both address spaces should have the same content. ``xv6`` achieves this by copying all the content of the physical memory used by the parent process into another physical memory and map them into the child's address space (see ``uvmcopy()`` at ``./kernel/vm.c``). This is not efficient at all, especially when the child process immediately calls ``exec()`` to run another program. Your task is to make ``xv6`` more memory-efficient across ``fork()`` by sharing physical memory between the parent and child process as much as possible using virtual memory techniques such as copy-on-write.


## Problem specification


### 1. Share the code segment between the parent and child process across ``fork()`` (25 points)

The first task is to make the parent process and child process share the same code segment across the ``fork()`` system call. Each page in the code segment is set to Read-only, so there is no problem when the corresponding page frame is mapped to multiple processes. Note that the parent process can fork multiple child processes and each child process also can fork another processes. You should make sure the page frames in the physical memory are released only when the last process that shares the code segment exits. 

Remember we are NOT using demand paging in this project. When a program is first loaded using the ``exec()`` system call, the whole content of the code and data segments are read from the executable file at once. 

You don't have to consider code segment sharing among the processes that invoke the ``exec()`` system call for the same executable file. Consider the following example:

```
$ cat | cat | cat
^p
1 sleep  init
2 sleep  sh
3 sleep  sh
4 sleep  cat
5 sleep  sh
6 sleep  cat
7 sleep  cat
```

As you can see, the ``sh`` process (PID 2) creates the total five processes (PID 3-7) to handle this command. Each ``cat`` process is first forked from the parent ``sh`` process and then calls ``exec()`` to run the ``cat`` program. Eventually, three processes with PID 4, 6, and 7 will load the code and data image from the same executable file, but you don't have to make them share the single code segment. To summarize, you need to focus on memory sharing across the ``fork()`` system call only.

### 2. Implement copy-on-write on data/stack/heap segment between the parent and child process (40 points)

Unlike code pages, we need to implement copy-on-write (COW) for data pages because they can be written by the parent or the child process. You can implement copy-on-write as follows:

* When a child process is created, map the page frames used by the parent process for data segment, stack segment, and heap segment into the address space of the child process, and then make them Read-only.

* When any of the parent and child process tries to modify the data in those regions, the RISC-V processor will raise a page fault due to the invalid permission on that page (cf. exception code 15: Store page fault in the above table)

* In the page fault handler, allocate a new page frame and copy the content of the original page frame into the newly allocated page frame.

* Modify the page table of the faulting process so that the corresponding PTE can point to the newly allocated page frame, with allowing the write permission on that page.

* As a result of this copy-on-write action, if the original page frame is being mapped by only a single process, give it a write permission too so that there will be no more copy-on-write. 

* Copy-on-write should be performed only for the page in which the page fault occurs. You should not copy the whole page frames that belong to data, stack, or heap segment.

Again, note that our ``xv6`` does not use demand paging. For example, when a process wants to grow the heap segment using the ``sbrk()`` system call, ``xv6`` allocates the corresponding page frames immediately at the time of the ``sbrk()`` system call. 

### 3. Make sure there is no memory leak (30 points)

In order to get the full credit in this project, you need to make sure there is no memory leak in your implementation. In the skeleton code, we have added a global variable called ``freemem`` in ``xv6``. The value of ``freemem`` indicates the number of free page frames currently available in the system. It is decremented by 1 when a page frame is allocated in ``kalloc()`` and incremented by 1 when a page frame is freed in ``kfree()``. The current value of ``freemem`` is also displayed when you press ``ctrl-p`` on ``xv6``. We have also added a system call named ``getfreemem()`` which returns the current value of ``freemem``. 

Whenever you return to the shell after executing a command, the value of ``freemem`` should remain exactly the same. Otherwise, it means either you forgot to free some page frames or you deallocated some page frames that you shouldn't. 

Before your submission, please make sure your implementation does not have memory leak by monitoring the value of ``freemem`` after running the following programs:

* forktest
* schedtest1
* usertests
* cat README | wc
* cat README | wc | wc | wc
* cat | cat | cat
* ...

### 4. Design document (5 points)

You need to prepare and submit the design document (in a single PDF file) for your implementation. Your design document should include the followings:

* Brief summary of modifications you have made
* How do you receive the page fault?
* How do you implement code segment sharing?
* How do you implement copy-on-write on data/stack/heap segment?
* When is a page frame freed and how?
* Other things you have considered in your implementation

## Skeleton code

Because we made several modifications (including a bug fix) in the ``xv6`` source code, you should download the fresh ``xv6-riscv-snu`` from Github. The skeleton code for this project (PA5) is available as a branch named ``pa5``. 

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ cd xv6-riscv-snu
$ git checkout pa5
```
After downloading, you have to set your STUDENTID again in the ``Makefile``. 

To help debugging and automatic grading, we have added a system call named ``v2p()`` and the test program called ``v2ptest`` (see ``./user/v2ptest.c``). For the given virtual address, the ``v2p()`` system call returns the beginning address of the corresponding page frame in the physical memory, if any. The ``v2ptest`` program shows the virtual-to-physical address mapping for the variables located in the data, stack, and heap segment using the ``v2p()`` system call.

The following shows the output when you run the ``v2ptest`` program on the skeleton code.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 3G -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

init: starting sh
$ ^p
1 sleep  init
2 sleep  sh
freemem = 32572 (pages)

$ v2ptest
init
&g: va=0x0000000000001A94 pa=0x0000000087F3F000
&s: va=0x0000000000003FBC pa=0x0000000087F3D000
hp: va=0x0000000000004000 pa=0x0000000087F46000
child: after fork()
&g: va=0x0000000000001A94 pa=0x0000000087F64000
&s: va=0x0000000000003FBC pa=0x0000000087F71000
hp: va=0x0000000000004000 pa=0x0000000087F57000
child: after g++
&g: va=0x0000000000001A94 pa=0x0000000087F64000
&s: va=0x0000000000003FBC pa=0x0000000087F71000
hp: va=0x0000000000004000 pa=0x0000000087F57000
child: after *hp='x'
&g: va=0x0000000000001A94 pa=0x0000000087F64000
&s: va=0x0000000000003FBC pa=0x0000000087F71000
hp: va=0x0000000000004000 pa=0x0000000087F57000
parent: after wait()
&g: va=0x0000000000001A94 pa=0x0000000087F3F000
&s: va=0x0000000000003FBC pa=0x0000000087F3D000
hp: va=0x0000000000004000 pa=0x0000000087F46000
$ ^p
1 sleep  init
2 sleep  sh
freemem = 32572 (pages)
```

Once you implement all the requirements of this project successfully, your output should be similar to the following. Note that the actual physical addresses and the value of ``freemem`` can vary depending on your implementation.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 3G -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

init: starting sh
$ ^p
1 sleep  init
2 sleep  sh
freemem = 32536 (pages)

$ v2ptest
init
&g: va=0x0000000000001A94 pa=0x0000000087F42000
&s: va=0x0000000000003FBC pa=0x0000000087F40000
hp: va=0x0000000000004000 pa=0x0000000087F49000
child: after fork()
&g: va=0x0000000000001A94 pa=0x0000000087F42000
&s: va=0x0000000000003FBC pa=0x0000000087F4B000
hp: va=0x0000000000004000 pa=0x0000000087F49000
child: after g++
&g: va=0x0000000000001A94 pa=0x0000000087F4C000
&s: va=0x0000000000003FBC pa=0x0000000087F4B000
hp: va=0x0000000000004000 pa=0x0000000087F49000
child: after *hp='x'
&g: va=0x0000000000001A94 pa=0x0000000087F4C000
&s: va=0x0000000000003FBC pa=0x0000000087F4B000
hp: va=0x0000000000004000 pa=0x0000000087F4D000
parent: after wait()
&g: va=0x0000000000001A94 pa=0x0000000087F42000
&s: va=0x0000000000003FBC pa=0x0000000087F40000
hp: va=0x0000000000004000 pa=0x0000000087F49000
$ ^p
1 sleep  init
2 sleep  sh
freemem = 32536 (pages)
```

You can see that the parent and child process share the same page frames just after the ``fork()`` system call, but as the child process modifies data, the corresponding page frame is COW'ed. You can notice that the physical address of the child's stack (0x87F4B000) is different from that of the parent's stack (0x87F40000) after ``fork()``. This is because the child process writes some data to its stack (hence, COW'ed) at the beginning of the ``printf()`` function as soon as it returns from the ``fork()`` system call.  


## Restrictions

* To make the problem easier, we assume a single-processor machine in this project. The ``CPUS`` variable that represents the number of CPUs in the target QEMU machine emulator is already set to 1 in the ``Makefile``.

* Do not add any system calls.

* You only need to modify those files in the ``./kernel`` directory. Changes to other source code will be ignored during grading.

## Hand in instructions

* Please remove all the debugging outputs before you submit. 
* To submit your code, please run ``make submit`` in the ``xv6-riscv-snu`` directory. It will create a file named ``xv6-PA5-STUDENTID.tar.gz`` file in the parent directory. Please upload the file to the server with your design document (in PDF format).
* By default, the ``sys.snu.ac.kr`` server is only accessible from the SNU campus network. If you want to access the server outside of the SNU campus, please send a mail to the TA.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delay.
* __You can use up to 5 _slip days_ during this semester__. If your submission is delayed by 1 day and if you decided to use 1 slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use in the QnA board of the submission server after each submission. Saving the slip days for later projects is highly recommended!
* Any attempt to copy others' work will result in heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
