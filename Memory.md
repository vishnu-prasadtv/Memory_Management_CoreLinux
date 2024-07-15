
# 1.2 Linux memory architecture

To execute a process, the Linux kernel allocates a portion of the memory area to the requesting process. 
The process uses the memory area as workspace and performs the required work. 

The kernel has to allocate space in a more dynamic manner. The number of running processes sometimes comes to tens of thousands and amount of memory is usually limited. Therefore, Linux kernel must handle the memory efficiently.

## 1.2.1 Physical and virtual memory

From a performance point of view, it is interesting to understand how the Linux kernel maps physical memory into virtual memory on both 32-bit and 64-bit systems.

As you can see in Figure 1-10, there are obvious differences in the way the Linux kernel has to address memory in 32-bit and 64-bit systems. Exploring the physical-to-virtual mapping in detail is beyond the scope of this article, so we highlight some specifics in the Linux memory architecture.

On 32-bit architectures such as the IA-32, the Linux kernel can directly address only the first gigabyte of physical memory (896 MB when considering the reserved range). Memory above the so-called ZONE_NORMAL must be mapped into the lower 1 GB. This mapping is completely transparent to applications, but allocating a memory page in ZONE_HIGHMEM causes a small performance degradation.

On the other hand, with 64-bit architectures such as x86-64 (also x64), ZONE_NORMAL extends all the way to 64 GB or to 128 GB in the case of IA-64 systems. As you can see, the overhead of mapping memory pages from ZONE_HIGHMEM into ZONE_NORMAL can be eliminated by using a 64-bit architecture.

## The Linux Memory Architecture

