# Suschecker web - technical change brief

28 Oct 2026 · revised

For Claude Code, working in the `suschecker-web` repo. The live site (commit 42eefb8, 23 Jul) reads the retired pillar model, so every scan renders 0/100 with no brand. This brief replaces its data layer and content to match the axis system the Bot already writes.

## Scope

Technical changes only: data binding, status logic, removals and page content. The current layout, components, type and colour stay as they are. Where a removed component leaves a gap, use the existing design system's patterns. Do not introduce new visual design.

Sources of truth, in order:

- `axis_output_spec_v1.0.md`: what the grade means and how it is shown.
- The Bot scorer `Calculate SustainScore1`, methodology 4.0.0.
- The Bot's `Append row in sheet` column schema (ops repo snapshot, 28 Oct).
- The live `Suschecker Results` sheet as read on 28 Oct: 48 rows, all methodology 4.0.0.

Nothing here changes the pipeline. The one pipeline change this work depends on is listed separately in section 10 and is an ops task, not part of this repo.

## 1. Data source and column contract

Keep the existing read: the Google Sheets visualisation endpoint, sheet `1q4MqSkfLaoljDVZ9NtLo9Ry4B2EeeetPULkxh7rhUPk`, `gid=0`, with columns resolved by header label. Only the labels change.

1.1 **Version guard.** Drop any row whose `methodologyVersion` is not 4.0.0 or later, before dedupe and before any count. Compare as semantic versions (major, minor, patch as numbers), never as strings: a string comparison puts "10.0.0" before "4.0.0". A missing or unparseable version is dropped. The sheet is clean today. The guard makes it impossible for a pre-rebuild row to be displayed.

1.2 **Dedupe.** One item per `userUrl`, keeping the row with the latest `timestamp`. Note the consequence: a later `scrape_failed` row replaces an earlier graded row for the same URL. That is the intended default, because it is the most recent truth. [Decision for Sam: alternatively keep the latest `ok` row and note that a later rescan failed. Implement the default unless told otherwise.]

1.3 **JSON columns.** `axisA`, `axisB`, `composition`, `unknownMaterials`, `certifications` and `brandCertifications` arrive as strings. Parse each one in a try/catch. A parse failure is treated as absent, never as zero.

| Column | Type | Feeds |
| --- | --- | --- |
| `scanId` | string | identity, pending-scan match |
| `timestamp` | ISO 8601 | ordering, dedupe |
| `userUrl` | URL | identity, result route, brand count (4.1) |
| `status` | `ok` \| `insufficient` \| `out_of_scope` | which view renders (section 2) |
| `failureReason` | `no_composition` \| `scrape_failed` \| `scrape_blocked` \| null | no-score copy (section 2) |
| `methodologyVersion`, `ecobalyseScoreVersion` | string | version guard, result footer |
| `brandName` | string | brand label (replaces `brand`) |
| `productName`, `productImage`, `category` | string | product header |
| `grade` | A-E | the grade |
| `position` | 0-1 | position within grade |
| `axisA_processingStages` | 0-4 | stages disclosed |
| `axisA_productPresent`, `axisA_productApplicable` | int | product count |
| `axisA_brandPresent`, `axisA_brandApplicable` | int | brand count |
| `axisA_programmes` | int | programmes count |
| `axisA` | JSON | element lists (below) |
| `axisB` | JSON | context block (below) |
| `axisB_caveats` | `;`-joined string | caveat list |
| `simulatorUrl` | URL | external check link |
| `composition` | JSON object, fibre to % | composition line |
| `knownPct`, `unknownMaterials` | number, JSON array | partial-recognition note |
| `certifications` | JSON array | product certifications |
| `brandCertifications` | JSON array | brand certifications |
| `ecobalyseStatus`, `ecobalyseError` | string | axis B availability |

**Fields inside `axisA`:**

- `product.documented[]`, `product.missing[]`
- `brand.documented[]`, `brand.profileUsed`, `brand.profileAgeDays`
- `programmes.count`, `programmes.of[]`
- `processingStages.present[]`, `processingStages.of`
- `elements["product.<name>"].applicable`

**Fields inside `axisB`:** `ecs`, `intensityPer100g`, `comparablePer100g`, `massKg`, `durability`, `anchors.low|reference|high`, `positionInRange`, `verdict`, `caveats[]`. Any of these may be null. Null renders as not available, never as 0.

`parseQuality` is internal and is not displayed.

## 2. Scan submission and status handling

