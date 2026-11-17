# Framework Laptop 13 Pro macOS Sequoia technical findings

**Coverage:** Boot attempts 1-97, through November 17, 2026  
**Platform:** Framework Laptop 13 Pro, Intel Panther Lake  
**Evidence:** [Boot attempt journal](logs/fw13p-attempts-sequoia.md) and [visual archive](logs/fw13p-visual-sequoia/)

## Summary

The project progressed from a boot that produced no visible kernel output to graphical Sequoia Recovery, a working Recovery Terminal, and an offline installation of macOS Sequoia 15.8.3. The installed desktop runs with basic display output, a built-in PS/2 keyboard, and a native I²C touchpad with gestures and configurable settings. Native touchpad stability has limited repeat testing.

The latest verified milestone is **native macOS Wi-Fi association and DHCP on the Intel BE211**. In Attempt 97, macOS joined an open 2.4 GHz network from its Wi-Fi menu, obtained and published an IPv4 lease on `en0`, and resolved the router over ARP. The diagnostic association ran for 15 seconds. The current data path carries DHCP and ARP. Ordinary IP traffic, internet access, and long-running connectivity remain unverified.

## Test platform and baselines

| Item | Recorded configuration |
| --- | --- |
| Processor | Intel Core Ultra X7 358H, 16 cores and 16 threads |
| Memory | 32 GB |
| Graphics | Intel Arc B390 (Xe3), PCI ID `8086:b082` |
| Wireless | Intel BE211 Wi-Fi 7, PCI ID `8086:e440`, subsystem `8086:0114`. Linux uses the SC firmware family and `iwlmld`. |
| Internal storage | ADATA LEGEND 900 NVMe, reported as `SLEG-900-001TC` in Recovery |
| Recovery target | Sequoia 15.4.1, build `24E263`, confirmed by `sw_vers` in Recovery Terminal |
| Installed target | Sequoia 15.8.3, build `24H241`, Darwin 24.6.0 |
| Bootloader | OpenCore DEBUG 1.0.8 on removable EFI |
| Firmware baseline | VT-d disabled. Linux DMI reported firmware version 03.02 in the initial inventory. |

The initial investigation used removable EFI changes, firmware settings, and read-only diagnostics. The project later expanded to an explicitly tested offline installation on a dedicated internal APFS partition. Recovery kernel analysis and installed-system analysis used separate kernel collections. Symbols, addresses, and binary patches in this report refer to their specific build.

The [attempt journal](logs/fw13p-attempts-sequoia.md) provides the chronological account. Some early EFI folders originally had separate Test numbers. In particular, the shared xHCI start trace called Test 17 in early notes corresponds to journal Attempt 25. Terminal proof is recorded as a user milestone in Attempt 40, with the booted EFI later identified as Attempt 37.

## 1. UEFI-to-kernel transition and CPU compatibility

### Attempts 1-12 - The last OpenCore line was ambiguous

Attempts 1-10 repeatedly ended in the OpenCore file log at `EXITBS:START`. The tests changed MMIO and virtual-map quirks, write protection and runtime permissions, CPUID emulation, CFG Lock and extra MSR workarounds, `ForceExitBootServices`, and a DMAR deletion. The patch and table-deletion logs showed that OpenCore applied several changes, with no corresponding visible boot progress. The elapsed times in the journal came from OpenCore logs. The time spent waiting after the last visible line was generally unavailable.

Attempt 11 disabled VT-d in firmware. The before and after Linux ACPI captures showed that the MADT changed from 16 enabled type-9 x2APIC records and a type-10 x2APIC NMI to 16 type-0 Local APIC records and a type-4 Local APIC NMI. The enabled CPU IDs and UIDs, IOAPIC, and two interrupt overrides were retained. Visible macOS kernel output appeared after this change, even though the OpenCore file still ended at `EXITBS:START`. The result supports an APIC compatibility explanation. Other effects of the firmware setting changed in the same test. The working firmware already supplies classic Local APIC records, so the tested EFI uses the firmware MADT.

The first visible kernel log ended near the AppleLockdownMode restoreOS parity check. Limiting startup to one CPU with `cpus=1` in Attempt 12 produced no further visible progress by itself.

### Attempts 13-20 - Legacy CPU objects advanced startup

