# Developing Oscillo

Oscillo is a single self-contained file — [`index.html`](index.html) — with no
build step, no dependencies, and no assets. The entire workflow is:

> **edit → refresh browser → commit → push**

Pushing to `main` **is** the deploy: GitHub Pages republishes
[oscillo.lekishamaharaj.com](https://oscillo.lekishamaharaj.com) about a minute
after every push.

## Where things live in `index.html`

| Section (banner comments) | What's there |
| --- | --- |
| `<style>` | All menu/HUD styling |
| utilities | Seeded RNG (`mulberry32`), value noise, music metadata |
| `AUDIO ENGINE` | `AudioEngine` class: lookahead scheduler, analyser features, voice toolkit (envelopes, kick, noise hits, procedural reverb IRs) |
| `AudioEngine.styles.*` | One generator per music style: `ambient`, `edm`, `orchestral`, `lofi` |
| `VISUALS` | `VisualEngine.modes.*` — one renderer per mood: `neon`, `playful`, `minimal`, `organic` |
| `APP` | Menu wiring, transport, keyboard, HUD, main rAF loop |

## Run locally

Open `index.html` in a browser — double-click it or `start index.html`. The app
makes zero network requests, so `file://` works with no server. (If you use VS
Code, the *Live Server* extension gives you auto-reload on save.)

Audio starts only after a user gesture (browser autoplay policy) — the
**Generate** button provides it.

## Testing tools built into the app

Open DevTools (F12) and use the `window.OSC` handle:

- **Jump to any combination without clicking:**
  `OSC.debug('edm', 'neon', 12345)`
  Styles: `ambient` `edm` `orchestral` `lofi` · moods: `neon` `playful` `minimal` `organic`
- **Seeds are reproducible.** The HUD shows the current seed; replay any piece
  exactly with `OSC.debug(style, mood, 0xTHESEED)`. If something looks or
  sounds broken, save the seed — it's a perfect bug report.
- **Press `D`** for a live fps meter. Keep it ≥ ~55; an adaptive-quality
  governor trims particle counts automatically on slow machines.
- **Audio health:** `OSC.audio.comp.reduction` in the console should sit
  between `0` and about `-12`. A huge negative number (hundreds) means a
  synthesis bug is blowing up the mix.
- Keyboard: `Space` play/pause · `R` regenerate · `F` fullscreen · `Esc` menu.

**Rule of thumb:** if you touch the audio engine, listen to all 4 styles; if
you touch the visuals, look at all 4 moods. Shared helpers mean one edit can
ripple across generators/renderers.

## Commit and deploy

```bash
git add index.html
git commit -m "Describe the change"
git push
```

`main` is the live site — only push what you've actually tried. Hard-refresh
the live site with `Ctrl+F5`; browsers cache aggressively.

For bigger or experimental features, branch first and merge when happy:

```bash
git checkout -b feature-name
# ...work, commit...
git checkout main
git merge feature-name
git push
```

## Adding a new music style or visual mood

The app is plug-in shaped:

- **New style:** add `AudioEngine.styles.yourstyle = function(E, r){ ... }`
  returning `{stepDur, step(t, i)}`; add an entry to `STYLE_META`; add one
  `<button class="card" data-style="yourstyle">` in the menu. Push timestamped
  events (`E.events.push({t, type:'kick'|'note'|'chord'|..., ...})`) so the
  visuals react to your notes.
- **New mood:** same pattern with `VisualEngine.modes`, `MOOD_META`, and a
  `data-mood` card. A mode factory gets `(V, rng, music)` and returns
  `{bg, frame(dt, features, events, t)}` — `features` carries normalized
  `bass/mid/treb/level`, `events` fires exactly when the audio clock reaches
  each scheduled note.

Everything else — menu, transport, seeding, sync — picks new entries up
automatically.

## Gotchas learned the hard way

- **Never delete the `CNAME` file.** Pages publishes from the `main` branch,
  and the custom domain lives in that file — removing it unbinds
  oscillo.lekishamaharaj.com.
- **Chromium `BiquadFilter` can go numerically unstable** (output explodes to
  the rails, then NaN-silence) when fed looped, mostly-near-silent
  impulse-train buffers — that's why the vinyl-crackle bed bakes its filtering
  into the buffer instead of using a live filter node. If a style suddenly
  screams or dies, check `OSC.audio.comp.reduction` first.
- **Buttons blur on click on purpose.** If a control kept focus, pressing
  `Space` would re-activate it instead of toggling playback.
- Keep the single-file constraint: no external scripts, fonts, audio, or
  images. Everything is synthesized — that's the point of the piece.
