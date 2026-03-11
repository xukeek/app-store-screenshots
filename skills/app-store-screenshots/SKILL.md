---
name: app-store-screenshots
description: Use when building App Store screenshot pages, generating exportable marketing screenshots for iOS apps, or creating programmatic screenshot generators with Next.js. Triggers on app store, screenshots, marketing assets, html-to-image, phone mockup.
---

# App Store Screenshots Generator

## Overview

Build a Next.js page that renders iOS App Store screenshots as **advertisements** (not UI showcases) and exports them via `html-to-image` at Apple's required resolutions. Screenshots are the single most important conversion asset on the App Store. The generator must support multiple locales, including `zh-CN`, when the user needs localized App Store assets. It should also generate localized App Store marketing metadata that fits Apple's current length limits.

## Core Principle

**Screenshots are advertisements, not documentation.** Every screenshot sells one idea. If you're showing UI, you're doing it wrong — you're selling a *feeling*, an *outcome*, or killing a *pain point*.

## Step 1: Ask the User These Questions

Before writing ANY code, ask the user all of these. Do not proceed until you have answers:

### Required

1. **App screenshots** — "Where are your app screenshots? Provide one directory per locale. For Chinese and English, provide two directories such as `.../screenshots/zh-CN/` and `.../screenshots/en-US/` containing PNG files of actual device captures."
2. **App icon** — "Where is your app icon PNG?"
3. **Brand colors** — "What are your brand colors? (accent color, text color, background preference)"
4. **Font** — "What font does your app use? (or what font do you want for the screenshots?)"
5. **Feature list** — "List your app's features in priority order. What's the #1 thing your app does?"
6. **Number of slides** — "How many screenshots do you want? (Apple allows up to 10)"
7. **Style direction** — "What style do you want? Examples: warm/organic, dark/moody, clean/minimal, bold/colorful, gradient-heavy, flat. Share App Store screenshot references if you have any."
8. **Locales** — "Which App Store locales do you need? List exact locale codes if you know them (for example: `en-US`, `zh-CN`, `zh-TW`, `ja-JP`). Which locale should be treated as the source copy?"
9. **Screenshot language** — "Confirm the screenshot directories match the target locales. If any locale is missing its own screenshot directory, call that out explicitly before proceeding."
10. **App Store metadata scope** — "Do you want localized `App Name`, `Subtitle`, `Promotional Text`, `Description`, and `Keywords` generated too? Which of these already exist, if any?"

### Optional

11. **Component assets** — "Do you have any UI element PNGs (cards, widgets, etc.) you want as floating decorations? If not, that's fine — we'll skip them."
12. **Additional instructions** — "Any specific requirements, constraints, or preferences?"

### Derived from answers (do NOT ask — decide yourself)

Based on the user's style direction, brand colors, and app aesthetic, decide:
- **Background style**: gradient direction, colors, whether light or dark base
- **Decorative elements**: blobs, glows, geometric shapes, or none — match the style
- **Dark vs light slides**: how many of each, which features suit dark treatment
- **Typography treatment**: weight, tracking, line height — match the brand personality
- **Color palette**: derive text colors, secondary colors, shadow tints from the brand colors
- **Localization strategy**: whether every locale can share the same layout, or whether certain locales (especially `zh-CN`, `zh-TW`, `ja-JP`, `ko-KR`, `de-DE`) need adjusted font size, line breaks, or slide copy length

### Locale Rules

- Treat locale support as a first-class requirement, not an afterthought.
- Keep the same narrative arc across locales unless the user explicitly wants market-specific messaging.
- Never ship translated headlines without checking visual fit. CJK copy and long German compound words often need different line breaks or slightly different sizing.
- Source screenshots must also be locale-specific. Do not assume one screenshot set can be reused across Chinese and English unless the user explicitly approves mixed-language UI.
- If the source screenshots are not localized, explicitly warn the user that the exported marketing copy may not match the in-app UI language.
- Default to supporting `zh-CN` cleanly when Chinese is requested. Do not assume English-only fonts or English-only copy rules.
- Treat App Store metadata as part of the same deliverable as the screenshots when the user asks for launch assets.
- The preview page must show Chinese and English at the same time when both locales are requested. Do not hide one locale behind a toggle by default.

### App Store Metadata Limits

Follow Apple's current limits when generating metadata:

- **App Name**: 30 characters max
- **Subtitle**: 30 characters max
- **Promotional Text**: 170 characters max
- **Description**: 4000 characters max
- **Keywords**: App Store Connect reference says 100 bytes; Apple's product page guidance says 100 characters total with comma-separated terms and no spaces after commas

