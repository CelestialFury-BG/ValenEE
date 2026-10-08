# Valen — Modern EE/EET Edition

> A fully modernized rebuild of Weimer's classic Valen mod, restored for Baldur's Gate II: Enhanced Edition and EET.

---

## What Is This?

The **Valen** mod adds Valen — the Chaotic Evil vampire fighter/thief from Bodhi's hold — as a fully joinable party companion in Baldur's Gate II. She has extensive interjection dialogue across Shadows of Amn and Throne of Bhaal, a vampiric power progression that scales with experience, a challenging Vampire Hunter encounter, and a second component that grants level-drain and undead protection to qualifying creatures and NPCs across the game.

Originally released by Weimer in the early 2000s, this edition is a ground-up technical modernization. The **content is preserved exactly** — every interjection, every item, every ability, every hunter bark — but the underlying WeiDU code has been rebuilt to work reliably on modern Enhanced Edition installs.

Key features:

- **Valen joins your party** if you side with Bodhi in Chapter 2
- **Vampiric power progression** — her claws, regeneration, charm, fear, and mist form all scale with XP
- **Vampire Hunter encounter** at 2.5M XP, with four unique hunters and a dedicated combat AI
- **Extensive interjections** across the SoA and ToB storylines, with a full FAQ list in the original readme
- **Optional Component 20** — level drain immunity for golems, elementals, demons, slimes, mists and specific animations; protection from undead for qualifying priests and paladins
- **Ten supported languages** with automatic English fallback

---

## What's New in This Edition

For a detailed technical comparison of how this edition differs from legacy builds of the same mod — including install safety, cross-platform behavior, and load-order compatibility — see [`COMPATIBILITY.md`](COMPATIBILITY.md).

The **2.x modernization** fixes a long list of legacy issues that plagued the original mod on EE installs. Highlights as of **v2.0.4**:

- **Explicit CRE field writes via `EE_SET_CRE_FIELDS`** — Valen's corrupted Dialogue resref (`VALE\x76` instead of `VALEN`) and her one-byte-misaligned known-spells table are both repaired by writing the values we already know from the mod tree
- **CRE v1.0 format guard** on every binary write — `READ_ASCII 0x04` + `STRING_EQUAL ~V1.0~`. The previous `READ_LONG` + integer comparison always failed because the version field is an ASCII string, not an integer, so every cleanup was silently skipped
- **Noober-era audio purge removed** — an unrelated BG1 cleanup block had been silently muting every one of Valen's sound slots. Removed outright, not narrowed
- **Load-order-safe 2DA patches** — `BUT_ONLY_IF_IT_CHANGES` on every shared table so we don't clobber other mods
- **SOURCE_SIZE guard corrected** — bumped from `> 0x274` to `>= 0x2A0` so the later `READ_ASCII 0x280 deathvar (32)` read can't out-of-bounds on a truncated v1.0 CRE
- **Layered language fallback** — new strings appear in every language automatically, without translator updates
- **10 supported languages** — English, French, Spanish, German, Polish, Italian, Russian, Chinese (Simplified), Chinese (Traditional), Japanese

See [`readme-ee_updates.md`](valenEE/readme-ee_updates.md) for the full changelog.

---

## Requirements

- **Baldur's Gate II: Enhanced Edition** (v2.0 or higher), **or**
- **EET** (Enhanced Edition Trilogy)

Baldur's Gate: Enhanced Edition is **not** supported — Valen is a Shadows of Amn companion, and her content begins in the Chapter 2 graveyard.

---

## Installation

### Windows

1. Download the latest release from the [Releases page](../../releases/latest).
2. Extract the archive into your **BG2EE/EET game folder** (the one containing `Baldur.exe`).
3. Run **`setup-valenEE.exe`** and follow the installer prompts.

If `setup-valenEE.exe` is not included in the archive, copy `weidu.exe` from your game folder and rename the copy to `setup-valenEE.exe`, then run it.

### macOS / Linux

