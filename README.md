# PCILeech VT-d Bypass Guide

## Safety and prerequisites

> [!WARNING]
> Changing BIOS images, flashing SPI chips, or editing UEFI variables can brick a board.
>
> Dump the original firmware first and keep that dump somewhere safe. If possible, have a CH341A programmer, a SOIC8 clip, and a 1.8 V adapter on hand before you start.

> [!IMPORTANT]
> Treat every board and BIOS version as its own target. Do not paste in offsets, firmware images, or scripts from somebody else's machine and expect them to line up.

## Contents

- [Safety and prerequisites](#safety-and-prerequisites)
- [Core concept](#core-concept)
- [Understand the PCIe path first](#understand-the-pcie-path-first)
- [Board selection](#board-selection)
- [Required tools and adapters](#required-tools-and-adapters)
- [Software downloads](#software-downloads)
- [Reference hardware photos](#reference-hardware-photos)
- [Backup routine before BIOS work](#backup-routine-before-bios-work)
- [AMIBCP workflow](#amibcp-check-failsafe-and-optimal-not-just-show)
- [VtdDxe fallback methods](#vtddxe-fallback-methods)
- [Platform-specific routes](#platform-specific-routes)
- [Flash and test order](#flash-and-test-order)
- [Troubleshooting table](#troubleshooting-table)
- [Final checklist](#final-checklist)
- [License](#license)

---

## Core concept

A working VT-d bypass setup is not just a `VT-d` toggle, and it is not a magic motherboard list.

### Target state

The target state is:

- the OS still sees VT-d/IOMMU as present;
- but the target DMA path is not being blocked by the permission/remapping layer in the normal way.

### Order of work

Use the following order:

1. **Get the PCIe route right first.**
2. **Tune BIOS/UEFI second.**

### Preferred route

The preferred route is usually:

```text
PCIe device -> CPU Root Port / IIO -> memory controller -> DDR
```

PCH/chipset-side paths can still enumerate, but they are more likely to produce no effect, unstable links, dead connections, or inconsistent state.

The important part is not simply "turn off VT permission." The real work is PCIe routing, ATS, DMA remapping, interrupt remapping, and the permission layer between the device and memory.

For many B760/H610 setups, the useful low-cost route is often:

```text
CPU-connected M.2 slot -> M.2-to-PCIe adapter -> target card
```

On AMD, do not expect Intel-style `VT-d` naming. You are usually dealing with `IOMMU`, `ACS`, `ATS`, PCIe ARI, SR-IOV, and similar options.

## Understand the PCIe path first

PCIe slots may look the same from the outside, but the upstream path is what matters.

Usually:

- the first full-length x16 slot is connected directly to the CPU;
- many secondary long slots, short slots, onboard controllers, and some M.2 slots come from the PCH/chipset side;
- devices behind the PCH go through extra bridge and DMI layers, which makes ATS, DMA remapping, interrupt remapping, and ACS more likely to get involved.

Do not start with:

> Can this motherboard's BIOS be modified?

Start with:

> Which Root Port is the target card actually attached to?

If that is not clear, buying USB drives, programmers, adapters, or editing AMIBCP settings will not produce reliable results.

## Board selection

| Platform | Usable direction | What matters | Trap to avoid |
| --- | --- | --- | --- |
| H610 / H-series | Basic H610/H610M boards; prefer CPU-connected M.2 or the first long PCIe slot. | Low cost, fewer variables, good for one-card, one-route testing. | Small boards have limited resources and many PCH slots. Budget does not mean every slot works. |
| B660 / B760 | Boards like ASUS PRIME B760M, MSI PRO B760M, or ASRock B760M; check whether `M.2_1` is CPU PCIe x4. | A common route is CPU M.2 to memory, then M.2-to-PCIe. | The second long slot is often from the PCH. The wrong adapter path may do nothing. |
| Z690 / Z790 | Usable, especially when the CPU M.2 and first x16 slot layout is clear. | More BIOS options, but write protection is also common. | Not ideal for beginners to buy an expensive Z board just for this. |
| X299 / X399 / X99 | Examples include ASUS PRIME X299-A II, ASRock X299 Taichi, and other X99/X299-style lab boards. | More CPU lanes and more M.2/PCIe combinations; useful for route testing. | GPU, M.2, and PCIe lane conflicts can cause strange enumeration or unstable device state. Do not assume a route is valid just because the device shows up. |
| B450 / B550 | Boards like MSI B450 TOMAHAWK or ASUS TUF B450M; check CPU x16 and CPU M.2 first. | AMD naming is usually IOMMU/ACS/ATS. Middle-layer options often need tuning and locking. | Do not force Intel VT-d menu names onto AMD boards. |
| B85 / older platforms | Prefer full ATX boards; find the complete CPU x16 main slot. | Fewer platform variables, but many old boards have cut-down slot layouts. | Small or heavily cut-down boards and rear/PCH slots can be unstable. |

## Required tools and adapters

The programmer is for backup and recovery. It does not improve the PCIe route by itself. In many B760/H610/X299 setups, the M.2-to-PCIe adapter or PCIe riser determines whether the target route is usable.

| Prepare | Search terms | Check before buying |
| --- | --- | --- |
| CH341A + SOIC8 clip | `CH341A programmer`, `SOIC8 clip`, `1.8V adapter` | SPI chip voltage, clip direction, and whether two reads match exactly. |
| UEFI tool USB drive | `SetupVar USB`, `BIOS variable tool`, `UEFI shell` | Must match the exact board model and BIOS version. There is no universal USB image for this. |
| M.2-to-PCIe adapter | `M.2 NVMe Key M to PCIe x4`, `M.2 PCIe extension` | Must use a CPU-connected NVMe M.2 slot. SATA M.2 is useless for this route. |
| PCIe riser | `PCIe x1 riser`, `PCIe 1x to 1x`, `PCIe 1x to 6x extension` | Check power, direction, cable length, and whether it matches the target route. |
| USB drive | `FAT32 8GB USB`, `FAT32 16GB USB` | Use it for BIOS files, UEFI Shell, and variable tools. Do not mix many boot tools on one drive. |

## Software downloads

Use original vendor or project pages whenever possible. Avoid repacked BIOS tools from random mirror sites, especially when the tool will touch firmware images or SPI flash.

| Purpose | Software | Download/source | Notes |
| --- | --- | --- | --- |
| CH341/CH340 USB driver | WCH CH341SER | [WCH CH341SER Windows driver](https://www.wch-ic.com/downloads/CH341SER_EXE.html) | Driver only. It does not replace proper SPI read/verify workflow. |
| SPI flash read/write | flashrom | [flashrom project](https://flashrom.org/) | Supports CH341A through `ch341a_spi` on supported platforms. Useful for read, erase, write, and verify workflows. |
| UEFI image inspection/editing | UEFITool | [LongSoft UEFITool releases](https://github.com/LongSoft/UEFITool/releases) | Use the correct branch/build for the task. NE builds are useful for inspection; older regular builds are often used for replace/write workflows. |
| IFR extraction | IFRExtractor-RS / Universal IFR Extractor | [IFRExtractor-RS releases](https://github.com/LongSoft/IFRExtractor-RS/releases) / [Universal IFR Extractor](https://github.com/LongSoft/Universal-IFR-Extractor) | Converts UEFI Internal Form Representation into readable text for Setup variable review. |
| Hex editing | HxD | [Official HxD download page](https://mh-nexus.de/en/downloads.php?product=HxD20) | Use for direct binary inspection when required. |
| UEFI shell environment | TianoCore EDK II Shell | [TianoCore ShellPkg](https://github.com/tianocore/tianocore.github.io/wiki/ShellPkg) / [archived X64 Shell.efi](https://github.com/tianocore/edk2-archive/blob/master/ShellBinPkg/UefiShell/X64/Shell.efi) | Put the shell binary in the expected EFI path, commonly `EFI/BOOT/BOOTX64.EFI`, when building a UEFI USB environment. |
| AMI BIOS configuration | AMIBCP / AMI Aptio utilities | [AMI Aptio V product page](https://www.ami.com/products/aptio-v/) | AMIBCP is not normally distributed as a public freeware download. Use a licensed/OEM source. Do not rely on unknown reposted binaries. |

## Reference hardware photos

These are reference photos only. Exact board revisions, clip wiring, adapters, and risers vary by vendor.

<table>
  <tr>
    <td align="center" width="50%">
      <strong>CH341A programmer with clip</strong><br>
      <img src="https://commons.wikimedia.org/wiki/Special:FilePath/CH341A%20USB%20with%20crocodile%20clip.jpg?width=640" alt="CH341A programmer with clip" width="360"><br>
      <a href="https://commons.wikimedia.org/wiki/File:CH341A_USB_with_crocodile_clip.jpg">Source: Wikimedia Commons</a>
    </td>
    <td align="center" width="50%">
      <strong>SOIC-8 clip</strong><br>
      <img src="https://commons.wikimedia.org/wiki/Special:FilePath/Pomona%205250%20SOIC-8%20clip.jpg?width=640" alt="SOIC-8 clip" width="360"><br>
      <a href="https://commons.wikimedia.org/wiki/File:Pomona_5250_SOIC-8_clip.jpg">Source: Wikimedia Commons</a>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%">
      <strong>PCIe/NVMe adapter example</strong><br>
      <img src="https://commons.wikimedia.org/wiki/Special:FilePath/Adata%20XPG%20SX8200%20Pro%20pcie%20adapter%20view.jpg?width=640" alt="PCIe NVMe adapter card" width="360"><br>
      <a href="https://commons.wikimedia.org/wiki/File:Adata_XPG_SX8200_Pro_pcie_adapter_view.jpg">Source: Wikimedia Commons</a>
    </td>
    <td align="center" width="50%">
      <strong>PCIe riser card example</strong><br>
      <img src="https://commons.wikimedia.org/wiki/Special:FilePath/Supermicro-SYS-5019C-MR-PCIe-x16-Riser.jpg?width=640" alt="PCIe riser card" width="360"><br>
      <a href="https://commons.wikimedia.org/wiki/File:Supermicro-SYS-5019C-MR-PCIe-x16-Riser.jpg">Source: Wikimedia Commons</a>
    </td>
  </tr>
</table>

## Backup routine before BIOS work

Before changing anything:

1. Confirm the full motherboard model, PCB revision, and current BIOS version. A tiny suffix difference can make a BIOS image incompatible.
2. Download the same or target BIOS version from the official board page. Record the filename, date, and version.
3. Prepare a FAT32 USB drive. Keep it simple and label it clearly.
4. If using a programmer: shut down, unplug power, press the case power button for about 10 seconds to discharge, then attach the SOIC8 clip. Align the red wire with pin 1, usually the dot or notch on the chip.
5. Read the SPI firmware twice. Save the dumps as something like `backup_1.bin` and `backup_2.bin`.
6. Compare SHA256 hashes. If the two dumps do not match, do not write anything back.
7. Copy the original backup to another disk or cloud storage. If the board bricks, that original dump may be the only clean recovery path.

> [!IMPORTANT]
> An official BIOS download is not the same thing as the full SPI contents on your board. Serial numbers, MAC addresses, ME/AGESA regions, and NVRAM data may live in the chip. If you can dump the original chip, dump the original chip.

## AMIBCP: check Failsafe and Optimal, not just Show

In AMIBCP, the left side is the menu tree. The right side contains variables and defaults.

`Show` only makes a menu visible. The real boot-time behavior usually follows `Failsafe`, `Optimal`, or the matching Setup variable.

Do not only change `Show` and assume the firmware behavior changed.

### Basic workflow

1. Make a working copy of the BIOS dump, for example `mod_work.bin`. Never edit the original backup directly.
2. Run AMIBCP as administrator.
3. Open `mod_work.bin` or the unpacked BIOS image you are working on.
4. Look under paths like:
   - `Advanced`
   - `Socket Configuration`
   - `IIO Configuration`
   - `Intel(R) VT for Directed I/O`
   - `System Agent`
   - `PCIe Configuration`
5. If you cannot find the option in the UI, search the IFR text for terms like:
   - `ATS`
   - `Directed I/O`
   - `DMA Remapping`
   - `Interrupt Remapping`
   - `Posted Interrupt`
   - `PassThrough DMA`
   - `Coherency`
6. Record the original value before changing each item.
7. Save as `mod_vtd_route.bin`.
8. Close AMIBCP, reopen the saved file, and confirm the values are still there.
9. Open the result in UEFITool once to confirm the structure still parses.

| AMIBCP area | What it means | What to change |
| --- | --- | --- |
| Left menu tree | BIOS setup page layout. It only tells you where the option lives. | Use it to find `Advanced`, `System Agent`, `IIO`, `PCIe Configuration`, or `Intel(R) VT for Directed I/O`. |
| `Access` / `Use` / `Show` | Menu visibility. This only exposes or hides a BIOS menu item. | Set it to `User` only if you want the option visible in BIOS. Do not count this as the real functional change. |
| `Failsafe` | The fallback/default profile used by the firmware. | For options you are intentionally changing, set this to the same target value as `Optimal` unless you have a specific recovery reason not to. |
| `Optimal` | The normal boot/default value used by most boards. | This is the main column to edit. If only `Show` is changed and `Optimal` is not changed, the boot behavior usually stays the same. |
| Setup variable | The NVRAM variable behind the menu item. | If AMIBCP cannot save the value, use IFR to get `VarStore`, `Offset`, width, and the exact value encoding. |

### First-pass AMI Aptio setting map

| Option family | Search strings | First-pass value | Notes |
| --- | --- | --- | --- |
| VT-d main switch | `Intel(R) VT for Directed I/O`, `VT-d`, `Directed I/O` | `Enabled` | Keep this enabled for the normal route so the OS still sees VT-d/IOMMU as present. Do not solve this by simply turning the main VT-d switch off. |
| Pass-through DMA | `PassThrough DMA`, `Pass Through DMA`, `DMA PassThrough` | `Enabled` | If the board exposes this option, keep it enabled on the first pass. |
| ATS | `ATS`, `PCIe ATS`, `ATS Support`, `Address Translation Services` | `Disabled` | This is one of the main permission-layer items. Set both `Failsafe` and `Optimal` to `Disabled` when the option exists. |
| Route-specific DMA remapping / middle-layer blocking | `DMA Remapping`, `DMA Remap`, `DMA Control`, `DMA Protection` | Usually test `Disabled` for the target route | Do not confuse a route-specific remapping control with the global VT-d switch. If there is only one global VT-d/DMA item, leave it for a separate test instead of disabling it blindly. |
| Interrupt remapping | `Interrupt Remapping`, `Interrupt Remap` | Keep original value on the first pass | If the route still fails, test this separately after recording the original value. Cold boot after changing it. |
| Posted interrupt / coherency | `Posted Interrupt`, `Coherency`, `Coherent`, `No Snoop` | Keep original value on the first pass | These can change stability. Do not change them until the slot/adapter route and ATS state are already confirmed. |
| PCIe grouping helpers | `ACS`, `ACS Control`, `ARI Forwarding`, `Relaxed Ordering` | `Enabled` when available | These affect grouping and bridge behavior. They do not replace ATS or DMA-remapping work. |

1. Open the BIOS image in AMIBCP.
2. Go through `Advanced`, `System Agent`, `IIO Configuration`, `PCIe Configuration`, and `Intel(R) VT for Directed I/O`.
3. For `Intel(R) VT for Directed I/O`, set `Failsafe = Enabled` and `Optimal = Enabled`.
4. For `PassThrough DMA`, set `Failsafe = Enabled` and `Optimal = Enabled` if the item exists.
5. For every `ATS` item under the target PCIe/IIO path, set `Failsafe = Disabled` and `Optimal = Disabled`.
6. For route-specific `DMA Remapping` or middle-layer blocking items, test `Disabled` in both `Failsafe` and `Optimal`.
7. Leave `Interrupt Remapping`, `Posted Interrupt`, and `Coherency` at their original values for the first boot.
8. If needed for visibility, set `Access` / `Use` to `User`, but still change `Failsafe` and `Optimal`. Visibility alone is not enough.
9. Save as a new file, reopen it in AMIBCP, and confirm every changed value still shows the intended state.
10. Open the saved image in UEFITool and confirm the firmware structure still parses.

### If AMIBCP cannot save the value

1. Extract the matching Setup module with UEFITool.
2. Parse it with IFR Extractor.
3. For the exact option, record `VarStore`, `Offset`, width, and all possible values.
4. Do not assume `Enabled = 1` and `Disabled = 0` until the IFR text confirms it.
5. Use a matching UEFI variable tool only for that board and BIOS version.
6. Read the original value first, write one variable, read it back, then cold boot and retest.
7. Do not run a batch of copied offsets from another motherboard.

> [!NOTE]
> The VT-d main switch and ATS are not the same thing. The target state is "VT-d still appears enabled, but the target route is not effectively blocked by that permission layer." Simply setting VT-d to `Disabled` often leaves the OS and tool state looking wrong.

## VtdDxe fallback methods

This is the firmware-module fallback path. Do not start here. Use it only after:

1. the target card has already been moved to a CPU-connected PCIe or M.2 route;
2. AMIBCP/IFR variable edits have been tested;
3. the board still shows the wrong functional behavior even though the visible VT-d state looks correct.

`VtdDxe` is the DXE-stage driver family that many AMI Aptio firmwares use for Intel VT-d initialization, DMAR-related setup, and DMA-remapping handoff. Vendors may rename or wrap it, so the target can appear as `VtdDxe`, `IntelVTdDxe`, `DxeVtd`, `SaVtdDxe`, or only a GUID with `VT-d`/`DMAR` strings inside the PE image.

The idea is to separate two states:

| State | What the user sees | What this section tries to change |
| --- | --- | --- |
| Visible setup state | BIOS menu or NVRAM says VT-d is enabled. | Keep this state enabled when the route depends on the OS seeing VT-d/IOMMU. |
| Functional DXE state | The firmware driver actually initializes remapping, DMAR, or lock-down behavior. | Stop or neutralize this part when normal variable edits are not enough. |

### Choose the least invasive path

| Situation | Prefer | Why |
| --- | --- | --- |
| AMIBCP or IFR variable edits already work. | Do not patch `VtdDxe`. | A variable-level change is easier to audit and recover. |
| Low-cost white-label or clone-brand board with visibly cut-down firmware. | Try module removal first, with a verified programmer backup. | These boards may already ship with incomplete dependency paths, so removing the remaining VT-d DXE module can be enough. |
| Mainstream ASUS/MSI/Gigabyte/ASRock board, workstation board, server board, or newer platform. | Prefer an early-return body patch. | The firmware may expect the FFS file, dependency section, or driver dispatch result to exist. |
| Early return boots but the route still behaves like stock. | Trace and patch a narrower function. | Another module may publish DMAR or perform the decisive lock/remap work. |
| Any patch causes black screen or setup hang. | Restore the original SPI backup. | Do not continue by stacking more changes on a bad image. |

These methods are board-BIOS-version specific. A patched `VtdDxe` body from one BIOS must not be reused on another BIOS, even if the motherboard model looks similar.

### Shared preparation

Before deleting or patching anything:

1. Work only on a copy of the full SPI dump, for example `mod_vtddxe_work.bin`.
2. Keep original and derived files separate:

   | File | Purpose |
   | --- | --- |
   | `backup_original.bin` | Verified full SPI backup. Never edit this file. |
   | `VtdDxe_original.ffs` | Original whole firmware file extracted before edits. |
   | `VtdDxe_original_body.efi` | Original PE32 image body. |
   | `VtdDxe_ret_success_body.efi` | Patched PE32 body that returns `EFI_SUCCESS`. |
   | `mod_vtddxe_remove.bin` | Full modified image with the VtdDxe FFS removed. |
   | `mod_vtddxe_ret.bin` | Full modified image with the early-return body patch. |

3. Use UEFITool NE for searching and inspection. Use a UEFITool build that can correctly remove or replace files on your Aptio generation.
4. Search the firmware image for:
   - `VtdDxe`
   - `VT-d`
   - `Directed I/O`
   - `DMAR`
   - `DMA Remapping`
   - `Intel(R) VT for Directed I/O`
5. Record the firmware file GUID, volume path, section type, PE32 image size, and compression section before changing anything.
6. Confirm the target is a DXE driver by checking for a PE32 image section and DXE dependency section. Do not remove setup forms, PEI modules, SMM drivers, or ACPI table storage just because they contain the string `VT-d`.
7. Extract both the complete FFS "as is" and the PE32 image body.
8. Reopen every saved image in UEFITool before flashing. If the firmware tree no longer parses cleanly, discard that image.

### Method A: remove the motherboard VtdDxe DXE module

> [!IMPORTANT]
> This removal method is not universal.
>
> It mainly fits low-cost, white-label, or clone-brand boards where the vendor has already cut down firmware modules for cost and profit reasons. On those boards, some VT-d dependency paths may already be incomplete, so removing the remaining VtdDxe module can work.
>
> On mainstream ASUS/MSI/Gigabyte/ASRock boards, server boards, workstations, and newer platforms, VtdDxe may be tied to other DXE, ACPI, IIO, or System Agent initialization logic. Removing it blindly can cause black screen, early boot hang, broken ACPI tables, or an unbootable BIOS image. For those boards, prefer the early-return method or a narrower function patch after confirming dependencies.

Goal:

- keep the setup menu and NVRAM state untouched;
- prevent the VtdDxe driver from being dispatched;
- stop VT-d initialization, DMAR handoff, or DMA-remapping setup handled by that driver.

Workflow:

1. Open `mod_vtddxe_work.bin` in UEFITool.
2. Locate the exact `VtdDxe`/`IntelVTdDxe` DXE driver.
3. Extract the target FFS "as is" and save it as `VtdDxe_original.ffs`.
4. Extract the PE32 image body and save it as `VtdDxe_original_body.efi`.
5. Use the firmware editor's remove operation on the whole VtdDxe FFS file, not only on one compressed child section.
6. Save the image as `mod_vtddxe_remove.bin`.
7. Close the editor, reopen `mod_vtddxe_remove.bin`, and confirm:
   - the firmware volumes parse without errors;
   - the target VtdDxe FFS is gone;
   - unrelated volumes and files are still present;
   - free-space padding and checksums are accepted by the tool.
8. If the modified BIOS is rejected by EZ Flash/M-Flash/Q-Flash, do not force the vendor flasher. Use the programmer or a board-specific flash method that can write the modified SPI image.
9. After flashing, enter BIOS first. Confirm the visible VT-d option is still in the intended state, then boot Windows and retest the target PCIe route.

Use this method only when:

- the board still boots normally without the VtdDxe driver;
- the visible BIOS VT-d switch must remain enabled;
- the goal is to stop the actual VT-d driver path instead of hiding the menu option.

Do not use it when:

- the board hangs before setup after removal;
- other DXE modules depend on protocols or events installed by this driver;
- the firmware volume rebuild changes too much structure for your recovery setup.

### Method B: replace VtdDxe with an early-return body

This is the cleaner fallback for boards that expect the VtdDxe FFS to remain present. Instead of deleting the firmware file, keep the same FFS, GUID, dependency section, and compression layout, then patch the PE32 image body so the driver's entry point returns success immediately.

Expected behavior:

- BIOS setup still shows the VT-d setting according to NVRAM/defaults.
- The VtdDxe FFS remains in the firmware tree.
- DXE dispatch sees the driver and receives `EFI_SUCCESS`.
- The driver's real VT-d initialization code does not run.
- If the OS still sees a working IOMMU/DMAR path, another module is doing the remaining work and must be traced separately.

Workflow:

1. Extract the PE32 image body from the target VtdDxe FFS as `VtdDxe_original_body.efi`.
2. Open the extracted body in a PE-aware tool such as Ghidra, IDA, CFF Explorer, PE-bear, or another tool that can show the PE optional header.
3. Record `AddressOfEntryPoint`.
4. Convert the entry-point RVA to a file offset using the PE section table. Do not patch a guessed text string offset.
5. For a normal x64 AMI DXE driver, patch the first bytes at the real entry point to return `EFI_SUCCESS`:

   ```text
   31 C0 C3
   ```

   This is:

   ```asm
   xor eax, eax
   ret
   ```

   `EFI_SUCCESS` is zero in `RAX` on x64 UEFI.

6. If the editor requires overwriting a fixed instruction window, leave the first three bytes as `31 C0 C3` and pad the rest of that small window with `90` (`NOP`). Do not shift file contents.
7. Save the patched body as `VtdDxe_ret_success_body.efi`.
8. In UEFITool, replace the original VtdDxe PE32 image body with `VtdDxe_ret_success_body.efi`. Do not replace the whole volume and do not change the module GUID unless you intentionally know why.
9. Save as `mod_vtddxe_ret.bin`.
10. Close and reopen `mod_vtddxe_ret.bin`, then verify:
    - the same VtdDxe FFS still exists;
    - the PE32 image body contains the `31 C0 C3` entry patch;
    - the firmware tree parses cleanly;
    - no unrelated module was changed.
11. Flash only after a clean parse and after the original SPI backup is already verified.
12. Boot into BIOS and confirm the visible VT-d setting is enabled.
13. Boot Windows and retest the target PCIe route.

### If early return is too aggressive

Some boards need VtdDxe to register events, install a protocol, or publish a harmless part of setup before the decisive VT-d programming step. If an entry-point return patch causes boot issues, use a narrower patch:

1. Load `VtdDxe_original_body.efi` in a disassembler.
2. Find calls or branches around strings and functions related to:
   - `DMAR`
   - `DmaRemapping`
   - `VtdEnable`
   - `EnableDmaRemapping`
   - `VtdInit`
   - `InstallAcpiTable`
   - `EFI_ACPI_DMAR`
3. Keep non-VT-d setup code intact.
4. Patch only the decisive function or branch so it returns success before enabling remapping, locking registers, or publishing the DMAR table.
5. Replace the PE32 body and retest from a cold boot.

### VtdDxe validation signs

Do not validate this by looking at the BIOS menu alone. The intended state is a mismatch:

- setup menu or NVRAM says VT-d is enabled;
- the target DMA route is not effectively blocked by the normal remapping path;
- the target card behavior changes compared with the unmodified BIOS;
- the result survives a full shutdown, PSU power drain, and cold boot.

If Windows still behaves exactly like the stock BIOS, check for a second module responsible for ACPI DMAR or IOMMU setup. Common follow-up search terms are `DMAR`, `VtdAcpi`, `AcpiPlatform`, `Iio`, `SaInit`, and `DmaRemapping`.

## Platform-specific routes

### ASUS and protected-variable cases

Some ASUS boards hide options deeply and use heavy write protection. Treat them as two possible workflows:

- with a programmer: flash the SPI or edit NVRAM directly;
- without a programmer: use a board-specific UEFI tool USB drive to change variables.

Neither route is universal. It must match the exact board and BIOS version.

#### Programmer route

1. Dump the SPI using the backup flow above.
2. Use AMIBCP/IFR to find PCIe ATS, DMA remapping, interrupt remapping, and middle-layer permission options.
3. Keep the main VT-d item enabled if that is required for the OS-visible state.
4. Disable or lock ATS and the route-specific middle-layer options as needed.
5. Reopen the saved file and verify the values.
6. Flash back to SPI and make sure `Verify` passes.
7. Boot into BIOS first, then enter the OS and test the target card.

#### UEFI USB route

1. Prepare a FAT32 USB drive.
2. Put the matching `BOOTX64.EFI`, variable tool, and scripts under `EFI/BOOT/`.
3. Enter the boot menu. Common keys:
   - ASUS: `F8`
   - MSI: `F11`
   - Gigabyte: `F12`
   - ASRock: `F11`
4. Choose `UEFI: <your USB drive>`, not Legacy mode.
5. Run only the variable script made for this exact board and BIOS. If you do not have the correct `VarOffset`, do not guess.
6. After saving and rebooting, do not clear CMOS unless you understand the consequence. Clearing CMOS may reset the variables you just changed.

### B760/H610 route: prioritize CPU-connected M.2

This platform family is easy to misunderstand.

The first long slot may be CPU-connected, but the target card shape, adapter, GPU usage, and lane sharing can make the route messy. On many budget boards, the cleaner method is to find `M.2_1`. If the manual says `From CPU` or `PCIe 4.0 x4 from CPU`, use an M.2-to-PCIe adapter from that slot.

#### Test flow

1. Remove extra drives, a second GPU, capture cards, and USB expansion cards.
2. Keep only the system drive and the target card.
3. Check the motherboard manual under `Storage` and `Expansion Slots`.
4. If `M.2_1` is CPU PCIe x4, test with that slot first.
5. Install the M.2-to-PCIe adapter firmly. Do not leave it hanging loose where it can short.
6. Connect external power if the adapter needs it.
7. Do not hot-plug while the machine is on.
8. If using a PCIe x1 riser, test the first required slot/path before judging the BIOS result.
9. In the OS, check whether the target route is stable, whether the device drops, and whether the state survives a cold boot.

B760/H610 is useful because some boards provide a cheap, clean, repeatable CPU-connected M.2 route. PCH slots producing no useful result is a common outcome.

### X299/X399/X99 route

These platforms have more CPU PCIe lanes, more slot combinations, and usually clearer IIO/PCIe menus. The downside is more sharing and conflicts. GPU, M.2, U.2, and PCIe slots can fight for lanes and cause strange enumeration.

#### Test flow

1. Use the official manual to mark CPU-connected x16/x8 slots, M.2 slots, PCH slots, and lane-sharing rules.
2. Put the target card on a CPU-connected slot first.
3. If the GPU occupies the clean CPU route, consider a CPU M.2-to-PCIe path.
4. Change one variable at a time:
   - slot first;
   - adapter second;
   - BIOS setting third.
5. If enumeration looks wrong, stop and re-check the lane sharing, adapter, and slot route before changing BIOS settings.

### B450/B550 and B85

B450/B550 are AMD platforms. The menu may not say VT-d. Look for IOMMU, ACS, ATS, PCIe ARI, SR-IOV, and similar terms. The target is still the middle permission layer and DMA/IOMMU path, not Intel menu names.

#### AMD

- find CPU x16 or CPU M.2 first;
- do not use chipset short slots as the main route;
- tune IOMMU/ACS/ATS and middle-layer blocking;
- lock defaults after editing.

#### B85 and older platforms

- prefer a full ATX board;
- test from the CPU x16 main slot first;
- avoid starting with strange short slots or cut-down board routes.

#### All platforms

- do not run memory-moving actions such as `reinc` during route testing;
- those operations can make the device drop immediately.

## Flash and test order

### Before flashing

Before flashing, confirm:

- exact board model;
- exact BIOS version;
- original backup exists;
- recovery tools are ready.

If EZ Flash, M-Flash, or Q-Flash refuses a modified BIOS, do not force it. Use a programmer or the variable route instead.

### Programmer write sequence

```text
Erase -> Blank Check -> Program -> Verify
```

Only remove the clip after all four steps pass.

### First boot and validation

First boot can take longer because of memory training. Do not cut power after 10 seconds. Wait 2-3 minutes before troubleshooting if there is still no display.

After you can enter BIOS, do not immediately load optimized defaults. If you used the variable route, also avoid clearing CMOS unless you intentionally want to reset variables.

In Windows, open Device Manager and use:

```text
View -> Devices by connection
```

Check which `PCI Express Root Port` the target card is under.

Test:

- whether the development board firmware is recognized;
- whether the intended effect appears;
- whether the result survives reboot and cold boot.

## Troubleshooting table

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| BIOS shows VT-d enabled, but the target card has no effect. | The route is going through PCH/southbridge, or ATS/middle-layer handling is not done. | Move to a CPU-connected slot or CPU M.2 route. Recheck ATS and `Failsafe`/`Optimal` values in AMIBCP. |
| System will not boot or shows a black screen after flashing. | Wrong BIOS, bad SPI write, or conflicting variables. | Power off, boot with minimum hardware, then write the original backup back with CH341A if needed. |
| Only `Show` was changed in AMIBCP, but behavior did not change. | `Show` only exposes the menu. | Change `Failsafe`, `Optimal`, or the correct Setup variable. Reopen the file after saving and confirm. |
| VtdDxe removal causes black screen or no setup screen. | The board depends on that DXE file, its protocols, or its ACPI/IIO side effects. | Restore the original SPI backup. If continuing, use the early-return method or a narrower function patch instead of deletion. |
| VtdDxe early-return patch boots, but behavior is still stock. | Another module publishes DMAR or performs the decisive VT-d/IOMMU setup. | Search for `DMAR`, `VtdAcpi`, `AcpiPlatform`, `Iio`, `SaInit`, and `DmaRemapping`; patch one confirmed branch at a time. |
| VT-d is enabled in BIOS but absent in Windows after patching. | The patch disabled too much, or the OS-visible DMAR/IOMMU path was removed instead of only neutralized. | Back up the failed image for comparison, restore stock, then retry with a narrower patch that keeps the visible handoff intact. |
| B760 adapter swap changes nothing. | Adapter direction, power, protocol, or M.2 route is wrong; M.2 may not be CPU-connected. | Test `M.2_1`, confirm Key M NVMe, use a short stable adapter, and power the adapter properly. |
| Device drops after running for a while. | Memory-moving actions, unstable power, bad riser, or PCH dead path. | Stop memory-moving actions, move to a CPU-connected route, shorten adapter cables, and retest from cold boot. |

## Final checklist

- Before buying a board, read the manual and check whether the first long slot is CPU-connected.
- Check whether `M.2_1` is CPU PCIe x4.
- For low cost, start with H610/B760 CPU M.2 routes.
- If you want fewer route headaches, consider X299/X99.
- On ASUS boards, check PCIe ATS first.
- On B760/B450-style routes, check middle-layer blocking and default locking first.
- Read the SPI twice with CH341A. If the two dumps do not match, do not write.
- In AMIBCP, do not only change `Show`; check `Failsafe` and `Optimal`.
- Use VtdDxe methods only after the slot route and normal variable route have been tested.
- For low-cost white-label or clone-brand boards, VtdDxe removal may be usable because the firmware is often already cut down.
- For mainstream or newer boards, prefer early-return or narrow function patches over deleting the whole VtdDxe module.
- Keep `backup_original.bin`, extracted VtdDxe files, and modified BIOS images clearly separated.
- Test only one route first: CPU slot or CPU M.2 adapter.
- Add GPU, drives, and other devices only after the target route is stable.
- Do not use `reinc`.
- Do not clear CMOS casually.
- Do not change ten variables at once. Change one thing, cold boot, retest, and record the result.

Conclusion: a stable VT-d bypass setup is not defined by one motherboard model or brand. It depends on a clean CPU-connected route, correct ATS/middle-layer handling, and repeatable cold-boot behavior. If those three requirements are met and problems still remain, move to the VtdDxe fallback methods.

## License

This project is released under the MIT License. See [LICENSE](LICENSE).