Because Apple documents keywords in two slightly different ways, use the safer implementation rule:

- Keep keyword strings under **100 bytes**
- Separate terms with commas
- Do not insert spaces after commas unless a multi-word keyword phrase truly needs them
- Validate final keyword strings byte-length, not only character-length, especially for `zh-CN`, `ja-JP`, and `ko-KR`

Metadata must also follow App Review guidance:

- Do not stuff metadata with unrelated terms, competitor names, trademarked names, or pricing language
- Do not make unverifiable claims in subtitles or screenshot text
- Keep metadata appropriate for all audiences

**IMPORTANT:** If the user gives additional instructions at any point during the process, follow them. User instructions always override skill defaults.

## Step 2: Set Up the Project

### Detect Package Manager

Check what's available, use this priority: **bun > pnpm > yarn > npm**

```bash
# Check in order
which bun && echo "use bun" || which pnpm && echo "use pnpm" || which yarn && echo "use yarn" || echo "use npm"
```

### Scaffold (if no existing Next.js project)

```bash
# With bun:
bunx create-next-app@latest . --typescript --tailwind --app --src-dir --no-eslint --import-alias "@/*"
bun add html-to-image

# With pnpm:
pnpx create-next-app@latest . --typescript --tailwind --app --src-dir --no-eslint --import-alias "@/*"
pnpm add html-to-image

# With yarn:
yarn create next-app . --typescript --tailwind --app --src-dir --no-eslint --import-alias "@/*"
yarn add html-to-image

# With npm:
npx create-next-app@latest . --typescript --tailwind --app --src-dir --no-eslint --import-alias "@/*"
npm install html-to-image
```

### Copy the Phone Mockup

The skill includes a pre-measured iPhone mockup at `mockup.png` (co-located with this SKILL.md). Copy it to the project's `public/` directory. The mockup file is in the same directory as this skill file.

### File Structure

```
project/
├── public/
│   ├── mockup.png              # Phone frame (included with skill)
│   ├── app-icon.png            # User's app icon
│   └── screenshots/            # User's app screenshots
│       ├── en-US/
│       │   ├── home.png
│       │   ├── feature-1.png
│       │   └── ...
│       └── zh-CN/
│           ├── home.png
│           ├── feature-1.png
│           └── ...
├── src/app/
│   ├── layout.tsx              # Font setup
│   └── page.tsx                # The screenshot generator (single file)
└── package.json
```

**The entire generator is a single `page.tsx` file.** No routing, no extra layouts, no API routes.

### Font Setup

```tsx
// src/app/layout.tsx
import { YourFont } from "next/font/google"; // Use whatever font the user specified
const font = YourFont({ subsets: ["latin"] });

export default function Layout({ children }: { children: React.ReactNode }) {
  return <html><body className={font.className}>{children}</body></html>;
}
```

If the user needs Chinese, Japanese, or Korean text, do **not** use a latin-only font setup. Use one of these approaches:

- A CJK-capable web font via `next/font/google` (for example `Noto_Sans_SC`, `Noto_Sans_TC`, `Noto_Sans_JP`, `Noto_Sans_KR`)
- A local font via `next/font/local` when the brand already has its own CJK font files
- A font stack fallback in the screenshot canvas styles so exports do not silently fall back to serif defaults

Example for Simplified Chinese:

```tsx
import { Inter, Noto_Sans_SC } from "next/font/google";

const latin = Inter({ subsets: ["latin"] });
const cjk = Noto_Sans_SC({ subsets: ["latin"], weight: ["400", "500", "600", "700"] });

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body className={`${latin.className} ${cjk.className}`}>{children}</body>
    </html>
  );
}
```

If multiple locales are requested, mirror that locale structure for screenshot assets too. The simplest safe convention is:

```text
public/screenshots/en-US/
public/screenshots/zh-CN/
```

The generator should read screenshots from the matching locale directory instead of trying to reuse one default screenshot folder for all languages.

## Step 3: Plan the Slides

### Screenshot Framework (Narrative Arc)

Adapt this framework to the user's requested slide count. Not all slots are required — pick what fits:

| Slot | Purpose | Notes |
|------|---------|-------|
| #1 | **Hero / Main Benefit** | App icon + tagline + home screen. This is the ONLY one most people see. |
| #2 | **Differentiator** | What makes this app unique vs competitors |
| #3 | **Ecosystem** | Widgets, extensions, watch — beyond the main app. Skip if N/A. |
| #4+ | **Core Features** | One feature per slide, most important first |
| 2nd to last | **Trust Signal** | Identity/craft — "made for people who [X]" |
| Last | **More Features** | Pills listing extras + coming soon. Skip if few features. |

**Rules:**
- Each slide sells ONE idea. Never two features on one slide.
- Vary layouts across slides — never repeat the same template structure.
- Include 1-2 contrast slides (inverted bg) for visual rhythm.
- Keep slide order consistent across locales so exports remain easy to review side by side.

## Step 4: Write Copy FIRST

Get all headlines approved before building layouts. Bad copy ruins good design.

Write the source-locale copy first, then localize it. Do not mechanically translate unfinished English copy.

### The Iron Rules

1. **One idea per headline.** Never join two things with "and."
2. **Short, common words.** 1-2 syllables. No jargon unless it's domain-specific.
3. **3-5 words per line.** Must be readable at thumbnail size in the App Store.
4. **Line breaks are intentional.** Control where lines break with `<br />`.

For non-English locales, adapt the rule instead of forcing English structure:

- `zh-CN` / `zh-TW`: optimize for 4-10 Chinese characters per line in most cases, and prefer 1-2 short lines over long stacked paragraphs.
- `ja-JP` / `ko-KR`: optimize for short visual phrases, not literal English-length translations.
- Languages with long words (`de-DE`, `fr-FR`, etc.): rewrite for compactness rather than shrinking the font aggressively.

### Three Approaches (pick one per slide)

| Type | What it does | Example |
|------|-------------|---------|
| **Paint a moment** | You picture yourself doing it | "Check your coffee without opening the app." |
| **State an outcome** | What your life looks like after | "A home for every coffee you buy." |
| **Kill a pain** | Name a problem and destroy it | "Never waste a great bag of coffee." |

### What NEVER Works

