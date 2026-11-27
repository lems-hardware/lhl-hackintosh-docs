# Framework Laptop 13 Pro macOS Sequoia technical findings

**Coverage:** Boot attempts 1-185, through November 26, 2026\
**Platform:** Framework Laptop 13 Pro, Intel Panther Lake  
**Evidence:** [Boot attempt journal](logs/fw13p-attempts-sequoia.md) and [visual archive](logs/fw13p-visual-sequoia/)

## Summary

I went from a boot with no visible kernel output to graphical Sequoia Recovery, then opened Terminal and installed macOS Sequoia 15.8.3 offline. The installed desktop runs with basic display output, a built-in PS/2 keyboard, and a native I²C touchpad with gestures and configurable settings. Native touchpad stability has limited repeat testing.

I confirmed **Safari HTTPS over the Intel BE211** in Attempt 100 on an open 2.4 GHz network. In Attempt 101, I removed the scheduled disconnect and used Wi-Fi for 3–4 hours without an unexpected drop. It also reconnected automatically after I opened the lid the next morning. The current path supports IPv4 DNS and HTTPS over TCP. Secured networks and additional bands remain open work.

I also brought up onboard audio in Attempt 103. The ALC285 microphone, speakers, wired headphones, volume/mute controls and automatic jack switching work, including across restarts. Extended audio stability and sleep/wake remain unverified.

I subsequently enabled the missing USB 2 ports, verified the external mouse and reduced receiver reconnect delay. Attempt 109 also published battery data and confirmed charger transitions. Sleep/wake, CPU power efficiency and wider USB qualification remain open.

I then initialized both GTs on B082, completed empty batches on all six engines and verified GPU-written values through CPU readback. Root userspace tests allocated and mapped memory, bound a GPU VM and verified an immediate GPU store with intact guards and successful cleanup. After the early OpenCore accelerator injection failed, normal macOS approval and an explicit start-only request successfully loaded the installed accelerator. Manual stage 7 then attached NMXAccel. Private Intel Vulkan libraries running as my normal console user completed a buffer fill, an application compute shader and a 64x64 offscreen vertex/fragment draw with exact readback. I subsequently built and loaded a private Metal plugin, verified Metal transfer, compute and offscreen rendering, and implemented the missing user-pointer mapping path for IOSurface host import. Attempt 168 checked all texture/surface pixels and final four-page cleanup. I subsequently verified a short 2 GiB cumulative randomized asynchronous workload and twelve visibly rendered CAMetalLayer frames through the explicitly loaded private device. Ordinary Metal discovery still returns no devices. Accelerated WindowServer, native display takeover and broader graphics stability remain unverified.

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

Stage 3 initializes PCI/DMA shim state and the newbus wrapper. GPU module SYSINIT registration belongs to stage 4. The DMA helper logged no IOMMU, consistent with disabled VT-d. It also called B082 older than gen11 because its generic stolen-memory classification tables lack that ID. That fallback string does not identify the hardware generation. The later exact-binary review found that xe uses its newer integrated GGC/DSM MMIO path. Attempt 120 then initialized successfully without patching the generic classification tables.

Attempts 117-118 alone did not establish GPU firmware initialization or engine completion. The later tests below establish GuC startup, completion and diagnostic memory writes, while rendered pixels, Metal, accelerated WindowServer and native display takeover remain unverified. Working display metadata and a published resident service are narrower results. Research into MeowXE3, Mubai's port and the same-model virtualized VF work informs the approach, but another machine's acceleration is not a result on this Framework.

See the [graphics evidence summary](logs/graphics-evidence.md) for exact core identities, original capture digests, provenance and test boundaries.

### Attempts 119-120 - A sleeping SRCU reader blocked initialization

Attempt 119 passed the prerequisites through stage 3, then froze during stage-4 initialization. The recovered panic matched the exact core UUID and reported thread_invoke: preemption_level 1. Its stack led through asynchronous queue/LRC cleanup, GGTT removal, TLB invalidation and a blocking fence wait. The panic occurred roughly two seconds after the request, not at the 60-second stage watchdog limit.

The subsequent exact-binary review identified drm_dev_enter holding drm_unplug_srcu across that wait. The compatibility layer aliased SRCU to ordinary RCU, which disabled preemption. SRCU permits sleeping readers. I added a Lilu/DebugEnhancer route for that DRM domain only, retaining reader protection and serialized grace periods without pinning preemption across the reader body. Other domains kept their original implementation.

Attempt 120 retained the same core and passed stages 1-4 with the scoped route installed. The cleanup path completed, drmn bound, and both GTs reported GuC 70.72.1 running with submission enabled. The driver enumerated six engines and enabled one MSI vector. Normal PC startup returned without a diagnostic bypass. Display metadata stayed unchanged, the frame counter advanced by 12 and I reported everything functional. This supports the SRCU diagnosis without qualifying every concurrency or hot-unplug path.

