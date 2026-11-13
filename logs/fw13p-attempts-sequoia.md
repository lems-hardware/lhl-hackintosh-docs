attempt 1 - Initial boot with DevirtualiseMmio enabled and SetupVirtualMap disabled. Kept the recommended memory mapping settings and didn't apply any CPU spoofing or MSR patches yet. OpenCore successfully loaded the kernel and injected both kexts, but stopped logging at EXITBS:START after around 198 seconds.

attempt 2 - Disabled DevirtualiseMmio to see if MMIO mapping was causing the issue. Everything loaded normally again, but still stopped at the same line. Took around 108 seconds this time.

attempt 3 - Enabled SetupVirtualMap while keeping DevirtualiseMmio disabled, thinking it might be something related to UEFI virtual memory mapping. No difference, same result after around 100 seconds. Also accidentally booted Windows during one of the tests, so that log wasn't useful.

attempt 4 - Restored SetupVirtualMap to false and enabled AppleXcpmCfgLock to work around the locked MSR 0xE2. OpenCore reported that the kernel patches succeeded, but the boot still ended at the same point. Around 98 seconds.

attempt 5 - Tried spoofing my Panther Lake CPU as a Comet Lake processor using Cpuid1Data and Cpuid1Mask. I figured macOS might be rejecting the newer CPU since Panther Lake is completely unsupported. The CPUID patch reported success, but there was still no visible progress. Around 99 seconds.

attempt 6 - Started experimenting with the memory mapping settings. Enabled EnableWriteUnprotector and disabled RebuildAppleMemoryMap and SyncRuntimePermissions. Also had ProvideCurrentCpuInfo enabled, which patched some CPU topology information. OpenCore reported successful patches, including rewriting MSR 0x35, but once again stopped at EXITBS:START after around 99 seconds.

attempt 7 - Restored the original memory mapping settings while keeping the CPUID spoof, AppleXcpmCfgLock, and ProvideCurrentCpuInfo enabled. Wanted to see if the CPU patches would work better with the original memory configuration. Still nothing, same line after around 102 seconds.

attempt 8 - Enabled ForceExitBootServices to see if the firmware was failing to exit UEFI boot services properly. OpenCore confirmed the setting was active, but the log still ended at EXITBS:START, this time after around 62 seconds. Not sure if that actually means ExitBootServices is failing though, since Apple's logging changes around that point.

attempt 9 - Enabled AppleXcpmExtraMsrs while keeping the CPU spoof, kexts, and memory settings the same. OpenCore reported that the patch succeeded, but there was still no visible difference.

attempt 10 - Added a DMAR table deletion while keeping ExtraMsrs enabled. OpenCore confirmed that it deleted the 192-byte table, but it still looked stuck at the same point.

attempt 11 - Disabled VT-d in the BIOS, and finally got actual macOS kernel text past EXITBS. Comparing the Linux ACPI dumps before and after showed that the firmware changed the CPU entries from x2APIC to classic Local APIC entries, and I didn't add a converted APIC table since the BIOS change had already changed the format. Still froze at “[AppleLockdownMode - LDMShouldEnforceParity] restoreOS check passed,” but this was actual progress. Also figured out that the OpenCore logs stop at EXITBS even when there is kernel text past there. The later stopping points have to be recorded from the screen. The times in the earlier attempts came from the logs, not from measuring how long I waited after the last line.

attempt 12 - Added cpus=1 to see if CPU startup or topology was the next issue. Kept VT-d disabled and left the rest of the EFI alone. Still stopped at restoreOS check passed with no visible panic. The USB became read-only around this point. Linux found corruption in the FAT filesystem's trash folder and made it read-only. Backed up the EFI and logs, unmounted it, and repaired it with fsck.

attempt 13 - Added a CPU compatibility SSDT after looking at this machine's ACPI tables. The firmware defines the CPUs as ACPI0007 Device objects instead of legacy Processor objects. Used OpenCore's SSDT-PLUG-ALT approach, but matched the added Processor objects to the 16 enabled CPU IDs from my own APIC table. Kept cpus=1, the CPU spoof, DMAR deletion, ExtraMsrs, kexts, and memory settings unchanged. This got further, as the kernel showed AppleACPICPU listing processor IDs 0 through 15, then PCI configuration finishing and DSMOS arriving. There are some NVMe assertion messages, but the driver then says “Successfully initialized NVMe drive,” so I can't say those messages are what stopped it. APFS loads, and the last visible line is near ktriage registration.

attempt 14 - Removed cpus=1 from the EFI, one more line IONVMeController::setPowerState was printed after ktriage.

attempt 15 - Attempted to stop macOS from attaching its NVMe driver to my internal SSD, by injecting class-code Data FFFFFFFF at PciRoot(0x0)/Pci(0x6,0x0)/Pci(0x0,0x0) similar logs but not much progress overall, still around the same stage.

