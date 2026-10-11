# atc profiles & fleet discovery guide

Guides: `workflow` · `profiles` (this page) · `snapshots` · `multi-agent` · `traps` (`atc guide <topic>`).

## 1. Three-tier fleet discovery

When you run `atc claim`, `atc` inspects the host outside `atc.lock` and picks a candidate in priority order:

| Tier | Candidate | Latency | What happens |
|---|---|---|---|
| `Tier 0` | Warm idle Android emulator or booted iOS Simulator matching the profile | `< 20 ms` | Immediate in-memory lease assignment; no boot wait. |
| `Tier 1` | Offline AVD in `~/.android/avd` or shutdown iOS Simulator (when a slot is free) | `3–60 s` | Reserves `state: "starting"`, boots outside `atc.lock`, waits for readiness (`sys.boot_completed=1` or `simctl bootstatus -b`), and activates the lease. |
| `Tier 2` | Offline AVD or Simulator when slots are full (`autoStopIdleOnContention: true`) | `10–90 s` | Stops the oldest unlocked warm virtual device on that platform first, then boots the requested offline device. |
| `Tier 3` | Creatable profile via `android emulator create` or `xcrun simctl create` (`--create-if-missing`) | `10–180 s` | Creates a deterministic `atc_*` AVD or iOS Simulator, then boots and leases it. |

## 2. Platform and profile filter flags

Pass only the constraints your task requires so `atc` can reuse warm virtual devices whenever possible:

| Flag | Examples | Meaning |
|---|---|---|
| `--android` / `--ios` / `--platform <p>` | `--ios`, `--android` | Select Android or iOS. On macOS, `atc` also selects iOS automatically in directories with `.xcodeproj`, `.xcworkspace`, or `Package.swift` and no `build.gradle*`. |
| `--type <type>` | `phone`, `tablet`, `foldable`, `desktop`, `watch` (alias `wear`), `xr`, `tv`, `automotive`, `resizable` | Canonical form factor across Android and iOS (`watch` and `wear` match both Wear OS and watchOS). A `resizable` AVD satisfies `phone`, `foldable`, `tablet`, or `desktop`. |
| `--api <spec>` / `--os <spec>` | `36`, `>=34`, `18.4`, `>=18.0`, `17.0..18.4` | Android SDK API level (`--api`) or iOS/watchOS/tvOS/visionOS version (`--os`). |
| `--play` / `--no-play` | `--play` | Require or exclude Google Play Store images (Android). |
| `--services <svc>` | `play`, `google_apis`, `aosp` | Exact Android system image service tier. |
| `--abi <abi>` | `arm64-v8a`, `x86_64` | CPU ABI constraint. |
| `--avd <name>` | `--avd Pixel_9`, `--avd "iPhone 16 Pro"` | Pin to a specific Android AVD name or iOS Simulator name. |
| `--udid <udid>` | `--udid D9746857-...` | Pin to an iOS Simulator UDID or physical iOS UDID / CoreDevice UUID. |
| `--serial <serial>` | `--serial emulator-5554` | Pin to a running emulator, simulator UDID, or physical device serial. |

```bash
# Any Play Store Android phone on API 35+ (prefers a warm emulator)
atc claim --type phone --api ">=35" --play

# iOS 18+ iPhone Simulator on macOS
atc claim --ios --type phone --os ">=18.0"

# Wear OS or watchOS watch
atc claim --android --type watch --api ">=35"
atc claim --ios --type watch --os ">=11.0"
```

## 3. Physical USB and Wi-Fi devices are opt-in

Developers often keep a personal Android phone or iPhone plugged into USB for charging or Wireless Debugging. `atc claim` defaults to `--kind emulator` so physical devices are never claimed unless requested.

```bash
# Physical Android phone by adb serial (from `atc status`)
atc claim --serial adb-2A191FDH3007B7-SAhkP3._adb-tls-connect._tcp

# Physical iPhone/iPad by hardware UDID (from `atc status --ios`)
atc claim --ios --kind physical --udid 00008120-000464CE2E44C01E

# Block all physical device claims on this machine (exits with code 6)
atc config set allowPhysicalDevices never
```

| Event while leased | Virtual device (AVD / Simulator) | Physical device (Android / iPhone) |
|---|---|---|
| Device drops offline | Holds `45 s` offline reservation (`firstSeenOfflineAtMs`). | Holds `15 s` USB glitch grace window. |
| Owner re-claims | Reboots virtual device in place and keeps original `expiresAtMs`. | Fails fast (`exit 1`) and records a disconnect tombstone; asks user to reconnect cable. |
| Host slot usage | Uses 1 slot (`maxRunningEmulators` or `maxRunningSimulators`). | Uses `0` virtual slots. |
