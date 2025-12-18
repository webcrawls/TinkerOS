# TinkerOS Bare-Metal Black Screen Debugging Guide

## Problem Description
Black screen on boot when running TinkerOS on real hardware (bare metal).

## Phase 1: Hardware/BIOS Verification (No Recompile)

### 1.1 BIOS Settings Checklist
Verify these settings in your BIOS/UEFI:

- [ ] **Boot Mode**: Legacy/CSM enabled (NOT UEFI-only)
- [ ] **Secure Boot**: DISABLED
- [ ] **Fast Boot**: DISABLED
- [ ] **HPET**: DISABLED (if option exists)
- [ ] **USB Legacy Support**: ENABLED
- [ ] **Port 60/64 Emulation**: ENABLED
- [ ] **SATA Mode**: Try BOTH options:
  - First try: IDE/Legacy/Compatibility mode
  - Then try: AHCI mode
- [ ] **Video**: Ensure no "Fast Boot Logo" or splash screen interfering

### 1.2 Boot Media Verification

**If using USB:**
- ✅ Must use `.img` file (e.g., `TinkerOS_USB_version.img`)
- ❌ `.iso` files will NOT boot from USB
- Use Rufus (Windows), Etcher, or `dd` (Linux) to write image
- Verify USB stick is detected as bootable device in BIOS

**If using CD/DVD:**
- Use `.iso` file for optical media
- Burn at slowest speed for reliability
- Verify disc after burning

### 1.3 Try Different Boot Options

If you see a boot menu, try:
1. **Text Mode** option first (more hardware compatible)
2. Different resolution options if available
3. Safe mode or minimal boot if offered

### 1.4 Hardware Compatibility Check

**Minimum Requirements:**
- 64-bit x86_64 CPU (not ARM, not Mac)
- 2GB RAM minimum
- PS/2 keyboard OR USB keyboard with legacy emulation
- PS/2 mouse OR USB mouse with legacy emulation
- Supported storage: IDE, SATA (Legacy or AHCI), NOT M.2 NVMe

**Common Problem Hardware:**
- Ultra-thin laptops (often lack PS/2 emulation)
- Chromebooks (incompatible)
- Systems newer than ~2018 (may lack CSM support)
- SSDs newer than 500GB (may not support old ATA commands)

---

## Phase 2: Add Debug Beep Codes (Requires Recompile)

Since video might be failing, add audible PC speaker beeps to identify where boot fails.

### 2.1 Add Beeps to KStart64.HC

Edit `/Kernel/KStart64.HC` to add a beep after 64-bit mode is entered:

```c
asm {
  USE64
  BTS	U32 [SYS_RUN_LEVEL],RLf_64BIT

  // DEBUG: Beep to indicate we reached 64-bit mode
  IN	AL,0x61
  OR	AL,3
  OUT	0x61,AL
  MOV	ECX,0x500000
@@debug_delay1:
  DEC	ECX
  JNZ	@@debug_delay1
  IN	AL,0x61
  AND	AL,0xFC
  OUT	0x61,AL

  //Set required bits for SSE instruction execution
  MOV_RAX_CR4
  BTS     RAX, CR4f_OSFXSR
  MOV_CR4_RAX
  // ... rest of code
```

**Meaning:** If you hear ONE beep, the system reached 64-bit mode successfully.

### 2.2 Add Beeps to KMain.HC

Edit `/Kernel/KMain.HC` to add beeps at various stages:

```c
U0 KMain()
{
  CBlkDev *bd, *initramfs;
  
  // DEBUG: 2 beeps = Entered KMain
  Snd(50); Busy(100); Snd;
  Snd(50); Busy(100); Snd;
  
  OutU8(0x61,InU8(0x61)&~3);
  adam_task=Fs;
  BlkPoolsInit;
  SysGlblsInit;
  Mem32DevInit;
  UncachedAliasAlloc;
  LoadKernel;
  
  // DEBUG: 3 beeps = About to init graphics
  Snd(60); Busy(100); Snd;
  Snd(60); Busy(100); Snd;
  Snd(60); Busy(100); Snd;
  
  SysGrInit;
  
  // DEBUG: 4 beeps = Graphics initialized
  Snd(70); Busy(100); Snd;
  Snd(70); Busy(100); Snd;
  Snd(70); Busy(100); Snd;
  Snd(70); Busy(100); Snd;
  
  StrCpy(Fs->task_name,"Adam Task CPU00");
  // ... rest of code
```

