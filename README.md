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
