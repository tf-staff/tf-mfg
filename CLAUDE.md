# TF-Manufacture — Trello Power-Up

Manufacturing documentation Power-Up for the UC Davis Tech Foundry (Davis/GBSF + Aggie Square, Sacramento). Engineers attach structured "manufacturing notes" to Trello project cards so previous work can be replicated. Internal tool, used daily by TF staff.

**Current version: 2.4** — the whole Power-Up is the single file `tf-manufacturing.html`, served via GitHub Pages from this repo. The Trello Power-Up's iframe connector URL points at that file. There is no build step: edit the file, bump `VERSION`, push.

## Release workflow (owner-approved standing permission)

The owner is not a developer; explain things plainly. **Always merge to `main` immediately after development and testing are complete** — no PR, no waiting for review. Develop on the session's working branch, test (see convention 3), then push the same commit to `main` so GitHub Pages serves it to Trello. Every commit message starts with the new version, e.g. `v2.5: add X`. Git history is the backup: to roll back ("go back to 2.3"), restore that version's file and ship it as a new commit with a bumped version — never rewrite or delete history. To undo one change from a multi-change release, revert only that change. Rollbacks don't touch Trello card data, but older versions may not render fields added after them (e.g. 2.3 shows UV notes as Other).

## Non-negotiable conventions

1. **Bump `var VERSION` on every functional or visual change.** The card-back section title renders as "TF-Manufacture v2.4", which is the only quick way to confirm Trello's iframe cache picked up a push.
2. **Color restrictions (owner-imposed, strict):** the UI uses only the Trello dark-card palette — charcoal grays (`--bg #242528`, `--panel`, `--line #46474b`), the gray→white text ramp (`--dimmer/--dim/--text/--head`), and Trello blue (`--blue #579dff` dark / `#0c66e4` light). No green/red/amber/purple. Emphasis and state are conveyed by text shade and weight only (e.g. Success/Failed outcomes render bright+bold, In progress/Partial dim). Blue appears only in small intentional accents: primary Save buttons, panel section titles, checkbox accents, input focus rings. Glyphs must be monochrome (use `︎` variation selector on ⚠; use currentColor SVGs, never emoji). Deviating colors are allowed only sparingly and with intention.
3. **Simulation-first development:** verify every change before shipping — `node --check` on the extracted script, then drive the real file in headless Chromium (Playwright is the established harness). The summary view is tested by framing `tf-manufacturing.html?view=summary&d=<encoded JSON>` inside a local harness page. Past regressions that mattered: a `.hidden` class with no CSS rule (fields "hidden" in logic but visible on screen — hence the global `.hidden{display:none!important}` and `[hidden]{display:none!important}` rules), and `.fgrp{display:flex}` overriding the native `hidden` attribute.
4. **Never silently truncate documentation.** Trello caps a plugin's card data at 4096 chars (shared scope). `MAX_BYTES = 3980`; saving a note that would exceed it is refused with a visible warning, and the header shows a live "Card storage" meter. Keep stored keys compact; empty fields are stripped by `packNote`.
5. Deliverables are used professionally as-is; keep polish high.

## Architecture — the four-mode chassis

Shared with the sibling Power-Ups (TF-Recharge, TF-Message, TF-Plan, TF-QMS — same `tf-staff` org, one repo each, same pattern). One HTML file boots into one of four modes based on `?view=` and framing:

- **connector** (no `?view=`, framed): hidden iframe Trello loads for every card. Registers `card-back-section`, reads `t.get('card','shared','mfg')` (1.5 s timeout race), builds the summary iframe URL with the data embedded in the `d` param (signed via `t.signUrl`) so the summary needs no Trello round-trip, estimates iframe height (`estimateFaceHeight`), and provides the "Open" modal action. Also listens on `BroadcastChannel('tf-mfg')` to perform **close-proof saves** relayed from a dying editor iframe.
- **summary** (`?view=summary`): compact read-only card-back view. Notes render inside one faint bounding box (`--mf-box`) with 1px hairline dividers between notes (TF-Message style); the empty state ("No manufacturing notes") renders as plain unboxed text to match TF-Recharge. Capped at `FC_MAX = 6` notes + "+N more". Fallback chain so it never sticks on loading: `d` param → `t.render` → 600 ms retry → 3.5 s forced finish.
- **editor** (`?view=editor`): full tool in a Trello modal (height 1100). Author tag from `t.member` as "First L.". Debounced save (800 ms) via `t.set('card','shared', {mfg})`; on `pagehide`/`beforeunload`/`visibilitychange` it saves directly AND relays the payload over BroadcastChannel to the connector (duplicate writes are idempotent).
- **standalone** (opened directly in a browser): full tool persisting to `localStorage['tf-manufacturing-local']`. No "simulator" language anywhere, ever (owner request) — it's just the tool with browser-local persistence.

