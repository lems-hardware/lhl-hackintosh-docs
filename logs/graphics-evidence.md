# Graphics evidence through Attempt 123, November 19-20, 2026

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
| 119 | Stage-4 panic during LRC/GGTT cleanup and a blocking fence wait. | Exact-binary follow-up identified a sleeping SRCU reader mapped to ordinary RCU. |
| 120 | Scoped DRM SRCU route installed, stages 1-4 passed, both GTs reported GuC 70.72.1 running with submission enabled. | No dedicated completion or memory-store test in this run. |
| 121 | Six empty batches completed with matching sequences, zero errors and 48 interrupts. | No shader or rendering workload. |
| 122 | Six GPU-written diagnostic values matched CPU readback, zero errors and 49 interrupts. | Separate diagnostic buffers, no panel pixel validation. |
| 123 | Root process opened the render node, queried xe 1.1.0 and closed successfully after stages 1-6. | Initial missing-helper refusal preserved. No userspace allocation, mapping or submission test. |

I reported everything working after Attempt 117 and everything operational after Attempt 118. Both finished with running 0 and an empty environment. These are individual captured runs, not extended stability qualification. No installed graphics driver, kernel cache or internal boot configuration was modified by these EFI experiments. The older installed WTL 0.0.48 copy was not removed. Attempts 117-118 show it refused because the selected 0.0.47 build was already loaded.

## Exact core identities

| Variant | SHA-256 | Loaded UUID |
| --- | --- | --- |
| Original pinned core, Attempt 114 | cc6dbfb14a9002f52a75955da428e4ce398044ef94d795ff683a3a25eff38972 | 661198CF-750E-32FB-B003-ECF14B5AA72F |
| B082 metadata-only, cap 2, Attempt 117 | 62fb840fdf3243858bcb1ccb27d60545cbfb1922ce6922ac96ec750f6de46687 | 76FC245E-06C1-5D4B-B937-C288B79E8F8A |
| B082 metadata-only, cap 3, Attempt 118 | c1f5e9c2f68db636f6f4075b623b7a45b9a3a114d3938916bd384f70ee9fb728 | 85326386-D1AC-5183-AAEC-585AA9956CA5 |
| B082 metadata-only, cap 4, Attempt 119 | 48b1bd854375bef6e582e2629e2729b4b71e2062715412c333ea9c200f8232fa | 0F8CC02C-14D3-5CD4-A8B9-BF89EF5F09DD |
| B082 metadata-only, cap 4, Attempt 120 | 48b1bd854375bef6e582e2629e2729b4b71e2062715412c333ea9c200f8232fa | 0F8CC02C-14D3-5CD4-A8B9-BF89EF5F09DD |
| B082 metadata-only, cap 5, Attempt 121 | fe96d5bd6f3e8374f13a5402515a23a7c6ba2777b922e51c7b3fea0777a7b2b5 | 83DAB9A7-F358-5F5D-8E2A-C5ED1014E21E |
| B082 metadata-only, cap 6, Attempt 122 | 8d64ff7496decac61007bb9c39d057d093616332cf50d5440d862faa77fe97cb | 8EA9EA5B-85A3-5B73-A4B3-7B66E0BEA740 |

