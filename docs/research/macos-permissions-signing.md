# macOS permissions and signing for the Stream Deck helper

Research for issue #6. Question: when the Stream Deck plugin's Node process spawns our Swift helper (AX reads of Teams, `CGEvent` posting), which process does TCC attribute the grant to, and what signing keeps that grant across rebuilds of a binary built on GitHub Actions with no Apple Developer Program membership?

Sources are numbered inline; full list at the end. Items marked **[unverified]** were not confirmed against a primary source.

## Recipe (summary)

1. **Sign the helper with a stable, self-made identity, not ad-hoc.** Create a self-signed "Code Signing" certificate in Keychain Access on the Mac [17] and sign with `codesign -f -s "<name>" -i <bundle-id> helper`. Ad-hoc (`-s -`) ties the designated requirement to that exact build, so every rebuild re-prompts [4][2].
2. **Expect the grant to land on Stream Deck.app, not the helper.** A helper spawned as a child of an app has the app as its TCC "responsible code" [1][2]. The prompt names Stream Deck and the toggle appears under Stream Deck in System Settings. Make the install docs say so.
3. **Two privileges, one pane.** AX reads need `Accessibility`; `CGEvent.post` needs `PostEvent`, which *also* appears under Privacy & Security > Accessibility; only `ListenEvent` (event taps) appears under Input Monitoring [6]. We need no Input Monitoring unless we add an event tap.
4. **No entitlements, no hardened runtime** for a non-sandboxed helper [5][8].
5. **Strip quarantine on install** (`xattr -d com.apple.quarantine`) because Sequoia removed the Control-click override and bare binaries fail `spctl` assessment [10][13][14].
6. **Build on `macos-14`/`macos-15` (arm64 M1 runners) with an explicit `-target arm64-apple-macos14`** (or SwiftPM `platforms: [.macOS(.v14)]`) so the deployment target doesn't float with the runner image [19][23][25].

## TCC attribution ("responsible code")

- Quinn (Apple DTS) states the rule set directly: run from Terminal, the responsible code is Terminal; "Spawned as a child process (a helper tool) by an app — The system treats the app as the responsible code"; a `launchd` daemon/agent is its own responsible code unless installed via `SMAppService` or tagged with `AssociatedBundleIdentifiers` [1]. The exact algorithm "is not documented, has changed in the past, and may well change in the future" [1].
- The design intent: "if an app contains a helper tool and the helper tool triggers a MAC prompt, we want: The app's name and usage description to appear in the alert. The user's decision to be recorded for the whole app, not that specific helper tool. That decision to show up in System Settings under the app's name" [2].
- Permissions are inherited down the process tree; `launchctl procinfo <pid>` prints `responsible pid`/`responsible path`, and a tool launched from iTerm2 shows iTerm2 there [3]. Our chain is Stream Deck.app -> bundled Node -> helper, so the helper's responsible process is Stream Deck.app. **[unverified]**: that the two-hop chain resolves the same as one hop; confirm with `launchctl procinfo` on the helper's pid.
- Consequence: the grant is one entry keyed on Stream Deck.app's (Elgato-signed) code identity, so our helper's signature only matters when attribution falls to the helper itself.
- `AXIsProcessTrustedWithOptions` "Returns whether the current process is a trusted accessibility client"; `kAXTrustedCheckOptionPrompt` prompts asynchronously without changing the return value [7]. Apple does not document bundle vs bare-binary differences; the check resolves via the responsible process, so a bare helper under Stream Deck returns true once Stream Deck is toggled on. For CGEvent use `CGPreflightPostEventAccess`/`CGRequestPostEventAccess` [5].
- The link breaks only if the child does something "odd" such as daemonising, or uses the undocumented `responsibility_spawnattrs_setdisclaim` spawn attribute (Chromium/Electron do) [2][3]. Node's `child_process.spawn` does neither. **[unverified]**: Stream Deck's Node launch does not disclaim; if it did, the prompt would name Node/the helper.
- Terminal testing caveat: a grant made while running from Terminal belongs to Terminal and does not carry over to Stream Deck [1].
- TCC expects a Mach-O main executable; scripts as `CFBundleExecutable` cause TCC problems [2]. The helper is Mach-O.

## Code signing and grant persistence

