# LEDGER — VMsvga2-modern

Current state. Rewritten as state changes. If this file and
`.claude/CLAUDE.md` disagree about the state of the project, **this file wins**.

**Last updated: 2026-08-14, session close.** Questions 1-4 answered (GLD
loading; contract size 92; surface-client coupling; **it builds** on Xcode
26.6, all four products, first try, no source changes). `GA/` read — the
producer side is complete. Cross-tree work: VMQemuVGA personality diff run;
its fix order and probe design are pre-registered in that project's ledger,
with **this tree's `VMsvga2Accel.cpp:592-604` designated the documentary
positive control** for the probe (a property set is a specification even
without a running instance). Remaining here: **Q5 only, blocked on hardware
that may not exist in this working context** (a VMware-SVGA II guest).
Nothing in this repository has been run.

---

## What this checkout is

A fork of Zenith432's VMsvga2 — display and acceleration driver for Mac OS X
10.5+ as a **VMware** guest, PCI `15AD:0405` — brought forward to build with
modern Xcode against `MacKernelSDK`.

Lineage, as attested by the user (only the last hop is verifiable here):

- **Zenith432** — original, SourceForge SVN, SPL 2.0, inactive since Jan 2016.
  Upstream recommends transitioning to VMware's own `VMwareGfx.kext`, shipped
  in `darwin.iso` from Fusion 7.0.0 onward.
- **Lee Rodriguez (GitHub: newhacker1746)** — forked the original and updated it.
- **startergo/VMsvga2-modern** — this checkout. Confirmed from `.git/config`:
  `origin = https://github.com/startergo/VMsvga2-modern.git`, branch `master`.

A read-only mirror of the original SVN trunk also exists at
`github.com/mirror/VMsvga2`.

---

## Why it is here

**Reframed 2026-08-14, near the top on purpose.** This tree arrived as a
possible **GLD reference** — that is what someone coming back will go
looking for. The GLD is **not the load-bearing part**: it is a forwarding
trampoline over Apple's GMA950 GLD + `GLRendererFloat` (Q1/Q2), and the GL
context user client is an Intel915-interface mock with a dead SVGA3D draw
API. What actually carries the contract:

1. **`VMsvga2Accel.cpp:592-604`** — the runtime property publication
   (FB-side `IOAccelTypes` path string, `IOAccelIndex`, `IOAccelRevision`,
   `IOCFPlugInTypes` copy, `AccelCaps`). This is now the **documentary
   positive control** for VMQemuVGA's probe — a property set is a
   specification even without a running instance.
2. **`AC/UC/VMsvga2Surface.cpp`** — the selector map: which selectors
   WindowServer/CGS/CGL actually call (named in comments), the two-task
   two-view lock model, the 2D-context cross-client channel.
3. **`GA/VMsvga2GA.cpp`** — the userspace half: `IOAccelFindAccelerator` →
   2D-context open, surface lifecycle via 2D selectors, reserved-slot
   ABI details.

