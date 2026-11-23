## lhl-hackintosh-docs

The current research platform is a **Framework Laptop 13 Pro** featuring Intel's Panther Lake architecture.

Specs:

| Component                    | Hardware                        |
| ---------------------------- | ------------------------------- |
| Laptop                       | Framework Laptop 13 Pro         |
| Processor                    | Intel Core Ultra X7 358H        |
| Integrated graphics          | Intel Arc B390 (Xe3)            |
| GPU PCI ID                   | `8086:B082`                     |
| Wireless                     | Intel BE211 Wi-Fi 7 (CNVio3)    |
| Installed experimental OS    | macOS Sequoia 15.8.3 (`24H241`) |
| Initial Recovery environment | macOS Sequoia 15.4.1            |
| Bootloader                   | OpenCore (external USB EFI)     |

This hardware is not officially supported by macOS. Significant compatibility work is required to initialize the operating system and its hardware drivers, along with most of my sanity.

## Current Progress

Research status: Xe3 graphics bring-up is the current focus. Normal macOS approval and an explicit module start now load the accelerator, and manual stage 7 attaches NMXAccel on the B390. Privately staged Intel Vulkan libraries enumerate the GPU, create a device and queue, complete a checked buffer fill and run an application compute shader. Attempt 154 rendered a 64x64 checkerboard with vertex and fragment shaders, with all 4096 pixels verified through readback. Metal, presentation and an accelerated desktop remain unverified. Onboard audio, the tested USB mouse path and battery/charger reporting remain available.

| Component                       | Status                  | Notes                                                                                                                                                                                                                   |
| ------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenCore boot                   | Working                 | Boots Sequoia Recovery and the installed system                                                                                                                                                                         |
| macOS Recovery                  | Working                 | Built-in Sequoia 15.8.3 Recovery and Terminal confirmed, with the same functioning features observed as the normal installation during rollback checks. Individual subsystem parity remains unqualified.                                                                                                        |
| macOS installation              | Working                 | Sequoia 15.8.3 installed offline to a dedicated internal APFS partition                                                                                                                                                 |
| Installed desktop               | Working                 | Setup completed, desktop and System Settings reached                                                                                                                                                                    |
| CPU compatibility               | Partial                 | Boots with CPUID spoofing and custom ACPI, macOS reports an unknown CPU                                                                                                                                                 |
| Integrated graphics/display | Partial | Basic display output remains in use. Accelerator startup and stage-7 attachment work in captured runs. Private Vulkan compute and offscreen rendering pass, while Metal and native display takeover remain untested. |
| Internal keyboard               | Working                 | VoodooPS2                                                                                                                                                                                                               |
| Touchpad                        | Working                 | Gestures, scrolling, and Trackpad settings work                                                                                                                                                                         |
| USB controllers | Working in tested configuration | USBToolBox and an explicit map create all 12 XHCI and five TXHC ports. Connector types remain provisional, with other ports and Type-C orientation unverified. |
| External USB storage | Working in tested setup | The SD Expansion Card enumerates at 5 Gb/s and FILES is writable. Broader device coverage and sleep/wake remain unverified. |
| External USB mouse | Working on tested port | Logitech M330 receiver buttons, scrolling and reconnect work on top-right XHCI HS02. Six latest reconnects reached HID startup in 0.52-0.54 s with a scoped 350 ms reset window. |
| Native I²C touchpad             | Working                 | VoodooI2CHID and VoodooInput attach in polling mode for touch reports, appears to be reliable throughout experiments                                                                                                    |
| Wi-Fi                           | Partial                 | The experimental BE211 driver loaded a public HTTPS page in Safari on an open 2.4 GHz network. A follow-up ran for 3–4 hours without unexpected disconnects and reconnected automatically after an overnight lid close. |
| GPU acceleration | Experimental, offscreen Vulkan verified | Transfer, compute and a small vertex/fragment graphics pipeline produce checked GPU output. No accelerated desktop, presentation or Metal demonstrated. |
| Onboard audio | Working in tested functions | ALC285 microphone, speakers, wired headphones, volume/mute and automatic jack switching work, including across restarts. Extended stability and sleep/wake remain unverified. |
| Webcam | Enumerated | The internal webcam appears in USB and Camera inventories. Video capture has not been tested. |
| Bluetooth | Not yet verified | Installed-system functional testing pending. |
| Battery and charger reporting | Working in captured cold boot | SMCBatteryManager 1.3.8 publishes battery data and correctly reports AC/battery/AC transitions. Restart behavior and long-run accuracy remain unverified. |
| Sleep/wake and CPU power management | Not yet verified | Battery publication and lid events are confirmed. The boot framebuffer blocks full system sleep. CPU power efficiency remains unverified. |

