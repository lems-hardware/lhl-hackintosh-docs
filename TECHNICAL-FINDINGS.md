# Framework Laptop 13 Pro macOS Sequoia technical findings

**Coverage:** Boot attempts 1-123, through November 20, 2026\
**Platform:** Framework Laptop 13 Pro, Intel Panther Lake  
**Evidence:** [Boot attempt journal](logs/fw13p-attempts-sequoia.md) and [visual archive](logs/fw13p-visual-sequoia/)

## Summary

I went from a boot with no visible kernel output to graphical Sequoia Recovery, then opened Terminal and installed macOS Sequoia 15.8.3 offline. The installed desktop runs with basic display output, a built-in PS/2 keyboard, and a native I²C touchpad with gestures and configurable settings. Native touchpad stability has limited repeat testing.

I confirmed **Safari HTTPS over the Intel BE211** in Attempt 100 on an open 2.4 GHz network. In Attempt 101, I removed the scheduled disconnect and used Wi-Fi for 3–4 hours without an unexpected drop. It also reconnected automatically after I opened the lid the next morning. The current path supports IPv4 DNS and HTTPS over TCP. Secured networks and additional bands remain open work.

I also brought up onboard audio in Attempt 103. The ALC285 microphone, speakers, wired headphones, volume/mute controls and automatic jack switching work, including across restarts. Extended audio stability and sleep/wake remain unverified.

I subsequently enabled the missing USB 2 ports, verified the external mouse and reduced receiver reconnect delay. Attempt 109 also published battery data and confirmed charger transitions. Sleep/wake, CPU power efficiency and wider USB qualification remain open.

I then loaded a separate Panther Lake xe core and passed bounded stages 1-3 on B082. Display-register and page-table comparisons stayed consistent through PCI bus setup, with a progressing frame counter. This is a kernel bring-up milestone, not graphics acceleration.

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

I started with removable EFI changes, firmware settings, and read-only diagnostics. Later, I tested an offline installation on a dedicated internal APFS partition. I used separate kernel collections to analyze Recovery and the installed system. Symbols, addresses, and binary patches in this report refer to their specific build.

The [attempt journal](logs/fw13p-attempts-sequoia.md) provides the chronological account. Some early EFI folders originally had separate Test numbers. In particular, the shared xHCI start trace called Test 17 in early notes corresponds to journal Attempt 25. I recorded reaching Terminal in the journal under Attempt 40. I later confirmed that I had booted the Attempt 37 EFI.

## 1. UEFI-to-kernel transition and CPU compatibility

### Attempts 1-12 - The last OpenCore line was ambiguous

Attempts 1-10 repeatedly ended in the OpenCore file log at `EXITBS:START`. I tested changes to MMIO and virtual-map quirks, write protection and runtime permissions, CPUID emulation, CFG Lock and extra MSR workarounds, `ForceExitBootServices`, and a DMAR deletion. The patch and table-deletion logs showed that OpenCore applied several changes, with no corresponding visible boot progress. The elapsed times in the journal came from OpenCore logs. The time spent waiting after the last visible line was generally unavailable.

For Attempt 11, I disabled VT-d in firmware. The before and after Linux ACPI captures showed that the MADT changed from 16 enabled type-9 x2APIC records and a type-10 x2APIC NMI to 16 type-0 Local APIC records and a type-4 Local APIC NMI. The enabled CPU IDs and UIDs, IOAPIC, and two interrupt overrides were retained. Visible macOS kernel output appeared after this change, even though the OpenCore file still ended at `EXITBS:START`. The result supports an APIC compatibility explanation. Other effects of the firmware setting changed in the same test. The working firmware already supplies classic Local APIC records, so the tested EFI uses the firmware MADT.

The first visible kernel log ended near the AppleLockdownMode restoreOS parity check. I limited startup to one CPU with `cpus=1` in Attempt 12 and saw no further progress.

### Attempts 13-20 - Legacy CPU objects advanced startup

The Panther Lake DSDT described CPU devices as `ACPI0007` objects. In Attempt 13, I added `SSDT-FW-CPU.aml`, which exposes 16 legacy ACPI `Processor` objects for Darwin, using the enabled IDs in the actual MADT. The firmware CPU objects were preserved. The kernel listed IDs 0-15, completed PCI configuration, reached DSMOS, and loaded APFS. Attempt 14 removed `cpus=1` and retained this progress.