The Panther Lake DSDT described CPU devices as `ACPI0007` objects. Attempt 13 added `SSDT-FW-CPU.aml`, which exposes 16 legacy ACPI `Processor` objects for Darwin, using the enabled IDs in the actual MADT. The firmware CPU objects were preserved. The kernel listed IDs 0-15, completed PCI configuration, reached DSMOS, and loaded APFS. Attempt 14 removed `cpus=1` and retained this progress.

The internal NVMe driver logged assertions and subsequently reported successful initialization. Its media proceeded to partition probing. Temporarily masking the NVMe match in Attempt 15 did not materially advance boot. Attempts 16-20 tested USB ownership, increased IOKit logging, and adapted DebugEnhancer to keep useful startup messages visible. The recurring late NVMe and power-management lines provided evidence that other kernel work continued while USB startup was still pending.

## 2. xHCI startup dependency and Recovery boot

### Attempts 21-33 - An indefinite resource wait

The Recovery kernel collection supplied the function symbols and instruction sequences used in the diagnostic hooks. The tracing followed the existing calls and recorded entry and return values. The shared `AppleUSBXHCI::start` entry was found at unslid address `0xffffff8002dce5e0` in the archived Recovery image.

| Attempts | Observed boundary |
| --- | --- |
| 21-22 | Both USB controllers reached IOKit startup. The four traced resource checks returned 1 for the providers and drivers. |
| 23-25 | Both controllers entered `AppleUSBXHCIPCI::start` and `AppleUSBXHCI::start` without visible returns. Changing `ReleaseUsbOwnership` did not clear the stall. |
| 26-28 | Mapping, register reads and capability traversal completed on the visible path. The startup trace reached `AppleUSBHostController::start`. One run recorded a five-minute wait. |
| 29-30 | Workloop, gate and matching setup returned. `waitForMatchingService` waited indefinitely for `AppleUSBHostResources`. The `IORTC` publication call returned and no USB host resources startup was visible at that checkpoint. |
| 31-32 | Resource candidates saw `IORTC`. The TypeC probe returned null. The TypeCBPC probe entered a wait for an ACPI device named `EC`, with no return recorded while either controller waited. |
| 33 | Adding a Darwin-only `EC` stub reached the graphical `support.apple.com/macsetup` input screen. |

`SSDT-FW-EC.aml` publishes `\_SB.PC00.LPCB.EC` with ID `ACID0001` and no EC register resources. The real Framework `EC0` remains present. Attempt 33 changed only the stub and its table entry relative to the preceding tracing EFI. The first GUI run lacked a full trace of the resource probe transition. Later Recovery diagnostics showed both `AppleUSBHostResources` waits and both controller starts returning successfully.

### Attempts 34-41 - Recovery and Terminal, with incomplete USB enumeration

Attempt 34 added VoodooPS2. Attempt 35 added the firmware's explicit keyboard and mouse hardware IDs and reached the Russian-language Recovery Utilities menu. Recovery continued to alternate between Utilities and the input-detection prompt in later variants. `prev-lang:kbd` was changed from `ru-RU:252` to `en-US:0`, and Attempt 37 reached English Recovery. The user later verified that keyboard shortcuts could open Terminal with this EFI. Terminal photographs show Sequoia 15.4.1 build `24E263`. Input behavior varied across boots. Neither the Logitech M330 receiver nor the external keyboard established a working USB input path in Recovery.

Attempt 41 exposed a separate problem after xHCI startup. Both controllers had drivers attached and successful startup traces. The IOUSB plane contained only its root and `diskutil` listed no external physical disks. The filtered IOService tree showed four SuperSpeed port objects under TXHC, no USB 2 objects, and no visible XHCI port objects. Linux read the following Supported Protocol ranges from the same hardware.

| Controller | Linux USB 2 hardware ports | Linux SuperSpeed hardware ports | Recovery query |
| --- | --- | --- | --- |
| XHCI `00:14.0` | 1-10 | 11-12 | No ports shown in the filtered query |
| TXHC `00:0d.0` | 1 | 2-5 | Four SuperSpeed port objects |

Both controllers logged `AppleUSBXHCI::createPorts: speed index 4 has inconsistent speed IDs (5 and 6)`. Read-only disassembly of the Recovery binary showed that the warning path continues capability parsing. The warning is consistent with the Linux register dump and does not independently explain missing USB 2 ports.

Linux traced the boot stick to TXHC bus 2, hub port 2, which maps to hardware port 3 (`SS02`), with a negotiated rate of 5000 Mb/s. A read-only PORTSC snapshot showed that port connected, powered and enabled in U0 at speed ID 4. The macOS port status and event delivery path remain unverified. The internal NVMe model in the kernel output is distinct from this USB stick.