Theme: Trello sets `data-color-mode` on `<html>`; `applyTheme` detects via `t.getContext()` with timed retries (150/500/1200/2500 ms) because the attribute arrives late.

## Data model

Card storage: `t.set/get('card','shared','mfg')` → `{ v:1, n:[packed notes], cmats:[custom materials] }`.

Note fields (compact keys, empty values stripped): `i` id · `t` timestamp · `by` author · `m` modality id · `p` platform id · `mt` material · `vr` variant · `mc` machine · `st` process settings · `of` offset (cut modalities) · `pr` profile used · `cp`/`cpn` custom cut profile saved / name · `vn`/`pn`/`qy` vendor / part number / quantity (hardware) · `bd` board · `ic im iy ik iw ig` ink ml per cartridge (UV) · `wl`/`cl` white-base / CMYK layers · `fs` finish (`flat`/`raised`) · `vw`/`vi`/`vc` vapor wall mm / intensity / chamber · `fl` design file name · `rq` requirement chip ids · `rx` requirement detail · `dv`/`dx` SOP deviation flag / description · `o` outcome (`wip/ok/part/fail`) · `lc` location (`''`=Undefined, `davis`, `as`) · `cm` "additional context provided in comments" flag · `nt` notes.

Custom materials (`cmats`): `{n, m, p}` — name, modality, platform (`''` = all platforms of that modality). Defined in the ⚙ Card Settings panel; merged into the Material datalist for matching modality/platform. Card-scoped today; **board-scoped storage is the agreed v-next if staff keep re-adding the same materials per card.**

## Form behavior (rolling + relevance-filtered)

The form is **rolling**: only the "Pull from card description" button and Modality chips show until a modality is picked (`#formRest` hidden). Modality chips are rendered alphabetically with **Other pinned last** (`modsSorted()`; data order in `MODS` is irrelevant).

Per-modality field relevance (`updateFieldVis`):
- Offset / Profile Used / Custom-cut-profile (`#cutGrp`): only `laser`, `wjet`, `blade` (`CUT_MODS`).
- Vendor / Part number / Quantity (`#hwGrp`): only `hw` (Hardware); machine hidden for it.
- UV fields (`#uvGrp`: board segment, per-cartridge ink ml row C/M/Y/K/W/Gloss, white-base + CMYK layer counts, finish Flat/Flat-raised): only `uvp`.
- Vapor fields (`#vpGrp`: wall thickness 1–20 mm step 0.5, intensity −3…+3, chamber 25/50/75/100%/Oversized): only `vapor`.
- Design Files (`design`): ONLY file name (`#fFl`) + Notes + comments checkbox; material/variant/machine/settings/requirements/deviation/outcome/location all hidden.
- Material/Variant hidden for `scan`, `dicom`, `design` (`NOMAT_MODS`).
- Values from hidden fields are dropped at collect time — nothing stores invisibly.

Material ↔ Variant: materials may be `{n, v:[variants]}`; typing/picking a material repopulates the Variant datalist (e.g. BioMed → Clear/Amber/Black/White; PA12 → HR/CB Color/Glass Bead). Datalist-backed inputs draw their own CSS chevron (`input.fin[list]`) because Chrome's native indicator is hover-only/invisible on dark inputs; the native indicator stays as an invisible click target.

Location: segmented Location Undefined (default) / Davis / Aggie Square. No machine→location auto-fill (removed deliberately — explicit beats inferred for documentation).

## Capability catalog (MODS)

Modalities: 3D Printing (FDM/FFF, CF-Reinforced FDM, MJF, SLA, PolyJet, Microfluidics, Filament Compounding, Other) · 3D Scanning · Assembly · Casting/Molding (Resin, Roto, Silicone, Injection, Compression) · Design Files · DICOM/Imaging · Hardware · Laser Cutting (CO2, Fiber) · Machining (CNC Mill, CNC Router, Lathe, Manual Mill, Support Equipment) · Medical Bolus · PCB · Sheet Metal · UV Printing · Vacuum Forming · Vapor Polishing · Vinyl/Blade Cutting · Waterjet · Other. ("Composites" was removed on request; notes saved under a removed modality degrade to OTH.)

