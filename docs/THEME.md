# THEME.md — WSU Clinical Logbook design tokens

**Derived from:** `docs/design/WSU_new_logo.png` (colours sampled directly from the file, 12 Aug 2026)
**Implemented in:** `src/App.pa.yaml` → `App.Formulas` as named formulas
**Rule:** screens reference the named formulas only. **Never hardcode a hex value in a screen.**

---

## 1. Colours sampled from the logo

The logo is a maroon wordmark beside a multi-coloured chevron mark. Pixel frequency across the 5 893 opaque pixels:

| Hex | Share | Where it appears |
|---|---|---|
| `#551516` | 57.5% | Wordmark "Walter Sisulu" — **the WSU signature colour** |
| `#CF8029` | 14.3% | Amber chevron |
| `#A02124` | 3.5% | Crimson chevron |
| `#B45736` | 2.8% | Terracotta chevron |
| `#8E7951` | 2.1% | Gold chevron |
| `#C35E54` | 1.5% | Coral chevron |
| `#A080A8` | 1.4% | Mauve chevron |
| `#546A55` | 1.0% | Sage chevron |

Deep maroon is the anchor; amber is the natural accent. The sage and crimson chevrons give brand-native starting points for success and error, so the semantic colours belong to the same family rather than being generic web green and red.

---

## 2. The palette

Every value below was checked with the WCAG 2.1 relative-luminance formula against all three background tokens (`#FFFFFF`, `#FAF7F6`, `#F7F3F3`). The figure quoted is the **worst case** of the three.

### Brand

| Token | Hex | Contrast | Role |
|---|---|---|---|
| `ThemePrimary` | `#551516` | 12.67 ✅ | Headings, primary buttons, key UI. Straight from the wordmark. |
| `ThemePrimaryDeep` | `#3A0E0F` | 15.26 ✅ | Pressed states, highest-emphasis text. |
| `ThemePrimaryLight` | `#7A2B2C` | 8.63 ✅ | Hover states, secondary brand surfaces. |
| `ThemeAccent` | `#CF8029` | 2.82 ⚠️ | **Fills and graphics only — never text.** Progress bars, chevrons, icon shapes. |
| `ThemeAccentDeep` | `#8A5316` | 5.73 ✅ | The text-safe amber. Use wherever accent-coloured *words* are wanted. |

> ⚠️ **`ThemeAccent` is the one trap in this palette.** At 2.82:1 it fails AA for text *and* falls just under the 3:1 non-text threshold on the canvas background. Use it for decorative fill only. It is safe as a solid button fill with white text **at large sizes only** (3.10:1 against white). For anything read as text, use `ThemeAccentDeep`.

### Neutrals

Warm-tinted greys, not pure neutral, so they sit with the maroon rather than fighting it.

| Token | Hex | Contrast | Role |
|---|---|---|---|
| `ThemeInk` | `#1C1416` | 16.42 ✅ | Primary body text. |
| `ThemeInkMuted` | `#5C4F52` | 7.08 ✅ | Secondary text, subtitles. |
| `ThemeInkSubtle` | `#736467` | 5.09 ✅ | Tertiary text, placeholders, captions. |
| `ThemeLine` | `#E5DCDD` | 1.22 | **Decorative dividers only** — fails 3:1, so never use it as the sole boundary of an interactive control. |
| `ThemeLineStrong` | `#8F8083` | 3.42 ✅ | Input borders and any boundary that carries meaning. Meets the 3:1 non-text requirement. |
| `ThemeSurface` | `#FFFFFF` | — | Cards, sheets. |
| `ThemeSurfaceAlt` | `#F7F3F3` | — | Subtle inset panels. |
| `ThemeCanvas` | `#FAF7F6` | — | Screen background. Warm off-white, softer than pure white on an outdoor phone screen. |

### Semantic

