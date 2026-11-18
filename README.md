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

Research status: subsystem bring-up is the current focus. Onboard audio is working, and further Wi-Fi development is paused.

| Component                       | Status                  | Notes                                                                                                                                                                                                                   |
| ------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenCore boot                   | Working                 | Boots Sequoia Recovery and the installed system                                                                                                                                                                         |
| macOS Recovery                  | Working, inconsistently | Graphical Recovery and Terminal confirmed. Cold/warm boot behavior varies. (based on testing before November 12)                                                                                                        |
| macOS installation              | Working                 | Sequoia 15.8.3 installed offline to a dedicated internal APFS partition                                                                                                                                                 |
| Installed desktop               | Working                 | Setup completed, desktop and System Settings reached                                                                                                                                                                    |
| CPU compatibility               | Partial                 | Boots with CPUID spoofing and custom ACPI, macOS reports an unknown CPU                                                                                                                                                 |
| Integrated graphics/display     | Partial                 | Basic display output works without Xe3/Metal acceleration                                                                                                                                                               |
| Internal keyboard               | Working                 | VoodooPS2                                                                                                                                                                                                               |
| Touchpad                        | Working                 | Gestures, scrolling, and Trackpad settings work                                                                                                                                                                         |
| USB controllers                 | Partial                 | Both xHCI controllers initialize, USB 2.0 root-hub port enumeration remains unresolved                                                                                                                                  |
| External USB storage            | Partial                 | Installer USB's ExFAT partition was accessible after manual mounting in Recovery, general device support remains unresolved                                                                                             |
| Native I²C touchpad             | Working                 | VoodooI2CHID and VoodooInput attach in polling mode for touch reports, appears to be reliable throughout experiments                                                                                                    |
| Wi-Fi                           | Partial                 | The experimental BE211 driver loaded a public HTTPS page in Safari on an open 2.4 GHz network. A follow-up ran for 3–4 hours without unexpected disconnects and reconnected automatically after an overnight lid close. |
| GPU acceleration                | Unsupported             | No functional Xe3 graphics acceleration                                                                                                                                                                                 |
| Onboard audio | Working in tested functions | ALC285 microphone, speakers, wired headphones, volume/mute and automatic jack switching work, including across restarts. Extended stability and sleep/wake remain unverified. |
| Bluetooth, webcam | Not yet verified | Installed-system testing pending |
| Sleep/wake and power management | Not yet verified        | Installed-system testing pending                                                                                                                                                                                        |

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

A chronological record of configuration changes, experimental boot results, debugging observations, and technical findings. The journal covers attempts 1-103, including the BE211 Wi-Fi experiments and onboard audio bring-up.

### Visual archive

[Framework Panther Lake: Sequoia boot photographs](logs/fw13p-visual-sequoia/)

Photographs documenting kernel output, driver initialization, Recovery milestones, and hardware investigations. May remain unsorted until later.

## Research Priorities

1. **(Paused) Wireless networking:** Extend BE211 support to secured networks and additional bands, and test more reconnect scenarios. WPA2/WPA3 and 5 GHz support remain open work.
2. **System stability and power:** Test battery reporting, sleep/wake, restarts, and power management on the installed system.
3. **USB compatibility:** Investigate xHCI port creation, USB 2.0 enumeration, and reliable external-device recognition.
4. **GPU acceleration:** Investigate possible Panther Lake Xe3 support through experimental Intel graphics driver work.
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
- [corpnewt / gibMacOS](https://github.com/corpnewt/gibMacOS), [MacKernelSDK](https://github.com/acidanthera/MacKernelSDK) and [cctools-port](https://github.com/tpoechtrager/cctools-port) for installer retrieval and build tooling. Apple and the upstream operating-system projects retain credit and rights for their software and driver sources.
