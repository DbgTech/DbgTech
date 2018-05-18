---
title: ReactOS启动过程之FreeLdr的I/O子系统
date: 2017-09-06 13:52:23
tags:
- Windbg
- ReactOS
- 启动过程
categories:
- ReactOS
---

在执行到FreeLdr.sys时，ReactOS的系统并未加载，对应的FAT32文件系统机制也没有初始化。这个时候要访问文件就需要FreeLdr.sys文件自己构建“文件系统”来访问磁盘上的文件。

这样就涉及到磁盘设备管理和文件系统两部分内容，即给定文件首先判断所属设备，然后根据路径找到文件。

#### 1. FreeLdr.sys的I/O子系统初始化 ####

在FreeLdr.sys的主函数BootMain()中调用FsInit()初始化I/O子系统。

```
VOID FsInit(VOID)
{
    ULONG i;

    RtlZeroMemory(FileData, sizeof(FileData));
    for (i = 0; i < MAX_FDS; i++)
        FileData[i].DeviceId = (ULONG)-1;

    InitializeListHead(&DeviceListHead);

    // FIXME: Retrieve the current boot device with MachDiskGetBootPath
    // and store it somewhere in order to not call again and again this
    // function.
}
```

初始化函数非常简单，将FileData全局数组进行初始化，将所有DeviceId成员设置为-1，表示没有指向内容设备。然后初始化设备列表头全局变量DeviceListHead。这个地方有一个FixMe，要在初始化时获取当前引导设备，存储起来防止反复调用，也是作为优化。

引导过程中，紧接着调用到RunLoader()函数时，会调用到MachInitializeBootDevices()，函数MachInitializeBootDevices其实是一个宏定义，它对应于`MachVtbl.InitializeBootDevices()`函数指针，而函数指针指向的是PcInitializeBootDevices()函数，该函数作用为调用BIOS读取磁盘功能，确定有多少个可见并且确实存在的磁盘，获取其信息。
<!-- more -->
```
BOOLEAN PcInitializeBootDevices(VOID)
{
    UCHAR DiskCount, DriveNumber;
    ULONG i;
    BOOLEAN Changed;
    BOOLEAN BootDriveReported = FALSE;
    CHAR BootPath[MAX_PATH];

    // 计算可见磁盘驱动数量
    DiskReportError(FALSE);
    DiskCount = 0;
    DriveNumber = 0x80;

    /*
     * 有一部分坏掉的BIOS，甚至有一些BIOS成功报告一些不存在的硬盘。因此首先设置读缓存为已知内容
     * 然后读取，如果BIOS报告成功，但是缓存值没变，那说明这块磁盘是有问题的。
     */
    memset(DiskReadBuffer, 0xcd, DiskReadBufferSize);
    while (MachDiskReadLogicalSectors(DriveNumber, 0ULL, 1, DiskReadBuffer))
    {
        Changed = FALSE;
        for (i = 0; !Changed && i < DiskReadBufferSize; i++)
        {
            Changed = ((PUCHAR)DiskReadBuffer)[i] != 0xcd;
        }
        if (!Changed)
        {
            TRACE("BIOS reports success for disk %d (0x%02X) but data didn't change\n",
                  (int)DiskCount, DriveNumber);
            break;
        }

        // 缓存BIOS银盘信息用于之后过程中使用
        GetHarddiskInformation(DriveNumber);

        // 校验设定的引导驱动器是否在找到的可见驱动器中
        if (FrldrBootDrive == DriveNumber)
            BootDriveReported = TRUE;

        DiskCount++;
        DriveNumber++;
        memset(DiskReadBuffer, 0xcd, DiskReadBufferSize);
    }
    DiskReportError(TRUE);

    PcBiosDiskCount = DiskCount;
    TRACE("BIOS reports %d harddisk%s\n",
          (int)DiskCount, (DiskCount == 1) ? "" : "s");

    // 获取正在启动的磁盘驱动的引导路径
    MachDiskGetBootPath(BootPath, sizeof(BootPath));

    // 如果是软盘或光盘，做特殊设置
    if ((FrldrBootDrive >= 0x80 && !BootDriveReported) ||
        DiskIsDriveRemovable(FrldrBootDrive))
    {
        /* TODO: Check if it's really a CDROM drive */

        PMASTER_BOOT_RECORD Mbr;
        PULONG Buffer;
        ULONG Checksum = 0;
        ULONG Signature;

        /* Read the MBR */
        if (!MachDiskReadLogicalSectors(FrldrBootDrive, 16ULL, 1, DiskReadBuffer))
        {
            ERR("Reading MBR failed\n");
            return FALSE;
        }

        Buffer = (ULONG*)DiskReadBuffer;
        Mbr = (PMASTER_BOOT_RECORD)DiskReadBuffer;

        Signature = Mbr->Signature;
        TRACE("Signature: %x\n", Signature);

        /* Calculate the MBR checksum */
        for (i = 0; i < 2048 / sizeof(ULONG); i++) Checksum += Buffer[i];
        Checksum = ~Checksum + 1;
        TRACE("Checksum: %x\n", Checksum);

        /* Fill out the ARC disk block */
        reactos_arc_disk_info[reactos_disk_count].DiskSignature.Signature = Signature;
        reactos_arc_disk_info[reactos_disk_count].DiskSignature.CheckSum = Checksum;
        strcpy(reactos_arc_disk_info[reactos_disk_count].ArcName, BootPath);
        reactos_arc_disk_info[reactos_disk_count].DiskSignature.ArcName =
            reactos_arc_disk_info[reactos_disk_count].ArcName;
        reactos_disk_count++;

        FsRegisterDevice(BootPath, &DiskVtbl);
        DiskCount++; // This is not accounted for in the number of pre-enumerated BIOS drives!
        TRACE("Additional boot drive detected: 0x%02X\n", (int)FrldrBootDrive);
    }

    return (DiskCount != 0);
}
```

首先调用MachDiskReadLogicalSectors()函数，读取从0x80开始的磁盘号，如果读出内容，则调用GetHarddiskInformation()获取硬盘信息，且将磁盘计数进行累加。获取引导的路径，如果是软盘或光盘，则进行特殊设置。

MachDiskReadLogicalSectors()也是一个宏，对应于PcDiskReadLogicalSectors()函数。首先调用DiskInt13ExtensionsSupported()判断是否支持INT13H扩展功能，如果支持直接调用INT13H扩展功能读取磁盘，否则调用默认的BIOS读取磁盘功能。对于INT 13H扩展支持要调用INT 13H 41H扩展功能，此处又涉及到对BIOS功能调用，依旧需要调用Int386()函数实现对BIOS功能的调用。DiskInt13ExtensionsSupported()函数中准备了INT 13H 41H扩展功能的参数，然后调用Int386()函数调用BIOS功能，前面解析过Int386()函数调用过程，这里不再详细解析。此处支持INT13H，即调用PcDiskReadLogicalSectorsLBA()函数。

