# Obsidian on iOS: requirements and restrictions for this plugin

Research for GitHub issue #2. Facts only, each tied to a primary source. Gaps are listed at the end.

Sources:
- **[Dev docs]** `obsidianmd/obsidian-developer-docs`, `main` branch, `en/` (published at docs.obsidian.md).
- **[Help]** `obsidianmd/obsidian-help`, `en/` (published at help.obsidian.md).
- **[Typings]** `node_modules/obsidian/obsidian.d.ts`, package `obsidian` version 1.13.1 (the version installed in this repo).
- **[Repo]** this repository at commit `bea98d0` (main, v0.5.3).

## 1. Manifest: `isDesktopOnly` and `minAppVersion`

- `isDesktopOnly` is "Whether the plugin can only be used on the desktop app, for example because it uses NodeJS or Electron APIs." [Dev docs: Reference/Manifest.md]. In the typings it is optional on `PluginManifest` [Typings, `isDesktopOnly?: boolean`]. In the docs page it is listed as required.
- Setting it to `true` stops mobile users from installing the plugin [Dev docs: Plugins/Getting started/Mobile development.md, "Disable your plugin on mobile devices"].
- Plugin-store rule: Node.js and Electron APIs "are only available in the desktop version of Obsidian"; a plugin that uses any of them "must" set `isDesktopOnly` to `true`. Web API alternatives are suggested (`SubtleCrypto` for `crypto`, `navigator.clipboard` for the clipboard) [Dev docs: Community directory/Submission requirements for plugins.md].
- `minAppVersion` is "The minimum required Obsidian version." [Dev docs: Reference/Manifest.md]. Store rule: set it to the minimum version the plugin is compatible with; if unknown, use the latest stable build [Dev docs: Submission requirements for plugins.md].
- The docs and typings state no iOS-specific `minAppVersion`. `CapacitorAdapter` (the mobile vault adapter) is marked `@since 1.7.2` [Typings].
- This repo's manifest: `"isDesktopOnly": false`, `"minAppVersion": "1.8.7"` [Repo: manifest.json]. So the manifest already allows mobile installs.

## 2. Platform detection

`Platform` flags (typings; also Dev docs: Reference/TypeScript API/Platform.md, since 0.12.2):

| Flag | Meaning (typings comment) |
|---|---|
| `isMobile` | The UI is in mobile mode |
| `isDesktop` | The UI is in desktop mode |
| `isMobileApp` | "running the capacitor-js mobile app" |
| `isIosApp` / `isAndroidApp` | Running the iOS / Android app |
| `isPhone` | Mobile app with very limited screen space |
| `isTablet` | Mobile app with sufficiently large screen space |
| `isMacOS` | "a macOS device, or a device that pretends to be one (like iPhones and iPads)" |
| `isSafari` | Running in Safari; "Typically used to provide workarounds for Safari bugs" |
| `resourcePathPrefix` | `file:///` on mobile; `app://random-id/` on desktop |

- The mobile-dev guide shows `Platform.isIosApp` / `Platform.isAndroidApp` as the way to branch on platform [Dev docs: Mobile development.md].
- `Command.mobileOnly?: boolean` exists [Typings line ~1718; Dev docs: Reference/TypeScript API/Command/mobileOnly.md].
- Repo usage: `Platform.isMacOS` is used for hotkey display and matching (`main.ts` ~lines 124-145), and `Platform.isMobile` only in the throwaway Q9 timing command (~line 634) [Repo]. Because `isMacOS` is true on iPhone/iPad per the typings, the existing hotkey code takes its macOS branch there.

## 3. Node / Electron / runtime restrictions

- "The Node.js API, and the Electron API aren't available on mobile devices. Any calls to these libraries made by your plugin or it's dependencies can cause your plugin to crash." [Dev docs: Mobile development.md, "Node and Electron APIs"]. The same text is repeated as a review guideline [Dev docs: Plugins/Releasing/Plugin guidelines.md, "Mobile"].
- Regex lookbehind is "only supported on iOS 16.4 and above", and some iPhone and iPad users may be on earlier versions. The docs suggest platform checks or a library to detect browser versions [Dev docs: Mobile development.md]. Plugin guidelines repeat it.
- Inspecting the webview on a real iOS device needs iOS 16.4 or later and a macOS computer (Web Inspector) [Dev docs: Mobile development.md]. Desktop emulation: `this.app.emulateMobile(true)` in the dev console; toggle with `this.app.isMobile` [same page]. The docs do not say how complete the emulation is.
- Repo state (static check): `main.ts` imports only from `obsidian` and `epubjs`; there are no `fs`/`path`/`os`/`crypto`/`electron` imports. The built `main.js` contains one `require`, for `"obsidian"`. No `(?<=` / `(?<!` lookbehind appears in `main.ts` or `main.js`. esbuild target and tsconfig target are ES2018 [Repo]. Whether epubjs and its dependencies (e.g. `@xmldom/xmldom`) behave on iOS was not checked here; this is a source-read only, nothing was run on a device.

## 4. Vault and adapter on iOS