Come here for those three. The GLD directory answers a different question
(what Apple's loader asks a GL bundle for — 92 names) and nothing more.

The VMQemuVGA project has reached the Apple acceleration contract as its
remaining frontier: whether WindowServer on 10.6 will composite an
`IOAccelSurface` whose producer is not a GLD plugin, and what implementing a
GLD would cost if one is required. This tree is the only known worked example
of that contract on a macOS guest.

The three files that answer questions VMQemuVGA is currently asking:

| File | Question it bears on |
|---|---|
| `AC/UC/VMsvga2Surface.cpp` | a real `IOAccelSurface` user client — VMQemuVGA's `VMAccelSurfaceClient` is a Catalina-era design being redone for 10.6 |
| `AC/UC/VMsvga2GLContext.cpp` | the GL context client a GLD talks to — the consumer side of the coupling question |
| `GLD/EntryPointNames.c` | the entry-point table Apple's OpenGL loader asks a GLD for — not derivable from any public header |

---

## Verified

Only what has actually been observed in this checkout.

| Fact | How |
|---|---|
| Remote is `startergo/VMsvga2-modern`, branch `master` | `.git/config` |
| `MacKernelSDK` is a submodule of `acidanthera/MacKernelSDK` | `.gitmodules` |
| Four bundles: FB kext, AC kext, GA CFPlugIn, GLD bundle | four Info.plists + `Generic.xcconfig` |
| Accelerator personality is `VMsvga2Accel`, `IOMatchCategory = IOAccelerator`, `IOProviderClass = IOPCIDevice`, `IOPCIPrimaryMatch = 0x040515AD`, `IOProbeScore = 100` | `Info-AC.plist` |
| AC depends on the FB kext (`net.osx86.driver.VMsvga2`), `IOGraphicsFamily 1.4`, `IOPCIFamily 1.6`, `com.apple.kpi.* 8.0.0` | `Info-AC.plist` |
| `IOCFPlugInTypes ACCF0000-0000-0000-0000-000a2789904e → VMsvga2GA.plugin` | `Info-AC.plist` |
| `Info-GLD.plist` is a plain `BNDL` with all identity templated from xcconfig | `Info-GLD.plist` |
| The user-client family exists: Device, 2DContext, GLContext, Surface, DVDContext, OCDContext | `AC/UC/` listing |
| GLD is source, not a binary drop: `VMsvga2GLDriver.c/.h`, `EntryPointNames.c/.h` | `GLD/` listing |
| **GLD discovery is a runtime property, not a plist key.** `VMsvga2Accel::start` does `setProperty("IOGLBundleName", ...)` on the accelerator object before `registerService()` | `AC/VMsvga2Accel.cpp:619-637` |
| As configured, the published GLD is **Apple's `AppleIntelGMA950GLDriver`**, not the bundle in this repo: the `#ifdef USE_OWN_GLD` branch (`"VMsvga2GLDriver"`) is dead because `USE_OWN_GLD` is defined in no build configuration | `AC/VMsvga2Accel.cpp:620-624` + `project.pbxproj` GCC_PREPROCESSOR_DEFINITIONS (all 8 entries checked: only `LOGGING_LEVEL`, `FRAGILE`, `VLOG_LOCAL` anywhere) |
| The repo's own GLD is a **forwarding trampoline, not a renderer**: `gldInitializeLibrary` dlopens Apple's `AppleIntelGMA950GLDriver` and `OpenGL.framework/.../GLRendererFloat`, dlsyms all 92 entry points into `bndl_ptrs[2][92]`, and the plugin forwards calls to them | `GLD/VMsvga2GLDriver.c:50-56, 71-87` |
| GLD contract size: **92 entry points** (`NUM_ENTRIES`), of which 47 are macro-generated generic forwarders (`GLD_DEFINE_GENERIC`; corrected 2026-08-15 — this row kept the stale 48 when the tally row was fixed); the hand-written ones read so far (`gldInitializeLibrary`, `gldGetVersion`, `gldGetRendererInfo`) also forward, with logging and a `#if 0` patch block | `GLD/EntryPointNames.h:32`, `EntryPointNames.c:33-38`, `VMsvga2GLDriver.c:40-46, 107-148` |
| The kernel-side device client **impersonates Intel hardware to the GLD**: `get_config` returns `c1 = 0x40000` ("GMA 950") on `version_major >= 11`, else `0` ("GMA 900"); comment says `c1` "used by GLD to discern Intel 915/965/Ironlake" | `AC/UC/VMsvga2Device.cpp:227-234` |
| Two Apple bug-workaround properties also set at runtime: `IODVDBundleName = "AppleVADriver"` always (AppleVA CFReleases NULL without it), and `IOGLBundleName = ""` when GL is off on `version_major >= 13` (libGFXShared bug from 10.9) | `AC/VMsvga2Accel.cpp:626-636` |
| **GLD contract size: 92 entry points** in three eras — 75 from 10.6 (3 of those marked "Discontinued OS 10.6.3": idx 29, 32, 33), 6 added 10.6.3 (idx 75-80), 11 from 10.5.8 (idx 81-91) — plus `gldInitializeLibrary`/`gldTerminateLibrary`, not in the table, dlsym'd by name | `GLD/EntryPointNames.c:33-136`, `VMsvga2GLDriver.c:71-77` |
| Shim implements 75 of 92 entries: **47 `GLD_DEFINE_GENERIC` macro forwarders + 28 hand-written** (28 = 18 pure typed forwarders + 2 `#if 0`-patched forwarders + 1 active patcher + 7 local stubs; 30 hand-written functions total including `gldInitializeLibrary`/`gldTerminateLibrary`, which are not table entries). **Corrected 2026-08-15 after a bot review flagged the tally not summing:** the session's original `48` counted the `#define GLD_DEFINE_GENERIC` line itself (invocations: 47), and `27 hand-written`/`20 typed forwarders` were derived/eyeballed, never re-counted. Cross-checks: 47+28=75=92−17 missing; binary exports 77 `gld*` = 75 + 2 init. **17 entries (idx 75-91, the 10.6.3 and 10.5.8 tails) have no function at all** — grep of `VMsvga2GLDriver.c` finds zero definitions | `GLD/VMsvga2GLDriver.c` (502 lines, read entire) |
| Shim behavior split: 1 active patcher (`gldGetString` overrides 0x1F00→"Zenith432", 0x1F01→"VMware SVGA II OpenGL Engine", 0x1F04→"VMsvga2GLDriver"); 2 disabled `#if 0` patchers (`gldGetRendererInfo`: flags\|0x4000, p[1]=0x17CD, p[17]=64, p[19]=64; `gldChoosePixelFormat`: flags\|0x4000, p[1]=0x501); 7 local stubs that never forward (CreateVertexArray/DestroyVertexArray→0, Create/DestroyComputeContext→−1, LoadHostBuffer/SyncBufferObject/SyncTexture→0); 18 pure typed forwarders; 47 generic forwarders | `GLD/VMsvga2GLDriver.c:107-502` |
| Shim **split-routes**: idx 0-3 (`gldGetVersion`, `gldGetRendererInfo`, `gldChoosePixelFormat`, `gldDestroyPixelFormat`) hardcoded to `bndl_ptrs[0]` = AppleIntelGMA950GLDriver; idx ≥ 4 to `bndl_ptrs[bndl_index]` with `bndl_index = 1` = **GLRendererFloat, Apple's generic software renderer**. Enumerate as GMA950, render in software. Commented-out `BNDL3` (`VMsvga2GLDriver.bundle/Contents/MacOS/GLRendererFloat`) shows the intended end-state: swap in an own executable named GLRendererFloat | `GLD/VMsvga2GLDriver.c:50-52, 87, 114, 130, 160, 184, 196` |
| **`VMsvga2GLContext.cpp` is NOT a command-stream translator — it is a mock of the Intel915 kernel-side interface.** All ~38 selectors are log-and-return: `finish()`, `wait_for_stamp()` return Success unconditionally; `nv_*` selectors return Unsupported; no code anywhere reads the command buffer payload. Buffer protocol is Intel915-modeled per comments ("Intel915 puts (submitStamp - 1) here", "Intel915 ors an optional flag @ IOAccelerator+0x924") | `AC/UC/VMsvga2GLContext.cpp:211-239, 560-571, 734-849` |
| **The SVGA3D rendering API on the accelerator is dead code.** `drawPrimitives`, `setTextureState`, `setRenderState` have zero callers tree-wide; `createContext`/`setRenderTarget`/`clear` are called only from `RectFill3D`, itself inside `#if 0`. Live SVGA3D usage is surface-ops only (createSurface, surfaceDMA2D, surfaceCopy, surfaceStretch, present, blit*) — the WindowServer surface path | `AC/VMsvga2Accel.cpp:882-929 (#if 0), 1026-1441` |
| Kernel-side GL selector table shifts by one on 10.8 exactly: `getTargetAndMethodForIndex` bumps `index` for `index >= kIOVMGLFilterControl` on every OS except `version_major==10 && version_minor==8` | `AC/UC/VMsvga2GLContext.cpp:292-295` |
| **Licence headers are heterogeneous and none of those sampled are SPL.** `VMsvga2Accel.cpp`, `VMsvga2Surface.cpp`, `EntryPointNames.c`, `VMsvga2GLDriver.c`: MIT-style permission notice. `VMsvga2GLContext.cpp`: MIT-style **plus "Portions Copyright (c) Apple Computer, Inc."** — mixed provenance. `FB/VMsvga2.cpp`: bare copyright, no permission grant. Only 5 files sampled; not a tree-wide survey | headers of the named files |
| **Surface client selector table: 18 selectors** (`kIOAccelNumSurfaceMethods`); `externalMethod` redirects `SetShapeBacking`/`SetShapeBackingAndLength` to `set_shape_backing_length_ext`, mirroring IONVSurface on 10.6 | `AC/UC/VMsvga2Surface.cpp:73-94, 200-232` |
| **Coupling: Apple userspace callers named in comments.** `surface_control` sel 1 ← `_CGXSynchronizeAcceleratedSurface`; sel 4 ← `CGLSetPBufferVolatileState`; sel 5 ← `CGXBackingStorePerformCompression`/`destroyBackingSurface`/`synchronizeBackingSurface`/`allocateBackingSurface`/`CGXBackingStoreDataAccess`. WindowServer blits via `surface_flush`; QuickTime via `SwapSurface` → `surface_flush_video` | `VMsvga2Surface.cpp:1493-1531, 1878-1882` |
| **Coupling: two-task, two-view lock model.** WindowServer owns all CGSSurfaces (CGSAddSurface is routed to it), locks via the surface client (`map[0]` in the creator task); the app locks via the 2D-context client → `context_lock_memory` (`map[1]` in the context's task). The driver may hand the two tasks **different memory** — exploited by `setup_trick_buffer` (all-black YUV view for WindowServer to satisfy VMware's overlay colorkey while the app writes real frames); comment calls this "a design flaw in Apple's CGSSurface technology" | `VMsvga2Surface.cpp:1083-1136, 1728-1749`; `VMsvga22DContext.cpp:401-411` |
| **Coupling: intra-kernel cross-client channel.** `findSurfaceForID` broadcasts `kIOMessageFindSurface` (vendor-specific msg 0x10) carrying `FindSurface{cgsSurfaceID, client}`; matching surface answers; the 2D context then calls surface methods directly (`copy_*_to_self`, `context_*`, `surface_flush_video`) | `VMsvga2Accel.h:37, 173`, `VMsvga2Accel.cpp:1820-1828`, `VMsvga2Surface.cpp:234-248`, `VMsvga22DContext.cpp:260-411` |
| `set_id_mode`: **`wID == 1` is the WindowServer's surface**; under screen object it triggers `createPrimaryScreen`. Publishes `CGSSurfaceID` property. Format map: 1555→`SVGA3D_X1R5G5B5`, 8888→`X8R8G8B8`, BGRA32→`A8R8G8B8` (but `m_pixel_format` reported back as 8888), YUV→`YUY2`, YUV2→`UYVY`; non-windowed modes rejected | `VMsvga2Surface.cpp:1304-1359` |
| `surface_flush` dispatches **six output strategies** (0=DMA direct, 1=DMA+copy, 2=DMA+stretch+copy [master-surface mode]; 3=blitToScreen direct, 4=via-3D [screen-object mode]; 5=GFB) by hw mode/format/scale; fence always; strategies 0-2 followed by `doPresent()`. GFB mode only supports X8R8G8B8 at 1:1 | `VMsvga2Surface.cpp:1380-1458` |
| **`detectBlitBug` is a runtime negative control**: DMA green pixels to a scratch SVGA3D surface and back, verify all == `0xFF000000`; on mismatch disable direct blit and cache the result ("Blit Bug: Yes") | `VMsvga2Surface.cpp:681-744` |
| 10.6 Window-Grab deadlock workaround: `wID==1 && options==0x5` → skip one write-lock (`bSkipWriteLockOnce`); comment "Window Grab works on OS 10.7, check what changed" | `VMsvga2Surface.cpp:1658-1666` |
| Framebuffer→surface and surface→surface copies are **CPU copies** (kernel mappings + `genericBlitCopy`), because WindowServer holds a write-lock and combines blit results with its own rendering in guest memory — host-VRAM-only blit impossible; "Dumbass WindowServer" fixup for `num_rects == 0`; surface→framebuffer deliberately unsupported ("not really necessary") | `VMsvga2Surface.cpp:1763-1773, 1898-1979, 1878-1895` |
| `IOAccelSurfaceInformation` as returned to clients: `address[0]` (+ byte offset), width/height from scale.buffer, `rowBytes`, `pixelFormat`, `colorTemperature[0] = 0x1CCCC` ("from GeForce.kext") | `VMsvga2Surface.cpp:377-396` |
| `get_state` always returns `kIOAccelSurfaceStateNone`; `surface_read` returns Unsupported (no surface readback path) | `VMsvga2Surface.cpp:1232-1239, 1285-1290` |
| YUV overlay: SVGA video-overlay units (`VideoSetRegsInRange` with fence), all-or-nothing clipping (device lacks dest-clipped overlay), `clear_yuv_to_black` fills 0x10801080 (UYVY) / 0x80108010 (YUY2), master surface released in video mode so resolution changes aren't thwarted, tearing accepted (no SyncFIFO after video flush) | `VMsvga2Surface.cpp:1054-1193, 1982-2024` |
| **It builds.** Both `ReleaseSnowLeo` and `ReleaseSnowLeoDebug`, `xcodebuild -target All`, exit 0, no source changes. Toolchain: Xcode 26.6 (17F113) on arm64 macOS 26.5 host, `SDKROOT=macosx` → MacOSX26.5.sdk (a 10.6 SDK is also installed but not used). Deployment-target warning: 10.6 below supported floor 10.13-26.5.99 — clamped, warning only | build logs `/tmp/vmsvga2-build-{release,debug}.log`, exit=0 both |
| **Artifacts**: 4 products, all `Mach-O … x86_64` (kexts: "64-bit kext bundle"), all adhoc-signed (`codesign --force --sign -`, "Sign to Run Locally", TeamIdentifier not set) — Xcode 26 default; the 10.6 era expects none. Plist templating resolved (AC kext: `net.osx86.driver.VMsvga2Accel` v1.2.6, `IOPCIPrimaryMatch 0x040515AD`, `IOCFPlugInTypes` present). 205 warnings per config, overwhelmingly `-Winconsistent-missing-override` | `ls build/ReleaseSnowLeo/`, `file`, `codesign -dv`, `plutil -p` |
| **GLD binary confirms the Q2 audit**: 77 exported `gld*` symbols = 75 table entries + `gldInitializeLibrary` + `gldTerminateLibrary`; `gldGenerateTexMipmaps`/`gldGetTextureLevel` etc. absent, as the source read predicted | `nm -gU build/ReleaseSnowLeo/VMsvga2GLDriver.bundle/Contents/MacOS/VMsvga2GLDriver` |
| Submodule pinned at `05094e5e88cec7caedbfb35e8449ed0db94bf95b`; checkout arrives uninitialized (`-` prefix in `git submodule status`) and `MacKernelSDK/` is empty until `git submodule update --init --recursive` | `git submodule status` before/after |
| CI recipe: GitHub Actions `macos-15-intel`, `xcodebuild -project VMsvga2.xcodeproj -target All -configuration ReleaseSnowLeo(Debug)`, packages all 4 products + dSYM per config | `.github/workflows/main.yml:17-75` |
| **Install destination for all four products — read 2026-08-14, per the build.md UNVERIFIED marker.** `INSTALL_PATH = "$(SYSTEM_LIBRARY_DIR)/Extensions"` appears at exactly two sites, both **project-level** configs (pbxproj:626, 710); **no target-level overrides exist**. So an **`install` build action** would place not only the two kexts but also `VMsvga2GA.plugin` and `VMsvga2GLDriver.bundle` as **top-level `/S/L/E` bundles** — not inside a kext's `Contents/PlugIns`. **Wording corrected 2026-08-15 after a bot review flagged the overstatement:** `INSTALL_PATH` is honored only by the `install` action; the plain `xcodebuild -target All` builds recorded above place products in `build/ReleaseSnowLeo*/` only, and nothing has been installed anywhere. Corroborating observations (not discovery proof): the runtime-published GLD name `AppleIntelGMA950GLDriver` refers on stock systems to a `/S/L/E` bundle, and the GLD shim's commented-out `BNDL3` path is also `/S/L/E/VMsvga2GLDriver.bundle/…`. **Still open:** whether `IOCFPlugInTypes`' value `VMsvga2GA.plugin` is resolved by path or bundle identifier (i.e., whether discovery *requires* `Contents/PlugIns` placement despite the install path), and where the GLD loader searches for `IOGLBundleName` values — `INSTALL_PATH` evidences the build's intent, not the loader's search semantics | `project.pbxproj:626, 710` (grep: only INSTALL_PATH sites); `.claude/rules/build.md:198-213` (the UNVERIFIED marker) |
| **KPI symbol check (user-directed, 2026-08-14): reserved-slot audit is clean for a 10.6 x86_64 target.** The kexts reference `_RESERVED*` padding up to: IOFramebuffer 1-31, IORegistryEntry 0-31, IOService 0-47, IOUserClient 0-15, OSMetaClass 0-7, OSObject 0-15, OSMetaClassBase 3-7. Every referenced slot is declared reserved in the 10.6 SDK headers under `__LP64__` (the `ReservedUsed` blocks are the 32-bit `#else` branch; IOFramebuffer slot 0 is Used in both SDKs and is *not* referenced; OSMetaClassBase 0-2 were consumed by `taggedRetain`/`taggedRelease`/third — 3-7 still reserved at `OSMetaClass.h:797-801`). 490 undefined symbols total; `IOSurfaceRoot` is reached via vtable only — **no link-time private-class symbols**. **Evidence chain, corrected per user (2026-08-14):** the kexts were read with the *host's* Xcode 26 nm (host tool on host-built binaries — sound); the 10.6 side is header text, no 10.6 tool involved. **nm on 10.6 is not reliable** (user), so the earlier suggestion of nm-against-a-10.6-kernel as arbiter is retracted — the arbiter is the guest load itself. The audit is also **empirically** grounded, not just header-grounded: the original code shipped and ran on the era with the same subclass set (user note) | `nm -u` (host tool) on both kexts; 10.6 SDK `Kernel.framework/Versions/A/Headers` (Downloads), read as text |
| **Personality diff vs VMQemuVGA run 2026-08-14** (live tree `~/VMQemuVGA` — sibling checkout; see that ledger for the entry): 1) `IOCFPlugInTypes` **absent everywhere** in VMQemuVGA (grep zero hits). 2) `AccelCaps` absent. 3) **FB-side trio incomplete and mistyped**: FB sets `IOGLBundleName="GLEngine"`+`IOAccelIndex=0` only inside `if (has_3d_support)` (`VMVirtIOFramebuffer.cpp:383-394`); `IOAccelTypes` is **never set on the FB**; VMQemuVGA's accelerator nubs set `IOAccelTypes = 7` as a **number** (`VMQemuVGAAccelerator.cpp:265`, `VMVirtIOGPU.cpp:479`), whereas the worked example sets FB.`IOAccelTypes` = **path string** of the accelerator (`VMsvga2Accel.cpp:597-598`; header documents the keys as consumed by `IOAccelFindAccelerator()`, `IOGraphicsInterfaceTypes.h:293-297`). 4) **Three accelerator-ish nubs with conflicting values**: `VMQemuVGAAccelerator` (FB-attached, registered, `IOGLBundleName="VMVirtIOGLEngine"`), `VMVirtIOGPUAccelerator` (`IOGLBundleName="GLEngine"`, `IOOpenGLRenderer=true`), `VMMetalPlugin` (`IOAccelTypes=2`, revision 1); plus `IOGLBundleName="com.apple.kpi.iokit"` at `VMVirtIOGPU.cpp:6196`. 5) Accelerator claims actively suppressed on the FB for Catalina-crash reasons (`VMVirtIOFramebuffer.cpp:1796-1803`) — same code path serves the 10.6 guest | cross-tree read; file:line as listed |
| **`GA/` read 2026-08-14** (all 4 files, 1360 lines). It is the **IOGraphicsAccelerator CFPlugIn**: factory answers `kIOGraphicsAcceleratorTypeID` (the `ACCF0000-…-904e` UUID), interface version/revision = `kCurrentGraphicsInterface{Version,Revision}` (2), a disassembly-driven reimplementation of **GeForceGA 1.5.48.6 (10.5.8)** (offsets and TBD address ranges cited throughout; factory UUID `03463B45-6FDD-4749-B6B7-15EB76BAA22F`, Probe order 2000) | `GA/VMsvga2GA.cpp:58-59, 70, 80-96, 289-292, 1083-1123` |
| **`vmStart` calls `IOAccelFindAccelerator(service, &accelerator, &framebufferIndex)` then `IOServiceOpen(accelerator, self, 2 /* 2D Context */)`** — userspace confirmation that the FB-side `IOAccelTypes`/`IOAccelIndex` keys are the GA plugin's discovery mechanism. Without the trio on the FB (path-string form), the plugin cannot start; this gives the VMQemuVGA personality-diff finding #1 a concrete userspace failure chain | `GA/VMsvga2GA.cpp:216-219` |
| **The app never opens the surface client.** `AllocateSurface(kIOBlitHasCGSSurface)` → `vmSetSurface(0x800)` → `kIOVM2DSetSurface(cgsSurfaceID, options\|format-bits)` on the **2D context**; kernel 2D context then locates the surface client via `findSurfaceForID`. `LockSurface` → `kIOVM2DLockMemory` → kernel `context_lock_memory` (map[1] into the app task). Format bits 0x400=UYVY/2vuy, 0x200=YUVS, 0x100=BGRA — matching the kernel's `set_surface` decode exactly | `GA/VMsvga2GA.cpp:446-533, 800-865`; `AC/UC/VMsvga22DContext.cpp:313-347` |
| Hard-won client-contract detail: after `LockSurface`, `surface->accessFlags = 2` is required — "It took me over a week to come up with the following line because QuickTime refuses to run under gdb — Zenith432, 8/27/2009" | `GA/VMsvga2GA.cpp:602-606` |
| **Two reserved interface slots are load-bearing**: `__gaInterfaceReserved[0] = vmWaitSurface`, `[1] = vmSetSurface` — Apple's CGS calls these through the reserved slots of `IOGraphicsAcceleratorInterface`; a reimplementation must fill them. **Corrected 2026-08-14 (later):** the reserved array is **`[24]`**, not 22 — the source comment at `VMsvga2GA.cpp:1112` ("22 slots reserved") is a miscount; verified identical across the 10.6 SDK, the 26.5 SDK, and vendored Apple IOGraphics 1.5.1 (`IOGraphicsInterface.h:141`). Additionally, the `IOGA_COMPAT` named slots `GetBlitProc` (between CopyCapabilities and Flush) and `WaitForCompletion` (between Flush and Synchronize) are **left NULL** by `_buildGAFTbl` — legacy accessors; Apple's own consumer tool (`readfb.c`) uses `GetBlitter`, never `GetBlitProc` | `GA/VMsvga2GA.cpp:1083-1115`; `IOGraphicsInterface.h:93-110, 141` (3 sources) |
| Blitters exposed: Fill (solid rects), Copy (copyrects), CopyRegion (framebuffer/CGSSurface); MemCopy/MemCopyRegion stubbed `Unsupported`; `vmSynchronize`→`UpdateFramebuffer` only with `kIOBlitSynchronizeFlushHostWrites`; `vmSwapSurface`→`kIOVM2DSwapSurface` (the QuickTime video path); `vmWaitComplete`→`kIOVM2DFinish`. `SUPPORT_CGLS` is defined in no build config of this fork → as built, `CopyCapabilities('cgls'/BGRA32)` returns Unsupported and MemCopyRegion is compiled out | `GA/VMsvga2GA.cpp:310-337, 382-407, 643-751, 1117-1123`; pbxproj GA definitions |