attempt 16 - UEFI > Quirks > ReleaseUsbOwnership changed from false to true, no changes from attempt 14, was testing if the OpenCore boot freeze could be caused by this documented issue, but apparently not.

attempt 17 - After some research, I determined that I need to get more verbose debug logs, so I added io=0xff msgbuf=1048576 and it displayed some probes failing, this could take a bit, it feels like the same block I ran into with EXITBS but yeah.

attempt 18 - Some more debugging: io=0xff was changed to io=0x6, not much changed, could be other issues rather than small things.

attempt 19 - Looked if there were any debug enhancers, and we find DebugEnhancer by acidanthera https://github.com/acidanthera/DebugEnhancer. Could prove useful later on, but these kernel visuals do look a tiny bit more detailed now.

attempt 20 - Modified DebugEnhancer, allowing it's failing hibernation hooks to be bypassed and show more driver logs on the end screen. From the new logs, both USB controllers match Apple’s xHCI driver and reach a start attempt, the internal NVMe initializes and reaches media/partition probing, but I still haven’t confirmed completed USB-controller startup or detection of the Recovery USB.

attempt 21 - Added a trace around macOS’s driver startup function. It prints ENTER before the original function runs and EXIT when it returns. The trace entered startup for both USB controllers, XHCI and TXHC but neither has a visible matching EXIT.

attempt 22 - Ruled out a resource wait in the four checks traced, the last frame shows successful returns (result=1) for: XHCI and its AppleUSBXHCIPCI driver, TXHC and its AppleUSBXHCIPCI driver. Both outer FWSTART calls still have no visible return. That narrows the stall to work after the resource checks, including the controller’s actual start function or later bookkeeping.

attempt 23 - The direct hook installed successfully: error 0, AppleUSBXHCIPCI::start was entered for both TXHC and XHCI. Neither call has a visible EXIT, while other kernel activity continues. The unfinished calls are now inside that USB startup function or something it calls, the preceding resource checks completed successfully.

attempt 24 - Back to experimenting, ReleaseUsbOwnership was set to true, tried it earlier, but these markers can now show whether it changes either controller’s startup behavior, but turns out that it doesn't have an effect.

attempt 25 - Added tracing around the shared AppleUSBXHCI::start function. Will help distinguish PCI-specific setup from shared controller initialization. Both controllers reach the shared AppleUSBXHCI::start function: FWBASE #5 ENTER provider=XHCI, FWBASE #10 ENTER provider=TXHC. Neither shows an EXIT. Their resource checks returned successfully beforehand, so the unfinished work is inside that shared startup function or something it calls.

attempt 26 - Added more checkpoints inside shared USB startup, covering memory lookup, mapping, register reads, and capability traversal. The final photo shows the capability walk reaching its end and the legacy capability lookup returning too. Waited five minutes with no further visible progress, still no Recovery. The earlier controller labels were off-screen, so this photo alone doesn't show how far both controllers got, but the visible calls did return, which means the unfinished work is further along.

attempt 27 - Added tracing around AppleUSBHostController::start and the later interrupt setup, and cut down the repeated capability logs so the useful lines stay on-screen. The XHCI controller passes its memory lookup, mapping, and register reads, finishes the capability walk, and returns from the legacy capability lookup. It then enters AppleUSBHostController::start with no visible EXIT. Both tracing masks show 3f, meaning all the early and later hooks installed. Other kernel activity still continues afterward, including the internal NVMe initializing and probing partitions. This narrows the unfinished work to the superclass startup function or something it calls, still no Recovery or Terminal.

attempt 28 - Booted the same tracing EFI again and took another photo. XHCI still gets through the memory lookup, mapping, register reads, capability walk, and legacy capability lookup, then enters AppleUSBHostController::start without a visible EXIT. Both tracing masks still show 3f. Other kernel activity continues afterward, but still no Recovery or Terminal, so the same startup function or something it calls remains the next place to look.

attempt 29 - The extra tracing finally showed what XHCI is waiting on. Mapper lookup, workloop and command gate setup, and matching notification all returned. The matching dictionary for AppleUSBHostResources was created, then WaitMatching entered with an unlimited timeout and never showed an EXIT in the photo. The internal NVMe kept going afterward. Looking at the same Recovery build, the USB resources service depends on IORTC, which AppleRTC publishes during startup. Still no Recovery or Terminal. The next thing to check is whether IORTC gets published and whether the USB resources service starts.

attempt 30 - Booted the EFI that traces AppleRTC and the USB resources service. At the same XHCI wait, the snapshot showed hooks=f and seen=18, so the IORTC publication call went in and came back, but there was still no USB resources start or registration in the trace. XHCI kept waiting for AppleUSBHostResources while the internal NVMe initialized. Next thing to check is what happens when macOS tries to match and probe the USB resources drivers. Still no Recovery or Terminal.

