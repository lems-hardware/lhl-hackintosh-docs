# LHL Hackintosh Documentation

This repository is part of **Lem's Hardware Lab (LHL)** and serves as a public archive of experiments involving macOS on unsupported PC hardware.

The goal is to document technical progress, failures, workarounds, and reproducible findings that may help future Hackintosh and driver development projects.

## Current Research: Intel Panther Lake

The primary research platform is a **Framework Laptop 13 Pro** featuring Intel's Panther Lake architecture.

| Component | Hardware |
|---|---|
| Laptop | Framework Laptop 13 Pro |
| Processor | Intel Core Ultra X7 358H |
| Integrated graphics | Intel Arc B390 (Xe3) |
| GPU PCI ID | `8086:B082` |
| Wireless | Intel BE211 Wi-Fi 7 (CNVio3) |
| Installed experimental OS | macOS Sequoia 15.8.3 (`24H241`) |
| Initial Recovery environment | macOS Sequoia 15.4.1 |
| Bootloader | OpenCore (external USB EFI) |

This hardware is not officially supported by macOS. Significant compatibility work is required to initialize the operating system and its hardware drivers.

## Current Progress

Research status: Experimental, installed Sequoia desktop reached on November 12, 2026, following 48 documented boot attempts.

| Component | Status | Notes |
|---|---|---|
| OpenCore boot | Working | Boots Sequoia Recovery and the installed system |
| macOS Recovery | Working, inconsistently | Graphical Recovery and Terminal confirmed; cold/warm boot behavior varies |
| macOS installation | Working | Sequoia 15.8.3 installed offline to a dedicated internal APFS partition |
| Installed desktop | Working, limited | Setup completed, desktop and System Settings reached |
| CPU compatibility | Partial | Boots with CPUID spoofing and custom ACPI, macOS reports an unknown CPU |
| Integrated graphics/display | Partial | Basic display output works without Xe3/Metal acceleration |
| Internal keyboard | Working | VoodooPS2 |
| Touchpad | Working, stability unknown | Gestures, scrolling, and Trackpad settings work |
| USB controllers | Partial | Both xHCI controllers initialize, USB 2.0 root-hub port enumeration remains unresolved |
| External USB storage | Partial | Installer USB's ExFAT partition was accessible after manual mounting in Recovery; general device support remains unresolved |
| Native I²C touchpad | Working, stability unknown | VoodooI2CHID and VoodooInput attach in polling mode for touch reports |
| Wi-Fi | Unsupported | No confirmed working macOS driver for Intel BE211 CNVio3 |
| GPU acceleration | Unsupported | No functional Xe3 graphics acceleration |
| Audio, Bluetooth, webcam | Not yet verified | Installed-system testing pending |
| Sleep/wake and power management | Not yet verified | Installed-system testing pending |

These statuses describe observations on this specific machine, not general Panther Lake compatibility. A booting desktop does not imply complete hardware support.

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

A chronological record of configuration changes, experimental boot results, debugging observations, and technical findings. The existing journal covers the earlier boot attempts, while the installation milestone is summarized above.

### Visual archive

[Framework Panther Lake: Sequoia boot photographs](logs/fw13p-visual-sequoia/)

Photographs documenting kernel output, driver initialization, Recovery milestones, and hardware investigations. Not every photograph has been matched to an exact attempt number.

## Research Priorities

1. **USB compatibility:** Investigate xHCI port creation, USB 2.0 enumeration, and reliable external-device recognition.
2. **Input devices:** Improve PS/2 initialization reliability and test native I²C touchpad stability.
3. **Wireless networking:** Research Intel BE211 CNVio3 support and possible interim connectivity options.
4. **System stability and power:** Test battery reporting, sleep/wake, restarts, and power management on the installed system.
5. **GPU acceleration:** Investigate possible Panther Lake Xe3 support through experimental Intel graphics driver work.

## Project Scope

This repository primarily contains research documentation, observations, and visual evidence.

It is **not currently a complete Hackintosh installation guide or a production-ready EFI distribution**. A public EFI release is not provided here.

Experimental results may be hardware-specific, incomplete, or subject to revision as additional evidence becomes available.

## License

Original documentation and research materials are licensed under the [Creative Commons Attribution 4.0 International License](LICENSE), unless otherwise stated.

Third-party code, software, and other materials retain their respective licenses.

---

**Lem's Hardware Lab (LHL)**

*Experimental hardware compatibility, operating-system research, and low-level software development.*