## 3. Input, offline installation, and I²C interrupt routing

### Attempts 42-54 - Separate input paths and a usable desktop

Firmware exposes the built-in keyboard as `PS2K` with hardware ID `MSFT0001`, compatible ID `PNP0303`, and IRQ 1. The EC's basic pointing fallback appears as `PS2M` with hardware ID `FRMW0002`, compatible ID `PNP0F03`, and IRQ 12 when enabled. The native PixArt `PIXA3854` touchpad is the HID-over-I²C `TPAD` child of `I2C5`, compatible with `PNP0C50`. Linux also independently identified the Keychron K3 Max on PCI xHCI as a USB 2 device.

The first I²C builds regressed to the `macsetup` screen or stalled earlier. Removing the I2C5 PCI match, running helper kexts alone, and adding the polling flag alone did not isolate a single cause. The quieter Attempt 43 baseline reached Terminal and showed the native controller and touchpad in IORegistry. The EFI was reused in Attempt 48. A cold boot demonstrated basic PS/2 touchpad movement with mouse emulation set to Auto. A restart from Recovery returned to input detection. The same EFI therefore produced different outcomes across boot paths.

Attempt 49 installed **Sequoia 15.8.3 offline**. The full `InstallAssistant.pkg` obtained with gibMacOS was prepared with UnPlugged in Recovery. The project used a manually mounted ExFAT installer-data partition and a dedicated roughly 150 GiB internal APFS target, booted from the removable OpenCore EFI. Setup completed and the installed desktop and System Settings opened. This install pathway provided local persistent captures for diagnosing later input failures.

On the desktop, the Attempt 43 baseline showed an initial PS/2 stream 1 timeout in 11 early snapshots. Three also showed a second timeout followed by mouse probe failure. Attempt 54 added `ps2rst=2` to skip the boot reset. Five new early captures showed keyboard and pointing HID registration without a recorded stream 1 timeout. Initial tests showed typing and basic touchpad input working across several boot methods. A later Linux KDE to macOS restart still failed input initialization. Connecting the USB mouse receiver was part of the test conditions. The mouse itself never responded.

### Attempts 50-59 - Dynamic HPET allocation blocked I2C5

The installed-system VoodooI2C tests eventually matched the exact controller `8086:e451` at `00:19.1`. Correcting the controller reference clock from 133 MHz to the 100 MHz value used by Linux let VoodooI2C recognize the Synopsys controller and finish bus timing setup. It repeatedly failed PCI interrupt registration with empty interrupt arrays. The touch-report polling option could only take effect after controller initialization.

| Attempt | Evidence |
| --- | --- |
| 56 | IOPCIFamily read INTB. AppleACPIPlatform resolved it to GSI 21, then `SetDeviceInterrupts` returned `0xe00002df` (`kIOReturnNoInterrupt`). The outer resolver reported success. |
| 57 | The allocator rejected the GSI 21 request with flags `0x7` before interrupt-controller lookup or publication. |
| 58 | AppleHPET had already claimed GSI 21 as non-shared, using flags `0x0`. The second request produced the logged allocation conflict. |
| 59 | A build-specific OpenCore patch cleared bit 21 in AppleHPET's local timer route mask. HPET used GSI 20, 22, 11 and 23. I2C5 then allocated, published and registered GSI 21 successfully. |

The HPET patch targets `configureTimer` in the installed `24H241` AppleHPET binary. The archived 95-byte pattern was unique in the verified collection and began at unslid address `0xffffff8001946374`. The patch preserves the firmware's I2C5 route. The controller uses GSI 21 for bus transactions, while VoodooI2CHID polls the touchpad for reports. VoodooInput and Apple's multitouch driver attached. Gestures, scrolling, two-finger secondary click and Trackpad settings worked. The touchpad also worked after a KDE-to-macOS restart and a macOS-to-macOS restart during follow-up Wi-Fi testing. These results are functional evidence with limited repeat coverage.

The removable OPENCORE volume also remounted read-only during several early and I²C tests. Linux reported FAT directory corruption while the block device reported write protection off. The read-only check showed damaged Spotlight and Trash entries, shared clusters and a dirty bit. The active EFI still matched the staged manifest in several affected runs. The user repaired the FAT volume after backup. The evidence establishes a filesystem failure alongside the boot investigation. Its timing and relation to individual tests remain undetermined.

