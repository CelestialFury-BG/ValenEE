## Valen — Modern EE/EET Edition

A complete modernization of Weimer's classic Valen mod, rebuilt for Enhanced Editions and EET.

------------------------------

## Overview

This project is a ground-up modernization of the original Valen mod by Weimer. It preserves the original content — the playable vampire NPC, her unique items and abilities, and her interjections into the Shadows of Amn and Throne of Bhaal storylines — while rebuilding the technical foundation for:

* Baldur's Gate II: Enhanced Edition (BG2EE)
* Enhanced Editions Trilogy (EET)
* Project Infinity
* Modern WeiDU standards (25100)

This edition introduces:

* A clean, modular component structure with DESIGNATED indexing
* Inline EE/EET-safe CRE, SPL, and cutscene modernization via patch functions
* Automated runtime UTF-8 conversion via HANDLE_CHARSETS
* Layered language TRAs (English base + overlay) with automatic fallback across all ten supported languages
* Component names and REQUIRE_PREDICATE messages localized through TRA refs
* Dynamic Throne of Bhaal epilogue String Reference resolution
* Strict component ordering and numbering for Project Infinity metadata alignment
* A fully sanitized, cross-platform lowercase file naming system
* Explicit field writes on the joinable Valen CRE (`valen.cre`) — script resrefs, script name, dialogue resref, and known-spells table are written from the values already present in the mod tree, replacing the corrupted fields the original file shipped with
* An explicit gender write so `valen.cre` is guaranteed Female regardless of the pristine file's contents
* A CRE v1.0 format guard on every binary write, so a future Beamdog format change skips rather than corrupts
* Load-order-safe `BUT_ONLY_IF_IT_CHANGES` guards on every 2DA and store patch
* A reusable CRE field-writing helper (`ee_cre_fields.tpa`) that any future joinable-NPC conversion can adopt

------------------------------

## Folder Structure (Verified Layout)

setup-valenEE.tp2
valenEE/
│
├── readme-valen.txt
├── readme-ee_updates.md
├── valenEE.json
├── valenEE.ini
│
├── lib/
├── tra/
├── dialogues/
├── scripts/
├── creatures/
├── items/
├── spells/
├── graphics/
└── backup/

This structure is:

* PI-friendly
* Linux/macOS-friendly (strict lowercase paths avoid case-sensitivity install crashes)
* EET-friendly
* Free of legacy override-clobbering `.ids` tables

------------------------------

## Components

The mod is divided into two modular components with explicit DESIGNATED indexing matching the metadata layer. Every component name is delivered through the TRA system, so the installer menu displays each name in the user's selected language.

### Component 10: Valen — Core NPC

Installs Valen as a joinable NPC with her items, spells, dialogues, portraits, AI script, and vampire hunter encounter. Applies the `EE_CRE_CLEANUP` and `EE_SET_CRE_FIELDS` modernization functions to `valen.cre` at install time.

Specific work performed at install time:

* **Field writes on `valen.cre`.** The original file ships with a corrupt Dialogue resref (`VALE\x76` instead of `VALEN`) and a misaligned known-spells table (`\x14` in entry 0, shifting every subsequent entry by one byte). Both fields are written explicitly from the values already present in the mod tree. Values written:
  * Override script = `valen`, Class script = `valen`, Script name = `valen`, Dialogue resref = `valen`
  * Known spell 1 = `valen` (Blighted by the Sun), 2 = `SPCL412` (Set Snare), 3 = `SPIN104` (Larloch's Minor Drain), 4 = `SPIN105` (Horror), 5 = `SPIN101` (Cure Light Wounds)
* **Display name and gender.** The joinable Valen CRE's Name and Tooltip fields are set to `setup.tra @17` (*"Valen"*), matching every line of her own dialogue. An explicit `WRITE_BYTE 0x0238 2` guarantees the CRE's gender byte is Female regardless of what the pristine file contains. The Chapter 6 encounter CRE (`c6valen.cre`) is deliberately left on `setup.tra @16` (*"Cynara"*) — that name is intentional for that specific scene.
* **CRE v1.0 format guard.** Every binary write on `valen.cre` is wrapped in `READ_ASCII 0x04 ver (4)` + `PATCH_IF (~%ver%~ STRING_EQUAL ~V1.0~)`. `valen.cre` is CRE v1.0 on BG2EE today; the guard protects against future Beamdog format changes and prevents the writes from landing on the wrong fields if the file ever ships as v2.2. See the version-field note below.
* **Deprecated opcode removal.** Four deprecated effect opcodes (142, 215, 248, 267) are stripped from the v1.0 CRE. These are legacy BG2 effects that do nothing on the EE engine and can confuse NI's effect viewer.
* **Cross-platform safety.** Script resrefs and death variables are lowercased on the joinable CRE so they resolve correctly on Linux and macOS case-sensitive filesystems.
* **XP and save clamps.** Negative XP values and out-of-range saving throws (outside EE's 0–20 bracket) are clamped to safe values.
* **Vampire hunter scripts.** `valensla.bcs` and `valenuh4.bcs` are copied and their placeholder strrefs (`99991`–`99995`) resolved to the correct localized strrefs from `setup.tra`. The hunters announce themselves with the correct text on every language install.
* **Portrait installation.** Large and small portraits are copied both to `override/` (for the CRE to reference) and to `portraits/` (so players can select them for their own PC).
* **Vanilla CRE renames.** `anast.cre` gets Valen's small portrait (guarded by the same v1.0 version check); `c6valen.cre` gets the Chapter 6 encounter display name from `setup.tra @16` (*"Cynara"*).
* **Cutscene hardening.** The main AI script (`valen.bcs`) runs through `EE_CUTSCENE_CLEANUP` to normalize `StartCutSceneMode()` / `CutSceneId()` calls for the EE engine.
* **Area script extensions.** `ar0902.bcs`, `ar0903.bcs`, `sht0902.bcs`, `sht0903.bcs`, and all priest AI scripts matching the regexp `.*\(PRIE\|CLER\|HEAL\|PRST\|SHAM\).*\.BCS` are extended with the appropriate scripts.
* **2DA load-order safety.** `pdialog.2da` is appended with the Valen row and pretty-printed with `BUT_ONLY_IF_IT_CHANGES` so it isn't written to `override/` unless changed. `valenend.2da` is written with both DEFAULT data columns replaced by the same resolved epilogue STRREF.
* **Localization.** All user-facing strings are delivered through the TRA system. See the Localization section below.

#### Version-field note (why v1.0, not v2.0)

The CRE version field at `0x04` is a 4-byte ASCII string (`"V1.0"` or `"V2.2"`), not an integer. `valen.cre` ships as `"V1.0"` on current BG2EE. The TP2 reads it with `READ_ASCII 0x04 ver (4)` and compares against `~V1.0~`. The earlier v1.0.1 code compared against `~V2.0~` by mistake and silently skipped every binary write; the readme previously repeated that wrong claim. The guard is now correct and matches the file.

### Component 20: Give More Creatures Protection From Level Drain & Undead

An independent tweak that does not require Component 10. Scans every v1.0 CRE in `override/` and grants protection items based on class, race, and animation.

Specific work performed at install time:

* **Level drain immunity for inherently-undead-immune creatures.** Golems, elementals, demons, slimes, mists, and a handful of specific animations (solar, antisolar, mephit portal, shadow altar) receive an undroppable `LEVIMM` item in their first usable equipment slot. These creatures already resist level drain in the base game; the item makes the immunity explicit so mods that check for the item can see it.
* **Protection from undead for priests and paladins.** Clerics, paladins, and various multi-class hybrids (Fighter/Cleric, Cleric/Mage, Cleric/Ranger, Cleric/Thief, Fighter/Mage/Cleric) receive an undroppable `PRODEAD` item. Party NPCs are excluded via a `pdialog.2da` death-variable check — they have their own kit-based protections and should not receive an unconditional bonus.
* **v1.0-only scope.** The component reads race and class from v1.0-specific offsets (`0x272`, `0x273`). v2.2 CREs are intentionally skipped rather than risk corruption from a layout that hasn't been verified field-by-field. This is documented in the TP2 header.
* **Rule 55 guard.** A `SOURCE_SIZE >= 0x2A0` check protects the deepest read in the patch body (`READ_ASCII 0x280 deathvar (32)`). A truncated v1.0 CRE can't abort the install.

**Load-order note.** This component scans every v1.0 `.cre` file present in `override/`, not a fixed list. If installed after other mods, their qualifying creatures will also receive the protection items. This is intentional — the component's purpose is to protect creatures based on what they *are*, not on which mod shipped them. Installing Valen before Solaufein means Valen's sweep runs before Solaufein's CREs exist, so Solaufein's own creatures won't be affected.

------------------------------

## Install Order

Install Valen **before** Solaufein. Solaufein's Component 20 checks for `valenj.dlg` at install time; if Valen isn't installed yet, the Solaufein-meets-Valen cross-mod interjections are silently skipped. The reverse order doesn't break anything, but you lose that content.

The mod is compatible with any load order for the rest of a typical BG2EE/EET install. Recommended order:

1. Core engine rules (fixpack, tweaks that affect resources)
2. **Valen**
3. Solaufein (for the cross-mod interjections)
4. General text tweak and UI packs
5. End-of-load overhauls (SCS, CDTweaks, EET_End)

------------------------------

## Localization

Every user-facing string in the mod is delivered through the TRA system:

* **Component names** in the WeiDU installer menu (`@1000` and `@1001` in each `setup.tra`)
* **`REQUIRE_PREDICATE` failure messages** (`@1002` — the "requires BG2EE or EET" text)
* **Valen's biography** (`@15`)
* **Chapter 6 encounter display name** (`@16` = *"Cynara"*)
* **Joinable Valen display name** (`@17` = *"Valen"*)
* **All item names and descriptions** for Valen's Armor and Claws tiers
* **Spell names** for Blighted by the Sun and Gaseous Form
* **Vampire hunter names and barks**
* **The ToB epilogue** (`@999999` in `epilogue.tra`)

### Supported Languages

Ten languages ship with the mod. Non-English installs layer the English `setup.tra` as a base, then the language-specific overlay on top. Missing refs fall back to English automatically.

| Index | Language |
|---|---|
| 0 | American English |
| 1 | Français |
| 2 | Español |
| 3 | Deutsch |
| 4 | Polski |
| 5 | Italiano |
| 6 | Русский |
| 7 | Chinese (Simplified) |
| 8 | Chinese (Traditional) |
| 9 | Japanese |

### Adding a New Language

Drop a `setup.tra` into a new folder under `valenEE/tra/`, add the matching `epilogue.tra` if you want to translate the ToB epilogue, and add one `LANGUAGE` line to the TP2. The English base layer fills any gaps.

**Translator note:** `@17 = ~Valen~` was added in v2.0.5 and currently exists only in the American base. Non-English installs fall back to the English string automatically. If you want to localize it, add `@17` to your language's `setup.tra` with the character's name in your language.

------------------------------

## Modernization Layer (TPA Infrastructure)

The modernization engines operate via proper `DEFINE_PATCH_FUNCTION` routines:

### CRE Cleanup (`ee_cre_cleanup.tpa`)

* Sweeps and purges deprecated visual effect opcodes (142, 215, 248, 267) from v1.0 CREs
* Normalizes script resrefs and death variables to lowercase for cross-platform safety
* Bounds faulty legacy saving throw allocations to proper EE 0–20 brackets
* Clamps negative XP values to 0
* Does not erase authored content; the Noober-era audio purge was removed in v2.0.3 (see changelog)

### CRE Field Writer (`ee_cre_fields.tpa`)

* Reusable helper for writing script resrefs, script name, dialogue resref, and known-spells table into a v1.0 CRE
* Takes one parameter per field, never a delimited string — parser-safe across WeiDU versions
* Scoped exclusively to explicit value writes; does not clean up, remove content, or recover data
* Usable by any future joinable-NPC conversion

### Spell Cleanup (`ee_spell_cleanup.tpa`)

* Remaps spell school root arrays and eliminates out-of-bounds corruption
* Iterates accurately through individual SPL v1.0 extended headers to scrub faulty projectile indicators

### Cutscene Cleanup (`ee_cutscene_cleanup.tpa`)

* Targets compilation blocks to secure `StartCutSceneMode()` transitions safely
* Enforces normalized `CutSceneId(Player1)` parameters to mitigate cross-platform crashing anomalies

------------------------------

## Installation

### Requirements

* BG2EE (v2.0 or higher) **or** EET (Enhanced Edition Trilogy)
* Valen requires Throne of Bhaal content (which all EE/EET installs include)

### To Install

Extract the mod folder directly into your main game directory, then run one of:

* `setup-valenEE.exe` (Windows)
* `weidu --install setup-valenEE.tp2` (macOS/Linux)

When prompted for language, choose your preferred language. Non-translated strings fall back to English automatically.

------------------------------

## Verification Checklist

After installation, the following can be verified in Near Infinity:

* **`override/valen.cre`** — Name and Tooltip read *"Valen"*. Version reads `V1.0`. Gender reads `FEMALE — 2` at offset `0x0238`. Dialogue reads `VALEN.DLG`. Override script and Class script read `VALEN.BCS`. Script name reads `valen`. Small portrait = `VALENS.BMP`, Large portrait = `VALENL.BMP`. Animation = `THIEF_FEMALE_HUMAN — 0x6310`. HP = 70/70.
* **`override/valen.cre` known spells** — Entries 1–5 read `valen.spl` (Blighted by the Sun), `SPCL412.spl` (Set Snare), `SPIN104.spl` (Larloch's Minor Drain), `SPIN105.spl` (Horror), `SPIN101.spl` (Cure Light Wounds). `# known spells` = 5.
* **`override/valen.cre` sound slots** — `LEADER` = *"Of course. I'm the best choice."* `TIRED` = *"I must rest soon. I am weak when I am tired."* `BORED` = *"There're far better things to do than sit and wait."* `BATTLE_CRY1` = *"You are foolish to face my mistress!"* `BATTLE_CRY4` = *"You will fall by my hand!"* `HURT` = *"I will require healing as soon as possible."* `SELECT_COMMON1` = *"I serve my mistress, and no other."* `SELECT_ACTION1` = *"It will be done."* Every other slot reads *"No such index"* or resolves to an empty strref.
* **`override/c6valen.cre`** — Name reads *"Cynara"*. This is intentional for the Chapter 6 encounter.
* **`override/anast.cre`** — Small portrait reads `VALENS`.
* **`override/valenuh1.cre` through `valenuh4.cre`** — Names read *"Buffy"*, *"Faith"*, *"Kendra"*, *"Van Helsing"*. Each has the correct kit (Undead Hunter Paladin for the three Slayers, Cleric/Ranger for Van Helsing), correct gender, and correct script assignments. `valenuh1` and `valenuh4` have dialogue bindings; `valenuh2` and `valenuh3` are silent by design.
* **`override/pdialog.2da`** — Contains a `Valen` row with exactly eight columns (identifier + seven data values), formatted with proper column alignment.
* **`override/valenend.2da`** — DEFAULT row has a resolved epilogue STRREF in both data columns, formatted with proper column alignment.
* **WeiDU installer menu** — Component names display in the selected language (falls back to English for any ref not yet translated in that language's `setup.tra`).

**In-game smoke test.** Recruit Valen, spawn a fight, and confirm:

1. Her portrait tooltip and party bar show *"Valen"*, not *"Cynara"*.
2. Her Set Snare ability fires (click the ability icon).
3. Her Larloch's Minor Drain, Horror, and Cure Light Wounds fire from AI (`valen.bcs`) when conditions warrant — these are script-driven, not player-castable.
4. Clicking her portrait plays her SELECT_COMMON barks.
5. She joins the party via the correct dialogue (`valen.dlg`).

**Important:** use a fresh save created after this install when diagnosing any in-game display, dialogue, or bark bug. BG saves embed strref *numbers*, not strings; a save made under an earlier install can show unrelated text even when the current install is correct. See Rule 30b.

------------------------------
## Credits

### Original Author
- Westley Weimer

### Contributing (original)
- Homunculus — scripting, balancing, and internal detail work on
  Valen's vampiric powers (per the original readme's Thanks section)

### Translations
- French — Ly Meng, Laurent Duvernet, Cocobard
- Spanish — Clan REO ([REO]-Arturo, [REO]-Drohen-Trohen,
  [REO]-Killfax, [REO]-Styx, Thamar, Artemis_Entreri)
- German — Sebastian de Waal, Thalantyr, Tanis Eichenblatt,
  Falk Swoboda
- Polish — Grzesiek Miazga
- Italian — Al17, Kelvan
- Russian — Aerie.ru
- Japanese — ironthrone
- Chinese (Simplified) — kalabaka, yun395
- Chinese (Traditional) — kalabaka, yun395

### Modern EE/EET Edition
- /u/celestialfury (structural refactoring, Project Infinity
  support, WeiDU logic stabilization)

### Tools
- WeiDU, Near Infinity, Project Infinity

------------------------------

## Changelog

### 2.0.5 — Name and gender corrections on the joinable Valen CRE

**Name fix**

* The joinable Valen CRE (`valen.cre`) was being named `setup.tra @16` (*"Cynara"*), which is the display name for the Chapter 6 encounter CRE (`c6valen.cre`). Applying that name to the joinable NPC made the party UI and portrait tooltip read *"Cynara"* while every line of Valen's own dialogue (`valen.tra`, `valenint.tra`, `valentob.tra`) referred to her as *"Valen"*.
* Added `@17 = ~Valen~` to `setup.tra`. The joinable `valen.cre` COPY block now uses `SAY NAME1 @17 SAY NAME2 @17`.
* The `c6valen.cre` block retains `SAY NAME1 @16 SAY NAME2 @16` — the Chapter 6 encounter display name is intentional and unchanged.
* Non-English installs fall back to the English `@17` automatically through the layered LANGUAGE blocks. Translators can add a localized `@17` to their own `setup.tra` at their own pace.

**Gender fix**

* Added an explicit `WRITE_BYTE 0x0238 2` inside the CRE v1.0 guard on `valen.cre`. CRE v1.0 stores gender in a single byte at `0x0238` (1 = Male, 2 = Female, 3 = Neither). The shipped file already reads 2, so this is a defensive write to prevent drift if the pristine file is ever regenerated from a Male template.
* No separate Sex field exists in CRE v1.0; `0x0238` is the only gender-related field.

**Verification performed**

* Confirmed in Near Infinity that all four vampire hunters (`valenuh1`–`valenuh4`) are unchanged and correct: names, kits (Undead Hunter Paladin for Buffy/Faith/Kendra, Cleric/Ranger for Van Helsing), genders, scripts, dialogue bindings, and known-spells tables all match the original mod's design.

### 2.0.4 — Explicit CRE field writes

**CRE field writes on `valen.cre`**

* Added `ee_cre_fields.tpa` with the reusable `EE_SET_CRE_FIELDS` patch function. Takes one `STR_VAR` parameter per field — never a delimited list — so it parses cleanly on WeiDU 25100 and any future version.
* Wrote Valen's Dialogue resref (`valen`), Override script (`valen`), Class script (`valen`), Script name (`valen`), and full known-spells table (`valen`, `SPCL412`, `SPIN104`, `SPIN105`, `SPIN101`) explicitly from the mod tree. The original `valen.cre` shipped with a corrupt Dialogue resref (`VALE\x76` instead of `VALEN`) and a misaligned known-spells table (`\x14` in entry 0, shifting every subsequent entry by one byte).
* The fix follows the same pattern Solaufein's TP2 already uses for his death variable and dialogue resref: write the value you know, don't build a recovery scanner.
* Added a v1.0 version guard (`READ_ASCII 0x04 cre_ver (4)` + `STRING_EQUAL ~V1.0~`) around all binary writes on `valen.cre`, matching the file format the mod actually ships.

### 2.0.3 — Cleanup scope correction

* Removed the Noober-era audio purge block from `ee_cre_cleanup.tpa`. It was pasted in from an unrelated BG1 mod, had no EE-compatibility role, and was silently wiping every sound slot on `valen.cre` (making Valen completely silent). Removed outright rather than narrowed.

### 2.0.2 — Localization pass and read-depth guard

* `BEGIN @1000` and `BEGIN @1001` — component names delivered through `setup.tra`.
* `REQUIRE_PREDICATE ... @1002` — the "requires BG2EE or EET" message.
* Bumped the Component 20 `SOURCE_SIZE` guard from `> 0x274` to `>= 0x2A0` so the later `READ_ASCII 0x280 deathvar (32)` read can't out-of-bounds on a truncated v1.0 CRE.

### 2.0.1 — CRE version-field guard fix

* Fixed the CRE version check: `READ_ASCII 0x04 ver (4)` + `STRING_EQUAL ~V1.0~`, not `READ_LONG 0x04` + integer comparison. The latter returned `0x302E3156` and silently failed against every real file, so every cleanup and every binary write was skipped without an error.
* Affected `valen.cre` cleanup and the `anast.cre` portrait write.

### 2.0.0 — Modern EE/EET Edition

* Overhauled `.tp2` logic into designated Project Infinity components.
* Converted destructive macro definitions inside `.tpa` libraries into functional `DEFINE_PATCH_FUNCTION` formats.
* Automated runtime UTF-8 conversion via `HANDLE_CHARSETS`.
* Layered language TRAs (English base + overlay) for all ten languages.
* Converted component names to `@n` TRA refs.
* `BUT_ONLY_IF_IT_CHANGES` guards on every 2DA and store patch.
* `PRETTY_PRINT_2DA` on every modified 2DA.
* Swapped hardcoded classic epilogue strings for dynamic `RESOLVE_STR_REF`.
* Lowercase file paths and script resrefs for cross-platform safety.
