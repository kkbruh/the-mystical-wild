# Design Review — The Mystical Wild

*Reviewed 2026-07-08 · branch `ui-improvemnts` · desktop (1440px) and mobile (390px) viewports rendered in headless Chrome.*

The site is a one-page wildlife photography portfolio (krishnaswild.com): a single `index.html` with inline CSS/JS, content driven by JS config objects (`CHAPTERS`, `PRINTS`, `PRICING`), hash-based views for Stories / Prints / About / Print-detail, deployed on GitHub Pages.

## Verdict

The visual design is genuinely strong — distinctive, cohesive, clearly art-directed rather than templated, and it holds up on mobile. The weaknesses are on the engineering side: repo cruft, a fragile dependency on an external CDN, image-delivery performance, and some duplication in the code.

---

## What works well

- **Art direction is the star.** Dark forest palette, italic Cormorant Garamond against letterspaced Inter caps, film-grain overlay, cinematic letterbox bars that retract on scroll, and a three-chapter narrative with per-chapter theme shifts (`theme-reign` cold blue for winter, `theme-herd` warm ochre for the elephants) cross-fading over 1.6s as you scroll. The theme-per-chapter idea is what makes the site memorable.
- **Craft details are above average for a hand-rolled page:** `prefers-reduced-motion` fully handled; passive scroll listeners and rAF-throttled parallax; IntersectionObserver reveals; print-detail tabs with correct ARIA roles and arrow-key navigation; `:focus-visible` styling; dependency-free `<details>` mobile menu.
- **Content-as-config is smart for the owner.** The `CHAPTERS` / `PRINTS` / `PRICING` objects mean the photographer can retitle, reorder, and reprice without touching markup. The pricing comments (edition strategy, FX rates, how to revive the retired open edition) read like good documentation.
- **The sales funnel is coherent:** story → "if a frame stayed with you" nudge → curated prints wall → per-print detail with sizes/materials and prefilled WhatsApp/email enquiries. Sensible for a no-backend, enquiry-based shop.
- **Image files are reasonably optimized** — WebP throughout, mostly 80–500 KB at ≤1920px.

---

## Mobile review (390px)

The responsive layout **holds up well** — no overflow, no clipping, no broken layouts:

- Hero title wraps to two lines cleanly; hamburger menu replaces the nav; hero section links stack vertically.
- Prints wall collapses to a clean single column with legible captions; the full-bleed prints hero works.
- About stacks photo-above-text correctly; print detail stacks image-above-options correctly.
- Scene stories drop their offset margins and read as a comfortable single column.

Mobile-specific improvements worth making:

1. **`100vh` → `100svh`/`100dvh`** (with `vh` fallback) for the hero and near-full-height sections. On iOS Safari/Chrome, `100vh` is taller than the visible viewport while the URL bar is shown, so the bottom of the hero — including the "Enter" link at `bottom:11vh` — can sit under browser chrome on first load. ✅ *Fixed 2026-07-08.* `.hero` (`index.html:118`), `.enter`'s `bottom` offset (`index.html:144`), `.scene-split` (`index.html:185`), and `footer` (`index.html:458`) now each declare the `vh` value first (fallback for browsers without `svh` support) followed by the equivalent `svh` value, which wins in supporting browsers via cascade — no JS, no @supports needed. Verified in headless Chrome (`puppeteer-core` against system Chrome) at 1440×900, 390×844, and 320×568: `.hero`'s computed height and `.enter`'s computed `bottom` now match `window.innerHeight` exactly (rather than the pre-fix `100vh`, which in a real mobile browser exceeds the visible area while the URL bar is shown), and `.enter`'s bounding rect stays fully inside `.hero`'s at every size tested, with zero console errors.
2. **Centered letterspaced text looks slightly off-center.** `letter-spacing` adds a trailing space to the last character; the `.enter` link compensates with `padding-left:.5em` but the `.eyebrow` (0.6em tracking) and `h1` don't. The eyebrow also wraps awkwardly at this width ("A PORTFOLIO IN THREE / CHAPTERS") — reduce its tracking or size on small screens.
3. **`.pd-size` price rows are overflow-prone at 320px.** ✅ *Fixed 2026-07-08.* The size label plus the `white-space:nowrap` price + FX conversions (`₹13,500 / $140 / £110`) fits at 390px but has little headroom on the smallest phones — at ≤359px the canvas tier's longer prices squeezed the size label into wrapping mid-word ("16 × 24" / "IN" on separate lines). **Fix applied:** at `max-width:360px` (`index.html:381-384`), `.pd-size em` gets `flex-wrap:wrap` and `.pd-size .fx` gets `flex:0 0 100%`, so the FX conversions drop to their own line under the price instead of competing with the size label for width; the size label itself is now wrapped in a `.nw` (nowrap) span (`index.html:918`) so it never breaks mid-word. Above 360px nothing changes. Verified in headless Chrome (`puppeteer-core` against system Chrome) on `#print-the-promise`: before the fix, canvas-tier rows at 320/340/359px wrapped their size label onto two lines; after the fix, all rows at those widths render the size label on one line and the price on one line, with FX conversions on a clean second line, while 375/390px and 1440px desktop are pixel-identical to before. Radio selection and the `#pd-selection`/CTA echo still update correctly after the change, with zero console errors.
4. **No responsive images.** Phones download the same ~200–500 KB, up-to-1920px WebPs as desktop. Fine on Wi-Fi, wasteful on cellular — `srcset` with a ~800px variant would roughly halve the transfer.
5. **The grain overlay animates infinitely** (6-step loop, 1.2s) even when idle. It's composited so it's cheap per frame, but it prevents the page from ever going fully idle — a small battery cost on mobile. Consider pausing it when the page is not visible, or dropping it on mobile.