These statuses may be hardware specific to this configuration or potentially even to this laptop only.

## Research Milestones

- **Attempt 11:** Progressed beyond the initial boot-services transition after changing firmware settings.
- **Attempt 13:** Improved CPU topology compatibility through custom ACPI modifications.
- **Attempts 20-32:** Used a modified DebugEnhancer kext to investigate USB controller initialization and the `AppleUSBHostResources` dependency.
- **Attempt 33:** Reached the first graphical macOS interface after introducing a macOS-specific EC device.
- **Attempt 35:** Reached macOS Sequoia Recovery Utilities.
- **Attempt 40:** Confirmed working keyboard input and opened Recovery Terminal.
- **Attempt 41:** Investigated missing USB 2.0 ports and inconsistent USB speed identifiers.
- **Attempt 48:** Confirmed basic touchpad functionality through PS/2 emulation, with inconsistent behavior across boot methods.
- **Attempt 49/November 12, 2026:** Completed an offline installation of **macOS Sequoia 15.8.3**, booted the installed system, and reached the desktop.
- **Attempt 59:** Patched AppleHPET to leave GSI 21 available for I2C5, enabling the native I²C touchpad.
- **Attempts 72-80:** Confirmed passive scanning in the native Wi-Fi menu, firmware link activation, and peer add and removal.
- **Attempts 81-86:** Allocated a management transmit queue and used session protection to complete Open System authentication and association.
- **Attempt 90:** Completed DHCP in the driver diagnostic.
- **Attempt 97:** Answered the extended BSS query with the active association rates. macOS kept the link up for the full test, obtained its own DHCP lease, and published IPv4 and DNS settings.
- **Attempts 100-101:** Reused completed transmit slots, loaded a public HTTPS page in Safari, and removed the scheduled disconnect. The connection held for hours and reconnected after an overnight lid close.
- **Attempts 102-103:** Located an AppleHDAController probe failure and enabled ALC285 audio using AppleALC 1.9.8, layout 66 and the `8086:51CA` controller compatibility ID. Microphone, speakers, wired headphones, volume/mute, automatic jack switching and operation across restarts were confirmed.

- **Attempts 104-105:** Traced missing USB port creation and added USBToolBox 1.2.0 with an explicit Framework map. All 17 ports were created, the Logitech receiver worked, and the webcam and fingerprint device enumerated.
- **Attempts 106-108:** Located slow reconnects in repeated null device-creation results. A 350 ms reset window scoped to the receiver port reduced reconnect time. All six latest trials succeeded on the first reset call, reaching HID startup in about half a second. An earlier trial still needed one retry, so broad stability is not established.
- **Attempt 109:** Added matching SMCBatteryManager 1.3.8. macOS published battery information and correctly switched between charging, discharging and charging after charger removal/reconnection.

- **Attempts 110-111:** Confirmed lid-close/open detection and identified `.Display_boot` as a system sleep preventer. Full hardware sleep/resume was not established.
- **Attempts 112-114:** Investigated MeowXE3 injection failure, verified its graphics imports in the installed system collection and successfully loaded a separate Panther Lake xe core at idle stage 0.
- **Attempts 115-116:** Caught two failed runtime patch applications before the collector issued a stage request. These are separate attempts, with the original capture labels preserved.
- **Attempts 117-118:** Directly adapted the pinned core for B082 metadata-only checks, then passed stages 1-3. PCI bus setup preserved the display metadata and progressing frame counter. Firmware loading, rendering and acceleration were not tested.
- **Attempts 119-120:** Diagnosed a stage-4 panic in a sleeping DRM SRCU reader and added a scoped compatibility fix. Both GTs initialized with GuC 70.72.1 running and submission enabled.
- **Attempts 121-122:** Completed empty batches on all six engines, then verified distinct GPU-written values through CPU readback. Display metadata stayed unchanged.
- **Attempt 123:** Queried xe 1.1.0 from a root macOS process through the render node and closed it successfully.
- **Attempts 124-128:** Verified userspace buffer allocation, CPU mapping, VM/syncobj lifecycle, bind/unbind completion and render-queue submission. The final private GPU store read back exactly 0x1280cafe with intact guards and successful cleanup.
- **Attempt 129:** Built a Sequoia accelerator candidate, but OpenCore rejected it during vtable patching after both graphics dependencies were missing from BootKC. The collector stopped at stage 0 and I reported all working. Stage 7 was not requested.
- **Attempt 130:** Standalone SRCU fix passed stages 1-6 and the private userspace GPU store again.
- **Attempts 131-135:** Isolated collection input selection and filesystem ownership, with diagnostic authentication still unresolved.
- **Attempts 136-138:** Built and inspected an uninstalled accelerator collection, verified its linkage and retained all four existing auxiliary entries. No installed collection changed.