### Attempts 121-122 - Engine completion and GPU memory readback

I raised the manual ceiling to stage 5 in Attempt 121 while retaining the same SRCU implementation. All six engines completed an empty batch with matching submitted/completed sequence numbers, zero errors and 48 interrupts. The engines were render, two copy engines, compute, video decode and video enhancement. Their names identify engine classes, not tested rendering or media functionality.

Attempt 122 raised the ceiling to stage 6. Each engine wrote a distinct expected value to a separate diagnostic allocation, then the CPU read it back after a successful fence. All six values matched, every error was zero and the driver recorded 49 interrupts. Stages 1-6 returned 1. The post-test display metadata stayed unchanged and the frame counter advanced by 12. I reported all working. Panel framebuffer CPU access remained blocked and its pixels were not verified. Diagnostic timings are not performance benchmarks.

### Attempt 123 - A macOS process queries the xe driver

I reused the exact Attempt 122 EFI. The first collector stopped at stage 0 because its helper was missing. After copying the existing helper beside the script, I repeated the same test. Stages 1-6 passed again, with 47 completion-test interrupts and 44 store-test interrupts. A root macOS process opened /dev/dri/renderD128, completed DRM_IOCTL_VERSION, received xe 1.1.0 with the description Intel Xe Graphics and closed the node successfully. The helper exited zero and I reported all functioning.

The limited inventory recorded IOAcceleratorFamily2 485.2 and IOGraphicsFamily 599, but no active accelerator in the queried classes or custom bundles in the scanned paths. The final display properties retained the successful stage-6 comparison. They were not a fresh hardware comparison after the version query. This establishes root userspace descriptor, ioctl and close access, not ordinary-user or sandbox access, GPU allocation, mapping, VM binding or userspace submission.

### Attempts 124-128 - Userspace GPU execution and verified output

I kept the exact Attempt 122 EFI for these tests. Attempt 124 created and CPU-mapped a 65536-byte native system-memory WB buffer, checked two patterns across all 16384 words with zero mismatches and explicitly cleaned up. Attempt 125 added a normal VM and a fresh binary synchronization object, with successful creation and destruction. Attempt 126 bound the buffer at GPU VA 0x1000000 using PAT index 2, observed the bind completion signal, then observed a separate unbind signal and cleaned up successfully. The PAT selection and ioctl layouts were derived from the exact core and matching xe definitions.

Attempt 127 selected render class 0, instance 0, GT 0, created a width-1 execution queue and submitted one bound MI_BATCH_BUFFER_END. The execution completion signaled on the second poll, the queue remained unbanned and cleanup succeeded. An empty batch established reported completion, not a checked data effect.

Attempt 128 replaced that batch with MI_STORE_DATA_IMM using the private VM address space. The destination at GPU VA 0x1000400 began as 0xdeadbeef. CPU readback after execution returned exactly 0x1280cafe, both adjacent guards stayed zero and all five batch dwords were unchanged. The queue stayed unbanned, unbind completion arrived and every explicit cleanup call returned zero. I reported all working. This establishes one userspace GPU-produced data result through the mapped buffer, beyond completion signaling alone. It is an immediate command, not a shader or rendered image. The final display properties retain the preceding stage-6 comparison rather than a new hardware check after userspace cleanup.

### Attempt 129 - Sequoia accelerator prelink failure

I built XeGfxAccel from the pinned WTL source with a macOS 15.0 compile target and without its Tahoe-specific forwarding macro. Exact installed family symbols exposed four integer aliases that needed named enum types, eliminating seven initially missing method names. I also stopped the shared NVIDIA temperature timer from being scheduled on Intel. Offline checks found all 892 final import names in the saved collections, but 484 were SystemKC-only and 27 had only local definitions. Sixteen inherited-vtable comparisons had no unexplained differences. Those checks did not prove early loader visibility or full runtime object-layout compatibility.

The separate EFI restored the core's original manual stage-7 ceiling with a distinct UUID, kept automatic startup at stage 0 and retained the same sleepable SRCU algorithm. The accelerator personality required B082 and the claim property, with no Metal plugin advertised or included. The collector required the exact accelerator UUID before initializing the GPU.

OpenCore reported IOAcceleratorFamily2 and IOGraphicsFamily missing from BootKC, substituted symbol-only kernel stubs, failed XeGfxAccel vtable patching and rejected its injection with Invalid Parameter. The adapted core and SRCU hooks loaded successfully. The collector refused the test, leaving stage=0, running=0, an empty environment and eight zero stage results. Neither queried accelerator class had a service. I reported all working. This confirms the early dependency-resolution boundary seen previously with MeowXE3, not an accelerator-start or GPU-initialization failure. The exact failing vtable remains unidentified. Stage 7 was not requested.