2.1 **Submission.** Keep the POST to `https://n8n-production-9085e.up.railway.app/webhook/suschecker-test` with body `{ url }`. This matches the live Webhook Trigger path. The webhook only exists while the Bot workflow is active in n8n; it is active as of 28 Oct. The request stays `no-cors`, so webhook errors are invisible to the site. The polling timeout is the only failure signal, and its copy must say the result has not arrived, not that the scan failed.

2.2 **Polling.** Keep the current poll (about 150 s), matching on `userUrl` and a `timestamp` later than submission. A newer row for the same URL replaces the old one.

2.3 **Status routing.** Exactly one view per item:

| `status` | `failureReason` | Renders |
| --- | --- | --- |
| `ok` | null | full result (section 3) |
| `insufficient` | `no_composition` | no-score: the page does not state fibre composition, so no grade is possible, and non-disclosure is itself the finding |
| `insufficient` | `scrape_failed` | no-score: the page could not be read this time; this says nothing about the brand; try again later |
| `insufficient` | `scrape_blocked` | no-score: the site blocked automated reading; this says nothing about the brand |
| `out_of_scope` | any | no-score: not a clothing item |
| anything else | any | no-score: the scan finished without a result |

The current build sends every `insufficient` row to the "doesn't state the fibre composition" copy. On 28 Oct, 8 of 13 insufficient rows were scrape failures, so the site currently states non-disclosure about brands whose pages were simply not read. This is the highest-priority fix in this brief.

2.4 **Remove the inference.** Delete the fallback that infers `ok` or `insufficient` from the presence of a numeric `score`. `status` is always written by the Bot.

## 3. Result view (`status: ok`)

Each element, the field that feeds it, and the rule that applies.

3.1 **Grade.** `grade`, A-E. Labelled as a *disclosure grade* wherever it appears, never as a sustainability or impact score.

3.2 **Position within grade.** `position`, 0-1, shown as placement inside the letter, not as a percentage score. It ranks products that share a letter. It does not move them between letters.

3.3 **What earns the letter.** The ladder, with the product's own rung marked:

- A: composition + assembly country + all four remaining processing stages.
- B: composition + assembly + one to three processing stages.
- C: composition + assembly + at least one brand supply-chain element (supplier list, audit reporting, manufacturing countries).
- D: composition + assembly.
- E: composition only, or assembly missing.

Show `axisA_processingStages` as n of 4 (`processingStages.present[]` names them: material origin, spinning, fabric, dyeing). State that brand disclosure can lift a product to C and no further.

3.4 **Product elements.** `axisA.product.documented[]` and `axisA.product.missing[]`, with `axisA_productPresent` of `axisA_productApplicable`. Elements with `applicable: false` (for example animal welfare on a non-animal fibre) are shown as not applicable, not as missing.

Display names: composition, materialOrigin = material origin, spinningCountry = spinning country, fabricCountry = fabric country, dyeingCountry = dyeing country, assemblyCountry = assembly country, printing, weight, price, certificationsStated = certifications stated, chemicalMgmt = chemical management, animalWelfare = animal welfare.

The axis spec also captures `packaging`, `endOfLifeMarket`, `transportMode` and `factoryName`. These are not in the grade set. Never list them as missing graded elements. Either omit them or show them under a separate "also stated" heading with no count.

3.5 **Brand elements.** `axisA.brand.documented[]` with `axisA_brandPresent` of `axisA_brandApplicable`. Elements: supplierList = supplier list, auditReporting = audit reporting, livingWage = living wage position, manufacturingCountries = manufacturing countries.

When `brand.profileUsed` is false, say no brand profile was available for this scan rather than listing four missing elements.

When `brand.profileUsed` is true, add one line: "Brand information comes from an automated scan of the brand's website and may be incomplete." Brand profiles are currently built by an interim pipeline (ops item 8.1 replaces it), and brand elements can already lift a product to C, so the result must not present them as complete. [Sam: confirm wording.]

3.6 **Programmes.** `axisA_programmes` and `axisA.programmes.of[]`: takeBack = take-back, repair, resale, guarantee. Counted, never shown as a proportion: there is no "out of 4".

3.7 **Composition.** `composition` as fibre and %. When `knownPct` is below 100 or `unknownMaterials` is non-empty, list the unrecognised fibres and say the environmental figure covers only the recognised share.

3.8 **Certifications.** Two separate lists that are never merged:

- `certifications`: stated on this product page.
- `brandCertifications`: held at brand level.

No "verified" wording on either. Nothing in the pipeline verifies a certificate.

3.9 **Environmental cost, context only.** From `axisB`, and visually and textually separate from the grade:

- `ecs`: total modelled environmental cost, in Ecobalyse impact points. Always show the unit. It is not money, and an unlabelled "cost" reads as euros.
- `intensityPer100g`: impact points per 100 g.
- `verdict` against conventional cotton (lower / around / higher), only when not null.
- Caveats from `axisB.caveats[]` in plain words. `default_mass_0.15kg_all_categories` becomes "weight not stated; modelled at Ecobalyse's 0.15 kg default". `durability_coefficient_not_returned` becomes "durability not modelled for this scan".

State once on the result that this figure does not affect the grade. When `ecobalyseStatus` is not ok, show the grade alone, with "environmental model unavailable for this scan".

3.10 **Check it yourself.** `simulatorUrl`, linked as the exact parameters in the French government's Ecobalyse simulator.

3.11 **Footer line.** Methodology `methodologyVersion`, Ecobalyse `ecobalyseScoreVersion`, and the scan date from `timestamp`.

## 4. Dashboard and ticker

4.1 **Stats.**

- Products scanned: count after the version guard and dedupe.
- Brands: distinct brand websites, from the host of `userUrl` (lowercase, leading `www.` removed). Do not count `brandName`: it comes from the product page and varies ("Levi's", "Levi Strauss"), so it double-counts. The pipeline now keys brands by domain, and this matches it.
- Graded: count of `status: ok`.
- Grade distribution: count per letter A-E.

Remove "average SustainScore" and "rated A or B".

4.2 **Filters.**

- Category from `category`.
- Brand from `brandName` (display only; grouping by domain is fine if simpler).
- Grade A-E. Drop F.
- Status: graded / not graded.
- Material from `composition` keys.

4.3 **Ordering.**

- Most recent: `timestamp`.
- Grade: letter, then `position` within the letter.
- Stages disclosed: `axisA_processingStages`.

Remove score high/low.

4.4 **Ranking chart.** Replace the score-axis scatter with grade on one axis and `position` within grade. Do not plot `axisB_ecs` across products: absolute ECS mixes parameterisations between products with and without a stated price (finding 46).

4.5 **Cards and ticker.** Grade letter, brand, category. No numeric score.

4.6 **Not-graded items.** `insufficient` and `out_of_scope` rows appear in the dashboard with their status, not with a grade. They are excluded from the grade distribution.

## 5. Remove

5.1 Fields read that no longer exist: `score`, `scoreBreakdown`, `materials`, `verified`, `unverified`, `country`, `price`, `currency`, `confidence`, `parseConfidence`, `telegramMessage`, `brand`.

5.2 Logic: `gradeFromPct` and the 85/70/55/40/25 bands, the F grade, `pillarDefs` and the pillar maxima (30/20/20/15), `penDefs` and every penalty, the `/100` display, and the score-inferred status fallback (2.4).

5.3 Components:

- The Four Pillars panel and its "average across all scans" label.
- The pillar radar fallback for the impact profile.
- Penalty chips.
- The "Verified" list and the "n verified" chip.
- Data confidence and confidence chips.
- The "typical item in the same category" benchmark (finding 22).
- The country and price summary lines.

5.4 The 16-category impact profile. Its input columns are no longer written by the Bot. It returns only after the Ecobalyse `/detailed` migration, which is open work in the ops repo.

5.5 `IMG_OVERRIDES` keyed by URL. `productImage` is written by the Bot.

The seven legacy columns written by the Bot's `Edit Fields` node (`materialFootprint`, `productCircularity`, `supplyChainLabor`, `environmentalCost`, `fastFashionPenalty`, `blendPenalty`, `syntheticPenalty`) are simply not read by the site. Their removal from the pipeline is section 10.

## 6. Content

Replace the copy on these pages. Facts only, from the axis spec.

6.1 **How it works.**

- Two axes, never combined. The grade measures how much of a product's supply chain the brand has published. The environmental figure is context from Ecobalyse, the French government's lifecycle model.
- The A-E ladder, and why it orders by rarity: composition and country of making are legally required on labels in most markets, so publishing them is compliance (D), not achievement.
- Why fibre does not grade: 12 of 16 modelled fibres sit within 12.4% of the range, and their order shuffles between Ecobalyse versions. Processing the tool cannot observe is about 79% of the modelled figure; fibre is about 19%.
- Known limitation: under the EU PEF weighting, polyester models better than cotton because land and water outweigh fossil feedstock and microfibre shedding. This is a documented weighting dispute, stated rather than corrected.
- What is not measured: nothing verifies that a stated composition is accurate, that a named country is where assembly happened, or that a certificate is current.
- Failure states: the tool refuses to grade, and says why, rather than filling gaps with defaults.