```
BOOLEAN PcDiskReadLogicalSectors(UCHAR DriveNumber, ULONGLONG SectorNumber, ULONG SectorCount, PVOID Buffer)
{
    BOOLEAN ExtensionsSupported;
    TRACE("PcDiskReadLogicalSectors() DriveNumber: 0x%x SectorNumber: %I64d SectorCount: %d Buffer: 0x%x\n",
          DriveNumber, SectorNumber, SectorCount, Buffer);

    /*
     * 校验看该磁盘是否是一个固定磁盘，如果是则确定是否支持INT13H扩展功能
     * 支持扩展，则调用扩展功能，否则退回到BIOS默认的读取硬盘的调用。
     */
    ExtensionsSupported = DiskInt13ExtensionsSupported(DriveNumber);

    if ((DriveNumber >= 0x80) && ExtensionsSupported)
    {
        TRACE("Using Int 13 Extensions for read. DiskInt13ExtensionsSupported(%d) = %s\n", DriveNumber, ExtensionsSupported ? "TRUE" : "FALSE");

        // LBA支持，则使用LBA方式读取磁盘，比较简单
        return PcDiskReadLogicalSectorsLBA(DriveNumber, SectorNumber, SectorCount, Buffer);
    }
    else
    {
        // LBA方式不支持，默认使用CHS形式读取磁盘
        return PcDiskReadLogicalSectorsCHS(DriveNumber, SectorNumber, SectorCount, Buffer);
    }

    return TRUE;
}

static BOOLEAN DiskInt13ExtensionsSupported(UCHAR DriveNumber)
{
    static UCHAR LastDriveNumber = 0xff;
    static BOOLEAN LastSupported;
    REGS RegsIn, RegsOut;

    TRACE("DiskInt13ExtensionsSupported()\n");
    if (DriveNumber == LastDriveNumber)
    {
        TRACE("Using cached value %s for drive 0x%x\n",
              LastSupported ? "TRUE" : "FALSE", DriveNumber);
        return LastSupported;
    }

    /*
     * 在从CD引导时，一些BIOS(e.g. Phoenix BIOS v6.00PG and Insyde BIOS shipping
     * with Intel Macs)报告扩展磁盘访问功能不支持，
     * 如果从CD引导，则简单返回TRUE。假设所有的El Torito BIOS支持INT 13扩展功能
     * 检验是否从光盘启动，只需要检查驱动号是否 >= 0x8A. 在Insyde BIOS是90H，其他的是0x9F
     */
    if (DriveNumber >= 0x8A)
    {
        LastSupported = TRUE;
        return TRUE;
    }

    LastDriveNumber = DriveNumber;

    /*
     * IBM/MS INT 13 Extensions - INSTALLATION CHECK
     * AH = 41h
     * BX = 55AAh
     * DL = drive (80h-FFh)  80H-FFh 为固定磁盘驱动
     * Return:
     * CF set on error (extensions not supported)
     * AH = 01h (invalid function)
     * CF clear if successful
     * BX = AA55h if installed
     * AH = major version of extensions
     * 01h = 1.x
     * 20h = 2.0 / EDD-1.0
     * 21h = 2.1 / EDD-1.1
     * 30h = EDD-3.0
     * AL = internal use
     * CX = API subset support bitmap
     * DH = extension version (v2.0+ ??? -- not present in 1.x)
     *
     * Bitfields for IBM/MS INT 13 Extensions API support bitmap
     * Bit 0, extended disk access functions (AH=42h-44h,47h,48h) supported
     * Bit 1, removable drive controller functions (AH=45h,46h,48h,49h,INT 15/AH=52h) supported
     * Bit 2, enhanced disk drive (EDD) functions (AH=48h,AH=4Eh) supported
     *        extended drive parameter table is valid
     * Bits 3-15 reserved
     */
    RegsIn.b.ah = 0x41;
    RegsIn.w.bx = 0x55AA;
    RegsIn.b.dl = DriveNumber;

    // 重置磁盘控制器
    Int386(0x13, &RegsIn, &RegsOut);

    if (!INT386_SUCCESS(RegsOut))
    {
        LastSupported = FALSE;  // 如果出错，CF被置位
        return FALSE;
    }

    if (RegsOut.w.bx != 0xAA55)
    {
        LastSupported = FALSE;  // 如果支持 BX = AA55h
        return FALSE;
    }

    if (!(RegsOut.w.cx & 0x0001))
    {
        /*
         * CX = API subset support bitmap.
         * Bit 0, extended disk access functions (AH=42h-44h,47h,48h) supported.
         */
        DbgPrint("Suspicious API subset support bitmap 0x%x on device 0x%lx\n",
                 RegsOut.w.cx, DriveNumber);
        LastSupported = FALSE;
        return FALSE;
    }

    LastSupported = TRUE;
    return TRUE;
}
```

PcDiskReadLogicalSectorsLBA()函数使用INT 13H 42H扩展功能读取磁盘，类似于DiskInt13ExtensionsSupported()函数的逻辑，不再详细叙述。

```
static BOOLEAN PcDiskReadLogicalSectorsLBA(UCHAR DriveNumber, ULONGLONG SectorNumber, ULONG SectorCount, PVOID Buffer)
{
    REGS RegsIn, RegsOut;
    ULONG RetryCount;
    PI386_DISK_ADDRESS_PACKET Packet = (PI386_DISK_ADDRESS_PACKET)(BIOSCALLBUFFER);

    TRACE("PcDiskReadLogicalSectorsLBA() DriveNumber: 0x%x SectorNumber: %I64d SectorCount: %d Buffer: 0x%x\n", DriveNumber, SectorNumber, SectorCount, Buffer);
    ASSERT(((ULONG_PTR)Buffer) <= 0xFFFFF);

    // 构建 磁盘地址 数据包，用于BIOS INT 13H读取磁盘
    RtlZeroMemory(Packet, sizeof(*Packet));
    Packet->PacketSize = sizeof(*Packet);
    Packet->Reserved = 0;
    Packet->LBABlockCount = (USHORT)SectorCount;
    ASSERT(Packet->LBABlockCount == SectorCount);
    Packet->TransferBufferOffset = ((ULONG_PTR)Buffer) & 0x0F;
    Packet->TransferBufferSegment = (USHORT)(((ULONG_PTR)Buffer) >> 4);
    Packet->LBAStartBlock = SectorNumber;

    /*
     * BIOS int 0x13, function 42h - IBM/MS INT 13 Extensions - EXTENDED READ
     * Return:
     * CF clear if successful
     * AH = 00h
     * CF set on error
     * AH = error code
     * Disk address packet's block count field set to the
     * number of blocks successfully transferred.
     */
    RegsIn.b.ah = 0x42;                 // 子功能 42h
    RegsIn.b.dl = DriveNumber;          // 驱动器号DL(0-软盘, 0x80-磁盘)
    RegsIn.x.ds = BIOSCALLBUFSEGMENT;   // DS:SI->磁盘地址信息数据包
    RegsIn.w.si = BIOSCALLBUFOFFSET;

    // 尝试读取三次
    for (RetryCount=0; RetryCount<3; RetryCount++)
    {
        Int386(0x13, &RegsIn, &RegsOut);

        if (INT386_SUCCESS(RegsOut)) // 读取成功返回
        {
            return TRUE;
        }
        else if (RegsOut.b.ah == 0x11) // 正确的ECC错误，那么数据是好的，可以返回
        {
            return TRUE;
        }
        else // 如果失败了，则进行下一次尝试
        {
            DiskResetController(DriveNumber);
            continue;
        }
    }

    // 走到这里，则是读失败了
    ERR("Disk Read Failed in LBA mode: %x (DriveNumber: 0x%x SectorNumber: %I64d SectorCount: %d)\n", RegsOut.b.ah, DriveNumber, SectorNumber, SectorCount);

    return FALSE;
}
```

判断了磁盘确实存在，且可读取内容，则调用GetHarddiskInformation()获取磁盘的信息。首先读取驱动器的第一个扇区，读取成功则认为是MBR，获取签名标记且计算MBR校验和，然后将它们存入`reactos_arc_disk_info`全局变量和FsRegisterDevice()注册表设备列表中。DiskGetPartitionEntry()实际是DiskGetMbrPartitionEntry()，它读取引导扇区条目，其中包括MBR中的四个分区的启动分区以及扩展分区中的启动分区。