The core derives from [InSnh-Gd/WTLIntelGfx 5ffc79fb9e53eb820899edd644b83681bea3aa39](https://github.com/InSnh-Gd/WTLIntelGfx/tree/5ffc79fb9e53eb820899edd644b83681bea3aa39), with [Mubai's changes at 180a42fafd1e1b8116bdc1f0312dec54be3812c5](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/tree/180a42fafd1e1b8116bdc1f0312dec54be3812c5). It reports version 0.0.47 and policy ptl-worker-heartbeat-v24. The local variants are binary adaptations, not source rebuilds. Each changes two machine-code bytes from the original plus UUID and existing code-page hash metadata. Both original ad-hoc code directories were refreshed and all 1750 page hashes verified. That consistency is not Apple signing trust or a qualified standalone installation.

The B082 branch selection changes the original B0A0 comparison at file offset 1868112. The manual stage ceiling changes the comparison at file offset 1857250. Both signatures were unique in the exact original binary and checked against disassembly. The first does not change any hardware ID. Attempt 120 retained the Attempt 119 core, and Attempt 123 retained the Attempt 122 core. The second adaptation retains the original bounds check, limiting requests progressively to 2, 3, 4, 5 and 6. Attempt 123 reuses the exact Attempt 122 EFI. Automatic startup remained disabled at stage 0. Runtime OpenCore patch entries were removed after Attempts 115-116, and loaded UUID checks identified the adapted executable before each request.

## Display and PCI evidence

The metadata candidate is 0x7c800000, spanning 5400 pages. Its PTE hash is 0xcb2cd07f1a70b5. Attempt 118 reports metadata-unchanged=1, pixel-verified=0 and post-bus result=1. The diagnostic text still says B0A0 because that upstream string was unchanged. The actual captured device is B082.

The stage-3 bus tree contains root0, pcib0, pci0, hostb0, vgapci0 and an unbound drmn0 child. The driver reports no IOMMU. It skips the generic stolen-memory probe with an older-than-gen11 message when B082 is absent from its classification tables. The independent GOP metadata probe found the candidate through MMIO. The later exact-binary review found xe using its newer GGC/DSM MMIO path, and Attempt 120 initialized without a generic classification patch. No blind B0A0 or older-generation identity substitution is justified by this message.

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
| 119 | `attempt-119-20261120-085444-692.tar.gz` | `1e7eb4f706a8f6e64fc7a46ee3d5f5f2de7ceb48794e578a1210538cab40d12a` |
| 120 | `attempt-120-20261120-092610-700.tgz` | `afc7517050244e936481d95e73b5c544a323c5afc8705d9f76bd103221072f0f` |
| 121 | `attempt-121-20261120-095011-696.tgz` | `b9960c791ea7528309cfe29d36dddf837344ab254e44d13a896f41de2aeafef8` |
| 122 | `attempt-122-20261120-112706-674.tgz` | `e121c612ea5b6d40cf95d33fb9ffc58ebb0134fae98ba01bb6fa7cc114800de9` |
| 123 | `attempt-123-20261120-114915-668.tgz` | `48913886354378909e3d45651197810c921da76c0741cb0e6d5d35e70cad78ce` |
| 123 | `attempt-123-20261120-115423-650.tgz` | `0cc9683fae434be1a8f52c82c59210deff4f0893299d9696871a78d1f96fa24b` |

Attempts 115 and 116 retain their original as-tested packages and capture filenames. Renumbering did not rewrite source evidence. Attempts 117 and 118 each passed matching OpenCore 1.0.8 validation, all 71 staged EFI hashes, file-reference checks and ZIP integrity checks. The loaded UUID establishes the selected core variant at runtime, while it is not an independent hash verification of every mounted EFI file.

## Initialization, SRCU and userspace evidence

The Attempt 119 stack was verified against all 16 XeGfx frames in the matching core. drm_dev_enter held drm_unplug_srcu while GGTT removal reached a fence wait. Its SRCU reader called ordinary RCU and pinned preemption, causing the blocking-with-preemption-disabled panic. Attempt 120 kept the core unchanged and added three exact-UUID/signature-checked Lilu routes for that DRM domain. Readers could sleep, while two reader epochs and serialized grace periods retained lifetime protection. Other domains used the original functions. This supports the diagnosis without qualifying general SRCU or hot-unplug behavior.

Attempt 120 reported two GTs, six engines, one MSI vector and GuC 70.72.1 running with submission enabled on both GTs. Attempt 121's six empty-batch fence completions had matching sequences and zero errors. Attempt 122's six distinct store values all matched CPU readback with zero errors. These writes used separate diagnostic allocations, not the panel buffer. Both tests finished idle with an empty environment, unchanged display metadata and a frame-counter advance of 12. The source and exact deployed store implementation differ, so stage 6 was reviewed from the binary rather than assumed from the older source.

Attempt 123 reused the exact Attempt 122 EFI and returned driver name xe, version 1.1.0 and description Intel Xe Graphics through command 0xc0406400. The native helper opened /dev/dri/renderD128, performed one VERSION ioctl and closed, with zero errors and exit status 0. It ran as root. The limited scan found IOAcceleratorFamily2 485.2 and IOGraphicsFamily 599, but no accelerator service in the queried classes or custom bundles in the scanned paths. Final display properties retained the stage-6 comparison, rather than a fresh hardware comparison after close.

Stage 7 remains blocked. Its claim property is not an acceleration test. Ordinary-user and sandbox access, userspace buffer allocation/mapping/VM binding/submission, shaders, Vulkan, Metal and native display takeover remain unverified. The [technical findings](../TECHNICAL-FINDINGS.md#8-display-power-and-xe3-core-bring-up) explain the progression, and the [attempt journal](fw13p-attempts-sequoia.md) retains my observations and the initial missing-helper refusal.

## Credits

The local sleepable SRCU compatibility route uses Lilu and DebugEnhancer. The core and diagnostic groundwork belongs to watchthelight, InSnh-Gd, Mubai and upstream contributors. [MeowXE3](https://github.com/cuadra8998/MeowXE3) and [ericek111's virtualized Xe3 work](https://github.com/ericek111/vibe-nullmoth-xe3-qemu) informed comparison research. Their different hardware and deployment results are not transferred to this Framework. See the [full credits](../TECHNICAL-FINDINGS.md#credits-and-references).
