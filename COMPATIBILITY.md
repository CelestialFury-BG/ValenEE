## A Note on Legacy Ports and This Edition

There are several community builds of the Valen mod in circulation. This section explains how this edition differs technically from legacy ports — not to critique any particular author's work, but because the differences determine whether the mod installs cleanly on a modern Enhanced Edition setup.

**All builds preserve the original Weimer content.** Every interjection, item, ability, hunter encounter, and epilogue is intact. If you only care about the writing, any build will show it to you.

**The technical foundation is where builds diverge.** Legacy ports were written for the original BG2 engine (2000–2013) and then moved to EE with minimal changes. This edition was rebuilt from the ground up for BG2EE and EET specifically.

### What Legacy Ports Typically Do

Most pre-EE mods, and the ports derived from them, share a common set of patterns that worked fine on the original engine but create problems on EE:

| Legacy Pattern | Why It Was Fine Then | Why It Matters on EE |
|---|---|---|
| Mixed-case filenames (`VALENS.BMP` referenced as `valens.bmp`) | Windows is case-insensitive | Aborts install on Linux and macOS |
| `Setup-` TP2 prefix (capital S) | Manual `weidu.exe` invocation | Breaks auto-discovery in Project Infinity and modern packagers |
| No `HANDLE_CHARSETS` directive | One system codepage per user | Russian and Chinese TRAs render as mojibake on modern systems |
| Unconditional `COPY` to `override/` | Mods were installed one at a time | Clobbers shared tables that other mods depend on |
| No `PRETTY_PRINT_2DA` on appends | The engine tolerated ragged columns | Column misalignment breaks subsequent mod parsing |
| Raw `WRITE_ASCII`/`WRITE_LONG` on binary offsets | The offset was stable for the BG2 file format | Corrupts files if the shipped EE format differs |
| `READ_LONG` + integer comparison for CRE version | Version field layout wasn't understood | Returns `0x302E3156` for `"V1.0"`, so every guard silently fails |
| Single monolithic component | Players wanted one install click | No dependency verification, no partial installs, no modularity |
| Static string pointers for ToB epilogues | String resources were stable | Epilogues misalign or display wrong text after EE patches |

### What This Edition Does Differently

Every item above has a corresponding fix in this edition:

- **All-lowercase filenames** — installs cleanly on Windows, Linux, and macOS
- **`setup-valenEE.tp2`** — auto-detected by Project Infinity and WeiduModPackager
- **`HANDLE_CHARSETS`** — every translation reads as UTF-8 natively, regardless of the installing system's codepage
- **`BUT_ONLY_IF_IT_CHANGES`** on every file-modifying `COPY` — shared tables are only written when actually modified, so this mod can be installed alongside SCS, Tweaks, and anything else that patches the same files
- **`PRETTY_PRINT_2DA`** on every 2DA patch — output matches BioWare's column-aligned format
- **`EE_CRE_CLEANUP` / `EE_SPELL_CLEANUP` / `EE_CUTSCENE_CLEANUP`** patch functions — sweep deprecated effect opcodes, clamp saving throws, and harden cutscene timing for the EE engine
- **`EE_SET_CRE_FIELDS`** (in `ee_cre_fields.tpa`) — a reusable joinable-NPC field writer that binds script name, dialogue resref, and the known-spells table at NI-confirmed CRE v1.0 offsets. One `STR_VAR` parameter per field, never a delimited list, so it parses cleanly on WeiDU 25100 and any future version
- **CRE v1.0 format guard** on every binary write — `READ_ASCII 0x04` + `STRING_EQUAL ~V1.0~`. Fixes the legacy pattern of `READ_LONG` + integer comparison, which always failed on the ASCII version field
- **Content-preserving cleanup** — the Noober-era audio purge block, which was pasted in from an unrelated BG1 mod and silently muted every one of Valen's sound slots, has been removed outright
- **Two modular components** with `DESIGNATED` IDs — Component 10 (Core NPC) and Component 20 (Protection tweak) install independently
- **Dynamic `RESOLVE_STR_REF` allocation** for the ToB epilogue — the ending screen always displays the correct text, in any language

### Full Comparison Table

| Capability | Legacy Port | This Edition |
|---|---|---|
| Target platform | Original BG2 engine, ported forward | BG2EE / EET, designed for EE |
| Filename casing | Mixed | All lowercase |
| TP2 prefix | `Setup-` (capital) | `setup-` |
| Translation encoding | System codepage | Native UTF-8 via `HANDLE_CHARSETS` |
| Load-order safety | None | Full `BUT_ONLY_IF_IT_CHANGES` |
| 2DA formatting | Ragged columns | `PRETTY_PRINT_2DA` |
| CRE version check | `READ_LONG` + integer comparison | `READ_ASCII` + `STRING_EQUAL ~V1.0~` |
| Binary file writes | Raw offsets, format-unaware | Format-aware patching via `lib/*.tpa`, with CRE v1.0 guard |
| Joinable-NPC field binding | Raw `WRITE_ASCII` at hand-picked offsets | `EE_SET_CRE_FIELDS` at NI-confirmed v1.0 offsets |
| Corrupt Dialogue resref | Shipped as-is (`VALE\x76`) | Repaired to `VALEN` |
| Misaligned known-spells table | Shipped as-is (`\x14` in entry 0) | Repaired, entries 1–5 written explicitly |
| Component structure | Single monolithic | Two modular components |
| Sound slots | Silently muted by Noober-era purge | Preserved; purge block removed |
| Languages | 4–8 (varies by build) | 10, with layered English fallback |
| EE cutscene hardening | None | `EE_CUTSCENE_CLEANUP` |
| Project Infinity metadata | None or partial | Full `valenEE.ini` + `valenEE.json` |
| Cross-mod install-order docs | None | Documented Valen-before-Solaufein requirement |

### Why This Matters for EE Users

On a lightly modded BG2EE install running on Windows, a legacy port will often work. The problems surface when any of these are true:

- **You're on Linux or macOS** — mixed-case filenames and non-UTF-8 TRAs will abort or garble the install
- **You're running a large mod stack** — unconditional `COPY` to `override/` overwrites tables that SCS, Tweaks, or other mods have already customized
- **You've installed another mod that ships Valen content** — duplicate `pdialog.2da` rows can corrupt the party menu
- **You play in a non-English language** — legacy ports only translate the strings their original translator chose to include
- **You want the corrupted CRE fields repaired** — legacy ports ship `valen.cre` exactly as it was in 2003, with a corrupt Dialogue resref and a misaligned known-spells table. This edition writes the correct values explicitly
- **You use SolaufeinEE alongside this mod** — the cross-mod interjections only compile when Valen is installed first

For all of these scenarios, this edition is the correct choice. For a Windows-only, English-only, minimal-mod install on original BG2, a legacy port will serve you fine — the content is the same either way.

### A Word on the Original Engine

This edition explicitly **refuses to install** on the original BG2 or on a non-EE game. This is deliberate: the binary file formats, script opcodes, and engine behavior for `StartCutSceneMode`, the CRE version field layout, and the sound slot table are all different between classic BG2 and BG2EE. Writing code that satisfies both engines means writing code that is optimal for neither. This edition commits to the EE family so that every patch, every cleanup routine, and every 2DA format assumption can be made without compromise.

If you're on the original engine, a pre-EE build of Valen is the right choice. If you're on BG2EE or EET — which is the overwhelming majority of active installs today — this edition is built specifically for you.
