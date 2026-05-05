# Hero Parallax Immersion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add mouse-tracking parallax depth and a page-load reveal to all four game pages so heroes feel like portals into living worlds, not static backdrops.

**Architecture:** Each page gets a self-contained JS IIFE (no new files, no dependencies) that: (1) animates the hero background from scale 1.12→1.06 on load, (2) runs a rAF loop with lerp-damped mouse tracking moving the background opposite to the cursor (±12px) and the content with the cursor (±5px), and (3) fades the hero content in from 12px below. The homepage additionally hooks into the existing carousel `goTo()` via a shared `fpgOnSlideChange` variable to reset parallax state on each slide change.

**Tech Stack:** Vanilla JS (`requestAnimationFrame`, `mousemove`, `matchMedia`), CSS `transform` + `will-change`, no new dependencies.

---

## File Map

| File | What changes |
|---|---|
| `fow.html` | `will-change: transform` on `.game-hero-bg` + `.game-hero-content`; parallax IIFE added to `<script>` |
| `expansion.html` | Identical to `fow.html` (same class names) |
| `dispelled.html` | Identical to `fow.html` (same class names) |
| `index.html` | `will-change: transform` on `.hero-bg` + `.hero-content`; `fpgOnSlideChange` hook declared + wired into `goTo()`; carousel parallax IIFE added |

> **Note:** The spec says the rAF loop should run only when the hero is in viewport. This plan keeps the loop running unconditionally — it is extremely cheap (pure math + one style set per frame) and the hero is the first thing visible on every page. An IntersectionObserver can be added later if profiling shows a need.

---

## Task 1: fow.html — Parallax CSS + script

**Files:**
- Modify: `fow.html`

- [ ] **Step 1: Add `will-change: transform` to `.game-hero-bg` in the CSS (around line 51)**

Find:
```css
.game-hero-bg {
  position: absolute; inset: 0; width: 100%; height: 100%;
  object-fit: cover; object-position: center 15%;
}
```
Change to:
```css
.game-hero-bg {
  position: absolute; inset: 0; width: 100%; height: 100%;
  object-fit: cover; object-position: center 15%;
  will-change: transform;
}
```

- [ ] **Step 2: Add `will-change: transform` to `.game-hero-content` (around line 63)**

Find:
```css
.game-hero-content {
  position: relative; z-index: 2;
  max-width: 1440px; width: 100%; margin: 0 auto; padding: 0 48px 88px;
}
```
Change to:
```css
.game-hero-content {
  position: relative; z-index: 2;
  max-width: 1440px; width: 100%; margin: 0 auto; padding: 0 48px 88px;
  will-change: transform;
}
```

- [ ] **Step 3: Replace the existing `<script>` block at the bottom of fow.html (around line 406)**

Find:
```html
  <script>
    const header = document.getElementById('site-header');
    window.addEventListener('scroll', () => { header.classList.toggle('scrolled', window.scrollY > 12); }, { passive: true });
  </script>
```

Replace with:
```html
  <script>
    const header = document.getElementById('site-header');
    window.addEventListener('scroll', () => { header.classList.toggle('scrolled', window.scrollY > 12); }, { passive: true });

    (function() {
      if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;
      const bgEl = document.querySelector('.game-hero-bg');
      const contentEl = document.querySelector('.game-hero-content');
      if (!bgEl || !contentEl) return;

      const BG_RANGE = 12, FG_RANGE = 5, LERP_FACTOR = 0.08;
      const REVEAL_MS = 1400, START_SCALE = 1.12, END_SCALE = 1.06;
      const isTouchOnly = window.matchMedia('(hover: none)').matches;
      let bgTX = 0, bgTY = 0, bgCX = 0, bgCY = 0;
      let fgTX = 0, fgTY = 0, fgCX = 0, fgCY = 0;
      let t0 = null;

      bgEl.style.transform = 'translate(0px,0px) scale(1.12)';

      contentEl.style.opacity = '0';
      contentEl.style.transform = 'translateY(12px)';
      contentEl.style.transition = 'opacity 0.9s ease 0.15s, transform 0.9s cubic-bezier(0.16,1,0.3,1) 0.15s';
      requestAnimationFrame(() => requestAnimationFrame(() => {
        contentEl.style.opacity = '1';
        contentEl.style.transform = 'translateY(0px)';
      }));
      setTimeout(() => { contentEl.style.transition = ''; }, 1400);

      if (!isTouchOnly) {
        document.addEventListener('mousemove', function(e) {
          const nx = e.clientX / window.innerWidth - 0.5;
          const ny = e.clientY / window.innerHeight - 0.5;
          bgTX = nx * -BG_RANGE; bgTY = ny * -BG_RANGE;
          fgTX = nx * FG_RANGE;  fgTY = ny * FG_RANGE;
        }, { passive: true });
      }

      function lerp(a, b, t) { return a + (b - a) * t; }

      function tick(ts) {
        if (!t0) t0 = ts;
        const elapsed = ts - t0;
        const progress = Math.min(elapsed / REVEAL_MS, 1);
        const ease = 1 - Math.pow(1 - progress, 3);
        const scale = START_SCALE - (START_SCALE - END_SCALE) * ease;

        if (!isTouchOnly) {
          bgCX = lerp(bgCX, bgTX, LERP_FACTOR); bgCY = lerp(bgCY, bgTY, LERP_FACTOR);
          fgCX = lerp(fgCX, fgTX, LERP_FACTOR); fgCY = lerp(fgCY, fgTY, LERP_FACTOR);
        }

        bgEl.style.transform = `translate(${bgCX.toFixed(2)}px,${bgCY.toFixed(2)}px) scale(${scale.toFixed(4)})`;

        if (!isTouchOnly && elapsed > REVEAL_MS) {
          contentEl.style.transform = `translate(${fgCX.toFixed(2)}px,${fgCY.toFixed(2)}px)`;
        }

        requestAnimationFrame(tick);
      }

      requestAnimationFrame(tick);
    })();
  </script>
```

