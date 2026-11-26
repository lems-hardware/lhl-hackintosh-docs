# Graphics evidence through Attempt 181, November 19-25, 2026

I tested the Framework Laptop 13 Pro with Arc B390 8086:b082 revision 04 on Sequoia 15.8.3 24H241, Darwin 24.6.0 and OpenCore DEBUG 1.0.8. Graphics, media and display IP are 30.00. The existing display is a 2880 x 1920 firmware framebuffer with a 22118400-byte allocation. Accelerator attachment, application compute and offscreen graphics are verified through private Vulkan and Metal paths. Later tests verified bounded imported-memory cleanup, a short 2 GiB cumulative randomized asynchronous workload and twelve visible CAMetalLayer frames through the explicitly loaded private Metal device. Ordinary Metal discovery still returns no devices, and accelerated WindowServer remains unverified.

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
| 124 | 64 KiB native system-memory allocation and CPU mapping, two patterns with zero mismatches, cleanup passed. | CPU roundtrip, no userspace GPU execution yet. |
| 125 | Normal VM and binary syncobj create/destroy passed after buffer checks. | No VM binding or execution. |
| 126 | Bind/unbind of one buffer range completed through independent syncobjs, cleanup passed. | No execution queue or workload. |
| 127 | One empty userspace render batch completed, queue stayed unbanned and cleanup passed. | Completion signaling, no checked GPU data effect. |
| 128 | Private GPU store read back 0x1280cafe with both guards and batch words intact, queue health and cleanup passed. | Immediate store, no shader or rendering test. |
| 129 | Sequoia accelerator injection failed during vtable patching after missing BootKC graphics dependencies. Collector stopped at stage 0, I reported all working. | No initialization or stage-7 request in this run. Accelerator start was not reached. |
| 130 | Standalone SRCU fix, stages 1-6 and private GPU store passed. | One exact core profile tested. |
| 131-132 | Uninstalled build refused, exit 31. | No candidate produced. |
| 133 | Both diagnostic bundles failed FAT32 ownership and authentication. | Shared rejection, not an accelerator-specific diagnosis. |
| 134 | Invalid disk-image type stopped capture. | No diagnostic run. |
| 135 | Root-owned HFS+ image worked, authentication diagnostics still failed. | Ownership resolved, linking untested. |
| 136 | Uninstalled accelerator collection built and inspected. | No runtime load. |
| 137 | Collection linkage matches, original auxiliary copy verified. | Four existing entries identified. |
| 138 | Five-entry auxiliary candidate built and inspected, expected UUIDs retained. | No installation or attachment. |
| 139 | Candidate accepted at boot, exact accelerator UUID enumerated. Start-only returned code 71. | No accelerator loaded, stage 0 idle. |
| 140 | Original collection restored and missing recorded bundle path identified. | No load requested. |
| 141 | Explicit bundle and collection discovery both succeed. | Read-only image, installed hashes unchanged. |
| 142 | Two normal-repository runs reached approval prompt. | Rebuild refused pending approval, cleanup verified. |
| 143 | Approval generated accelerator collection, reboot required. | After removal original membership regenerated, new hash. Runtime startup untested. |
| 144 | Regenerated baseline membership and BootKC/SystemKC linkage verified. | Preserved separately from the byte-different original backup. |
| 145 | Normal approval and installed bundle discovery persisted across reboot. | Automatic module loading absent, manager lookup errors recorded. |
| 146 | Explicit start-only request returned zero and exact accelerator UUID loaded. | Stage 0 idle, no service attachment requested. |
| 147 | New core loaded, SRCU profile rejected its UUID. | Preflight stopped at stage 0, not a stage-7 failure. |
| 148 | Exact SRCU profile passed, all stages succeeded, NMXAccel attached. | Ten active-service observations, no rendering tested yet. |
| 149 | XGDR userspace client open and close returned zero. | Kernel independently confirmed client creation. |
| 150 | Private Intel ICD enumerated B390, advertised Vulkan 1.4.363. | Process-local loading, not API conformance. |
| 151 | Logical device and family 0 queue created, then destroyed. | No optional features or extensions enabled. |
| 152 | Vulkan buffer fill and fence completed, all 1024 words matched. | One 4 KiB transfer workload, no application shader. |
| 153 | Application compute shader passed all 1024 index-dependent outputs. | No graphics draw or presentation in this run. |
| 154 | Vertex/fragment draw passed all 4096 RGBA pixels and saved RGB image verification. | Offscreen rendering, no Metal, presentation or desktop acceleration. |
| 155 | Metal runtime and Vulkan host-import capability audit passed, CLT installation confirmed. | Capability queries, no Metal plugin load. |
| 156 | Private Metal plugin compilation stopped at omitted vulkan/vulkan.h. | Packaging failure, no binary produced. |
| 157 | Complete headers allowed a Sequoia-targeted plugin build, local signing and verification. | Build only, not loaded. |
| 158 | Exact private plugin loaded and registered its expected Objective-C classes. | No Metal device created yet. |
| 159 | Private Metal device initialized with registry identity matching NMXAccel. | No workload submitted. |
| 160 | Metal buffer fill matched all 4352 bytes including guards. | Transfer only, no application shader. |
| 161 | Metal source compilation and translated compute matched all 1024 outputs and guards. | Private compute workload, no presentation. |
| 162 | Metal vertex/fragment rendering matched all 4096 pixels, guards and saved RGB image. | Offscreen graphics, no presentation. |
| 163 | IOSurface host import failed at VM-bind errno 45, later submission failed. | Pixel checks not reached, preview skipped. |
| 164 | Disabling host import for the isolated comparison yielded matching surface/image pixels. | Diagnostic copy path, not the final fix. |
| 165 | Exact-core tracing linked renderer HMM -45 directly to VM-bind -45. | Missing population stub confirmed. |
| 166 | Actual four-page IOSurface range prepared, inspected and released successfully. | Temporary wiring only, original HMM error preserved. |
| 167 | Four real pages produced four SG/DMA entries covering 16384 bytes, ordered teardown returned. | No mapping published, original HMM error preserved. |
| 168 | Retained host import, exact texture/surface pixels and final four-page release to active=0 passed. | Bounded normal lifetime, direct presentation and broader stability unverified. |
| 169 | Four imported commits returned before gate release, then no callbacks within ten seconds. Cleanup released all sixteen pages. | No data verification reached. Timeout cause remains unresolved. |
| 170 | All eight ordinary/imported, one/two-queue and gated/ungated retained-reference controls passed, 32 commands total. | Small workloads with application references retained. |
| 171 | All eight retained/early-release controls passed, 32 commands with independent readback and imported-surface checks. | Extra GPU readback copy differs from Attempt 169. Original timeout not explained. |
| 172 | Ordinary retained fill-only case exited 142 before commit/gate checkpoints. | Alarm included initialization. No HMM calls, no useful worker sample. |
| 173 | Instrumented ordinary fill-only replay passed. Samples caught logger I/O during slow initialization. | Logging latency contribution, not proof of the earlier timeout causes. |
| 174 | All four log-destination controls passed. Local logging was faster in both comparisons. | Earlier timeouts remain causally unresolved. |
| 175 | Dynamic import storage passed sixteen early-release fill jobs, checked bytes/guards and all sixty-four page releases. | Storage device also changed. Bounds remain 64 pages/import and sixteen active mappings. |
| 176 | Sixteen asynchronous imported compute jobs passed, with all 63488 transformed words independently verified. | Default sizing, two queues, no general invalidation qualification. |
| 177 | Stage 5 returned EBUSY before rendering. Exact binary exposed a watchdog ownership retirement race. | Ownership flag not sampled at rejection. No after-collection hashes in this early-stop capture. |
| 178 | Scoped watchdog retirement handshake passed all stages. Sixteen queued frames passed pixel, guard and cleanup checks. | Original request called once, no watchdog bypass. Offscreen rendering only. |
| 179 | Initial eight ungated jobs made out-of-order callback progress, but verifier lifetime check stopped after one verified job. | Compiled verifier established autorelease-owner retention. Not a 2 GiB pass. |
| 180 | All 802 randomized ungated jobs passed exactly 2 GiB cumulative payload checks, with all 360 mappings freed. | 4.253-second scope, not 2 GiB simultaneous residency or extended stability. Independent reference hashes, successful raw outputs not archived. |
| 181 | Twelve CAMetalLayer frames rendered visibly through the explicit private device, with all callbacks and cleanup passing. | Native Metal discovery returned zero devices. Presented timestamps were zero. No accelerated WindowServer claim. |

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
| B082 metadata-only, original cap 7, Attempt 129 | 3bdf09bf724a9a9824efbb7ba4c8066b33e195c6abd1eb9c1a1f6a0354a797ea | CC829CA0-00CE-581C-86D7-0AE03022A333 |
| B082 metadata-only, manual cap 7, Attempts 147-154 | 281e42700bf1321793856798d243f101c43daf95ade762255e025ffd8ede4f79 | 711C3CE5-C9AE-5311-AFFC-4E6AE247F529 |