## 4. Intel BE211 Wi-Fi bring-up

### Attempts 60-72 - Firmware startup, scanning, and the native interface

The experimental AirportItlwm build started from the BE200 fork's integrated BZ transport and added the BE211 SC A0 / WH B0 match. The firmware image was Intel `iwlwifi-sc-a0-wh-b0-c106.ucode`, corresponding to Linux's working Core 106 image. The source baseline was `FIX31-20261109` at `ff82f3e5647ce3885f5d48417b6356a6a23d2163`. The kext was cross-built for Sequoia on Linux using Clang, cctools ld64 and MacKernelSDK commit `3f750085caa17ec3a7880f11c11bf4f48cd6a164`.

- **60-62:** MSI-X, 36-bit direct DMA and firmware ALIVE v8 worked. Two embedded PNVM segments were staged with version `0x08ec954f`. A working Linux boot reported the same completion status `0x00050001`, allowing the probe to continue through INIT_EXTENDED_CFG, PHY configuration v3 and INIT_COMPLETE.
- **63-67:** NVM_GET_INFO returned a v5 response, with TX and RX antenna masks `0x3`, LAR enabled and 110 channel entries. SHARED_MEM_CFG v4, antenna configuration, Bluetooth coexistence, SoC timing, thermal reporting, scan configuration and device power commands returned success. MCC_UPDATE returned the world-domain profile. NVM and regulatory gates controlled channel eligibility.
- **68-69:** A passive scan across 11 channels completed in about 1.26 seconds. The first run counted receive notifications. The next parsed v8 RX descriptors and verified 26 beacon frames from 22 BSSIDs.
- **70-72:** The first interface-registration run panicked at `ether_ifattach + 0x1b`. The panic UUID matched the built kext, and disassembly found a `ud2` after a non-void helper fell through. Changing the helper to `void` let Skywalk and BSD registration complete. macOS listed Wi-Fi as `en0`. The native Wi-Fi switch stayed on and the menu populated with nearby 2.4 GHz networks.

### Attempts 73-90 - Firmware contexts, association, and diagnostic DHCP

| Attempts | Finding |
| --- | --- |
| 73-78 | MAC and PHY contexts worked, followed by inactive LINK_CONFIG v8 and PHY binding. LINK_CONFIG activation initially triggered `ADVANCED_SYSASSERT 0x2010303E`. Activation succeeded with a 100 TU beacon interval, default contention settings, TGN QoS and corrected OFDM rate bits. This tested the payload as a combination. STA_CONFIG v3 then added a firmware peer. |
| 79-81 | Management queue setup reached a v2 scheduler reply. With a 128-slot ring, 32 KiB descriptors and an 8 KiB byte-count table, the reply named queue 1. Removing the preallocated host descriptor ring from that slot let the driver accept queue 1 and confirm empty-queue removal. The meaning of reply flags `0x1` remained unresolved. |
| 82-85 | Authentication TX initially timed out with no completion or matched reply. Changing priority and letting firmware select the rate reproduced the failure. Waiting for firmware to start a 878 TU session-protection window enabled a successful Open System TX completion and a status-zero AP reply. |
| 86-87 | The probe sent authentication and association to an eligible open access point, received an AID, and programmed that AID into the firmware station and MAC contexts before ending session protection. A beacon from the same peer confirmed reception after the handoff. |
| 88-90 | The best-effort data queue consumed the first Discover descriptor, but TX status `0x83` produced no matching Offer. Selecting the AP's lowest advertised basic legacy rate, 12 Mb/s on antenna A, produced a successful Discover and Offer. Attempt 90 added Request and ACK and negotiated a 7,200-second lease in the driver diagnostic. |

The management exchange and low-rate data test exercised distinct firmware paths. The lease in Attempt 90 belonged to a short-lived driver diagnostic. That build cleared the association before macOS could use an address.

### Attempts 91-97 - macOS Wi-Fi handoff and native DHCP

**91-93: The menu could initiate an exchange.** A native Wi-Fi menu request invoked the diagnostic join. The driver honored a valid requested BSSID and required an eligible open 2.4 GHz beacon. Immediate retries sometimes missed the pinned radio even when another BSSID on the same SSID was visible. Several requests completed firmware association and diagnostic DHCP. Attempt 93 held the association for two seconds after the ACK. Thirteen peer beacons arrived, the last at 1,816 ms. A subsequent authentication transmission completed with no matching AP response before the timeout.

