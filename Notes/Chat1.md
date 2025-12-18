User: Introduce me to this codebase. I am specifically interested in the steps that occur during the bare-metal boot. Provide me some notes.

GitHub Copilot: I'll help you understand the TinkerOS codebase and its bare-metal boot process. Let me explore the relevant files. 

Read [](file:///workspaces/TinkerOS/README.md)

Read [](file:///workspaces/TinkerOS/Baremetal.md)

Read [](file:///workspaces/TinkerOS/StartOS.HC)

Read [](file:///workspaces/TinkerOS/Adam/BootRAM.HC)

Read [](file:///workspaces/TinkerOS/Kernel)

Read [](file:///workspaces/TinkerOS/MakeHome.HC)

Read [](file:///workspaces/TinkerOS/Kernel/KMain.HC#1-1), lines 1 to 200

Read [](file:///workspaces/TinkerOS/Kernel/KStart64.HC)

Read [](file:///workspaces/TinkerOS/Once.HC#1-1), lines 1 to 80

Read [](file:///workspaces/TinkerOS/Kernel/KStart16.HC#1-1), lines 1 to 150

Read [](file:///workspaces/TinkerOS/Kernel/KStart32.HC)

Searched for text `CORE0_32BIT_INIT`, 6 results

Read [](file:///workspaces/TinkerOS/Kernel/KLoad.HC#1-1), lines 1 to 100

Searched for text `BlkDevsInitAll`, 3 results

Read [](file:///workspaces/TinkerOS/Kernel/BlkDev/DskAddDev.HC#100-100), lines 100 to 383

Read [](file:///workspaces/TinkerOS/Adam/MakeAdam.HC)

Perfect! Now I have enough context to provide comprehensive notes. Let me compile the boot sequence information:

## TinkerOS Bare-Metal Boot Sequence - Technical Notes

### **Overview**
TinkerOS is a fork of TempleOS (95% compatible) that improves hardware support for modern machines. It's a 64-bit operating system written in HolyC that requires legacy BIOS boot (CSM mode), PS/2 keyboard/mouse, and supports both IDE/Legacy SATA and AHCI SATA modes.

---

### **Boot Sequence Stages**

#### **1. Stage 1: 16-bit Real Mode (KStart16.HC)**

**Entry Point:** `CORE0_16BIT_INIT`

The kernel begins as a 16-bit binary loaded by the bootloader at address `MEM_BOOT_BASE`.

**Key Operations:**
- Sets up stack at `(BOOT_RAM_LIMIT-BOOT_STK_SIZE) / 16`
- Relocates code to proper segment
- Sets `SYS_RUN_LEVEL` to `RLF_16BIT`
- **VBE Graphics Initialization** (unless text mode):
  - Queries BIOS INT 0x10 for VBE 2.0+ support
  - Scans available video modes matching 32-bit color depth
  - Finds mode matching `FB_WIDTH` x `FB_HEIGHT` 
  - Stores frame buffer address in `SYS_FRAME_BUFFER`
  - Saves mode information in `VBE_MODE` structure
- **Memory Detection:**
  - Uses BIOS INT 0x15 E820 function to build memory map
  - Stores results in `MEM_E820` array
  - Falls back to E801 function if needed
- Loads GDT (Global Descriptor Table)
- Jumps to 32-bit protected mode: `JMP CGDT.boot_cs:CORE0_32BIT_INIT`

---

#### **2. Stage 2: 32-bit Protected Mode (KStart32.HC)**

**Entry Point:** `CORE0_32BIT_INIT`

**Key Operations:**
- Sets `SYS_RUN_LEVEL` to `RLF_32BIT`
- **Address Patching:**
  - Reads patch table at `SYS_BOOT_PATCH_TABLE_BASE`
  - Relocates all absolute addresses by adding `MEM_BOOT_BASE`
  - Sets `RLf_PATCHED` flag
- **Hardware Discovery:**
  - `SYS_FIND_PCIBIOS_SERVICE_DIR`: Locates PCI BIOS services
  - `SYS_FIND_PCI_SERVICES`: Enumerates PCI devices
- **Page Table Initialization:**
  - `SYS_INIT_PAGE_TABLES`: Sets up multi-level paging
    - PML4 (512 GB level)
    - PML3 (1 GB level)  
    - PML2 (2 MB level)
  - Identity maps physical memory
- **Heap Initialization:**
  - `SYS_INIT_16MEG_SYS_CODE_BP`: Sets up 16MB system code base pointer
- **Long Mode Transition:**
  - Calls `SYS_ENTER_LONG_MODE`
  - Enables PAE (Physical Address Extension) in CR4
  - Sets PML4 address in CR3
  - Enables IA32_EFER.LME (Long Mode Enable)
  - Enables paging in CR0
  - Far jump to 64-bit code segment

---

#### **3. Stage 3: 64-bit Long Mode (KStart64.HC)**

**Key Operations:**
- Sets `RLf_64BIT` flag
- **SSE/FPU Initialization:**
  - Enables OSFXSR bit in CR4 for SSE
  - Clears EM (emulation) bit in CR0
  - Initializes FPU with FNINIT
  - Saves initial FPU state with FXSAVE
- **CPU0 Initialization:**
  - Sets up first CPU structure at `SYS_FIXED_AREA`
  - Initializes GS base register for CPU-local storage
- **Adam Task Setup:**
  - Creates Adam HeapCtrl at `SYS_FIXED_AREA+CSysFixedArea.adam_hc`
  - Sets heap signature `HEAP_CTRL_SIGNATURE_VAL`
  - Initializes memory management linked lists
  - Sets task signature `TASK_SIGNATURE_VAL`
  - Sets stack to `MEM_ADAM_STK`
  - Calls `TaskInit` for the Adam task
  - Sets FS base register for task-local storage
- **Jump to Kernel Main:**
  - `JMP KMain` - enters C code

---

#### **4. Stage 4: Kernel Initialization (KMain.HC)**

**Entry Point:** `KMain()`

**Key Operations:**

**4.1 Global System Initialization:**
- `BlkPoolsInit`: Initialize block device memory pools
- `SysGlblsInit`: 
  - Detect cache line width via CPUID
  - Check for hypervisor (reads CPUID 0x40000000)
  - Initialize keyboard state tables
  - Setup lookup tables (reverse bits, set bits)
  - Initialize disk cache
  - Setup floating point tables (pow10 lookup)
  - Initialize sound queues
  - Initialize PCI device list

**4.2 Memory & Graphics:**
- `Mem32DevInit`: Initialize 32-bit device memory regions
- `UncachedAliasAlloc`: Allocate uncached memory alias for MMIO
- `LoadKernel`: Load kernel symbols and exports
- `SysGrInit`: Initialize graphics subsystem
  - Text mode: 80x25 VGA at 0xB8000
  - Graphics mode: `GR_WIDTH` x `GR_HEIGHT` frame buffer
  - Allocate screen buffers (`raw_scrn_image`, `raw_scrn_img2`)

**4.3 Task & Interrupt Setup:**
- Setup Adam task name: "Adam Task CPU00"
- `SysDefinesLoad`: Load system defines
- `Core0Init`: Initialize CPU0 specific structures
- `IntInit1`: Stage 1 interrupt initialization
- Set `RLf_RAW` flag (can now use RawPrint for debugging)
- Print banner: "TinkerOS V%5.3f"
- `TimersInit`: Program PIT timer 0 to `SYS_TIMER0_PERIOD`
- `IntsInit`: Full interrupt descriptor table setup
- `IntInit2`: Enable interrupt controller
- Execute WBINVD (Write Back and Invalidate Cache)
- Enable interrupts: `SetRFlags(RFLAGG_NORMAL)`
- Set `RLf_INTERRUPTS` flag
- `TimeCal`: Calibrate timestamp counter

**4.4 Block Device Initialization:**
- `BlkDevsInitAll`: Critical boot function
  - Detects AHCI controller presence via `AHCIInit`
  - If AHCI found:
    - Sets `AHCIMode=TRUE`
    - Calls `SwitchToAHCI` (changes drive types to AHCI)
    - Probes AHCI SATA/SATAPI devices
  - If no AHCI:
    - Falls back to IDE/Legacy mode
    - Probes IDE ATA/ATAPI devices via port I/O
  - On failure: prints error and halts
  - Partitions are detected via MBR/GPT reading
  - Supports FAT32, RedSea, NTFS, ISO9660 filesystems

**4.5 InitRAMFS Mounting (if present):**
- If `sys_initramfs_ptr` set (Live USB boot):
  - Unmounts existing 'B:' drive
  - Creates RAM disk at 'B:'
  - Sets type `BDT_RAM`
  - Mounts initramfs from memory
  - Creates Home and `/Tmp` directories

**4.6 Root Filesystem:**
- `DskChg(':')`: Change to boot drive
- Loads boot drive symbol tables
- **Executes StartOS.HC**

---

#### **5. Stage 5: System Services (StartOS.HC)**

**Key Operations:**
- Disable echo: `Option(0,0)`
- Include kernel headers (`KernelA.HH`, `KernelB.HH`, `KernelC.HH`)
- Include compiler headers (`CompilerA.HH`, `CompilerB.HH`)
- Enable warnings (parentheses, duplicate types)
- Purge Adam's hash table
- `#include "/Adam/MakeAdam"`: Compile Adam system services
  - Loads core libraries: memory, math, graphics, devices
  - Compiles compiler and debugger
  - Initializes DolDoc system, hash tables, tasks
- Create terminal window: `DocTermNew; WinVert(2,10);`
- **Spawn Window Manager:** `Spawn(&WinMgrTask,NULL,"Window Mgr")`
- Set window inhibit flags
- `RegInit`: Initialize registry
- Set `RLf_REGISTRY` flag
- Check registration status (via `ins_reg.registered`)
- `WallPaperInit`: Initialize wallpaper system
- Create `/Tmp` directory if drive is writable
- Set `RLf_HOME` flag
- **Execute `~/MakeHome`** (user customization)

---

#### **6. Stage 6: User Environment (MakeHome.HC)**

**Key Operations:**
- Execute `~/HomeLocalize`: Localization settings
- Include optional components:
  - Boot utilities (`/Adam/Opt/Boot/MakeBoot`)
  - System utilities (`/Adam/Opt/Utils/MakeUtils`)
  - Autocomplete system
- Set editor warnings (overwrite, duplicate open)
- VM-specific drivers:
  - AC97 sound for VMs
  - VMWare mouse driver
  - VirtualBox guest additions (commented out)
  - OPL2 FM synthesis
- Load symbol maps: `MapFileLoad("::/Kernel/Kernel")`
- Include user customizations (`~/HomeWrappers`, `~/HomeSys`)
- Adam task enters **server mode** (message loop)

---

#### **7. Stage 7: First Terminal (Once.HC)**

Executed by first spawned terminal window.

**Boot Source Handling:**

**BOOT_SRC_ROM:** Asks to continue booting from hard drive

**BOOT_SRC_DVD:** Live CD boot
- Asks: "Install onto hard drive?"
- If yes: runs `/Misc/OSInstall`
- Offers system tour

**BOOT_SRC_RAM/BOOT_SRC_HARDDRV:** Normal boot
- If text mode (unexpected): Explains video mode issue
- Otherwise: Shows "Tip of the Day"
- Displays customization guide
- Offers tour if available
- Loads user menu: `DocBottom; InFile("~/PersonalMenu");`

---

### **Key Data Structures**

**CKernel:** Located at `SYS_KERNEL`, contains:
- `boot_src`: Boot source (ROM/DVD/RAM/HARDDRV)
- `boot_blk`: Boot block number
- `sys_run_level`: Bitfield of system initialization stages
- `sys_frame_buffer`: Frame buffer physical address
- `sys_vbe_mode_pitch`: Frame buffer pitch
- `sys_initramfs_ptr/size`: InitRAMFS for Live USB

**Boot Sources:**
- `BOOT_SRC_NULL`: Uninitialized
- `BOOT_SRC_ROM`: BIOS ROM boot
- `BOOT_SRC_DVD`: CD/DVD boot
- `BOOT_SRC_RAM`: RAM disk soft reboot
- `BOOT_SRC_HARDDRV`: Hard drive boot

**Run Level Flags (RLf_*):**
- `RLf_16BIT`, `RLf_32BIT`, `RLf_64BIT`: CPU mode
- `RLf_VGA`: Graphics mode enabled
- `RLf_PATCHED`: Address relocation complete
- `RLf_RAW`: Can use RawPrint
- `RLf_INTERRUPTS`: Interrupts enabled
- `RLf_16MEG_ADAM_HEAP_CTRL`: Adam heap ready
- `RLf_REGISTRY`: Registry initialized
- `RLf_HOME`: User environment loaded
- `RLf_DOC`: Document system ready

---

### **Soft Reboot (BootRAM.HC)**

TinkerOS supports **soft reboots** without hardware reset:

1. Read `Kernel.BIN.C` file into high memory
2. Copy preserved state (boot source, frame buffer, initramfs)
3. Halt all CPUs except CPU0 (`MPHalt`)
4. Copy reboot code to high memory (`SYS_RAM_REBOOT`)
5. Copy kernel image to `mem_boot_base`
6. Execute high memory reboot code
7. Disable paging, clear long mode
8. Jump to `CORE0_32BIT_INIT` in 32-bit mode
9. Boot continues from Stage 2

---

### **Boot Device Detection**

**AHCI Mode (Modern):**
- Scans PCI bus for AHCI controller (class 01:06:01)
- Reads HBA (Host Bus Adapter) memory-mapped registers
- Detects SATA and SATAPI devices on ports
- Uses DMA for disk I/O

**IDE/Legacy Mode (Fallback):**
- Probes standard IDE ports (0x1F0, 0x170, etc.)
- Sends ATA IDENTIFY commands
- Uses PIO (Programmed I/O) for disk access
- Supports master/slave devices

**Partition Detection:**
- Reads MBR (Master Boot Record) at sector 0
- Parses partition table (4 primary entries)
- Supports extended partitions (types 5, 15)
- Recognizes: FAT32, RedSea (native), NTFS, ISO9660
- Auto-mounts drives starting from 'C:'

---

### **Memory Layout**

```
0x00000000 - 0x000FFFFF  : Low 1MB (BIOS, VGA, boot code)
0x00100000 - MEM_BOOT_BASE: Kernel code/data
MEM_BOOT_BASE+          : System heap start
0xB8000                 : VGA text buffer (text mode)
sys_frame_buffer        : Linear frame buffer (graphics)
mem_mapped_space        : MMIO device space
MEM_PML4                : Page table structures
```

---

### **Critical Configuration**

Defined in KCfg.HH during compilation:
- `CFG_TEXT_MODE`: Boot to text mode
- `CFG_MEM_INIT`: Initialize memory
- `CFG_DONT_PROBE`: Skip hardware probing
- `mount_ide_auto_hd_let`: First HD letter ('C')
- `mount_ide_auto_cd_let`: First CD letter ('T')
- `boot_drv_let`: Boot drive letter
- Graphics resolution (FB_WIDTH x FB_HEIGHT)

---

This boot process showcases a complete journey from 16-bit BIOS to a fully functional 64-bit multitasking operating system with graphics, filesystem support, and a JIT compiler—all in a remarkably compact codebase!