---
name: rea
description: Reverse engineer a target with REA from a short request such as "/rea <thing>". Default specialty is Autodesk Revit add-ins/plugins (.NET assemblies, .addin manifests, .bundle packages), but it routes any binary, package or app. Use when the user types /rea or asks to "rea", reverse engineer, or understand how a plugin/app/binary works.
argument-hint: "<target or goal, e.g. 'o plugin X do Revit' | path | app name>"
---

# /rea

`/rea <anything>` means: reverse engineer that thing with REA and explain how it works.
Answer in the user's language (the user writes Portuguese).

## 1. Interpret the request

Split `$ARGUMENTS` into **target** (what) and **goal** (why: understand a feature, list
commands, find how it licenses/talks to a server, compare versions, rebuild it).
No goal given → default goal: "what it is, how it is structured, what it does, what it talks to".

Resolve the target in this order, stop at the first hit:
1. An absolute/relative path or archive the user gave.
2. A name → look in the known locations (section 3), then the workspace. One clear match: use it.
   Several matches: list them and ask. Never pick an example app on the user's behalf.
3. Nothing found → ask for the path or installer. Do not guess.

Authorization: REA is for software the user owns, is licensed to inspect, or may analyze for
interoperability. If the request looks like bypassing licensing/protection of someone else's
product, stop and ask about scope first (see `DISCLAIMER.md`). Analysis is static by default;
never run the target, inject, or modify anything unless the user explicitly asks for that.

## 2. Make sure REA is available

- Tools `inspect_managed_artifact`, `open_binary`, ... present (MCP `rea`) → just use them.
- Else prefer the `rea` CLI; else `npx -y rea-agents@latest <command> --json`
  (needs Node >= 22.19). Command = tool name with `_` → `-`.
- Registering the MCP server is `npx rea-agents setup` — it shows a plan and needs the
  user's approval. Never write agent config or install Hopper/Ghidra/IDA/Java/Node unprompted.
- Full workflow reference (read only what the target needs):
  `.agents/skills/reverse-engineer-anything/SKILL.md` and its `references/`;
  managed code: `docs/managed-code-analysis.md`; native: `docs/native-investigation.md`.

## 3. Revit plugin playbook

A Revit add-in is normally **managed .NET** (C#/VB), so static CIL inspection is the right
first route; no Revit installation is needed to read it.

**Anatomy**
- `*.addin` — XML manifest the Revit loads. Read it first (plain text). Key fields:
  `<Assembly>` (dll path), `<FullClassName>`, `<AddInId>` (GUID), `<VendorId>`,
  `<Type>` = `Application` (`IExternalApplication`: ribbon, events, startup) or
  `Command` (`IExternalCommand`).
- `*.dll` — the real code. Entry points implement `IExternalApplication`,
  `IExternalCommand`, `IExternalDBApplication`, `IExternalEventHandler`, `IUpdater`.
- `.bundle` folders — Autodesk App Store style: `PackageContents.xml` maps
  Revit versions (`SeriesMin`/`SeriesMax`) to `Contents/<year>/…` assemblies.
- Common companions: resources (icons/RFA families/templates), Nuget/ILMerge'd libs,
  config JSON/XML, WPF/XAML (BAML in resources), native helper DLLs (P/Invoke), SQLite.

**Where to look** (Windows): `%AppData%\Autodesk\Revit\Addins\<year>\`,
`%ProgramData%\Autodesk\Revit\Addins\<year>\`, `%ProgramData%\Autodesk\ApplicationPlugins\`,
`%AppData%\Autodesk\ApplicationPlugins\`, vendor installers (MSI/EXE/ZIP).
Installers/MSI/ZIP: inventory/extract with REA artifact tools, keep exact member identity.

**Workflow (static, summary-first)**
1. Inventory: the `.addin`, `PackageContents.xml`, every dll/exe/resource; record paths + SHA-256.
2. `inspect_managed_artifact` with `detail:"summary"` on each dll: target framework
   (`net48` for Revit ≤2024, `net8` for 2025+), assembly/MVID, native/mixed-mode markers,
   obfuscation hints (mangled names, unreadable metadata, `ConfusedBy`, packed resources).
3. Find entry points: `inspect_managed_members` (use `type_names` for selected types when
   large) → types implementing the Revit interfaces above; `OnStartup`/`Execute` bodies.
4. Trace behavior from entry points: ribbon commands/buttons → `Execute` →
   Revit API calls (`Transaction`, `FilteredElementCollector`, `Parameter`, `Element.Create*`,
   `ExternalEvent`, `UIApplication.*` events) → data written to the model.
5. Outside the model: network (`HttpClient`, `WebClient`, sockets, SignalR), auth/licensing
   (license checks, machine IDs, tokens), file/registry/config access, telemetry, update
   mechanisms, process launches, dynamic loading (`Assembly.Load`, `Reflection.Emit`).
6. Native boundary: `inspect_managed_native_boundaries` for P/Invoke; analyze the native
   DLL with `open_binary` (Ghidra/Hopper/IDA are bring-your-own) only if the question needs it.
7. Versions: `compare_managed_members` across plugin builds / Revit-year builds.
8. Rebuild: only when asked. Use ordinary coding tools; `import_managed_reconstruction`
   can check decompiled code against verified identities. Decompiled output is inference.

**Revit-specific cautions**
- Revit API identity = type/member names in `RevitAPI.dll`/`RevitAPIUI.dll`; reference
  assemblies come from the Revit install, not the plugin — do not treat them as plugin code.
- Heavily obfuscated plugins (.NET Reactor, ConfuserEx, Dotfuscator, Eazfuscator): say what
  was and was not recoverable; do not claim intent from mangled names.
- .NET Framework (≤2024) vs .NET 8 (2025+): affects which runtime/decompiler routes apply.
- Plugin might be a Dynamo package (`pkg.json`, `.dyf/.dyn`) or pyRevit/IronPython scripts:
  those are text/JSON/Python — read directly, REA is for the compiled parts.
- Static analysis does not show runtime behavior. Taking over the machine, running Revit
  or attaching a debugger/Frida is a separate step needing explicit approval and a Windows host.

## 4. Report

Per `reverse-engineer-anything`: separate **observed / inferred / unknown**, cite Evidence
IDs and artifact SHA-256, state coverage and limits, never imply the code was executed.
Finish with: what the plugin is, entry points/commands, key flows, external dependencies,
open questions, and the next step you propose (not performed). Close any opened session
with `close_binary`.