attempt 31 - Added tracing for the three USB resources probes. All three saw IORTC on their provider. The base probe returned an object and the TypeC probe returned null. The snapshot showed that the TypeCBPC probe entered, but it hadn't recorded a return or any candidate reaching startup. XHCI entered the same wait for AppleUSBHostResources while the internal NVMe kept going. Still no Recovery or Terminal. Next thing to check is what happens after the base probe returns and whether the TypeCBPC probe finishes.

attempt 32 - Added a trace to see if the TypeCBPC probe was waiting for the ACPI device named EC. Both USB controllers reached the same wait for AppleUSBHostResources, and at both points the trace showed that TypeCBPC had entered the EC wait with no return recorded. No USB resources candidate had reached startup in either snapshot, but I still don't know if the EC wait is what keeps the base candidate from starting. The internal NVMe kept going afterward. Still no Recovery or Terminal.

attempt 33 - I reached a graphical interface with a URL named support.apple.com/macsetup on there. Added a macOS-only EC stub alongside the Framework's real EC0, with the same tracing as attempt 32. This was the first boot that made it past the verbose screen to a graphical interface.

attempt 34 - Added VoodooPS2 with its keyboard and mouse plugins to see if the built-in keyboard and the touchpad's PS/2 fallback would work. Still ended up at the same macsetup screen with no input detected. All 38 EFI files matched, and the OpenCore log showed that all three drivers were injected. The log ends at EXITBS, so I still don't know whether the drivers found the devices or got stuck during initialization. Checked the VoodooPS2 2.3.7 matching code against my ACPI tables. The driver looks for the standard PS/2 names and my firmware lists those as compatible IDs. The actual hardware IDs are MSFT0001 for the keyboard and FRMW0002 for the mouse.

