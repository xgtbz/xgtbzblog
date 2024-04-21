+++
title = 'Physical to Virtual Mapping'
date = 2024-04-21T13:54:05+01:00
draft = false
+++

## Preliminary

In this short blog post, we will be dissecting the innards of routines such as `MmMapLockedPages` and `MmMapIoSpace` to understand how the Virtual Memory Manager (`VMM`) maps physical memory to the kernel virtual address (`VA`) space at a high level. Mapping physical memory is a powerful technique that can be utilised for several purposes, such as overwriting read only virtual memory without changing the protection, and updating paging structures themselves. The point of this blog post isn't to meticulously demonstrate each & every feature involved, but to explain general concepts with the appropriate detail for practical use and development. We will begin by describing the different uses of mapping physical memory throughout the operating system (`OS`).

## PML4 Auto Entry

As we all know, paging structure entries, `PxE`s (e.g `PDPTe`, `PTE`, `PML4e`), are consulted during the virtual to physical translation process to provide information on a given virtual range. They contain useful data required in the translation process, such as the `PFN` to the next paging structure ( or the target physical address be it the `PDE/PTE` ), the `G` bit which indicates if entries should remain in the `TLB` after a new `CR3` is loaded, and the `R/W` bit. The `VMM` must constantly update these structures to reflect the current state of a given virtual memory range, for example, when a `VA` is written to for the first time the `dirty bit` in the `PTE/PDE` must be set. However, these paging structure entries reside in physical memory, and processor instructions only use `VA`s, so how does the `VMM` update these structures using their physical addresses? The answer is: the physical addresses of `PxEs` are mapped to `VA`s via the `PML4` Auto Entry.

The Auto entry is an entry within the `PML4` that references the `PML4` again instead of providing the `PFN` to the next paging structure (which would be the `PDPT`). This lags the whole translation behind, so we end up with a virtual address mapping the page table itself. Let's pretend the auto entry is at index `0x15f` and understand what happens step by step:

![Image](https://i.ibb.co/Jps8SdZ/page-table-translation.png)

* First, bits `39-47` (which would be set the `0x15f` in this case) of the `VA` mapping this paging structure would be used as an index into the `PML4`.

* Entry `0x15f` would return the `PFN` of the `PML4`, since it is the auto entry.

* Bits `30-38` (the `PDPT` offset) of the `VA` would be used as an offset into the `PML4`.

* This time, the `PML4` will return the `PFN` of the `PDPT`.

* Bits `20-29` (the `PD` offset) of the `VA` would be used as an offset into the `PDPT`. This would return the `PFN` of the `PD`.

* Lastly, bits `12-20` (the `PT` offset) of the `VA` would be used as an offset into the `PD`. This would return the `PFN` of the `PT`. This is the Page Table mapped by the `VA`, from this you can access any `PTE` in the table.

The `VA` mapping the lowest (first) `PTE` is known as the `PteBase`, and is also the lowest kernel `VA` used. Note that on windows 7, the auto entry is hardcoded at offset `0x1ed`, but on Windows 10 & 11 it is randomised on boot. Though there is only a single `PML4` auto entry, setting more bits to the auto entry in a given `VA` will lag the translation behind by the amount of auto entry references there are in the `VA`. For example, setting bits `30-38` (`PDPT` offset) to the `PML4` auto entry as well as bits `39-47` (`PML4` offset) would result in the `PD` being resolved at the end of the translation process. System routines such as `MiGetPteAddress` and `MiFillPteHierarchy` are used to retrieve the virtual addresses that map to `PxEs`.

## Operating System Use Cases
### → Working Set Trimming

Another example of when the `VA` mapping a paging structure is consulted is during working set trimming. Working set list entries (`WSLe`) are kept in no particular order in the Working Set List (`WSL`), so given a regular virtual address that has been removed from a `WS` how does the `VMM` locate the corresponding `WSLe` to dicard? Well, it turns out the `VA` of the `PTE` for that virtual address is computed ( via `MiDeleteVirtualAddresses` → `MiDeletePagablePteRange` → `MiFillPteHierarchy` ), then used to resolve the `PFN`, which can be used as an index into the `PFN` Database, returning the corresponding `MMPFN` structure. Now, the appropriate `WSL` index can be extracted by the `VMM` from the `MMPFN` structure. Let's decipher the computation performed by `MiGetPteAddress`:

```cpp
__int64 __fastcall MiGetPteAddress(unsigned __int64 VirtualAddress)
{
  return ((VirtualAddress >> 9) & 0x7FFFFFFFF8i64) - 0x98000000000i64;
}
```

This calculation consists of:
* Shifting the `VA` 9 bits to the right, now the `PDPT` index is applied to the `PML4` slot

* Setting the least significant 3 bits to 0

* Subtracting some hardcoded value 0x98000000000

This hardcoded value is suppose to represent the `PteBase`, which is dynamically retrieved at `run-time` based off the auto-entry. Earlier, we looked at how the auto-entry lags the translation process behind, but now we've shifted the virtual address by 9 bits to the right, so the `PDPT` offset is applied to the `PML4` slot. So now, during translation, the `PML4` is not referenced more than once ( only from the `CR3` now ). So in essence, this translation gives us another `VA`, which maps the `PT` used to map the `VirtualAddress` field we passed to the routine. The `PT` index is used as an offset in the `PT` to resolve the `PTE`, so we end up with the `VA` of the `PTE`.

### → Demand Zero Fault

On Windows, newly allocated virtual addresses aren't mapped to physical pages until they are initially referenced. This means when a virtual address is referenced for the first time, a page fault is triggered and the `VMM` allocates a new physical page to map to the virtual address. These pages are known as zero initialised pages & are found on the `Zero List`, hence "**demand zero fault**". Zero initialised pages have no `PTE` pointing to them, they are free for the `VMM` to use to map to new virtual addresses. However, when the `Zero List` is empty, the `VMM` consults the `Free List` to steal a free page that can be used for the mapping. The free list contains pages that aren't being pointed to by any `PTE`, however, they were once pointed to by a `PTE` and still contain the stale data from the old `VA` it was mapped to. So, before pages from the free list can be used, they must be zeroed, their contents have to be erased. However, this brings the same problem as before, pages on the free list aren't mapped to any VA and processor instructions do not interface directly with physical instructions; so they must be mapped to `VAs` in order to be zeroed.

There is a special system thread dedicated to zeroing pages from the free list, known as the `Zero Page Thread`. This executes `MiZeroPhyscialPage` which calls `MiMapPageInHyperspaceWorker` to perform the mapping. Interestingly, `MiMapPageInHyperspaceWorker` uses a single `VA` to map free pages to be zeroed, meaning every call to `MiMapPageInHyperspaceWorker` must be followed with a subsequent call to `MiUnmapPageInHyperspaceWorker`. To ensure this condition is satisfied, the routine raises the `IRQL` to `DPC` to prevent preemption, the second argument to `MiUnmapPageInHyperspaceWorker` being the `IRQL` before it was raised.

## Memory Descriptor Lists

As Microsoft Documentation states, a Memory Descriptor List (`MDL`) is used to describe the physical layout of a virtual memory range. This is because contiguous virtual memory may be scattered in physical memory as a result of fragmentation. A common techqniue used by attackers to overwrite read-only memory (`ROM`) is to allocate an `MDL` to describe some virtual memory buffer, and then map the physical pages, stored in the `MDL`, corresponding to the virtual memory buffer to another virtual range with a different memory protection. This gives you two separate virtual address ranges that resolve to the same physical address, but have different memory protections, analogous to two separate views of a section oject. Now we've looked at how physical memory is mapped to virtual memory by the `OS`, we should shift our focus to examining the different ways that mapping physical memory can be leveraged by an attacker in `ring 0`, and how it works for different routines such as `MmMapMapLockedPagesSpecifyCache`.

```cpp
bool OverwriteRom(void* VA, void* NewValue, uint64 Size) {
    // Allocate the MDL
    MDL* Mdl = IoAllocateMdl(VA, Size, FALSE, FALSE, NULL);

    if (!Mdl) return false;

    // Ensure pages remain resident
    MmProbeAndLockPages(Mdl, KernelMode, IoReadAccess);

    // Establish the mapping
    void* MappedRange = MmMapLockedPagesSpecifyCache(Mdl, KernelMode, 
            MmNonCached, NULL, FALSE, NormalPagePriority);

    if (MappedRange == nullptr)
    {
        MmUnlockPages(Mdl);
        return false;
    }

    // Change the protection of the mapped range
    MmProtectMdlSystemAddress(Mdl, PAGE_READWRITE);

    // Write to the mapped RW range
    RtlCopyMemory(MappedRange, NewValue, Size) 
    
    // Cleanup
    MmUnmapLockedPages(MappedRange, Mdl);
    MmUnlockPages(Mdl);
    IoFreeMdl(Mdl);

    return true;
}
```

Instead of using an `MDL`, the same effect can be achieved with `MmMapIoSpace`, which would be more suitable for a smaller virtual memory range.

```cpp
bool OverwriteRom(void* VA, void* NewValue, uint64 Size) {

    PHYSICAL_ADDRESS PA = MmGetPhysicalAddress(VA);

    // Default RW/RWX
    void* MappedRange = MmMapIoSpace(PA, Size, MmNonCached);
    RtlCopyMemory(MappedRange, NewValue, Size);

    MmUnmapIoSpace(MappedRange, Size);
}
```

When we statically analyse both of these routines to understand how the mapping works, we end up with a call to `MiReservePtes` in both cases. A global qword passed as the first argument, and a 32-bit integer as the next. To understand the purpose of this function and their arguments, we should be aware of how the `VMM` manages the `system PTE range`, and what it is used for.

```cpp
MiReservePtes((__int64)&qword_140C69600, (unsigned int)v13);
```

## MiReservePtes

The `system pte` region is a dynamically allocated portion of the kernel virtual address space housing `MDL` mappings, drivers, kernel stacks and more. To manage this region, the `VMM` maintains two structures of type `MI_SYSTEM_PTE_TYPE` named `MiSystemPteInfo` and `MiKernelStackPteInfo`.

```cpp
struct _MI_SYSTEM_PTE_TYPE
{
    struct _RTL_BITMAP Bitmap;                                              //0x0
    struct _MMPTE* BasePte;                                                 //0x8
    ULONG Flags;                                                            //0xc
    enum _MI_SYSTEM_VA_TYPE VaType;                                         //0x10
    ULONG* FailureCount;                                                    //0x14
    ULONG PteFailures;                                                      //0x18
    union
    {
        ULONG SpinLock;                                                     //0x1c
        struct _EX_PUSH_LOCK* GlobalPushLock;                               //0x1c
    };
    volatile ULONG TotalSystemPtes;                                         //0x20
    ULONG Hint;                                                             //0x24
    ULONG LowestBitEverAllocated;                                           //0x28
    struct _MI_CACHED_PTES* CachedPtes;                                     //0x2c
    volatile ULONG TotalFreeSystemPtes;                                     //0x30
}; 
```

The first field is a `bitmap` representing a portion of the system `pte` range. Each bit corresponds to a `large page`, 2MB, however keep in mind regular pages can be allocated within this region. These buffers are stored at the start of system `pte` region themselves, adjacently, with 4MB reserved for each of them, so 8MB of the system `pte` range is reserved for this purpose. Allocations to this region are perfomed via a call to `MiReservePtes`, which takes one of these `bitmaps` as its first argument - the second being the amount of pages to be reserved. From this, we can deduce that `MiReservePte`s can be used to reserve a virtual address to map physical pages into. The routine that actually implements the physical memory mapping using the `PTEs` allocated by `MiReservePtes` is `MiFillSystemPtes`, while `MiExpandPtes` is used to assign free large pages to the `bitmap`.

```cpp
    MMPTE* Pte = MiReservePtes((_MI_SYSTEM_PTE_TYPE *)&MiSystemPteInfo, (unsigned int)PageCount);
    if ( !Pte )
      return NULL;
    if ( (int)MiFillSystemPtes((_DWORD)Pte .... ); 
```

## Conclusion 
You should now have a solid understanding of how physical memory is mapped to virtual on the `OS`, and the different use cases of this mapping. We've looked at some of the root function ( `MiReservePtes` ) called by all routines that map physical memory to the system pte virtual address range ( e.g `MmAllocateMappingAddress` ) at a high level, and how the system `pte` range is managed by the `VMM`.