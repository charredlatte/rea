# Reverse-engineering the Canva desktop app with REA

Canva desktop is an **Electron** app. Its product logic ships as JavaScript inside
`app.asar`; the native executable is mostly stock Electron (Chromium/V8). So the
in-scope, high-signal path is REA's **static JavaScript/Electron** analysis. It reads
the app's files **without running Canva and without any network access**.

Verified in this session: REA 6.9.0 built and ran here; the commands below were
exercised end-to-end on a fixture Electron app (IPC + context-bridge + native add-on),
and they recovered the bridge and traced a channel renderer -> main -> native add-on
with per-line evidence.

## 1. Locate Canva's app files (your authorized local install)

- Windows: `%LocalAppData%\Programs\Canva\resources\app.asar`
  (or `...\Canva\resources\` holding `app.asar` + unpacked `.node` add-ons)
- macOS: `/Applications/Canva.app/Contents/Resources/app.asar`
- Linux: `/opt/Canva/resources/app.asar` (deb) or the AppImage's mounted `resources/`

`resources/app.asar.unpacked/` (if present) holds native `.node` add-ons extracted
from the archive; keep it beside the asar.

## 2. Confirm what it is (no extraction needed)

    rea inspect-desktop-application "<...>/Canva/resources" --json
    rea inspect-electron-fuses       "<...>/Canva/<main binary>" --json

First detects the Electron convention (`resources/app.asar`) and lists next probes.
Second reads the packaged Electron fuse bytes (ASAR integrity, nodeIntegration,
runAsNode, etc.) without launching the app.

## 3. Statically reconstruct the app (the core step)

REA reads the ASAR directly; no manual unpack required:

    rea analyze-javascript-application "<...>/Canva/resources/app.asar" \
      --detail summary --evidence_output canva.evidence.json --json

Returns, with evidence: module/import graph, source maps, routes, Electron boundaries
(BrowserWindow webPreferences, preload entrypoints, contextBridge APIs + exposed
members), the **IPC surface** (channels, `ipcRenderer` transmissions, `ipcMain`
handlers, pairing), native add-on bindings, protocol registrations, V8 cached-data.
Coverage, limitations and unknowns are preserved (missing != absent).

Project pages from the saved bundle without re-analyzing:

    rea inspect-analysis-view ./view-request.json --json   # modules / references / facts
    rea recover-javascript-sources "<...>/app.asar" ...     # readable modules w/ provenance

## 4. Trace a feature across layers (the "how does X work" step)

Build a trace input from the saved evidence and seed it (channel / api / route / string
/ module / native-export / node-id). Example for an IPC channel:

    rea analyze-javascript-application "<...>/app.asar" --json > app-ev.json
    node --input-type=module -e 'import {readFileSync,writeFileSync} from "node:fs";
      const application=JSON.parse(readFileSync("app-ev.json","utf8"));
      writeFileSync("trace-input.json",JSON.stringify(
        {application,seed:{kind:"channel",value:"<the-channel>"},direction:"both"}));'
    rea trace-application-feature ./trace-input.json --json

Returns a graph linking renderer invoke -> IPC channel -> main handler -> downstream
(e.g. native add-on), each node carrying a source-range or artifact-path location.
This is the Notion "Electron clipboard bridge" showcase, applied to Canva.

## 5. (Optional) native .node add-ons

Any Canva-specific `.node` add-on found in step 3 is native code. To go deeper than the
binding, point a native provider at it: Ghidra / IDA / Hopper / Kuna, all BRING-YOUR-OWN
(REA does not install them). Most Canva logic is JS, so this is only for specific
native pieces.

## Running it for real from this session

Two options, both keeping to your authorized local install:

- **Drop it here:** copy `app.asar` (and `app.asar.unpacked/` if present) and drop it
  on this cat via the Catio. I'll run steps 2-4 here and report findings with evidence.
- **Attach your machine:** bring the machine with Canva online and share the install
  folder; I'll run REA against it in place.

Scope: static only, no execution of Canva, no network, your own authorized install.
I won't fetch Canva from anywhere or probe Canva's servers.