### Confirmed on a real device (Android Chrome, krishnaswild.com) — FIXED 2026-07-08 on `ui-improvemnts`

Items 6 and 7 below were fixed and verified in headless Chrome at 390px (radio dots render, checked state fills accent, single hairline between material groups, selection echo updates on size change and feeds the CTAs, scene-mark segments no longer orphan, NBSP glue keeps em-dashes attached):

6. **Text wrapping reads badly on narrow screens.** ✅ *Fixed.*
   - The scene meta line (`.scene-mark`) — e.g. "SCENE 04 · THE HERD · MONSOON 2025" — wrapped with a single orphaned word ("2025") on its own line; the 0.45em tracking at 10px makes the line far wider than it looks. **Fix applied:** an `nw()` helper wraps each `·`-separated segment in `white-space:nowrap` spans (scene marks and print-card edition lines), so lines break only at the separators.
   - Story paragraphs (`.scene-text p`) kept near-double line spacing (1.85) at ~7 words per line, reading as disconnected strips, and em-dashes landed at line starts ("— the old leading"). **Fix applied:** mobile line-height tightened to 1.7, and a `glue()` helper inserts a no-break space before em-dashes (story paragraphs, print notes, material descriptions) so the dash stays attached to the previous word.
7. **The size picker on the print page didn't read as interactive** (`#print-the-*` view) — this is what registers as "tabs not working" on mobile. ✅ *Fixed.*
   - There was **no visible control affordance** — the checked state was only a subtle text-color shift, nearly invisible on a dark phone screen. **Fix applied:** each row now has a radio dot (`.pd-size > span::before`) that fills with the accent color when selected, with a `:focus-visible` ring for keyboard users.
   - **A stray double hairline** between the matte and canvas groups (last row's `border-bottom` + next group's `border-top`). **Fix applied:** `.pd-mat:not(:last-child) .pd-size:last-of-type{border-bottom:0}` — a single rule now separates the groups.
   - The selected size fed the WhatsApp/email enquiry text but nothing near the CTAs echoed it. **Fix applied:** a `#pd-selection` line above the buttons ("Selected — Hahnemühle Daguerre canvas · 12 × 18 in — ₹8,500") updates on every size change and tab switch, from the same value the CTAs use.

---

## Issues by priority

### 1. Four story images hotlink `cdn.myportfolio.com`