**94: BSD link-up alone left too little time for DHCP.** The first host packet experiment opened a 15-second `en0` window for native DHCP and ARP. Two windows initially recorded zero firmware submissions. Follow-up `ipconfig` snapshots showed that macOS could see the link active and enter DHCP INIT. A synchronized capture then showed an approximately 10.9-second gap between the active-link message and `no SSID`. IPConfiguration logged a broadcast transmit attempt four milliseconds before link-down, while tcpdump and driver counters saw no DHCP or ARP traffic. The delay implicated Wi-Fi link publication and current-network handoff. The capture did not identify the specific blocked call.

**95-96: macOS recognized the join and immediately requested leave.** Moving the join into a worker let the menu request return promptly. The driver posted actual peer metadata, a first-beacon event, and successful ConnectComplete. Native WCL logged Joined Bss. The packet window nonetheless closed within about a millisecond on Attempt 95. Attempt 96 traced a native WCL leave request about 43 microseconds after the extended BSS getter returned `kIOReturnNotReady` (`0xe00002d8`). The getter expected a net80211 RUN state and the experimental association was maintained in firmware.

**97: macOS obtained and published its own lease.** The extended BSS getter used six legacy rates from the live BE211 association. The query returned success and native WCL reported associated and link up. On channel 11, the unmodified macOS DHCP client sent Discover and Request through `en0`. Firmware reported success with zero retries on both, and the driver delivered Offer and ACK back to macOS. IPConfiguration entered BOUND, assigned a /22 IPv4 address, received the router's ARP reply, and published IPv4, DNS and DHCP state. Tcpdump independently observed the exchange. The full window reported 13 TX submissions, seven frames queued for BSD input, and an ACK forwarded.

Scheduled teardown removed the address. A native leave during cleanup coincided with a queue-removal and worker result of 6 after the packet window itself reported success. This is consistent with a teardown race. An automatic rejoin obtained another ACK and reached BOUND. The capture did not cover the entire second window. The driver still restricts the test data path to DHCP and ARP on an open 2.4 GHz network.

## 5. Current status and evidence limits

| Area | Verified outcome | Open question |
| --- | --- | --- |
| Boot and CPU | Recovery, Terminal and the installed desktop boot via OpenCore with CPUID emulation, classic firmware LAPIC entries and the Darwin CPU SSDT. | Long-run stability and native CPU power management. |
| Storage and installation | Internal NVMe initialization, an offline APFS installation and a bootable installed desktop. | General external device support and reproducibility across configurations. |
| Display | Basic output on the built-in panel and usable desktop. | Xe3/Metal acceleration, scaling behavior and graphics power usage. |
| Keyboard | Functional built-in PS/2 input. Skipping the PS/2 boot reset improved captured startup behavior. | Reliability across all restart paths. |
| Touchpad | Native PIXA3854 input with gestures and settings. | Long-run stability, sleep/wake and continued coexistence with PS/2 fallback. |
| External USB | Both xHCI controllers started in Recovery. A manually mounted ExFAT partition was usable for offline installation in a later run. | Missing USB 2 port publication, general hotplug and reliable external HID. |
| Wi-Fi | Native discovery, open 2.4 GHz association, macOS DHCP, and router ARP. | General IP traffic, actual DNS resolution, internet and captive portal access, durable joins, secured networks and other bands. |
| Other devices | Audio, Bluetooth, webcam and power features have no completed verification in this series. | Functional peripheral and power-management tests. The firmware coexistence command is separate from Bluetooth function. |

### Evidence and reproducibility

- Early evidence consists of OpenCore files, Linux ACPI and PCI captures, photographs, and traced entry and return markers. Screen captures can omit earlier lines. The absence of a return marker has lower evidentiary weight than a confirmed return code.
- Recovery terminal photos record the actual OS version, loaded drivers and registry state. The post-install collector captured early and late kernel logs and IORegistry. Service registration establishes attachment, while typing, pointer movement and gestures require functional observation.
- The investigation archives staged EFI files, changed-file hashes, OpenCore 1.0.8 validator output and file references where available. Later Wi-Fi tests also preserve driver source patches, build logs and captured output. A manifest match establishes the tested files when measured. Exact USB matches were unavailable for some early and follow-up runs.
- The DHCP findings distinguish firmware TX completion, packets visible on `en0`, and IPConfiguration publishing a lease. Attempt 97 provides evidence at all three boundaries. It does not establish a longer-lived connection or successful DNS queries.
