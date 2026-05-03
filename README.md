# AndroidLibXrayLite (v2rayVN fork)

Android / gomobile bindings for [Xray-core](https://github.com/XTLS/Xray-core).
This repository produces `libv2ray.aar`, the native `.aar` that
[v2rayVN](https://github.com/sevaktigranyan305-netizen/v2rayNG) (our Android
client fork of v2rayNG) consumes through `V2rayNG/app/libs/libv2ray.aar`.

## The v2rayVN fork stack

| Component | Role | Repository |
|---|---|---|
| **Xray-core (fork)** | Server / core with L3 virtualnet (VLESS-as-VPN) | https://github.com/sevaktigranyan305-netizen/Xray-core |
| **3x-ui (fork)** | Admin panel with per-client virtual-IP (`vnetIp`) IPAM | https://github.com/sevaktigranyan305-netizen/3x-ui |
| **AndroidLibXrayLite (fork)** | gomobile bindings producing `libv2ray.aar` | *this repo* |
| **v2rayVN (Android client)** | Android VPN app built on top of `libv2ray.aar` | https://github.com/sevaktigranyan305-netizen/v2rayNG |

## Why this fork exists

Upstream [2dust/AndroidLibXrayLite](https://github.com/2dust/AndroidLibXrayLite)
builds its `libv2ray.aar` directly against
[XTLS/Xray-core](https://github.com/XTLS/Xray-core). That produces a `.aar`
that does **not** contain the L3 virtualnet additions (VLESS-as-VPN, per-client
`vnetIp` allocation, TUN-over-VLESS framing) that our [Xray-core
fork](https://github.com/sevaktigranyan305-netizen/Xray-core) introduces.

This fork pins the build against our Xray-core through a Go module `replace`
directive in `go.mod`:

```
replace github.com/xtls/xray-core => github.com/sevaktigranyan305-netizen/Xray-core <pinned-commit>
```

so every `libv2ray.aar` built here ships the fork's xray binary and can drive
a `VpnService`-owned TUN end-to-end.

Apart from the `replace` directive and a few minor build-script tweaks, the Go
glue code (`libv2ray_main.go`, `shared.go`, `process_fd_info.go`) is kept as
close to upstream as possible so future upstream sync is cheap.

## Build requirements

| Tool | Version | Notes |
|---|---|---|
| JDK | 17 or 21 | Either works; v2rayVN's CI uses Temurin 21. |
| Android SDK | API 36+ | `cmdline-tools;latest` + a recent `platforms;android-N` package. |
| Android NDK | `28.2.13676358` | The exact version v2rayVN's CI installs. Older NDKs may still work but this is what we pin. |
| Go | `go.mod` version (currently `1.26+`) | Use the version declared in this repo's `go.mod`, not the one declared in Xray-core's. |
| `gomobile` | latest from `golang.org/x/mobile/cmd/gomobile` | Installed on first build via `go install`. |

On Ubuntu / Debian:

```bash
# JDK + basic tooling
sudo apt install -y openjdk-21-jdk-headless unzip

# Go (skip if you already have 1.24+)
curl -fsSL https://go.dev/dl/go1.25.0.linux-amd64.tar.gz | \
    sudo tar -C /usr/local -xz
export PATH=$PATH:/usr/local/go/bin:$(go env GOPATH)/bin

# Android cmdline-tools + SDK + NDK
mkdir -p $HOME/android && cd $HOME/android
curl -fsSL -o cmdline-tools.zip \
    https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip
unzip -q cmdline-tools.zip -d sdk/cmdline-tools
mv sdk/cmdline-tools/cmdline-tools sdk/cmdline-tools/latest
export ANDROID_HOME=$HOME/android/sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$PATH
yes | sdkmanager --licenses
sdkmanager 'platforms;android-36' 'build-tools;36.0.0' 'platform-tools' \
           'ndk;28.2.13676358'
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/28.2.13676358
```

## Build instructions

```bash
git clone --recursive \
    https://github.com/sevaktigranyan305-netizen/AndroidLibXrayLite.git
cd AndroidLibXrayLite

# 1) stage geoip/geosite assets so they end up baked into the .aar
mkdir -p assets data
bash gen_assets.sh download
cp -v data/*.dat assets/

# 2) one-time gomobile install
go install golang.org/x/mobile/cmd/gomobile@latest
gomobile init

# 3) resolve modules (fork's xray-core via replace directive)
go mod tidy

# 4) build the .aar
gomobile bind -v -androidapi 24 -trimpath \
    -ldflags='-s -w -buildid=' ./
```

After the build you get two artefacts in the repository root:

- `libv2ray.aar` — the library itself (native + Kotlin/Java bridge).
- `libv2ray-sources.jar` — Java sources, useful for IDE navigation.

Copy `libv2ray.aar` into v2rayVN:

```bash
cp libv2ray.aar /path/to/v2rayNG/V2rayNG/app/libs/libv2ray.aar
```

and build the APK as described in the [v2rayVN
README](https://github.com/sevaktigranyan305-netizen/v2rayNG#readme).

## How v2rayVN consumes this

v2rayVN's CI does **not** download a pre-built `libv2ray.aar` from a GitHub
release of this repo. Instead, the v2rayVN workflow checks out this repo as a
git submodule and runs the build steps above in-tree. This lets every commit
on this repo automatically flow into the next v2rayVN APK without a manual
release step here.

The relevant v2rayVN workflow steps live in
[`.github/workflows/build.yml`](https://github.com/sevaktigranyan305-netizen/v2rayNG/blob/master/.github/workflows/build.yml)
under "Build libv2ray.aar from AndroidLibXrayLite submodule".

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `gomobile: command not found` | `$GOPATH/bin` is not on `PATH`. | `export PATH=$PATH:$(go env GOPATH)/bin` |
| `ANDROID_NDK_HOME not set` or `gomobile init` fails | NDK path not exported or wrong version. | Install `ndk;28.2.13676358` via `sdkmanager` and `export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/28.2.13676358`. |
| Build succeeds but v2rayVN crashes with `UnsatisfiedLinkError` on `libgojni.so` | `.aar` was built for an ABI v2rayVN doesn't include, or with a mismatched NDK. | Rebuild with exactly the NDK version above; v2rayVN ships `arm64-v8a`, `armeabi-v7a`, `x86`, `x86_64`. |
| L3 / VPN features (vnetIp preamble, TUN-over-VLESS) don't work in the app | `go.mod` `replace` directive didn't pick up the fork's xray-core — you're building against stock upstream. | `go mod tidy && go mod why github.com/xtls/xray-core` should show the `sevaktigranyan305-netizen/Xray-core` path. If not, check the commit SHA pinned in `go.mod` / `go.sum` and rerun `gomobile bind`. |
| `gomobile bind` hangs or OOMs on a small VM | gomobile shells out to `go build` for every ABI × every arch; each process can use ~2 GB. | Build on a box with at least 8 GB RAM, or pass `-target=android/arm64` to limit ABIs. |

## Credits

- Upstream [2dust/AndroidLibXrayLite](https://github.com/2dust/AndroidLibXrayLite)
  — the gomobile glue code (`libv2ray_main.go`, `shared.go`, etc.) is upstream
  with minor patches.
- [XTLS/Xray-core](https://github.com/XTLS/Xray-core) — the upstream of our
  core fork.
- [gomobile](https://pkg.go.dev/golang.org/x/mobile/cmd/gomobile) — the Go →
  Android binding toolchain.

## License

Same license as upstream AndroidLibXrayLite. See the individual upstream
sources for details.
