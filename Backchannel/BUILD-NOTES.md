# Build notes: Backchannel for iPhone and iPad

Every step, version and fix found while building, newest milestone last.
The plan and rules are in `CLAUDE.md` and `KICKOFF.md` (from the starter
kit) and `mikesimone/backchannel` → `apple/`.

## I0: build Signal-iOS unmodified (started 2026-11-07)

Result so far: **builds and starts in the Simulator.** Not yet built for
or run on a real iPhone, and not yet linked; that needs the iPhone
connected and App IDs registered on the signing team (below).

### Machine and versions

| What | Version |
|---|---|
| Mac | Apple Silicon, macOS 27 (Darwin 27.0.0) |
| Xcode | 27.2 (27B52), plus the Metal Toolchain 27B47 |
| Signal-iOS | `main` at `32b101abe2` (2026-11-05), version 8.37 |
| Deployment target | iOS 15.0 |
| Dependencies | Swift Package Manager; libsignal 0.106.1, RingRTC 2.74.0, GRDB 5.26.0-signal.1, SQLCipher 4.6.1-signal.1 and others, resolved by Xcode |
| Signing team | 404 Cybersecurity LLC, `YA83Q8FTH3` |
| Bundle ID prefix | `net.mikesimone.backchannel` (Brian's choice; matches Android and the Mac app) |

### Clone

The fork is `breed007/backchannel-ios` (a fork of `signalapp/Signal-iOS`,
default branch only), to move to Mike when he wants it.

```sh
git clone --depth 1 https://github.com/breed007/backchannel-ios.git
git remote add upstream https://github.com/signalapp/Signal-iOS.git
```

There are no submodules any more. A blobless full clone kept failing
partway on a flaky connection ("RPC failed; curl 56", then a checkout
that never finished); the shallow clone went through first time. Fetch
history later with `git fetch --unshallow` when the sync work needs it.

### Build for the Simulator

```sh
xcodebuild -downloadComponent MetalToolchain   # once; about 840 MB
xcodebuild -workspace Signal.xcworkspace -scheme Signal \
  -configuration Debug -destination 'generic/platform=iOS Simulator' \
  build CODE_SIGN_STYLE=Manual CODE_SIGN_IDENTITY=- \
  DEVELOPMENT_TEAM= PROVISIONING_PROFILE_SPECIFIER=
```

- **Metal Toolchain.** Xcode 27 doesn't ship it; the first build failed
  with "cannot execute tool 'metal' due to missing Metal Toolchain".
- **Spaces in the path.** `Scripts/update_plist_info.sh` ran
  `cd $PROJECT_DIR` unquoted and failed under `~/Claude Code/...`
  ("cd: /Users/breed/Claude: No such file or directory"). A symlinked path
  doesn't help: Xcode resolves it. Fixed by quoting it, marked
  `# BACKCHANNEL:`. This is the only upstream change so far, and it's a
  build script, not app code.
- **Sign the Simulator build ad hoc.** With `CODE_SIGNING_ALLOWED=NO` the
  app builds but quits at launch: it can't open its app group
  ("container_create_or_lookup_app_group_path_by_app_group_identifier:
  client is not entitled"). An ad-hoc signature (`CODE_SIGN_IDENTITY=-`)
  embeds the Simulator entitlements and it starts.
- Result: `Signal.app` (bundle ID `org.whispersystems.signal`, the
  upstream prefix) launches to Signal's welcome screen ("Continue",
  "Restore or Transfer Account") on an iPhone 17 Pro Simulator.

### For the device build (next)

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
Modes and App Groups. Beyond those, the entitlements ask for
carrier-constrained networking (Apple grants it on request) and Wi-Fi
Aware; expect to turn those off too if provisioning refuses them.

### What CLAUDE.md got wrong

- "Building": there's no `--recurse-submodules` and no
  `make dependencies` any more. Dependencies are Swift packages that
  Xcode resolves, and `BUILDING.md` now says only to open the workspace,
  set the team and turn off the capabilities above.
- It doesn't mention the Metal Toolchain download, the space-in-path
  failure, or that a Simulator build needs an (ad-hoc) signature.
- Upstream paths: `ProvisioningQRCodeViewController.swift` moved to
  `Signal/Provisioning/UserInterface/`. The other paths in its table
  still exist at `32b101abe2`.
- The capability list is longer than `BUILDING.md`'s: carrier-constrained
  networking, Wi-Fi Aware, Enhanced Security (`hardened-process`), and in
  the App Store variants VoIP push (`pushkit.unrestricted-voip`).
