# Contribution 1: Link Embeds in Dark Mode Unreadable

**Contribution Number:** 1
**Student:** Lakshmi Sravani Kondeti
**Issue:** [stoatchat/for-web#669 — Link Embeds in Dark Mode Unreadable](https://github.com/stoatchat/for-web/issues/669)
**Status:** Phase II — In Progress

---

## Why I Chose This Issue

I chose this issue because it involves a clear, reproducible visual bug — link embed cards in dark mode that remain styled as if they were in light mode, causing poor readability. This kind of UI/styling bug is approachable for someone learning open source contribution workflows while also being genuinely impactful to users.

This issue also aligns well with my learning goals for this program. I want to build hands-on experience with TypeScript-based frontend projects and understand how modern design systems like Material Design 3 handle theming. Digging into CSS custom properties, color token systems, and Solid.js component structure gives me practical exposure to skills that are directly relevant to frontend development.

---

## Understanding the Issue

### Problem Description

When a user posts a URL in a chat channel and the app is in dark mode (especially with the **Monochrome** color scheme), the link embed card renders with a light-colored background and white text on top of it — making the content nearly unreadable. The embed card does not adapt to the dark theme the way the rest of the UI does.

### Expected Behavior

The link embed card should always use colors that match the active theme. In dark mode, the embed should display with a dark background and appropriately contrasting (light) text, consistent with the rest of the chat interface.

### Current Behavior

In dark mode with the Monochrome color scheme (and also Tonal Spot, per issue comments), the embed card renders with a **light gray background** while the title text uses the **primary color** which resolves to white in dark mode. This creates a white-text-on-light-gray contrast problem that makes the embed almost unreadable. The issue was first reported on February 12, 2026, and confirmed by a contributor on February 26, 2026 to be specific to the monochrome scheme.

### Affected Components

- **Primary:** `packages/client/components/ui/components/features/messaging/elements/TextEmbed.tsx` — the link preview card component
- **Secondary:** `packages/client/components/ui/components/features/messaging/elements/Attachment.tsx` — file embeds also show styling issues in dark mode (noted separately in issue comments)

---

## Reproduction Process

### Environment Setup

The project is a pnpm monorepo using [mise-en-place](https://mise.jdx.dev/) as the task runner.

**Prerequisites:**
- [Git](https://git-scm.com/install/) with submodule support
- [mise-en-place](https://mise.jdx.dev/getting-started.html) (manages Node + pnpm versions automatically — no manual version switching needed)

**Step-by-step setup:**

```bash
# 1. Clone your fork (with submodules — critical!)
git clone --recursive https://github.com/LKONDETI/for-web
cd for-web

# 2. Install Node.js 26.2.0 + pnpm 11.3.0 via mise (reads from .mise.toml automatically)
mise install:frozen

# 3. Build internal packages (stoat.js, UI components, lingui i18n)
#    This MUST run before the dev server — submodule packages need a build step
mise build:deps

# 4. Configure .env to connect to the official hosted backend (no local server needed)
cp packages/client/.env.example packages/client/.env
# Open packages/client/.env and comment out the local URL variables:
#   #VITE_API_URL=http://localhost:14702
#   #VITE_WS_URL=ws://localhost:14703
#   #VITE_MEDIA_URL=http://localhost:14704
#   #VITE_PROXY_URL=http://localhost:14705
# With these commented out, the client auto-connects to stoat.chat backend

# 5. Start the dev server
mise dev
# Navigate to http://local.revolt.chat:5173
```

**Key challenge:** `mise build:deps` must complete before `mise dev` works. The monorepo's internal packages (`stoat.js`, `solid-livekit-components`, `js-lingui-solid`) are git submodules that require a build step. Skipping this causes the dev server to fail with missing module errors.

### Steps to Reproduce

1. Launch the Stoat web client (local dev or https://stoat.chat/app)
2. Open Settings → Appearance → set Color Scheme to **Monochrome** and Mode to **Dark**
3. Navigate to any chat channel
4. Post a URL (e.g., `https://github.com`) and observe the link embed that appears below the message
5. **Observed result:** The embed card shows a light gray background with white title text — very low contrast, difficult to read

### Reproduction Evidence

- **Commit showing reproduction:** [4bf46dcf](https://github.com/LKONDETI/for-web/commit/4bf46dcf) — current HEAD of fork; the buggy `TextEmbed.tsx` with `primary-container` tokens exists at this state ([view file](https://github.com/LKONDETI/for-web/blob/4bf46dcf/packages/client/components/ui/components/features/messaging/elements/TextEmbed.tsx))
- **Screenshots/logs:** See original issue screenshot — [4d44e3f8](https://github.com/user-attachments/assets/4d44e3f8-e492-4690-92e9-bd74466736d3)
- **My findings:** The `TextEmbed` component uses `--md-sys-color-primary-container` as the card background. In the `SchemeMonochrome` variant of Material Design 3, this token resolves to a **light gray in dark mode** rather than a dark color — the opposite of what we'd expect. The title uses `--md-sys-color-primary` which becomes white in dark mode, causing white-on-light-gray contrast.

---

## Solution Approach

### Analysis

The root cause is in `TextEmbed.tsx` (lines 17–31 and 57–65). The `Base` styled component uses M3 color tokens from the `primary` color role:

```
background: var(--md-sys-color-primary-container)
color:      var(--md-sys-color-on-primary-container)
border:     var(--md-sys-color-primary)
```

And the `Title` component uses:
```
color: var(--md-sys-color-primary) !important
```

In most M3 schemes, these tokens correctly switch between light and dark values. However, in **SchemeMonochrome**, the `primary-container` role generates a **light gray** even in dark mode — because monochrome schemes have limited tonal range and the Material color utilities don't always guarantee a dark container in this variant. The result is a light card in dark mode.

### Proposed Solution

Replace the `primary-container` / `on-primary-container` tokens with **surface-level tokens** — specifically `surface-container-high` (background) and `on-surface` (text). Surface tokens are semantically designed for UI containers and are **guaranteed to be dark in dark mode across all M3 scheme variants**, including Monochrome and Tonal Spot.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The embed card uses primary color role tokens that fail to produce a dark background in the Monochrome M3 scheme, resulting in an unreadable light card in dark mode.

**Match:** Looking at `Attachment.tsx`, file embeds use `inverse-surface` / `inverse-on-surface` tokens. Another option seen in `Container.tsx` is `surface-container` for hover states. The `surface-container-high` / `on-surface` pair is a well-established M3 pattern for elevated UI card containers.

**Plan:**
1. In `TextEmbed.tsx`, update `Base` styled component:
   - Change `background` from `var(--md-sys-color-primary-container)` → `var(--md-sys-color-surface-container-high)`
   - Change `color` from `var(--md-sys-color-on-primary-container)` → `var(--md-sys-color-on-surface)`
2. Update `Title` styled component:
   - Verify `var(--md-sys-color-primary)` still provides readable contrast on the new background; adjust to `var(--md-sys-color-primary)` or `var(--md-sys-color-on-surface-variant)` as needed
3. Manually test all theme variants (Monochrome, Tonal Spot, default) in both light and dark modes
4. Verify the left accent border still renders correctly with `--md-sys-color-primary`

**Implement:** Working in fork [LKONDETI/for-web](https://github.com/LKONDETI/for-web) — will link feature branch and commits here as work progresses

**Review:**
- [ ] No hardcoded colors — only CSS variable tokens used
- [ ] Follows 2-space indentation style
- [ ] No prop destructuring (Solid.js pattern — using `splitProps` where needed)
- [ ] Semantic color token usage consistent with rest of codebase

**Evaluate:** Visual check in all combinations of (Monochrome, Tonal Spot, Vibrant) × (Light, Dark) modes. Embed should be readable in all cases.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: No existing embed unit tests — document this gap for the PR
- [ ] Test case 2: Verify correct CSS variable names are applied in the component
- [ ] Test case 3: N/A (visual contrast cannot be fully unit-tested)

### Integration Tests

- [ ] Verify embed renders correctly in Monochrome dark mode (manual + screenshot)
- [ ] Verify embed renders correctly in Tonal Spot dark mode (manual + screenshot)

### Manual Testing

Testing will be done by running the local dev server (`mise dev`), posting links in a test channel, and toggling through all theme variants + dark/light mode combinations. Screenshots will be attached to the PR comparing before and after.

---

## Implementation Notes

### Week 1 Progress

Explored the codebase to understand the project structure and locate the relevant embed component. Identified the root cause in `TextEmbed.tsx` — the `primary-container` color token resolves to light gray in Monochrome dark mode. Reviewed how the Material Design 3 theme system works in this codebase (`materialTheme.ts`, `stoatWebTheme.ts`, `LoadTheme.tsx`). Set up and understood the development environment (pnpm monorepo + mise task runner).

### Week 2 Progress

Successfully stood up the local development environment using `mise install:frozen` → `mise build:deps` → `mise dev`. Configured `packages/client/.env` to connect to the official stoat.chat backend (commented out local URL variables), so no local Stoat server was needed.

**Bug Reproduction — confirmed:**
1. Opened http://local.revolt.chat:5173 and logged in
2. Went to **Settings → Appearance** → set Color Scheme to **Monochrome** + Mode to **Dark**
3. Posted a URL (e.g., `https://github.com`) in a chat channel
4. Observed the link embed card: **light gray background with white title text** — near-zero contrast, unreadable

**Root cause confirmed in `TextEmbed.tsx`:**
- `Base` component (lines 17–31) uses `--md-sys-color-primary-container` as background → resolves to light gray in Monochrome dark mode
- `Title` component (line 63) uses `--md-sys-color-primary` → resolves to white in dark mode
- Result: white text on a light gray card = unreadable

**Fix planned — token swap in `TextEmbed.tsx`:**

| Property | Token (current) | Token (proposed) | Why |
|---|---|---|---|
| `background` | `--md-sys-color-primary-container` | `--md-sys-color-surface-container-high` | Reliably dark in all M3 dark variants |
| `color` | `--md-sys-color-on-primary-container` | `--md-sys-color-on-surface` | Correct contrast pair for surface tokens |
| `border` | `--md-sys-color-primary` | keep as-is | Accent color, readable on dark surface |
| Title `color` | `--md-sys-color-primary` | keep as-is (verify) | Link color — verify contrast after fix |

Exact diff (lines 27–28 of `TextEmbed.tsx`):
```diff
- color: "var(--md-sys-color-on-primary-container)",
- background: "var(--md-sys-color-primary-container)",
+ color: "var(--md-sys-color-on-surface)",
+ background: "var(--md-sys-color-surface-container-high)",
```

Next step: implement the change, verify visually across all theme variants (Monochrome, Tonal Spot, Vibrant) × (Light, Dark), then open a PR.

### Code Changes

- **Files to modify:** `packages/client/components/ui/components/features/messaging/elements/TextEmbed.tsx` (lines 27–28)
- **Key commits:** TBD — implementation pending
- **Approach decisions:** Chose `surface-container-high` / `on-surface` over `inverse-surface` / `inverse-on-surface` because inverse tokens are designed for floating elements (snackbars, tooltips) that need to visually stand out against the surface — not for embedded cards within the content flow. `surface-container-high` is the semantically correct M3 token for elevated card containers, and it is guaranteed to produce a dark background in all M3 dark mode variants including Monochrome.

---

## Pull Request

**PR Link:** TBD

**PR Description:** TBD — will adapt content from the Solution Approach and Testing Strategy sections above

**Maintainer Feedback:**
- TBD

**Status:** Not yet submitted

---

## Learnings & Reflections

### Technical Skills Gained

- How Material Design 3's dynamic color system works — specifically how `SchemeMonochrome`, `SchemeTonalSpot`, etc. generate color tokens differently
- How Solid.js + Panda CSS styled components work together for component-level theming
- How CSS custom properties (`--md-sys-color-*`) are generated at runtime and applied globally
- How pnpm monorepos and mise task runners organize multi-package projects

### Challenges Overcome

TBD — will document challenges encountered during implementation

### What I'd Do Differently Next Time

TBD — will reflect after the PR is submitted and reviewed

---

## Resources Used

- [Material Design 3 Color System](https://m3.material.io/styles/color/system/overview)
- [stoatchat/for-web GitHub Issue #669](https://github.com/stoatchat/for-web/issues/669)
- [@material/material-color-utilities documentation](https://github.com/material-foundation/material-color-utilities)
- [Solid.js documentation — splitProps](https://www.solidjs.com/docs/latest#splitprops)
- [Panda CSS styled components](https://panda-css.com/docs/concepts/styled-system)