The internal NVMe driver logged assertions and subsequently reported successful initialization. Its media proceeded to partition probing. I temporarily masked the NVMe match in Attempt 15 and saw no significant boot progress. Attempts 16-20 tested USB ownership, increased IOKit logging, and adapted DebugEnhancer to keep useful startup messages visible. The recurring late NVMe and power-management lines provided evidence that other kernel work continued while USB startup was still pending.

## 2. xHCI startup dependency and Recovery boot

### Attempts 21-33 - An indefinite resource wait

I used the exact Recovery kernel collection to locate the function symbols and instruction sequences for the diagnostic hooks. The tracing followed the existing calls and recorded entry and return values. I found the shared `AppleUSBXHCI::start` entry at unslid address `0xffffff8002dce5e0` in the archived Recovery image.

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

I added VoodooPS2 in Attempt 34. After adding the firmware's explicit keyboard and mouse hardware IDs in Attempt 35, I reached the Russian-language Recovery Utilities menu. Recovery continued to alternate between Utilities and the input-detection prompt in later variants. I changed `prev-lang:kbd` from `ru-RU:252` to `en-US:0` and reached English Recovery in Attempt 37. I later used keyboard shortcuts to open Terminal with the same EFI. Terminal photographs show Sequoia 15.4.1 build `24E263`. Input behavior varied across boots. Neither the Logitech M330 receiver nor the external keyboard established a working USB input path in Recovery.

Attempt 41 exposed a separate problem after xHCI startup. Both controllers had drivers attached and successful startup traces. The IOUSB plane contained only its root and `diskutil` listed no external physical disks. The filtered IOService tree showed four SuperSpeed port objects under TXHC, no USB 2 objects, and no visible XHCI port objects. I read the following Supported Protocol ranges from the same hardware under Linux.

| Controller | Linux USB 2 hardware ports | Linux SuperSpeed hardware ports | Recovery query |
| --- | --- | --- | --- |
| XHCI `00:14.0` | 1-10 | 11-12 | No ports shown in the filtered query |
| TXHC `00:0d.0` | 1 | 2-5 | Four SuperSpeed port objects |

Both controllers logged `AppleUSBXHCI::createPorts: speed index 4 has inconsistent speed IDs (5 and 6)`. Read-only disassembly of the Recovery binary showed that the warning path continues capability parsing. The warning is consistent with the Linux register dump and does not independently explain missing USB 2 ports.

Under Linux, I traced the boot stick to TXHC bus 2, hub port 2, which maps to hardware port 3 (`SS02`), with a negotiated rate of 5000 Mb/s. A read-only PORTSC snapshot showed that port connected, powered and enabled in U0 at speed ID 4. The macOS port status and event delivery path remain unverified. The internal NVMe model in the kernel output is distinct from this USB stick.

## 3. Input, offline installation, and I²C interrupt routing

### Attempts 42-54 - Separate input paths and a usable desktop

Firmware exposes the built-in keyboard as `PS2K` with hardware ID `MSFT0001`, compatible ID `PNP0303`, and IRQ 1. The EC's basic pointing fallback appears as `PS2M` with hardware ID `FRMW0002`, compatible ID `PNP0F03`, and IRQ 12 when enabled. The native PixArt `PIXA3854` touchpad is the HID-over-I²C `TPAD` child of `I2C5`, compatible with `PNP0C50`. Linux also independently identified the Keychron K3 Max on PCI xHCI as a USB 2 device.

The first I²C builds regressed to the `macsetup` screen or stalled earlier. Removing the I2C5 PCI match, running helper kexts alone, and adding the polling flag alone did not isolate a single cause. The quieter Attempt 43 baseline reached Terminal and showed the native controller and touchpad in IORegistry. The EFI was reused in Attempt 48. I confirmed basic PS/2 touchpad movement on a cold boot with mouse emulation set to Auto. A restart from Recovery returned to input detection. The same EFI therefore produced different outcomes across boot paths.

In Attempt 49, I installed **Sequoia 15.8.3 offline**. I obtained the full `InstallAssistant.pkg` with gibMacOS and prepared it with UnPlugged in Recovery. I booted from the removable OpenCore EFI, manually mounted the ExFAT installer-data partition, and used a dedicated internal APFS target of roughly 150 GiB. I completed setup and opened the installed desktop and System Settings. This install pathway provided local persistent captures for diagnosing later input failures.