6.2 **About.** Remove "single honest number", "0–100 SustainScore", "A to F", "16-category life-cycle assessment to produce a grade" and "when we can’t verify a claim". Restate the claim as: how much of a product's sustainability profile the brand has published, with a modelled environmental cost alongside as context. Keep the paragraph on brands being unable to pay.

6.3 **FAQ.**

- "How accurate are the scores?" becomes: a disclosure grade, not a sustainability verdict; it reports what was published, not whether it is true.
- "Why did a product I like score low?": the cause is missing supply-chain disclosure, not blends or material.
- "What is the environmental cost?" adds: measured in Ecobalyse impact points, not money; context only, does not affect the grade; and gives the 0.15 kg default-mass caveat.
- Add: "Why does a scan say the page couldn't be read?" (`scrape_failed` / `scrape_blocked`).
- Add: "Why no grade?" (`no_composition`).

6.4 **Hero line, footer and meta text.** Remove "0–100 SustainScore", "materials, labour and a full 16-category life-cycle assessment" and "Scores are indicative, modelled from … EF 3.1". Replace with the disclosure-grade claim plus "environmental cost modelled with Ecobalyse v7.0.0".

## 7. Hold - do not ship yet

7.1 No brand-level grades, brand pages or brand leaderboards until ops item 8.1 (Profiler reads Discovery records) lands. Brand elements appear only inside a product result, as in 3.5.

7.2 No "new brand, check back shortly" state. The new-brand flow is not designed yet. When it is, the site already polls the sheet, so a second, updated row for the same `userUrl` is the expected delivery path.

7.3 No profile-age display. `brand.profileAgeDays` is null on every cache hit (finding 75).

7.4 No not-assessed brand state. Brand fields are still two-state upstream (finding 73). Show what `brand.documented[]` holds and nothing more.

7.5 No parent-group attribution yet. Once ops corporate-domain mapping lands, a product's brand data may come from its parent group (for example Inditex for Zara), and the result will need to say so. Nothing to build now; the field does not exist yet.

## 8. Runtime bugs in the live build

8.1 After opening a result, How it works, About and FAQ render blank. The console shows `TypeError: Cannot read properties of undefined (reading 'view')` in `componentDidUpdate`, repeated on every update. Guard the previous-state read.

8.2 SVG attribute errors: the strings `{{ dp.path }}`, `{{ selected.rada… }}`, `{{ d.x }}` and `{{ d.y }}` reach the DOM unrendered. These are the pillar radar and the ranking scatter. Both are removed or replaced in 4.4 and 5.3.

8.3 On mobile width the header nav overlaps the logo and the Dashboard button is not clickable (observed at 400 px). Fix within the existing layout; listed so it is not lost.

## 9. Acceptance checks

1. No rendered label, heading, chip or claim uses "/100", "SustainScore", "pillar", "penalty", "verified" or "confidence" as a description of the product or its grade. Explanatory copy that says what is *not* verified (6.1, 3.8) is allowed. Check by inspecting components and copy, not by raw text search alone.
2. Every result shows a brand name where `brandName` is non-empty.
3. A `scrape_failed` row and a `no_composition` row show different copy, and neither of the scrape states attributes non-disclosure to the brand.
4. A row with `methodologyVersion` 3.x added to a copy of the sheet does not appear anywhere. A row with `10.0.0` does appear.
5. A row with `axisB` null and `ecobalyseStatus` failed shows its grade, and shows no environmental figure and no 0.
6. How it works, About and FAQ render after visiting a result, with no console errors.
7. Dashboard counts match a manual count of the sheet after the version guard and dedupe, with brands counted by domain.
8. Every displayed ECS figure carries its unit.

## 10. Ops dependency (not this repo)

The Bot's `Edit Fields` node still assigns `materialFootprint`, `productCircularity`, `supplyChainLabor`, `environmentalCost`, `fastFashionPenalty`, `blendPenalty` and `syntheticPenalty` from a `scoreBreakdown` the scorer no longer emits. The names are saved with a leading `=` (ops `outstanding_work` 11.1). Remove those seven assignments, then delete the seven empty columns from the sheet.

This is a change to the Bot. `push-nodes.js` deliberately refuses the Bot, and updating it while active re-registers the webhook. Do it as its own ops task: confirm against the `Edit Fields` execution view from a fresh scan first, make the change deliberately, then run a test scan. The site does not depend on this change; it simply ignores those columns.