- [ ] **Step 4: Visual verify `http://localhost:3000/fow.html`**

Start server if not running: `python3 -m http.server 3000 --directory /Users/terrangreenfield/fartofwar.com &`

Check all three behaviors:
1. **Load reveal:** Hard refresh (Cmd+Shift+R). The battlefield image starts slightly zoomed in and smoothly pulls back over ~1.4s. "FART / OF WAR" and the buttons fade in from ~12px below their final position.
2. **Parallax:** Move cursor left → background drifts right, title drifts left. Move cursor right → background drifts left, title drifts right. Motion is smooth/inertial (lerp), not instant.
3. **No edge bleed:** The hero background image never reveals a white/dark gap at any edge during parallax travel.

Take a screenshot: `screencapture -x -l $(osascript -e 'tell application "Safari" to return id of window 1') /Users/terrangreenfield/fartofwar.com/screenshots/parallax_fow.png`

- [ ] **Step 5: Commit**

```bash
git -C /Users/terrangreenfield/fartofwar.com add fow.html
git -C /Users/terrangreenfield/fartofwar.com commit -m "fow: add parallax depth + load reveal to hero"
```

---

## Task 2: expansion.html — Apply identical parallax

**Files:**
- Modify: `expansion.html`

- [ ] **Step 1: Add `will-change: transform` to `.game-hero-bg` (around line 33)**

Find:
```css
.game-hero-bg { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; object-position: center 15%; }
```
Change to:
```css
.game-hero-bg { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; object-position: center 15%; will-change: transform; }
```

- [ ] **Step 2: Add `will-change: transform` to `.game-hero-content` (around line 36)**

Find:
```css
.game-hero-content { position: relative; z-index: 2; max-width: 1440px; width: 100%; margin: 0 auto; padding: 0 48px 88px; }
```
Change to:
```css
.game-hero-content { position: relative; z-index: 2; max-width: 1440px; width: 100%; margin: 0 auto; padding: 0 48px 88px; will-change: transform; }
```

- [ ] **Step 3: Replace the `<script>` block at the bottom of expansion.html (around line 301)**

Find:
```html
  <script>
    const header = document.getElementById('site-header');
    window.addEventListener('scroll', () => { header.classList.toggle('scrolled', window.scrollY > 12); }, { passive: true });
  </script>
```
Replace with the exact same script block from Task 1 Step 3 (class names `.game-hero-bg` and `.game-hero-content` are identical).

- [ ] **Step 4: Visual verify `http://localhost:3000/expansion.html`**

Same three checks as Task 1 Step 4. Take a screenshot to `screenshots/parallax_expansion.png`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/terrangreenfield/fartofwar.com add expansion.html
git -C /Users/terrangreenfield/fartofwar.com commit -m "expansion: add parallax depth + load reveal to hero"
```

---

## Task 3: dispelled.html — Apply identical parallax

**Files:**
- Modify: `dispelled.html`

- [ ] **Step 1: Add `will-change: transform` to `.game-hero-bg` (around line 33)**

Find:
```css
.game-hero-bg { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; object-position: center top; }
```
Change to:
```css
.game-hero-bg { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; object-position: center top; will-change: transform; }
```

- [ ] **Step 2: Add `will-change: transform` to `.game-hero-content` (around line 36)**

Find:
```css
.game-hero-content { position: relative; z-index: 2; max-width: 1440px; width: 100%; margin: 0 auto; padding: 0 48px 88px; }
```
Change to:
```css
.game-hero-content { position: relative; z-index: 2; max-width: 1440px; width: 100%; margin: 0 auto; padding: 0 48px 88px; will-change: transform; }
```

- [ ] **Step 3: Replace the `<script>` block at the bottom of dispelled.html (around line 255)**

Find:
```html
  <script>
    const header = document.getElementById('site-header');
    window.addEventListener('scroll', () => { header.classList.toggle('scrolled', window.scrollY > 12); }, { passive: true });
  </script>
