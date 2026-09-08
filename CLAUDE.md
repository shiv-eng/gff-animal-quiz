# What GFF Animal Are You

A viral personality quiz for the Global Fintech Fest crowd. Ten one-tap questions, one of 28 animal archetypes as a result, built so people cannot resist sharing it.

GFF 2026 runs 9 to 11 September at Jio World Centre, BKC, Mumbai. The audience is fintech founders, bankers, regulators, VCs, product and policy people. Most of them are on LinkedIn all day. The site is unofficial and has no affiliation with GFF, PCI or NPCI.

## Status

`index.html` is a complete working prototype. Single file, no build step, no dependencies, no backend. Open it in a browser and the whole flow runs.

It is a design and interaction prototype, not production code. The share buttons fire toasts instead of opening apps, and the counter on the cover is hardcoded.

## The concept

The result is presented as a naturalist's specimen plate, not a quiz result. Catalogue number, Latin binomial, rarity stamp, field measurements, natural habitat, field call, known weakness. The framing is what makes it feel collected rather than generated, and collected things get posted.

Three mechanics carry the sharing, and none of them should be removed without a replacement:

1. **The pairing.** Every result names another animal and says "Everyone has one. Send this to yours." It gives the user a specific person to forward to. This is the strongest loop in the product.
2. **The rarity stamp.** "3% of the floor" converts a result into a brag.
3. **The link preview.** The share screen shows the exact card that will land in a WhatsApp chat: "Priyank is the Octopus. Only 3% of GFF gets this one." This is a first-class screen, not decoration. In production it needs real OG image generation.

## Running it

```
open index.html
```

Fonts load from Google Fonts, so it needs a network connection to look right. Everything else works offline.

## Structure

Currently one file with two `<script>` blocks. The first holds the `ANIMALS` dataset, the second holds questions, scoring and rendering. If you split it, keep this shape:

```
index.html
src/
  data/animals.js      ANIMALS
  data/questions.js    Q
  data/traits.js       TB
  scoring.js           classify(), trait maths
  screens/             cover, name, quiz, classify, result, gallery
  styles.css
```

Do not introduce a framework. The whole thing is under 30KB of logic and the value is in the copy and the scoring, not the runtime.

## Data model

### `ANIMALS` (28 entries, keyed by slug)

```js
octopus: {
  g:     "🐙",                      // glyph
  n:     "Octopus",                 // rendered as "The Octopus"
  lat:   "Octopus omnipresens",     // fake binomial, italic
  r:     3,                         // rarity % shown in the stamp
  tag:   "Somehow, you are involved in all of it.",
  leave: ["7 new WhatsApp groups", ...],  // 4 items, see note below
  hab:   "Panel → coffee → booth → different panel → coffee again",
  quote: "Let's take this offline.",      // the field call
  weak:  "47 open opportunities. Zero calendar.",
  pair:  "ant"                            // key of another ANIMALS entry
}
```

`leave` items that begin with a number get the number split into a display column. `/^(\d+)\s+([\s\S]+)$/`. Items without a leading number render as plain rows. Keep writing them with leading numbers, the column is doing real work visually.

`pair` must always resolve to a valid key. Nothing validates this at runtime yet, so add a check if you touch the data.

### `Q` (10 questions)

```js
{
  f: "ARRIVAL",                      // field label, top right of the quiz screen
  q: "9:40am, Jio World Centre. What is your actual first move?",
  o: [
    { x: "Coffee. Obviously coffee.",
      a: { sloth:2, duck:1, butterfly:1 },   // animal weights
      t: [0,0,0,0,0] }                       // trait deltas
  ]
}
```

`t` is `[hustle, shipping, vision, rigour, spotlight]`.

Questions run 4 to 6 options. More than 6 breaks the single-screen mobile layout.

### `TB` (trait baseline per animal)

`[hustle, shipping, vision, rigour, spotlight, buzzwordProbability]`, all 0 to 100. The sixth value drives the "Odds you say ecosystem out loud" bar, which is a fixed joke per species rather than a computed number.

## Scoring

Raw score is the sum of `a` weights across the ten chosen options. The winner is not the raw maximum.

```js
TOTW[k] = Σ over questions of (max weight k can receive in that question)
winner  = argmax( score[k] / TOTW[k] ** 0.7 )
tiebreak: lower ANIMALS[k].r wins
```

**The normalisation is load bearing.** The first version used raw scores and sent 38% of users to three animals while leaving the Monkey mathematically unreachable. Dividing by total available signal means an animal cannot win just by appearing in more questions, and the 0.7 exponent keeps thinly-covered animals from winning off a single answer.

