# AGENTS.md — LPLC

This file is the single entry point for **every** AI coding agent working in this repository
(Claude Code, Codex, Cursor, Copilot, Gemini CLI, …). There is deliberately no tool-specific
instruction file — no `CLAUDE.md`, no `.cursorrules` — so there is one set of instructions to keep
current.

**LPLC** is a TwinCAT 3 PLC library (Structured Text in TwinCAT's XML file formats). It is the
PLC-side companion to [lhmi](https://github.com/afarzin1/LHMI), whose faceplates bind to the `.HMI`
struct every LPLC device exposes.

## Read before editing

1. **[`CONVENTIONS.md`](CONVENTIONS.md), completely.** It is the canonical rule set: type names,
   member prefixes (role, never type — no Hungarian notation), the `.HMI` pattern, the
   `abs_fb_Device` base, self-contained FBs (§5), folders, file format (§8), and registering files
   in the `.plcproj` (§9). Read it in bounded chunks until EOF.
2. The design note in [`Docs/`](Docs/) for the block you are touching, if it has one.
3. [`ROADMAP.md`](ROADMAP.md) for what is planned and what is known to be broken. When you finish
   or plan work, update it in the same change.

## Working rules

- **You cannot compile here.** Building needs TwinCAT XAE on Windows. Keep edits mechanical and
  consistent, check XML well-formedness, and tell the user what needs a TwinCAT build to confirm.
- **Every new file is registered** in `PLCLibrary/PLCLibrary.plcproj` (Compile item, plus a Folder
  item for a new folder), with casing that matches the disk. An unregistered file is silently
  ignored by TwinCAT.
- **Fresh GUIDs** for every new POU/DUT/Method/Property/Folder `Id`.
- **New files:** UTF-8 without BOM, LF, 4-space indentation. Leave the style of existing files
  alone.
- **`.HMI` structs are a contract with lhmi.** Adding a member is compatible; renaming, retyping or
  removing one breaks every HMI bound to it. Call such changes out in `CHANGELOG.md` and bump the
  version accordingly.
- **Version:** one SemVer number, `<ProjectVersion>` in `PLCLibrary/PLCLibrary.plcproj`. TwinCAT
  regenerates `Version/Global_Version.TcGVL` from it on build.
- **Safety:** this library is not safety-rated. Never present an interlock in it as a safety
  function.
- **No site data:** never commit real hostnames, IP addresses, credentials, customer or machine
  names. Use neutral placeholders.