```
static VOID GetHarddiskInformation(UCHAR DriveNumber)
{
    PMASTER_BOOT_RECORD Mbr;
    PULONG Buffer;
    ULONG i;
    ULONG Checksum;
    ULONG Signature;
    CHAR ArcName[MAX_PATH];
    PARTITION_TABLE_ENTRY PartitionTableEntry;
    PCHAR Identifier = PcDiskIdentifier[DriveNumber - 0x80];

    // 读取MBR
    if (!MachDiskReadLogicalSectors(DriveNumber, 0ULL, 1, DiskReadBuffer))
    {
        ERR("Reading MBR failed\n");
        return;
    }

    Buffer = (ULONG*)DiskReadBuffer;
    Mbr = (PMASTER_BOOT_RECORD)DiskReadBuffer;

    Signature = Mbr->Signature;
    TRACE("Signature: %x\n", Signature);

    Checksum = 0; // 计算 MBR校验和
    for (i = 0; i < 512 / sizeof(ULONG); i++)
    {
        Checksum += Buffer[i];
    }
    Checksum = ~Checksum + 1;
    TRACE("Checksum: %x\n", Checksum);

    // 填充ARC 磁盘块
    reactos_arc_disk_info[reactos_disk_count].DiskSignature.Signature = Signature;
    reactos_arc_disk_info[reactos_disk_count].DiskSignature.CheckSum = Checksum;
    sprintf(ArcName, "multi(0)disk(0)rdisk(%lu)", reactos_disk_count);
    strcpy(reactos_arc_disk_info[reactos_disk_count].ArcName, ArcName);
    reactos_arc_disk_info[reactos_disk_count].DiskSignature.ArcName =
        reactos_arc_disk_info[reactos_disk_count].ArcName;
    reactos_disk_count++;

    sprintf(ArcName, "multi(0)disk(0)rdisk(%u)partition(0)", DriveNumber - 0x80);
    FsRegisterDevice(ArcName, &DiskVtbl);

    i = 1;  // 添加分区
    DiskReportError(FALSE);
    while (DiskGetPartitionEntry(DriveNumber, i, &PartitionTableEntry))
    {
        if (PartitionTableEntry.SystemIndicator != PARTITION_ENTRY_UNUSED)
        {
            sprintf(ArcName, "multi(0)disk(0)rdisk(%u)partition(%lu)", DriveNumber - 0x80, i);
            FsRegisterDevice(ArcName, &DiskVtbl);
        }
        i++;
    }
    DiskReportError(TRUE);

    // 将校验和和签名转换为标识串，唯一标识当前磁盘驱动
    Identifier[0] = Hex[(Checksum >> 28) & 0x0F];
    Identifier[1] = Hex[(Checksum >> 24) & 0x0F];
    Identifier[2] = Hex[(Checksum >> 20) & 0x0F];
    Identifier[3] = Hex[(Checksum >> 16) & 0x0F];
    Identifier[4] = Hex[(Checksum >> 12) & 0x0F];
    Identifier[5] = Hex[(Checksum >> 8) & 0x0F];
    Identifier[6] = Hex[(Checksum >> 4) & 0x0F];
    Identifier[7] = Hex[Checksum & 0x0F];
    Identifier[8] = '-';
    Identifier[9] = Hex[(Signature >> 28) & 0x0F];
    Identifier[10] = Hex[(Signature >> 24) & 0x0F];
    Identifier[11] = Hex[(Signature >> 20) & 0x0F];
    Identifier[12] = Hex[(Signature >> 16) & 0x0F];
    Identifier[13] = Hex[(Signature >> 12) & 0x0F];
    Identifier[14] = Hex[(Signature >> 8) & 0x0F];
    Identifier[15] = Hex[(Signature >> 4) & 0x0F];
    Identifier[16] = Hex[Signature & 0x0F];
    Identifier[17] = '-';
    Identifier[18] = 'A'; // FIXME: Not always 'A' ...
    Identifier[19] = 0;
    TRACE("Identifier: %s\n", Identifier);
}

BOOLEAN DiskGetMbrPartitionEntry(UCHAR DriveNumber, ULONG PartitionNumber, PPARTITION_TABLE_ENTRY PartitionTableEntry)
{
    MASTER_BOOT_RECORD MasterBootRecord;
    PARTITION_TABLE_ENTRY ExtendedPartitionTableEntry;
    ULONG ExtendedPartitionNumber;
    ULONG ExtendedPartitionOffset;
    ULONG Index;
    ULONG CurrentPartitionNumber;
    PPARTITION_TABLE_ENTRY ThisPartitionTableEntry;

    // 读取主引导扇区
    if (!DiskReadBootRecord(DriveNumber, 0, &MasterBootRecord))
    {
        return FALSE;
    }

    CurrentPartitionNumber = 0;
    for (Index=0; Index<4; Index++) // 遍历主引导扇区中的分区表，四个项目
    {
        ThisPartitionTableEntry = &MasterBootRecord.PartitionTable[Index];
        if (ThisPartitionTableEntry->SystemIndicator != PARTITION_ENTRY_UNUSED &&
            ThisPartitionTableEntry->SystemIndicator != PARTITION_EXTENDED &&
            ThisPartitionTableEntry->SystemIndicator != PARTITION_XINT13_EXTENDED)
        {
            CurrentPartitionNumber++;
        }

        if (PartitionNumber == CurrentPartitionNumber) // 分区号与当前分区号相等
        {
            RtlCopyMemory(PartitionTableEntry, ThisPartitionTableEntry, sizeof(PARTITION_TABLE_ENTRY));
            return TRUE;
        }
    }

    // 想要扩展分区条目，这里会循环遍历磁盘上的所有的扩展分区
    // 返回想要获取的分区的条目信息
    ExtendedPartitionNumber = PartitionNumber - CurrentPartitionNumber - 1;

    // 设置初始的相对其实扇区为0，因为扩展分区的起始分区是相对于它的父分区计算的
    ExtendedPartitionOffset = 0;
    for (Index=0; Index<=ExtendedPartitionNumber; Index++)
    {
        // 获取扩展分区表条目
        if (!DiskGetFirstExtendedPartitionEntry(&MasterBootRecord, &ExtendedPartitionTableEntry))
        {
            return FALSE;
        }

        // 调整分区的相对起始扇区
        ExtendedPartitionTableEntry.SectorCountBeforePartition += ExtendedPartitionOffset;
        if (ExtendedPartitionOffset == 0)
        {
            // 设置父扩展分区的起始扇区
            ExtendedPartitionOffset = ExtendedPartitionTableEntry.SectorCountBeforePartition;
        }
        // 读取分区引导记录，PBR(DBR)
        if (!DiskReadBootRecord(DriveNumber, ExtendedPartitionTableEntry.SectorCountBeforePartition, &MasterBootRecord))
        {
            return FALSE;
        }

        // 获取第一个真实的分区表条目 MasterBootRecord已经是扩展分区中的分区表了
        if (!DiskGetFirstPartitionEntry(&MasterBootRecord, PartitionTableEntry))
        {
            return FALSE;
        }

        // 纠正分区的起始扇区
        PartitionTableEntry->SectorCountBeforePartition += ExtendedPartitionTableEntry.SectorCountBeforePartition;
    }

    // 执行到这里应该已经得到了正确的条目，返回TRUE即可
    return TRUE;
}

static const DEVVTBL DiskVtbl =
{
    DiskClose,
    DiskGetFileInformation,
    DiskOpen,
    DiskRead,
    DiskSeek,
};

VOID FsRegisterDevice(CHAR* Prefix, const DEVVTBL* FuncTable)
{
    DEVICE* pNewEntry;
    SIZE_T Length;

    TRACE("FsRegisterDevice() Prefix = %s\n", Prefix);
    Length = strlen(Prefix) + 1;
    pNewEntry = FrLdrTempAlloc(sizeof(DEVICE) + Length, TAG_DEVICE);
    if (!pNewEntry)
        return;
    pNewEntry->FuncTable = FuncTable;
    pNewEntry->ReferenceCount = 0;
    pNewEntry->Prefix = (CHAR*)(pNewEntry + 1);
    memcpy(pNewEntry->Prefix, Prefix, Length);

    InsertHeadList(&DeviceListHead, &pNewEntry->ListEntry);
}
```