On the desktop, the Attempt 43 baseline showed an initial PS/2 stream 1 timeout in 11 early snapshots. Three also showed a second timeout followed by mouse probe failure. In Attempt 54, I added `ps2rst=2` to skip the boot reset. Five new early captures showed keyboard and pointing HID registration without a recorded stream 1 timeout. Initial tests showed typing and basic touchpad input working across several boot methods. A later Linux KDE to macOS restart still failed input initialization. I also tested with the USB mouse receiver connected. The mouse never responded.

### Attempts 50-59 - Dynamic HPET allocation blocked I2C5

I matched VoodooI2C to the exact controller `8086:e451` at `00:19.1`. Correcting the controller reference clock from 133 MHz to the 100 MHz value used by Linux let VoodooI2C recognize the Synopsys controller and finish bus timing setup. It repeatedly failed PCI interrupt registration with empty interrupt arrays. The touch-report polling option could only take effect after controller initialization.

| Attempt | Evidence |
| --- | --- |
| 56 | IOPCIFamily read INTB. AppleACPIPlatform resolved it to GSI 21, then `SetDeviceInterrupts` returned `0xe00002df` (`kIOReturnNoInterrupt`). The outer resolver reported success. |
| 57 | The allocator rejected the GSI 21 request with flags `0x7` before interrupt-controller lookup or publication. |
| 58 | AppleHPET had already claimed GSI 21 as non-shared, using flags `0x0`. The second request produced the logged allocation conflict. |
| 59 | A build-specific OpenCore patch cleared bit 21 in AppleHPET's local timer route mask. HPET used GSI 20, 22, 11 and 23. I2C5 then allocated, published and registered GSI 21 successfully. |

The HPET patch targets `configureTimer` in the installed `24H241` AppleHPET binary. The archived 95-byte pattern was unique in the verified collection and began at unslid address `0xffffff8001946374`. The patch preserves the firmware's I2C5 route. The controller uses GSI 21 for bus transactions, while VoodooI2CHID polls the touchpad for reports. VoodooInput and Apple's multitouch driver attached. I verified gestures, scrolling, two-finger secondary click and Trackpad settings. The touchpad also worked after a KDE-to-macOS restart and a macOS-to-macOS restart during follow-up Wi-Fi testing. These results are functional evidence with limited repeat coverage.

The removable OPENCORE volume also remounted read-only during several early and I²C tests. Linux reported FAT directory corruption while the block device reported write protection off. The read-only check showed damaged Spotlight and Trash entries, shared clusters and a dirty bit. The active EFI still matched the staged manifest in several affected runs. I backed up and repaired the FAT volume. The evidence establishes a filesystem failure alongside the boot investigation. Its timing and relation to individual tests remain undetermined.

## 4. Intel BE211 Wi-Fi bring-up

### Attempts 60-72 - Firmware startup, scanning, and the native interface

I based the experimental AirportItlwm build on the BE200 fork's integrated BZ transport and added the BE211 SC A0 / WH B0 match. The firmware image was Intel `iwlwifi-sc-a0-wh-b0-c106.ucode`, corresponding to Linux's working Core 106 image. The source baseline was `FIX31-20261109` at `ff82f3e5647ce3885f5d48417b6356a6a23d2163`. I cross-built the kext for Sequoia on Linux using Clang, cctools ld64 and MacKernelSDK commit `3f750085caa17ec3a7880f11c11bf4f48cd6a164`.

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

**95-96: macOS recognized the join and immediately requested leave.** I moved the join into a worker so the menu request could return promptly. The driver posted actual peer metadata, a first-beacon event, and successful ConnectComplete. Native WCL logged Joined Bss. The packet window nonetheless closed within about a millisecond on Attempt 95. Attempt 96 traced a native WCL leave request about 43 microseconds after the extended BSS getter returned `kIOReturnNotReady` (`0xe00002d8`). The getter expected a net80211 RUN state and the experimental association was maintained in firmware.

**97: macOS obtained and published its own lease.** I updated the extended BSS getter to use six legacy rates from the live BE211 association. The query returned success and native WCL reported associated and link up. On channel 11, the unmodified macOS DHCP client sent Discover and Request through `en0`. Firmware reported success with zero retries on both, and the driver delivered Offer and ACK back to macOS. IPConfiguration entered BOUND, assigned a /22 IPv4 address, received the router's ARP reply, and published IPv4, DNS and DHCP state. Tcpdump independently observed the exchange. The full window reported 13 TX submissions, seven frames queued for BSD input, and an ACK forwarded.