Machine lists are real TF fleet (from the equipment pages): Bambu X1C/H2D/H2C, Prusa XL, Voron 2.4, OrangeStorm Giga, Stratasys F-120, Vision Miner 22 IDEX, Markforged Mark II, HP Jet Fusion 5200 & 580 Color, Form 3B/3BL/4B/4BL, Stratasys J5 MediJet & Digital Anatomy, CADworks3D ProFluidics 285D, Filabot EX2, Kern Micro 24 (200W, Davis) / LC50 LaserCELL (450W, AS) / FC50 FiberCELL (1000W, AS), HAAS UMC 500 & SR-3015, Kent KTM-3VKF, Acer E-Mill, Jet GBH-1340A, Shopbot 4x4, Shaper Origin, Wazer Pro, Langmuir Titan 25T, Formech 450 DT, Silhouette Cameo 4 Pro, Morgan Press GT-100T, LMS PIM-MODEL-150A, eufyMake E1 UV Printer. Keep wattages in the Kern names (disambiguates the two CO2 machines in old notes). "Plastics" is banned as a material — too vague; list specific polymers.

Laser offsets pair colors (red/black/blue/green/yellow/pink) with fits (strong interference / interference / light interference / clearance / generous clearance); the Offset datalist offers all 30 combos and values are free-text, signed, often several per line. Typical laser settings format: "60% power · 40 psi air · 1.5 ips · 10,000 Hz".

## Description parser ("Pull from card description")

Client intake (Jotform → Zapier) compiles a structured block into the card description. The parser (`parseDesc`) flattens raw Trello markdown (links → text, backslash escapes stripped, tokens stripped, whitespace collapsed) and slices values between ~50 known labels (`DESC_LABELS`) using **word-start-anchored regex boundaries** (plain `indexOf` once truncated "Rigid body…" to "Rig" via the `ID ` label — don't regress this).

Extraction and inference rules:
- `Services:` → modality (`SERVICE_TO_MOD`).
- `Desired Printing Technology:` **or** `Desired 3D Printing Technology:` → platform (`TECH_TO_PF`); a printing technology **implies** the 3D Printing modality even when Services is absent (real cards lack it).
- Client Defined Material fields → material, preferring the platform-matched field, else first non-empty (which then implies its platform). The intake form's "**Stereolithograpy** Materials:" typo is matched verbatim with the corrected spelling as fallback — keep both.
- Non-empty Characteristics (Color/Opacity, Mechanical, Resolution, Resistance, Biocompatibility, Cost) → requirement chips + detail line; `All Important Features:` prepends.
- `Collection/Delivery Preference:` → location (davis / aggie square|sacramento → as).
- No intake block → "No service details found in the description." Never guess.

In Trello the button reads `t.card('desc')`; standalone it opens a paste box. A "Pulled: …" receipt tells the engineer exactly what came from the client. If intake labels change upstream, extend `DESC_LABELS` + mapping tables.

## Version history (summary)

1.0 chassis+catalog · 1.1 variants, machines, location, comments flag, new modalities · 1.2 Kern lasers · 1.3 laser offset/profile fields · 1.4 monochrome palette sweep · 1.5 relevance filtering, settings panel/custom materials, placeholder purge, color-restraint pass · 1.6 fixed `.hidden`/`[hidden]` CSS (filtering actually visible) · 1.7 "simulator" language removed · 1.8 datalist chevrons · 1.9 description parser · 2.0 renamed TF-Manufacture, version in title, summary bounding box · 2.1 summary dividers + unboxed empty state · 2.2 parser hardening (modality inference, word boundaries, label variant) · 2.3 Assembly/Hardware modalities, Composites removed, rolling form, alphabetical chips · 2.4 UV Printing (eufyMake, per Hayley Kumagai's spec), Vapor Polishing, Design Files.

## Backlog / agreed future directions

- Board-scoped custom materials and a board-level laser offset library (per material/thickness/fit) if per-card definitions prove repetitive — same settings UI, `t.set('board','shared',…)`.
- Spillover storage (REST API comment/attachment) only if the 4 KB cap actually bites in practice (~8–15 detailed notes per card).
- "More…" collapse for the modality chip row if 18 chips feels crowded in real use.