最后PcInitializeBootDevices()获取正在启动的扇区的路径，调用MachDiskGetBootPath()函数。这个函数也是一个宏定义，其调用函数为PcDiskGetBootPath()。如果全局变量中已经保存，直接复制内容返回；否则调用DiskGetBootPath()函数进行获取。DiskGetBootPath()函数根据全局变量FrldrBootDrive值确定是哪一种引导（软盘，光盘，硬盘等），对于磁盘引导，需要获取引导分区（BootPartition），从而构造引导路径。

```
BOOLEAN PcDiskGetBootPath(OUT PCHAR BootPath, IN ULONG Size)
{
    // FIXME: Keep it there, or put it in DiskGetBootPath?
    // Or, abstract the notion of network booting to make
    // sense for other platforms than the PC (and this idea
    // already exists), then we would need to check whether
    // we were booting from network (and: PC --> PXE, etc...)
    // and if so, set the correct ARC path. But then this new
    // logic could be moved back to DiskGetBootPath...

    if (*FrldrBootPath)
    {
        /* Copy back the buffer */
        if (Size < strlen(FrldrBootPath) + 1)
            return FALSE;
        strncpy(BootPath, FrldrBootPath, Size);
        return TRUE;
    }

    // FIXME! FIXME! Do this in some drive recognition procedure!!!!
    if (PxeInit())
    {
        strcpy(BootPath, "net(0)");
        return TRUE;
    }
    return DiskGetBootPath(BootPath, Size);
}

BOOLEAN DiskGetBootPath(OUT PCHAR BootPath, IN ULONG Size)
{
    if (*FrldrBootPath)
        goto Done;

    /* 0x49 is our magic ramdisk drive, so try to detect it first */
    if (FrldrBootDrive == 0x49)    // ReactOS特有
    {
        /* This is the ramdisk. See ArmDiskGetBootPath too... */
        // sprintf(FrldrBootPath, "ramdisk(%u)", 0);
        strcpy(FrldrBootPath, "ramdisk(0)");
    }
    else if (FrldrBootDrive < 0x80) // 软盘启动
    {
        /* This is a floppy */
        sprintf(FrldrBootPath, "multi(0)disk(0)fdisk(%u)", FrldrBootDrive);
    }
    else if (FrldrBootPartition == 0xFF)  // 光盘启动
    {
        /* Boot Partition 0xFF is the magic value that indicates booting from CD-ROM (see isoboot.S) */
        sprintf(FrldrBootPath, "multi(0)disk(0)cdrom(%u)", FrldrBootDrive - 0x80);
    }
    else   // 硬盘启动
    {
        ULONG BootPartition;
        PARTITION_TABLE_ENTRY PartitionEntry;

        // 硬盘启动
        if (!DiskGetActivePartitionEntry(FrldrBootDrive, &PartitionEntry, &BootPartition))
        {
            ERR("Invalid active partition information\n");
            return FALSE;
        }

        FrldrBootPartition = BootPartition;

        sprintf(FrldrBootPath, "multi(0)disk(0)rdisk(%u)partition(%lu)",
                FrldrBootDrive - 0x80, FrldrBootPartition);
    }

Done:
    // 复制内容到返回参数缓存中
    if (Size < strlen(FrldrBootPath) + 1)
        return FALSE;
    strncpy(BootPath, FrldrBootPath, Size);
    return TRUE;
}
```

DiskGetActivePartitionEntry()函数获取活动分区条目，以及活动分区。同样要读取磁盘的主引导分区表，然后判断是否存在引导分区（BootIndicator是否为0x80，活动分区标记）。这里在结尾处有一个判断条件，只允许存在一个可引导分区。

```
BOOLEAN DiskGetActivePartitionEntry(UCHAR DriveNumber,
                                    PPARTITION_TABLE_ENTRY PartitionTableEntry,
                                    ULONG *ActivePartition)
{
    ULONG BootablePartitionCount = 0;
    ULONG CurrentPartitionNumber;
    ULONG Index;
    MASTER_BOOT_RECORD MasterBootRecord;
    PPARTITION_TABLE_ENTRY ThisPartitionTableEntry;

    *ActivePartition = 0;

    // 读取主引导记录
    if (!DiskReadBootRecord(DriveNumber, 0, &MasterBootRecord))
    {
        return FALSE;
    }

    CurrentPartitionNumber = 0;
    for (Index=0; Index<4; Index++)
    {
        ThisPartitionTableEntry = &MasterBootRecord.PartitionTable[Index];

        if (ThisPartitionTableEntry->SystemIndicator != PARTITION_ENTRY_UNUSED &&
            ThisPartitionTableEntry->SystemIndicator != PARTITION_EXTENDED &&
            ThisPartitionTableEntry->SystemIndicator != PARTITION_XINT13_EXTENDED)
        {
            CurrentPartitionNumber++;

            // Test if this is the bootable partition
            if (ThisPartitionTableEntry->BootIndicator == 0x80)
            {
                BootablePartitionCount++;
                *ActivePartition = CurrentPartitionNumber;

                // Copy the partition table entry
                RtlCopyMemory(PartitionTableEntry,
                              ThisPartitionTableEntry,
                              sizeof(PARTITION_TABLE_ENTRY));
            }
        }
    }

    // Make sure there was only one bootable partition
    if (BootablePartitionCount == 0)
    {
        ERR("No bootable (active) partitions found.\n");
        return FALSE;
    }
    else if (BootablePartitionCount != 1)
    {
        ERR("Too many bootable (active) partitions found.\n");
        return FALSE;
    }

    return TRUE;
}
```

至此PcInitializeBootDevices()函数就解析完了，它的作用就是确定有多少磁盘存在，且存在磁盘要可读数据；根据存在磁盘号，读取其引导信息。最后获取引导路径，它会被保存到全局变量中。

1. FsRegisterDevice()将磁盘设备信息写入DeviceListHead列表中
2. FrldrBootPath全局变量保存了当前的引导路径（multi(0)disk(0)rdisk(0)partition(*)）
3. `reactos_arc_disk_info`保存了磁盘的信息，校验和，签名等

#### 2. Arc文件读写 ####

上面初始化了磁盘驱动器的信息，将所有的磁盘信息统计后保存到了全局变量中，方便在“ARC文件系统”（姑且这么称呼）使用。对于文件的操作其实比较简单，打开文件，读取文件，写文件，寻址文件内容，关闭文件。

在初始化完设备信息后，RunLoader调用了IniFileInitialize()函数，读取INI文件，将ini中的设置信息保存到全局信息中，以备之后的启动过程使用。IniFileInitialize()函数的源代码如下所示。

