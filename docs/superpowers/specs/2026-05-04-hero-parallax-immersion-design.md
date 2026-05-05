# Hero Parallax Immersion — Design Spec

**Date:** 2026-05-04  
**Status:** Approved

---

## Goal

Make visitors feel like they've stepped *into* the game worlds rather than looking at a website about them. The wardrobe analogy: crossing a threshold into a living space, not browsing a brochure.

---

## Scope

Four pages: `index.html`, `fow.html`, `expansion.html`, `dispelled.html`.  
Pure vanilla JS + CSS. No new dependencies. Inline in each HTML file.

---

## Core Effect: Parallax Depth + Mouse-Tracking

### Two depth layers on every hero section

The hero art (gas ghost, wizard, cave characters) is baked into the single `hero-bg` image — there is no separate foreground element. Depth is achieved through a **differential movement** between the background image and the foreground content block:

**Layer 1 — Background (the world):**  
The `.hero-bg` image moves opposite to the cursor at ±12px — mouse moves left → world drifts right. This is the primary parallax effect.

**Layer 2 — Content block (floating in front):**  
The `.hero-content` text/CTA block moves in the *same* direction as the cursor at ±5px. The differential between content (+5px) and background (-12px) = 17px of perceived depth — the text and buttons feel like they're floating closer to you than the world behind. No new elements or image assets needed.

### Animation quality

- Both layers use `transform` only — no layout properties, no reflow. Smooth 60fps.
- Movement is damped via lerp (linear interpolation factor ~0.08 per frame): `current += (target - current) * 0.08`. This gives the floating, inertial quality — the world eases into position rather than snapping.
- `requestAnimationFrame` loop runs only when the hero is in viewport.
- Overflow: `overflow: hidden` on `.hero-section` clips any edge bleed. Hero image gets a small initial scale of `1.06` to provide travel room without revealing edges.

### Intensity values

| Layer | Direction | Max travel (each axis) | Lerp factor |
|---|---|---|---|
| Background (`.hero-bg`) | Opposite to cursor | ±12px | 0.08 |
| Content (`.hero-content`) | Same as cursor | ±5px | 0.08 |

Movement is mapped from cursor position relative to viewport center: `(mouseX / viewportWidth - 0.5) * maxTravel * -1` for background, `* +1` for content.

---

## Page Load Reveal

On every page load, the hero starts at `scale(1.06)` (already set for parallax travel room) and transitions to `scale(1.0)` over 1.4 seconds using a custom ease-out curve (`cubic-bezier(0.16, 1, 0.3, 1)`).

Simultaneously, the hero content block (title, tagline, CTAs) fades up from `translateY(12px), opacity: 0` to `translateY(0), opacity: 1` over 0.9 seconds with a 0.15s delay. This makes the text feel like it crystallizes out of the world rather than appearing on a page.

Both transitions use CSS classes toggled by JS on `DOMContentLoaded`:
- `.hero-reveal-active` on the `.hero-section` triggers the scale transition
- `.hero-content-visible` on `.hero-content` triggers the text reveal

---

## Homepage Carousel Interaction

The homepage cycles through three slides (FOW, Expansion, Dispelled). Additional behavior on top of existing fade:

- On each slide change, the incoming slide's background resets to `scale(1.06)` momentarily, then transitions back to `scale(1.0)` over 0.8s — a compressed version of the load reveal, reinforcing "entering a new world."
- The parallax lerp target resets to center (0, 0) on slide change, so each incoming world starts level rather than inheriting the previous cursor offset.
- The outgoing slide's parallax position is left as-is during its opacity fade-out (invisible anyway).

---

## Mobile Behavior

Mouse-tracking parallax is disabled on touch devices (`window.matchMedia('(hover: none)')`).  
The page load reveal (scale + text fade-up) still plays on mobile — it works purely on timers, no interaction required.

---

## Files Changed

| File | Changes |
|---|---|
| `index.html` | Load reveal + mouse parallax + carousel reset behavior |
| `fow.html` | Load reveal + mouse parallax |
| `expansion.html` | Load reveal + mouse parallax |
| `dispelled.html` | Load reveal + mouse parallax |

No new files. No new dependencies.

---

## What Is Not In Scope

- Particle systems / ambient animation (deferred — not Approach A)
- Page transition ceremonies between pages (deferred — not Approach A)
- Sound / audio
- Any sections below the hero fold
