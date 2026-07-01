# WC26 — Circular World Cup 2026 Bracket

A **static, single-page** web widget that renders the 2026 World Cup knockout
bracket as **concentric rings** (Round of 32 on the outside → Final at the
center, trophy in the middle) and **updates itself in the browser** as matches
finish. Every team is a **circular flag icon**.

→ Open `index.html` — it's a single file, host it on GitHub Pages.

## How it works

GitHub Pages only serves files — it never runs code. All the "live" behavior
happens in the visitor's browser:

```
open page → browser runs the inline JS → fetch() the results feed
          → build + draw the bracket → re-poll every 2 min
```

No backend, no build step, no API key.

### Pipeline (all in `index.html`)

```
fetchResults()   ONLY source-specific code — returns the raw match array
  ↓
normalize()      → map of match num → match
  ↓
buildModel()     static skeleton + results overlay; winners propagate inward
  ↓
layout()         polar (radius, angle) per node; inner angle = mean of children
  ↓
render()         SVG lines/dots/trophy + circular flag <img>s, diff-animated
```

### Key ideas

- **Static skeleton, results overlay.** The bracket shape is fixed and known in
  advance, so the 31-match tree (`CHILDREN` map) is hard-coded once. The feed
  only supplies *results*; winners propagate inward automatically and any
  unresolved slot shows a pending dot (TBD).
- **Flags fill inward.** All 32 teams sit on the outer ring. When a match
  resolves, the winner's flag appears one ring closer to the center (gold
  border); the loser greys out. Over the ~3 weeks of knockouts the flags march
  toward the trophy.
- **Geometry.** The 16 leaf matches are placed at evenly spaced angles; every
  inner match sits at the **mean of its two children's angles**, computed
  bottom-up, so connectors stay clean and converge symmetrically on the center.
  A DFS of the tree reproduces the reference layout exactly.

## Data source

[`openfootball/worldcup.json`](https://github.com/openfootball/worldcup.json)
— public domain, **no API key**, **CORS-enabled** (`access-control-allow-origin: *`,
verified), ~36 KB:

```
https://raw.githubusercontent.com/openfootball/worldcup.json/master/2026/worldcup.json
```

It's a git repo, not a live wire, so results lag minutes-to-hours behind the
final whistle — fine for a bracket that fills in over weeks. The feed is
isolated behind `fetchResults()` + `normalize()` (the adapter boundary), so a
true live feed is a one-function swap.

**Winner rule:** check penalties (`p`) → extra time (`et`) → full time (`ft`),
in that order.

## Flags

No image files are shipped. Each country maps to a 2-letter ISO code and the
flag URL is built on the fly via [flagcdn.com](https://flagcdn.com):

```
https://flagcdn.com/w160/de.png   // Germany
```

Circular icons = the square image clipped with CSS `border-radius:50%`. The only
hand-authored data is the team-name → ISO table (`ISO` in `index.html`); it must
match the exact strings the feed uses (e.g. `Bosnia & Herzegovina`, `England` →
`gb-eng`).

## Update / failure behavior

- Polls every **120 s**; diffs against the last results so only newly-resolved
  matches animate (a pop-in), no full-bracket flicker.
- On fetch error the **last-good bracket stays on screen** with a quiet
  "offline, retrying" stamp — it never blanks.

## Develop

It's one self-contained file. Open `index.html` in a browser. To preview a
specific tournament state offline, mock the feed response.