Verified over 300,000 uniform random runs: every one of the 28 lands between 1.6% and 7.0%. If you edit `Q` or add an animal, re-run that simulation and check the floor is still above 1%.

Real users answer coherently rather than randomly, so live distribution will be more concentrated than the simulation. That is correct and desirable.

### Trait bars

```js
user  = min(99, round(rawTrait / maxT[j] * 100))     // maxT = [26,26,26,26,22]
final = clamp(round(user * 0.45 + TB[winner][j] * 0.55), 11, 99)
```

The 55% weighting toward the species baseline is deliberate. Pure user scores produced flat, samey bars that undercut the archetype. Never let a bar hit 0 or 100, both read as broken rather than funny.

## Screen flow

```
cover → name → quiz (×10) → classifying (1.75s) → result → share
                                                      ↓
                                                   gallery
```

Notes on behaviour that matters:

- **Quiz** advances automatically 230ms after a tap. No next button. Back is available and preserves the previous pick.
- **Classifying** is a deliberate 1.75s pause with shuffling glyphs. Do not shorten it. The delay is what makes the reveal land.
- **Name** is optional and only exists to personalise the share preview.
- **Result** animates the stat bars in with a 70ms stagger, once, on entry.
- Progress renders as ten slots, not a percentage bar.

`prefers-reduced-motion` is respected via a `.reduce` class on body.

## Design system

The visual concept is a museum specimen plate on a dark botanical ground. Do not drift toward a generic dark SaaS card layout.

```
--ground     #13241E   deep museum green, all quiz screens
--ground-2   #1B3128
--plate      #F0E9D8   the specimen card, result screen only
--ink        #16211C   type on the plate
--ink-soft   #5F6F66
--rule       #C3B99D   hairlines inside the plate
--marigold   #E8A33D   primary action, selection, progress
--vermilion  #BE4227   rarity stamp, buzzword bar
```

Type is Fraunces for display and the Latin binomials, Instrument Sans for UI. Fraunces uses `SOFT` and `WONK` axes on headings, which is where most of the character comes from. Do not substitute a default serif.

The plate has a double border (outer edge plus an inset rule via `::before`) and a fine dot texture. The stamp is rotated 2.5 degrees. These small imperfections are what stop it reading as a template.

Layout is mobile-first, max-width 430px, centred, with safe-area padding top and bottom. It should be designed and reviewed at phone width. Desktop is a centred column and that is fine.

## Copy rules

The jokes are the product. Treat copy changes with the same care as code changes.

- Specific beats clever. "It's basically DPI, but agentic" works because it is a real sentence someone says. "Innovating at the intersection" does not.
- Punch at behaviour, not at people. The Sloth came for the tote bag. The Sloth is not stupid.
- Every archetype needs one line that stings slightly. The weakness field is where the sharing impulse actually comes from, because being seen is more compelling than being flattered.
- Reference real GFF texture: UPI, DPI, circulars, RBI, tokenisation, agentic AI, Bharat, cross-border rails, the Q&A microphone, BKC traffic.
- No em dashes.
- Sentence case throughout. The only uppercase is the catalogue metadata, where it encodes structure.

## What still needs building

**Before it can go live**

- Real share intents. `wa.me/?text=`, LinkedIn share URL, X intent URL. Currently stubbed with toasts.
- Per-result URLs, `/animal/octopus`, so a shared link opens on the result and offers to take the quiz.
- OG image generation per animal, with the name baked in. The share preview screen is currently a mock of something that does not exist yet. This is the single highest-value production task, because the preview is the loop.
- A real counter for the cover tally. Currently hardcoded to 41,208.
- Analytics on the funnel: cover → start → completion → share, and result distribution by animal.

**Worth considering**

- A "tag your pair" flow where sending it to your pairing unlocks a combined card. Doubles the loop.
- Rarity computed live from actual results, so the stamp becomes true rather than decorative.
- A downloadable plate as PNG, for people who post screenshots to LinkedIn rather than links.

## Constraints

- Stays unofficial. Keep the disclaimer line on the result screen. Do not use GFF, PCI or NPCI logos, marks or colours.
- No accounts, no login, no email capture. Friction kills the loop.
- No data collection beyond anonymous result counts.
- Must work on a mid-range Android phone on conference wifi. Keep it a single lightweight page.

## Definition of done for a change

1. Every one of the 28 animals is still reachable above 1% over 300k simulated runs.
2. Every `pair` key resolves.
3. The full flow completes on a 375px viewport without horizontal scroll.
4. Reduced motion still works.
5. The result screen fits a screenshot that reads well in a LinkedIn feed at thumbnail size.