- **Feature lists as headlines**: "Log every item with tags, categories, and notes"
- **Two ideas joined by "and"**: "Track X and never miss Y"
- **Compound clauses**: "Save and customize X for every Y you own"
- **Vague aspirational**: "Every item, tracked"
- **Marketing buzzwords**: "AI-powered tips" (unless it's actually AI)

### Copy Process

1. Write 3 options per slide using the three approaches
2. Read each at arm's length — if you can't parse it in 1 second, it's too complex
3. Check: does each line have 3-5 words? If not, adjust line breaks
4. Present options to the user with reasoning for each

### Localization Process

1. Approve the source-locale messaging first.
2. Create a locale table for every slide, for example `headline.en-US`, `headline.zh-CN`.
3. Rewrite per locale for brevity and natural phrasing. Do not do literal translation if the result is visually weak.
4. Re-check line breaks and thumbnail readability for every locale.
5. If one locale needs noticeably different sizing or line breaks, encode that in data instead of hardcoding one global headline style.

### Chinese Copy Guidance

- Prefer natural product copy, not textbook translation.
- Keep claims concrete and compact.
- Avoid awkward mixtures of English sentence rhythm and Chinese wording.
- Good `zh-CN` screenshot copy is usually shorter than the English original, not longer.
- If the app already has established Chinese terminology, reuse it consistently across all slides.

### Reference Apps for Copy Style

- **Raycast** — specific, descriptive, one concrete value per slide
- **Turf** — ultra-simple action verbs, conversational
- **Mela / Notion** — warm, minimal, elegant

## Step 5: Generate App Store Metadata

If the user wants launch-ready marketing assets, generate localized metadata alongside the screenshots.

### Required Metadata Set

For each locale, generate:

- **App Name**
- **Subtitle**
- **Promotional Text**
- **Description**
- **Keywords**

If useful, also suggest but clearly separate:

- **What's New**
- **In-App Purchase display names/descriptions**
- **Custom Product Page variant copy**

### Metadata Workflow

1. Start from the approved positioning and screenshot narrative.
2. Generate a source-locale metadata draft first.
3. Localize by rewriting for that market, not by literal translation.
4. Validate every field against Apple's length constraints.
5. Return both the final text and the measured length for each field.
6. Flag any field that is near the limit so the user can shorten it more safely later.

### Metadata Output Format

Return metadata in a locale-keyed structure such as:

```ts
type AppStoreMetadata = {
  locale: string;
  name: { text: string; length: number };
  subtitle: { text: string; length: number };
  promotionalText: { text: string; length: number };
  description: { text: string; length: number };
  keywords: { text: string; charLength: number; byteLength: number };
};
```

Also present a human-readable table for review.

### Metadata Writing Rules

- **App Name**: brand-led, specific, distinct, under 30 characters
- **Subtitle**: one clear value proposition, under 30 characters
- **Promotional Text**: campaign-oriented or update-oriented, under 170 characters, not keyword stuffing
- **Description**: lead with the strongest user value in the opening sentence because that text is the most visible
- **Keywords**: comma-separated, deduplicated, no competitor names, no generic filler like `app`, no repeated words already covered by brand/app name

### Multi-Language Metadata Rules

- Generate metadata separately for every locale. Do not share one English metadata block across all languages.
- Chinese and English must be delivered as separate locale blocks and exported into separate directories if files are written.
- For Chinese metadata, optimize for natural search terms used by native speakers, not direct transliterations of English marketing language.
- For keywords in CJK locales, byte length is the real constraint that will usually bite first.
- If a locale's app name must stay in English for brand reasons, document that decision explicitly instead of pretending it is translated.

### Validation Requirements

Before finalizing metadata, check:

- `name.length <= 30`
- `subtitle.length <= 30`
- `promotionalText.length <= 170`
- `description.length <= 4000`
- `keywords.byteLength <= 100`

If files are written locally, save them per locale, for example:

```text
exports/
  en-US/
    metadata.json
    metadata.md
  zh-CN/
    metadata.json
    metadata.md
```

## Step 6: Build the Page

### Architecture

```
page.tsx
├── Constants (W, H, SIZES, design tokens from user's brand)
├── Locale config (supported locales, source locale, per-locale font handling)
├── Phone component (mockup with screen overlay)
├── Caption component (label + headline, locale-aware)
├── Decorative components (blobs, glows, shapes — based on style direction)
├── Screenshot1..N components (one per slide, optionally locale-aware)
├── SCREENSHOTS array (registry)
├── COPY map / locale dictionaries
├── ScreenshotPreview (ResizeObserver scaling + hover export)
├── LocaleColumn / LocaleSection (one section per locale)
└── ScreenshotsPage (side-by-side locale grid + toolbar + export logic)
```

### Export Sizes (Apple Required — iPhone only, portrait)

```typescript
const SIZES = [
  { label: '6.9"', w: 1320, h: 2868 },
  { label: '6.5"', w: 1284, h: 2778 },
  { label: '6.3"', w: 1206, h: 2622 },
  { label: '6.1"', w: 1125, h: 2436 },
] as const;
```

Design at the LARGEST size (1320x2868) and scale down for export.

If multiple locales are requested, design at one master size but export a full set per locale.

### Rendering Strategy

Each screenshot is designed at full resolution (1320x2868px). Two copies exist:

1. **Preview**: CSS `transform: scale()` via ResizeObserver to fit a grid card
2. **Export**: Offscreen at `position: absolute; left: -9999px` at true resolution

When supporting multiple locales:

1. Show every requested locale in the page at the same time so copy and screenshots can be reviewed side by side.
2. For Chinese and English, default to a two-column layout on desktop: `zh-CN` and `en-US`.
3. On smaller screens, stack locale sections vertically, but still keep both locales rendered in the same page.
4. Render export nodes per locale, or re-render one export node with explicit locale state before capture.
5. Export every locale into its own directory so App Store upload is unambiguous.
6. For Chinese and English specifically, use separate directories such as `exports/zh-CN/` and `exports/en-US/`.

### Phone Mockup Component

The included `mockup.png` has these pre-measured values:

```typescript
const MK_W = 1022;  // mockup image width
const MK_H = 2082;  // mockup image height
const SC_L = (52 / MK_W) * 100;   // screen left offset %
const SC_T = (46 / MK_H) * 100;   // screen top offset %
const SC_W = (918 / MK_W) * 100;  // screen width %
const SC_H = (1990 / MK_H) * 100; // screen height %
const SC_RX = (126 / 918) * 100;  // border-radius x %
const SC_RY = (126 / 1990) * 100; // border-radius y %
```

```tsx
function Phone({ src, alt, style, className = "" }: {
  src: string; alt: string; style?: React.CSSProperties; className?: string;
}) {
  return (
    <div className={`relative ${className}`}
      style={{ aspectRatio: `${MK_W}/${MK_H}`, ...style }}>
      <img src="/mockup.png" alt=""
        className="block w-full h-full" draggable={false} />
      <div className="absolute z-10 overflow-hidden"
        style={{
          left: `${SC_L}%`, top: `${SC_T}%`,
          width: `${SC_W}%`, height: `${SC_H}%`,
          borderRadius: `${SC_RX}% / ${SC_RY}%`,
        }}>
        <img src={src} alt={alt}
          className="block w-full h-full object-cover object-top"
          draggable={false} />
      </div>
    </div>
  );
}
```

### Typography (Resolution-Independent)

All sizing relative to canvas width W:

| Element | Size | Weight | Line Height |
|---------|------|--------|-------------|
| Category label | `W * 0.028` | 600 (semibold) | default |
| Headline | `W * 0.09` to `W * 0.1` | 700 (bold) | 1.0 |
| Hero headline | `W * 0.1` | 700 (bold) | 0.92 |

Locale-specific adjustments are expected:

- `zh-CN` headlines often look better with slightly tighter line height and less tracking.
- CJK fonts can render visually larger at the same nominal size; reduce font size slightly before reducing the number of lines.
- Never apply English-style letter spacing to Chinese text.

### Phone Placement Patterns

Vary across slides — NEVER use the same layout twice in a row:

**Centered phone** (hero, single-feature):
```
bottom: 0, width: "82-86%", translateX(-50%) translateY(12-14%)
```

**Two phones layered** (comparison):
```
Back: left: "-8%", width: "65%", rotate(-4deg), opacity: 0.55
Front: right: "-4%", width: "82%", translateY(10%)
```

**Phone + floating elements** (only if user provided component PNGs):
```
Cards should NOT block the phone's main content.
Position at edges, slight rotation (2-5deg), drop shadows.
If distracting, push partially off-screen or make smaller.
```

### "More Features" Slide (Optional)

Dark/contrast background with app icon, headline ("And so much more."), and feature pills. Can include a "Coming Soon" section with dimmer pills.

## Step 7: Export

### Why html-to-image, NOT html2canvas

`html2canvas` breaks on CSS filters, gradients, drop-shadow, backdrop-filter, and complex clipping. `html-to-image` uses native browser SVG serialization — handles all CSS faithfully.

### Export Implementation

```typescript
import { toPng } from "html-to-image";

// Before capture: move element on-screen
el.style.left = "0px";
el.style.opacity = "1";
el.style.zIndex = "-1";

const opts = { width: W, height: H, pixelRatio: 1, cacheBust: true };

// CRITICAL: Double-call trick — first warms up fonts/images, second produces clean output
await toPng(el, opts);
const dataUrl = await toPng(el, opts);

// After capture: move back off-screen
el.style.left = "-9999px";
el.style.opacity = "";
el.style.zIndex = "";
```

### Key Rules

- **Double-call trick**: First `toPng()` loads fonts/images lazily. Second produces clean output. Without this, exports are blank.
- **On-screen for capture**: Temporarily move to `left: 0` before calling `toPng`.
- **Offscreen container**: Use `position: absolute; left: -9999px` (not `fixed`).
- **Resizing**: Load data URL into Image, draw onto canvas at target size.
- 300ms delay between sequential exports.
- Set `fontFamily` on the offscreen container.
- **Numbered filenames**: Prefix exports with zero-padded index so they sort correctly: `01-hero-1320x2868.png`, `02-freshness-1320x2868.png`, etc. Use `String(index + 1).padStart(2, "0")`.
- **Per-locale directories are required**: export each language into its own folder, for example `exports/zh-CN/01-hero-1320x2868.png` and `exports/en-US/01-hero-1320x2868.png`.
- **Do not mix locales in one directory** and do not rely on locale suffixes in filenames alone.
- **Font readiness matters more for CJK**: wait for `document.fonts.ready` before export when using Chinese fonts.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| All slides look the same | Vary phone position (center, left, right, two-phone, no-phone) |
| Decorative elements invisible | Increase size and opacity — better too visible than invisible |
| Copy is too complex | "One second at arm's length" test |
| Floating elements block the phone | Move off-screen edges or above the phone |
| Plain white/black background | Use gradients — even subtle ones add depth |
| Too cluttered | Remove floating elements, simplify to phone + caption |
| Too simple/empty | Add larger decorative elements, floating items at edges |
| Headlines use "and" | Split into two slides or pick one idea |
| No visual contrast across slides | Mix light and dark backgrounds |
| Export is blank | Use double-call trick; move element on-screen before capture |
