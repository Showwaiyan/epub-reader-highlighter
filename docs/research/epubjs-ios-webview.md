# How does epub.js 0.3.x behave in iOS WKWebView?

Answers GitHub issue #3. Facts with sources only; no recommendations. Researched 2026-11-09 against epub.js 0.3.93 (the version in this repo's `package.json`, read from `node_modules/epubjs/src`).

Confidence labels: **[source]** = read from epub.js source in this repo; **[doc]** = official documentation or spec data; **[report]** = a GitHub/forum report by a third party (not an owner statement); **[not found]** = no primary source located.

## 1. Platform context

- Obsidian's developer docs say iOS can be inspected from macOS on iOS 16.4+ via WebKit's Web Inspector, and that Node/Electron APIs are unavailable on mobile. They do not name WKWebView or Capacitor, and do not give memory or resource limits. [doc] https://docs.obsidian.md/Plugins/Getting+started/Mobile+development
- `Platform.isIosApp` / `Platform.isAndroidApp` exist for branching. `this.app.emulateMobile(true)` is a desktop simulation only, not an iOS test. [doc] same page.
- That Obsidian mobile is built on Capacitor appears only as an unverified forum user belief. [report] https://forum.obsidian.md/t/what-technology-obsidian-mobile-is-developed-with/40125
- Capacitor's own docs state it uses WKWebView on iOS. [doc] https://capacitorjs.com/docs/ios
- Obsidian's docs list regex lookbehind as supported only on iOS 16.4+. [doc] same Obsidian page.

## 2. Iframe construction in epub.js 0.3.93

All [source] in `node_modules/epubjs/src/managers/views/iframe.js` unless noted:

- The rendition creates one `<iframe>` per section (`create()`, ~line 76). `sandbox` is set to `allow-same-origin`; `allow-scripts` is appended only when `allowScriptedContent` is true (default false); `allow-popups` only if `allowPopups` (lines 93-100).
- Comment at line 87: `scrolling = "no"; // Might need to be removed: breaks ios width calculations`.
- Content loading method: if `"srcdoc" in iframe`, the default method is `srcdoc`, else `write` (`document.open/write`) (lines 130-138, 399-417). A `blobUrl` method also exists: it builds a Blob with MIME `application/xhtml+xml`, assigns it to `iframe.src`, and revokes it on destroy (lines 399-402, 817-818). The method can be forced via the `method` render option (`managers/default/index.js` line 41: "srcdoc, blobUrl, write").
- `show()` toggles `transform: translateZ(0)` and reads `offsetWidth`, commented "Remind Safari to redraw the iframe" (~line 543).
- `contents.js` has WebKit-specific branches: collapsed-range bounds (links WebKit bug 138949), incorrect rects in columns, `-webkit-line-box-contain` for glyph clipping, and a Safari column cut-off note (lines ~633-670, 1072, 1100-1102).
- `managers/default/index.js` ~lines 179-215: orientation handling cites iOS 10.3 mis-measuring on resize (ampproject/amphtml#8479), uses a 500 ms timeout, and skips resizes while the window is square.

## 3. `srcdoc` and blob iframes on iOS

- `srcdoc` iframe support: Safari 6+ per MDN browser-compat-data (iOS Safari data is mirrored from Safari). [doc] https://github.com/mdn/browser-compat-data/blob/main/html/elements/iframe.json
- `srcdoc` documents have location `about:srcdoc` and inherit the parent's origin unless sandboxed; relative URLs resolve against the embedder. [doc] https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe
- BCD note for `sandbox` `allow-same-origin` in Safari: "Safari blocks script execution without `allow-scripts` even if `allow-same-origin` is set", e.g. event handlers bound to nodes inside the iframe throw blocked-script errors. [doc] same BCD file. (epub.js attaches its listeners from the parent via `contentDocument`; see section 4.)
- epub.js README: scripted content is disabled by default by sandboxing the iframe; `allowScriptedContent: true` enables it. [doc] https://github.com/futurepress/epub.js/blob/master/README.md
- A react-native-webview user reported (iOS, RN 0.62.2, webview 11.16.0) that, inside an epub.js iframe, injected-JS selection messaging did not work and Safari's console said "scripts is blocked on iframe"; the maintainer attributed it to epub.js sandboxing and pointed at the scripted-content docs; closed without a code change. [report] https://github.com/react-native-webview/react-native-webview/issues/2350
- Blob-URL iframes on iOS: **[not found]** no primary source specific to WKWebView located. epub.js source shows the `blobUrl` method exists but is not the default.
- Related community report: removing `method: 'default'` from `renderTo` options resolved `touchstart` not firing on the rendition (see section 5). [report] https://github.com/futurepress/epub.js/issues/925

## 4. Events from iframe to host

All [source]:

- `Contents` registers `DOM_EVENTS` listeners (`{passive: true}`) on the iframe document: `keydown, keyup, keypressed, mouseup, mousedown, mousemove, click, touchend, touchstart, touchmove` (`utils/constants.js` line 4; `contents.js` ~lines 885-912).
- `Rendition.passEvents` re-emits each of those on the rendition with `(event, contents)`, plus the contents' `selected` as rendition `selected(cfiRange, contents)` (`rendition.js` ~lines 870-907).
- Selection detection: a `selectionchange` listener on the iframe document (passive), debounced 250 ms; after the timeout it reads `window.getSelection()` and, if a non-collapsed range exists, emits `selected` (CFI string) and `selectedRange` (Range) (`contents.js` ~lines 930-975). It does not listen to touchend/mouseup for selection.
- No `pointer*` events are in the forwarded list.

## 5. `selected` and `markClicked` on touch

- `markClicked`: highlight/underline/mark elements get both `click` and `touchstart` listeners that emit `MARK_CLICKED` (`iframe.js` ~lines 630-666, 714-718) [source]. Highlights are drawn by the `Pane` SVG overlay (`new Pane(iframe, element)`) in the host document above the iframe; `mark()` creates an `<a>` inside the iframe document. [source]
- Because both `touchstart` and `click` are bound, a single tap on a mark can invoke the emitter and any supplied callback twice on a touch device (a touch tap normally yields both events). Derived from the code; not run on iOS. [source, inference]
- Issue #494 (`selected`/`selectedRange` not delivered on rendition): maintainer said the rendition was listening to the view rather than contents, fixed in v0.3.6 (Dec 2016). Later commenters (2017, 2021, 2022) still reported non-delivery; one found it fired once `width` was passed to `renderTo`. Not iOS-specific. [report] https://github.com/futurepress/epub.js/issues/494
- Issue #925 (`touchstart` on rendition not firing, v0.3.83): maintainer asked whether Chrome device emulation was used; community workaround of deleting `method: 'default'` from `renderTo` options, confirmed by one later commenter; issue still open; a Jan 2026 commit in an unrelated project references it for "tap navigation not working on iOS Safari". [report] https://github.com/futurepress/epub.js/issues/925
- Issue #904 (iPad, Jan 2019): dragging selection handles collapses the selection to one character in many books. Reporter traced it to WebKit (iframes offset from top of page); WebKit bug 193663 closed as duplicate of 188374; reporter confirmed fixed in iOS 12.2. Reporter's interim workaround was a custom-handle patch to 0.3.66. [report] https://github.com/futurepress/epub.js/issues/904 ; https://bugs.webkit.org/show_bug.cgi?id=193663
- epubjs-react-native PR #32 (merged): on newer iOS versions the page is not sent `touchend` when a touch that began on a selection handle ends, so a reader that waited for touchend to show its menu never showed it; the fix was to send the selection after a short timeout. Wait length and iOS versions not stated. [report] https://github.com/cambriz-createdbyhumans/epubjs-react-native/pull/32
- No official epub.js document describing `selected` behavior on iOS was found. **[not found]**

## 6. Memory use for large books

- The default manager shows one section at a time; the continuous manager "will display as many sections as need to fill the screen, and preload the next section offscreen" and "is less performant than the default method". [doc] https://github.com/futurepress/epub.js/blob/master/README.md
- Continuous manager defaults: `offset: 500`, `offsetDelta: 250`; views that are not visible are scheduled for `trim` via a timeout/queue (`managers/continuous/index.js` lines 19-20, 187-230). Destroying a view removes its iframe and revokes any blob URL (`iframe.js` ~805-830). [source]
- An epub.js GitHub search for memory/large-book issues returned only #151 (desktop Chrome crash with a large file, 2015, closed). [report]
- WKWebView memory: Apple forum answers say the WebContent process limit cannot be raised and depends on device RAM and load, with memory-pressure warnings near the limit; the termination delegate that distinguishes memory kills is private. Forum/bug-tracker threads, not owner documentation. [report] https://developer.apple.com/forums/thread/133449 ; https://developer.apple.com/forums/thread/108434 ; https://bugs.webkit.org/show_bug.cgi?id=162334
- No documented per-book or per-iframe size figure for epub.js in WKWebView was found. **[not found]**

## 7. This repo's configuration (for reference)

From `main.ts`: the reading view uses `renderTo` with `flow: paginated|scrolled`, `manager: default|continuous`, `spread: "none"`, no `method` override and no `allowScriptedContent`; it listens to `rendition.on("selected", ...)`. An offscreen scan rendition (`manager: "default"`) is also created for page counting. [source]

## Gaps

- No source found on blob-URL iframes under WKWebView specifically.
- No primary source on whether Obsidian iOS adds its own handling to iframes or text selection.
- Nothing here was tested on a device or simulator.