**Beep Code Meanings:**
- **1 beep**: Reached 64-bit mode
- **2 beeps**: Entered KMain() successfully
- **3 beeps**: About to initialize graphics
- **4 beeps**: Graphics initialized (if you don't see screen, graphics mode failed)
- **No beeps**: Failed before 64-bit mode (16-bit or 32-bit stage)

---

## Phase 3: Add Text Mode Debug Output

Force text mode and add verbose output.

### 3.1 Force Text Mode Boot

Edit `/Kernel/KCfg.HH` before compiling:

```c
#define CFG_TEXT_MODE 1  // Force text mode (more compatible)
```

Recompile the kernel. Text mode uses VGA 80x25 which works on almost all hardware.

### 3.2 Add Early VGA Text Output

Edit `/Kernel/KStart32.HC` to write directly to VGA buffer:

```c
CORE0_32BIT_INIT::
  PUSH	U32 RFLAGG_START
  POPFD
  MOV	EAX,SYS_START_CR0
  MOV_CR0_EAX

  // DEBUG: Write "32" to VGA text buffer (top-left corner)
  MOV	EAX,0xB8000
  MOV	WORD [EAX],0x0F33    // '3' white on black
  MOV	WORD [EAX+2],0x0F32  // '2' white on black

  MOV	AX,CGDT.boot_ds
  MOV	DS,AX
  // ... rest of code
```

If you see "32" on screen = 32-bit mode works.

### 3.3 Add VGA Output in 64-bit Mode

Edit `/Kernel/KStart64.HC`:

```c
@@05:	POP	RBP
  RET1	8

// DEBUG: Write "64" to VGA buffer
  MOV	RAX,0xB8000
  MOV	WORD [RAX+4],0x0F36   // '6' white on black
  MOV	WORD [RAX+6],0x0F34   // '4' white on black

//Init CPU0 Struct
  PUSH	SYS_FIXED_AREA+CSysFixedArea.adam
  // ... rest of code
```

If you see "3264" = Both 32-bit and 64-bit modes reached.

### 3.4 Add Debug Output in KMain

After `SysGrInit`, add verbose diagnostics:

```c
SysGrInit;

// DEBUG: Display what we initialized
"DEBUG: Graphics Init Complete\n";
"  Frame Buffer: 0x%X\n", text.fb_alias;
"  VGA Mode: %s\n", Bt(&sys_run_level,RLf_VGA) ? "Graphics" : "Text";
"  Cols: %d  Rows: %d\n", text.cols, text.rows;
"  GR_WIDTH: %d  GR_HEIGHT: %d\n", GR_WIDTH, GR_HEIGHT;
"  FB_WIDTH: %d  FB_HEIGHT: %d\n", FB_WIDTH, FB_HEIGHT;
"\nPress any key to continue...\n";
GetChar;
```

---

## Phase 4: Video Mode Debugging

If graphics mode is failing, diagnose the VBE initialization.

### 4.1 Add Debug to KStart16.HC

Edit `/Kernel/KStart16.HC` in the VBE detection section:

```asm
@@05:
  MOV	AX, VBE_INFO
  MOV	SI, CVBEInfo.video_modes[AX]
  MOV	GS, CVBEInfo.video_modes+2[AX]
  MOV	DI, TEMP_VBE_MODE
  MOV   DX, VBE_VID_MODES
  
  // DEBUG: Beep to indicate VBE found
  IN	AL,0x61
  OR	AL,3
  OUT	0x61,AL
  MOV	CX,0x8000
@@vbe_beep:
  DEC	CX
  JNZ	@@vbe_beep
  IN	AL,0x61
  AND	AL,0xFC
  OUT	0x61,AL
  
@@06:
  MOV 	AX, GS:[SI]
  // ... rest of VBE code
```

**Short beep during boot** = VBE BIOS found, scanning modes.

### 4.2 Log Available Video Modes

Add this to KMain after SysGrInit:

```c
SysGrInit;

// DEBUG: Show what video modes were detected
"=== Video Mode Debug ===\n";
"Requested: %d x %d\n", FB_WIDTH, FB_HEIGHT;
"Frame Buffer: 0x%X\n", sys_frame_buffer;
"VBE Pitch: %d\n", sys_vbe_mode_pitch;

if (!sys_frame_buffer) {
  "\n*** WARNING: No frame buffer found! ***\n";
  "Video mode %d x %d not supported by hardware.\n", FB_WIDTH, FB_HEIGHT;
  "Falling back to text mode.\n\n";
}

// List detected modes
U16 *modes = VBE_VID_MODES;
"Detected VBE modes:\n";
I64 i;
for (i = 0; i < 64 && modes[i*2]; i++) {
  "  %d x %d\n", modes[i*2+1], modes[i*2];
}
"========================\n\n";
```

---

## Phase 5: Serial Port Debugging (Advanced)

If nothing else works, output debug info to serial port.

### 5.1 Setup Serial Output

Add to beginning of KMain:

```c
U0 SerialDebugInit() {
  OutU8(0x3F8 + 1, 0x00);    // Disable interrupts
  OutU8(0x3F8 + 3, 0x80);    // Enable DLAB
  OutU8(0x3F8 + 0, 0x03);    // Divisor low byte (38400 baud)
  OutU8(0x3F8 + 1, 0x00);    // Divisor high byte
  OutU8(0x3F8 + 3, 0x03);    // 8N1
  OutU8(0x3F8 + 2, 0xC7);    // Enable FIFO
  OutU8(0x3F8 + 4, 0x0B);    // RTS/DTR
}

U0 SerialDebugChar(U8 ch) {
  while (!(InU8(0x3F8 + 5) & 0x20));  // Wait for empty
  OutU8(0x3F8, ch);
}

U0 SerialDebugStr(U8 *str) {
  while (*str) {
    SerialDebugChar(*str++);
  }
}

U0 KMain() {
  SerialDebugInit;
  SerialDebugStr("KMain started\r\n");
  
  // ... rest of code, add SerialDebugStr() calls at each stage
```

Connect via null modem cable and monitor with screen/minicom/PuTTY at 38400 baud.

---

## Phase 6: Common Failure Points & Solutions

### 6.1 No Beeps at All
**Problem:** System not reaching code execution
**Solutions:**
- Check if boot media is actually booting (should see boot menu)
- Verify boot order in BIOS
- Try different USB port (USB 2.0 ports work better than 3.0)
- Boot media may be corrupted - rewrite image

### 6.2 1 Beep Only (Stuck After 64-bit Mode)
**Problem:** Failing in early KMain initialization
**Solutions:**
- RAM issue - test with memtest86+
- Incompatible CPU features
- Check if at least 2GB RAM available

### 6.3 3 Beeps (Stuck at Graphics Init)
**Problem:** SysGrInit() hanging
**Solutions:**
- Use text mode (CFG_TEXT_MODE = 1)
- Video card incompatible with requested resolution
- Try different resolution in KCfg.HH

### 6.4 4 Beeps But Still Black Screen
**Problem:** Graphics initialized but not displaying
**Solutions:**
- Frame buffer address incorrect for hardware
- Monitor doesn't sync to the resolution
- Try text mode as fallback
- Check monitor cable/connection

### 6.5 Boots to Garbled Screen
**Problem:** Graphics mode partially working
**Solutions:**
- Resolution mismatch - try 640x480
- Pitch/stride calculation wrong for GPU
- Use text mode

---

## Quick Diagnostic Decision Tree

```
Start: Black screen on boot
│
├─ Hear ANY beeps?
│  ├─ NO → Check boot media, BIOS settings, rewrite image
│  │      Try different USB port, verify boot order
│  │
│  └─ YES → How many beeps?
│     ├─ 1 beep → Failed in early KMain
│     │          Check RAM (need 2GB+), try text mode
│     │
│     ├─ 2 beeps → Failed before graphics
│     │           Try text mode, check hardware compatibility
│     │
│     ├─ 3 beeps → Failed during graphics init
│     │           FORCE text mode (CFG_TEXT_MODE=1)
│     │           Or try different resolution
│     │
│     └─ 4+ beeps → Graphics init completed but no display
│                   Monitor sync issue, try different resolution
│                   Check frame buffer address detection
│
└─ Alternative: Try Text Mode First
   Edit KCfg.HH: #define CFG_TEXT_MODE 1
   Recompile kernel, test again
```

---

## Quick Fix: Compile Text-Mode Debug Kernel

For fastest troubleshooting, compile with these settings in `/Kernel/KCfg.HH`:

```c
// Force text mode
kernel_cfg->opts[CFG_TEXT_MODE] = TRUE;

// Enable verbose debugging
kernel_cfg->opts[CFG_HEAP_INIT] = TRUE;
kernel_cfg->opts[CFG_MEM_INIT] = TRUE;

// Don't probe (faster boot, better for debugging)
// kernel_cfg->opts[CFG_DONT_PROBE] = TRUE;  // Only if you know your hardware
```

Compile and test. Text mode has much better hardware compatibility.

---

## Additional Resources

- Hardware compatibility list: `Doc/Baremetal/Machines/`
- Run `SysSurvey;` on working systems to document hardware
- Check video modes: After boot, run `Dir("C:/Home/VBEModes.DD");` if available
- TinkerOS docs: `Baremetal.md`

---

## Example: Complete Debug Build

Here's a minimal modification set for maximum debug output:

**1. Add to KStart64.HC (after BTS RLf_64BIT):**
```asm
  // Beep: reached 64-bit
  IN   AL,0x61
  OR   AL,3
  OUT  0x61,AL
  MOV  ECX,0x500000
@@d1: DEC ECX
  JNZ  @@d1
  IN   AL,0x61
  AND  AL,0xFC
  OUT  0x61,AL
```

**2. Add to KMain.HC (beginning):**
```c
U0 KMain() {
  CBlkDev *bd, *initramfs;
  Snd(50); Busy(100); Snd; // 1 beep
  Snd(50); Busy(100); Snd; // 2 beeps = in KMain
  
  OutU8(0x61,InU8(0x61)&~3);
  adam_task=Fs;
  BlkPoolsInit;
  SysGlblsInit;
  Mem32DevInit;
  UncachedAliasAlloc;
  LoadKernel;
  
  Snd(60); Busy(100); Snd; // 3 beeps = before graphics
  SysGrInit;
  Snd(70); Busy(100); Snd; // 4 beeps = after graphics
  
  // ... rest of function
```

**3. Recompile kernel:**
```bash
cd /Kernel
#include "Kernel"
```

Test and listen for beep patterns!