- On mobile the adapter is `CapacitorAdapter`; on desktop it is `FileSystemAdapter` (typings: `CapacitorAdapter` "Implementation of the vault adapter for mobile devices", since 1.7.2). Both implement `DataAdapter`, which includes `exists`, `stat`, `list`, `read`, `readBinary`, `write`, `mkdir`, `remove`, `getResourcePath` [Typings]. `FileSystemAdapter` additionally exposes desktop-only members (e.g. `getBasePath`); the typings mark at least one such member as "Not available on mobile" (~line 4941 is a `Platform`-adjacent comment; verify per member before use).
- Plugin guidelines: prefer the Vault API (`app.vault`) over the Adapter API (`app.vault.adapter`) for caching and serial file operations; use `normalizePath()` for user-defined paths [Dev docs: Plugin guidelines.md, "Vault"].
- Repo usage: the book is read with `app.vault.readBinary(file)` (`main.ts` ~line 4178). Annotation sidecars and exports use `app.vault.adapter` (`read`, `write`, `exists`, `list`, `mkdir`, `remove`; ~lines 765-1052) [Repo]. The `DataAdapter` methods listed exist on `CapacitorAdapter` per the typings.
- iCloud, as documented in the help site [Help: Getting started/Sync your notes across devices.md]:
  - iCloud Drive is listed as a supported way to sync vaults between iOS and macOS. Vaults must live in `iCloud Drive/Obsidian/[Vault Name]`.
  - Keeping the vault folder downloaded is "optional, but strongly recommended to keep load times fast" (macOS: Keep Downloaded; or disable Optimize Mac Storage).
  - FAQ: "If services like OneDrive or iCloud offload files ... Obsidian can't access them, causing sync issues." The fix given is to keep files downloaded.
  - Do not sync one vault with several services at once (data conflicts or corruption).
- NOT FOUND in primary sources: any documented behavior of `vault.readBinary`, `adapter.read`, `exists`, or `list` when an iCloud file is a not-yet-downloaded placeholder (error type, hang, or empty result). The help text above says only that Obsidian "can't access" offloaded files. The typings carry no doc comments on this. Verifying needs a test on a device or the closed-source app.

## 5. Views, leaves, modals on a phone

- Mobile workspace: `workspace.leftSplit` / `rightSplit` are `WorkspaceSidedock | WorkspaceMobileDrawer`. A leaf's `parent` can be a `WorkspaceMobileDrawer` on mobile [Typings ~lines 7752-7757, 8234-8240].
- `Workspace.openPopoutLeaf` throws "if the app does not support popout windows (i.e. on mobile ...)" [Typings ~line 7898].
- Views are created as `DeferredView` until visible (Obsidian 1.7.2+); check `leaf.view instanceof MyView` before use [Dev docs: Plugins/Guides/Defer views.md]. Guideline: do not keep references to custom views; use `getLeavesOfType()`; do not detach leaves in `onunload` [Dev docs: Plugin guidelines.md, "Workspace", "Resource management"].
- `FileView` has `navigation: boolean` ("File views can be navigated by default"); `View` has a `navigation` flag "Whether or not the view is intended for navigation" [Typings ~lines 3144, 7596]. The typings do not document what it changes on mobile.
- Repo: `EpubView extends FileView`, registered with `registerExtensions(["epub"], VIEW_TYPE_EPUB)` (`main.ts` ~lines 601, 3664). Modals are `Modal` subclasses (NoteModal, SearchModal, HighlightListModal, etc.) [Repo].
- NOT FOUND in primary sources: documented API or docs for hiding the view header on mobile, full-screen leaf on phone, the mobile navbar, or safe-area insets (`env(safe-area-inset-*)`). The Dev docs pages searched (Views.md, Workspace.md, Modals.md, Mobile development.md, the CSS variables reference) contain no such guidance. `grep` of this repo's `styles.css` finds no `safe-area`, `env(`, or `is-mobile`/`is-phone` rules. The `Platform.isPhone` / `isTablet` flags and `isMobile` UI mode are the only documented hooks.
- Modals: the Modal docs/typings describe `modalEl`, `containerEl`, `contentEl`, `titleEl`; no mobile-specific behavior is documented in the pages read (Dev docs: Plugins/User interface/Modals.md and Reference/TypeScript API/Modal.md; the Modals page was only skimmed for the word "mobile").

## 6. Plugin-store rules touching mobile (summary)

From [Dev docs: Submission requirements for plugins.md] and [Plugin guidelines.md]:
1. Node/Electron use requires `isDesktopOnly: true`.
2. `minAppVersion` must be the real minimum.
3. Review guidelines flag Node/Electron calls and regex lookbehind under "Mobile".
4. No default hotkeys for commands is a guideline ("Avoid setting a default hotkey for commands"); this repo has a configurable highlight hotkey, which is a separate matter from command hotkeys.
5. No rule was found that mandates iOS testing or a mobile-specific listing label.

## Not covered / gaps

- iCloud placeholder-file behavior (see section 4).
- Header hiding, full-screen leaf, safe areas (section 5).
- Behavior of epubjs, iframes (`srcdoc`/blob), `navigator.clipboard`, or `Blob` downloads inside the iOS WKWebView. These were not found in Obsidian primary sources; they are WebKit facts outside the Obsidian docs.
- Nothing was executed on a device or in mobile emulation.
