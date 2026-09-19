# html-games

A collection of tiny browser games, shipped as one installable PWA on GitHub Pages.

## The core idea

**These games must run well on an old, small iPhone — the iPhone 5S is the reference device.**

That is the whole point of the project, and it drives every technical decision below. A game
that only feels good on a modern phone or a desktop browser has missed the target. When in
doubt, optimise for the 5S and let newer devices get the same thing, scaled up.

What "the reference device" means concretely:

| Property | iPhone 5S |
| --- | --- |
| Viewport | 320 x 568 CSS px (4-inch) |
| Max iOS | 12.5.7 |
| Browser engine | Safari 12 |
| CPU | A7 — assume slow JS and slow layout |

Secondary goals, in order: works offline, installs to the home screen, loads instantly,
stays trivially simple to add a game to.

## Non-negotiable constraints

1. **No dependencies, no build step, no framework.** No npm, no bundler, no transpiler.
   The GitHub Actions workflow copies files and runs one `sed`. Keep it that way.
2. **One game = one self-contained `.html` file** in `src/`, with its `<style>` and
   `<script>` inline. The only shared script is `pwa.js`. This is intentional: it keeps each
   game readable end-to-end and means adding a game touches almost nothing else.
3. **Everything must work offline** once cached by the service worker.
4. **Touch first.** Always set `touch-action: manipulation` and a transparent
   `-webkit-tap-highlight-color` on the page, and make tap targets finger-sized.
   For latency-sensitive controls, add a `touchstart` handler with `preventDefault()`
   alongside `click` to dodge the ghost-click and Safari's 300 ms tap delay — this is what
   `tictac.html` does for its cells and New Game button. `sudoku.html` currently relies on
   `click` alone; that is an inconsistency, not a decision, so prefer the `tictac.html`
   pattern for new work.
5. **Respect the notch and the home indicator.** Page padding uses
   `env(safe-area-inset-*, 0px)` with a `calc()` base, and the viewport meta carries
   `viewport-fit=cover`.
6. **No horizontal scrolling and no pinch-zoom.** The viewport meta sets
   `maximum-scale=1.0, user-scalable=no`. Boards must fit 320 px wide.

## Safari 12 compatibility — what you may and may not use

Verified against the current code, which sticks to this line.

**Safe to use (already used here):** `const`/`let`, arrow functions, template literals,
`Array.prototype.map/slice/push`, CSS Grid, Flexbox, CSS custom properties (`var(--x)`),
`calc()`, `env()`, `@keyframes`, Service Workers, the Cache API.

**Do NOT use** — unsupported on Safari 12, will hard-fail:

- Optional chaining `?.` and nullish coalescing `??` (Safari 13.1+)
- `:has()` (15.4), `aspect-ratio` (15), `dvh` / `svh` units (15.4)
- `ResizeObserver` (13.1), `Promise.allSettled` (13), `String.matchAll` (13)
- `structuredClone`, `BigInt`, top-level `await`, ES modules via `type="module"` in
  practice — stick to classic scripts

**Known caveat, unresolved:** `gap` inside a `display: flex` container only landed in
Safari 14.1. Several places use it (`sudoku.html` mode-select / controls / numpad,
`tictac.html` toolbar rows), so on a real 5S those gaps collapse to zero. Grid `gap` is
fine (Safari 12 supports it). If you touch that CSS, prefer margins on the children, or
Grid, over flex `gap`. This is from browser support data, not from testing on hardware.

## Sizing approach

Two patterns are in use; both are acceptable, prefer the first for anything grid-shaped:

- **Measure and size in JS** (`sudoku.html` → `sizeBoard()`): read
  `document.documentElement.clientWidth`, subtract padding and gaps, clamp the cell size,
  then set `gridTemplateColumns` and a `--cell-font` custom property in pixels. This is the
  robust option and avoids needing media queries at all.
- **`minmax()` tracks with `width: fit-content`** (`tictac.html` → `#board`): lets the grid
  shrink to the 320 px viewport on its own.

There are deliberately **no `@media` queries** anywhere in the project.

## Layout of the repo

```
src/
  index.html      Menu — a card grid linking to each game
  sudoku.html     Sudoku, 4x4 and 6x6, procedurally generated + on-screen numpad
  tictac.html     Tic Tac Toe, two players on one device, scoreboard + confetti
  arkanoid.html   Brick breaker on a 2D canvas, one-finger paddle, endless levels
  snake.html      Snake on a 2D canvas, swipe to steer, speeds up with every bite
  pwa.js          Service-worker registration and auto-reload on update
  sw.js           Cache-first service worker with a build-stamped cache name
  manifest.json   PWA manifest (standalone, theme #4A90E2)
  icon-192.png  icon-512.png  apple-touch-icon.png
.github/workflows/static.yml   Build and deploy to GitHub Pages
```

## Deploy and the update mechanism

Push to `main` → the workflow copies `src/*` into `build/`, replaces `__BUILD_ID__` in
`sw.js` with the 7-character commit SHA, and publishes to Pages.

That SHA becomes `CACHE_NAME`, so every deploy is a fresh cache. The update path is:
`install` precaches and calls `skipWaiting()` → `activate` deletes every other cache and
calls `clients.claim()` → `pwa.js` hears `controllerchange` and reloads the page once. It
also calls `registration.update()` whenever the tab becomes visible, which is what makes an
installed home-screen app pick up new versions without being reinstalled.

**Consequence for you:** the cache is keyed on the commit, not on file contents. Any change
that ships invalidates everything, and there is nothing to bump by hand.

## Adding a new game — checklist

1. Create `src/<game>.html`, self-contained, following the conventions above.
2. Add it to `ASSETS_TO_CACHE` in `src/sw.js`. **Easy to forget; the game will not work
   offline without it.**
3. Add a `<a class="game-card">` entry to the grid in `src/index.html`.
4. Include `<script src="pwa.js"></script>` before `</body>`.
5. Check it at 320 x 568 with the device toolbar before pushing.

## Testing

There is no test suite, no linter, and no dev server script. To run locally, serve `src/`
over HTTP (service workers need an origin — `file://` will not do), e.g.
`python3 -m http.server -d src 8000`. Note that `sw.js` will contain the literal
`__BUILD_ID__` locally, since substitution only happens in CI.

Verification is manual: a 320 px-wide viewport in the browser device toolbar at minimum,
and real hardware when it matters.

## Style

Plain, old-school JavaScript — `function` declarations, IIFEs to capture loop variables,
direct `document.getElementById` calls, no abstraction layers. Match it. The code is meant
to be readable top-to-bottom by someone who does not know the project.