After Attempt 129, the graphics integration boundary was a supported dependency/loading route before the accelerator could attach. Exact disassembly shows that stage 7 publishes a claim and reruns matching, which alone would not prove rendering. Later attachment, normal console-user access and Vulkan shader results are recorded below. Sandbox access, Metal, accelerated WindowServer, native display takeover and extended graphics stability remain unverified.

### Attempts 130-138 - Standalone SRCU and accelerator collection building

I separated the DRM SRCU compatibility route into XeGfxSRCUFix 0.1.0. With the working stage-6 core, Attempt 130 repeated all six engine completions and GPU stores, and verified the private userspace store of 0x1280cafe with intact guards and successful cleanup.

Attempts 131-132 failed collection building before a candidate was produced. Attempt 133 identified invalid FAT32 ownership on both accelerator and control bundles. Attempt 134 stopped at an invalid disk-image option. Attempt 135 corrected it using a temporary HFS+ image, verified root ownership and strict signing checks, but both bundles still failed the separate authentication diagnostic. This did not establish corrupted Apple collections or an accelerator-specific ABI fault.

Attempt 136 successfully built and inspected an uninstalled auxiliary collection containing XeGfxAccel 0.9. I verified the exact installed BootKC and SystemKC linkage in Attempt 137 and saved a verified copy of the original auxiliary collection. Attempt 138 built a new candidate containing AppleMobileDevice 4.0, HighPointIOP 4.4.5, HighPointRR 4.22.1, my original Stage 2 XeGfx 0.0.48 probe and XeGfxAccel 0.9. All expected executable UUIDs matched. Original source and installed collection hashes remained unchanged.

This establishes collection building and membership, not runtime loading or equivalent boot behavior. The next experiment will require a verified rollback route. Stage 7, accelerator attachment, rendering and Metal remain untested.

### Attempts 139-143 - Boot acceptance, bundle discovery and normal approval

I verified the original auxiliary collection backup from built-in Sequoia 15.8.3 Recovery, including a temporary copy and rename with matching root ownership, permissions and SHA256. Recovery ran the same functioning features as my regular installation during those checks. I did not qualify each subsystem individually.

In Attempt 139, macOS booted with the five-entry candidate and listed the expected accelerator UUID in the kernel auxiliary inventory, but the start-only identifier request returned code 71. Attempt 140 verified restoration of the original collection and identified a missing bundle at the temporary build repository path in the prior management log. Attempt 141 found the bundle and collection entry by explicit read-only repository inspection.

In both Attempt 142 runs, the ordinary installed repository lookup found XeGfxAccel. macOS displayed an extension approval prompt and its management logs reported a rebuild refusal pending user approval. The diagnostic signature errors persisted. Both runs removed the temporary bundle and preserved all original collection hashes.

Attempt 143 advanced through normal approval to a reboot-required collection generation. The generated collection included XeGfxAccel 0.9, HighPointIOP, HighPointRR and AppleMobileDevice, but omitted the old installed XeGfx 0.0.48. The running EFI core remained 0.0.47, and the accelerator service never loaded. After bundle removal macOS generated another collection containing all four original entries and their original executable UUIDs, without XeGfxAccel. The regenerated hash differs from the earlier backup. Original membership is restored, not byte-for-byte identity. Full linkage qualification of the new baseline remains pending. BootKC and SystemKC matched their original hashes in the captured run.

At Attempt 143, the ordinary approval route could generate an accelerator-containing collection, but runtime startup and attachment remained unverified. The later explicit start-only request and Stage 7 tests below resolved that runtime boundary without a security-policy bypass. The separate signature diagnostic errors were not established as the cause of a fatal load failure.

### Attempts 144-149 - Approved startup and Stage 7 attachment

I verified that the regenerated auxiliary baseline retained all four original entries and linked to the expected BootKC and SystemKC. Its bytes differed from the original backup, so I preserved it as a separate identified rollback baseline. After normal approval and reboot, the accelerator-containing collection and installed bundle remained available. An explicit start-only request in Attempt 146 returned zero and loaded XeGfxAccel 0.9 with its expected UUID. The earlier restricted lookup-table error also appears in that capture, so it was not sufficient evidence of a fatal loader block.

