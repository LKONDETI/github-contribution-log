# Plan: Fix Link Embeds in Dark Mode — Issue #669

## Context

Lakshmi Sravani Kondeti is working on Contribution #1 for CodePath's AI301 open source program.
- **Issue:** [stoatchat/for-web#669](https://github.com/stoatchat/for-web/issues/669) — "Link Embeds in Dark Mode Unreadable"
- **Status:** Phase II — Bug reproduced, fix planned, implementation pending.

---

## Root Cause

In `packages/client/components/ui/components/features/messaging/elements/TextEmbed.tsx`, the `Base` styled component uses Material Design 3 color tokens from the **primary** color role:

```
background: var(--md-sys-color-primary-container)   ← line 28
color:      var(--md-sys-color-on-primary-container) ← line 27
border:     var(--md-sys-color-primary)               ← line 29
```

And the `Title` component uses:
```
color: var(--md-sys-color-primary) !important   ← line 63
```

In most M3 schemes these tokens correctly switch between light/dark. However, in **SchemeMonochrome** dark mode, `primary-container` generates a **light gray** — the same as light mode — because the Monochrome scheme has limited tonal range and doesn't guarantee a dark container. The `primary` token resolves to **white** in dark mode.

Result: white text on a light gray background = unreadable.

---

## Proposed Fix

Replace `primary-container` tokens with **surface-level tokens** in `TextEmbed.tsx`:

| Property | Token (before) | Token (after) | Why |
|---|---|---|---|
| `background` | `--md-sys-color-primary-container` | `--md-sys-color-surface-container-high` | Guaranteed dark in all M3 dark variants |
| `color` | `--md-sys-color-on-primary-container` | `--md-sys-color-on-surface` | Correct contrast pair for surface tokens |
| `border` | `--md-sys-color-primary` | keep as-is | Accent color, readable on dark surface |
| Title `color` | `--md-sys-color-primary` | keep as-is (verify) | Link color — verify contrast post-fix |

### Exact diff (TextEmbed.tsx, lines 27–28)

```diff
- color: "var(--md-sys-color-on-primary-container)",
- background: "var(--md-sys-color-primary-container)",
+ color: "var(--md-sys-color-on-surface)",
+ background: "var(--md-sys-color-surface-container-high)",
```

**Why `surface-container-high` over `inverse-surface`?**
Inverse tokens (`inverse-surface` / `inverse-on-surface`) are designed for floating elements (snackbars, tooltips) that need to stand out visually against the main surface. `surface-container-high` is the semantically correct M3 token for elevated card containers within the content flow — and it's guaranteed dark in all M3 dark mode variants including Monochrome and Tonal Spot.

---

## Local Environment Setup

The project is a pnpm monorepo using [mise-en-place](https://mise.jdx.dev/) as the task runner.

**Prerequisites:**
- [Git](https://git-scm.com/install/) with submodule support
- [mise-en-place](https://mise.jdx.dev/getting-started.html) (manages Node + pnpm versions automatically)

**Step-by-step:**

```bash
# 1. Clone your fork (with submodules — critical!)
git clone --recursive https://github.com/LKONDETI/for-web
cd for-web

# 2. Install Node.js 26.2.0 + pnpm 11.3.0 via mise
mise install:frozen

# 3. Build internal packages (stoat.js, UI components, lingui i18n)
#    MUST run before the dev server — submodule packages need a build step
mise build:deps

# 4. Configure .env to connect to official hosted backend (no local server needed)
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

> **Key challenge:** `mise build:deps` must complete before `mise dev` works. Skipping it causes the dev server to fail with missing module errors.

---

## Reproduction Steps

1. Open http://local.revolt.chat:5173 and log in
2. Go to **Settings → Appearance** → Color Scheme: **Monochrome** → Mode: **Dark**
3. Navigate to any chat channel
4. Post a URL (e.g., `https://github.com`)
5. Observe the link embed card — **light gray background + white title text** = unreadable

---

## Implementation Steps

1. Open `packages/client/components/ui/components/features/messaging/elements/TextEmbed.tsx`
2. On line 27, change `on-primary-container` → `on-surface`
3. On line 28, change `primary-container` → `surface-container-high`
4. Run `mise dev` and verify visually

---

## Verification Checklist

- [ ] Monochrome + Dark: embed card is dark, text is readable
- [ ] Tonal Spot + Dark: embed card still looks correct (no regression)
- [ ] Monochrome + Light: embed card still looks correct
- [ ] Default + Light: embed card still looks correct
- [ ] No hardcoded colors — only CSS variable tokens used
- [ ] Follows 2-space indentation style
- [ ] No prop destructuring (Solid.js pattern)

---

## Files to Modify

- `packages/client/components/ui/components/features/messaging/elements/TextEmbed.tsx` (lines 27–28)

## PR

- Fork: [LKONDETI/for-web](https://github.com/LKONDETI/for-web)
- Branch: TBD
- PR: TBD
