# Husk on iOS 17 / arm64e with Dopamine — crash-loop fix

Copy the contents of this archive over the root of your `HuskIOS17dopa` clone.

Four files change:

```
src/ios-jit/husk-ios-jit.c      crash sentinel + guards on all three JIT routes
src/ios-jit/husk-ios-jit.h      two new exports
src/app/Husk/JITBootstrap.swift reads the sentinel, reports the real reason
scripts/package_ipa.sh          stop stripping dynamic-codesigning from .tipa
```

## What was happening

The app was not crashing at launch. It ran for ~683 ms and died inside the JIT
prewarm. From the logs:

```
[jit] claiming 512 MiB of JIT memory now
[husk-jit] self: MAP_JIT refused (Operation not permitted)
[husk-jit] MAP_JIT probe: kernel granted protection 0x5 (execute yes)
```

`husk_self_dual_mapping()` tries three routes in order:

1. **StikDebug** — not installed, fails cleanly.
2. **MAP_JIT** — needs `dynamic-codesigning`. Returned `EPERM`.
3. **Plain RWX** — reserves memory, makes a writable alias via `vm_remap`,
   `vm_protect`s the executable view, writes `movz w0, #0x5a5a ; ret` through
   the alias and **calls it**.

Route 3 is the killer. On iOS 17 / arm64e the kernel answers a call into an
unsigned page with **SIGKILL**, not a signal. `husk_self_selftest()` installs
SIGBUS / SIGSEGV / SIGILL handlers around that call, and none of them can catch
SIGKILL — so `siglongjmp` never runs, ReportCrash writes nothing, and nothing
reaches Analytics. The app vanishes and the log stops mid-sentence.

Husk's own gate for route 3 is `CS_DEBUGGED || access("/var/jb")`. Under
Dopamine both are true and the execution is *still* killed, so the gate does not
mean what it is being asked to mean.

The loop was unbreakable because `JITBootstrap.selfGrantTried` is an in-memory
`static var`. It resets every launch, so the fatal branch was re-entered every
time and the app never lived long enough to reach Settings and pick another
method.

## Why MAP_JIT was unavailable

Entitlements read out of the two release files:

| key | `Husk.ipa` | `Husk.tipa` |
|---|---|---|
| `get-task-allow` | yes | yes |
| `com.apple.developer.kernel.increased-memory-limit` | yes | yes |
| `com.apple.developer.kernel.extended-virtual-addressing` | yes | yes |
| `dynamic-codesigning` | **yes** | **no** |

`scripts/package_ipa.sh` stripped it from `.tipa` builds on purpose, alongside
two genuinely private entitlements:

```bash
BANNED_KEYS=( "com.apple.private.cs.debugger"
              "dynamic-codesigning"
              "com.apple.private.skip-library-validation" )
```

`dynamic-codesigning` is not private and TrollStore does not reject it —
RootHelper's `signApp` reads entitlements off the binary and keeps them. So
stripping it removed the only JIT route that does not involve executing an
unsigned page, and the file named for TrollStore became the one that could not
use TrollStore's own JIT route.

## What the patch does

**Crash sentinel.** Before each dangerous branch, a marker is written and
`fsync`'d to `Documents/husk-jit-<route>.attempt`, and unlinked once the branch
returns. A marker still present at the next launch names the route that killed
the process, and that route is skipped. One crash instead of a loop.

`fsync` before the call matters: SIGKILL gives no chance to flush afterwards.

The filename has no leading dot on purpose. Husk sets `UIFileSharingEnabled`, so
`Documents` shows up under **On My iPhone → Husk** in the Files app — deleting
the marker there retries the route, no jailbreak tooling needed.

Covered: `mapjit`, `plain`, and the standalone `husk_ios_jit_mapjit_works()`
probe, which also ends in a call into a page it wrote.

`husk_route_arm()` returns false if it cannot write the marker, and the caller
then **skips the route**. An unrecorded attempt that dies is the exact failure
this exists to prevent.

**Honest failure message.** `JITBootstrap.lastFailure` used to blame "Husk runs
inside another container app", which is the wrong thing to chase on a jailbroken
device. It now reports the real cause and the three concrete fixes.

**Packaging.** `dynamic-codesigning` is kept in `.tipa` builds. Set
`HUSK_STRIP_DYNAMIC_CODESIGNING=1` for the old behaviour. The validation block
below now expects the key in both packages, and its conditional is an `if`
rather than `[ ... ] && VAR+=...`, which under `set -e` makes the line's exit
status the failed test's.

## New API

```c
const char *husk_ios_jit_poisoned_route(void);  /* "mapjit" | "plain" | "probe" | NULL */
void        husk_ios_jit_clear_poison(void);
```

Swift side: `JITBootstrap.poisonedRoute`, `.poisonedExplanation`,
`.clearPoisonedRoutes()`.

`clearPoisonedRoutes()` has no button wired to it yet — `JITSetupView.swift` is
the place for one ("Try All Methods Again"). Deleting the marker in the Files app
does the same thing in the meantime.

## Before you rebuild

The patch stops the crash loop; it does not by itself grant JIT. Worth trying
first, since it needs no rebuild at all:

1. Delete the current install.
2. Install **`Husk.ipa`**, not `Husk.tipa`, through **full TrollStore** (not
   Lite). The extension only decides which app iOS hands the file to; TrollStore
   installs plain `.ipa` files and preserves the entitlements in the binary.
   That gets `dynamic-codesigning`, MAP_JIT succeeds, and route 3 never runs.
3. If you are on TrollStore Lite: it has no CoreTrust bypass and leans on the
   jailbreak's AMFI patches, so entitlement granting is less reliable. Do not
   disable AppSync Unified.

Fallback that sidesteps entitlements entirely: install **StikDebug** alongside
Husk. That is the debugger route and the one upstream treats as canonical.

Your log also showed `TrollStore install: false` — Husk looks for a
`_TrollStore` / `_TrollStoreLite` marker next to the bundle and found neither,
so whatever installed it did not leave one. Worth confirming what actually
performed the install.