- TCC persists decisions by recording the code's designated requirement (DR): "Each time your app tries to access the microphone, macOS checks that this version of the app satisfies the original DR" [4]. The database stores it in the `csreq` column as a compiled requirement blob [29].
- "Ad hoc signed code ... has a DR but it's tied to that specific version of the code ... If you tweak the code and run it again, macOS repeats that prompt" [4]; "This relies on your code having a stable code signing identity" [2]. `codesign` itself says ad-hoc "identifies exactly one instance of code" [9].
- `codesign` "applies a default designated requirement based on the code signing identity you supply" [4]. With a self-signed certificate "the leaf and root refer to the same single certificate" [18], so the default DR pins to that certificate plus the `-i` identifier; a rebuild signed with both satisfies it. **[unverified]**: exact default DR text for a non-Apple identity; check with `codesign -d -r- helper` and keep the identifier stable (the man page warns against reusing identifiers across programs [9]).
- Quinn on identity: "your code must be signed with a stable signing identity. So it's fine to use Apple Development signing ... You don't need to do the Developer ID signing / notarisation dance" [5]. Without an Apple account, a Keychain Access self-signed certificate is the available stable identity; Apple notes only that self-signed certs "can be useful if the person signing it is trusted" [17].
- Hardened runtime (`-o runtime`) is required only "To upload a macOS app to be notarized" [8]; it adds library validation and other restrictions [8][30]. Not needed here. Skip `--timestamp` as well **[unverified]**.
- Reset for testing: `tccutil reset Accessibility <bundle-id>` [16]; Stream Deck's bundle id is the relevant one.

## Gatekeeper and quarantine

- Gatekeeper assesses quarantined downloads on first open [28]. GitHub artifact zips downloaded in a browser carry `com.apple.quarantine`, and extraction propagates it. **[unverified]**: `gh run download`/`curl` do not set the attribute (they are not quarantine-aware); verify with `xattr -l`.
- A quarantined, non-notarized bare tool is blocked when run, and `spctl -a` rejects it with "the code is valid but does not seem to be an app"; removing the attribute lets it run [13]. Double-clicking a tool in Finder always blocks (document logic, known bug) [12].
- macOS Sequoia: "users will no longer be able to Control-click to override Gatekeeper"; they must use System Settings > Privacy & Security [10], where an "Open Anyway" button appears after a failed launch [11]. `spctl --master-disable` is gone; disabling Gatekeeper on Sequoia "requires setting a Management Option" [15].
- Personal-use fix: `xattr -d com.apple.quarantine <helper>` (or `-r` on the `.sdPlugin` folder) [14] in the install step, once per copy. Signing does not remove the attribute **[unverified]**.

## Entitlements

- "Posting events with `CGEvent` does not require any specific entitlements. It does, however, require user approval in System Settings > Privacy & Security" [5]. AX likewise needs no entitlement outside the sandbox; AX-gated APIs are incompatible with App Sandbox, so do not sandbox the helper [6]. Embed an `Info.plist` with `CFBundleIdentifier` (`-sectcreate __TEXT __info_plist`) so TCC logs and `tccutil` have a stable id **[unverified: necessity for a non-bundled child]**.

## GHA toolchain and runtime compatibility