| Token | Hex | On white | On own tint | Role |
|---|---|---|---|---|
| `ThemeSuccess` | `#3D5A3E` | 7.69 ✅ | 6.83 ✅ | Complete, verified, Valid entries. Deepened from the logo's sage. |
| `ThemeWarning` | `#8A5316` | 6.31 ✅ | 5.74 ✅ | Flagged entries, location unavailable. |
| `ThemeError` | `#9A1B1E` | 8.24 ✅ | 7.18 ✅ | Errors, destructive actions. From the crimson chevron. |
| `ThemeSuccessTint` | `#EDF3ED` | — | — | Success badge background. |
| `ThemeWarningTint` | `#FDF3E4` | — | — | Warning badge background. |
| `ThemeErrorTint` | `#FBECEC` | — | — | Error badge background. |
| `ThemeAccentTint` | `#FDF1E3` | — | — | Accent card background. |

White text on every solid semantic fill passes AA: Success 7.69, Warning 6.31, Error 8.24.

---

## 3. Type

`ThemeFont` = **Open Sans** — one of the fonts Power Apps ships natively, so it needs no external loading on a slow connection.

| Token | Size | Weight | Role |
|---|---|---|---|
| `ThemeSizeDisplay` | 28 | Semibold | Splash / screen hero |
| `ThemeSizeHeading` | 22 | Semibold | Screen titles |
| `ThemeSizeSubhead` | 17 | Semibold | Section headers |
| `ThemeSizeBody` | 15 | Normal | Body copy — the mobile floor for comfortable reading |
| `ThemeSizeCaption` | 13 | Normal | Metadata, helper text |

Body text never goes below 13. Nothing relies on colour alone to convey meaning.

---

## 4. Spacing, radius, touch

A 4-point scale: `ThemeSpaceXS` 4 · `ThemeSpaceS` 8 · `ThemeSpaceM` 16 · `ThemeSpaceL` 24 · `ThemeSpaceXL` 32.

Radius: `ThemeRadius` 12 (cards) · `ThemeRadiusSmall` 8 (chips, inputs).

`ThemeTouchTarget` = **48** — the minimum height for anything tappable, matching the WCAG 2.5.5 / platform guidance for thumb use. Primary actions sit near the bottom of the screen, per DECISION.md §2.7.

---

## 5. How to use it

```powerfx
// correct
Fill: =ThemePrimary
Color: =ThemeInk
Size: =ThemeSizeBody

// wrong — never do this
Fill: =RGBA(85, 21, 22, 1)
```

`ColorValue()` is evaluated once per named formula, not per render, so there is no per-frame cost.

**Adding a colour:** add it here with its contrast figure first, then add the named formula. A token that has not been contrast-checked does not belong in the palette.

---

## 6. The accessibility checker's static-text findings

Power Apps' built-in accessibility checker reports ~33 errors against these screens, of the form *"`lblYearVal.TabIndex`: Missing tab stop"* and *"`lblYearVal.FocusedBorderThickness`: Focus isn't showing"*. **These are expected and are not being fixed.**

The checker treats every `Label` and `Image` as interactive, because those controls expose an `OnSelect` property. Every label in these screens is static text — a field caption, a value, a heading. Its suggested fix, `TabIndex = 0`, would put all 30-odd static labels into the keyboard tab order, forcing a keyboard or switch user to tab through every caption to cross the screen. That is materially worse for the people the rule exists to protect.

All static labels and images therefore carry **`TabIndex: =-1`**, which is the correct marking for a non-interactive element. The checker does not recognise `-1` as satisfying the rule, so the findings persist. When real interactive controls arrive (buttons, inputs, the Pen Input), they get `TabIndex: =0` and a visible focus border, and **their** findings must be taken seriously.

Two related notes:

- The decorative rule on `scrHome` is a 1px `GroupContainer`, not a `Rectangle` — `Rectangle` is treated as interactive and demands an accessible label that a decorative divider should not have.
- `tmrSplashRoute.AutoStart` raises a warning aimed at autoplaying audio and video. This is a routing timer with no media. Ignore it.

Screen-reader support that *does* matter here is in place: images carry `AccessibleLabel`, and the splash status and not-registered heading use `Live: =Live.Polite` so they are announced when they appear.

---

## 7. Verifying contrast

Ratios above come from the WCAG 2.1 formula (relative luminance, `(L1+0.05)/(L2+0.05)`). Thresholds applied: **4.5:1** normal text · **3:1** large text and non-text UI boundaries. Re-run the check whenever a token changes — every value in §2 was computed, not estimated.