Attempt 147 restored the manual stage-7 ceiling and removed -nvoff, but the SRCU compatibility plugin rejected the new core UUID. The collector stopped at stage 0 before initialization. I added that exact identity to the supported profiles while retaining header, symbol-offset and function-signature checks. In Attempt 148, all stages passed, the core published its claim, NMXAccel started on B082 and IOGraphicsAccelerator2::start returned 1. All 32 user-client carriers were ready, and the accelerator remained registered, matched and active across ten observations. Attempt 149 then opened and closed its documented XGDR client successfully, with independent kernel confirmation of client creation. This established attachment and a userspace connection, not rendering by itself.

### Attempts 150-154 - Private Vulkan compute and graphics

I kept the Attempt 148 EFI and placed the original MeowXE3 1.0.12 libdrm and Intel Vulkan ICD privately on FILES. The helper preloads libdrm under its original install-name identity, then calls the ICD entry points directly. This is a process-local test, not a standard system Vulkan installation. No Metal plugin is advertised by the installed accelerator, and no custom graphics library was installed into the system graphics directories by these tests.

The normal console-user helper negotiated ICD interface 7 and enumerated Intel Arc B390 (PTL), 8086:b082, advertising Vulkan 1.4.363. This version is a reported capability, not a conformance result. Attempt 151 created a logical device without optional features or extensions and retrieved family 0's queue. Attempt 152 submitted one 4 KiB buffer fill, waited successfully on its fence and checked all 1024 words against 0x1280cafe.