1. Extract the mod folder into your BG2EE/EET game directory.
2. Open a terminal in that directory and run:

       weidu --install setup-valenEE.tp2

### Project Infinity

The mod ships with `valenEE.ini` and `valenEE.json` metadata. Point PI at the extracted mod folder and it will detect the components automatically.

---

## Install Order Note (Solaufein Cross-Mod Content)

If you also use **SolaufeinEE**, install **Valen before Solaufein**. Solaufein's Component 20 checks for `valenj.dlg` at install time; if Valen isn't installed yet, the Solaufein-meets-Valen cross-mod interjections (about 156 extra strings of dialogue) are silently skipped.

- **Valen → Solaufein:** cross-mod interjections compile. This is the recommended order.
- **Solaufein → Valen:** installs cleanly, but the cross-mod interjections are absent.

Both installs are valid; the difference is only whether the extra dialogue is present.

---

## Components

The installer offers two modular components. Component 10 is required for Valen herself; Component 20 is an independent tweak.

| # | Component | What It Does | Requires |
|---|---|---|---|
| **10** | **Valen: Core NPC** | Valen joins your party with her items, spells, portraits, AI script, vampiric ability progression, and the Vampire Hunter encounter. Includes explicit CRE field writes, CRE v1.0 format guard, deprecated-opcode cleanup, and cross-platform lowercase normalization. | — |
| **20** | **Give More Creatures Protection From Level Drain & Undead** | Independent tweak: level drain immunity for inherently-undead-immune creatures, and protection from undead for qualifying priests and paladins. Scans every v1.0 CRE in `override/`, not a fixed list. | — |

**Recommended install:** Component 10. Component 20 is optional but pairs well with any evil or undead-heavy playthrough.

---

## Languages

The installer will present component names, item descriptions, and dialogue in your selected language. Every translation is a **layered overlay** — any string missing from a language file automatically falls back to English, so no install ever fails on a missing ref.

Supported languages: **American English · Français · Español · Deutsch · Polski · Italiano · Русский · Chinese (Simplified) · Chinese (Traditional) · Japanese**

---

## How It Works

Under the hood, this edition uses proper WeiDU `DEFINE_PATCH_FUNCTION` routines rather than legacy macros:

- **`ee_cre_cleanup.tpa`** — sweeps deprecated effect opcodes, clamps out-of-range saving throws, normalizes script resrefs and death variables to lowercase for cross-platform safety, and clamps negative XP to 0. Does not erase authored content.
- **`ee_cre_fields.tpa`** — the reusable `EE_SET_CRE_FIELDS` writer. One `STR_VAR` parameter per field, never a delimited list. Used to bind Valen's script resrefs, script name, dialogue resref, and known-spells table at NI-confirmed CRE v1.0 offsets. Any future joinable-NPC conversion can adopt it.
- **`ee_spell_cleanup.tpa`** — cleans up corrupt spell school values and out-of-bounds projectile fields inside extended spell headers.
- **`ee_cutscene_cleanup.tpa`** — hardens `StartCutSceneMode()` transitions against timing-based engine freezes.

All 2DA modifications use `PRETTY_PRINT_2DA` for column-aligned output and `BUT_ONLY_IF_IT_CHANGES` for load-order safety. No file is written to `override/` unless it was actually modified.

---

## Credits

- **Original Mod Author:** Weimer
- **Modern EE/EET Edition:** /u/celestialfury
- **French translation:** Ly Meng, Laurent Duvernet, Cocobard
- **Spanish translation:** Clan REO
- **German translation:** Sebastian de Waal, Thalantyr, Tanis, Falk
- **Polish translation:** Grzesiek Miazga
- **Italian translation:** Al17 and Kelvan
- **Russian translation:** AERIE.ru
- **Chinese translation:** yun395, kalabaka
- **Japanese translation:** ironthrone
- **Tools:** WeiDU · Near Infinity · Project Infinity

---

## Links

- [Full changelog](valenEE/readme-ee_updates.md)
- [Original Valen readme](valenEE/readme-valen.txt)
- [Report a bug or request a feature](../../issues)