Scheduled teardown removed the address. A native leave during cleanup coincided with a queue-removal and worker result of 6 after the packet window itself reported success. This is consistent with a teardown race. An automatic rejoin obtained another ACK and reached BOUND. The capture did not cover the entire second window. That build limited the test data path to DHCP and ARP on an open 2.4 GHz network.

### Attempts 98-101 - Safari and sustained Wi-Fi

**98-99.** I first added a gateway ping after native DHCP. Firmware completed three ICMP Echo Requests with zero retries, and the gateway answered ARP. No echo replies reached macOS. I then added IPv4 DNS and HTTPS over TCP. The capture showed DNS replies and acknowledged payload in both directions on port 443. Background macOS traffic exhausted the 76 web transmit slots, and the driver counted 279 drops at that limit. Safari reported no internet connection. The capture did not establish the cause of the browser error.

**100.** I replaced the single-use transmit slots with a 128-slot ring. The driver reclaimed descriptors only when firmware reported a scheduler position within the outstanding range. I loaded `example.com` in Safari on CISD-Guest. The capture showed matching DNS replies and two-way TCP payload with one of the returned addresses on port 443. Over 45 seconds, the ring wrapped 261 times and reclaimed all 33,444 submitted descriptors. Peak occupancy was 32, with zero ring-full events or ignored completions. Firmware reported 27 failed completions. The scheduled teardown still coincided with the late WCL leave and queue-removal result 6.

**101.** I removed the scheduled disconnect and took the SSID from the macOS join request. The driver still checks for an eligible open 2.4 GHz network and honors a valid requested BSSID. I used Wi-Fi for 3–4 hours without an unexpected disconnect. Wi-Fi reconnected automatically after I reopened the lid the next morning. I did not record the network name, the exact sleep state or the reconnect latency.

## 5. Onboard audio - Attempts 102-103

Linux identified a Realtek ALC285 codec (`10ec:0285`, subsystem `f111:000f`) behind the Panther Lake audio controller `8086:e428` at `00:1f.3`, ACPI `\_SB.PC00.HDAS`. Speakers use codec pin `0x17` with DAC `0x03`, headphones use `0x21` with DAC `0x02`, and the internal microphone uses pin `0x12`. I selected AppleALC layout 66 because its output routes match this wiring.

**102.** I added official AppleALC 1.9.8 DEBUG and `alcid=66 -alcdbg` to the Attempt 101 EFI. AppleALC loaded, renamed HDAS to HDEF and selected the requested layout. AppleHDAController 600.2 reached its probe but returned failure, with no attached codec or audio device. Its generic PCI class match included the hardware class, so the failure occurred after matching. AppleALC logged no controller-specific modification for `8086:e428`. This was earlier than codec routing.

