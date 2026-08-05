# Determinare

A species-identification trainer built on iNaturalist. It shows you research-grade
photos (or sounds) of the taxonomic groups and regions you pick, you name the
species, and it tracks what you keep getting wrong.

Live app: `index.html` served as a static page (GitHub Pages from this repo).

---

## Orientation for a new session

Read this section first; it is the part that saves you time.

**The entire application is one file: [`index.html`](index.html) (~5,900 lines).**
There is no build step, no bundler, no package.json, no server, no backend, no
API key, and no test suite. You edit `index.html` and reload the browser. That is
the whole development loop.

This is a deliberate design constraint, not an accident — see
[Constraints](#constraints-do-not-break-these) before proposing any change that
adds tooling.

### Running it locally

A static file server is enough. The repo ships a launch config
([`.claude/launch.json`](.claude/launch.json)) that runs:

```bash
python3 -m http.server 8743
```

Then open `http://localhost:8743`. Opening the file over `file://` mostly works
but will break `fetch` against some endpoints, so prefer the server.

There is no hot reload — edit, then hard-reload the page.

### Deploying

Push to `main`. GitHub Pages serves `index.html` from the repo root. Nothing else
is required; there is no build or release step.

---

## How the file is organised

The `<script>` block is split into twelve numbered sections, in the style of a
tidy R script. The index is repeated at the top of the script. To jump to one,
grep for its banner comment (line numbers drift, banners don't):

```bash
grep -n "^// == " index.html
```

| # | Section | What lives there |
|---|---|---|
| 1 | Configuration & small helpers | All tunable constants, `escapeHtml`, `levenshtein`, `shuffle`, `toast` |
| 2 | Talking to the iNaturalist API | `apiGet` (retry + timeout), taxon/place autocomplete, observation fetch |
| 3 | Turning observations into quiz questions | `toQuestions` — the observation → question shape mapping |
| 4 | Game state | The single `game` object; certificate species lists; SRS constants |
| 5 | Players & scoring, saved in the browser | Profiles, `localStorage` load/save, backup/restore, difficult list, SRS, streak |
| 6 | Weak spots, milestones & share-codes | Family/genus/species rollups, accuracy maths, export/import codes |
| 7 | Autocomplete | Setup pickers, the `"dro rot"` abbreviation matcher, typo-tolerant answer checking |
| 8 | Setup screen | Taxon/region/certificate/custom-list pickers, first-run intro, page translation (i18n) |
| 9 | Playing a lesson | Question queue, GBIF gap-filling, all four answer modes, hints, review pile |
| 10 | Rendering | Screens, photo gallery, lightbox, range maps, feedback banner, cow mascot, results, progress panel |
| 11 | Wiring up the controls | All `addEventListener` calls, in one place |
| 12 | Start-up | Nine calls, in order — this is the whole boot sequence |

The four screens are plain `<section class="screen">` elements in the markup:
`#setup`, `#play`, `#done`, `#progressPanel`. `showScreen(name)` toggles them.

Four overlays sit outside those screens and are shown by toggling `.hidden`:
`#lightbox`, `#learnPopup`, `#mapPopup` and `#sheet`. The last one holds both
the Options controls and the language picker — they were folds on the setup
screen once, and only their position in the markup changed, so the ids every
handler reads are the same. `openSheet("options" | "lang")` shows one panel.

CSS is a single `<style>` block above the markup. There is no CSS framework.

---

## State and persistence

Everything is stored in the browser. There is no account system and no server —
"players" are just named local profiles.

| `localStorage` key | Contents |
|---|---|
| `determinare.v2` | The whole store: profiles, family cache, imported compare codes, custom lists |
| `determinare.stats.v1` | Legacy store, migrated on first load if present (`migrateOldStore`) |
| `determinare.intro.v1` | Whether the first-run walkthrough has been seen |
| `determinare.lang` | Selected UI language |

The in-memory `game` object (section 4) holds both the session config and the
active profile's data. Note the deliberate aliasing: `game.stats`, `game.srs`,
`game.confusions`, `game.daily`, `game.notes`, `game.difficult` and `game.wrongQ`
always point at the **active profile's** Maps, so the rest of the code never has
to know profiles exist. If you add a per-profile field, you must add it in three
places: the `game` object, `saveStore`, and `loadStore`.

Maps are serialised as arrays of values and rebuilt on load — `JSON.stringify`
cannot handle a `Map` directly.

Every `localStorage` write is wrapped in `try/catch`. Storage can be blocked
(private windows, sandboxed previews) and the app must keep working in memory
only. Do not remove those guards.

---

## External services

No API key is needed for any of these. Everything is a plain browser `GET`.

| Host | Used for | Where |
|---|---|---|
| `api.inaturalist.org` | Observations, photos, sounds, taxon & place autocomplete, species counts | Section 2 |
| `api.gbif.org` | Extra photos for image-poor species; distribution maps; per-country record counts for "Where?" mode | Section 9 (`gapFillGbif`, `gbifCountryCounts`), section 10 |
| `tile.gbif.org` | Basemap tiles for both maps | Section 10 |
| `en.wikipedia.org` | Species summary, etymology and ecology text | Section 10 |
| `translate.googleapis.com` | Whole-page translation | Section 8 (i18n) |
| `unpkg.com` | Leaflet 1.9.4 (the only JS dependency) | `<head>` |
| `fonts.googleapis.com` | Fraunces + Inter | `<head>` |

Two notes worth knowing before you touch either:

- **Leaflet is loaded unconditionally** even though only the optional map
  features use it. That was a conscious trade: it is small, and loading it
  eagerly avoids runtime `<script>` injection.
- **Translation does not use Google's website-translator widget.** There is a
  hand-rolled i18n layer in section 8 that calls the translation endpoint
  directly and re-translates injected text through a `MutationObserver`. This
  was done because the widget was flaky and could not cleanly revert. It also
  leaves Latin species names untouched. Page text is sent to Google when a
  language is selected.

---

## Domain behaviour a newcomer will get wrong

These are the non-obvious rules baked into the code. Most are documented inline
too, but they are easy to break from a distance.

**iNaturalist's `species_counts` endpoint is only reliable for its first ~500
results.** Paging past that returns errors or duplicates. So `fetchTopSpecies`
takes exactly one page, and `growChecklist()` adds every species you are actually
shown during play. That is why nothing on screen is ever unspellable even though
the initial checklist is capped. Do not "fix" this by adding pagination.

**Answer checking is deliberately forgiving.** `isCorrect` accepts genus +
epithet only, tolerates typos via Levenshtein distance scaled to word length
(`wordTolerance`), and accepts abbreviations like `dro rot` for
*Drosera rotundifolia*. In certificate mode it also accepts the *other*
taxonomy's name, because exam checklists and iNaturalist disagree on many names.

**Certificates are fixed species lists, not filters.** The Swiss field-botany
(InfoFlora 200/400/600), butterfly (BDM Z7), moss, lichen and bird lists are
hard-coded arrays in section 4, mapped to iNaturalist taxon ids. Each certificate
carries its own checklist taxonomy alongside the iNaturalist name, and
`game.taxonomy` selects which one is shown and accepted. They are queried in
chunks of `CERT_CHUNK` (60) to keep URLs short.

**Choose mode's hint ladder is deliberately shorter than Type's.** `buildHints`
takes the mode and stops after the family and locality tiers for Choose: the
genus tier and the "species word begins…" tier would hand over a question whose
answer is already on screen. Because the bulk `resolveFamilies()` only covers
species you already have stats for, `resetHints` fetches the family for a
first-time species on its own (`resolveFamilyFor`) so the ladder is never empty.

**"Where?" mode asks for a region, not a country.** A widespread species has
thirty countries and no single right one, so `GEO_REGIONS` maps ISO country
codes to about twenty coarse regions and `buildWhereQuestion` rolls GBIF's
country facets up into them. The answer is the region holding the most records;
distractors are regions where GBIF has *none*, so a wrong option is verifiably
wrong. A species too cosmopolitan to have one honest answer (no region reaches
`GEO_MIN_SHARE`) hands the question back to Choose rather than asking it badly.

**Every map box needs `z-index: 0`.** Leaflet gives its own panes z-index
400–700. Without a stacking context on the containing box those paint over the
answer input's suggestion dropdown (z-index 20) while you are typing. `.obs-map`,
`.study-map` and `.where-map` all pin themselves to 0 for this reason.

**The setup screen's stats panel has no family count on purpose.** iNaturalist
has no endpoint for "distinct families recorded in a place", and deriving one
means resolving the ancestry of every species in the list — a dozen-plus extra
requests per keystroke. Genera are free (a genus is the first word of the
scientific name) and are marked `+` when the 500-species page is a floor rather
than the whole list.

**Multiple-choice distractors and look-alike pairs must stay within one iconic
group.** Every question carries `iconic` (`"Plantae"`, `"Aves"`, …) precisely so
a grasshopper never turns up as an option among plants. `byGroup` enforces this,
and `loadStore` drops legacy cross-group confusion pairs that older versions
saved. (An inline comment in `toQuestions` still points at a `sameGroup` helper
that no longer exists — the logic lives in `byGroup`.)

**Round size can be `Infinity`.** "No limit" is literally `Infinity`, and every
comparison against `round.size` already behaves correctly with it. The one
exception is the progress bar, which switches to a running count — see
`updateProgress()`.

**The observation link is revealed only after answering.** `revealObs` shows the
iNaturalist link post-answer; showing it earlier would give the answer away.

**Spaced repetition is Leitner boxes**, intervals `[0, 1, 3, 7, 16, 35, 90]` days
for boxes 1–6. Correct promotes one box, wrong drops straight to box 1. The daily
streak is driven by `DAILY_GOAL` (15 species/day).

**Names come from people, so escape them.** Player names, share-code imports and
custom list contents all reach `innerHTML`. Use `escapeHtml` — the codebase does
this consistently and it is the app's only real injection surface.

---

## Constraints (do not break these)

1. **Single file, zero build.** No npm, no bundler, no transpiler, no framework.
   If a change seems to need one, it almost certainly needs a different design.
2. **No backend, no accounts, no API keys.** All persistence is `localStorage`;
   sharing works by copy-paste share-codes, not a server.
3. **Keep the numbered sections.** New code goes inside the section it belongs
   to. If you add a section, update the index comment at the top of the script.
4. **Match the existing comment voice.** The file is heavily commented in plain,
   explanatory prose that says *why*, not *what*. Terse or absent comments will
   look out of place.
5. **Guard every storage and media access.** Broken photo URLs, blocked storage
   and rate-limited API calls are all expected conditions with existing fallback
   paths (`onMediaBroken`, `fallbackPhoto`, the 429 retry in `apiGet`).

---

## Testing

There is no automated test suite. Verification is manual, in a browser:

- Start a round on the default (Plantae · World) and answer a few questions.
- Check the browser console for errors and the network tab for failed calls.
- Exercise the mode you touched: Type / Choose / Study / Where?, photo vs sound,
  a certificate, a custom list, the review pile, and the progress panel's three tabs.
- For anything touching maps, check both the small inert one and the enlarged
  pop-up, and confirm the suggestion dropdown still covers the map while typing.
- Test with `localStorage` cleared to confirm first-run behaviour (the intro
  walkthrough and empty-profile paths).

---

## License

Copyright (c) 2025-2026 Marco Barandun. All rights reserved.

This is proprietary software, not open source. The source code is public so
that it can be viewed and run, but no permission is granted to copy, modify,
redistribute, re-host, or create derivative works from it without prior written
permission. See [`LICENSE`](LICENSE) for the full terms.