```
BOOLEAN IniFileInitialize(VOID)
{
    FILEINFORMATION FileInformation;
    ULONG FileId; // File handle for freeldr.ini
    PCHAR FreeLoaderIniFileData;
    ULONG FreeLoaderIniFileSize, Count;
    ARC_STATUS Status;
    BOOLEAN Success;
    TRACE("IniFileInitialize()\n");

    Status = IniOpenIniFile(&FileId); // 打开 freeldr.ini
    if (Status != ESUCCESS)
    {
        UiMessageBoxCritical("Error opening freeldr.ini or file not found.\nYou need to re-install FreeLoader.");
        return FALSE;
    }

    Status = ArcGetFileInformation(FileId, &FileInformation); // 获取文件大小
    if (Status != ESUCCESS || FileInformation.EndingAddress.HighPart != 0)
    {
        UiMessageBoxCritical("Error while getting informations about freeldr.ini.\nYou need to re-install FreeLoader.");
        return FALSE;
    }
    FreeLoaderIniFileSize = FileInformation.EndingAddress.LowPart;

    // 分配内存缓存整个 freeldr.ini
    FreeLoaderIniFileData = FrLdrTempAlloc(FreeLoaderIniFileSize, TAG_INI_FILE);
    if (!FreeLoaderIniFileData)
    {
        UiMessageBoxCritical("Out of memory while loading freeldr.ini.");
        ArcClose(FileId);
        return FALSE;
    }

    // 从磁盘上读取 freeldr.ini文件
    Status = ArcRead(FileId, FreeLoaderIniFileData, FreeLoaderIniFileSize, &Count);
    if (Status != ESUCCESS || Count != FreeLoaderIniFileSize)
    {
        UiMessageBoxCritical("Error while reading freeldr.ini.");
        ArcClose(FileId);
        FrLdrTempFree(FreeLoaderIniFileData, TAG_INI_FILE);
        return FALSE;
    }

    // 解析.ini文件内容
    Success = IniParseFile(FreeLoaderIniFileData, FreeLoaderIniFileSize);

    // 清空申请资源，包括打开文件，申请内存
    ArcClose(FileId);
    FrLdrTempFree(FreeLoaderIniFileData, TAG_INI_FILE);

    return Success;
}
```

代码中包含了文件操作的文件打开函数ArcOpen()，获取文件信息ArcGetFileInformation()，读取文件内容的函数ArcRead()，关闭文件的函数ArcClose()。

首先看打开文件，ArcOpen()方法，第一个参数Path是一个ARC路径（形如“multi(0)disk(0)rdisk(0)partition(*)\FreeLdr.ini"），传出的参数是一个FileId，它其实是FileData数组的一个索引值，也即打开的文件有最大上限数`MAX_FDS`。FileData在前面初始化时也看到了对它的初始化，将所有的FileId初始化为0xFFFFFFFF。

```
#define MAX_FDS 60

typedef struct tagFILEDATA
{
    ULONG DeviceId;
    ULONG ReferenceCount;
    const DEVVTBL* FuncTable;
    const DEVVTBL* FileFuncTable;
    VOID* Specific;
} FILEDATA;

typedef struct tagDEVICE
{
    LIST_ENTRY ListEntry;
    const DEVVTBL* FuncTable;
    CHAR* Prefix;
    ULONG DeviceId;
    ULONG ReferenceCount;
} DEVICE;

static FILEDATA FileData[MAX_FDS];
static LIST_ENTRY DeviceListHead;
```

看代码之前，先看两个结构体，一个是FILEDATA，即FileData数组的成员类型，里面包含了FileId，即ArcOpen的返回值；ReferenceCount标识文件被引用计数；FuncTable为磁盘等设备操作的函数，它们与设备相关，即读取磁盘和读取CD的这组函数不同；FileFuncTable为文件系统的文件操作函数。另外一个结构体为DEVICE，它是设备列表DeviceListHead的元素类型，可见它内部包含了必要的成员，保存设备信息。其中DeviceId用于标识设备号，FileData数组的索引值，标识当前设备是否被打开。FuncTable保存的是操作当前设备的函数；ReferenceCount为当前设备的引用计数，标识是否打开以及被引用的次数；Prefix就是前面的ArcName，类似引导路径字符串。

