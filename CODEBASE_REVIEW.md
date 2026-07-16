# Focal Landing — Codebase Review

**Reviewer:** SpeeHive AI
**Date:** 2026-07-16
**Branch reviewed:** `main`

---

## 1. Project Overview

Focal Landing is a **Next.js 16** single-page landing site for *Focal Shipping Services LLC*, a maritime logistics company. It uses a cinematic scroll-driven experience with canvas-rendered frame sequences (video-like transitions), a corporate services section, and a contact form.

| Attribute         | Value                       |
|-------------------|-----------------------------|
| Framework         | Next.js 16.2.9 (App Router) |
| React             | 19.2.4                      |
| Styling           | Tailwind CSS v4 (via PostCSS plugin) |
| Language          | TypeScript 5                |
| Icons             | lucide-react 1.21           |
| Output            | Static export (`output: 'export'`) |
| Asset size        | ~190 MB (1,200 JPEG frames across 5 clips) |

---

## 2. Architecture & Structure

```
focal_landing/
├── app/
│   ├── components/
│   │   └── FocalJourney.tsx    # 1,270-line monolith — entire page
│   ├── globals.css              # Tailwind imports + custom animations
│   ├── icon.png
│   ├── layout.tsx               # Root layout, fonts, metadata
│   └── page.tsx                 # Thin wrapper rendering FocalJourney
├── public/
│   ├── frames/clip1..clip5/     # 240 frames × 5 clips (JPEG)
│   └── focal-logo-full.png
├── eslint.config.mjs
├── next.config.ts
├── package.json
├── postcss.config.mjs
└── tsconfig.json
```

### Observations

- **Monolithic component.** The entire 1,270-line `FocalJourney.tsx` contains state management, canvas rendering logic, animation orchestration, navigation, a services data array, the services grid, a modal, a contact form, and the footer — all in a single component file.
- **No route separation.** The app uses `output: 'export'` (static generation), which is appropriate for a landing page. However, the single-file approach limits maintainability.
- **No shared components or utilities.** Icons, data structures, animation helpers, and UI fragments are not extracted.

---

## 3. Code Quality Assessment

### 3.1 Lint & Type Check Results

| Check     | Status | Details |
|-----------|--------|---------|
| `tsc`     | Pass   | No type errors |
| `eslint`  | **Fail** | 3 errors, 7 warnings |

**Errors:**

| Line | Rule | Description |
|------|------|-------------|
| 363 | `react-hooks/immutability` | `animateStats` called before declaration in a `useEffect` — violates hook immutability rule |
| 762, 762 | `react/no-unescaped-entities` | Unescaped `"` characters in JSX (should use `&quot;` or `&#34;`) |

**Key Warnings:**

| Line | Rule | Description |
|------|------|-------------|
| 13 | `@typescript-eslint/no-unused-vars` | `ChevronDown` imported but never used |
| 209, 213, 365 | `react-hooks/exhaustive-deps` | Missing dependencies: `drawCurrentFrame`, `transitionTo` |
| 534, 567, 1250 | `@next/next/no-img-element` | Using `<img>` instead of `next/image` — impacts LCP and bandwidth |

### 3.2 TypeScript

- **Strict mode** is enabled — good.
- Types are well-defined for the `Service` interface and render state.
- The `renderStateRef` pattern (duplicating state in a ref for animation access) works but adds complexity.

### 3.3 Code Smells

1. **God component.** `FocalJourney.tsx` handles everything. Consider splitting into:
   - `HeroSection`
   - `JourneySections` (Sections 1–5)
   - `CorporateSection` (Section 6 with stats, services grid, global ports, ISO quality, contact form, footer)
   - `Navbar`
   - `Preloader`
   - `ServiceModal`
   - `ContactForm`

2. **Inline SVG course-line decorations.** Each section has a nearly identical SVG block with slight directional changes. This should be extracted into a `MaritimeCourseLine` component with props for direction.

3. **Services data defined inline.** The 10-item `services` array is hardcoded inside the component. Move to a separate `data/services.ts` file.

4. **Magic numbers.** `TOTAL_FRAMES = 240`, mix durations (`800`, `3800`), stat targets (`15`, `8`, `1200`, `250`), batch size (`25`) are all hardcoded. Consider extracting to named constants.

5. **Hardcoded clip loading.** `preloadBackgroundClips` loops through clips 2–5 individually. This could be a parameterized loop.

6. **Unused import.** `ChevronDown` is imported but never used.

---

## 4. Performance Considerations

### 4.1 Asset Loading

- **190 MB of JPEG frames** loaded eagerly on page mount. This is the biggest concern.
  - Clip 1 (240 frames) is prioritized and shows a loading bar — good UX.
  - Clips 2–5 load in the background after Clip 1 finishes — reasonable strategy.
  - However, **no lazy loading or intersection-observer-based deferred loading** exists for clips 3–5.