The core derives from [InSnh-Gd/WTLIntelGfx 5ffc79fb9e53eb820899edd644b83681bea3aa39](https://github.com/InSnh-Gd/WTLIntelGfx/tree/5ffc79fb9e53eb820899edd644b83681bea3aa39), with [Mubai's changes at 180a42fafd1e1b8116bdc1f0312dec54be3812c5](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/tree/180a42fafd1e1b8116bdc1f0312dec54be3812c5). It reports version 0.0.47 and policy ptl-worker-heartbeat-v24. The local variants are binary adaptations, not source rebuilds. The capped variants change two machine-code bytes from the original plus UUID and existing code-page hash metadata. Attempt 129 restores the original stage-7 ceiling, leaving one selector byte changed plus UUID and hash metadata. Both original ad-hoc code directories were refreshed and all 1750 page hashes verified. That consistency is not Apple signing trust or a qualified standalone installation.

The B082 branch selection changes the original B0A0 comparison at file offset 1868112. The manual stage ceiling changes the comparison at file offset 1857250. Both signatures were unique in the exact original binary and checked against disassembly. The first does not change any hardware ID. Attempt 120 retained the Attempt 119 core, and Attempt 123 retained the Attempt 122 core. The second adaptation retains the original bounds check, limiting requests progressively to 2, 3, 4, 5 and 6. Attempts 123-128 reuse the exact Attempt 122 EFI. Attempt 129 adds the accelerator, updates the boot marker and changes the core/SRCU accepted identity while keeping the SRCU algorithm unchanged. Automatic startup remained disabled at stage 0. Runtime OpenCore patch entries were removed after Attempts 115-116, and loaded UUID checks identified the adapted executable before each request.

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
| 124 | `attempt-124-20261120-131755-659.tgz` | `6089cda6f63419dd53ee0386bd8eca0cbece34ea1095315ec22f82b1e6885f0d` |
| 125 | `attempt-125-20261120-133229-756.tgz` | `9b0bca7061c451eaaff63796aa0dc6ee07ba0451488754c6724dd96885297e45` |
| 126 | `attempt-126-20261120-140057-760.tgz` | `85e8e7b198ba5099b6c831a266d8da4bca59e3f8538586ad93c36f92f047b4b2` |
| 127 | `attempt-127-20261120-142616-680.tgz` | `6407112aa9c4713f84734545782f445e6750ea535bae3759668b5d3453bd82bd` |
| 128 | `attempt-128-20261120-144626-670.tgz` | `da0d8e3e3d7f73ff29f0282ad4a8637ec27b2a328b6ca8bdecd2288ea96ffed8` |
| 129 | `attempt-129-20261120-151303-679.tgz` | `974b9c669790f0b5a6bb6f428609dcd1de1f2d34f17a645ce7e9595c04fdb407` |
| 130 | `attempt-130-20261120-213522-989.tgz` | `1291f84716b5f11fbcecb8c89151e91b5af92c41f7ee749757d39c12e3f31e49` |
| 131 | `attempt-131-20261120-220224-656.tgz` | `aca15246460e7eb4a8fdbb1b1631e176090e4703461be2bd8548448a8d3b6e30` |
| 132 | `attempt-132-20261121-123226-663.tgz` | `6f688f075f525b827398c40bc069e722c1f4b6a6d877209eb06d3248a3b590bb` |
| 133 | `attempt-133-20261121-141217-659.tgz` | `f69b5a43569a10479526233f112b6e84bf0f08ff918e4510fa5b17e9734412ab` |
| 134 | `attempt-134-20261121-142830-681.tgz` | `2be4b5751c84636968443bf57b061750847b3aaa71d961c9313d2dbb4cd6b669` |
| 135 | `attempt-135-20261121-144421-644.tgz` | `bd23756c1ca2c652e2f5131b9821f0f9709208a475249e1b9b38599ac96e2952` |
| 136 | `attempt-136-20261121-145740-686.tgz` | `0aeb726e17be76d0671b0c62aaf84187cd898f0805c51a1fecf510a7c4b7a6df` |
| 137 | `attempt-137-20261121-151020-678.tgz` | `d23e6379f8e53ac3b7843ba8445a5e088c95d363fc201f318a52d6e1d64d8bae` |
| 138 | `attempt-138-20261121-154101-653.tgz` | `2070c02058dcab4ba3e60f8cf66513d8cde67f383bc18fc17aae9aee3ac7ecb2` |
| 139 | `attempt-139-20261121-224353-667.tgz` | `255ffe020651cdbd34ccb9668f230aeebc3f2b133f04f4f7244a4c0f0bc8136a` |
| 140 | `attempt-140-20261121-225806-689.tgz` | `72e802cc4343fe8506985553eb7ca8be7f5fde48dbd6dae503464cc9062f6a4c` |
| 141 | `attempt-141-20261121-231631-665.tgz` | `3659cff1b68924ee33e206d9e357d15149d12ae0390ab2150c25a968ea0a07c7` |
| 142 | `attempt-142-20261121-233126-662.tgz` | `7a25aebf230e1d3075a6fda27d5388b910c6e1b9802746401786b80caa0c4dcb` |
| 142 | `attempt-142-20261121-233222-855.tgz` | `90f9b8a97f9a63ef7e03353c22f05bc42f9fbe42ca346ccfe17f73dbed32914f` |
| 143 | `attempt-143-20261121-235858-681.tgz` | `54e05e32839a1a1fbbdce3ea203b78257e6f3dccd48d6f9a0962854dffb7bd08` |
| 144 | `attempt-144-20261122-124700-634.tgz` | `94640c5c23b056f3eedccf60c37db641849dc6ec0201dae212e1e57858e6a0bd` |
| 145 | `attempt-145-postboot-20261122-130122-664.tgz` | `d8574a1feab7bf53a745890cb1558ab587cb6aacf8d8724b7af43c3853961a3a` |
| 145 | `attempt-145-prepare-20261122-125741-733.tgz` | `c7b3f654eb90d453e1837e8a69032cc8efe6bc42e4a8ba5e618256ae2f040ba4` |
| 146 | `attempt-146-20261122-131341-675.tgz` | `fd8143c11bccf33e9d44fe67e2d67b9a7b5ee5645c65bedeac154a7d2841fb40` |
| 147 | `attempt-147-20261122-135029-698.tgz` | `8d5b0ad22a732c2ab92d406db9f6498180be8f39f73fc96dd9af95ddf94ab491` |
| 147 | `attempt-147-20261122-135646-660.tgz` | `a17fabf0abd7aa9f916d102ff200d7cef6ec4c0ae0a986568d9eb647d0317290` |
| 148 | `attempt-148-20261122-141736-673.tgz` | `1535c2a2ea66848ea39e961b75ff3bbcb91c8f2e3e72a999a08c48c698106c7d` |
| 149 | `attempt-149-20261122-145550-677.tgz` | `331178144ced1f4db49eeaf4cda7a659505a9d617d99180dd22083c381062aa2` |
| 150 | `attempt-150-20261122-150759-672.tgz` | `a57c46675e5566639806458241d52316b150041b9d8579af1cbebc48ee7c04a2` |
| 151 | `attempt-151-20261122-152040-746.tgz` | `de51d264b2a8749da6a88737b1125b611bc06b1befe6d8754a81c75fc534012a` |
| 152 | `attempt-152-20261122-153522-695.tgz` | `055e4c52cb53cbe17e7cbca84554a78536e3e14bf9996acab70b0b913ba6e64b` |
| 153 | `attempt-153-20261122-154723-658.tgz` | `51de88cd68e30c28fad3be75e6f7eefdb3726e3d091e3dc46636f46b3621b317` |
| 154 | `attempt-154-20261122-160957-693.tgz` | `da73b02a8811fb9ce9eaf809d6d441ff3838957dade20add9e0ac5ec794191e6` |

Attempts 115 and 116 retain their original as-tested packages and capture filenames. Renumbering did not rewrite source evidence. Attempts 117 and 118 each passed matching OpenCore 1.0.8 validation, all 71 staged EFI hashes, file-reference checks and ZIP integrity checks. The loaded UUID establishes the selected core variant at runtime, while it is not an independent hash verification of every mounted EFI file.

### Metal and host-import capture digests

| Original archive | SHA-256 |
| --- | --- |
| attempt-155-20261123-183929-659.tgz | 88fea788b3db90dd90e2f9a5da3fe0e3599b1900890ed9e1b0c3dff039dfa237 |
| attempt-156-20261123-203851-673.tgz | ecee858b8e22a370325266337d0132bebe4881944d4da46819ffb0cbdede0d6e |
| attempt-157-20261123-221759-645.tgz | febd5fed3e0718f70e2a733118cfcb64acb9d9c5c4a594c1b29b8913d9d0250e |
| attempt-158-20261123-223725-674.tgz | 969985756d990f3710cbadff3255ac5a1a74fbc7561da38be04d28b762f10218 |
| attempt-159-20261124-122043-682.tgz | 84cfc1155d56a382f8a297b23dd124ac1925774840ef5e1507fa6c1c1adbb51b |
| attempt-160-20261124-123550-676.tgz | a30c83f1e24c4474fdd6cc80a2e3201980bcaf5a21723d82a07a4e0fe3f7d2a2 |
| attempt-161-20261124-125001-688.tgz | 03121b3cca717b867f0ef5f66370c60b2ca4c6ad055746a769436b52de0072e3 |
| attempt-162-20261124-130158-674.tgz | 5c4e103aba419151d74a2c807d4b8be457e9b9a9594bfbe6994af8e80c696982 |
| attempt-163-20261124-131537-691.tgz | 2e2f5507ee17e9e6990f97b4ff10bee1a41354f9fbfc9b2344e5e9c6c476f682 |
| attempt-164-20261124-132708-654.tgz | 54d5feec41984ddf89dd6a202bf820b7c0cc075b00eff492a8379434027ad3c5 |
| attempt-165-20261124-161356-654.tgz | 0fcd3fff3affd444448c2cb1c0c5c63d409809cbaf0199e368cd74f5b8014e17 |
| attempt-166-20261124-164145-679.tgz | f5eb8ad4200d4eb260abe5efbe55b92f33e1020d5be10cb1e2276485979c87cc |
| attempt-167-20261124-181807-675.tgz | f23102a41397596dc453b38235ce45e9f7621b952214aa6bbfcb3e0df0f55c9f |
| attempt-168-20261124-183738-674.tgz | 1d0999ab8ab49a64cd9f5ba45593bfcba8ed941b6aa7c035b35f359a7948c908 |

## Initialization, SRCU and userspace evidence

The Attempt 119 stack was verified against all 16 XeGfx frames in the matching core. drm_dev_enter held drm_unplug_srcu while GGTT removal reached a fence wait. Its SRCU reader called ordinary RCU and pinned preemption, causing the blocking-with-preemption-disabled panic. Attempt 120 kept the core unchanged and added three exact-UUID/signature-checked Lilu routes for that DRM domain. Readers could sleep, while two reader epochs and serialized grace periods retained lifetime protection. Other domains used the original functions. This supports the diagnosis without qualifying general SRCU or hot-unplug behavior.

Attempt 120 reported two GTs, six engines, one MSI vector and GuC 70.72.1 running with submission enabled on both GTs. Attempt 121's six empty-batch fence completions had matching sequences and zero errors. Attempt 122's six distinct store values all matched CPU readback with zero errors. These writes used separate diagnostic allocations, not the panel buffer. Both tests finished idle with an empty environment, unchanged display metadata and a frame-counter advance of 12. The source and exact deployed store implementation differ, so stage 6 was reviewed from the binary rather than assumed from the older source.

Attempt 123 reused the exact Attempt 122 EFI and returned driver name xe, version 1.1.0 and description Intel Xe Graphics through command 0xc0406400. The native helper opened /dev/dri/renderD128, performed one VERSION ioctl and closed, with zero errors and exit status 0. It ran as root. The limited scan found IOAcceleratorFamily2 485.2 and IOGraphicsFamily 599, but no accelerator service in the queried classes or custom bundles in the scanned paths. Final display properties retained the stage-6 comparison, rather than a fresh hardware comparison after close.

Attempts 124-128 then established a 65536-byte WB buffer CPU roundtrip, VM/syncobj lifecycle, bind/unbind completion, width-1 render-queue execution and one checked GPU store. The private PPGTT store targeted VA 0x1000400 inside the buffer bound at 0x1000000, using PAT 2. Its initial 0xdeadbeef was replaced by 0x1280cafe, both guard words stayed zero and the five command dwords were unchanged. Queue BAN stayed zero and all explicit cleanup calls succeeded. The final properties retain the preceding stage-6 metadata check. This does not qualify general coherency, performance or graphics stability.

Attempt 129's source-built accelerator has SHA-256 445ac1eab9f471bb74c062b1411e83cd049a5f4e7b01894653cfdeae4d34ca6d and UUID D63BCE8E-236F-3178-BB25-D30D3502B8A1. It was built for a macOS 15.0 compile target without NM_TAHOE, with corrected named enums and no scheduled NVIDIA temperature reader on Intel. It was not loaded. OpenCore could not access the real graphics-family superclasses during early prelinking and rejected accelerator vtable patching with Invalid Parameter. The exact failed vtable is not identified. The core and SRCU route loaded, but the collector stopped at stage 0 with all results zero. Empty accelerator registry captures and the saved loaded-driver inventory corroborate the refusal. I reported all working.

At Attempt 129, Stage 7 remained untested because injection failed before initialization or claim. Its claim property alone is not an acceleration test. Later startup, attachment and Vulkan results are recorded below. The [technical findings](../TECHNICAL-FINDINGS.md#8-display-power-and-xe3-core-bring-up) explain the progression, and the [attempt journal](fw13p-attempts-sequoia.md) retains my observations and the initial missing-helper refusal.

## Standalone SRCU and collection build evidence

The standalone SRCU plugin preserved the verified GPU command path in Attempt 130. Attempts 131-135 isolated collection input selection, FAT32 ownership and diagnostic authentication failures. Attempts 136 and 138 successfully built uninstalled auxiliary collections against the exact installed BootKC and SystemKC, with Attempt 138 retaining all four existing entries plus XeGfxAccel 0.9. Attempt 137 preserved a verified copy of the original collection and confirmed matching dependency UUIDs. The installed collections remained unchanged. Runtime loading, stage-7 attachment, rendering and Metal remain unverified.

Attempt 130 loaded XeGfxSRCUFix UUID `478F6C94-E3F8-3280-9178-52D5822A10B6`. The Attempt 136 candidate SHA-256 is `5a0c8f980d41274ba206d1f9bc4c204e214f6806f18996527faabb46ea7b3ae6`. Attempt 138 candidate SHA-256 is `f8b0a6ff9a393c3d979238526563580964cb53462a23cfa675334afeefc1bfdf`. Both builds were uninstalled. Raw collection files and capture archives remain private.

## Runtime loader and approval evidence

Attempt 139 booted the Attempt 138 candidate and enumerated the accelerator UUID, but the start-only request returned 71. Attempt 140 confirmed original restoration and identified the missing bundle at the temporary build path. Attempt 141 showed that both the exact bundle and the linked entry were discoverable in explicit repository inspection. No startup was established.

Attempt 142 reproduced the normal approval prompt in two runs. The manager refused an automatic library rebuild pending user approval. Normal lookup found the experimental bundle, and both runs removed it and preserved the original collection hashes. Attempt 143 advanced to a system-generated collection UUID `CD497DA6-0EC5-D224-CD66-90579A5797F0`, SHA-256 `fc14f094735e82ba01778f2e2b8aca7f3d791f62c953c38f84bb2c03a7d3f0fc`, containing the expected accelerator UUID and three legacy entries. The daemon reported that a reboot was required. Separate diagnostic authentication errors persisted. The accelerator was never in the loaded inventory. Stage 0 remained idle.

After removal of the temporary bundle, my inventory showed a regenerated auxiliary collection UUID `5F92863E-7A74-89F9-95BC-3A030E33C5B7`, SHA-256 `adaf398a69e8f9c8c5bf3c51de58f3f8adf207cf7bc413aeb630096aba76eb52`. It contains all four original entries and the original executable UUIDs, with no accelerator. Attempt 144 subsequently verified its expected BootKC/SystemKC linkage and preserved the regenerated baseline. Its bytes still differ from the original backup. Recovery rollback copy and rename checks passed. Raw collections and approval logs remain private.

## Approved startup and Stage 7 attachment

Attempt 145 retained the verified accelerator at /Library/Extensions/XeGfxAccel.kext through normal approval and reboot. The resulting auxiliary collection has UUID 8BE70CE0-5B77-A3C0-321A-4A0BC4E254E5 and SHA-256 02bbdcf14c759091eb6805ca00e67e5abd41be704300a36c82093de605feccc2. It includes XeGfxAccel 0.9, both HighPoint drivers and AppleMobileDevice, with the expected collection linkage. My older installed Stage 2 core is omitted. The active core continues to come from EFI.

Attempt 146's explicit start-only request returned zero and loaded XeGfxAccel UUID D63BCE8E-236F-3178-BB25-D30D3502B8A1. The installed executable SHA-256 is 5a2c2711ceb77a5fb5dcadfe3d09cb0e19e8ae5971478ce65d77a19725af0056. This identifies the later installed artifact separately from the earlier Attempt 129 prelink candidate. Earlier manager lookup-table errors still appear, but did not prevent the explicit load.

Attempt 147 raised the manual ceiling to 7 while retaining automatic stage 0 and removed -nvoff. The old SRCU plugin rejected the new core UUID before any stage request. Attempt 148 added the exact core profile and retained independent header, offset and signature checks. Its rebuilt plugin UUID is AA00C4EA-0EC6-359F-9544-858415F15C1C. The collector passed stages 1-6, including the private userspace store, before requesting stage 7.

The final result array was [0, 1, 1, 1, 1, 1, 1, 1]. The core published its claim, NMXAccel started on B082, IOGraphicsAccelerator2::start returned 1 and all 32 user-client carriers became ready. The same accelerator remained registered, matched and active across ten samples. Attempt 149 opened client type 0x58474452, XGDR, and closed it successfully. The kernel independently logged client creation. No external method or ioctl was sent through that client in the open/close test.

## Private Vulkan workload evidence

Attempts 150-154 use the unchanged Attempt 148 EFI and unmodified libraries from MeowXE3 1.0.12, staged privately on FILES. The native helper preloads libdrm globally before loading the Intel ICD, preserving the original library install names. It calls the ICD interface directly and runs as my normal console user in a clean environment. These tests do not install a standard Vulkan loader or register a Metal plugin with macOS.

| Private library | SHA-256 |
| --- | --- |
| libdrm.2.dylib | 442f71fbd6d64a7f0cf36a4bb470a8dcd1288abaa15fdba49a576cac24893e82 |
| libvulkan_intel.dylib | 67954e43cb2b805ad6397b979c453bdedf5d5cb07a0302ae95917f2253e1f92f |

ICD interface negotiation succeeded at version 7. The driver enumerated one integrated Intel Arc B390 (PTL), 8086:b082, advertising API 1.4.363. That is an advertised property, not conformance certification. Attempt 151 reported three queue families with one queue each, selected family 0 and created a logical device with no optional features or extensions.

Attempt 152 allocated a 4096-byte buffer using host-visible coherent memory type 1, initialized a sentinel, submitted vkCmdFillBuffer and applied a transfer-to-host barrier. The five-second fence wait succeeded. All 1024 words matched 0x1280cafe. Attempt 153 created a compute shader and dispatched 16 workgroups of 64 threads. All 1024 words matched 0x1280cafe XOR (i multiplied by 0x01010101), with unsigned 32-bit arithmetic. The saved index 0, 1 and 1023 values were 0x1280cafe, 0x1381cbff and 0x1183c801.

Attempt 154 created a 64x64 optimal-tiled RGBA8 image and a vertex/fragment graphics pipeline. A red clear was replaced by the full-image checkerboard draw. Submission, fence completion and image-to-buffer copy succeeded, and all 4096 RGBA pixels matched, including opaque alpha. The helper saved the returned RGB bytes as a PPM. Independent parsing confirmed every RGB pixel and exactly 2048 green and 2048 magenta pixels. PPM does not retain alpha, so its alpha evidence comes from the helper's full RGBA comparison.

Explicit resource, device and instance cleanup completed in these successful runs. NMXAccel remained active, the core ended idle at stage 7 with an empty environment, installed collection hashes stayed unchanged and I reported display/input working. These are bounded workloads in captured runs, not arbitrary-shader, full-API, repeated-use or long-duration qualification. No display surface, swapchain, presentation, Metal or accelerated WindowServer was tested. The firmware display remains in use.

## Offscreen rendering artifact

![Attempt 154 GPU checkerboard readback](graphics-artifacts/attempt-154-render.png)

This PNG is a lossless conversion of the actual 64x64 GPU readback saved in Attempt 154. It is not a CPU-generated reference image or a desktop screenshot. The original PPM SHA-256 is fbd1c466141bce0c727bf3e515fdbc5d02291f2f1e3932ca5ea3811a3e97b5ce. The private capture includes the original PPM, verification output and independent analysis.


## Private Metal and IOSurface host-import evidence

Attempts 157-162 used a source-built private NVMTLDriver targeting Sequoia. The plugin has SHA-256 cc873490b968d2213ed36bf8916721af6767469bcb89378cbd267f1ec16439d2 and UUID 5DEB2270-A748-3E2C-8650-2AF107B95BDE. Compiler-runtime components were preserved from the same original release archive. The plugin was loaded only in the helper process, not advertised globally. Transfer, source-compiled compute and offscreen graphics each passed their independent output/guard checks.

The IOSurface import failure was traced to the exact core's unsupported HMM population stub. Temporary wiring and SG/DMA probes qualified the real memory and helper ABI before the final implementation. The successful mapping plugin in Attempt 168 is XeGfxUserptrTrace 0.4.0, UUID 18B1BD0A-D6E7-3CC0-95F9-FBF84DC48A32. It retains page/SG/DMA ownership and writes only the binary-verified SG pointer and notifier sequence, with separate unmap and final-free paths.

The renderer imported 16384 bytes across four pages with host import enabled. HMM and all 20 logged binds returned zero. All 4096 texture pixels, 512 guard bytes and CPU-visible surface bytes matched. The saved BGRA surface and PPM were independently verified as exactly 2048 red and 2048 cyan pixels. The helper and separate CPU preview exited zero. DMA unmap and SG free returned, and USERPTR_FREED reported released=4 active=0 within the five-second cleanup snapshot. The collector's manual-results field was blank, but I confirmed alongside the archive that it worked and everything remained working.

The implementation is limited to 64 pages per mapping and 16 simultaneous mappings on the exact tested core. The runtime's private render image is copied into imported surface memory. This is not native cross-process image sharing or direct presentation. General invalidation semantics, larger buffers, broader concurrency and long-term stability are unqualified. Installed collection hashes remained unchanged.

### Host-import rendering artifact

![Attempt 168 verified IOSurface output](graphics-artifacts/attempt-168-render.png)

Actual RGB readback from the successful host-import run, enlarged with nearest-neighbor scaling. This image and the separate AppKit CPU preview are evidence of checked output, not accelerated WindowServer.

## Asynchronous execution, stress and presentation evidence

The [technical findings](../TECHNICAL-FINDINGS.md#attempts-169-174---asynchronous-lifetime-controls-and-logging-latency) explain the control experiments, exact-core handoff race and verifier-lifetime correction. The [journal](fw13p-attempts-sequoia.md) preserves the original first-person entries through Attempt 181. These are confirmed captures, including failed diagnostics, rather than a claim that every workload or application is supported.

### Later build identities and bounds

| Component | Version | Loaded UUID | Executable SHA-256 |
| --- | --- | --- | --- |
| XeGfx core | 0.0.47 | 711C3CE5-C9AE-5311-AFFC-4E6AE247F529 | 281e42700bf1321793856798d243f101c43daf95ade762255e025ffd8ede4f79 |
| XeGfxAccel | 0.9 | D63BCE8E-236F-3178-BB25-D30D3502B8A1 | 5a2c2711ceb77a5fb5dcadfe3d09cb0e19e8ae5971478ce65d77a19725af0056 |
| XeGfxUserptrTrace, from Attempt 175 | 0.5.0 | CD4C9B4D-8121-3DEF-8741-607790E0F39B | 6eeba835e7bbef27938917ff36806ba335d5c42e95e66ad884b800e85f35d48d |
| XeGfxStageHandoffFix, from Attempt 178 | 0.1.0 | A73E31C0-C94F-3FF7-A298-D70C0CE4FC36 | 0efad57a2ac5a12b744469d48b95e64b3ae393c99a188ee50a8428154678b9a5 |

The private Metal driver used for the later tests has UUID 5DEB2270-A748-3E2C-8650-2AF107B95BDE and executable SHA-256 cc873490b968d2213ed36bf8916721af6767469bcb89378cbd267f1ec16439d2. Dynamic import storage remains limited to 64 pages per mapping and sixteen active mappings. The ordinary runtime render wait was retained, including during the ungated stress test. Removing artificial event gates did not disable its safeguards.

Attempt 180 checked all 802 job outputs and guards, then retired every resource. A separate reference reproduced all saved job hashes. Successful raw outputs were not archived. The exact 2 GiB total was cumulative over about 4.253 seconds, with peak useful payload 80756736 bytes. Kernel import accounting ended at zero after 360 mapping frees and 9884 page releases. The configured successful-bind trace logged only its 256-call prefix.

Attempt 181 separately confirmed that ordinary Metal discovery returned nil/zero devices. Explicit private-device loading nevertheless produced a visible twelve-frame CAMetalLayer window without a CPU preview fallback. All GPU completions, presented callbacks and worker returns passed. Both imported drawable mappings were freed with thirty-two page holds released. All presented timestamps were zero. Visibility was manually observed, without automated drawable pixel readback, and system-wide graphics acceleration remains unverified.

### Original asynchronous and presentation capture digests

| Original archive | SHA-256 |
| --- | --- |
| attempt-169-20261124-194650-662.tgz | 5d0083fec70bda8d0791d4b1e9195806fb8969ba2c029fda1727e24d1edc85fa |
| attempt-170-20261124-200353-685.tgz | ae1c245cc4369d463f4cf83d5e4e617ffbaef90f4950ed9aed3d85fed229492f |
| attempt-171-20261124-202533-677.tgz | e2a3449430495ec454722d688dd2555fd7b5c08ff05d8a157b97e95632356e11 |
| attempt-172-20261124-210413-659.tgz | 72b065b0f903fad43ab6b509228a8a92c079b72ffa3f7093a0edcbe046196e55 |
| attempt-173-20261124-214026-720.tgz | 2e2ca739a7cd4a92be64829378fdf47bb7ab128e976914f360f2039906fd43e7 |
| attempt-174-20261125-104619-678.tgz | d2c82e37917f05a17dcb4ccf093f2d648ce5a39ecd0315cbf5c81942fd24461c |
| attempt-175-20261125-115943-752.tgz | c0acf24a2ec8b4a96000fce50741b0d69bc278344ee4fe63e3d10bd094faa948 |
| attempt-176-20261125-122518-671.tgz | cd053da4c96a57e25435212b4a532d8429d91ad11edd6c3ceade1d58518c474f |
| attempt-177-20261125-132434-682.tgz | b754836a8cbebeddb82d1dad866efddee4dfa3d8c6e7212cc7a7f60514a8f078 |
| attempt-178-20261125-134151-641.tgz | 4325450777e7acf48309e83f43befc50979baa7b2e232e1b40e941ed3819c49a |
| attempt-179-20261125-223413-766.tgz | ab5640008b7afb4388367482825784cc24757704a4943b29a80ef665e1088fcb |
| attempt-180-20261125-224442-655.tgz | 0329f6fd9ac95c55ca5724e1ab459e322cc0da524fb354d2c51db1eda77c5318 |
| attempt-181-20261125-230943-673.tgz | ba03fc417f92b97657257f58356198f63084dbcac610a4e06115857548dd7499 |

## Credits

The local sleepable SRCU compatibility route uses Lilu and DebugEnhancer. The core and diagnostic groundwork belongs to watchthelight, InSnh-Gd, Mubai and upstream contributors. NullMoth provides the shared accelerator foundation, and NVIDIA open-gpu-kernel-modules supplies build headers. [MeowXE3](https://github.com/cuadra8998/MeowXE3) supplied the original 1.0.12 packaged Intel Vulkan ICD and libdrm used unchanged here, with credit to Mesa/ANV, Intel, libdrm and the NullMoth port contributors. Khronos supplies the Vulkan headers, glslang and SPIR-V validation tools used in my helpers. [ericek111's virtualized Xe3 work](https://github.com/ericek111/vibe-nullmoth-xe3-qemu) informed comparison research. [Mubai's pinned userptr work](https://github.com/yangyishuo1124/Mubai-s-Hackintosh-Logs/tree/f7164ac366a74dfcc641c2098c9908affd26a462) informed the later dynamic ownership comparison. Apple documents [CAMetalLayer](https://developer.apple.com/documentation/QuartzCore/CAMetalLayer), used for the presentation helper. Their different hardware and deployment results are not transferred to this Framework. See the [full credits](../TECHNICAL-FINDINGS.md#credits-and-references). Third-party licenses remain with their respective projects.