![image](https://github.com/vishnu-prasadtv/core-linux/assets/175235558/dc807b81-883b-4a82-8a51-96cda3d21e52)



## Virtual memory addressing layout

Figure 1-11 shows the Linux virtual addressing layout for 32-bit and 64-bit architecture.

On 32-bit architectures, the maximum address space that single process can access is 4GB. This is a restriction derived from 32-bit virtual addressing. In a standard implementation, the virtual address space is divided into a 3 GB user space and a 1 GB kernel space. There is some variants like 4 G/4 G addressing layout implementing.

On the other hand, on 64-bit architecture such as x86_64 and ia64, no such restriction exits. Each single process can benefit from the vast and huge address space.

![image](https://github.com/vishnu-prasadtv/core-linux/assets/175235558/22f37824-1cdd-45b3-95d6-c88c66b95137)


## 1.2.2 Virtual memory manager

The physical memory architecture of an operating system is usually hidden to the application and the user because operating systems map any memory into virtual memory. If we want to understand the tuning possibilities within the Linux operating system, we have to understand how Linux handles virtual memory. As explained in 1.2.1, “Physical and virtual memory”, applications do not allocate physical memory, but request a memory map of a certain size at the Linux kernel and in exchange receive a map in virtual memory. As you can see in Figure 1-12, virtual memory does not necessarily have to be mapped into physical memory. If your application allocates a large amount of memory, some of it might be mapped to the swap file on the disk subsystem.

Figure 1-12 shows that applications usually do not write directly to the disk subsystem, but into cache or buffers. The pdflush kernel threads then flushes out data in cache/buffers to the disk when it has time to do so or if a file size exceeds the buffer cache. Refer to “Flushing a dirty buffer” on page 22.

![image](https://github.com/vishnu-prasadtv/core-linux/assets/175235558/212cc042-caa4-4efe-b27d-7bb03b993445)


## The Linux virtual memory manager

Closely connected to the way the Linux kernel handles writes to the physical disk subsystem is the way the Linux kernel manages disk cache. While other operating systems allocate only a certain portion of memory as disk cache, Linux handles the memory resource far more efficiently. The default configuration of the virtual memory manager allocates all available free memory space as disk cache. Hence it is not unusual to see productive Linux systems that boast gigabytes of memory but only have 20 MB of that memory free.

In the same context, Linux also handles swap space very efficiently. Swap space being used does not indicate a memory bottleneck but proves how efficiently Linux handles system resources. See “Page frame reclaiming” for more detail.

## Page frame allocation

A page is a group of contiguous linear addresses in physical memory (page frame) or virtual memory. The Linux kernel handles memory with this page unit. A page is usually 4 K bytes in size. When a process requests a certain amount of pages, if there are available pages the Linux kernel can allocate them to the process immediately. Otherwise pages have to be taken from some other process or page cache. The kernel knows how many memory pages are available and where they are located.

## Buddy system

The Linux kernel maintains its free pages by using a mechanism called a buddy system. The buddy system maintains free pages and tries to allocate pages for page allocation requests. It tries to keep the memory area contiguous. If small pages are scattered without consideration, it might cause memory fragmentation and it’s more difficult to allocate a large portion of pages into a contiguous area. It could lead to inefficient memory use and performance decline. Figure 1-13 illustrates how the buddy system allocates pages.

![image](https://github.com/vishnu-prasadtv/core-linux/assets/175235558/a025eceb-56e9-4f4c-aec6-e13c41d799e2)


When the attempt of pages allocation fails, the page reclaiming is activated. Refer to “Page frame reclaiming”.

You can find information on the buddy system through /proc/buddyinfo. For details, refer to “Memory used in a zone”.

## Page frame reclaiming

If pages are not available when a process requests to map a certain amount of pages, the Linux kernel tries to get pages for the new request by releasing certain pages (which were used before but are not used anymore and are still marked as active pages based on certain principles) and allocating the memory to a new process. This process is called page reclaiming. kswapd kernel thread and try_to_free_page() kernel function are responsible for page reclaiming.

While kswapd is usually sleeping in task interruptible state, it is called by the buddy system when free pages in a zone fall short of a threshold. It tries to find the candidate pages to be taken out of active pages based on the Least Recently Used (LRU) principle. The pages least recently used should be released first. The active list and the inactive list are used to maintain the candidate pages. kswapd scans part of the active list and check how recently the pages were used and the pages not used recently are put into the inactive list. You can take a look at how much memory is considered as active and inactive using the vmstat -a command. For detail refer to 2.3.2, “vmstat” on page 42.

kswapd also follows another principle. The pages are used mainly for two purposes: page cache and process address space. The page cache is pages mapped to a file on disk. The pages that belong to a process address space (called anonymous memory because it is not mapped to any files, and it has no name) are used for heap and stack. Refer to 1.1.8, “Process memory segments” on page 8. When kswapd reclaims pages, it would rather shrink the page cache than page out (or swap out) the pages owned by processes.

## Page out and swap out: The phrases “page out” and “swap out” are sometimes confusing. The phrase “page out” means take some pages (a part of entire address space) into swap space while “swap out” means taking entire address space into swap space. They are sometimes used interchangeably.

A large proportion of page cache that is reclaimed and process address space that is reclaimed might depend on the usage scenario and will affect performance. You can take some control of this behavior by using /proc/sys/vm/swappiness. Refer to 4.5.1, “Setting kernel swap and pdflush behavior” on page 109 for tuning details.

# swap

As we stated before, when page reclaiming occurs, the candidate pages in the inactive list which belong to the process address space may be paged out. Having swap itself is not problematic situation. While swap is nothing more than a guarantee in case of over allocation of main memory in other operating systems, Linux uses swap space far more efficiently. As you can see in Figure 1-12 on page 13, virtual memory is composed of both physical memory and the disk subsystem or the swap partition. If the virtual memory manager in Linux realizes that a memory page has been allocated but not used for a significant amount of time, it moves this memory page to swap space.

Often you will see daemons such as getty that will be launched when the system starts up but will hardly ever be used. It appears that it would be more efficient to free the expensive main memory of such a page and move the memory page to swap. This is exactly how Linux handles swap, so there is no need to be alarmed if you find the swap partition filled to 50%. The fact that swap space is being used does not indicate a memory bottleneck; instead it proves how efficiently Linux handles system resources.

# Process Memory in Linux

A process uses its own memory area to perform work. The work varies depending on the situation and process usage. A process can have different workload characteristics and different data size requirements. The process has to handle a of variety of data sizes. To satisfy this requirement, the Linux kernel uses a dynamic memory allocation mechanism for each process. The process memory allocation structure is shown in Figure 1-7.

![image](https://github.com/vishnu-prasadtv/core-linux/assets/175235558/fec50631-c9f1-40fb-a5da-9535f6ca866c)

The process memory area consist of these segments

• Text segment: The area where executable code is stored.

• Data segment: The data segment consists of these three areas.

    – Data: The area where initialized data such as static variables are stored.

    – BSS: The area where zero-initialized data is stored. The data is initialized to zero.

    – Heap: The area where malloc() allocates dynamic memory based on the demand. The heap grows towards higher addresses.

• Stack segment: The area where local variables, function parameters, and the return address of a function is stored. The stack grows toward lower addresses.

The memory allocation of a user process address space can be displayed with the pmap command. You can display the total size of the segment with the ps command. 
