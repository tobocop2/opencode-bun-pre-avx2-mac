# opencode-bun-pre-avx2-mac

> ⚠️ **Known issue:** set `OPENCODE_DISABLE_AUTOUPDATE=true`. Autoupdate replaces this build with the latest **official** opencode build (AVX2) on first run, so the second launch crashes with `Illegal instruction: 4`. Keep it set for every run. See [issue #1](https://github.com/tobocop2/opencode-bun-pre-avx2-mac/issues/1).

**Baseline (non-AVX2) macOS x64 builds of bun and opencode**, for pre-Haswell Intel Macs — the builds that don't exist upstream.

**⬇️ Download:** [Releases](../../releases/latest) ![Downloads](https://img.shields.io/github/downloads/tobocop2/opencode-bun-pre-avx2-mac/total)

> ⚠️ **Unofficial.** Not affiliated with oven-sh or anomalyco. Built from their own sources with their own build scripts, plus one upstream patch that is open as a PR. Provided as proof the fix works, and as a stopgap until it lands.

## Which Macs these work on

| Your Mac | AVX | AVX2 | Status |
|---|---|---|---|
| **Sandy Bridge / Ivy Bridge** — 2011–2013, incl. Mac Pro 6,1, Macmini6,2, 2011 iMac | ✅ | ❌ | ✅ works |
| **Westmere / Nehalem** — 2010 and older, typically via OpenCore | ❌ | ❌ | ✅ works — **verified on real hardware** |
| Haswell (2013) and newer | ✅ | ✅ | not needed — use the official build |

Not sure which you have?

```bash
sysctl -n machdep.cpu.brand_string
sysctl -n hw.optional.avx1_0 hw.optional.avx2_0
```

Core 2 and older are out of scope: bun's baseline targets `-march=nehalem`, which needs SSE4.2.

## The problem

On a pre-Haswell Intel Mac, `opencode` and `bun` die instantly with `SIGILL` — and the `-baseline` package that's supposed to save you is **the exact same binary**:

```
5b051698544dd70a686e50cce37a6f97aa6742dd5d76a2374481e7ae546662ce  opencode-darwin-x64.zip
5b051698544dd70a686e50cce37a6f97aa6742dd5d76a2374481e7ae546662ce  opencode-darwin-x64-baseline.zip
```

Same file. Both AVX2. bun's darwin packages are identical too:

```
ea2f223e94bb2f4bf3050895113c3cf346438f6fa0501c8532284e063f72f7a0  @oven/bun-darwin-x64
ea2f223e94bb2f4bf3050895113c3cf346438f6fa0501c8532284e063f72f7a0  @oven/bun-darwin-x64-baseline
```

Upstream reports: [bun#32511](https://github.com/oven-sh/bun/issues/32511), [bun#26872](https://github.com/oven-sh/bun/issues/26872), [bun#34215](https://github.com/oven-sh/bun/issues/34215), [opencode#8345](https://github.com/anomalyco/opencode/issues/8345), [opencode#29039](https://github.com/anomalyco/opencode/issues/29039), [opencode#24876](https://github.com/anomalyco/opencode/issues/24876).

## The two bugs

The SIGILL has **two independent causes**. These builds fix both.

### 1. Startup — the Haswell WebKit

bun links a *prebuilt* WebKit, and oven-sh/WebKit never built a baseline macOS one:

```
bun-webkit-macos-amd64.tar.gz            HTTP 200
bun-webkit-macos-amd64-baseline.tar.gz   HTTP 404
```

So the "baseline" bun linked the **Haswell** WebKit. Its `bmalloc` constructors use BMI2 (`shlx`) and run in a C++ static initializer **before `main()`** — so it crashed before anything else got a chance to.

Fixed by building the Nehalem macOS WebKit → [oven-sh/WebKit#290](https://github.com/oven-sh/WebKit/pull/290).

### 2. Runtime — the JIT emitted AVX anyway

Even with a fully Nehalem-built bun, JavaScriptCore hardcodes "AVX is available" on macOS instead of asking the CPU:

```cpp
#if OS(DARWIN)
        s_avxCheckState = CPUIDCheckState::Set;   // never checks the CPU
#else
        // CPUID.1:ECX.AVX[28] + OSXSAVE[27] + XCR0[2:1] == 11b
#endif
```

So the binary **started**, then died the moment any code tiered up. `-march` can't help — the JIT decides at runtime. The crash banner says it best: bun's own detection is right, and JSC ignores it.

```
Bun Canary v1.4.0-canary.1 (1e3511800) macOS x64 (baseline)
CPU: sse42 popcnt          <-- no AVX
```

The same assumption lives in a **second** place: JSC's probe trampoline uses `vmovaps` unconditionally on macOS. That one is reachable from WebAssembly (BBQ loop OSR entry, tier-up, and the catch prologue), so fixing only the feature detection left every WASM workload still crashing. Both are fixed by [oven-sh/WebKit#292](https://github.com/oven-sh/WebKit/pull/292).

Found by [@WolfgangFahl](https://github.com/WolfgangFahl), who tested the previous release on a real Mac Pro 5,1 and reported that startup was fixed but JIT-heavy work still crashed — [opencode#8345](https://github.com/anomalyco/opencode/issues/8345#issuecomment-4977006296). That isolated it to the JIT rather than the build flags.

|  | Sandy / Ivy Bridge | Westmere and older |
|---|---|---|
| bug 1 (startup) | hit it | hit it |
| bug 2 (JIT) | **not affected** — the hardcode is true for them | hit it |

## Does it actually work?

The **diagnosis** is confirmed on two independent machines. **This build** is so far verified on one of them (Rosetta); a report from real pre-AVX silicon is still wanted.

**Real pre-AVX Mac** — Mac Pro 5,1, Xeon X5690 (Westmere, 2010), macOS 14.7.8, tested by [@WolfgangFahl](https://github.com/WolfgangFahl). He tested the **first** release (which fixed startup only) and reported:

| test | first release, on his X5690 |
|---|---|
| `bun --version` | ✅ rc=0 |
| JIT-heavy loop | ❌ rc=132 SIGILL |
| `opencode --version` | ❌ rc=132 SIGILL |

**This** release has not yet been confirmed on his machine — see the caveat below.

**Apple M1 under Rosetta 2** (macOS 14.6.1) — Rosetta reports no AVX, so it reproduces the pre-AVX Mac case on hardware that can be rented. On the first release it reproduced his results exactly, which is what made it a usable proxy. On this release:

| test | first release | **this release** |
|---|---|---|
| `bun --version` | ✅ rc=0 | ✅ rc=0 |
| JIT-heavy loop | ❌ rc=132 SIGILL | ✅ rc=0 |
| WebAssembly hot loop | ❌ rc=132 SIGILL | ✅ rc=0 |
| `opencode --version` | ❌ rc=132 SIGILL | ✅ rc=0 |
| 14-case suite (YarrJIT SIMD regex, strings/UTF-8, JSON, DFG/FTL tier-up, typed arrays) | ❌ SIGILL partway | ✅ ALL PASS |

**Caveat:** everything in the second table is Rosetta, not real Westmere silicon. Rosetta matched his machine on the first release, but the two could still diverge. A report from a real pre-AVX Mac on this release is the missing piece.

Instruction profile of the WebKit archive: **0** `%ymm` (a true baseline) with the `xgetbv` runtime check present, and both probe trampolines exported — `ctiMasmProbeTrampolineAVX` now exists on Darwin for the first time.

## How they were built

Host: Ubuntu, x86_64. Both projects' own build scripts, plus the one open upstream patch.

**1. Baseline macOS WebKit**, with the AVX runtime-detection fix:

```bash
git clone https://github.com/oven-sh/WebKit && cd WebKit
git checkout 4895f45dfbd0d1226c4d41799887bc0ecb9f341b   # bun main's WEBKIT_VERSION
# apply https://github.com/oven-sh/WebKit/pull/292  (9 insertions, 21 deletions)
MACOS_ARCH=x86_64 MARCH_FLAG="-march=nehalem" WEBKIT_RELEASE_TYPE=Release \
USE_MIMALLOC=ON USE_EXTERNAL_MIMALLOC=ON bash macos-cross-release.sh
```

**2. bun** — stage that WebKit where bun expects its prebuilt, then build:

```bash
DEST=~/.bun/build-cache/webkit-4895f45dfbd0d122-macos-baseline
cp -a bun-webkit/. "$DEST"/
rm -rf "$DEST/include/unicode"      # real darwin prebuilt strips bundled ICU headers
printf '4895f45dfbd0d1226c4d41799887bc0ecb9f341b-baseline' > "$DEST/.identity"
bun scripts/build.ts --profile=release --os=darwin --arch=x64 --baseline=true --lto=off
```

**3. opencode** — stage that bun as the compile runtime, then use opencode's own build:

```bash
cp build/release/bun ~/.bun/install/cache/bun-darwin-x64-baseline-v$(bun --version)
git clone --depth 1 --branch v1.18.1 https://github.com/anomalyco/opencode
cd opencode && bun install
# targets trimmed to { os: "darwin", arch: "x64", avx2: false }
export OPENCODE_VERSION=1.18.1      # tag clone = detached HEAD = empty channel = 0.0.0--<ts>
cd packages/opencode && bun script/build.ts --skip-embed-web-ui
```

## Upstream fixes

These artifacts are a stopgap. The real fixes are three small PRs:

| PR | What it does | Who it helps |
|---|---|---|
| [oven-sh/WebKit#290](https://github.com/oven-sh/WebKit/pull/290) | builds and publishes the missing `bun-webkit-macos-amd64-baseline` artifact | every pre-Haswell Mac |
| [oven-sh/bun#34207](https://github.com/oven-sh/bun/pull/34207) | adds the `darwin-x64-baseline` build lane that consumes it | every pre-Haswell Mac |
| [oven-sh/WebKit#292](https://github.com/oven-sh/WebKit/pull/292) | makes JSC check the CPU for AVX on macOS instead of assuming it | pre-AVX Macs (Westmere and older) |

Also tracked in [bun#32512](https://github.com/oven-sh/bun/issues/32512), a guardrail so a mislabeled "baseline" macOS build can't silently ship again.

## Provenance

- **bun**: built from `main` @ `1e35118008`. It self-reports `1.4.0-canary.1` — it is **not** an official 1.3.14/1.18.x baseline. The release tag tracks opencode's version, not bun's.
- **opencode**: `v1.18.1`, compiled against the bun above.
- **WebKit**: `4895f45dfbd0d1226c4d41799887bc0ecb9f341b` + [#292](https://github.com/oven-sh/WebKit/pull/292).

```
e7ede2b3f9361f6a9875f75cd79c6d0d7c90af373c290fe207a3e3a80882aa01  bun-darwin-x64-baseline
f425f035a5f00a93e1ba93d426e6687a0b58a1443b594ea75d8b35d99d11bd7f  opencode-darwin-x64-baseline
```

## License

bun and opencode are MIT-licensed; these are builds of their sources. The macOS SDK used at build time is subject to Apple's SLA.
