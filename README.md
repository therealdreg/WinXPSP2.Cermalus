# WinXPSP2.Cermalus
Understanding WinXPSP2.Cermalus by Dreg

Plese, consider make a donation: https://github.com/sponsors/therealdreg

The build of WinXPSP2.Cermalus is a dropper.

The dropper:

   1. Enabling read and write in code area, VirtualProtect PAGE_READWRITE.

   2. Generating CRC for APIs and Modules names:

      ntoskrnl.exe: wcscmp, ZwUnmapViewOfSection, ZwQueryInformationFile, ZwOpenFile, ZwOpenDirectoryObject, ZwMapViewOfSection, ZwCreateSection, ZwClose, PsSetCreateThreadNotifyRoutine, PsSetCreateProcessNotifyRoutine, PsRemoveCreateThreadNotifyRoutine, ProbeForWrite, ProbeForRead, ObReferenceObjectByHandle, ObDereferenceObject, MmUnlockPages, MmProbeAndLockPages, MmGetSystemRoutineAddress, KeSetTimer, KeServiceDescriptorTable, KeInitializeTimer, KeInitializeSpinLock, KeInitializeDpc, KeBugCheck, IoFreeMdl, IoDriverObjectType, IoDeleteDevice, IoCreateFile, IoCreateDevice, IoCompleteRequest, IoAllocateMdl, ExFreePool, ExAllocatePool, DbgPrintReturnControlC, DbgPrintEx & DbgPrint.

      hal.dll: KeAcquireSpinLock, KeGetCurrentIrql & KeReleaseSpinLock

      ntdll.dll: ZwEnumerateBootEntries, ZwDebugActiveProcess, ZwEnumerateBootEntries & ZwOpenFile

      kernl32.dll: CloseHandle, CreateFileA, CreateFileMappingA, DeleteFileA, GetFullPathNameA, LoadLibraryA, MapViewOfFile, UnmapViewOfFile, VirtualAlloc, VirtualFree, & WriteFile

      advapi.dll: CloseServiceHandle, ControlService, CreateServiceA, DeleteService, OpenSCManagerA, OpenServiceA & StartServiceA

   3. Restore the old code area permissions.

   4. Jump to real ring3 entry point of WinXPSP2.Cermalus

   5. When ring3 EP finish, It shows a MessageBox with title "[WinXPSP2.Cermalus by Pluf/7A69ML]" and text "[first step]" and the process finish.

There is only two difference between dropper and the infected file: the CRC of API's, Modules and the use of delta offset, the dropper is not relocatable code... but when the dropper jumps to real ring3 entry point ... obviously yes.

Real ring3 entry point a.k.a entry point of a infected file a.k.a WinXPSP2.Cermalus:

   1. It inserts a SEH frame and Gets the base of kernel32.dll using PEB, Ldr, InInitializationOrderModuleList ...

   2. It loads the modules and APIs, if there is an error it jumps to original entry point of the infected file.

   3. It checks if the driver is present, if this is true it jumps to original entry point. It checks the presence using ZwEnumerateBootEntries API: ZwEnumerateBootEntries( 0xBEBE, 0xCAFE ) ... in english the mean of "bebe cafe" is "drink coffee" :-). If ZwEnumerateBootEntries returns "0" the driver is present.

   4. It dumps the driver called "cermalus.sys", after It loads the driver using the SCManager (all code in "nonpagedpool"). About the driver... description: evilinside, the driver also contains the WinXPSP2.Cermalus ring3 code, It generates the checksum...

   5. The load of the driver is using The OpenSCManagerA API, It stop the service with SERVICE_CONTROL_STOP arg in ControlService and It deletes the entry of the driver in SCManager data base with DeleteService API.

   6. It creates (again) the service, calling CreateServiceA and StartServiceA

   7. Ring3 code finish, It deletes SEH frame and It jumps to original entry point of the infected file aka OEP.

cermalus.sys:

   1. It uses delta offset (relocatable code).

   2. It uses the"stack scandown technique" to find the base address of ntoskrnl. It gets the ret address of IopLoadDriver of IoManager of ntoskrnl. With the ret address of IopLoadDriver it finds the base address of ntoskrnl (page per page).

   3. It gets the ring0 APIs (remember the CRC). It gets the base address of the hal.dll using the "stack scandown technique" and the MmGetSystemRoutineAddress of ntoskrnl (instead of PsLoadedModuleList). It calls to MmGetSystemRoutineAddress with the API name of HAL.dll: HalInitSystem (only need a unicode_string). With the address of HalInitSystem It uses the "stack scandown technique" to get the base address of hal.

   4. It creates a nonpagedpool area with the ExAllocatePool API. This area is the global data of the driver. It stores the global dalta to delta ptr, [ebp] == global data. The delta ptr is in code area (no write area). For writing in the code area it clears WP bit in the CR0 register.

   5. It strores the last data resolved in the stack in the global data area, also It stores DriverObject->DriverSection value in the global data. It can access to DriverSection field because the driver starts with SCManager.

   6. It gets the index of services using the KiServiceTable. It uses ntdll.dll to get the index of the services. It gets the index of Zw APIs in ntdll.dll and find it in the KiServiceTable. It get the index reading the mov instruction, for example: mov eax, 0x27 = b8 27 00 00 00. It checks the B8 opcode in the wrapper function and get the index, if the index is higher than the number of services it exist an error. Finally, It stores the address of service.

   7. For the hooking of services and other task It raises a IRQL 2 (DISPATCH LEVEL), before the IRQL was in passive, system context. For this porpouse it calls to KeInitializeSpinLock with PKSPIN_LOCK and after KeAcquireSpinLock with PKIRQL and PKSPIN_LOCK. If the IRQL is not DISPATCH it jumps to "wdog" if not it hooks the services.

   8. It clears WP bit in the CR0 register for hooking the nt services, It inserts a jmp near direct aka E9Inm32b in the hooked services. It hooks: NtDebugActiveProcess, NtOpenFile and NtEnumerateBootEntries.

   9. It hooks the EAT Modules.

  10. It hides the driver from module list, It unlink the module_list entry of PsLoadedModuleList. It hides th drver from Object directory

  11. It restores the IRP to passive and restores WP bit.

  12. It start wdog aka watchdog, watchdog is a system thread wich check the integrity of driver code every time. If the code is modified it clear the driver and reboot the machine with bugcheck.

  13. It patch the wdog labels: DbgPrint, KeBugCheck, KeInitializeDpc, KeInitializeTimer and KeSetTimer.

  14. It also registers the driver_unload routine. But... in the Cermalus case this is not necessary.

Now is the time of service hook routines:

   1. NtOpenFile: This routine infects the .exe, except the .exes inside windows directory. It checks if the .exe is already infected.

   2. NtEnumerateBootEntries: It returns STATUS_SUCCESS when the args are: "0xBEBE, 0xCAFE".

   3. NtDebugActiveProcess: It blocks the attach to ring3 process.

   4. DbgPrint/DbgPrintEx/DbgPrintReturnControlC: It blocks the debug using DbgPrint*

   5. PsSetCreateProcessNofityRoutine/PsSet//RemoveCreateThreadNotifyRoutine/: It returns STATUS_SUCCESS, but the hook is empty. It is useful to evade software monitors like ProcMon..

File infection:

   1. It checks if the file is already infected: if the real file size is less or diferent than PointerToRawData + SizeOfRawData the file is infected. Yeah false positives, but the author it has a purpose (a random factor).