`index.html:652, 665, 671, 720` — "The Watchful", both Chapter I triptych frames, and "The Quiet Rival" load from Adobe Portfolio's CDN with cache-busting query params. If that account lapses or the URLs rotate, those scenes silently break. Local WebP files exist for everything else; bring these four in-repo. ✅ *Fixed 2026-07-08.* Downloaded the four full-res JPEGs from the CDN, re-encoded to WebP (`cwebp -q 82`, matching the site's existing quality/size range) as `the-watchful.webp`, `the-cautious.webp`, `the-sentinel.webp`, and `the-quiet-rival.webp`, and swapped the four `img:` values in the `CHAPTERS` config from CDN URLs to these local filenames. No JPG duplicate was added, since the review already flags the existing JPG-but-unused-duplicates (§3) as dead weight to remove, not a pattern to extend. Verified in headless Chrome (`puppeteer-core` against system Chrome) serving the repo over a local static server: all four scenes render their images locally (`naturalWidth: 1920`, `complete: true`), zero network requests to `myportfolio.com` are made, no `cdn.myportfolio.com` string remains anywhere in the rendered DOM, and there are no new console errors (the page's one pre-existing `favicon.ico` 404 — tracked separately under "Missing basics" — is unrelated and unchanged by this fix).

### 2. Image delivery is unoptimized where it matters most

- The hero (`the-question-v3.webp`) is a **CSS background**: it can't be `preload`ed responsively or get `srcset`, and it fades in only after a 0.9s animation delay on top of the fetch. For a photography site, hero LCP is the one metric worth engineering. Add `<link rel="preload" as="image">` at minimum.
- **No image has `width`/`height` or `srcset`** — layout shift in the prints grid, full-res downloads on mobile.
- Full-bleed story plates are backgrounds too, so they can't lazy-load (the prints-grid `<img loading="lazy">` is already right).

### 3. Repo hygiene

✅ *Fixed 2026-07-08.* All four bullets below addressed in one cleanup commit:

- **`the-mystical-wild-index.html` is a stale near-copy of `index.html`** (1025 vs 1046 lines) — a drift hazard: one edit to the wrong file and the live site diverges. Delete it. **Fix applied:** deleted. It still pointed at the old `kkbruh.github.io` Pages URL in its `og:url`/`og:image`/`twitter:image` meta tags and predated the favicon, `100svh`, and theme fixes already merged into `index.html` — confirming it was drifting, not a maintained mirror.
- **Working files committed:** `polish.diff`, `reign images 2.zip`, raw `_DSC4490.JPG` (2.8 MB). The zip and raw bloat every clone and are publicly downloadable on Pages. **Fix applied:** all three deleted.
- **Every image ships as both `.jpg` and `.webp`** but the HTML references only `.webp` (plus one `.jpg` for `og:image`). The rest of the JPGs (~500–700 KB each) are dead weight. **Fix applied:** deleted the 22 `.jpg` files whose `.webp` counterpart is what `index.html` actually references, plus 4 superseded `.webp` files (`the-ambush-v2.webp`, `the-dominant.webp`, `the-dominant-v2.webp`, `the-question-v2.webp` — each has a `-v3` file that's the one in use) and `old-gold.jpg`/`old-gold.webp`, which weren't referenced by any filename in `index.html` at all. `print-the-promise.jpg` was kept — it's the live `og:image` target, and some social-media crawlers don't render WebP link previews.
- **README is a stub; every commit is "Add files via upload"** — no usable history. Even one-line commit messages ("localize CDN images") make the repo maintainable. **Fix applied:** replaced the one-line `README.md` with real documentation (site structure, the `CHAPTERS`/`PRINTS`/`PRICING` config pattern, local dev, deployment). The commit-message half of this is already improving — recent commits on `ui-improvemnts` use real descriptive messages instead of "Add files via upload".

Verified in headless Chrome (`puppeteer-core` against system Chrome) serving the repo over a local static server: zero console errors, zero failed or 4xx/5xx requests, and every `<img>` element across the stories/prints/about/print-detail views loads successfully (`naturalWidth > 0`) once its view is navigated to — the only images reporting as not-yet-loaded on first paint were the lazy-loaded prints-grid thumbnails outside the initial viewport, which resolved as soon as `#prints` was visited.

### 4. Missing basics

- **No favicon / apple-touch-icon** — a noticeable gap for a brand this polished (also affects home-screen bookmarks on mobile). ✅ *Fixed 2026-07-08.* Added `favicon.svg` (an italic serif "K" monogram, moss-on-forest, matching the header's typeface style and palette), `favicon.png` (512×512 raster fallback for browsers without SVG-favicon support), and `apple-touch-icon.png` (180×180, opaque square per Apple's spec — no baked-in rounding/transparency, since iOS applies its own mask). Wired up via three `<link>` tags in `index.html`'s `<head>` (`index.html:12-14`). **Correction:** the first version embedded the actual Cormorant Garamond webfont in the SVG as a base64 `@font-face`, which rendered correctly in an `<img>`/headless-Chrome screenshot but showed as a blank dark square in a real browser tab — tab-icon rasterization appears not to execute custom `@font-face` in the favicon's own document context. Switched to a system serif (`Georgia,'Times New Roman',serif`, italic) with no font-loading dependency, confirmed against a real device screenshot showing only a plain square where the glyph should be. Verified in headless Chrome: all three assets resolve `200` with correct MIME types (`image/svg+xml`, `image/png`, `image/png`) served from a local static server, the `<link>` tags appear correctly in the rendered DOM, and the monogram is legible at 180px down to 16px, with zero console errors.
- No `404.html` (hash routing mostly avoids it, but any bad path gets the GitHub default).
- No structured data — `Product` / `VisualArtwork` JSON-LD on the prints would be a cheap SEO win.

### 5. All content is client-rendered

The chapters and prints grid exist only as JS objects injected into an empty `<main>` / `.prints-grid`. Google executes JS, but other crawlers, some link previews, and no-JS visitors get an empty page. For a single static page this is fixable by pre-rendering the HTML and keeping the config for interactivity only — worth doing if search traffic for prints ever matters.

### 6. Smaller code and design nits

- **Duplicate deterrent code:** the contextmenu/drag blockers are registered twice (`index.html:879–882` and `1042–1043`). One block should go. ✅ *Fixed 2026-07-08.* Merged into the single block near the top (`index.html:887-892`), keeping the `.prints-hero` selector and `dragstart` handler that only existed in the duplicate. Verified in headless Chrome (`puppeteer-core` against system Chrome): dispatching `contextmenu`/`dragstart` on the live page now triggers exactly one `preventDefault()` call per event type, with no console errors.
- **Authoring text can leak to visitors:** the empty-scene placeholder renders "Add the image URL in the CHAPTERS config — the scene is ready for it." (`index.html:845`). All scenes currently have images so it never shows, but the hint should be dev-facing only.
- **`.scene:nth-child(odd/even)` is fragile.** ✅ *Fixed 2026-07-08.* It counted *all* siblings inside the chapter, so the `.ch-head` and `.triptych` `<div>`s shifted the parity — e.g. in Chapter I, "The Dominant" (1st scene) and "The Question" (3rd scene) rendered on *opposite* sides instead of matching. **Fix applied:** `.scene`/`.scene-split` alternation (`index.html:186-187, 203-207, 490-491`) now uses `:nth-of-type` instead of `:nth-child` — since every scene is an `<article>` and `.ch-head`/`.triptych` are `<div>`s, `:nth-of-type` counts only scene siblings and ignores the interleaved divs, so alternation is now deterministic by scene order regardless of triptychs or reordering. Verified in headless Chrome: comparing computed `margin-left`/`margin-right` (and `grid-template-columns` for split scenes) before/after across all three chapters — before, scene 1 and scene 3 in Chapter I had mismatched patterns; after, every odd-positioned scene matches and every even-positioned scene matches, with zero console errors.
- **Contrast:** the faintest text tier (`--text-faint`, 0.32 alpha) at 9–10px letterspaced caps (the `Fin` line, some terms text) is below WCAG AA. The 0.55 "dim" tier is fine; keep the faint tier decorative-only. ✅ *Fixed 2026-07-08.* `footer .fin` (`index.html:466`) was hardcoded to `rgba(232,228,216,.3)` — not even wired to the theme system, so it stayed the "reign" mist color under the herd/fall themes too. **Fix applied:** switched it to `var(--mist-dim)` (0.55 alpha), which is theme-aware and already confirmed fine by this review. The other faint-tier uses (`.empty-glyph`, `.empty-label`, `.empty-hint`) are the dev-facing empty-scene placeholder that never renders live (all scenes have images), so they were left as decorative-tier per the recommendation above. Verified in headless Chrome: computed contrast ratio of `footer .fin` against `--bg` in all three themes was ~2.3:1 (fail) before the fix and ~5.1:1 (pass WCAG AA) after, across `theme-fall`/`theme-reign`/`theme-herd`, with zero console errors.
- **Artwork invisible to screen readers in classic scenes:** full-bleed plates are CSS backgrounds with no accessible name, while split/portrait scenes use real `<img alt>`. Real `<img>` everywhere fixes accessibility *and* enables lazy-loading (see §2).
- **Hash-view switches don't manage focus.** ✅ *Fixed 2026-07-08.* A keyboard or screen-reader user who activated "Recommended Prints" was left focused inside a now-`display:none` section with no announcement. **Fix applied:** `.prints-title`, `.about-title`, and `.pd-title` got `tabindex="-1"` plus a moss-colored `:focus` outline; `applyRoute()` now focuses the relevant heading whenever the hash actually changes (tracked via `lastRouteHash`, so navigating print → print re-focuses the new print's title even though the `data-view` stays `"print"`). The `stories` view is unchanged (no single heading to focus; existing scroll behavior stands). Verified in headless Chrome: dispatching real navigation through prints/about/print-detail/print-detail/stories moves `document.activeElement` to the correct `<h2>` each time (with the right text), with zero console errors.
  - **Regression: the outline showed for mouse users too.** ✅ *Fixed 2026-07-08.* Since the heading is focused programmatically on every route change, the `:focus` selector painted a persistent-looking box around "The ones I would hang first." (etc.) even for plain mouse-click navigation, not just keyboard users. **Fix applied:** `index.html:312` now uses `:focus-visible` instead of `:focus`, so the ring only renders when the browser's own heuristic says the focus came from keyboard interaction. Verified in headless Chrome: simulated mouse-click nav to `#prints` now computes `outline-style: none` on `.prints-title`, while Tab+Enter keyboard nav to the same link still computes `outline-style: solid` (`:focus-visible` matches), with zero console errors.
- **Anti-save measures** (blocked right-click/drag, `user-select:none`) are trivially bypassed via devtools and break legitimate actions like mobile long-press sharing. Since only screen-res files are served anyway (per the code comment), consider whether the UX cost buys anything.

---

## Print-detail tabs: spec verification (change 5 of the polish list)

The "tabbed print-detail options" refactor (two tabs sharing one enquire-button row, replacing two stacked option groups with four buttons) is **already implemented** in the current `index.html`. The spec describes an older version of the page — the pre-refactor markup it references (`.pd-open`, `.pd-open-actions`, `#pd-open-wa`, `#pd-open-mail`, A3/A2 radios) no longer exists anywhere in the repo. `polish.diff` is a stale diff from that same era.

Verified functionally in headless Chrome (with `OPEN_EDITION` enabled in a scratch copy, loading `#print-the-promise` and driving clicks/keys):

| Spec requirement | Result |
|---|---|
| Single shared CTA pair (`#pd-wa` / `#pd-mail`), no per-panel buttons | ✅ exactly 2 CTAs; no `#pd-open-wa`/`#pd-open-mail` |
| Default active tab = Signed edition | ✅ `aria-selected` correct; open panel `hidden` |
| Hidden panels use the `hidden` attribute, not display classes | ✅ |
| CTAs rebuild on size-radio change | ✅ e.g. `…"The Promise", Hahnemühle Daguerre canvas · 24 × 36 in — ₹26,500` |
| CTAs rebuild on tab switch | ✅ open tab → `…the open framed print of "The Promise"` / subject `Open print enquiry — The Promise` |
| `role=tablist/tab/tabpanel`, `aria-selected`, `aria-controls` wiring | ✅ (`index.html:913-917`) |
| Roving tabindex + Left/Right arrow switching | ✅ ArrowLeft from Open → Signed selected, tabindex `0/-1` (`index.html:963-981`) |
| Tab styling: sans caps, accent + 1px accent underline for active, `--text-dim` inactive, `.collect` CTAs, no new colors | ✅ (`index.html:364-373`) |
| Hash routing unaffected | ✅ |

Notes:

- **The tabs are dormant in production**: `OPEN_EDITION = null` (`index.html:614`, "open framed print retired Jul 2026"), so live visitors see only the signed panel and one CTA pair. Restoring the commented config value (`{ price: 2999, spec: 'A4 · framed · matte' }`) brings the full tab UI back, and it works.
- Consequently, the "tabs not working well on mobile" report cannot be about these tabs — they don't render on the live site. What reads as broken tabs on the print page is the **size-picker radio rows** (§7 in the mobile findings above): invisible selected state, stray double hairline, and no selection echo near the CTAs.

---

## Suggested order of attack

1. ~~Delete the stale HTML copy, zip, diff, and unused JPGs; write a real README.~~ **Done 2026-07-08** (see repo hygiene §3).
2. ~~Localize the four portfolio-CDN images.~~ **Done 2026-07-08** (see issue §1 above).
3. ~~Add a favicon~~ **(done 2026-07-08, see "Missing basics" §4)**, `width`/`height` on images, hero preload, and ~~`100svh` for the hero~~ **(svh done 2026-07-08, see mobile findings §1)**.
4. ~~Fix the real-device mobile findings: scene-mark orphan wrapping, mobile line-height, and the size-picker affordance/double-hairline on the print page.~~ **Done 2026-07-08** (see mobile findings §6–7).
5. ~~Fold in the small code fixes: duplicate listeners, focus on view switch, `nth-child` alternation, faint-text contrast.~~ **Done 2026-07-08** (all four items).
6. (Later, if worth it) `srcset` for mobile, pre-rendered HTML, JSON-LD for prints.