**103.** Following [Mubai's Panther Lake audio investigation](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/blob/5f0de92f42b3655d6539b91fe3a630b8262096f3/HARDWARE_FOLLOWUP_2026-11-13.md), I added only `device-id = ca510000` at `PciRoot(0x0)/Pci(0x1F,0x3)`, presenting compatibility ID `8086:51ca`. I retained layout 66 and all driver binaries. AppleALC selected its existing Raptor Lake controller modification and logged patch application. AppleHDAController attached, discovered the ALC285 codec and started AppleHDADriver. The mounted EFI matched all 60 staged files.

The cold-boot inventory listed Built-in Microphone, Built-in Line Input and Built-in Output using Internal Speakers. I confirmed microphone operation, audible playback through speakers and wired headphones, volume and mute controls, automatic jack switching, and continued operation across restarts. These functional results are my direct hardware observations. Line-input functionality was not separately tested. Extended stability and sleep/wake remain unverified.

The initial Attempt 102 driver copy reported filesystem metadata errors. Attempt 103 archived the installed files and kernel collection directly, with an empty archive error log. The saved Attempt 103 evidence was a partial collector folder without its completion marker or final outer archive. Restart and functional follow-ups were reported separately, without additional captures. SimpleHDA was not used on this Sequoia system. No new manual AppleHDA binary patch was added.

## 6. Installed-system USB ports and reconnect timing

### Attempts 104-105 - Port descriptions blocked USB 2 publication

I inspected the installed 24H241 kernel collection separately from the earlier Recovery image. Read-only tracing showed correct protocol discovery and attempted creation of all hardware ports. AppleUSBXHCIPCI returned null for all 12 XHCI ports and TXHC USB 2 port 1, while TXHC SuperSpeed ports 2-5 returned objects. Both createPorts functions returned zero despite the missing ports. The firmware RHUB descriptions exposed only USR1/USR2 under XHCI and the four SuperSpeed children under TXHC.

In Attempt 105, official USBToolBox 1.2.0 bypassed the runtime ACPI child descriptions and supplied an explicit map: XHCI hardware ports 1-10 as HS01-HS10 and 11-12 as SS01-SS02, with TXHC hardware port 1 as HS01 and 2-5 as SS01-SS04. All 17 port-creation calls then returned objects. This combined experiment fixes the observed failure but does not isolate the separate contribution of ACPI bypass and map injection.

The Logitech 046d:c542 receiver worked on top-right XHCI HS02 at 12 Mb/s, including buttons, scrolling and reconnect. The internal webcam enumerated on HS06 in both USB and Camera inventories, and Goodix enumerated on HS07. Those inventories do not establish working video or fingerprint authentication. The SD Expansion Card remained at 5 Gb/s with FILES writable. Connector types are still provisional 255 values, and other physical ports, Type-C orientation and sleep/wake are unqualified.

### Attempts 106-108 - Avoiding the slow reset retry sequence

Attempt 106 traced connection handling, resetAndCreateDevice and enumeration completion. Slow cycles returned null six times at increasing arguments 50-300, then succeeded at 350, with approximately one-second waits between attempts. Connection-handler entry to HID startup took about 8.2 seconds. Once enumeration succeeded, HID startup took only about 25-32 ms. One trial reached HID startup quickly despite a later manual pointer-response mark, so reset retries did not explain every perceived delay.

The exact installed binary converts the unsigned argument to a millisecond-scale deadline for its reset loop. Attempt 107 substituted a minimum of 350 only on HS02 at location 14200000. The original function still runs once and determines its result. This affects any USB 2 device on that one mapped port, not just Logitech. The three reconnects reached HID startup in 1.95, 0.50 and 0.55 seconds. The first still needed one retry.

Attempt 108 retained the policy and observed existing host createDevice/getPortStatus calls without adding status queries. All six prompted reconnects succeeded on the first reset call and reached HID startup in 0.52-0.54 seconds. Human insertion-to-pointer intervals were 0.99-1.55 seconds, including reaction time. Captured successful creation used status 0x1102. Disconnected queries returned 0x2. The host trace reached its 128-call limit before the sixth insertion, while the separate reset/enumeration trace continued and confirmed success. The intermittent failure did not recur, so its exact cause remains unresolved. No broader stability or universal timing guarantee follows from this run.

## 7. Battery reporting and charger detection

### Attempt 109 - Matching SMCBatteryManager published the battery

The preceding macOS registry already exposed BAT1 (PNP0C0A) and ACAD (ACPI0003), but SMCBatteryManager was absent. The archived firmware provides _BIF, _BIX, _BST and _PSR through a ByteAcc SystemIO region. This did not justify treating its wider declared fields as an EmbeddedControl access problem. I retained the working EC stub and all other ACPI tables.

Adding official SMCBatteryManager 1.3.8 after matching VirtualSMC 1.3.8 found one battery and one adapter, registered the plugin and enabled AppleSmartBatteryManager. In the cold-boot capture, pmset reported 30% charging on AC, 32% discharging after unplugging, then 33% charging after reconnecting. ExternalConnected and IsCharging changed correctly. The battery service reported 4640 mAh design capacity, 4514 mAh full capacity, 42 cycles and voltage/current data. System Information reported Normal condition. These populated values are not a calibration or controlled power-consumption measurement.

I withdrew my initial report of frozen status after allowing for periodic updates. The matching source uses 1-second quick and 60-second normal polling plus notifications, so this does not prove a fixed 30-second interval. All saved phases show the appropriate power state. Other functions were reported working. Restart behavior, long-run accuracy, charge-limit/full-charge boundaries, lid events, sleep/wake and CPU power efficiency remain unverified. The collector made no persistent power-setting changes.

See the [measurement and evidence summary](logs/usb-power-evidence.md) for capture digests, trial timings and scope limits.

## 8. Display power and Xe3 core bring-up

### Attempts 110-111 - Lid notifications and the boot framebuffer

I captured menu-sleep and lid-close/open behavior without changing persistent power settings. The lid events were about 32.5 seconds apart. The kernel repeatedly rejected full sleep with kPMChildPreventSystemSleep and retained .Display_boot in the system-preventer list. The boot time stayed unchanged and the sleep/wake counter remained zero. DarkWake/FullWake labels therefore do not establish completed hardware sleep and resume. This confirms lid detection and identifies a display power-management lead, not a working sleep implementation.

The installed IONDRVSupport and IOGraphicsFamily modules were both version 599. The initial collector skipped the 373112832-byte system collection because its cap was 256 MiB. That collection was captured separately in Attempt 113. I did not clear the sleep-preventer vote or patch display power behavior.

### Attempts 112-113 - MeowXE3 dependency resolution

I tried the MeowXE3 1.0.12 core with -nvoff, leaving out its display replacement, Metal plugin and installed-system deployment. OpenCore reported IOGraphicsFamily absent from the Boot Kernel Collection, used a symbol-only stub, failed vtable patching and rejected injection with Invalid Parameter. The core never loaded. This does not test its runtime disable gate or B082 initialization.

The exact installed SystemKernelExtensions.kc contains IOGraphicsFamily 599 with the loaded UUID. All 76 IOFramebuffer import symbols used by the release core are present, including the parent vtable and metaclass. This strengthens the dependency-timing explanation, but symbol presence does not prove vtable-slot ABI compatibility or identify the exact failed table. The release Metal plugin declares minimum macOS 26.0 and is not qualified for Sequoia. The release firmware inventory also does not establish support for this panel's display IP 30.00.

### Attempt 114 - A separate core loads at stage 0

I selected Mubai's pinned Panther Lake xe core, derived from the InSnh-Gd WTLIntelGfx fork. This binary has no IOFramebuffer imports or IOGraphicsFamily dependency. All 444 imported symbol names exist in the exact archived installed Boot Kernel Collection, although that alone is not an ABI test.

The corrected boot loaded the expected 0.0.47 UUID and XeGfxResident policy ptl-worker-heartbeat-v24. Stage and running stayed zero, with an empty environment and zero stage results. Desktop and input worked. My initial submission had booted the older Attempt 112 EFI and was retained as a copy mismatch. OpenCore's attempted Exclude blocker for the installed older 0.0.48 driver returned Unsupported. I did not remove that installed copy or claim the exclusion succeeded.

### Attempts 115-116 - Runtime patch application failed before probing

Attempt 115 could not resolve the two local-symbol Base anchors. Attempt 116 removed those anchors, but OpenCore still replaced zero occurrences of either exact byte signature. Both collectors refused the request before any metadata test ran. These are distinct attempts, not revisions. The original second capture is still named attempt-115-20261119-160603-671.tgz and is explicitly indexed as Attempt 116.

### Attempts 117-118 - Metadata baseline and PCI bus setup

I directly adapted two verified instruction bytes in the pinned core. One selects its existing metadata-only branch for B082 instead of B0A0. The other limits manual stage requests, first to 2 and then to 3. These changes do not spoof the GPU identity or import B0A0 initialization workarounds. Each variant has a distinct UUID and refreshed original ad-hoc code-page hashes. No installed driver or cache was changed. Automatic startup remained at stage 0.

Attempt 117 passed stages 1-2. The metadata comparison returned 1 and the frame counter advanced by 13. The framebuffer candidate covered 5400 pages and 22118400 bytes at 0x7c800000. CPU framebuffer access stayed blocked and pixel verification stayed false.

Attempt 118 repeated that prerequisite successfully, then passed stage 3. vgapci0 attached to 8086:b082 and its drmn0 child remained unbound. The post-bus metadata comparison returned 1 with unchanged metadata and a frame-counter advance of 12. All three stage results were 1, the core ended idle at stage 3 and I reported everything operational. The existing 2880 x 1920 framebuffer stayed active.

Stage 3 initializes PCI/DMA shim state and the newbus wrapper. GPU module SYSINIT registration belongs to stage 4. The DMA helper logged no IOMMU, consistent with disabled VT-d. It also called B082 older than gen11 because its generic stolen-memory classification tables lack that ID. That fallback string does not identify the hardware generation. Whether the later xe path needs those skipped generic globals must be checked before changing that classification or dismissing it.

These captures do not establish GuC/other GPU firmware initialization, engine completion, pixel correctness, Metal, accelerated WindowServer or native display takeover. Working display metadata and a published resident service are narrower results. Research into MeowXE3, Mubai's port and the same-model virtualized VF work informs the approach, but another machine's acceleration is not a result on this Framework.

See the [graphics evidence summary](logs/graphics-evidence.md) for exact core identities, original capture digests, provenance and test boundaries.

## 9. Current status and evidence limits

| Area | Verified outcome | Open question |
| --- | --- | --- |
| Boot and CPU | Recovery, Terminal and the installed desktop boot via OpenCore with CPUID emulation, classic firmware LAPIC entries and the Darwin CPU SSDT. | Long-run stability and native CPU power management. |
| Storage and installation | Internal NVMe initialization, an offline APFS installation and a bootable installed desktop. | Broader external device support and installation reproducibility across configurations. |
| Display | Basic desktop output, resident xe core load and successful metadata/PCI bus stages 1-3. | Stage-4 initialization, GPU firmware, rendering, Xe3/Metal acceleration and graphics power usage. |
| Keyboard | Functional built-in PS/2 input. Skipping the PS/2 boot reset improved captured startup behavior. | Reliability across all restart paths. |
| Touchpad | Native PIXA3854 input with gestures and settings. | Long-run stability, sleep/wake and continued coexistence with PS/2 fallback. |
| External USB | All 17 mapped ports publish, the tested Logitech mouse works, and the SD reader remains accessible at 5 Gb/s. Six latest same-port reconnects reached HID startup in 0.52-0.54 s. | Provisional connector types, other ports/orientations, occasional reset retry and sleep/wake. |
| Wi-Fi | Native open 2.4 GHz joins, DHCP, DNS and Safari HTTPS. One 3–4-hour run and one automatic reconnect after an overnight lid close. | Repeatability across access points and sleep states, broader traffic support, captive portals, secured networks and other bands. |
| Audio | ALC285 microphone, speakers, wired headphones, volume/mute and automatic jack switching, including across restarts. | Extended stability, sleep/wake and separately tested line input. |
| Battery | Battery data publication and charger disconnect/reconnect confirmed in a cold boot with SMCBatteryManager 1.3.8. | Restart behavior, accuracy, charge limits and long-run stability. |
| Other devices and power | Webcam and fingerprint device enumerate. Lid-close/open events are detected. | Actual video, Bluetooth, fingerprint function, full sleep/resume and CPU power efficiency. The boot framebuffer currently prevents full sleep. Firmware coexistence success does not establish Bluetooth function. |

### Evidence and reproducibility

- I collected early evidence from OpenCore files, Linux ACPI and PCI captures, photographs, and traced entry and return markers. Screen captures can omit earlier lines. The absence of a return marker has lower evidentiary weight than a confirmed return code.
- Recovery terminal photos record the actual OS version, loaded drivers and registry state. The post-install collector captured early and late kernel logs and IORegistry. Service registration establishes attachment, while typing, pointer movement and gestures require functional observation.
- I archived staged EFI files, changed-file hashes, OpenCore 1.0.8 validator output and file references where available. For later Wi-Fi tests, I also saved driver source patches, build logs and captured output. A manifest match establishes the tested files when measured. Exact USB matches were unavailable for some early and follow-up runs.
- Attempt 97 verified DHCP at three points, from firmware TX completion through packets visible on `en0` to IPConfiguration publishing the lease. Attempt 100 pairs the Safari page load with DNS and TCP capture. My Attempt 101 duration and reconnect findings come from direct use without a collector trace.

## Credits and references

OpenCore, Lilu, VirtualSMC, VoodooPS2 and AppleALC are [Acidanthera projects](https://github.com/acidanthera). Native touchpad support builds on [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) and [VoodooInput](https://github.com/acidanthera/VoodooInput). The wireless foundation is [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) and [Ersiter's BE200 fork](https://github.com/Ersiter/AirportItlwm-BE200), with Intel firmware and the original component notices retained.

[Mubai's hardware follow-up](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/blob/5f0de92f42b3655d6539b91fe3a630b8262096f3/HARDWARE_FOLLOWUP_2026-11-13.md) supplied the controller compatibility-ID lead for Attempt 103. AppleALC's [controller definitions](https://github.com/acidanthera/AppleALC/blob/1.9.8/Resources/Controllers.plist) and [ALC285 resources](https://github.com/acidanthera/AppleALC/blob/1.9.8/Resources/ALC285/Info.plist) establish the upstream patch and layout provenance. Layout 66 is credited there to R-a-s-c-a-l. My Framework-specific outcomes and evidence limits are recorded above.

[USBToolBox 1.2.0](https://github.com/USBToolBox/kext/tree/1.2.0), by Dhinak G. and contributors, provides the runtime ACPI bypass and port-map injection used in Attempt 105. lvs1974 and DebugEnhancer contributors supplied the tracing foundation. usrsse2 and [VirtualSMC contributors](https://github.com/acidanthera/VirtualSMC/tree/1.3.8) supplied SMCBatteryManager. My diagnostic routes and scoped reset-window experiment were derived from the exact installed binary.

Installer retrieval and Linux cross-builds used [gibMacOS](https://github.com/corpnewt/gibMacOS), [MacKernelSDK](https://github.com/acidanthera/MacKernelSDK), LLVM/Clang and [cctools-port](https://github.com/tpoechtrager/cctools-port). Apple and all upstream contributors retain credit and rights for their components.

Graphics credits: [watchthelight / WTLIntelGfx](https://github.com/watchthelight-labs/WTLIntelGfx), [InSnh-Gd's Panther Lake fork](https://github.com/InSnh-Gd/WTLIntelGfx/tree/5ffc79fb9e53eb820899edd644b83681bea3aa39) and [Mubai's pinned work](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/tree/180a42fafd1e1b8116bdc1f0312dec54be3812c5) provide the core and metadata diagnostics. [cuadra8998 / MeowXE3](https://github.com/cuadra8998/MeowXE3) and [ericek111's Xe3 VF work](https://github.com/ericek111/vibe-nullmoth-xe3-qemu) provide comparison research. NullMoth, HighDelay, Intel, Linux DRM/xe, FreeBSD drm-kmod/LinuxKPI, Mesa, libdrm, LLVM and linux-firmware contributors retain credit and their original licenses. The VF work uses a Linux-owned physical function and virtualized scanout, which differs from this physical panel path.

## Graphics follow-up through Attempt 123

I initialized both Xe3 GTs with GuC 70.72.1 after fixing a scoped SRCU compatibility mismatch. All six engines then completed empty batches and produced matching GPU memory writes and CPU readbacks. A root macOS process opened the render node, queried xe 1.1.0 and closed cleanly. The existing display and input remained usable in these captures. Userspace GPU allocation, mapping, command submission, rendering, Vulkan, Metal and extended graphics stability remain unverified.

Attempt 119 panicked in queue/LRC cleanup when a GGTT invalidation waited on a fence with preemption disabled. Exact-binary analysis identified the DRM unplug SRCU reader alias to ordinary RCU. Attempt 120 kept the same core and routed only that domain through sleepable readers and serialized grace periods. Initialization then completed. This supports the diagnosis without qualifying every SRCU or hot-unplug path.

Attempt 121 completed empty batches on six engines with zero errors and 48 interrupts. Attempt 122 verified six distinct GPU-written values with matching CPU readbacks, zero errors and 49 store-test interrupts. Attempt 123 repeated stages 1-6, then opened /dev/dri/renderD128 and completed DRM_IOCTL_VERSION and close with zero errors. It ran as root. The initial missing-helper capture stopped at stage 0 and is not a driver failure. The follow-up found IOAcceleratorFamily2 485.2 and IOGraphicsFamily 599, with no active accelerator entries in the queried classes.

Stage 7 remains blocked. Its claim property alone would not establish acceleration. The test writes targeted diagnostic buffers, while panel-framebuffer CPU access remained blocked. The final display metadata in Attempt 123 was the saved stage-6 result, not a new physical resampling after the version query. No shader, rendered image, ANV/Vulkan or Metal result is claimed.

The xe core and port credit remains with watchthelight, InSnh-Gd, Mubai, Linux/Intel xe and FreeBSD LinuxKPI. The local SRCU compatibility route uses the Acidanthera/Lilu and DebugEnhancer foundations. Raw captures, Apple kernel collections and machine-specific registry data remain private.
