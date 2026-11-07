# Build notes: Backchannel for iPhone and iPad

Every step, version and fix found while building, newest milestone last.
The plan and rules are in `CLAUDE.md` and `KICKOFF.md` (from the starter
kit) and `mikesimone/backchannel` → `apple/`.

## I0: build Signal-iOS unmodified (started 2026-11-07)

Result so far: **builds, installs and starts on a real iPhone beside
official Signal, but an unmodified iPhone build can't link** (below). The
link test moves to an iPad (arriving 2026-11-08), where Signal offers
linking as it ships.

### Machine and versions

| What | Version |
|---|---|
| Mac | Apple Silicon, macOS 27 (Darwin 27.0.0) |
| Xcode | 27.2 (27B52), plus the Metal Toolchain 27B47 |
| Signal-iOS | release tag `8.35.1.1964` (`4f019b8`), the same version as official Signal on the test iPhone |
| Deployment target | iOS 15.0 |
| Dependencies | CocoaPods, checked in as the `Pods` submodule: LibSignalClient 0.105.0, SignalRingRTC 2.73.0, GRDB.swift 5.26.0 with SQLCipher 4.6.1 |
| Signing team | 404 Cybersecurity LLC, `YA83Q8FTH3` |
| Bundle ID prefix | `net.mikesimone.backchannel` (Brian's choice; matches Android and the Mac app) |

### Build from release tags, not `main`

The first builds came from `main` (`32b101abe2`, 8.37 in progress). Don't:

- `main` is unreleased. Its registration has a new paid "Signal login"
  step that loads an App Store product (`fetchSignalLoginProduct`). Under
  another bundle ID there's no product, and `owsFailDebug("couldn't fetch
  product")` in `RegistrationCoordinatorImpl.nextStepForSignalLogin` stops
  a Debug build. The registration state is saved first, so every launch
  after that crashed again, and Signal showed "Couldn't Open Signal"
  (Export Database, Submit Debug Log, Open Signal Instead).
- `main` has also moved from CocoaPods to Swift packages, so it builds
  differently from the releases the fork will follow (I3 merges release
  tags only).

The I0 work is on branch `backchannel-i0-release`, from `8.35.1.1964`.
(`backchannel-i0` has the same commits on `main`, kept for reference.)

### Clone and dependencies

The fork is `breed007/backchannel-ios` (a fork of `signalapp/Signal-iOS`,
default branch only), to move to Mike when he wants it.

```sh
git clone --depth 1 https://github.com/breed007/backchannel-ios.git
cd backchannel-ios
git remote add upstream https://github.com/signalapp/Signal-iOS.git
git fetch --depth 1 upstream tag 8.35.1.1964
git checkout -b backchannel-i0-release 8.35.1.1964
git submodule update --init --depth 1 Pods \
  SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests
Pods/SignalRingRTC/bin/set-up-for-cocoapods   # downloads RingRTC 2.73.0
```

The last two lines are what `make dependencies` does, minus its
`git -C Pods clean`/`reset`. A blobless full clone kept failing partway
on a flaky connection ("RPC failed; curl 56", then a checkout that never
finished); the shallow clone went through. Fetch history later with
`git fetch --unshallow` when the sync work needs it.

### One-time Xcode setup

```sh
xcodebuild -downloadComponent MetalToolchain   # about 840 MB
```

Xcode 27 doesn't ship it, and the build fails with "cannot execute tool
'metal' due to missing Metal Toolchain" without it.

### Upstream changes so far

All marked `BACKCHANNEL:`:

1. `Scripts/update_plist_info.sh` ran `cd $PROJECT_DIR` unquoted and
   failed under `~/Claude Code/...` ("cd: /Users/breed/Claude: No such
   file or directory"). A symlinked path doesn't help: Xcode resolves it.
   Now quoted.
2. The three Debug entitlements files (below).
3. `Signal/Signal-Info.plist`: `CFBundleDisplayName` is "Backchannel"
   instead of `${PRODUCT_NAME}`, so the test build isn't mistaken for
   official Signal on the same phone. `PRODUCT_NAME`, the executable and
   the module stay "Signal" until I1.

### Build for the Simulator

```sh
xcodebuild -workspace Signal.xcworkspace -scheme Signal \
  -configuration Debug -destination 'generic/platform=iOS Simulator' \
  build CODE_SIGN_STYLE=Manual CODE_SIGN_IDENTITY=- \
  DEVELOPMENT_TEAM= PROVISIONING_PROFILE_SPECIFIER=
```

Sign the Simulator build ad hoc. With `CODE_SIGNING_ALLOWED=NO` it builds
but quits at launch, because it can't open its app group
("container_create_or_lookup_app_group_path_by_app_group_identifier:
client is not entitled"). With the ad-hoc signature it starts to Signal's
welcome screen. (Checked on `main`; the release builds the same way.)

### Build for an iPhone

Debug builds use `Signal/Signal.entitlements`,
`SignalNSE/SignalNSE.entitlements` and
`SignalShareExtension/SignalShareExtension.entitlements`. With the
prefix, the team needs:

- App IDs `net.mikesimone.backchannel.signal`,
  `net.mikesimone.backchannel.signal.SignalNSE` and
  `net.mikesimone.backchannel.signal.shareextension`;
- app groups `group.net.mikesimone.backchannel.signal.group` (and
  `.staging`);
- keychain group `$(AppIdentifierPrefix)net.mikesimone.backchannel.signal`.

Per Signal's `BUILDING.md`, turn off Push Notifications, Apple Pay,
Communication Notifications and Data Protection, and keep Background
Modes and App Groups. The Debug entitlements drop those, plus
carrier-constrained networking and Wi-Fi Aware, which Apple grants only
on request. Associated Domains, iCloud key-value storage, Enhanced
Security (`hardened-process`) and the keychain group stay, and provision
without trouble.

```sh
xcodebuild -workspace Signal.xcworkspace -scheme Signal \
  -configuration Debug -destination 'id=<device UDID>' \
  -allowProvisioningUpdates -allowProvisioningDeviceRegistration build \
  DEVELOPMENT_TEAM=YA83Q8FTH3 SIGNAL_BUNDLEID_PREFIX=net.mikesimone.backchannel \
  CODE_SIGN_STYLE=Automatic
xcrun devicectl device install app --device <device UDID> \
  ~/Library/Developer/Xcode/DerivedData/<...>/Build/Products/Debug-iphoneos/Signal.app
```

The team and prefix are passed on the command line, so nothing about the
signing team is committed (CLAUDE.md rule 10). Xcode, signed in to an
Apple ID on the team, registered the App IDs and app groups itself
(Brian approved this, 2026-11-07) and made "iOS Team Provisioning
Profile" profiles for all three bundle IDs. The build is signed by team
`YA83Q8FTH3`, `application-identifier`
`YA83Q8FTH3.net.mikesimone.backchannel.signal`.

Installed on an iPhone 18 Pro Max (2026-11-07) next to official Signal
8.35.1 from the App Store; it started to Signal's welcome screen, named
Backchannel. Removed again afterward, so nobody taps through to
registration by mistake.

### An unmodified iPhone build can't link

Signal decides between registering and linking by device type alone:

```swift
// Signal/Registration/RegistrationUtils.swift
static var isPrimaryByDefault: Bool {
    return !UIDevice.current.isIPad
}
```

On an iPhone the welcome screen offers "Continue" (register a number) and
"Restore or Transfer Account" (Secure Backup, transfer from an old phone,
or register without transferring). The link button (a chain icon,
`RegistrationSplashViewController`) is added only when
`isPrimaryByDefault` is false, which means only on an iPad. Nothing in
8.35.1 lets an iPhone link without a code change.

Registering is never the answer: it takes the account from official
Signal (CLAUDE.md rule 2).

So:

- **I0's link test** runs on an iPad with this same unmodified build:
  the iPad shows Signal's link QR code, and official Signal on the iPhone
  scans it.
- **I1** makes the iPhone linked-only. Returning `false` from
  `isPrimaryByDefault` sends an iPhone straight to Signal's own link
  screen (`ProvisioningController`); registration is then hidden.

Linking a test build that lives on the same iPhone as official Signal
means showing its QR code on another screen. For that, `devicectl device
capture screenshot` can read the phone's screen over the cable, a QR
decoder on the Mac can pull the code out, and the Mac can show it large,
refreshed each time the code changes (every 45 s). That's the same
problem I2's first-run flow solves for users.

### What CLAUDE.md got wrong

- "Signal iOS 8.22 (2026-08-05) added linked iPhones, so upstream already
  has the linking path": **not in 8.35.1.** The provisioning code is
  there, but only an iPad reaches it (above). I1 has to open it on
  iPhone.
- "Building": right for release tags (`Pods` submodule,
  `make dependencies`), but `main` has moved to Swift packages. When a
  release ships from that, the build steps change.
- It doesn't mention the Metal Toolchain download, the space-in-path
  failure, or that a Simulator build needs an (ad-hoc) signature.
- Upstream paths: `ProvisioningQRCodeViewController.swift` is in
  `Signal/Provisioning/UserInterface/`. The other paths in its table
  exist at `8.35.1.1964`.
- The capability list is longer than `BUILDING.md`'s: carrier-constrained
  networking, Wi-Fi Aware, Enhanced Security (`hardened-process`), and in
  the App Store variants VoIP push (`pushkit.unrestricted-voip`).