**Recommendations:**
- Consider serving clips as **WebM/MP4 video** instead of 1,200 individual JPEGs. A 5-clip video would be ~5–15 MB vs. 190 MB.
- If JPEG frames are required, implement **progressive loading** — only preload the next clip when the user approaches its section.
- Add `<link rel="preload">` for clip 1's first frame.

### 4.2 Canvas Rendering

- Canvas resizes on every window resize event without debounce — could cause jank.
- `drawCurrentFrame` is called both via `useEffect` on `renderState` and via the resize handler, which could cause double draws.

### 4.3 Image Usage

- `<img>` is used for the logo in three places (preloader, navbar, footer). Using `next/image` with appropriate `sizes` would improve performance and enable modern formats (WebP/AVIF).
- The logo images are small and appear in multiple contexts — consider using `priority` on the navbar logo.

---

## 5. Accessibility

| Issue | Severity | Description |
|-------|----------|-------------|
| No `<main>` landmark wrapping scroll content | Medium | The scroll container should be a `<main>` element |
| No `aria-label` on icon-only buttons | Medium | The mobile menu toggle, close buttons need accessible labels |
| No keyboard navigation for scroll sections | Medium | Section navigation only works via mouse click/scroll |
| Form has no validation or error states | Medium | Contact form fields have no `required`, `aria-describedby`, or error messaging |
| Color contrast | Low | Yellow-on-white text (`text-brand-yellow`) may fail WCAG AA for small text |
| No skip-to-content link | Low | Users must tab through the entire navbar to reach content |
| Footer links (`<a href="#">`) are non-functional | Low | Placeholder links should use `aria-disabled="true"` |

---

## 6. UX & Design

### Strengths
- Cinematic scroll-driven experience with canvas frame transitions is visually impressive.
- Clean typography hierarchy using Inter + Geist Mono.
- Consistent brand color system (blue `#1d75f2`, yellow `#f0f528`).
- Responsive mobile menu with animated toggle.
- The "OPERATIONAL 24/7/365" pill with live ping animation is a nice touch.
- Service detail modal is well-designed with structured information.

### Issues
- **No `<noscript>` fallback.** The entire experience requires JavaScript. Users with JS disabled see a blank page.
- **Snap scrolling** can feel jarring on some devices. Consider adding `scroll-snap-type: y proximity` as a gentler alternative.
- **Contact form is non-functional.** The `onSubmit` handler calls `e.preventDefault()` with no actual submission logic. This should either integrate with a backend or show a "coming soon" state.
- **Stats animation** re-runs every time section 6 is re-entered (no "already animated" guard). The numbers reset to 0 and animate again.
- **Footer links** (`SAFETY POLICY`, `ANTI-CORRUPTION`, `TERMS`) point to `#` — should be real pages or marked as coming soon.

---

## 7. Security

| Item | Status | Notes |
|------|--------|-------|
| No secrets in code | Pass | No API keys or secrets committed |
| `.env*` in `.gitignore` | Pass | Environment files excluded |
| No external API calls | Pass | Static export, no server-side code |
| Form data exposure | **Caution** | Contact form collects vessel names, emails — no data protection measures in place |

---

## 8. Testing

- **No test suite exists.** There are no test files, no testing library in dependencies, and no test script in `package.json`.
- For a landing page this is somewhat acceptable, but consider adding:
  - A smoke test to verify the page renders
  - Visual regression tests (e.g., Chromatic, Percy) for the canvas animations
  - Accessibility tests (axe-core)

---

## 9. Build & Deployment

- `output: 'export'` generates a fully static site — ideal for CDN deployment (Vercel, Cloudflare Pages, Netlify).
- The `public/frames/` directory (190 MB) will be included in the build output. Ensure the hosting platform supports large static deployments.
- No CI/CD configuration found (no `.github/workflows`, no `vercel.json`).

---

## 10. Recommendations Summary

### High Priority
1. **Fix lint errors** — resolve the 3 ESLint errors (animateStats ordering, unescaped entities).
2. **Replace `<img>` with `next/image`** for the logo — improves performance and follows Next.js best practices.
3. **Extract components** from `FocalJourney.tsx` to improve maintainability and code readability.
4. **Debounce the resize handler** to prevent layout thrashing.

### Medium Priority
5. **Move `services` data** to a separate file.
6. **Add `<noscript>` fallback** for users without JavaScript.
7. **Fix accessibility gaps** — add `aria-label` to icon buttons, form validation, and skip-to-content link.
8. **Make contact form functional** or clearly indicate it's a placeholder.

### Low Priority
9. **Add ESLint to CI** to catch regressions.
10. **Consider video format** for frame sequences to dramatically reduce asset size.
11. **Add a basic smoke test** for the landing page.
12. **Clean up unused import** (`ChevronDown`).

---

*This review was generated by SpeeHive AI as part of the Asana task workflow.*