- `macos-14` and `macos-15` labels are Apple Silicon (M1, arm64); Intel is `macos-15-intel` [19]. Current images: macos-14 = 14.8.9, Xcode 15.4 default, SDKs 14.0–15.2 [20]; macos-15 = 15.7.9, Xcode 16.4 default, SDKs 15.0–26.2 [21]. Xcode 15.4 ships Swift 5.10, Xcode 16.4 Swift 6.1 [22]; select with `sudo xcode-select -s /Applications/Xcode_16.4.app`.
- The Swift runtime ships in the OS since Swift 5 ABI stability [24], so an arm64 build for macOS 13+ needs no bundled runtime.
- Deployment target: pass `-target arm64-apple-macos14` (Apple's own makefile example uses `-target x86_64-apple-macos10.12` / `arm64-apple-macos11`; merge with `lipo -create` for universal) [23]. SwiftPM defaults to macOS 10.13 in the 6.1 toolchain and 12.0 on `main` [25]; set `platforms:` explicitly. **[unverified]**: bare `swiftc` default target when neither is given.
- Native arm64 output from an arm64 runner runs on an Apple Silicon Mac at or above the deployment target; no universal build is needed for one personal Mac.

## Constraints on the pipeline and install step

- **Build (GHA):** `swift build -c release` with explicit target; either ad-hoc sign on the runner (placeholder only) or import the self-signed `.p12` into a temporary keychain (`security create-keychain`, `security import ... -f pkcs12`, `security set-key-partition-list -S apple-tool:,apple:`) and sign there [26]. Simpler: sign on the Mac at install time, so the private key never leaves the Mac.
- **Install (Mac):** copy helper into `.sdPlugin`; `xattr -d com.apple.quarantine`; `codesign -f -s "<self-signed>" -i <id> helper`; `codesign -d -r- helper` to confirm the DR is certificate-based, not `cdhash`; restart Stream Deck; on first use, approve **Stream Deck** under Accessibility; verify with `launchctl procinfo`.
- **Plugin:** Node 20/24 from the manifest `Nodejs.Version` [27]; spawn the helper with a plain `child_process.spawn`, no `detached`/`setsid`, to keep the responsibility chain [2].
- **Rebuild:** same cert + identifier keeps the DR; the grant lives on Stream Deck.app anyway. If prompts recur, run `tccutil reset Accessibility com.elgato.StreamDeck` [16].

## Sources

1. Quinn "The Eskimo!" (Apple DTS), *How to grant command line tools full disk access*, https://developer.apple.com/forums/thread/756510
2. Quinn, *On File System Permissions* (Responsible Code, TCC, TCC and Main Executables), https://developer.apple.com/forums/thread/678819
3. Tor Arne Vestbø (Qt), *The Curious Case of the Responsible Process*, https://www.qt.io/blog/the-curious-case-of-the-responsible-process
4. Apple TN3127 *Inside Code Signing: Requirements*, https://developer.apple.com/documentation/technotes/tn3127-inside-code-signing-requirements
5. Quinn, thread *CgEvent post works from command line, but not from app*, https://developer.apple.com/forums/thread/724603
6. Quinn, thread on `PostEvent`/`ListenEvent`/`Accessibility` mapping, https://developer.apple.com/forums/thread/820594
7. Apple, `AXIsProcessTrustedWithOptions`, https://developer.apple.com/documentation/applicationservices/1459186-axisprocesstrustedwithoptions
8. Apple, *Hardened Runtime*, https://developer.apple.com/documentation/security/hardened-runtime
9. `codesign(1)`, https://keith.github.io/xcode-man-pages/codesign.1.html
10. Apple, *Updates to runtime protection in macOS Sequoia*, https://developer.apple.com/news/?id=saqachfa
11. Apple, *Safely open apps on your Mac*, https://support.apple.com/en-us/102445
12. Quinn, *Resolving Gatekeeper Problems*, https://developer.apple.com/forums/thread/706379
13. Thread *Gatekeeper blocks command line tool after signing and notarization*, https://developer.apple.com/forums/thread/757647
14. `xattr(1)`, https://ss64.com/mac/xattr.html
15. `spctl(8)`, https://ss64.com/mac/spctl.html
16. `tccutil(1)`, https://ss64.com/mac/tccutil.html
17. Apple, *Create self-signed certificates in Keychain Access*, https://support.apple.com/guide/keychain-access/create-self-signed-certificates-kyca8916/mac
18. Apple Code Signing Guide, *Requirement Language*, https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/RequirementLang/RequirementLang.html
19. GitHub, *GitHub-hosted runners reference*, https://docs.github.com/en/actions/reference/runners/github-hosted-runners
20. actions/runner-images, `macos-14-arm64-Readme.md`, https://github.com/actions/runner-images/blob/main/images/macos/macos-14-arm64-Readme.md
21. actions/runner-images, `macos-15-arm64-Readme.md`, https://github.com/actions/runner-images/blob/main/images/macos/macos-15-arm64-Readme.md
22. Apple, *Xcode support — version table*, https://developer.apple.com/support/xcode/
23. Apple, *Building a universal macOS binary*, https://developer.apple.com/documentation/apple-silicon/building-a-universal-macos-binary
24. swift.org, *Swift 5 Released!*, https://www.swift.org/blog/swift-5-released/
25. swiftlang/swift-package-manager `Sources/PackageModel/Platform.swift` (tag `swift-6.1-RELEASE` vs `main`), https://github.com/swiftlang/swift-package-manager/blob/main/Sources/PackageModel/Platform.swift
26. GitHub, *Installing an Apple certificate on macOS runners for Xcode development*, https://docs.github.com/en/actions/use-cases-and-examples/deploying/installing-an-apple-certificate-on-macos-runners-for-xcode-development
27. Elgato, Stream Deck SDK *Manifest* reference, https://docs.elgato.com/streamdeck/sdk/references/manifest
28. Apple Platform Security, *Gatekeeper and runtime protection*, https://support.apple.com/guide/security/gatekeeper-and-runtime-protection-sec5599b66df/web
29. Microsoft Security, *powerdir* write-up (TCC.db `csreq` column), https://www.microsoft.com/en-us/security/blog/2022/01/10/new-macos-vulnerability-powerdir-could-lead-to-unauthorized-user-data-access/
30. Apple, *Disable Library Validation Entitlement*, https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.disable-library-validation