```
Replace with the exact same script block from Task 1 Step 3.

- [ ] **Step 4: Visual verify `http://localhost:3000/dispelled.html`**

Same three checks as Task 1 Step 4. Take a screenshot to `screenshots/parallax_dispelled.png`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/terrangreenfield/fartofwar.com add dispelled.html
git -C /Users/terrangreenfield/fartofwar.com commit -m "dispelled: add parallax depth + load reveal to hero"
```

---

## Task 4: index.html — Carousel parallax + reveal

This is the most complex task. The homepage cycles four slides via an existing carousel IIFE. The parallax code hooks in via a `fpgOnSlideChange` variable.

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add `will-change: transform` to `.hero-bg` CSS rule (around line 224)**

Find:
```css
    .hero-bg {
      position: absolute;
      inset: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
      object-position: center 15%;
    }
```
Change to:
```css
    .hero-bg {
      position: absolute;
      inset: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
      object-position: center 15%;
      will-change: transform;
    }
```

- [ ] **Step 2: Add `will-change: transform` to `.hero-content` CSS rule (around line 255)**

Find:
```css
    .hero-content {
      position: relative;
      z-index: 2;
      max-width: 1440px;
      width: 100%;
      margin: 0 auto;
      padding: 0 48px 88px;
    }
```
Change to:
```css
    .hero-content {
      position: relative;
      z-index: 2;
      max-width: 1440px;
      width: 100%;
      margin: 0 auto;
      padding: 0 48px 88px;
      will-change: transform;
    }
```

- [ ] **Step 3: Declare `fpgOnSlideChange` at the top of the `<script>` block (around line 1537)**

Find the opening of the script block:
```html
  <script>
    // Close announcement bar
```
Change to:
```html
  <script>
    var fpgOnSlideChange;

    // Close announcement bar
```

- [ ] **Step 4: Add the `fpgOnSlideChange` callback at the end of `goTo()` inside the carousel IIFE (around line 1620)**

Find:
```js
      function goTo(n) {
        slides[current].classList.remove('active');
        dots[current].classList.remove('active');
        dots[current].setAttribute('aria-selected', 'false');
        current = (n + slides.length) % slides.length;
        slides[current].classList.add('active');
        dots[current].classList.add('active');
        dots[current].setAttribute('aria-selected', 'true');
        applyTheme(current);
        updateStats(current);
      }
```
Change to:
```js
      function goTo(n) {
        slides[current].classList.remove('active');
        dots[current].classList.remove('active');
        dots[current].setAttribute('aria-selected', 'false');
        current = (n + slides.length) % slides.length;
        slides[current].classList.add('active');
        dots[current].classList.add('active');
        dots[current].setAttribute('aria-selected', 'true');
        applyTheme(current);
        updateStats(current);
        if (typeof fpgOnSlideChange === 'function') fpgOnSlideChange(current);
      }