Attempt 153 created an application compute pipeline and dispatched 16 workgroups of 64 threads. All 1024 output words matched the index-dependent expression 0x1280cafe XOR (i multiplied by 0x01010101), using unsigned 32-bit arithmetic. Attempt 154 then created a 64x64 RGBA8 image, cleared it red and drew a full-image triangle with vertex and fragment shaders producing alternating green/magenta tiles. Submission, fence completion and image-to-buffer copying succeeded. Every RGBA pixel matched, including alpha. Independent analysis of the saved PPM verified every RGB pixel, with 2048 green and 2048 magenta pixels. The [preserved image](logs/graphics-evidence.md#offscreen-rendering-artifact) is actual readback data.

Resource cleanup completed in these successful runs. NMXAccel stayed active, the core finished idle at stage 7, collection hashes stayed unchanged and display/input remained functional. These tests establish specific transfer, compute and offscreen graphics workloads on the B390. They do not establish arbitrary shader compatibility, the full Vulkan feature set, non-coherent-memory behavior, repeated-use stability, a display surface, swapchain, presentation, Metal or accelerated WindowServer. The firmware display path remains in use.

### Attempts 155-162 - Private Metal build, compute and rendering

I audited the runtime interfaces and built the pinned Metal plugin with Apple Clang 17 and the macOS 26.2 SDK, targeting macOS 15.0. The first package failed because I omitted vulkan/vulkan.h. Supplying the complete header set produced a signed and locally verified x86_64 bundle. Process-local loading registered the expected classes, then Metal device creation returned a B390 device whose registry identity matched NMXAccel.

A shared-buffer blit verified all 4352 bytes with intact guards. The next helper compiled Metal source to Apple AIR, translated it to SPIR-V and ran a compute shader whose 1024 output words all matched. Attempt 162 rendered a 64x64 green/magenta checkerboard through vertex and fragment shaders. All 4096 pixels and guards matched, and independent image checks confirmed the saved output. These were private application workloads, not system-wide Metal discovery or desktop acceleration.

### Attempts 163-168 - IOSurface host import and retained mapping lifetime

The first IOSurface workload failed during host import with VM-bind errno 45, followed by device loss and command-submission failure. A process-local copy-path comparison passed, isolating the import route. Exact-binary inspection and runtime tracing then confirmed that xe_hmm_userptr_populate_range was an unsupported stub returning -45, with empty unmap and SG-free helpers. I retained the comparison as diagnostic evidence and implemented the missing mapping path rather than leaving host import disabled.

Attempt 166 wired and released the actual four-page IOSurface range with a matching owner context. Attempt 167 exercised genuine shim page holds, SG construction, DMA mapping and ordered teardown across all 16384 bytes. Both deliberately preserved the original HMM error until those primitives were qualified.

Attempt 168 retained the pages, SG table and DMA handle, published the binary-verified SG pointer and notifier sequence, and supplied symmetric unmap/free hooks. HMM population and all 20 logged VM-bind calls returned zero. The runtime confirmed host import with the bypass unset. Rendering completed without error. All 4096 BGRA pixels and both 256-byte guards matched, and CPU-visible IOSurface bytes matched both the expected checkerboard and separate texture readback. Independent checks confirmed 2048 red and 2048 cyan pixels in the saved surface/image. Final DMA unmap, SG free and all four page releases returned the active mapping count to zero. I confirmed everything remained working, and installed collection hashes stayed unchanged.

The runtime still renders into private images and copies output into the imported surface pages. The preview displays captured output through AppKit on the CPU. This does not establish native cross-process image aliasing, direct Metal presentation or accelerated WindowServer. The implementation is restricted to the exact tested core, aligned ranges of at most 64 pages and 16 simultaneous mappings. Its wired-page model does not supply general Linux HMM invalidation semantics. Normal final cleanup is verified for this workload, while remapping, broader concurrency, capacity and long-term stability need additional hardware qualification.

### Attempts 169-174 - Asynchronous lifetime controls and logging latency

I compared ordinary Metal buffers with imported IOSurface memory, one or two queues, shared-event gates and retained or early-release application references. Attempt 169 returned all four commits while the gate was closed, but received no callbacks within ten seconds after release. Cleanup still freed all sixteen page holds. Attempts 170-171 passed all tested combinations, including early release, without changing the mapping implementation. These later workloads included a separate GPU readback copy, so they did not identify the original timeout's cause.

Attempt 172 stopped before any commit-return or gate-release checkpoint in an ordinary, non-imported fill-only case. Status 142 was consistent with the ninety-second process alarm, whose budget included initialization. This showed that a timeout could occur without userptr population, but did not establish a deadlock. Attempt 173 repeated that case successfully. Paired checkpoints measured about 47.1 seconds in the traced main scope, including 33.6 seconds initializing the device. Samples caught NVMTL logger file close and metadata operations during initialization.

In Attempt 174, the unchanged helper passed both local/FILES log-destination comparisons in opposite orders. Local initialization took 0.107/0.118 seconds versus 0.233/0.894 seconds on FILES. Last callbacks arrived about 3.98/3.55 ms after signaling locally versus 29.80/568.83 ms on FILES. This supports diagnostic I/O affecting latency. It does not prove the cause of Attempt 169's callback timeout or Attempt 172's process alarm.

### Attempts 175-176 - Dynamic import ownership and asynchronous compute

I moved page vectors and mapping records to dynamic allocation with synchronized ownership and exact cleanup. XeGfxUserptrTrace 0.5.0 still enforces 64 pages per import and sixteen simultaneous mappings. Dynamic storage does not mean unlimited capacity or complete Linux HMM invalidation semantics. I also switched the collector storage to a USB 3.2 Gen 1 device, so the subsequent pass cannot isolate allocation changes from the storage change.

Attempt 175 completed sixteen imported 16 KiB fill jobs over two queues with early application-reference release. Every target stayed alive while its gate was closed, all callbacks and independent data/guard checks passed, and all sixteen custom deallocators ran. Attempt 176 replaced fills with shader read/modify/write. Independent calculations verified all 63488 transformed words plus guards, original imported surfaces matched, and unused readback sentinels were unchanged. Each attempt freed all sixteen mappings and sixty-four page holds. The peak active mapping count was four in Attempt 175 and five in Attempt 176, returning to zero in both. These qualify the tested sizes and normal retirement paths, not allocation failure or arbitrary invalidation behavior.

### Attempts 177-178 - Watchdog retirement and queued offscreen rendering

Attempt 177 stopped before rendering when Stage 5 returned Resource busy after Stage 4 had completed and the core appeared idle. Exact-core disassembly showed that a stage handler can clear its active flag and return while the watchdog still owns a separate claim. The next request returns EBUSY until the watchdog releases that claim. This is a concrete handoff race consistent with the capture, although the ownership flag was not sampled at the instant of rejection. This early-stop archive contains baseline collection hashes but no after-hash comparison.

I added a separate exact-core XeGfxStageHandoffFix 0.1.0 plugin. Its serialized wrapper calls the original request once and waits for read-only retirement checks, bounded to two seconds. It neither retries EBUSY nor writes watchdog flags or bypasses the watchdog. Attempt 178 passed all six handoffs, waiting 25.9-94.7 ms after each request, with claim, active and ready zero before returning.

The unchanged render test then completed sixteen distinct 64x64 frames across two queues with the runtime's normal render wait enabled. Independent checks verified all 65536 pixels through both the original surfaces and GPU readback, plus 8192 guard bytes. All texture/readback weak references cleared, all sixteen mappings and sixty-four page holds were released, and the active mapping count ended at zero.

### Attempts 179-180 - Randomized ungated 2 GiB workload

I removed the artificial event gates while retaining corruption checks, four queues and a rolling window of eight independent jobs. Attempt 179 reached out-of-order callbacks for the initial eight jobs but verified only one output before a retirement deadline. Compiled-code inspection identified a harness lifetime bug: obtaining readback contents retained/autoreleased its owner until the surrounding pool drained, while the harness waited for that owner's weak reference to clear inside the same pool. This established an impossible retirement condition for that reference, not failed GPU execution. Other retention was not excluded by that capture.

Attempt 180 introduced a verifier-local autorelease pool that drains before retirement checks. With the same seeded workload, EFI, runtime and deadlines, all 802 jobs completed, verified and retired. The mix was 197 fills, 216 compute transforms, 205 shuffled copy chains and 184 offscreen renders, totaling exactly 2147483648 payload bytes. The helper checked every payload word, 6569984 readback guard bytes, original imported surfaces and another 1441792 original imported-buffer guard bytes. A separate planner reproduced all job fields and an independently written reference matched all 802 saved verification hashes. Successful raw GPU buffers were not archived, so this is independent reference-hash comparison against the helper's reports, not a second examination of every output buffer.

All 802 callbacks and commit returns were accounted for. There were 119 non-sequential worker-admission transitions and 120 out-of-job-ID-order completion transitions. Peak admitted jobs was eight, which does not establish eight hardware engines executing simultaneously. Peak useful payload was 80756736 bytes, not 2 GiB resident at once. All 360 HMM populations and mapping retirements succeeded, all 9884 page holds were released, and peak active kernel mappings was five before returning to zero. The successful VM-bind trace covers its configured 256-call prefix rather than every bind. All 6430 operation records paired correctly. The stress scope lasted about 4.253 seconds, so this is finite workload qualification rather than sustained stability or a GPU-only performance benchmark.

### Attempt 181 - Visible CAMetalLayer presentation through the private device

A separate ordinary Metal discovery process returned no default device and an empty device list. I then used the established explicit private-device loader with a 384x384 AppKit window and a 128x128 BGRA CAMetalLayer. All twelve returned CAMetalDrawable textures were NVMTLTexture objects of the expected size and format, belonging to the same device. The test used framebuffer-only drawables, normal presentDrawable calls and the runtime's default render-wait policy. No CPU image-preview or readback-display fallback was used.

All twelve frames completed with command-buffer status 4 and no error, delivered presented callbacks and returned from their submission workers. I saw the checkerboard changing and reported no display or peripheral issues. This confirms visible windowed GPU rendering through the explicitly loaded private device. It does not establish system-wide Metal discovery, normal application integration, accelerated WindowServer or native display takeover. Drawable pixels were manually observed rather than automatically read back. All presentedTime values were zero, so the callbacks do not establish usable display timestamps or vblank synchronization.

Two 65536-byte imported drawable mappings were reused across the twelve frames, then freed with all thirty-two page holds released. Peak active mappings was two and the final count was zero. All 22 logged binds, 94 operation pairs and six watchdog handoffs passed. The core ended idle at Stage 7 and NMXAccel remained active. Installed collection hashes were unchanged in the completed later tests. A scheduled sample could not examine the exiting helper and supplied no call graph, which does not indicate a stall.

### Attempt 182 - Bundle signing passed, collector path collision blocked loading

attempt 182 - I tested the normal Metal bundle preparation with the unchanged working EFI. The FILES template passed its fifteen file checks, the copied bundle was ad-hoc signed and signature verification passed. Independent comparisons confirmed that signing preserved the executable's code and constant-string sections, and its UUID stayed unchanged. Ordinary Metal discovery still returned no default device and an empty device list, as expected without installation or registry advertisement. The class-loading preflight then exited with status 4 before printing bundle information or attempting to load the driver. I traced this to a collector variable collision. An earlier inventory loop reused the prepared bundle's path variable and left it pointing at the unmatched /System/Library/GPUBundles/* pattern. The helper therefore received the wrong path and NSBundle creation returned nil. A shell replay reproduced the clobbered value. This is a collector failure before the intended driver-loading test, not evidence that the signed Metal plugin cannot load. No device was instantiated or GPU work submitted by that helper, and there were no userptr population calls. The prerequisite stages passed through 7, the userspace GPU store and XGDR open/close checks passed, and all six watchdog handoffs completed with ownership flags clear. The core ended idle, NMXAccel remained active, installed collection hashes stayed unchanged and I reported everything working. Nothing was installed or advertised. The next test needs a dedicated bundle-path variable and explicit path checks before repeating the class-loading preflight.

### Attempt 183 - Signed FILES bundle class loading passed

attempt 183 - I repeated the signed FILES bundle preflight with the unchanged working EFI and a protected bundle-path variable. Both the collector and helper recorded the correct path, and the resolved path matched. The bundle signed successfully and passed both signature checks. Independent comparisons of the archived prepared tree matched the template except for the expected signature change in the executable, with its code, constant-string sections and UUID preserved. NSBundle loaded without error, resolved the exact NVMTLDevice principal class and initWithAcceleratorPort: selector, and confirmed that the selector implementation came from the expected prepared executable. The helper printed its pass marker and exited zero without enable or ICD overrides. This confirms the previous collector path collision did not recur. The plugin's normal Objective-C class-load routines ran, but the helper did not call the device initializer, instantiate a device or submit GPU commands. Ordinary Metal discovery in a separate process still returned no default device and zero devices. Nothing was installed or advertised. All stages passed through 7, all six watchdog handoffs returned zero with ownership flags clear, and the separate prerequisite GPU store and XGDR open/close checks passed. No new imported mappings were recorded and cleanup passed. The core ended idle, NMXAccel remained active, installed collection hashes stayed unchanged and everything remained working. This qualifies signed bundle class loading from FILES, while installed dependency resolution, device initialization and native application discovery remain separate integration steps requiring a reviewed installation proposal. I think I should go back to userptr and finish it, a lot of the issues mubai and WTL have encountered could have been related to this.

### Attempt 184 - Larger imports and 32 active mappings passed

attempt 184 - I tested the userptr 0.6.0 candidate with plugin-owned flat SG tables and explicit per-owner and global admission budgets. Only the userptr executable and plist changed in the external EFI, with the previous working EFI preserved for rollback. The exact candidate loaded and its guarded hooks installed. The Metal helper compiled without diagnostics and all eight groups passed, covering 39 imports. Individual imports of 4, 64, 101, 102, 103, 1024 and 5402 pages succeeded. The largest allocation contained 22118400 useful bytes plus two 4096-byte guards, matching the useful byte count of a tightly packed 2880x1920 BGRA image. This was an imported buffer rather than a full-resolution drawable texture. A separate group retained 32 small imported buffers, and kernel publication records confirmed 32 active mappings before retirement. GPU fills checked through the original IOSurface and CPU-written patterns copied by the GPU into separate readbacks all matched. The helper checked 28057600 useful bytes in each direction and 319488 original guard bytes. The independent analyzer checked the workload and saved reports, while successful raw GPU outputs were not archived. All 39 HMM populations and 93 logged binds returned zero. Every mapping was freed and all 6928 page holds released, with zero active mappings at the end. All imported-buffer deallocators completed, all stages and watchdog handoffs passed, the separate GPU store and XGDR checks succeeded, and the core ended idle with NMXAccel active. Installed collection hashes stayed unchanged and everything remained working. The capacity scope lasted about 0.317 seconds. This exceeds the old 64-page and 16-mapping limits in the tested workloads, while the configured maximum capacities, budget exhaustion, global lock contention, general invalidation, cross-process ownership and longer concurrent workloads remain unqualified.

### Attempt 185 - Independent clients and completed-work process exit passed

attempt 185 - I tested independent memory clients with the unchanged working userptr 0.6.0 EFI. The helper compiled without diagnostics. Two separate processes each imported sixteen guarded four-page buffers and reached a pipe rendezvous while retaining them. Kernel publications attributed sixteen mappings to each PID, with interleaved publication progress and 32 active mappings before retirement. Both clients passed GPU fills checked through their original IOSurface pointers and CPU-written patterns copied by the GPU into separate readbacks, then reported all normal deallocators. These were independent allocations rather than a shared surface transferred between processes. A third client imported and checked sixteen buffers, then called exit(0) while its objects remained retained, after all GPU commands had completed. Its sixteen kernel mappings were freed and the active count returned to zero before a fourth recovery client published its four imports. The recovery client passed its checks and normal retirement too. Across all clients, 52 imports checked 425984 useful bytes in each direction and 425984 guard bytes. All 52 HMM populations and 138 logged binds returned zero, all 52 mappings were freed and all 208 page holds released. The three phase cleanup boundaries accounted for 32, 16 and 4 frees and each ended at zero active mappings. The independent analyzer checked every saved client, workload count and kernel transition, while raw successful GPU buffers were not archived. All stages, watchdog handoffs, the separate GPU store and XGDR checks passed. The core ended idle with NMXAccel active, installed collection hashes stayed unchanged and everything remained working. This qualifies small independent-client activity and process-exit cleanup after completed GPU work, while shared-surface IPC, pending-work exit, address replacement, general invalidation, quota exhaustion and sustained contention remain unqualified.


### Standalone userptr publication

The [lhl-userptr-patch](https://github.com/lems-hardware/lhl-userptr-patch) repository publishes XeGfxUserptrFix 0.6.0 with source, checksums, original tested artifacts and exact rename provenance. The renamed build has UUID E8A629A6-2786-3E53-8792-9FDAE1810596 and awaits its own hardware capture. Attempts 184-185 used XeGfxUserptrTrace 0.6.0, UUID FBC73E09-EF26-32D2-A22E-0EE089A4D277. Publication did not change the working EFI.

The new configured limits are 64 MiB per import, 128 MiB per owner and 256 MiB globally, with 64 records per owner and 256 globally. These are policy ceilings beyond the complete hardware qualification so far. Shared-surface IPC, budget exhaustion, allocation-failure recovery and process termination with outstanding GPU work remain priorities. General HMM invalidation, address replacement and global-lock contention measurements also remain open.

## 9. Current status and evidence limits

| Area                     | Verified outcome                                                                                                                                                                  | Open question                                                                                                                                                                                                      |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Boot and CPU             | Recovery, Terminal and the installed desktop boot via OpenCore with CPUID emulation, classic firmware LAPIC entries and the Darwin CPU SSDT.                                      | Long-run stability and native CPU power management.                                                                                                                                                                |
| Storage and installation | Internal NVMe initialization, an offline APFS installation and a bootable installed desktop.                                                                                      | Broader external device support and installation reproducibility across configurations.                                                                                                                            |
| Display and GPU | Firmware desktop retained, both GTs initialized, accelerator startup and stage-7 attachment confirmed. Private Vulkan and Metal transfer, compute and offscreen graphics pass exact readback. Host import, a desktop-sized buffer, 32 mappings across independent clients, completed-work process-exit cleanup, a 2 GiB cumulative async workload and twelve visible private-device CAMetalLayer frames pass. | Ordinary Metal discovery, normal application integration, accelerated WindowServer, native display takeover, general invalidation, full API coverage and extended stability. |
| Keyboard                 | Functional built-in PS/2 input. Skipping the PS/2 boot reset improved captured startup behavior.                                                                                  | Reliability across all restart paths.                                                                                                                                                                              |
| Touchpad                 | Native PIXA3854 input with gestures and settings.                                                                                                                                 | Long-run stability, sleep/wake and continued coexistence with PS/2 fallback.                                                                                                                                       |
| External USB             | All 17 mapped ports publish, the tested Logitech mouse works, and the SD reader remains accessible at 5 Gb/s. Six latest same-port reconnects reached HID startup in 0.52-0.54 s. | Provisional connector types, other ports/orientations, occasional reset retry and sleep/wake.                                                                                                                      |
| Wi-Fi                    | Native open 2.4 GHz joins, DHCP, DNS and Safari HTTPS. One 3–4-hour run and one automatic reconnect after an overnight lid close.                                                 | Repeatability across access points and sleep states, broader traffic support, captive portals, secured networks and other bands.                                                                                   |
| Audio                    | ALC285 microphone, speakers, wired headphones, volume/mute and automatic jack switching, including across restarts.                                                               | Extended stability, sleep/wake and separately tested line input.                                                                                                                                                   |
| Battery                  | Battery data publication and charger disconnect/reconnect confirmed in a cold boot with SMCBatteryManager 1.3.8.                                                                  | Restart behavior, accuracy, charge limits and long-run stability.                                                                                                                                                  |
| Other devices and power  | Webcam and fingerprint device enumerate. Lid-close/open events are detected.                                                                                                      | Actual video, Bluetooth, fingerprint function, full sleep/resume and CPU power efficiency. The boot framebuffer currently prevents full sleep. Firmware coexistence success does not establish Bluetooth function. |

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

Graphics credits: [watchthelight / WTLIntelGfx](https://github.com/watchthelight-labs/WTLIntelGfx), [InSnh-Gd's Panther Lake fork](https://github.com/InSnh-Gd/WTLIntelGfx/tree/5ffc79fb9e53eb820899edd644b83681bea3aa39) and [Mubai's pinned work](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/tree/180a42fafd1e1b8116bdc1f0312dec54be3812c5) provide the core, Panther Lake port and diagnostic foundation. The local DRM-only sleepable SRCU route uses Lilu and DebugEnhancer. [cuadra8998 / MeowXE3](https://github.com/cuadra8998/MeowXE3) and [ericek111's Xe3 VF work](https://github.com/ericek111/vibe-nullmoth-xe3-qemu) provide comparison research. NullMoth, HighDelay, Intel, Linux DRM/xe, FreeBSD drm-kmod/LinuxKPI, Mesa, libdrm, LLVM and linux-firmware contributors retain credit and their original licenses. [NVIDIA open-gpu-kernel-modules 615.78.08](https://github.com/NVIDIA/open-gpu-kernel-modules/tree/615.78.08) supplied headers required by the shared accelerator source build. The VF work uses a Linux-owned physical function and virtualized scanout, which differs from this physical panel path.