- **Attempts 139-141:** Verified Recovery rollback operations, booted the linked accelerator collection, and identified the missing original bundle behind the start-only error 71. Original collection restored, explicit bundle discovery confirmed.
- **Attempts 142-143:** Normal repository placement reached the macOS approval prompt. Approval generated a new auxiliary collection and reported that a reboot was required. Cleanup triggered another rebuild restoring the original membership. The accelerator remains unloaded.
- **Attempts 144-146:** Verified the regenerated rollback baseline, retained the accelerator through approval and reboot, then successfully loaded its exact module with an explicit start-only request. Earlier manager lookup errors did not prevent that load.
- **Attempts 147-149:** A missing SRCU profile first stopped the test before GPU initialization. Adding the exact core identity allowed manual stage 7 to attach NMXAccel, start Apple's accelerator superclass and open/close the XGDR userspace interface.
- **Attempts 150-152:** Unmodified private Intel Vulkan libraries enumerated B390, created a logical device and queue, and completed a 4 KiB buffer fill with all 1024 words matching CPU readback.
- **Attempts 153-154:** An application compute shader produced all 1024 expected output words. A vertex/fragment graphics pipeline then rendered a 64x64 green-and-magenta checkerboard, with all 4096 RGBA pixels matching and the saved RGB image independently verified. [Captured rendering](logs/graphics-evidence.md#offscreen-rendering-artifact).


### Offline installation milestone

The successful installation used:

1. The full Sequoia 15.8.3 `InstallAssistant.pkg` downloaded through [gibMacOS](https://github.com/corpnewt/gibMacOS).
2. `UnPlugged.command` to prepare and launch the full installer from Sequoia Recovery.
3. A manually mounted ExFAT installer-data partition on the existing OpenCore USB drive.
4. A dedicated approximately 150 GiB APFS partition (`Macintosh HD`) on the internal SSD.
5. The existing external OpenCore EFI to boot the macOS installer and installed system.

The procedure succeeded on this system, but has not been established as a repeatable general-purpose installation method for other Panther Lake devices.

For exact configuration changes, observations, and unresolved questions from the earlier boot work, refer to the complete attempt log.

## Documentation and Evidence

### Boot attempt journal

[Framework Panther Lake: macOS Sequoia boot attempts](logs/fw13p-attempts-sequoia.md)

A chronological record of configuration changes, experimental boot results, debugging observations, and technical findings. The journal covers attempts 1-154, including BE211 Wi-Fi, audio, USB, battery/lid reporting, accelerator attachment and verified Vulkan transfer, compute and offscreen graphics output. The [graphics evidence summary](logs/graphics-evidence.md) records exact builds, results and limitations.

### Visual archive

[Framework Panther Lake: Sequoia boot photographs](logs/fw13p-visual-sequoia/)

Photographs documenting kernel output, driver initialization, Recovery milestones, and hardware investigations. May remain unsorted until later.

## Research Priorities

1. **(Paused) Wireless networking:** Extend BE211 support to secured networks and additional bands, and test more reconnect scenarios. WPA2/WPA3 and 5 GHz support remain open work.
2. **System stability and power:** Qualify battery reporting across restarts and investigate the boot framebuffer sleep preventer and CPU power efficiency. Lid notifications already work.
3. **USB compatibility:** Qualify additional physical ports, Type-C orientation and sleep/wake. Correct provisional connector metadata and monitor the remaining occasional reconnect retry.
4. **GPU acceleration (current focus):** Investigate compatible Metal integration and presentation while preserving the verified private Vulkan path. Accelerator attachment, compute and offscreen graphics work in captured runs. Native display takeover, accelerated WindowServer and broader stability remain open.
5. **(Complete) Input devices:** Improve PS/2 initialization reliability and test native I²C touchpad stability.

## Project Scope

This repository primarily contains research documentation, observations, and visual evidence.

It is **not currently a complete Hackintosh installation guide or a production-ready EFI distribution**. A public EFI release is not provided here as it is unknown how stable it is in its current form.

Experimental results may be hardware-specific, incomplete, or subject to revision as additional evidence becomes available.

## License

Original documentation and research materials are licensed under the [Creative Commons Attribution 4.0 International License](LICENSE), unless otherwise stated.

Third-party code, software, and other materials retain their respective licenses.

## Credits

- [Acidanthera](https://github.com/acidanthera) for OpenCore, Lilu, VirtualSMC, VoodooPS2 and AppleALC. The ALC285 layout 66 profile is credited upstream to R-a-s-c-a-l in the [codec resources](https://github.com/acidanthera/AppleALC/blob/1.9.8/Resources/ALC285/Info.plist).
- [VoodooI2C contributors](https://github.com/VoodooI2C/VoodooI2C) and [VoodooInput contributors](https://github.com/acidanthera/VoodooInput) for the native touchpad stack.
- [OpenIntelWireless](https://github.com/OpenIntelWireless/itlwm) and [Ersiter's AirportItlwm-BE200 fork](https://github.com/Ersiter/AirportItlwm-BE200) for the wireless driver foundation. Intel supplies the firmware under its separate redistribution terms. My BE211 changes and build provenance are in [lhl-be211-wireless](https://github.com/lems-hardware/lhl-be211-wireless).
- [yangyishuo1124 / Mubai](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/blob/5f0de92f42b3655d6539b91fe3a630b8262096f3/HARDWARE_FOLLOWUP_2026-11-13.md) for documenting the Panther Lake audio controller compatibility-ID experiment that informed Attempt 103. That record also cites [VedCloud's Meteor Lake work](https://github.com/VedCloud/Meteor-Lake-Xe-LPG-Metal-Graphics) as an earlier `51CA` compatibility reference. I retained my Framework's ALC285 layout and verified the result on this machine.
- [USBToolBox / Dhinak G. and contributors](https://github.com/USBToolBox/kext/tree/1.2.0) for runtime ACPI port-description bypass and map injection. The Framework map and reset-window experiment were derived from this machine's captures and exact installed kernel collection.
- lvs1974 and DebugEnhancer contributors for the tracing foundation, and usrsse2 and VirtualSMC contributors for SMCBatteryManager.
- [watchthelight](https://github.com/watchthelight-labs/WTLIntelGfx), [InSnh-Gd](https://github.com/InSnh-Gd/WTLIntelGfx) and [Mubai](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs) for the xe core, Panther Lake port and metadata-only diagnostics used in Attempts 114-129. NullMoth supplied the shared accelerator foundation, and [NVIDIA open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules/tree/615.78.08) supplied required build headers. [cuadra8998 / MeowXE3](https://github.com/cuadra8998/MeowXE3), [ericek111 / NullMoth Xe3 VF work](https://github.com/ericek111/vibe-nullmoth-xe3-qemu), and their upstream contributors informed the comparison research. Their reported acceleration is not presented as a Framework result.
- [corpnewt / gibMacOS](https://github.com/corpnewt/gibMacOS), [MacKernelSDK](https://github.com/acidanthera/MacKernelSDK) and [cctools-port](https://github.com/tpoechtrager/cctools-port) for installer retrieval and build tooling. Apple and the upstream operating-system projects retain credit and rights for their software and driver sources.
- [cuadra8998 / MeowXE3 1.0.12](https://github.com/cuadra8998/MeowXE3) for the packaged Intel Vulkan ICD and libdrm used unchanged in the private userspace tests, together with the Mesa/ANV, Intel, libdrm and NullMoth port contributors. [Khronos](https://github.com/KhronosGroup) supplies the Vulkan headers, glslang and SPIR-V validation tools used by the test helpers. Those upstream projects retain their original licenses.