```

- [ ] **Step 5: Add the homepage parallax IIFE at the end of the `<script>` block, before `</script>`**

After the closing `})();` of the email popup IIFE, add:

```js
    // ── Hero parallax + reveal ─────────────────────────────
    (function() {
      if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;

      const slideEls   = document.querySelectorAll('.hero-slide');
      const bgEls      = Array.from(slideEls).map(s => s.querySelector('.hero-bg'));
      const contentEls = Array.from(slideEls).map(s => s.querySelector('.hero-content'));
      if (!bgEls.length || !bgEls[0]) return;

      const BG_RANGE = 12, FG_RANGE = 5, LERP_FACTOR = 0.08;
      const REVEAL_MS = 1400, SLIDE_REVEAL_MS = 800;
      const START_SCALE = 1.12, END_SCALE = 1.06;
      const isTouchOnly = window.matchMedia('(hover: none)').matches;
      let bgTX = 0, bgTY = 0, bgCX = 0, bgCY = 0;
      let fgTX = 0, fgTY = 0, fgCX = 0, fgCY = 0;
      let pageT0 = null, slideT0 = null, activeIdx = 0;

      bgEls.forEach(bg => { if (bg) bg.style.transform = 'translate(0px,0px) scale(1.12)'; });

      function revealContent(el) {
        if (!el) return;
        el.style.opacity = '0';
        el.style.transform = 'translateY(12px)';
        el.style.transition = 'opacity 0.9s ease 0.15s, transform 0.9s cubic-bezier(0.16,1,0.3,1) 0.15s';
        requestAnimationFrame(() => requestAnimationFrame(() => {
          el.style.opacity = '1';
          el.style.transform = 'translateY(0px)';
        }));
        setTimeout(() => { el.style.transition = ''; }, 1400);
      }
      revealContent(contentEls[0]);

      fpgOnSlideChange = function(n) {
        activeIdx = n;
        slideT0 = null;
        bgTX = 0; bgTY = 0; fgTX = 0; fgTY = 0;
        if (bgEls[n]) bgEls[n].style.transform = 'translate(0px,0px) scale(1.12)';
        revealContent(contentEls[n]);
      };

      if (!isTouchOnly) {
        document.addEventListener('mousemove', function(e) {
          const nx = e.clientX / window.innerWidth - 0.5;
          const ny = e.clientY / window.innerHeight - 0.5;
          bgTX = nx * -BG_RANGE; bgTY = ny * -BG_RANGE;
          fgTX = nx * FG_RANGE;  fgTY = ny * FG_RANGE;
        }, { passive: true });
      }

      function lerp(a, b, t) { return a + (b - a) * t; }

      function tick(ts) {
        if (!pageT0) pageT0 = ts;
        if (!slideT0) slideT0 = ts;

        const pageElapsed  = ts - pageT0;
        const slideElapsed = ts - slideT0;
        const isFirstLoad  = activeIdx === 0 && pageElapsed < REVEAL_MS;
        const elapsed  = isFirstLoad ? pageElapsed  : slideElapsed;
        const duration = isFirstLoad ? REVEAL_MS    : SLIDE_REVEAL_MS;

        const progress = Math.min(elapsed / duration, 1);
        const ease  = 1 - Math.pow(1 - progress, 3);
        const scale = START_SCALE - (START_SCALE - END_SCALE) * ease;

        if (!isTouchOnly) {
          bgCX = lerp(bgCX, bgTX, LERP_FACTOR); bgCY = lerp(bgCY, bgTY, LERP_FACTOR);
          fgCX = lerp(fgCX, fgTX, LERP_FACTOR); fgCY = lerp(fgCY, fgTY, LERP_FACTOR);
        }

        const activeBg = bgEls[activeIdx];
        if (activeBg) {
          activeBg.style.transform = `translate(${bgCX.toFixed(2)}px,${bgCY.toFixed(2)}px) scale(${scale.toFixed(4)})`;
        }

        if (!isTouchOnly && elapsed > duration) {
          const activeContent = contentEls[activeIdx];
          if (activeContent) {
            activeContent.style.transform = `translate(${fgCX.toFixed(2)}px,${fgCY.toFixed(2)}px)`;
          }
        }

        requestAnimationFrame(tick);
      }

      requestAnimationFrame(tick);
    })();
```

- [ ] **Step 6: Visual verify `http://localhost:3000/index.html`**

1. **Load reveal on slide 0:** Hard refresh → FPG brand hero (kids in field) starts slightly zoomed in and pulls back over 1.4s. "FRIED PICKLE GAMES" title and buttons fade in from below.
2. **Parallax on slide 0:** Move cursor — image drifts opposite, text drifts with cursor. Smooth inertial motion.
3. **Slide change (wait 7.5s or click a dot to advance to slide 1 — FOW):**
   - Incoming FOW image starts slightly zoomed in and pulls back over 0.8s.
   - "FART / OF WAR" title fades in from below within the new slide.
   - Parallax position resets smoothly (no jump).
4. **Subsequent slides:** Verify slides 2 (Dispelled) and 3 (Expansion) behave identically to slide 1.
5. **No edge bleed:** At any cursor position, the hero bg never shows a gap at any edge.

Take a screenshot to `screenshots/parallax_homepage.png`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/terrangreenfield/fartofwar.com add index.html
git -C /Users/terrangreenfield/fartofwar.com commit -m "homepage: add parallax depth + reveal to hero carousel"
```

---

## Task 5: Push to Vercel

- [ ] **Step 1: Push to production**

```bash
git -C /Users/terrangreenfield/fartofwar.com push fpg2 main
```

- [ ] **Step 2: Verify live site**

Open `https://fpg-test-website-2.vercel.app` and confirm parallax + reveal work on the deployed build.