```
ARC_STATUS ArcOpen(CHAR* Path, OPENMODE OpenMode, ULONG* FileId)
{
    ARC_STATUS Status;
    ULONG Count, i;
    PLIST_ENTRY pEntry;
    DEVICE* pDevice;
    CHAR* DeviceName;
    CHAR* FileName;
    CHAR* p;
    CHAR* q;
    SIZE_T Length;
    OPENMODE DeviceOpenMode;
    ULONG DeviceId;

    // 打印状态信息
    TRACE("Opening file '%s'...\n", Path);
    *FileId = MAX_FDS;

    // 搜索最后一个')'，它分割了设备与路径
    FileName = strrchr(Path, ')');
    if (!FileName)
        return EINVAL;
    FileName++;

    // 计算"()"数量，将它替代为"(0)"
    Count = 0;
    for (p = Path; p != FileName; p++)
    {
        if (*p == '(' && *(p + 1) == ')')
            Count++;
    }

    // 复制设备名字，将"()"替换为"(0)"
    Length = FileName - Path + Count;
    if (Count != 0)
    {
        DeviceName = FrLdrTempAlloc(FileName - Path + Count, TAG_DEVICE_NAME);
        if (!DeviceName)
            return ENOMEM;
        for (p = Path, q = DeviceName; p != FileName; p++)
        {
            *q++ = *p;
            if (*p == '(' && *(p + 1) == ')')
                *q++ = '0';
        }
    }
    else
    {
        DeviceName = Path;
    }

    // 搜索设备
    if (OpenMode == OpenReadOnly || OpenMode == OpenWriteOnly)
        DeviceOpenMode = OpenMode;
    else
        DeviceOpenMode = OpenReadWrite;

    pEntry = DeviceListHead.Flink;
    while (pEntry != &DeviceListHead)
    {
        pDevice = CONTAINING_RECORD(pEntry, DEVICE, ListEntry);
        if (strncmp(pDevice->Prefix, DeviceName, Length) == 0)
        {
            // 设备名称相同，则认为找到了
            if (pDevice->ReferenceCount == 0)
            {
                // 搜索空白FileData成员
                for (DeviceId = 0; DeviceId < MAX_FDS; DeviceId++)
                {
                    if (!FileData[DeviceId].FuncTable)
                        break;
                }
                if (DeviceId == MAX_FDS)
                    return EMFILE;

                // 打开设备
                FileData[DeviceId].FuncTable = pDevice->FuncTable;
                Status = pDevice->FuncTable->Open(pDevice->Prefix, DeviceOpenMode, &DeviceId);
                if (Status != ESUCCESS)
                {
                    FileData[DeviceId].FuncTable = NULL;
                    return Status;
                }
                else if (!*FileName)
                {
                    // 调用者想要打开原始设备
                    *FileId = DeviceId;
                    pDevice->ReferenceCount++;
                    return ESUCCESS;
                }

                // 尝试检测文件系统
#ifndef _M_ARM
                FileData[DeviceId].FileFuncTable = IsoMount(DeviceId);
                if (!FileData[DeviceId].FileFuncTable)
#endif
                    FileData[DeviceId].FileFuncTable = FatMount(DeviceId);
#ifndef _M_ARM
                if (!FileData[DeviceId].FileFuncTable)
                    FileData[DeviceId].FileFuncTable = NtfsMount(DeviceId);
                if (!FileData[DeviceId].FileFuncTable)
                    FileData[DeviceId].FileFuncTable = Ext2Mount(DeviceId);
#endif
#if defined(_M_IX86) || defined(_M_AMD64)
                if (!FileData[DeviceId].FileFuncTable)
                    FileData[DeviceId].FileFuncTable = PxeMount(DeviceId);
#endif
                if (!FileData[DeviceId].FileFuncTable)
                {
                    // 无法检测文件系统
                    pDevice->FuncTable->Close(DeviceId);
                    FileData[DeviceId].FuncTable = NULL;
                    return ENODEV;
                }

                pDevice->DeviceId = DeviceId;
            }
            else
            {
                DeviceId = pDevice->DeviceId;
            }
            pDevice->ReferenceCount++;
            break;
        }
        pEntry = pEntry->Flink;
    }
    if (pEntry == &DeviceListHead)
        return ENODEV;

    // 执行到这里，已经找到设备并打开，保存了FileId，FileData[DeviceId].FileFuncTable
    // 包含了打开文件所需要的调用的函数。
    /* Search some room for the device */
    for (i = 0; i < MAX_FDS; i++)
    {
        if (!FileData[i].FuncTable)
            break;
    }
    if (i == MAX_FDS)
        return EMFILE;

    // 跳过前面的反斜线
    if (*FileName == '\\')
        FileName++;

    // 打开文件
    FileData[i].FuncTable = FileData[DeviceId].FileFuncTable;
    FileData[i].DeviceId = DeviceId;
    *FileId = i; // 调用FatOpen()
    Status = FileData[i].FuncTable->Open(FileName, OpenMode, FileId);
    if (Status != ESUCCESS)
    {
        FileData[i].FuncTable = NULL;
        *FileId = MAX_FDS;
    }
    return Status;
}

static ARC_STATUS DiskOpen(CHAR* Path, OPENMODE OpenMode, ULONG* FileId)
{
    DISKCONTEXT* Context;
    UCHAR DriveNumber;
    ULONG DrivePartition, SectorSize;
    ULONGLONG SectorOffset = 0;
    ULONGLONG SectorCount = 0;
    PARTITION_TABLE_ENTRY PartitionTableEntry;
    CHAR FileName[1];

    if (!DissectArcPath(Path, FileName, &DriveNumber, &DrivePartition))
        return EINVAL;

    if (DrivePartition == 0xff)
    {
        SectorSize = 2048;  // 是一个CD-ROM设备
    }
    else
    {
        // 或者是软盘设备(DrivePartition == 0)或者是磁盘设备
        // (DrivePartition != 0 && DrivePartition != 0xFF)
        // 无论是那个，他们的扇区大小都是 512字节
        SectorSize = 512;
    }

    if (DrivePartition != 0xff && DrivePartition != 0)
    {
        if (!DiskGetPartitionEntry(DriveNumber, DrivePartition, &PartitionTableEntry))
            return EINVAL;

        SectorOffset = PartitionTableEntry.SectorCountBeforePartition;
        SectorCount = PartitionTableEntry.PartitionSectorCount;
    }
#if 0 // FIXME: Investigate
    else
    {
        SectorCount = 0; /* FIXME */
    }
#endif

    Context = FrLdrTempAlloc(sizeof(DISKCONTEXT), TAG_HW_DISK_CONTEXT);
    if (!Context)
        return ENOMEM;

    Context->DriveNumber = DriveNumber;
    Context->SectorSize = SectorSize;
    Context->SectorOffset = SectorOffset;
    Context->SectorCount = SectorCount;
    Context->SectorNumber = 0;
    FsSetDeviceSpecific(*FileId, Context); // 放到DEVICE的Specific字段

    return ESUCCESS;
}

const DEVVTBL FatFuncTable =
{
    FatClose,
    FatGetFileInformation,
    FatOpen,
    FatRead,
    FatSeek,
    L"fastfat",
};

const DEVVTBL* FatMount(ULONG DeviceId)
{
    PFAT_VOLUME_INFO Volume;
    UCHAR Buffer[512];
    PFAT_BOOTSECTOR BootSector = (PFAT_BOOTSECTOR)Buffer;
    PFAT32_BOOTSECTOR BootSector32 = (PFAT32_BOOTSECTOR)Buffer;
    PFATX_BOOTSECTOR BootSectorX = (PFATX_BOOTSECTOR)Buffer;
    FILEINFORMATION FileInformation;
    LARGE_INTEGER Position;
    ULONG Count;
    ULARGE_INTEGER SectorCount;
    ARC_STATUS Status;

    // 为卷信息分配数据结构
    Volume = FrLdrTempAlloc(sizeof(FAT_VOLUME_INFO), TAG_FAT_VOLUME);
    if (!Volume)
        return NULL;
    RtlZeroMemory(Volume, sizeof(FAT_VOLUME_INFO));

    // 读取引导扇区
    Position.HighPart = 0;
    Position.LowPart = 0;
    Status = ArcSeek(DeviceId, &Position, SeekAbsolute);
    if (Status != ESUCCESS)
    {
        FrLdrTempFree(Volume, TAG_FAT_VOLUME);
        return NULL;
    }
    Status = ArcRead(DeviceId, Buffer, sizeof(Buffer), &Count);
    if (Status != ESUCCESS || Count != sizeof(Buffer))
    {
        FrLdrTempFree(Volume, TAG_FAT_VOLUME);
        return NULL;
    }

    // 校验是否引导扇区有效，无效则返回
    if (!RtlEqualMemory(BootSector->FileSystemType, "FAT12   ", 8) &&
        !RtlEqualMemory(BootSector->FileSystemType, "FAT16   ", 8) &&
        !RtlEqualMemory(BootSector32->FileSystemType, "FAT32   ", 8) &&
        !RtlEqualMemory(BootSectorX->FileSystemType, "FATX", 4))
    {
        FrLdrTempFree(Volume, TAG_FAT_VOLUME);
        return NULL;
    }

    // 确定扇区数量
    Status = ArcGetFileInformation(DeviceId, &FileInformation);
    if (Status != ESUCCESS)
    {
        FrLdrTempFree(Volume, TAG_FAT_VOLUME);
        return NULL;
    }
    SectorCount.HighPart = FileInformation.EndingAddress.HighPart;
    SectorCount.LowPart = FileInformation.EndingAddress.LowPart;
    SectorCount.QuadPart /= SECTOR_SIZE;

    Volume->DeviceId = DeviceId; // 缓存设备ID，用于打开设备检索

    // 打开卷，获取卷的信息
    if (!FatOpenVolume(Volume, BootSector, SectorCount.QuadPart))
    {
        FrLdrTempFree(Volume, TAG_FAT_VOLUME);
        return NULL;
    }

    FatVolumes[DeviceId] = Volume;   // 记录FAT卷信息

    return &FatFuncTable;  // 成功，返回操作函数
}

ARC_STATUS FatOpen(CHAR* Path, OPENMODE OpenMode, ULONG* FileId)
{
    PFAT_VOLUME_INFO FatVolume;
    FAT_FILE_INFO TempFileInfo;
    PFAT_FILE_INFO FileHandle;
    ULONG DeviceId;
    BOOLEAN IsDirectory;
    ARC_STATUS Status;

    if (OpenMode != OpenReadOnly && OpenMode != OpenDirectory)
        return EACCES;

    DeviceId = FsGetDeviceId(*FileId);
    FatVolume = FatVolumes[DeviceId];

    TRACE("FatOpen() FileName = %s\n", Path);
    RtlZeroMemory(&TempFileInfo, sizeof(TempFileInfo));
    Status = FatLookupFile(FatVolume, Path, DeviceId, &TempFileInfo);
    if (Status != ESUCCESS)
        return ENOENT;

    // FatLookupFile查找文件，校验获取文件信息是否和调用者预期打开的文件/目录相同
    IsDirectory = (TempFileInfo.Attributes & ATTR_DIRECTORY) != 0;
    if (IsDirectory && OpenMode != OpenDirectory)
        return EISDIR;
    else if (!IsDirectory && OpenMode != OpenReadOnly)
        return ENOTDIR;

    FileHandle = FrLdrTempAlloc(sizeof(FAT_FILE_INFO), TAG_FAT_FILE);
    if (!FileHandle)
        return ENOMEM;

    RtlCopyMemory(FileHandle, &TempFileInfo, sizeof(FAT_FILE_INFO));
    FileHandle->Volume = FatVolume;

    FsSetDeviceSpecific(*FileId, FileHandle);
    return ESUCCESS;
}
```

