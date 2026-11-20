# Graphics evidence through Attempt 118, November 19-20, 2026

I tested the Framework Laptop 13 Pro with Arc B390 8086:b082 revision 04 on Sequoia 15.8.3 24H241, Darwin 24.6.0 and OpenCore DEBUG 1.0.8. Graphics, media and display IP are 30.00. The existing display is a 2880 x 1920 firmware framebuffer with a 22118400-byte allocation. The following results do not establish acceleration.

## Captured milestones

| Attempt | Result | Boundary |
| --- | --- | --- |
| 110 | Lid events detected, full sleep rejected with .Display_boot preventing system sleep. | No completed hardware sleep/resume demonstrated. |
| 111 | Loaded display modules identified as version 599. Initial system collection exceeded collector cap. | Instruction bytes were not captured in this run. |
| 112 | MeowXE3 injection rejected during vtable patching after missing early graphics dependency. | Core did not load. No hardware initialization tested. |
| 113 | Installed system collection captured and verified. All 76 release-core IOFramebuffer imports exist. | Symbol availability does not establish ABI compatibility. |
| 114 | Corrected EFI loaded the separate pinned xe core, idle at stage 0. | Initial wrong-EFI capture preserved separately. No GPU initialization. |
| 115 | Local-symbol patch anchors not found. | Collector refused, no stage request. |
| 116 | Runtime byte signatures had zero replacements. | Collector refused, no stage request. Original filename still says 115. |
| 117 | Stages 1-2 returned 1. Metadata result 1, frame delta 13. | CPU pixel access blocked, pixels not verified. |
| 118 | Stages 1-3 returned 1. vgapci attached, drmn unbound, post-bus metadata result 1 and frame delta 12. | No stage-4 GPU driver registration or firmware/engine test. |

I reported everything working after Attempt 117 and everything operational after Attempt 118. Both finished with running 0 and an empty environment. These are individual captured runs, not extended stability qualification. No installed graphics driver, kernel cache or internal boot configuration was modified by these EFI experiments. The older installed WTL 0.0.48 copy was not removed. Attempts 117-118 show it refused because the selected 0.0.47 build was already loaded.

## Exact core identities

| Variant | SHA-256 | Loaded UUID |
| --- | --- | --- |
| Original pinned core, Attempt 114 | cc6dbfb14a9002f52a75955da428e4ce398044ef94d795ff683a3a25eff38972 | 661198CF-750E-32FB-B003-ECF14B5AA72F |
| B082 metadata-only, cap 2, Attempt 117 | 62fb840fdf3243858bcb1ccb27d60545cbfb1922ce6922ac96ec750f6de46687 | 76FC245E-06C1-5D4B-B937-C288B79E8F8A |
| B082 metadata-only, cap 3, Attempt 118 | c1f5e9c2f68db636f6f4075b623b7a45b9a3a114d3938916bd384f70ee9fb728 | 85326386-D1AC-5183-AAEC-585AA9956CA5 |

