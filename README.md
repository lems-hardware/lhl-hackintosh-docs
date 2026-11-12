# LHL Hackintosh Documentation

This repository is part of **Lem's Hardware Lab (LHL)** and serves as a public archive of experiments involving macOS on unsupported PC hardware.

The goal is to document technical progress, failures, workarounds, and reproducible findings that may help future Hackintosh and driver development projects.

## Current Research: Intel Panther Lake

The primary research platform is a **Framework Laptop 13 Pro**, featuring Intel's Panther Lake architecture.

| Component | Hardware |
|---|---|
| Laptop | Framework Laptop 13 Pro |
| Processor | Intel Core Ultra X7 358H |
| Integrated graphics | Intel Arc B390 (Xe3) |
| GPU PCI ID | `8086:B082` |
| Wireless | Intel BE211 Wi-Fi 7 |
| Primary experimental OS | macOS Sequoia 15.4.1 |
| Bootloader | OpenCore |

This hardware is not officially supported by macOS. Significant compatibility work is required to initialize the operating system and its hardware drivers.

## Current Progress

**Research status: Experimental, 40+ documented boot attempts**

| Component | Status | Notes |
|---|---|---|
| OpenCore boot | Working | Kernel loads successfully |
| macOS Recovery | Working, inconsistently | Graphical Recovery environment reached |
| Recovery Terminal | Working | Interactive terminal access confirmed |
| CPU compatibility | Partial | CPUID spoofing and custom ACPI configuration |
| Internal keyboard | Working | VoodooPS2 |
| Touchpad | Partial | Basic pointer movement through PS/2 emulation; inconsistent initialization |
| USB controllers | Partial | Both xHCI controllers start, but USB 2.0 port enumeration remains unresolved |
| External USB storage | Not working | No external storage detected in Recovery |
| Native I²C touchpad | Under investigation | VoodooI2C compatibility experiments |
| Wi-Fi | Unsupported | Intel BE211 driver research required |
| GPU acceleration | Unsupported | Xe3 graphics driver development required |

These results represent observed behavior on the research hardware, some entries may not be completely confirmed.

## Research Milestones

- **Attempt 11:** Progressed beyond the initial boot-services transition after changing firmware settings.
- **Attempt 13:** Improved CPU topology compatibility through custom ACPI modifications.
- **Attempts 20–32:** Used a modified DebugEnhancer kext to investigate USB controller initialization and the `AppleUSBHostResources` dependency.
- **Attempt 33:** Reached the first graphical macOS interface after introducing a macOS-specific EC device.
- **Attempt 35:** Reached macOS Sequoia Recovery Utilities.
- **Attempt 40:** Confirmed working keyboard input and opened Recovery Terminal.
- **Attempt 41:** Investigated missing USB 2.0 ports and inconsistent USB speed identifiers.
- **Attempt 48:** Confirmed basic touchpad functionality through PS/2 emulation, with inconsistent behavior across boot methods.

For exact configuration changes, observations, and unresolved questions, refer to the complete attempt log.

## Documentation and Evidence

### Boot attempt journal

[Framework Panther Lake: macOS Sequoia boot attempts](logs/fw13p-attempts-sequoia.md)

A chronological record of configuration changes, experimental boot results, debugging observations, and technical findings.

### Visual archive

[Framework Panther Lake: Sequoia boot photographs](logs/fw13p-visual-sequoia/)

Photographs documenting kernel output, driver initialization, Recovery boot milestones, and hardware investigations. Unfortunately I did not record down exact attempts for each corresponding photo, but they still can be used. 

## Research Priorities

Current areas of investigation include:

1. **USB compatibility:** Diagnosing xHCI port creation, USB 2.0 enumeration, and external device detection.
2. **Wireless networking:** Investigating possible Intel BE211 support under macOS.
3. **Input devices:** Improving PS/2 initialization reliability and exploring native I²C multitouch.
4. **GPU acceleration:** Researching potential Panther Lake Xe3 enablement through experimental Intel graphics drivers.
5. **Boot reliability:** Understanding hardware initialization differences between cold boots and warm restarts.

## Project Scope

This repository primarily contains research documentation, observations, and visual evidence.

It is **not currently a complete Hackintosh installation guide or a production-ready EFI distribution**.

Experimental results may be hardware-specific, incomplete, or subject to revision as additional evidence becomes available.

## License

Original documentation and research materials are licensed under the [Creative Commons Attribution 4.0 International License](LICENSE), unless otherwise stated.

Third-party code, software, and other materials retain their respective licenses.

---

**Lem's Hardware Lab (LHL)**

*Experimental hardware compatibility, operating-system research, and low-level software development.*