注意这里的FileData数组中，一方面保存打开的文件，另外一方面保存打开的设备。如果一个元素的保存的是设备信息，它的DeviceId为空，但是FuncTable保存了磁盘操作函数，FileFuncTable保存了文件系统操作函数；对于保存文件信息的元素，DeviceId保存了文件所在的设备在FileData数组中的索引，FuncTable包含的是文件系统的文件操作函数，FileFuncTable则为空。

ArcOpen()函数一方面要调用DiskOpen()打开磁盘，其实就是获取要打开的文件所在磁盘的信息，会将信息保存到DISKCONTEXT结构体中，然后将动态分配的结构体内存挂到打开设备对应的FileData数组中对应元素的Specific字段。

```
typedef struct tagDISKCONTEXT
{
    UCHAR DriveNumber;
    ULONG SectorSize;
    ULONGLONG SectorOffset;
    ULONGLONG SectorCount;
    ULONGLONG SectorNumber;
} DISKCONTEXT;
```

这里分析的是ReactOS系统启动过程，所以使用硬盘引导，而硬盘被设置的是FAT32文件格式，所以这里会调用FatMount()加载文件系统，它会将DeviceId索引的设备信息记录到FatVolumes全局数组中，索引使用设备在FileData数组中相同的值，一一对应；最后操作成功则返回FAT文件系统的操作函数集合。打开文件调用的Open函数即是FatOpen()函数。

FatOpen()调用FatLookupFile()在磁盘上查找文件，保存FAT_FILE_INFO结构中，其中涉及到对FAT32文件系统的分析，不做详细分析，可以参考FAT32文件系统格式。找到数据之后将它赋值到该文件对应FileData数组中元素的Specific元素中，以作后用。

打开一个文件，即将磁盘设备信息存放到一个FileData数组的元素中，文件信息存放到一个FileData数组的元素中，文件信息链接到设备信息上，返回文件信息的索引值，即完成文件打开。

剩下的几个函数就比较简单了，源码如下所示，都是调用对应的FuncTable中的函数，其实就是FatGetFileInformation()，FatRead()，FatClose()等。

```
ARC_STATUS ArcClose(ULONG FileId)
{
    ARC_STATUS Status;

    if (FileId >= MAX_FDS || !FileData[FileId].FuncTable)
        return EBADF;

    Status = FileData[FileId].FuncTable->Close(FileId);
    if (Status == ESUCCESS)
    {
        FileData[FileId].FuncTable = NULL; // 将操作函数设置为NULL
        FileData[FileId].Specific = NULL;
        FileData[FileId].DeviceId = -1;
    }
    return Status;
}

ARC_STATUS ArcRead(ULONG FileId, VOID* Buffer, ULONG N, ULONG* Count)
{
    if (FileId >= MAX_FDS || !FileData[FileId].FuncTable)
        return EBADF;
    return FileData[FileId].FuncTable->Read(FileId, Buffer, N, Count);
}

ARC_STATUS ArcSeek(ULONG FileId, LARGE_INTEGER* Position, SEEKMODE SeekMode)
{
    if (FileId >= MAX_FDS || !FileData[FileId].FuncTable)
        return EBADF;
    return FileData[FileId].FuncTable->Seek(FileId, Position, SeekMode);
}

ARC_STATUS ArcGetFileInformation(ULONG FileId, FILEINFORMATION* Information)
{
    if (FileId >= MAX_FDS || !FileData[FileId].FuncTable)
        return EBADF;
    return FileData[FileId].FuncTable->GetFileInformation(FileId, Information);
}
```

这些函数如下代码块所示。在前面打开文件时已经获取文件的基本信息，所以FatGetFileInformation()就比较简单了，直接将之前的`FAT_FILE_INFO`结构体中的数据赋值一下即可。

```
ARC_STATUS FatGetFileInformation(ULONG FileId, FILEINFORMATION* Information)
{
    PFAT_FILE_INFO FileHandle = FsGetDeviceSpecific(FileId);

    RtlZeroMemory(Information, sizeof(FILEINFORMATION));
    Information->EndingAddress.LowPart = FileHandle->FileSize;
    Information->CurrentAddress.LowPart = FileHandle->FilePointer;

    TRACE("FatGetFileInformation() FileSize = %d\n",
        Information->EndingAddress.LowPart);
    TRACE("FatGetFileInformation() FilePointer = %d\n",
        Information->CurrentAddress.LowPart);

    return ESUCCESS;
}
```

FatRead()函数很简单，它将调用转到了FatReadFile()方法中，FatReadFile()完成数据读取。由于读取数据时是按照簇进行读取，所以要根据文件大小计算读取的簇数。根据文件所处位置，以及文件大小，可能会出现多种可能性。函数中给出了一种通用的情况，根据这种情况来设计代码逻辑，具体内容可以查看下面的源代码及其注释。

读取内容使用两个函数，FatReadPartialCluster()读一部分簇内容；FatReadClusterChain()函数读取几个整簇的数据。两个函数最后都会调用到FatReadVolumeSectors()函数，该函数进一步调用到ArcRead()函数。ArcRead()函数在上面见到过，与前面不同的地方是第一个参数传入的是DeviceId，找到文件对应的设备的FileData数组元素。它的FuncTable对应Disk*类的几个函数，实际调用的是DiskRead()函数。

