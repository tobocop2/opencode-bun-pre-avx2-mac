# macos-baseline-builds

**Baseline (non-AVX2) macOS x64 builds of bun and opencode**, for pre-Haswell Intel Macs — the builds that don't exist upstream.

**⬇️ Download:** [Releases](../../releases/latest)

> ⚠️ **Unofficial.** Not affiliated with oven-sh or anomalyco. Built from their own sources with their own build scripts. Provided as proof the builds are possible, and as a stopgap while the upstream PRs land.

## The problem

If your Intel Mac predates Haswell (2013), `opencode` dies instantly with `SIGILL` — and the `-baseline` package that's supposed to save you is **the exact same binary**:

```
5b051698544dd70a686e50cce37a6f97aa6742dd5d76a2374481e7ae546662ce  opencode-darwin-x64.zip
5b051698544dd70a686e50cce37a6f97aa6742dd5d76a2374481e7ae546662ce  opencode-darwin-x64-baseline.zip
```

Same file. Both AVX2. That's because bun's darwin packages are identical too:

```
ea2f223e94bb2f4bf3050895113c3cf346438f6fa0501c8532284e063f72f7a0  @oven/bun-darwin-x64
ea2f223e94bb2f4bf3050895113c3cf346438f6fa0501c8532284e063f72f7a0  @oven/bun-darwin-x64-baseline
```

…which is because **oven-sh/WebKit never builds a baseline macOS WebKit**, so bun can't build a baseline macOS bun:

```
bun-webkit-macos-amd64.tar.gz            HTTP 200
bun-webkit-macos-amd64-baseline.tar.gz   HTTP 404
```

Upstream reports: [opencode#29039](https://github.com/anomalyco/opencode/issues/29039), [opencode#24876](https://github.com/anomalyco/opencode/issues/24876).

## What's here

| artifact | what it is |
|---|---|
| `bun-darwin-x64-baseline` | bun built for `darwin-x64` at `-march=nehalem`, against a baseline macOS WebKit |
| `opencode-darwin-x64-baseline` | opencode v1.18.1 compiled against that bun |

Most people want the **opencode** one.

## Does it actually work?

| binary | AVX2 (`ymm`) instructions |
|---|---|
| shipped `opencode-darwin-x64-baseline` | 248,019 |
| **this `opencode-darwin-x64-baseline`** | **14,407** |
| shipped `@oven/bun-linux-x64-baseline` *(known-good on real pre-AVX2 hardware)* | 12,328 |

A baseline build isn't "zero AVX2" — the residual sits behind CPUID runtime dispatch. What matters is the profile matches the **linux** baseline, which I verified on a real pre-AVX2 CPU (Sandy Bridge Xeon, AVX + SSE4.2, no AVX2): the linux baseline runs `opencode run "say hi"` to completion, while the AVX2 build crashes (~21 s, ~3.9 GB RSS, segfault).

**Honest caveat:** I could not test the *macOS* binaries on a real pre-AVX2 Mac — no cloud provider rents one. The evidence is (a) the AVX2 profile above, and (b) the same `-march=nehalem` recipe empirically working on pre-AVX2 silicon on linux. If you try it on a real Mac, please report back.

## How they were built

Host: Ubuntu, x86_64. Nothing here is a fork — both use the projects' own build scripts.

**1. Baseline macOS WebKit** — oven-sh/WebKit's own cross-compile recipe, with the one flag that's never been used together with macOS:

```bash
git clone https://github.com/oven-sh/WebKit && cd WebKit
git checkout 4895f45dfbd0d1226c4d41799887bc0ecb9f341b   # bun main's WEBKIT_VERSION
MACOS_ARCH=x86_64 MARCH_FLAG="-march=nehalem" WEBKIT_RELEASE_TYPE=Release \
USE_MIMALLOC=ON USE_EXTERNAL_MIMALLOC=ON bash macos-cross-release.sh
# -> libJavaScriptCore.a with 0 AVX2 instructions (vs 58,967 in the shipped one)
```

**2. bun** — stage that WebKit where bun expects its prebuilt, then build:

```bash
cp -a bun-webkit/. ~/.bun/build-cache/webkit-4895f45dfbd0d122-macos-baseline/
rm -rf ~/.bun/build-cache/webkit-4895f45dfbd0d122-macos-baseline/include/unicode
printf '4895f45dfbd0d1226c4d41799887bc0ecb9f341b-baseline' > .../.identity
bun scripts/build.ts --profile=release --os=darwin --arch=x64 --baseline=true --lto=off
```

**3. opencode** — stage that bun as the compile runtime, then use opencode's own build:

```bash
cp build/release/bun ~/.bun/install/cache/bun-darwin-x64-baseline-v1.3.14
bun packages/opencode/script/build.ts    # targets trimmed to darwin-x64 avx2:false
```

## Upstream fix

These artifacts are a stopgap. The real fix is two small PRs:

- **oven-sh/WebKit** — add a `bun-webkit-macos-amd64-baseline` lane: `<PR link>`
- **oven-sh/bun** — add `{ os: "darwin", arch: "x64", baseline: true, … }` to `buildPlatforms`: `<PR link>`

## Provenance

- bun: built from `main` @ `1e35118008` (reports as `1.4.0-dev`, **not** an official 1.3.14 baseline)
- opencode: `v1.18.1`, compiled against the bun above
- WebKit: `4895f45dfbd0d1226c4d41799887bc0ecb9f341b`

```
bede5eb3c917fa353d6b067fd7055d42774db7a7d5a09c21fa6cdd05e3d8efee  bun-darwin-x64-baseline
7daf210676d2efa9627619ab1823b141c861aa362f8a828db836472d9790e61f  opencode-darwin-x64-baseline
```

## License

bun and opencode are MIT-licensed; these are builds of their sources. The macOS SDK used at build time is subject to Apple's SLA.
