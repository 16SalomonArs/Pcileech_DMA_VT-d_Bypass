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
- [Recommended route order in 2026](#recommended-route-order-in-2026)
- [Understand the PCIe path first](#understand-the-pcie-path-first)
- [Board selection](#board-selection)
- [Required tools and adapters](#required-tools-and-adapters)
- [Software downloads](#software-downloads)
- [Backup routine before BIOS work](#backup-routine-before-bios-work)
- [AMIBCP workflow](#amibcp-check-failsafe-and-optimal-not-just-show)
- [Platform-specific routes](#platform-specific-routes)
- [VtdDxe fallback methods](#vtddxe-fallback-methods)
- [DMA firmware-side DMAR rewrite](#dma-firmware-side-dmar-rewrite)
- [Flash and test order](#flash-and-test-order)
- [Troubleshooting table](#troubleshooting-table)
- [Final checklist](#final-checklist)
- [License](#license)

---

## Core concept

Do not read this as a "turn off VT-d and win" checklist. A working setup is a route problem first, a firmware problem second, and only then a tool problem.

### What a good state looks like

The end state we want:

- the OS still sees VT-d/IOMMU as present;
- but the target DMA path is not being blocked by the permission/remapping layer in the normal way.

### Order of work

Work in this order:

1. **Get the PCIe route right first.**
2. **Tune BIOS/UEFI second.**
3. **Use firmware-module or DMA-firmware methods only after the route is proven.**

### Route to build first

The route I try to build first is:

```text
PCIe device -> CPU Root Port / IIO -> memory controller -> DDR
```

PCH/chipset-side paths can still enumerate, but they are more likely to produce no effect, unstable links, dead connections, or inconsistent state.

The common mistake is staring at the VT-d menu item and ignoring the path. The useful work is PCIe routing, ATS, DMA remapping, interrupt remapping, and the permission layer between the endpoint and memory.

For many low-end Intel 600/700-series setups, the useful route is often:

```text
CPU-connected M.2 slot -> M.2-to-PCIe adapter -> target card
```

On AMD, do not expect Intel-style `VT-d` naming. You are usually dealing with `IOMMU`, `AMD-Vi`, `ACS`, `ATS`, PCIe ARI, SR-IOV, and similar options.

## Recommended route order in 2026

For Intel, start with cheap current boards that can be repeated without guessing. For AMD, go the other way and prefer higher-end boards with better CPU lane layout and native ACS/IOMMU exposure. Old HEDT boards still have value on a bench, but they are no longer the first thing I would buy for this job.

| Priority | Route | Use it when | Skip it when |
| --- | --- | --- | --- |
| 1 | Cut-down Intel 600/700-series BIOS route | You have a low-cost B660/B760/Z690/Z790 board with CPU-direct M.2/PCIe, visible VMD/PCH-FW/PTT controls, and an older official BIOS available. | The target slot is PCH-side, VMD cannot be disabled, or the board manual does not identify CPU-direct lanes. |
| 2 | High-end AMD CPU-direct route | You have a higher-end AMD board with native `AMD-Vi`, visible ACS/IOMMU controls, and a documented CPU-direct x16 or CPU-connected M.2 route. | The target slot is chipset-side, the board hides ACS/IOMMU controls, or you are trying to prove the route through a cheap cut-down AMD board. |
| 3 | Normal AMIBCP/IFR variable route | The board exposes ATS, DMA Remapping, VT-d, IOMMU, or pre-boot IOMMU variables cleanly. | Only `Show` changes are possible and defaults cannot be locked or verified. |
| 4 | VtdDxe fallback | Variable edits do not work, but BIOS recovery is ready and the board's firmware structure is understood. | You cannot recover a bad flash or cannot identify the exact VtdDxe module. |
| 5 | DMA firmware-side DMAR rewrite | The DMA device firmware can run an autonomous early-boot task and the PCIe route already works. | The card only works after Windows boots, the endpoint is already IOMMU-blocked, or there is no safe early DMA write window. |
| Legacy | X299/X399/X99 route | You already own the board and use it as a lane-rich lab platform. | You are buying hardware today only for this method. It is no longer the preferred route. |

## Understand the PCIe path first

PCIe slots may look the same from the outside. The upstream path is what decides whether the route is worth testing.

Usually:

- the first full-length x16 slot is connected directly to the CPU;
- many secondary long slots, short slots, onboard controllers, and some M.2 slots come from the PCH/chipset side;
- devices behind the PCH go through extra bridge and DMI layers, which makes ATS, DMA remapping, interrupt remapping, and ACS more likely to get involved.

Do not start with this question:

> Can this motherboard's BIOS be modified?

Start with this one:

> Which Root Port is the target card actually attached to?

If that part is unknown, programmers, USB tools, adapters, and AMIBCP edits mostly turn into noise.

## Board selection

| Platform | Usable direction | What matters | Trap to avoid |
| --- | --- | --- | --- |
| H610 / H-series | Basic H610/H610M boards; prefer CPU-connected M.2 or the first long PCIe slot. | Intel is the cheaper-the-better direction here: fewer features, fewer locks, fewer lane-sharing surprises. | Small boards have limited resources and many PCH slots. Budget does not mean every slot works. |
| B660 / B760 | Typical low-cost boards; check whether `M.2_1` or another M.2 slot is CPU PCIe x4. | A common route is CPU M.2 to memory, then M.2-to-PCIe. | The second long slot is often from the PCH. The wrong adapter path may do nothing. |
| Z690 / Z790 | Only treat as useful when using a low-end/cut-down board with clear CPU-direct M.2 or first x16 routing. | Same 600/700-series logic as B660/B760: CPU-direct route, old usable BIOS, VMD/PTT controls, and visible VT-d/IOMMU options. | Do not buy an expensive high-end Z board just for this. More features often mean more locks, more lane sharing, and more variables. |
| X299 / X399 / X99 | Legacy lab route only. Use it if you already own the board. | More CPU lanes and more M.2/PCIe combinations; useful for controlled route experiments. | Do not buy into X99/X299 today just for this. GPU, M.2, and PCIe lane conflicts can still cause strange enumeration or unstable device state. |
| AM5 / AM4 AMD | Prefer higher-end boards first; check CPU x16 and CPU-connected M.2 before anything else. | AMD is the opposite of the Intel budget-board route: better boards usually provide cleaner native `AMD-Vi`, ACS, lane wiring, and BIOS exposure. | Do not force Intel VT-d menu names onto AMD boards. Do not treat old AM3+/early AM4 or low-end AMD boards as the main proof route. |
| B85 / older platforms | Prefer full ATX boards; find the complete CPU x16 main slot. | Fewer platform variables, but many old boards have cut-down slot layouts. | Small or heavily cut-down boards and rear/PCH slots can be unstable. |

## Required tools and adapters

The programmer is for backup and recovery. It will not make a bad PCIe route good. In many B660/B760/Z690/Z790/H610 setups, the M.2-to-PCIe adapter or PCIe riser determines whether the target route is usable.

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

## Backup routine before BIOS work

Before touching the BIOS:

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

Changing `Show` only makes the menu visible. It is not the same as changing what the board boots with.

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

## Platform-specific routes

### Protected-variable boards

Some boards hide these options deeply and add heavy write protection. Treat them as two possible workflows:

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
3. Enter the boot menu and use the board manual to find the boot-menu key.
4. Choose `UEFI: <your USB drive>`, not Legacy mode.
5. Run only the variable script made for this exact board and BIOS. If you do not have the correct `VarOffset`, do not guess.
6. After saving and rebooting, do not clear CMOS unless you understand the consequence. Clearing CMOS may reset the variables you just changed.

### Intel 600/700 route: prioritize CPU-connected M.2

This is the platform family people misread the most.

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

Intel 600/700-series boards are useful because some low-end boards provide a cheap, clean, repeatable CPU-connected M.2 route. PCH slots producing no useful result is a common outcome.

#### Cut-down Intel 600/700-series BIOS route

I use this route for cheap B660/B760/Z690/Z790 boards with cut-down firmware, especially boards where the vendor clearly simplified the normal stack. Do not treat it as a universal Intel 600/700-series recipe.

The short version:

```text
oldest usable BIOS + PCH-FW/PTT off + VMD off + CPU-direct PCIe/M.2 route
```

Use the oldest official BIOS version that still supports the CPU and board revision. Some newer BIOS builds add stronger default handling around PCH firmware, VMD, storage remapping, or VT-d-related initialization, which can make the same route stop working.

In this section, "PCH-FW off" means disabling firmware TPM / Intel PTT in the PCH-FW or trusted-computing menu. It does not mean randomly disabling Intel ME, flash descriptor regions, or other platform-management firmware.

##### Hardware placement

For the target card, stay on a CPU-direct route:

| Device | Recommended placement | Notes |
| --- | --- | --- |
| System disk | Prefer an NVMe M.2 SSD. | SATA boot was not verified for this route. Do not use SATA behavior as proof that the PCIe route is good. |
| Target card through adapter | `NVME M.2_C` with a Key-M M.2-to-PCIe adapter, when this slot is CPU-direct on the board. | On the sample cut-down BIOS route, `M.2_C` was the useful adapter slot. Force `M.2_C Link Speed = Gen3` if the adapter is unstable. |
| Target card through slot | Any confirmed CPU-direct PCIe slot, usually `PCIEX16_1`. | Check the board manual. If the slot is behind PCH/chipset, do not use it as the main route. |
| PCH/chipset slots | Avoid for the target card. | `PCIEX4`, `PCIEX1`, secondary long slots, and some M.2 slots may be PCH-side depending on the board. |

If the board manual says `M.2_C` is chipset/PCH-side, do not use it for this method. Use the first CPU-connected x16 slot or another confirmed CPU-direct M.2 slot instead.

##### BIOS options to change

Menu names vary by vendor. On this cut-down UEFI layout, these are the settings worth checking.

| Area | Option name to look for | Set to | Why |
| --- | --- | --- | --- |
| BIOS version | Official board BIOS | Oldest usable version | Start from the least locked firmware version that still supports the board and CPU. |
| PCH-FW / TPM / PTT | `PCH-FW Configuration`, `TPM Device Selection`, `Firmware TPM`, `Intel PTT` | Disable firmware TPM / disable PTT | This is what I mean by "PCH-FW off" here. If the menu exposes `TPM Device Selection`, set it to `Disable Firmware TPM`. |
| VMD / RST | `VMD`, `Intel VMD Controller`, `VMD Setup Menu`, `Map this Root Port under VMD`, `Intel RST` | Disabled | Keep NVMe and PCIe storage off the VMD/RST remapping path. The target route should not appear under an Intel VMD controller. |
| VT-d visible state | `VT-d` | Enabled | Keep the visible VT-d state enabled. Do not solve this route by turning the main VT-d switch off. |
| Pre-boot IOMMU behavior | `Control Iommu Pre-boot Behavior` | `Disable IOMMU` | Keeps VT-d visible while preventing the pre-boot IOMMU path from taking control too early. |
| SR-IOV | `SR-IOV Support` | Disabled | Removes another virtualization/remapping helper from the route. |
| Above 4G decode | `Above 4G Decoding` / `Above 4G address space decoding` | Enabled | Leave enabled in this route unless it causes a separate device conflict. |
| Re-Size BAR | `Re-Size BAR Support` | Enabled | Leave enabled in this route if the board already uses it successfully. Test separately if the target card behaves differently. |
| PCIe ASPM | `Native ASPM`, `DMI Link ASPM Control`, `ASPM`, `L1 Substates`, `DMI ASPM`, `DMI Gen3 ASPM`, `PEG - ASPM` | Disabled | Remove PCIe power-saving state changes while testing the route. |
| PCIe clock gating | `PCI Express Clock Gating` | Keep default, usually Enabled | Not a main bypass switch. I only touch it when the adapter drops or link training is unstable. |
| Target M.2 link | `NVME M.2_C Link Speed` | Gen3 first | Gen3 is the safer adapter test setting. Raise speed only after the route is stable. |
| CPU x16 link | `PCIEX16_1 Link Speed` | Gen4 or Auto | Use this only if the target card is installed in the CPU-direct x16 slot. Force Gen3 if the adapter or card is unstable. |
| NVMe slots | `NVME M.2_A`, `NVME M.2_B`, `NVME M.2_C` | Enabled | Keep the slot used by the system disk and adapter enabled. |
| SATA | `SATA Controller` | Enabled only if needed | The route here was verified around M.2/NVMe. SATA boot/storage is still unproven for this path. |
| SATA power saving | `Active LPM Support` / `LPM` | Disabled | Avoid storage power management changes during testing. |
| CPU virtualization | `Intel (VMX) Virtualization Technology` | Enabled | VMX is not the same as VT-d. Keep it enabled unless you are testing a separate CPU virtualization issue. |

##### Step-by-step setup

1. Flash or downgrade to the oldest official BIOS version that supports the board and CPU.
2. Load the BIOS once and record the current settings before changing anything.
3. If the existing Windows install uses BitLocker, suspend it or save the recovery key before changing TPM/PTT or VMD. Changing these options can make the current boot disk ask for recovery or stop booting.
4. Install the system disk in an NVMe M.2 slot. Prefer M.2/NVMe only for the first test; do not use SATA behavior as the baseline.
5. Install the target card in one of these routes:
   - `M.2_C -> M.2-to-PCIe adapter -> target card`, if `M.2_C` is CPU-direct on this board;
   - `PCIEX16_1 -> target card`, if the first x16 slot is the cleaner CPU-direct route.
6. Disable PCH-FW/PTT:
   - set `TPM Device Selection = Disable Firmware TPM`;
   - if `Intel PTT` appears separately, set it to `Disabled`.
7. Disable VMD everywhere it appears:
   - `Intel VMD Controller = Disabled`;
   - `Map this Root Port under VMD = Disabled`;
   - any per-M.2 VMD mapping = `Disabled`.
8. Keep `VT-d = Enabled`.
9. Set `Control Iommu Pre-boot Behavior = Disable IOMMU`.
10. Set `SR-IOV Support = Disabled`.
11. Disable PCIe ASPM items:
    - `Native ASPM = Disabled`;
    - `DMI Link ASPM Control = Disabled`;
    - `ASPM = Disabled`;
    - `L1 Substates = Disabled`;
    - `DMI ASPM = Disabled`;
    - `DMI Gen3 ASPM = Disabled`;
    - `PEG - ASPM = Disabled`.
12. Set the adapter route link speed:
    - for `M.2_C` adapter route: `NVME M.2_C Link Speed = Gen3`;
    - for `PCIEX16_1` route: start with `Auto` or `Gen4`, then force `Gen3` if training is unstable.
13. Save settings, shut down fully, remove AC power for a short drain, then cold boot.
14. In Windows, open Device Manager and check `View -> Devices by connection`.
15. Confirm the target card is not under an Intel VMD/RST controller and not behind a PCH-side root port.
16. Test the target behavior. If the result only works after warm reboot but not cold boot, repeat the cold-boot test before calling the route stable.

##### Failure rules

- If the target card appears under `Intel VMD Controller`, VMD is still active. Go back and disable every VMD mapping option.
- If `M.2_C` does nothing, verify whether `M.2_C` is really CPU-direct on that exact board. If not, move to `PCIEX16_1` or another confirmed CPU-direct M.2 slot.
- If the BIOS has no PCH-FW/PTT or VMD controls, try the oldest official BIOS first. If the options are still absent, this specific route may not apply to that board.
- If SATA is involved and the result changes, remove SATA from the first-pass test. Use M.2/NVMe as the baseline because SATA was not verified for this route.
- If the link drops or the adapter is inconsistent, force the target route to Gen3 before changing firmware modules.

### Legacy X299/X399/X99 route

Treat this as an old lab route. If the board is already on the bench, it is still useful. If you are buying hardware today, do not start with X99/X299.

These platforms have more CPU PCIe lanes and more slot combinations, but the downside is also more sharing and conflicts. GPU, M.2, U.2, and PCIe slots can fight for lanes and cause strange enumeration. Old BIOS builds, aging boards, and mixed adapter quality can waste more time than a current Intel 600/700-series CPU-direct M.2 route.

#### Test flow

1. Use the official manual to mark CPU-connected x16/x8 slots, M.2 slots, PCH slots, and lane-sharing rules.
2. Put the target card on a CPU-connected slot first.
3. If the GPU occupies the clean CPU route, consider a CPU M.2-to-PCIe path.
4. Change one variable at a time:
   - slot first;
   - adapter second;
   - BIOS setting third.
5. If enumeration looks wrong, stop and re-check the lane sharing, adapter, and slot route before changing BIOS settings.
6. If you are buying hardware today, return to the Intel 600/700-series route first instead of chasing X99/X299 board variance.

### AMD and older fallback boards

AMD platforms do not use Intel VT-d wording. Look for `IOMMU`, `AMD-Vi`, `ACS`, `ATS`, `PCIe ARI`, `SR-IOV`, and similar terms. The target is still the middle permission layer and DMA/IOMMU path, not the name printed in the BIOS menu.

#### AMD

For AMD, native `AMD-Vi` and native `ACS` are good signs when picking a platform. They mean the board has a cleaner IOMMU/PCIe isolation foundation. That does not mean ACS is always left on for every PCILeech/DMA test. Treat ACS as a controlled variable:

```text
normal passthrough / VFIO grouping -> ACS Auto or Enabled
PCILeech / DMA route testing       -> test Auto/Enabled first, then Disabled as a cold-boot comparison
```

The first pass should prove that the route is sane. After that, change ACS only as one variable.

AMD hardware selection is the reverse of the cheap Intel route. Do not chase the lowest-end AMD board for this job. A higher-end board is usually better because it gives more CPU lanes, cleaner slot wiring, native `AMD-Vi`, better ACS exposure, and fewer hidden chipset-side traps.

Hard rule for AMD: the target card must sit on a CPU-direct PCIe path. A chipset-side slot can enumerate and still be useless for this route. Use the first CPU x16 slot, a documented CPU-connected M.2 slot with a Key-M M.2-to-PCIe adapter, or a many-lane CPU slot whose lane sharing has already been checked in the manual. Do not use a short chipset slot, a secondary chipset-fed long slot, or an unknown M.2 slot as proof.

#### AMD platform pick list

| Platform family | Socket class | Practical role | Feasibility notes | Use today? |
| --- | --- | --- | --- | --- |
| TRX50-class | sTR5 | Enthusiast / many-lane bench | Native `AMD-Vi` and `ACS`, many PCIe 5.0 lanes. Good when many CPU-direct slots are needed. | Good if budget allows. |
| WRX90-class | sWR5 | Workstation bench | Full AMD IOMMU feature set and many PCIe 5.0 lanes. Strong route-testing platform, but overkill for most builds. | Good if already planned for workstation use. |
| WRX80-class | sWRX8 | Older workstation / PRO bench | Native ACS, PCIe 4.0 is enough, used boards can be reasonable. | Good if already available or cheap. |
| X870E / X870-class | AM5 | Current high-end desktop | Current CPUs, native ACS, PCIe 5.0 options. Good for clean CPU-direct M.2 or x16 testing. | Good current choice. |
| X670E / X670-class | AM5 | Current high-end desktop | Native ACS, useful x16 bifurcation such as x8/x8 on many boards. Good when GPU and adapter routes must coexist. | Good current choice. |
| B650E / B650-class | AM5 | Current cost-effective route | Native `AMD-Vi`; ACS is usually present or at least handled cleanly by the platform, but slot wiring must be checked carefully. | Usable value route, not the first pick if budget allows higher-end AM5. |
| X570-class | AM4 | Previous high-end desktop | Native ACS, PCIe 4.0, mature BIOS. Still useful for dual-card or CPU-direct route tests. | Usable if already owned. |
| B550-class | AM4 | Previous cost-effective route | Native `AMD-Vi`; ACS support depends on board/BIOS exposure, and CPU-direct M.2/x16 routing must be verified. | Existing-board or cheap fallback only. |
| X470 / X370-class | AM4 | Old AM4 fallback | ACS may be optional, missing, or inconsistent. Good enough for light tests, not a main recommendation. | Existing-board only. |
| 990FX-class | AM3+ | Retro fallback | Has old IOMMU capability, but the platform is too old for a main build. | Not recommended as main route. |

#### AMD BIOS setup order

Do not change ten AMD options at once. Use this order and cold boot between meaningful changes:

1. Confirm the target card is on a CPU-direct route. This is mandatory, not a preference:
   - first CPU x16 slot;
   - CPU-connected M.2 slot through a Key-M M.2-to-PCIe adapter;
   - on many-lane workstation platforms, a CPU slot that is not lane-shared with the wrong device.
   If the path is chipset-side, stop here and move the card before changing BIOS options.
2. Keep CPU virtualization available:
   - `SVM Mode = Enabled` when present;
   - this is not the same switch as IOMMU, but disabling it creates unnecessary noise.
3. Keep the OS-visible IOMMU state present:
   - `IOMMU = Enabled`;
   - `AMD-Vi = Enabled` if shown separately.
4. Set ACS baseline:
   - `ACS = Auto` or `Enabled` for the first route sanity test;
   - if the target card enumerates correctly but the DMA path has no effect, test `ACS = Disabled` as the next cold-boot comparison;
   - do not use an OS-side ACS override as proof that the hardware route is good.
5. Disable target-route translation helpers when exposed:
   - `ATS = Disabled`;
   - route-specific DMA/IOMMU remapping controls = test `Disabled` only after recording the original value.
6. Keep grouping helpers conservative:
   - `ARI Forwarding = Auto` on the first pass;
   - `SR-IOV = Disabled` unless the test specifically needs it.
7. Keep decode and training stable:
   - `Above 4G Decoding = Enabled` on modern platforms;
   - force the adapter route to Gen3 or Gen4 if Auto/Gen5 training is unstable;
   - disable PCIe ASPM during route testing.
8. Save, shut down fully, remove AC briefly, then cold boot. Warm reboot results are not enough.

#### AMD ACS decision

For AMD, the useful rule is:

```text
ACS exists and works -> platform is usually better
ACS enabled          -> better for normal passthrough and IOMMU grouping
ACS disabled         -> worth testing only as a PCILeech/DMA bypass variable
```

So the operation is not "always open ACS" and not "always close ACS". First prove the CPU-direct route with ACS on `Auto` or `Enabled`; then, if the route is visible but still blocked, make ACS the only changed option and retest with ACS off.

Keep a small table while testing:

| Test | IOMMU / AMD-Vi | ACS | ATS | Result |
| --- | --- | --- | --- | --- |
| Baseline | Enabled | Auto / Enabled | Disabled if exposed | Confirms slot, adapter, and enumeration. |
| ACS comparison | Enabled | Disabled | Same as baseline | Checks whether ACS is part of the blocking path. |
| Recovery | Original value | Original value | Original value | Confirms the board can return to a known-good state. |

#### B85 and older platforms

- prefer a full ATX board;
- test from the CPU x16 main slot first;
- avoid starting with strange short slots or cut-down board routes.

#### All platforms

- do not run memory-moving actions such as `reinc` during route testing;
- those operations can make the device drop immediately.

## VtdDxe fallback methods

This is a BIOS-module fallback. I would not start here; use it only after:

1. the target card has already been moved to a CPU-connected PCIe or M.2 route;
2. AMIBCP/IFR variable edits have been tested;
3. the board still shows the wrong functional behavior even though the visible VT-d state looks correct.

`VtdDxe` is the DXE-stage driver family that many AMI Aptio firmwares use for Intel VT-d initialization, DMAR-related setup, and DMA-remapping handoff. Vendors may rename or wrap it, so the target can appear as `VtdDxe`, `IntelVTdDxe`, `DxeVtd`, `SaVtdDxe`, or only a GUID with `VT-d`/`DMAR` strings inside the PE image.

Separate these two states before patching anything:

| State | What the user sees | What this section tries to change |
| --- | --- | --- |
| Visible setup state | BIOS menu or NVRAM says VT-d is enabled. | Keep this state enabled when the route depends on the OS seeing VT-d/IOMMU. |
| Functional DXE state | The firmware driver actually initializes remapping, DMAR, or lock-down behavior. | Stop or neutralize this part when normal variable edits are not enough. |

### Choose the least invasive path

| Situation | Prefer | Why |
| --- | --- | --- |
| AMIBCP or IFR variable edits already work. | Do not patch `VtdDxe`. | A variable-level change is easier to audit and recover. |
| Low-cost white-label or clone-brand board with visibly cut-down firmware. | Try module removal first, with a verified programmer backup. | These boards may already ship with incomplete dependency paths, so removing the remaining VT-d DXE module can be enough. |
| Mainstream retail board, workstation board, server board, or newer platform. | Prefer an early-return body patch. | The firmware may expect the FFS file, dependency section, or driver dispatch result to exist. |
| Early return boots but the route still behaves like stock. | Trace and patch a narrower function. | Another module may publish DMAR or perform the decisive lock/remap work. |
| Any patch causes black screen or setup hang. | Restore the original SPI backup. | Do not continue by stacking more changes on a bad image. |

These edits are tied to one board and one BIOS version. Do not move a patched `VtdDxe` body from one image to another just because the model name looks close.

### Shared preparation

Before deleting or patching the module:

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
> Module removal is not universal.
>
> It mainly fits low-cost, white-label, or clone-brand boards where the vendor has already cut down firmware modules for cost and profit reasons. On those boards, some VT-d dependency paths may already be incomplete, so removing the remaining VtdDxe module can work.
>
> On mainstream retail boards, server boards, workstations, and newer platforms, VtdDxe may be tied to other DXE, ACPI, IIO, or System Agent initialization logic. Removing it blindly can cause black screen, early boot hang, broken ACPI tables, or an unbootable BIOS image. For those boards, prefer the early-return method or a narrower function patch after confirming dependencies.

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
8. If the modified BIOS is rejected by the built-in flasher, do not force it. Use the programmer or a board-specific flash method that can write the modified SPI image.
9. After flashing, enter BIOS first. Confirm the visible VT-d option is still in the intended state, then boot Windows and retest the target PCIe route.

Good fit:

- the board still boots normally without the VtdDxe driver;
- the visible BIOS VT-d switch must remain enabled;
- the goal is to stop the actual VT-d driver path instead of hiding the menu option.

Bad fit:

- the board hangs before setup after removal;
- other DXE modules depend on protocols or events installed by this driver;
- the firmware volume rebuild changes too much structure for your recovery setup.

### Method B: replace VtdDxe with an early-return body

For boards that expect the VtdDxe FFS to stay present, this is usually the cleaner fallback. Instead of deleting the firmware file, keep the same FFS, GUID, dependency section, and compression layout, then patch the PE32 image body so the driver's entry point returns success immediately.

What should happen:

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

   Assembly:

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

Do not call this good just because the BIOS menu looks right. The useful result is a mismatch:

- setup menu or NVRAM says VT-d is enabled;
- the target DMA route is not effectively blocked by the normal remapping path;
- the target card behavior changes compared with the unmodified BIOS;
- the result survives a full shutdown, PSU power drain, and cold boot.

If Windows still behaves exactly like the stock BIOS, check for a second module responsible for ACPI DMAR or IOMMU setup. Common follow-up search terms are `DMAR`, `VtdAcpi`, `AcpiPlatform`, `Iio`, `SaInit`, and `DmaRemapping`.

## DMA firmware-side DMAR rewrite

The third fallback works from the DMA device side. Leave the motherboard image alone, add a small DMAR-rewrite stage to the DMA firmware, let the device find the host ACPI `DMAR` table in physical memory early, patch it, fix the checksum, and then stop touching it.

Only try this after the hardware route already works. A PCH path, active VMD path, unstable riser, or bad M.2 adapter will not be saved by a DMAR patch.

### What this method changes

`DMAR` is the ACPI table that tells the OS how Intel VT-d DMA remapping hardware is exposed. The BIOS can still show VT-d as enabled, but if the OS receives a modified DMAR table, it may build a different IOMMU/remapping view.

| Layer | Normal behavior | Firmware-side rewrite goal |
| --- | --- | --- |
| BIOS setup | VT-d option remains enabled. | Keep the visible state unchanged. |
| ACPI DMAR table | Describes DRHD units, device scopes, RMRR entries, and flags. | Remove, narrow, or neutralize the remapping description that catches the target route. |
| OS boot | OS reads DMAR once during early boot and configures IOMMU. | Patch before that read. A late patch usually has no useful effect. |
| Target PCIe path | Device is still physically on the same CPU-direct route. | The route stays the same; only the OS remapping view changes. |

### When it can work

Timing is the whole game. The device has to read and write host physical memory early enough:

1. PCIe link is trained and the DMA endpoint is usable.
2. Host memory containing ACPI tables is readable.
3. IOMMU blocking has not already been fully locked for that endpoint.
4. The patch runs before Windows or another OS parses ACPI DMAR.
5. The edited table has a valid ACPI checksum.

If the OS has already parsed DMAR and configured IOMMU, changing the memory copy of the table afterward is usually too late. At that point you are only changing stale table bytes, not the already-built kernel IOMMU state.

### Firmware task design

Add the rewrite as a short, one-shot boot stage in the DMA firmware.

```text
PCIe link up
  -> wait for stable config space/BAR state
  -> locate RSDP
  -> parse XSDT/RSDT
  -> find DMAR
  -> identify target DRHD/device scope
  -> patch selected bytes
  -> recompute ACPI checksum
  -> read back and verify
  -> disable the rewrite stage until next cold boot
```

Do not spam writes in a loop. The firmware should patch once, verify once, then stop. Continuous writes during OS boot can cause inconsistent reads, ACPI checksum races, or crashes.

### Locate the DMAR table

Use the ACPI chain when possible instead of blind pattern matching.

#### Preferred path: RSDP -> XSDT -> DMAR

1. Search physical memory for the RSDP signature:

   ```text
   RSD PTR
   ```

   The real RSDP signature is eight bytes; the last byte is a space.

   Standard search ranges to try first:

   ```text
   EBDA area
   0x000E0000 - 0x000FFFFF
   ```

2. Validate the RSDP checksum:
   - ACPI 1.0 checksum covers the first 20 bytes.
   - ACPI 2.0+ extended checksum covers the full RSDP length.
3. Prefer `XsdtAddress` when present. Use `RsdtAddress` only as a fallback.
4. Read the XSDT/RSDT header and validate:
   - signature is `XSDT` or `RSDT`;
   - length is sane;
   - checksum is valid.
5. Iterate table pointers and read each ACPI table header.
6. Stop when the table signature is:

   ```text
   DMAR
   ```

#### Fallback path: physical scan for DMAR

If the firmware cannot reliably find RSDP, scan physical memory for `DMAR`, then validate the candidate as an ACPI table:

1. Signature at offset `0x00` is `DMAR`.
2. Length at offset `0x04` is sane and page-readable.
3. Revision is plausible.
4. OEM fields are printable or match the board style.
5. ACPI checksum over the full length equals zero.

Do not patch a raw `DMAR` string unless the full table validates. Firmware images, logs, and stale memory can contain the same string.

### Parse the DMAR contents

The ACPI table header is followed by Intel DMAR-specific fields and remapping structures.

Important structures:

| Structure | Type | Meaning | Patch relevance |
| --- | --- | --- | --- |
| DRHD | `0` | DMA Remapping Hardware Unit Definition. | Main target. It maps PCIe devices or whole segments to VT-d remapping hardware. |
| RMRR | `1` | Reserved Memory Region Reporting. | Usually leave alone unless you know the target is trapped by a reserved region. |
| ATSR | `2` | Root-port ATS reporting. | Relevant if the board uses ATS rules around the target path. |
| RHSA | `3` | Remapping hardware static affinity. | Usually not the first patch target. |
| ANDD | `4` | ACPI namespace device declaration. | Usually not the first patch target. |

For each DRHD:

1. Read the DRHD length.
2. Read the flags.
3. Check `INCLUDE_PCI_ALL` bit.
4. Read the PCI segment number.
5. Parse all device scopes under that DRHD.
6. Compare device scopes with the target card's bus/device/function path.

The target device identity is the PCI requester ID:

```text
Requester ID = Bus:Device.Function
```

Use the BDF observed by the DMA firmware/config-space view, then confirm it from Windows:

```text
Device Manager -> View -> Devices by connection
```

### Choose a patch strategy

Start with the least destructive patch that changes the target route.

| Patch strategy | What to edit | Result | Risk |
| --- | --- | --- | --- |
| Remove target device scope from a DRHD | Edit only the DRHD device-scope list that matches the target BDF/path. | Target device may no longer be assigned to that remapping unit. | Best targeted option, but only works when the target is explicitly scoped. |
| Clear `INCLUDE_PCI_ALL` on the matching DRHD | Clear bit 0 in the DRHD flags. | Devices not explicitly listed may stop being covered by that DRHD. | Broader effect. Can affect many devices on the segment. |
| Neutralize a target ATSR scope | Remove or narrow the ATSR scope matching the target root port. | Can stop ATS-related handling for that route. | Only useful when ATSR is actually involved. |
| Hide the whole DMAR table | Corrupt signature or remove the XSDT/RSDT DMAR pointer. | OS may behave as if no DMAR table exists. | Very broad. VT-d may disappear instead of looking enabled. Use only as a diagnostic, not the preferred route. |

Do not start by deleting the entire DMAR table if the goal is "VT-d visible, target route not effectively blocked." Full table removal often changes the OS-visible state too much and makes debugging harder.

### Device-scope patch workflow

Use this path when the target device or root port appears inside a DRHD device scope.

1. Locate the matching DRHD.
2. Copy the original DMAR table to a scratch buffer inside the device firmware.
3. Parse each child device scope.
4. Match the target path:
   - endpoint BDF, if directly scoped;
   - upstream root port path, if the table describes the bridge path.
5. Remove only that device scope from the DRHD structure.
6. Reduce the DRHD structure length by the removed scope length.
7. Reduce the total DMAR table length by the same amount.
8. Move following bytes forward inside the scratch buffer.
9. Recompute the DMAR ACPI checksum.
10. Write the complete patched DMAR table back to the original physical address.
11. Read back and verify:
    - signature is still `DMAR`;
    - table length matches;
    - all nested structure lengths still walk cleanly;
    - checksum over the full table is zero.

Build the patched table in device scratch RAM first, then write the final contiguous table back once. Editing lengths and moving bytes directly in host memory is asking for a half-written table.

### INCLUDE_PCI_ALL patch workflow

Use this path when the target is covered by a broad DRHD with `INCLUDE_PCI_ALL`.

1. Find the DRHD that covers the target segment.
2. Confirm its flags include `INCLUDE_PCI_ALL`.
3. Confirm there is not a more specific DRHD already covering the target path.
4. Clear bit 0 of the DRHD flags in the scratch buffer.
5. Recompute the DMAR ACPI checksum.
6. Write the patched table back.
7. Cold boot and check whether the target route behavior changes.

This is a coarse patch. On many consumer boards, `INCLUDE_PCI_ALL` can cover almost the whole PCI segment. Clearing it may affect more devices than intended. If Windows loses devices, storage, USB, or GPU stability, restore the original behavior and use a narrower patch.

### Checksum rule

Every ACPI table has an 8-bit checksum. The sum of all bytes in the table must equal zero modulo 256.

Patch rule:

```text
table.Checksum = 0
sum = byte_sum(table[0 : table.Length])
table.Checksum = (0x100 - (sum & 0xff)) & 0xff
```

After writing the patched table, read it back. The check is simple:

```text
byte_sum(table[0 : table.Length]) & 0xff == 0
```

If the checksum is wrong, the OS may ignore the table or fall back to a different path.

### Minimal firmware pseudocode

Treat this as the shape of the logic, not a copy-paste payload. The DMA read/write functions depend on the actual FPGA/MCU firmware.

```c
bool dmar_rewrite_once(void) {
    uint64_t rsdp = find_valid_rsdp();
    if (!rsdp) return false;

    struct acpi_table xsdt = read_xsdt_or_rsdt(rsdp);
    uint64_t dmar_pa = find_table(xsdt, "DMAR");
    if (!dmar_pa) return false;

    uint8_t dmar[MAX_DMAR_SIZE];
    size_t len = dma_read_table(dmar_pa, dmar, sizeof(dmar));
    if (!valid_acpi_table(dmar, len, "DMAR")) return false;

    struct pci_bdf target = get_own_or_configured_target_bdf();
    bool changed = patch_target_drhd_scope(dmar, &len, target);

    if (!changed) {
        changed = clear_matching_drhd_include_all(dmar, len, target.segment);
    }

    if (!changed) return false;

    acpi_fix_checksum(dmar, len);
    dma_write(dmar_pa, dmar, len);

    uint8_t verify[MAX_DMAR_SIZE];
    dma_read(dmar_pa, verify, len);
    return valid_acpi_table(verify, len, "DMAR");
}
```

Trigger it once after PCIe link training, before the OS loader finishes ACPI discovery. If the firmware has a boot-state indicator, gate this task to the early boot window. If not, run it only after a cold boot and disable it after the first successful patch.

### Firmware implementation example

The DMA device has to run this by itself. A normal PCILeech-style workflow where the host-side tool sends commands after Windows is already up is usually too late. The firmware needs an autonomous boot-stage task inside the FPGA logic, softcore, MCU, or embedded controller that can issue PCIe memory reads and writes before the OS finishes ACPI parsing.

Required firmware pieces:

| Piece | What it must do |
| --- | --- |
| PCIe link state gate | Wait until the endpoint is in a stable link state, usually LTSSM `L0`. |
| Physical memory read/write primitive | Read and write host physical memory through the DMA engine. |
| Small scratch buffer | Hold one copy of the DMAR table while patching. 4 KB is usually enough, but keep a larger cap such as 16 KB. |
| Target selector | Know which BDF/root-port path should be excluded or narrowed. Use a build-time config first. |
| ACPI parser | Locate RSDP, XSDT/RSDT, and DMAR; validate checksums before patching. |
| One-shot latch | Run once per cold boot, then stop writing. |
| Debug log | Store status codes in device RAM/registers so the host tool can read what happened later. |

The firmware state machine should look like this:

```text
RESET
  -> WAIT_PCIE_L0
  -> WAIT_HOST_MEMORY_READY
  -> READ_ONLY_PROBE
  -> PATCH_DMAR_IN_SCRATCH
  -> WRITE_BACK_ONCE
  -> VERIFY_READBACK
  -> DONE_LOCKED
```

Start read-only. First build a read-only version that finds RSDP/XSDT/DMAR and records addresses, lengths, checksums, and the detected DRHD list. Only enable write-back after the parser is stable.

#### Firmware-side host access layer

Keep the DMAR logic independent from the actual DMA core. The board-specific firmware only needs to provide these primitives:

```c
#define MAX_DMAR_SIZE 0x4000

struct pci_bdf {
    uint16_t segment;
    uint8_t bus;
    uint8_t device;
    uint8_t function;
};

bool host_read_phys(uint64_t pa, void *dst, uint32_t len);
bool host_write_phys(uint64_t pa, const void *src, uint32_t len);
uint64_t fw_get_configured_rsdp_pa(void);
uint64_t fw_get_configured_dmar_pa(void);
uint64_t fw_time_ms(void);
bool pcie_link_is_l0(void);
void fw_log_u64(uint32_t tag, uint64_t value);
```

Examples:

- On an FPGA design, `host_read_phys()` and `host_write_phys()` wrap the memory read/write TLP engine.
- On a softcore/MCU design, these functions write DMA descriptors into the PCIe engine and wait for completion.
- On a USB-controlled PCILeech design, the same logic must be moved into the device-side firmware. If it waits for a PC tool after Windows boots, the DMAR patch window is already gone.

#### Minimal ACPI structures

Use packed structures and validate every length before trusting a pointer.

```c
#pragma pack(push, 1)

struct acpi_header {
    char signature[4];
    uint32_t length;
    uint8_t revision;
    uint8_t checksum;
    char oem_id[6];
    char oem_table_id[8];
    uint32_t oem_revision;
    uint32_t creator_id;
    uint32_t creator_revision;
};

struct rsdp2 {
    char signature[8];
    uint8_t checksum;
    char oem_id[6];
    uint8_t revision;
    uint32_t rsdt_address;
    uint32_t length;
    uint64_t xsdt_address;
    uint8_t extended_checksum;
    uint8_t reserved[3];
};

struct dmar_drhd {
    uint16_t type;
    uint16_t length;
    uint8_t flags;
    uint8_t size;
    uint16_t segment;
    uint64_t register_base;
};

struct dmar_scope {
    uint8_t type;
    uint8_t length;
    uint8_t flags;
    uint8_t reserved;
    uint8_t enumeration_id;
    uint8_t start_bus;
    /* Followed by one or more 2-byte path entries: device, function. */
};

#pragma pack(pop)
```

For DMAR parsing, the first remapping structure starts after:

```text
ACPI header:        36 bytes
DMAR host width:     1 byte
DMAR flags:          1 byte
DMAR reserved:      10 bytes
Total offset:       48 bytes
```

Use:

```c
#define DMAR_STRUCT_OFFSET 48
#define DMAR_TYPE_DRHD 0
#define DRHD_INCLUDE_PCI_ALL 0x01
```

#### RSDP and DMAR finder

The implementation below shows the firmware-side flow. It deliberately does not use any OS API.

On modern UEFI boards, do not assume the RSDP will always be discoverable only through the legacy EBDA and `0xE0000-0xFFFFF` scan. ACPI says UEFI systems pass the RSDP pointer through the EFI configuration table; the RSDP structure is still in physical memory, but its address may be outside the old legacy scan range. For DMA-device firmware, use this order:

1. Best: store a known-good `rsdp_pa` / `dmar_pa` from a baseline dump for this exact board and BIOS version.
2. Good: scan known ACPI reclaim/reserved ranges if the firmware has a platform-specific memory map.
3. Fallback: scan EBDA and `0xE0000-0xFFFFF` for legacy-compatible boards.
4. Last resort: broad physical scan for `RSD PTR ` and `DMAR`, with strict checksum and length validation.

If the firmware only implements the legacy scan, say so clearly in the build notes. It is not enough for many current B660/B760 UEFI boards.

```c
static uint8_t byte_sum(const uint8_t *p, uint32_t len) {
    uint8_t s = 0;
    for (uint32_t i = 0; i < len; i++) s = (uint8_t)(s + p[i]);
    return s;
}

static bool sig4(const void *p, const char *s) {
    const uint8_t *b = (const uint8_t *)p;
    return b[0] == s[0] && b[1] == s[1] && b[2] == s[2] && b[3] == s[3];
}

static bool valid_acpi_table(const uint8_t *buf, uint32_t cap, const char *sig) {
    if (cap < sizeof(struct acpi_header)) return false;

    const struct acpi_header *h = (const struct acpi_header *)buf;
    if (!sig4(h->signature, sig)) return false;
    if (h->length < sizeof(struct acpi_header) || h->length > cap) return false;

    return byte_sum(buf, h->length) == 0;
}

static bool read_acpi_header(uint64_t pa, struct acpi_header *h) {
    return host_read_phys(pa, h, sizeof(*h)) &&
           h->length >= sizeof(*h) &&
           h->length <= MAX_DMAR_SIZE;
}

static uint64_t scan_rsdp_range(uint64_t start, uint64_t end) {
    uint8_t buf[sizeof(struct rsdp2)];

    for (uint64_t pa = start; pa + sizeof(struct rsdp2) <= end; pa += 16) {
        if (!host_read_phys(pa, buf, sizeof(buf))) continue;
        if (__builtin_memcmp(buf, "RSD PTR ", 8) != 0) continue;

        struct rsdp2 *r = (struct rsdp2 *)buf;
        if (byte_sum(buf, 20) != 0) continue;
        if (r->revision >= 2 && r->length >= sizeof(struct rsdp2)) {
            if (r->length > sizeof(buf)) continue;
            if (byte_sum(buf, r->length) != 0) continue;
        }
        return pa;
    }

    return 0;
}

static uint64_t find_rsdp(void) {
    uint64_t configured_rsdp = fw_get_configured_rsdp_pa();
    if (configured_rsdp) {
        uint8_t buf[sizeof(struct rsdp2)];
        if (host_read_phys(configured_rsdp, buf, sizeof(buf)) &&
            __builtin_memcmp(buf, "RSD PTR ", 8) == 0 &&
            byte_sum(buf, 20) == 0) {
            return configured_rsdp;
        }
    }

    uint16_t ebda_seg = 0;

    if (host_read_phys(0x40e, &ebda_seg, sizeof(ebda_seg))) {
        uint64_t ebda = ((uint64_t)ebda_seg) << 4;
        uint64_t rsdp = scan_rsdp_range(ebda, ebda + 1024);
        if (rsdp) return rsdp;
    }

    return scan_rsdp_range(0x000e0000, 0x00100000);
}

static uint64_t find_dmar_from_xsdt(uint64_t xsdt_pa) {
    uint8_t xsdt_buf[4096];
    struct acpi_header h;

    if (!read_acpi_header(xsdt_pa, &h)) return 0;
    if (!sig4(h.signature, "XSDT")) return 0;
    if (h.length > sizeof(xsdt_buf)) return 0;
    if (!host_read_phys(xsdt_pa, xsdt_buf, h.length)) return 0;
    if (!valid_acpi_table(xsdt_buf, h.length, "XSDT")) return 0;

    uint32_t count = (h.length - sizeof(struct acpi_header)) / 8;
    const uint64_t *entry = (const uint64_t *)(xsdt_buf + sizeof(struct acpi_header));

    for (uint32_t i = 0; i < count; i++) {
        struct acpi_header th;
        if (!read_acpi_header(entry[i], &th)) continue;
        if (sig4(th.signature, "DMAR")) return entry[i];
    }

    return 0;
}

static uint64_t find_dmar_table(void) {
    uint64_t configured_dmar = fw_get_configured_dmar_pa();
    if (configured_dmar) {
        struct acpi_header h;
        if (read_acpi_header(configured_dmar, &h) && sig4(h.signature, "DMAR")) {
            return configured_dmar;
        }
    }

    uint64_t rsdp_pa = find_rsdp();
    if (!rsdp_pa) return 0;

    struct rsdp2 r;
    if (!host_read_phys(rsdp_pa, &r, sizeof(r))) return 0;

    if (r.revision >= 2 && r.xsdt_address) {
        uint64_t dmar = find_dmar_from_xsdt(r.xsdt_address);
        if (dmar) return dmar;
    }

    /* Add RSDT fallback if the target board only exposes ACPI 1.0 tables. */
    return 0;
}
```

If the compiler does not support `__builtin_memcmp`, replace it with the firmware's own byte compare helper.

#### Patch option 1: clear INCLUDE_PCI_ALL

I start with this lab patch because it does not move table bytes. It proves the firmware can find DMAR, edit it, fix checksum, write back, and verify.

```c
static void acpi_fix_checksum(uint8_t *table) {
    struct acpi_header *h = (struct acpi_header *)table;
    h->checksum = 0;
    h->checksum = (uint8_t)(0u - byte_sum(table, h->length));
}

static bool clear_drhd_include_all_for_segment(uint8_t *dmar,
                                               uint32_t len,
                                               uint16_t segment) {
    if (!valid_acpi_table(dmar, len, "DMAR")) return false;

    uint32_t off = DMAR_STRUCT_OFFSET;
    bool changed = false;

    while (off + 4 <= len) {
        uint16_t type = *(uint16_t *)(dmar + off + 0);
        uint16_t slen = *(uint16_t *)(dmar + off + 2);

        if (slen < 4 || off + slen > len) return false;

        if (type == DMAR_TYPE_DRHD && slen >= sizeof(struct dmar_drhd)) {
            struct dmar_drhd *drhd = (struct dmar_drhd *)(dmar + off);

            if (drhd->segment == segment &&
                (drhd->flags & DRHD_INCLUDE_PCI_ALL)) {
                drhd->flags &= (uint8_t)~DRHD_INCLUDE_PCI_ALL;
                changed = true;
            }
        }

        off += slen;
    }

    if (!changed) return false;
    acpi_fix_checksum(dmar);
    return byte_sum(dmar, ((struct acpi_header *)dmar)->length) == 0;
}
```

It is broad. If it hits too much, move to the device-scope removal patch.

#### Patch option 2: remove one matching device scope

When the target root port or endpoint appears as a DRHD device scope, this is the cleaner final route.

In practice, configure the firmware with the scope path you want to remove. Do not guess it only from the endpoint BDF. For example:

```c
struct scope_match {
    uint8_t start_bus;
    uint8_t path_len;
    uint8_t path[8][2]; /* device, function */
};
```

Scope match helper:

```c
static bool scope_matches(const struct dmar_scope *s,
                          const struct scope_match *m) {
    if (s->start_bus != m->start_bus) return false;
    if (s->length != sizeof(struct dmar_scope) + m->path_len * 2) return false;

    const uint8_t *path = (const uint8_t *)(s + 1);
    for (uint8_t i = 0; i < m->path_len; i++) {
        if (path[i * 2 + 0] != m->path[i][0]) return false;
        if (path[i * 2 + 1] != m->path[i][1]) return false;
    }

    return true;
}
```

Remove the scope from the table copy:

```c
static bool remove_matching_scope_from_drhd(uint8_t *dmar,
                                            uint32_t *len_io,
                                            const struct scope_match *match) {
    uint32_t len = *len_io;
    struct acpi_header *h = (struct acpi_header *)dmar;

    if (!valid_acpi_table(dmar, len, "DMAR")) return false;

    uint32_t off = DMAR_STRUCT_OFFSET;

    while (off + 4 <= len) {
        uint16_t type = *(uint16_t *)(dmar + off + 0);
        uint16_t slen = *(uint16_t *)(dmar + off + 2);

        if (slen < 4 || off + slen > len) return false;

        if (type == DMAR_TYPE_DRHD && slen >= sizeof(struct dmar_drhd)) {
            uint32_t scope_off = off + sizeof(struct dmar_drhd);
            uint32_t drhd_end = off + slen;

            while (scope_off + sizeof(struct dmar_scope) <= drhd_end) {
                struct dmar_scope *scope = (struct dmar_scope *)(dmar + scope_off);

                if (scope->length < sizeof(struct dmar_scope) ||
                    scope_off + scope->length > drhd_end) {
                    return false;
                }

                if (scope_matches(scope, match)) {
                    uint32_t remove_len = scope->length;
                    uint32_t tail_src = scope_off + remove_len;
                    uint32_t tail_len = len - tail_src;

                    __builtin_memmove(dmar + scope_off, dmar + tail_src, tail_len);

                    *(uint16_t *)(dmar + off + 2) = (uint16_t)(slen - remove_len);
                    h->length -= remove_len;
                    *len_io = h->length;

                    acpi_fix_checksum(dmar);
                    return byte_sum(dmar, h->length) == 0;
                }

                scope_off += scope->length;
            }
        }

        off += slen;
    }

    return false;
}
```

If the firmware toolchain does not provide `__builtin_memmove`, implement a byte-safe `memmove` helper because source and destination overlap.

#### Full one-shot rewrite task

Now tie the firmware pieces together.

```c
enum dmar_patch_status {
    DMAR_PATCH_IDLE = 0,
    DMAR_PATCH_NO_LINK = 1,
    DMAR_PATCH_NO_DMAR = 2,
    DMAR_PATCH_READ_FAIL = 3,
    DMAR_PATCH_INVALID = 4,
    DMAR_PATCH_NO_CHANGE = 5,
    DMAR_PATCH_WRITE_FAIL = 6,
    DMAR_PATCH_VERIFY_FAIL = 7,
    DMAR_PATCH_OK = 8,
};

static uint8_t dmar_buf[MAX_DMAR_SIZE];
static bool dmar_patch_done;

enum dmar_patch_status dmar_rewrite_boot_task(void) {
    if (dmar_patch_done) return DMAR_PATCH_IDLE;
    if (!pcie_link_is_l0()) return DMAR_PATCH_NO_LINK;

    uint64_t dmar_pa = find_dmar_table();
    if (!dmar_pa) return DMAR_PATCH_NO_DMAR;

    struct acpi_header h;
    if (!read_acpi_header(dmar_pa, &h)) return DMAR_PATCH_READ_FAIL;
    if (h.length > sizeof(dmar_buf)) return DMAR_PATCH_INVALID;

    if (!host_read_phys(dmar_pa, dmar_buf, h.length)) {
        return DMAR_PATCH_READ_FAIL;
    }

    if (!valid_acpi_table(dmar_buf, h.length, "DMAR")) {
        return DMAR_PATCH_INVALID;
    }

    uint32_t new_len = h.length;
    bool changed = false;

    /* Preferred final patch: remove a configured target scope. */
    struct scope_match target_scope = {
        .start_bus = 0,
        .path_len = 1,
        .path = { { 0x01, 0x00 } }, /* example only: device 1, function 0 */
    };

    changed = remove_matching_scope_from_drhd(dmar_buf, &new_len, &target_scope);

    /* Lab fallback: prove the write path by clearing INCLUDE_PCI_ALL. */
    if (!changed) {
        changed = clear_drhd_include_all_for_segment(dmar_buf, new_len, 0);
    }

    if (!changed) return DMAR_PATCH_NO_CHANGE;

    if (!host_write_phys(dmar_pa, dmar_buf, new_len)) {
        return DMAR_PATCH_WRITE_FAIL;
    }

    uint8_t verify[64];
    if (!host_read_phys(dmar_pa, verify, sizeof(verify))) {
        return DMAR_PATCH_VERIFY_FAIL;
    }

    struct acpi_header *vh = (struct acpi_header *)verify;
    if (!sig4(vh->signature, "DMAR") || vh->length != new_len) {
        return DMAR_PATCH_VERIFY_FAIL;
    }

    dmar_patch_done = true;
    fw_log_u64(0x444d4152, dmar_pa); /* "DMAR" */
    fw_log_u64(0x444d4c45, new_len); /* "DMLE" */

    return DMAR_PATCH_OK;
}
```

The `target_scope` values must come from the actual board's DMAR dump. For a reusable firmware, expose them as build-time constants or small configuration registers:

```text
target segment
target start bus
target path length
target path device/function bytes
fallback mode: disabled / clear INCLUDE_PCI_ALL / read-only
```

#### Where to call it

Call the rewrite task from the device firmware's early main loop:

```c
void firmware_main_loop(void) {
    while (1) {
        service_pcie_core();
        service_usb_or_uart_debug();

        if (!dmar_patch_done) {
            enum dmar_patch_status st = dmar_rewrite_boot_task();
            fw_log_u64(0x44535453, st); /* "DSTS" */
        }

        service_normal_dma_mode();
    }
}
```

For an FPGA-only design, the same flow becomes a hardware FSM:

```text
state WAIT_L0:
  if ltssm == L0 -> SCAN_RSDP

state SCAN_RSDP:
  issue MRd over EBDA and 0xE0000-0xFFFFF ranges
  validate "RSD PTR " and checksum

state READ_XSDT:
  issue MRd for XSDT header and entries

state READ_DMAR:
  issue MRd for DMAR header and full table

state PATCH:
  edit scratch RAM, fix checksum

state WRITE_DMAR:
  issue MWr for patched table

state VERIFY:
  issue MRd for first header bytes and checksum sample

state DONE:
  block further writes until reset
```

The language is not the point. The patch has to run from the DMA device before the host OS consumes the ACPI table.

### Exact host-memory patch examples: RSDP and DMAR

This section is specifically about DMA device firmware patching host physical memory. It is not a motherboard BIOS patch and it is not an OS driver patch.

The DMA firmware has two practical choices:

| Method | Host memory writes | When to use |
| --- | --- | --- |
| In-place DMAR patch | Write the existing host `DMAR` table bytes directly. RSDP is only read, not modified. | Best first implementation. Fewer moving parts. |
| RSDP -> shadow XSDT -> shadow DMAR redirect | Write a patched DMAR copy into a known host scratch page, write a cloned XSDT that points to it, then patch host RSDP `XsdtAddress`. | Use when the original DMAR table area is not safe to edit in place, or when you want an easy on/off redirect. |

DMA firmware cannot ask the host OS to allocate memory at this stage. If you use the shadow method, `SHADOW_XSDT_PA` and `SHADOW_DMAR_PA` must be target-specific physical pages that are confirmed writable and not used by the firmware/boot path. If you do not have such a page, use the in-place DMAR method first.

#### RSDP fields to edit

RSDP lives in host physical memory. The DMA firmware finds it, reads it, and optionally modifies the pointer fields.

For ACPI 2.0+ RSDP:

| Offset | Field | Size | Patch use |
| --- | --- | --- | --- |
| `0x00` | Signature `RSD PTR ` | 8 | Do not change. |
| `0x08` | ACPI 1.0 checksum | 1 | Fix only if changing first 20 bytes, such as `RsdtAddress`. |
| `0x10` | `RsdtAddress` | 4 | Optional legacy redirect to a cloned RSDT. Usually not needed on modern x64 Windows if XSDT is present. |
| `0x14` | RSDP length | 4 | Do not change. |
| `0x18` | `XsdtAddress` | 8 | Main redirect field. Point it to cloned XSDT if using the shadow method. |
| `0x20` | Extended checksum | 1 | Must fix after changing `XsdtAddress`. |

To redirect XSDT from DMA firmware:

```text
old host RSDP at rsdp_pa
  rsdp_pa + 0x18 <- SHADOW_XSDT_PA
  rsdp_pa + 0x20 <- new extended checksum
```

If you also change `RsdtAddress` at `0x10`, fix both:

```text
rsdp_pa + 0x08 <- checksum over first 20 bytes
rsdp_pa + 0x20 <- checksum over full RSDP length
```

Example DMA firmware function:

```c
static bool patch_host_rsdp_xsdt(uint64_t rsdp_pa, uint64_t new_xsdt_pa) {
    struct rsdp2 r;

    if (!host_read_phys(rsdp_pa, &r, sizeof(r))) return false;
    if (__builtin_memcmp(r.signature, "RSD PTR ", 8) != 0) return false;
    if (r.revision < 2 || r.length < sizeof(struct rsdp2)) return false;
    if (r.length > sizeof(r)) return false;
    if (byte_sum((const uint8_t *)&r, r.length) != 0) return false;

    r.xsdt_address = new_xsdt_pa;
    r.extended_checksum = 0;
    r.extended_checksum = (uint8_t)(0u - byte_sum((const uint8_t *)&r, r.length));

    if (!host_write_phys(rsdp_pa + 0x18, &r.xsdt_address, sizeof(r.xsdt_address))) {
        return false;
    }
    if (!host_write_phys(rsdp_pa + 0x20, &r.extended_checksum, sizeof(r.extended_checksum))) {
        return false;
    }

    struct rsdp2 verify;
    if (!host_read_phys(rsdp_pa, &verify, sizeof(verify))) return false;
    return verify.xsdt_address == new_xsdt_pa &&
           byte_sum((const uint8_t *)&verify, verify.length) == 0;
}
```

#### XSDT fields to edit

XSDT is an ACPI table. The DMA firmware can either edit the original XSDT entry in place or build a shadow XSDT copy.

| Offset | Field | Size | Patch use |
| --- | --- | --- | --- |
| `0x00` | Signature `XSDT` | 4 | Do not change. |
| `0x04` | Length | 4 | Keep same length if only replacing one table pointer. |
| `0x09` | Checksum | 1 | Must fix after changing any entry. |
| `0x24` | First table pointer | 8 | XSDT entries begin here. Replace the entry that points to old DMAR. |

Entry count:

```text
entry_count = (Xsdt.Length - 0x24) / 8
```

Shadow XSDT builder:

```c
static bool build_shadow_xsdt_with_new_dmar(uint64_t old_xsdt_pa,
                                            uint64_t old_dmar_pa,
                                            uint64_t new_dmar_pa,
                                            uint64_t shadow_xsdt_pa) {
    uint8_t xsdt[4096];
    struct acpi_header h;

    if (!read_acpi_header(old_xsdt_pa, &h)) return false;
    if (!sig4(h.signature, "XSDT")) return false;
    if (h.length > sizeof(xsdt)) return false;
    if (!host_read_phys(old_xsdt_pa, xsdt, h.length)) return false;
    if (!valid_acpi_table(xsdt, h.length, "XSDT")) return false;

    uint64_t *entry = (uint64_t *)(xsdt + sizeof(struct acpi_header));
    uint32_t count = (h.length - sizeof(struct acpi_header)) / 8;
    bool replaced = false;

    for (uint32_t i = 0; i < count; i++) {
        if (entry[i] == old_dmar_pa) {
            entry[i] = new_dmar_pa;
            replaced = true;
            break;
        }
    }

    if (!replaced) return false;

    acpi_fix_checksum(xsdt);
    if (byte_sum(xsdt, ((struct acpi_header *)xsdt)->length) != 0) return false;

    return host_write_phys(shadow_xsdt_pa, xsdt, ((struct acpi_header *)xsdt)->length);
}
```

#### DMAR fields to edit

DMAR is the table that usually matters for this method.

ACPI header:

| Offset | Field | Size | Patch use |
| --- | --- | --- | --- |
| `0x00` | Signature `DMAR` | 4 | Do not change unless testing full DMAR hide. |
| `0x04` | Length | 4 | Change only if removing bytes, such as deleting a device scope. |
| `0x09` | Checksum | 1 | Must fix after every patch. |

DMAR-specific header:

| Offset | Field | Size | Patch use |
| --- | --- | --- | --- |
| `0x24` | Host Address Width | 1 | Usually do not change. |
| `0x25` | DMAR flags | 1 | Usually do not change first. |
| `0x30` | First remapping structure | variable | Walk structures from here. |

DRHD structure layout:

| Relative offset inside DRHD | Field | Size | Patch use |
| --- | --- | --- | --- |
| `0x00` | Type | 2 | `0` means DRHD. |
| `0x02` | Length | 2 | Decrease if removing a scope. |
| `0x04` | Flags | 1 | Bit 0 is `INCLUDE_PCI_ALL`; clear for broad test. |
| `0x05` | Size | 1 | Register-set size encoding. Do not treat this byte as reserved. |
| `0x06` | PCI Segment | 2 | Match target segment. |
| `0x08` | Register Base Address | 8 | Do not change. |
| `0x10` | First device scope | variable | Remove or narrow matching scope. |

Device scope layout:

| Relative offset inside scope | Field | Size | Patch use |
| --- | --- | --- | --- |
| `0x00` | Scope type | 1 | Endpoint, bridge, IOAPIC, HPET, etc. |
| `0x01` | Scope length | 1 | Number of bytes to remove if deleting this scope. |
| `0x02` | Scope flags | 1 | Leave unchanged unless the exact flag is understood. |
| `0x03` | Reserved | 1 | Leave unchanged. |
| `0x04` | Enumeration ID | 1 | Usually leave as is. |
| `0x05` | Start bus | 1 | Match target root-port path. |
| `0x06` | Device/function path bytes | variable | Each pair is `device, function`. |

#### Direct in-place DMAR patch demo

This demo leaves RSDP and XSDT unchanged. The DMA firmware reads old DMAR, patches it in a scratch buffer, then writes the patched bytes back to the same `old_dmar_pa`.

```c
static bool patch_dmar_in_place(uint64_t old_dmar_pa,
                                const struct scope_match *target_scope) {
    uint8_t dmar[MAX_DMAR_SIZE];
    struct acpi_header h;

    if (!read_acpi_header(old_dmar_pa, &h)) return false;
    if (!sig4(h.signature, "DMAR")) return false;
    if (h.length > sizeof(dmar)) return false;

    if (!host_read_phys(old_dmar_pa, dmar, h.length)) return false;
    if (!valid_acpi_table(dmar, h.length, "DMAR")) return false;

    uint32_t new_len = h.length;
    bool changed = remove_matching_scope_from_drhd(dmar, &new_len, target_scope);

    if (!changed) {
        changed = clear_drhd_include_all_for_segment(dmar, new_len, 0);
    }

    if (!changed) return false;
    if (!host_write_phys(old_dmar_pa, dmar, new_len)) return false;

    uint8_t verify[MAX_DMAR_SIZE];
    if (!host_read_phys(old_dmar_pa, verify, new_len)) return false;

    return valid_acpi_table(verify, new_len, "DMAR");
}
```

Good fit:

- the existing DMAR memory area is writable by the DMA device;
- you do not need an on/off redirect;
- you want the simplest proof that the device firmware can change host DMAR before OS parsing.

#### RSDP redirect + shadow DMAR demo

This demo changes host RSDP, so the OS reaches a cloned XSDT and a cloned patched DMAR table.

Write order matters:

```text
1. Read original RSDP.
2. Read original XSDT.
3. Find original DMAR pointer in XSDT.
4. Read original DMAR.
5. Build patched DMAR in device scratch buffer.
6. Write patched DMAR to SHADOW_DMAR_PA.
7. Build cloned XSDT where old DMAR pointer is replaced by SHADOW_DMAR_PA.
8. Write cloned XSDT to SHADOW_XSDT_PA.
9. Patch host RSDP XsdtAddress to SHADOW_XSDT_PA.
10. Fix and verify RSDP extended checksum.
```

Firmware demo:

```c
#define SHADOW_XSDT_PA TARGET_SPECIFIC_SHADOW_XSDT_PA
#define SHADOW_DMAR_PA TARGET_SPECIFIC_SHADOW_DMAR_PA

static bool patch_via_rsdp_redirect(const struct scope_match *target_scope) {
    uint64_t rsdp_pa = find_rsdp();
    if (!rsdp_pa) return false;

    struct rsdp2 r;
    if (!host_read_phys(rsdp_pa, &r, sizeof(r))) return false;
    if (r.revision < 2 || !r.xsdt_address) return false;

    uint64_t old_xsdt_pa = r.xsdt_address;
    uint64_t old_dmar_pa = find_dmar_from_xsdt(old_xsdt_pa);
    if (!old_dmar_pa) return false;

    uint8_t dmar[MAX_DMAR_SIZE];
    struct acpi_header dh;
    if (!read_acpi_header(old_dmar_pa, &dh)) return false;
    if (!host_read_phys(old_dmar_pa, dmar, dh.length)) return false;
    if (!valid_acpi_table(dmar, dh.length, "DMAR")) return false;

    uint32_t new_dmar_len = dh.length;
    bool changed = remove_matching_scope_from_drhd(dmar, &new_dmar_len, target_scope);
    if (!changed) {
        changed = clear_drhd_include_all_for_segment(dmar, new_dmar_len, 0);
    }
    if (!changed) return false;

    if (!host_write_phys(SHADOW_DMAR_PA, dmar, new_dmar_len)) return false;

    if (!build_shadow_xsdt_with_new_dmar(old_xsdt_pa,
                                         old_dmar_pa,
                                         SHADOW_DMAR_PA,
                                         SHADOW_XSDT_PA)) {
        return false;
    }

    if (!patch_host_rsdp_xsdt(rsdp_pa, SHADOW_XSDT_PA)) return false;

    return true;
}
```

For this redirect method, the only original host structure that must be modified is RSDP:

```text
RSDP + 0x18 = SHADOW_XSDT_PA
RSDP + 0x20 = fixed extended checksum
```

The original XSDT and original DMAR can stay untouched. The OS follows RSDP to the shadow XSDT, then follows the shadow XSDT's DMAR entry to the shadow DMAR.

Good fit:

- you have confirmed safe shadow pages;
- you want to preserve the original ACPI tables for comparison;
- in-place DMAR writes are unreliable or overwritten before OS parsing.

Bad fit:

- you do not know a safe writable host physical page;
- the platform uses only RSDT and no XSDT, unless you also implement cloned RSDT redirection;
- the OS or firmware has already consumed ACPI before the DMA firmware runs.

### Validation

Validate it in two places.

Firmware-side validation:

- RSDP checksum valid.
- XSDT/RSDT checksum valid.
- DMAR checksum valid before patch.
- DMAR checksum valid after patch.
- DRHD structure walk does not overrun the table length.
- Patched bytes read back exactly.

Windows-side validation:

1. Cold boot, not warm reboot.
2. Open Device Manager:

   ```text
   View -> Devices by connection
   ```

3. Confirm the target card is still on the intended CPU-direct root port.
4. Confirm the target is not under VMD/RST.
5. Compare behavior against the same BIOS with DMAR rewrite disabled.
6. If using an ACPI dump tool, compare the `DMAR` table before and after the firmware rewrite.

### Failure rules

- If the patch only works on warm reboot, the timing is wrong. Move the rewrite earlier.
- If Windows loses VT-d/IOMMU completely, the patch is too broad. Avoid hiding the whole DMAR table.
- If the machine hangs during boot, restore the firmware to a read-only/logging mode and verify the checksum and structure lengths.
- If nothing changes, the OS may already have parsed DMAR before the write, the endpoint may already be blocked, or another ACPI/IOMMU table path may be involved.
- If storage disappears, VMD/RST or a broad `INCLUDE_PCI_ALL` patch likely affected more devices than intended.

## Flash and test order

### Before flashing

Before flashing, confirm these four things:

- exact board model;
- exact BIOS version;
- original backup exists;
- recovery tools are ready.

If the built-in flasher refuses a modified BIOS, do not force it. Use a programmer or the variable route instead.

### Programmer write sequence

```text
Erase -> Blank Check -> Program -> Verify
```

Only remove the clip after all four steps pass.

### First boot and validation

First boot can be slow because of memory training. Do not cut power after 10 seconds. Wait 2-3 minutes before troubleshooting if there is still no display.

After you can enter BIOS, do not immediately load optimized defaults. If you used the variable route, also avoid clearing CMOS unless you intentionally want to reset variables.

In Windows, open Device Manager and use:

```text
View -> Devices by connection
```

Check which `PCI Express Root Port` the target card is under.

Then test:

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
| DMA firmware DMAR rewrite changes nothing. | The patch ran after the OS already parsed DMAR, or the card was already blocked before the write. | Move the rewrite earlier in the device firmware boot flow, verify read/write access before OS handoff, and compare a cold boot with the rewrite disabled. |
| DMAR rewrite causes boot hang. | Bad checksum, broken structure length, or a broad patch such as hiding the whole table. | Revert to read-only logging, validate RSDP/XSDT/DMAR checksums, then patch only the target DRHD scope or matching `INCLUDE_PCI_ALL` bit. |
| DMAR rewrite makes VT-d disappear completely. | The whole DMAR table was hidden or corrupted instead of narrowed. | Restore the original table behavior and use a targeted device-scope or DRHD flag patch. |
| Intel 600/700 adapter swap changes nothing. | Adapter direction, power, protocol, or M.2 route is wrong; M.2 may not be CPU-connected. | Test `M.2_1`, confirm Key M NVMe, use a short stable adapter, and power the adapter properly. |
| Cut-down Intel 600/700 board still has no effect after BIOS settings. | VMD may still be active, PCH-FW/PTT may still be enabled, or the card is still on a PCH route. | Disable every VMD mapping, set firmware TPM/PTT off, then move the card to `M.2_C` or `PCIEX16_1` only if that route is CPU-direct. |
| Target card appears under Intel VMD/RST in Windows. | VMD was not fully disabled. | Return to BIOS and disable `Intel VMD Controller`, per-port VMD mapping, and RST/VMD storage mapping. |
| `M.2_C` adapter route trains but drops or behaves inconsistently. | Link speed or adapter signal quality is unstable. | Force `NVME M.2_C Link Speed = Gen3`, use a shorter adapter path, and cold boot retest. |
| AMD route enumerates but has no effect. | The target card may be on a chipset-side slot or a non-CPU M.2 route. | Move it to the first CPU x16 slot, a documented CPU-connected M.2 adapter route, or a verified CPU lane slot on a higher-end board. |
| AMD board groups devices well, but the DMA route still has no effect. | ACS may be helping normal IOMMU grouping while still participating in the blocking path for this route. | Keep `IOMMU/AMD-Vi` enabled, then test `ACS = Disabled` as the only changed BIOS variable and cold boot. |
| AMD route becomes unstable after turning ACS off. | The board or slot depends on ACS behavior for clean routing/grouping. | Restore `ACS = Auto` or `Enabled`, then continue with ATS or route-specific IOMMU options instead. |
| AMD old-platform board behaves inconsistently. | Early AM4 or AM3+ IOMMU/ACS handling is uneven. | Treat it as a fallback only. Move the same card and adapter to a current AM5 or late AM4 platform before blaming the firmware method. |
| Device drops after running for a while. | Memory-moving actions, unstable power, bad riser, or PCH dead path. | Stop memory-moving actions, move to a CPU-connected route, shorten adapter cables, and retest from cold boot. |

## Final checklist

- Before buying a board, read the manual and check whether the first long slot is CPU-connected.
- Check whether `M.2_1` is CPU PCIe x4.
- Intel is the low-end-board route: start with H610/B660/B760 and only use Z690/Z790 when it is a low-end/cut-down board with a clear CPU-direct route.
- On cut-down B660/B760/Z690/Z790 boards, test the oldest official BIOS before assuming the route is impossible.
- For the cut-down Intel 600/700 route, disable PCH-FW/PTT and VMD before testing the adapter.
- Use X99/X299 only as an existing lab platform; do not make it the current first-choice route.
- On protected-variable boards, check PCIe ATS first.
- On B760/B450-style routes, check middle-layer blocking and default locking first.
- Keep VT-d enabled for visibility, but set pre-boot IOMMU behavior to disabled when using the cut-down Intel 600/700 route.
- Use `M.2_C` with an M.2-to-PCIe adapter only when that slot is CPU-direct on the exact board.
- Prefer an M.2/NVMe system disk for this route. SATA was not verified as the baseline path.
- AMD is the high-end-board route: prefer higher-end AM5, many-lane, or workstation-class platforms with native `AMD-Vi` and visible ACS/IOMMU controls.
- On AMD, the target card must be on a CPU-direct PCIe slot or a documented CPU-connected M.2 adapter route. Chipset-side slots are not a valid proof route.
- For AMD normal passthrough, ACS is usually a plus; for PCILeech/DMA testing, ACS is a controlled variable, not a fixed answer.
- On AMD, keep `IOMMU/AMD-Vi` enabled for visibility, then compare `ACS Auto/Enabled` against `ACS Disabled` with cold boots.
- Do not use old AM3+ or early AM4 as the main proof platform unless you are only doing fallback testing.
- Read the SPI twice with CH341A. If the two dumps do not match, do not write.
- In AMIBCP, do not only change `Show`; check `Failsafe` and `Optimal`.
- Use VtdDxe methods only after the slot route and normal variable route have been tested.
- For low-cost white-label or clone-brand boards, VtdDxe removal may be usable because the firmware is often already cut down.
- For mainstream or newer boards, prefer early-return or narrow function patches over deleting the whole VtdDxe module.
- Keep `backup_original.bin`, extracted VtdDxe files, and modified BIOS images clearly separated.
- Use DMA firmware-side DMAR rewrite only after the physical PCIe/M.2 route is already stable.
- Run DMAR rewrite before the OS parses ACPI; late writes usually do not change the active IOMMU state.
- After any DMAR rewrite, fix the ACPI checksum and verify the patched table by reading it back.
- Prefer targeted DRHD device-scope or `INCLUDE_PCI_ALL` edits over hiding the whole DMAR table.
- Test only one route first: CPU slot or CPU M.2 adapter.
- Add GPU, drives, and other devices only after the target route is stable.
- Do not use `reinc`.
- Do not clear CMOS casually.
- Do not change ten variables at once. Change one thing, cold boot, retest, and record the result.

Bottom line: a stable VT-d bypass setup is not defined by one motherboard model or brand. It depends on a clean CPU-connected route, correct ATS/middle-layer handling, and repeatable cold-boot behavior. If those three requirements are met and problems still remain, move to the VtdDxe fallback methods or DMA firmware-side DMAR rewrite.

## License

This project is released under the MIT License. See [LICENSE](LICENSE).