attempt 35 - Apparently my Framework is drunk on vodka (it's in language russian). Added the Framework's keyboard and mouse hardware IDs to the VoodooPS2 match lists. This time I reached the actual macOS Sequoia Recovery utilities window. The photos show Time Machine, Reinstall macOS Sequoia, Safari, and Disk Utility. The mouse, keyboard, and touchpad still didn't respond, so I couldn't open Terminal. All 38 files on the USB matched. I still don't know if the new IDs helped get past the prompt or whether VoodooPS2 attached.
Recovery opened in Russian. The EFI sets prev-lang:kbd to ru-RU:252, so for English, I'll use en-US:0. The Linux capture found the built-in PS/2 keyboard and mouse on IRQs 1 and 12.

attempt 36 - Enabled XhciPortLimit and reached the Recovery utilities again. The interface was still in Russian and input still didn't respond.

attempt 37 - Booted with the English language setting and ReleaseUsbOwnership enabled. Recovery opened in English, but the mouse and built-in keyboard still didn't respond. Terminal remains unopened. I think the mouse and built-in keyboard are my input priorities instead of figuring out external keyboard or such for now.

attempt 38 - Repeated the PS/2 startup messages so I could photograph them. The PS/2 controller and keyboard both returned success, but the mouse probe failed after a timeout on input stream 1. The photos confirm keyboard startup, with key delivery still unverified. This boot stopped at the macsetup prompt again. Attempt 37 had reached the Recovery utilities window.

attempt 39 - It seems like the mouse driver is required for recovery, as comparing 37 to 39, that was the only difference and 39 didn't get to recovery.

attempt 40 - Okay, I might be kinda dumb but, turns out keyboard worked early on. I just didn't know the shortcuts. I got to terminal, but I should probably figure out how to get mouse operational. Touchpad and mouse do not work yet. In terminal, I figured out that 3:2 aspect ratio in general makes macOS not scale as well, such as the startup logo being scaled to 16:9 where the logo isn't centered on the screen and everything being scaled to a small scale. After leaving it running for a bit, it appears that the lack of graphics acceleration makes the laptop go under much more load than expected. Ran a few commands in terminal, and figured that the next step is to partition my USB stick down so that i can save logs and analyze from there.

attempt 41 - Checked USB from Recovery Terminal. Both xHCI controllers started and found AppleUSBHostResources, but the IOUSB tree showed only its root and diskutil listed no external physical disks. The IOService query showed four SuperSpeed ports under TXHC and no USB 2 ports. Both controllers logged a speed-ID conflict in createPorts. Back in Linux, the protocol tables showed USB 2 ports 1 through 10 on XHCI and port 1 on TXHC. The controllers advertise USB 2, but I still don't know why those ports didn't show up in Recovery. I'll troubleshoot that later I guess.

attempt 42 - Tried the native I²C trackpad with VoodooI2C and VoodooI2CHID. I added the Panther Lake I²C controller ID to the kext and enabled polling. This boot went back to the macsetup input detection screen and nothing responded. I couldn't get to Recovery Utilities or Terminal, so I still don't know whether the I²C controller or trackpad actually attached.

attempt 43 - Went back to the Attempt 37 EFI and removed verbose boot. I got into Recovery Terminal again. The PS/2 controller, keyboard, and mouse drivers were loaded. IORegistry showed the I2C5 controller and the trackpad’s TPAD entry with PNP0C50. This EFI had no VoodooI2C drivers, and I still haven’t confirmed pointer movement.

attempt 44 - Added the VoodooI2C DEBUG controller and its GPIO and Services plugins to the Attempt 43 EFI. It went back to the macsetup input detection screen. The touchpad didn’t respond, and I couldn’t reach Recovery Utilities or Terminal. The HID and VoodooInput plugins were left out for this test, so I still don’t know whether the I²C controller attached.

attempt 45 - Kept the VoodooI2C DEBUG kexts from Attempt 44 and removed the match for the Framework’s I²C controller (8086:e451). It still went back to the macsetup input detection screen. I retried Attempt 43 afterward and it worked again.

attempt 46 - Disabled the VoodooI2C core and kept VoodooGPIO, VoodooI2CServices, and the polling flag. It still stopped at the macsetup input detection screen. All 46 files on the USB matched, and the OpenCore log showed both helper kexts were injected. I couldn’t verify what the helpers did after the kernel took over.

attempt 47 - Went back to the working Attempt 43 EFI and added only the -vi2c-force-polling boot argument. This time the boot stalled before the macsetup input detection screen appeared. I didn’t reach Recovery Utilities or Terminal, and I don’t know exactly where it stopped.

attempt 48 - Reused the Attempt 43 EFI and got into Recovery Terminal. PS2 Mouse Emulation was already set to Auto in BIOS. The PS/2 controller timed out on input stream 1, but ApplePS2Mouse started afterward and showed up as an active pointing device in IORegistry. On a cold boot, I reached the Recovery GUI and the touchpad worked for basic input. Multitouch gestures didn’t work. When I restarted from the Recovery GUI using the same EFI, I ended up back at the macsetup input detection screen.

attempt 49 - Used UnPlugged https://github.com/corpnewt/UnPlugged to obtain the macOS pkg, and it worked. Now, it's a matter of how much I can get working on this thing. I feel like this is a massive breakthrough. Reached desktop.

attempt 50 - Took the Attempt 43 EFI that boots the installed Sequoia desktop and added a newer VoodooI2C DEBUG build with the Framework’s I²C controller ID and polling on I2C5. I kept the PS/2 drivers enabled, but both the built-in keyboard and touchpad stopped responding. After going back to Attempt 43, I pulled the saved logs. In the four boots that showed the new I²C stack, macOS registered a PS/2 keyboard service, but I couldn’t tell from the log whether it received any keypresses. None of those four boots showed a PS/2 pointing service or confirmed that VoodooI2C attached to I2C5 or found the PixArt touchpad. I also found that I’d carried over a 133 MHz controller clock, while Linux uses 100 MHz for this exact controller. I don’t know yet whether the driver got far enough for that to matter. PS/2 input had already varied between cold boots and restarts, and I didn’t have the boot method recorded for each session.

attempt 51 - Kept the I²C kexts in the EFI and removed the match for I2C5 to see what happened without the driver attaching. The logs showed VoodooI2CServices starting, with no driver attached to I2C5. macOS registered the PS/2 keyboard, but the mouse probe hit a second timeout on stream 1 and failed. I’d already seen the same mouse failure on Attempt 43.

attempt 52 - Restored the I2C5 match and changed the controller clock to 100 MHz. This time VoodooI2C attached, recognized the Synopsys controller, and got through bus timing setup. It then failed to register the PCI interrupt, and IORegistry showed empty interrupt arrays on I2C5. The driver stopped before it could publish the touchpad or use polling. macOS registered the PS/2 keyboard again, and the mouse probe failed after another stream 1 timeout. The logs show the keyboard service was present in both attempts, but I couldn’t tell whether it received keypresses.

attempt 53 - Kept the Attempt 52 EFI and enabled PCI interrupt logging to see how macOS routed I2C5. Both logging flags were active, but the two saved kernel snapshots didn’t show a routing decision. VoodooI2C recognized the controller at 100 MHz and failed to register its PCI interrupt again. IORegistry still showed empty interrupt arrays on I2C5. macOS registered the PS/2 keyboard, and the mouse probe failed after a second timeout on stream 1. The built-in keyboard worked for a second or two, then stopped responding. It still showed up as active in the saved IORegistry capture.