## Not verified — do not assume

- ~~**That anything builds.**~~ **Answered 2026-08-14 — it builds.** Both
  configurations, all four products, first attempt, zero source changes
  (see Verified). Correction of this section's earlier text ("no build has
  been attempted"), which was true until Q4 but was left standing here
  after the build succeeded — the ledger contradicted its own Verified
  table until 2026-08-14, caught by the user.
  What the build does **not** establish: that the deployment-target clamp
  (10.6 below the 10.13 floor, warning only) changes nothing in the
  generated code; reserved-slot audit says the ABI is compatible, the rest
  is unexamined.
- **That anything runs.** No load has been attempted in this working
  context: the 2026-08-14 build artifacts exist only in
  `build/ReleaseSnowLeo*/` and this session installed and loaded nothing
  anywhere. Whether the kexts load on any guest OS is untested — that is
  Q5.
- ~~How the GLD bundle is discovered~~ **Answered 2026-08-14** — see Verified.
  The earlier guess ("may be published programmatically from
  `VMsvga2Accel.cpp`") was correct. What remains unknown is only *where* Apple's
  userspace loader searches for a bundle once it has the name — the tree
  publishes the name and says nothing about lookup paths.
- **The lineage above the `origin` remote.** User-attested, not checked against
  commit history.
- **That Apple's GLD actually loads and opens these user clients at runtime.**
  Read from code, never observed. Q2's finding sharpens this: it is not just
  unverified at runtime — the GL hardware path is **unfinished in source**
  (mock context client, dead 3D API). Any "this worked on era X" claim needs a
  run, and the honest prior is that GL rendering never worked via the hardware
  path in any era of this tree.
- **Whether `AppleIntelGMA950GLDriver.bundle` exists on any given guest OS.**
  User-attested as doubtful beyond ~10.7-10.8 (GMA 950 Macs are 2006-era).
  Tree-internal counter-evidence: the code reports `c1 = 0x40000` ("GMA 950")
  to the GLD on `version_major >= 11` and carries 10.9 workarounds, i.e.
  Zenith432 maintained the strategy past Lion. Neither side verified.
- **The capability ceiling of this route** (GMA950-class GL 1.4/2.0-era per
  the user). Unverified; note the shim's rendering half is GLRendererFloat
  (software), so in `USE_OWN_GLD` builds the ceiling is Apple's software
  renderer, not GMA950 hardware.
- **Licence, tree-wide.** Sampled headers (5 files) show MIT-style or bare
  copyright — **no SPL text found in the sample**, contradicting the SPL 2.0
  assumption in `.claude/CLAUDE.md`. The CLAUDE.md licence section is stale or
  wrong; headers remain the record. Full survey not done.

---

## Open questions, in the order worth answering

1. ~~How is the GLD loaded?~~ **Answered 2026-08-14.** `IOGLBundleName` is
   `setProperty`'d on the accelerator at `AC/VMsvga2Accel.cpp:619-637`, before
   `registerService()`. As built it names **Apple's `AppleIntelGMA950GLDriver`**;
   the branch naming this repo's own `VMsvga2GLDriver` bundle requires
   `USE_OWN_GLD`, which no build configuration defines.
   **Correction 2026-08-14 (later same day):** the Q1 write-up here first said
   the discovery mechanism "materially lowers the cost estimate" of VMQemuVGA's
   GLD path. That was wrong twice over, per user pushback and per Q2's
   findings: (a) the cost *moves* (to the kernel-side contract), it does not
   vanish; (b) the expensive part — a command-stream decoder — **was never
   written in this tree either** (see Q2). What is cheap and real: the
   discovery mechanism itself, one `setProperty` call.
2. ~~How large is the GLD contract?~~ **Answered 2026-08-14.** 92 named entry
   points in three eras + 2 init entries. Shim implements 75 (47 generic
   forwarders, 18 pure typed forwarders, 2 disabled-patch forwarders, 1 active
   patcher, 7 local stubs); 17 tail entries unimplemented; rendering half routes to
   GLRendererFloat (software). See Verified for `file:line`. The load-bearing
   finding came from auditing the claim that `VMsvga2GLContext.cpp` translates
   GMA950 streams: **it does not.** It is an Intel915-interface mock
   (allocate, map, answer queries, return Success). The SVGA3D draw API on the
   accelerator has no live callers; the only live SVGA3D path is surface
   operations. Consequence for VMQemuVGA: the reusable assets here are the
   kernel-side selector contract (numbering, argument shapes, struct layouts,
   buffer/stamp protocol, the 10.8 selector shift) and the mock-as-strategy
   pattern — not a translator. If VMQemuVGA goes the "borrow an Apple GLD"
   route, the GMA950-batch→virgl decoder is work that exists nowhere yet.
3. ~~What does `VMsvga2Surface.cpp` do that VMQemuVGA's surface client does
   not?~~ **Answered 2026-08-14** (file read end to end, 2024 lines, plus the
   2D-context call sites). It answers VMQemuVGA's coupling question directly:
   **the selectors are called, and by whom.** The surface path on 10.6 does
   not require a GLD at all — WindowServer is the creator and locker of every
   CGSSurface; apps attach through the GA plugin and the 2D-context client;
   the only CGL touchpoint is `surface_control` selector 4
   (`CGLSetPBufferVolatileState`). Coupling happens through three channels:
   the surface-client selectors (called by WindowServer/CGS internals),
   the 2D-context selectors (called by QuickTime/GA, dispatching kernel-side
   into the surface object via `kIOMessageFindSurface`), and the two-task
   two-view lock model (map[0] to WindowServer, map[1] to the app —
   different memory allowed). What VMsvga2 has that a bare transport lacks:
   real backing management (client/VRAM/GMR, grow-only), six dispatchable
   output strategies behind one `surface_flush`, fences on every DMA, a
   runtime negative control (`detectBlitBug`), format/negotiation tables,
   the Window-Grab deadlock workaround, and a YUV overlay path. Prediction
   for VMQemuVGA instrumentation: on its 10.6 guest, WindowServer should call
   `set_id_mode`(wID=1), `set_shape*`, `write_lock`, `surface_control`
   (1, 4, 5), and `surface_flush`; QuickTime-only paths add `swap_surface`.
   If those selectors stay silent, the surface was never connected — that is
   the negative control VMQemuVGA has not yet run.
   **Reframed by the user, 2026-08-14:** VMQemuVGA's Phase A saw zero
   surface-client lines — so there, WindowServer created a CGS surface and
   never opened a client on the accelerator. The open question is not
   "will it composite a non-GLD producer" (this tree says yes, by design)
   but "why does WindowServer not treat VMQemuVGA's accelerator as a surface
   provider at all" — see the personality-diff checklist in Next step.
   **Cost-estimate correction (user, 2026-08-14):** do **not** book the
   surface path as readback elimination. In this tree every surface backing
   is CPU-visible guest memory (client backing = app memory; GMR = guest RAM;
   VRAM = guest VRAM aperture); `surface_flush` DMAs guest→host; there is no
   host-resident-only CGS surface path. The split removes the host→guest
   readback and the drawRect/double-buffer dance; it adds guest-resident
   compositing buffers, DMA-out, and WindowServer's CPU combine
   (`copy_framebuffer_region_to_self`). Whether that is a net win is
   unmeasured — measure before justifying work with it.
4. ~~Does it build?~~ **Answered 2026-08-14 — yes.** Both configurations,
   all four targets, x86_64-only, on Xcode 26.6 / macOS 26.5 SDK / arm64 host,
   zero source changes. Signing is Xcode-default adhoc. `.claude/rules/build.md`
   updated with confirmed facts. What a build does **not** establish: whether
   the binaries load on any guest OS (Q5), and whether any of the 10.6-era
   runtime assumptions survive the deployment-target clamp (warning only).
5. **Is there a host/guest combination available to run it?** It matches VMware
   SVGA II, not virtio-gpu or QXL, so the VMQemuVGA guests do not exercise it.

---

## Next step

1. ~~Personality diff~~ **Done 2026-08-14** — results in Verified and in
   VMQemuVGA's ledger. The follow-on belongs to the VMQemuVGA project:
   `ioreg` arbitration on the 10.6 guest before any property change, then
   the minimal FB-side trio as a pre-registered experiment (prediction
   recorded in VMQemuVGA's ledger).
2. ~~Read `GA/`~~ **Done 2026-08-14** — see Verified. Prediction scored 3/4:
   confirmed the IOGraphicsAccelerator CFPlugIn, the 2D-context channel,
   and no-GLD; **corrected** — the plugin never opens the surface client and
   never sees `IOAccelSurfaceInformation`; both surface lifecycle operations
   ride the 2D-context selectors, and the kernel bridges to the surface
   object. Full producer-side picture complete.
3. **Q5 (host/guest):** find a VMware-SVGA guest to load the freshly built
   kexts. The KPI audit says linkage is clean; the first load attempt is the
   real arbiter. This is the only open question left in this tree — and it
   is **blocked on hardware that may not exist in this working context**:
   the device is `15AD:0405`, not virtio-gpu or QXL, so the VMQemuVGA
   guests do not exercise it. If no VMware host is ever available here,
   this question closes as "unanswerable locally" and the tree remains a
   read/build reference.