```
ARC_STATUS FatRead(ULONG FileId, VOID* Buffer, ULONG N, ULONG* Count)
{
    PFAT_FILE_INFO FileHandle = FsGetDeviceSpecific(FileId);
    BOOLEAN Success;

    // 调用老式的 读方法
    Success = FatReadFile(FileHandle, N, Count, Buffer);

    // 检查是否成功
    if (Success)
        return ESUCCESS;
    else
        return EIO;
}

/*
 * FatReadFile()
 * 读取BytesToRead字节的数据，在BytesRead中返回读出的字节数
 */
BOOLEAN FatReadFile(PFAT_FILE_INFO FatFileInfo, ULONG BytesToRead, ULONG* BytesRead, PVOID Buffer)
{
    PFAT_VOLUME_INFO Volume = FatFileInfo->Volume;
    ULONG            ClusterNumber;
    ULONG            OffsetInCluster;
    ULONG            LengthInCluster;
    ULONG            NumberOfClusters;
    ULONG            BytesPerCluster;

    TRACE("FatReadFile() BytesToRead = %d Buffer = 0x%x\n", BytesToRead, Buffer);
    if (BytesRead != NULL)
    {
        *BytesRead = 0;
    }

    // 如果要读取的内容位置比文件本身大了，则直接返回成功且BytesRead == 0
    if (FatFileInfo->FilePointer >= FatFileInfo->FileSize)
    {
        return TRUE;
    }

    // 如果要读内容多于文件本身，则重置读取字节数
    if ((FatFileInfo->FilePointer + BytesToRead) > FatFileInfo->FileSize)
    {
        BytesToRead = (FatFileInfo->FileSize - FatFileInfo->FilePointer);
    }

    //
    // 现在必须做最多三次计算。下面画了一个图
    // CurrentFilePointer -+
    //                     |
    //    +----------------+
    //    |
    // +-----------+-----------+-----------+-----------+
    // | Cluster 1 | Cluster 2 | Cluster 3 | Cluster 4 |
    // +-----------+-----------+-----------+-----------+
    //    |                                    |
    //    +---------------+--------------------+
    //                    |
    // BytesToRead -------+
    //
    // 1 - 第一个计算（和读取）会将文件指针对齐到下一个簇
    //     边界（如果支持读取那么多）
    // 2 - 下一个计算（和读取）会读取文件覆盖了的所有完整簇的，即2簇和3簇
    // 3 - 最后一次计算（和读取）会读入最后一簇中剩下的数据
    //
    BytesPerCluster = Volume->SectorsPerCluster * Volume->BytesPerSector;

    // 如果文件指针没有对齐到簇边界，要进行第一次计算
    if (FatFileInfo->FilePointer % BytesPerCluster)
    {
        // 第一次读时计算
        ClusterNumber = (FatFileInfo->FilePointer / BytesPerCluster);
        ClusterNumber = FatFileInfo->FileFatChain[ClusterNumber];
        OffsetInCluster = (FatFileInfo->FilePointer % BytesPerCluster);
        LengthInCluster = (BytesToRead > (BytesPerCluster - OffsetInCluster)) ? (BytesPerCluster - OffsetInCluster) : BytesToRead;

        // 读取数据，更新 BytesRead, BytesToRead, FilePointer, & Buffer
        if (!FatReadPartialCluster(Volume, ClusterNumber, OffsetInCluster, LengthInCluster, Buffer))
        {
            return FALSE;
        }
        if (BytesRead != NULL)
        {
            *BytesRead += LengthInCluster;
        }
        BytesToRead -= LengthInCluster;
        FatFileInfo->FilePointer += LengthInCluster;
        Buffer = (PVOID)((ULONG_PTR)Buffer + LengthInCluster);
    }

    // 第二次读数据时做计算（如果一个簇没有包含所有文件数据）
    if (BytesToRead > 0)
    {
        // 确定需要读几个完整的簇
        NumberOfClusters = (BytesToRead / BytesPerCluster);
        if (NumberOfClusters > 0)
        {
            ClusterNumber = (FatFileInfo->FilePointer / BytesPerCluster);
            ClusterNumber = FatFileInfo->FileFatChain[ClusterNumber];

            // 读数据，更新BytesRead, BytesToRead, FilePointer, & Buffer
            if (!FatReadClusterChain(Volume, ClusterNumber, NumberOfClusters, Buffer))
            {
                return FALSE;
            }
            if (BytesRead != NULL)
            {
                *BytesRead += (NumberOfClusters * BytesPerCluster);
            }
            BytesToRead -= (NumberOfClusters * BytesPerCluster);
            FatFileInfo->FilePointer += (NumberOfClusters * BytesPerCluster);
            Buffer = (PVOID)((ULONG_PTR)Buffer + (NumberOfClusters * BytesPerCluster));
        }
    }

    // 如果完整簇没有读完，则还需要再计算一次
    if (BytesToRead > 0)
    {
        ClusterNumber = (FatFileInfo->FilePointer / BytesPerCluster);
        ClusterNumber = FatFileInfo->FileFatChain[ClusterNumber];

        // 读数据，更新BytesRead, BytesToRead, FilePointer, & Buffer
        if (!FatReadPartialCluster(Volume, ClusterNumber, 0, BytesToRead, Buffer))
        {
            return FALSE;
        }
        if (BytesRead != NULL)
        {
            *BytesRead += BytesToRead;
        }
        FatFileInfo->FilePointer += BytesToRead;
        BytesToRead -= BytesToRead;
        Buffer = (PVOID)((ULONG_PTR)Buffer + BytesToRead);
    }

    return TRUE;
}

BOOLEAN FatReadVolumeSectors(PFAT_VOLUME_INFO Volume, ULONG SectorNumber, ULONG SectorCount, PVOID Buffer)
{
    LARGE_INTEGER Position;
    ULONG Count;
    ARC_STATUS Status;

    // 搜索到正确位置
    Position.QuadPart = (ULONGLONG)SectorNumber * 512;
    Status = ArcSeek(Volume->DeviceId, &Position, SeekAbsolute);
    if (Status != ESUCCESS)
    {
        TRACE("FatReadVolumeSectors() Failed to seek\n");
        return FALSE;
    }

    // 读取数据
    Status = ArcRead(Volume->DeviceId, Buffer, SectorCount * 512, &Count);
    if (Status != ESUCCESS || Count != SectorCount * 512)
    {
        TRACE("FatReadVolumeSectors() Failed to read\n");
        return FALSE;
    }

    // Return success
    return TRUE;
}

static ARC_STATUS
DiskRead(ULONG FileId, VOID* Buffer, ULONG N, ULONG* Count)
{
    DISKCONTEXT* Context = FsGetDeviceSpecific(FileId);
    UCHAR* Ptr = (UCHAR*)Buffer;
    ULONG Length, TotalSectors, MaxSectors, ReadSectors;
    BOOLEAN ret;
    ULONGLONG SectorOffset;

    TotalSectors = (N + Context->SectorSize - 1) / Context->SectorSize;
    MaxSectors   = DiskReadBufferSize / Context->SectorSize;
    SectorOffset = Context->SectorNumber + Context->SectorOffset;

    ret = TRUE;

    while (TotalSectors)
    {
        ReadSectors = TotalSectors;
        if (ReadSectors > MaxSectors)
            ReadSectors = MaxSectors;

        ret = MachDiskReadLogicalSectors(Context->DriveNumber,
                                         SectorOffset,
                                         ReadSectors,
                                         DiskReadBuffer);
        if (!ret)
            break;

        Length = ReadSectors * Context->SectorSize;
        if (Length > N)
            Length = N;

        RtlCopyMemory(Ptr, DiskReadBuffer, Length);

        Ptr += Length;
        N -= Length;
        SectorOffset += ReadSectors;
        TotalSectors -= ReadSectors;
    }

    *Count = (ULONG)(Ptr - (UCHAR*)Buffer);

    return (!ret) ? EIO : ESUCCESS;
}

ARC_STATUS FatSeek(ULONG FileId, LARGE_INTEGER* Position, SEEKMODE SeekMode)
{
    PFAT_FILE_INFO FileHandle = FsGetDeviceSpecific(FileId);

    TRACE("FatSeek() NewFilePointer = %lu\n", Position->LowPart);

    if (SeekMode != SeekAbsolute)
        return EINVAL;
    if (Position->HighPart != 0)
        return EINVAL;
    if (Position->LowPart >= FileHandle->FileSize)
        return EINVAL;

    FileHandle->FilePointer = Position->LowPart;
    return ESUCCESS;
}

ARC_STATUS FatClose(ULONG FileId)
{
    PFAT_FILE_INFO FileHandle = FsGetDeviceSpecific(FileId);

    if (FileHandle->FileFatChain) FrLdrTempFree(FileHandle->FileFatChain, TAG_FAT_CHAIN);
    FrLdrTempFree(FileHandle, TAG_FAT_FILE);

    return ESUCCESS;
}
```

DiskRead()函数其实调用的是MachDiskReadLogicalSectors()函数，它对应于PcDiskReadLogicalSectors()函数，这个函数在之前的内容中解释过。

FatSeek()比较简单，判断参数的有效性后，直接将打开文件的读取指针FilePointer设置偏移即可。FatClose()函数释放掉打开文件时文件存储在磁盘存储的信息。这些函数都比较简单，不再详述。

** 参考资料 **

ReactOS FreeLdr源码
[ReactOS-Freeldr磁盘及文件管理](https://0cch.com/2011/06/02/reactos-freeldre7a381e79b98e58f8ae69687e4bbb6e7aea1e79086/)

**修订历史**

* 2017-09-06 17:21:23		完成文章

By Andy @2017-09-06 17:21:23