The core derives from [InSnh-Gd/WTLIntelGfx 5ffc79fb9e53eb820899edd644b83681bea3aa39](https://github.com/InSnh-Gd/WTLIntelGfx/tree/5ffc79fb9e53eb820899edd644b83681bea3aa39), with [Mubai's changes at 180a42fafd1e1b8116bdc1f0312dec54be3812c5](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/tree/180a42fafd1e1b8116bdc1f0312dec54be3812c5). It reports version 0.0.47 and policy ptl-worker-heartbeat-v24. The local variants are binary adaptations, not source rebuilds. Each changes two machine-code bytes from the original plus UUID and existing code-page hash metadata. Both original ad-hoc code directories were refreshed and all 1750 page hashes verified. That consistency is not Apple signing trust or a qualified standalone installation.

The B082 branch selection changes the original B0A0 comparison at file offset 1868112. The manual stage ceiling changes the comparison at file offset 1857250. Both signatures were unique in the exact original binary and checked against disassembly. The first does not change any hardware ID. The second retains the original bounds check, limiting requests to 2 or 3. Automatic startup remained disabled at stage 0. Runtime OpenCore patch entries were removed after Attempts 115-116, and loaded UUID checks identified the adapted executable before each request.

## Display and PCI evidence

The metadata candidate is 0x7c800000, spanning 5400 pages. Its PTE hash is 0xcb2cd07f1a70b5. Attempt 118 reports metadata-unchanged=1, pixel-verified=0 and post-bus result=1. The diagnostic text still says B0A0 because that upstream string was unchanged. The actual captured device is B082.

The stage-3 bus tree contains root0, pcib0, pci0, hostb0, vgapci0 and an unbound drmn0 child. The driver reports no IOMMU. It skips the generic stolen-memory probe with an older-than-gen11 message when B082 is absent from its classification tables. The independent GOP metadata probe found the candidate through MMIO. Whether later xe setup needs the skipped generic globals remains unresolved. No blind B0A0 or older-generation identity substitution is justified by this message.

The exact installed system collection used in Attempt 113 has SHA-256 ad52d32857cefb2d6a8d6604f34953f5aac08db257fb5a709059f57b1dd0c974 and contains IOGraphicsFamily 599 UUID CA8B49BC-3655-312C-91AA-FB848251C78E. The MeowXE3 1.0.12 release core has 76 IOFramebuffer imports, all present there. This does not resolve their early availability to OpenCore or slot-level compatibility. MeowXE3's release Metal plugin targets macOS 26.0 minimum, so this package is not assumed to work on Sequoia.

## Original private capture digests

Raw archives, Apple kernel collections, reconstructed analysis binaries, serial numbers and unrelated registry/network data are not published here. Digests identify the evidence retained privately.

| Attempt | Original archive | SHA-256 |
| --- | --- | --- |
| 110 | `attempt-110-20261119-122500-732.tgz` | `0a5875258c4e0781873e8f2b92c8795e8a8b839b561bd6c702af8a158a0edc4c` |
| 111 | `attempt-111-20261119-124535-719.tgz` | `9372b677ad3814accb43769510208d7f6599a82727fdd617079b18ecbad0e090` |
| 112 | `attempt-112-20261119-132214-685.tgz` | `1b1ef10d04668cc42163cdebf8d450d492dee003af631f1eae9e6151a23808e3` |
| 113 | `attempt-113-20261119-133202-647.tgz` | `c4b1ddf422cdbd8fe541180546832292dbd7a7ea924b57d9cbb194c3b69a2c38` |
| 114 (initial copy mismatch) | `attempt-114-20261119-134915-689.tgz` | `b3cb16bb1109f99f667ef4dd3cf19b7b7b00e9944e7d52389c04a9cc8eb98444` |
| 114 (corrected) | `attempt-114-20261119-150902-660.tgz` | `5d28413fa4bab979cf1105e7b187fbe66226751fd92a813ada2e16c9a52038ee` |
| 115 | `attempt-115-20261119-154957-690.tgz` | `f44fc84996c3c9ad65fef7f31ea70b930f59a7bc0aaaab5e3738091a0191cdca` |
| 116 | `attempt-115-20261119-160603-671.tgz` | `bb54b4ee89b68247b7d2b9fc5f62f0a02818792f8a74f1a74d66dd7cfd896ef6` |
| 117 | `attempt-117-20261120-075943-644.tgz` | `1be04d396e0188624decfddf9ccc936b698772dda889e7c31c7461c4b2126524` |
| 118 | `attempt-118-20261120-083011-684.tgz` | `5c27638e68fca8a4dc20e8d4788e605e26a21f9ce980610a5fed45477d5d619c` |

Attempts 115 and 116 retain their original as-tested packages and capture filenames. Renumbering did not rewrite source evidence. Attempts 117 and 118 each passed matching OpenCore 1.0.8 validation, all 71 staged EFI hashes, file-reference checks and ZIP integrity checks. The loaded UUID establishes the selected core variant at runtime, while it is not an independent hash verification of every mounted EFI file.

## Credits

The core and diagnostic groundwork belongs to watchthelight, InSnh-Gd, Mubai and upstream contributors. [MeowXE3](https://github.com/cuadra8998/MeowXE3) and [ericek111's virtualized Xe3 work](https://github.com/ericek111/vibe-nullmoth-xe3-qemu) informed comparison research. Their different hardware and deployment results are not transferred to this Framework. See the [full credits](../TECHNICAL-FINDINGS.md#credits-and-references).
