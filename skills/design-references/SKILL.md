---
name: design-references
description: Look at how good apps solve a screen before designing it, with Mobbin's MCP tools (search_screens, search_flows) — when to search, standard mode before deep, one query per screen or flow, the platform picked by layout, the pattern taken and never the brand, no Mobbin image in the repo, and the references listed in the PR. Use when designing a new screen or flow such as sign-in, onboarding, a paywall, settings, checkout, search or an empty state.
---

# Design references

Before designing a new screen or flow, look at how good apps already solve it. Mobbin has real apps' screens and flows; its MCP tools are `search_screens` and `search_flows`. Without them, skip this and design from the project's design system.

## When

- A new screen or flow: sign-in, onboarding, paywall, settings, empty states, checkout, search.
- Not for a bug fix, a technical task, or a small change to a screen that already has its shape.

## How to search

- **Pass `mode: "standard"` to `search_screens`.** The default is `deep`, which spends Mobbin credits: use it only when standard finds nothing useful. Pass `output_destination: "code"`.
- **One screen or one flow per query, with the kind of app**: "fitness app onboarding", "banking app settings screen". Search the flow too when the task is several steps.
- **Pick the platform by layout, not by target.** Mobbin only knows `ios` and `web`. The narrow layout (Android and iOS phones) searches `ios`; the wide one (desktop, web, tablets) searches `web`. A screen whose layout changes at a width limit gets both searches, and each layout is drawn from its own references.
- **Look at the images**, not only the metadata. Three to five apps are enough.

## What to take

- **The pattern, never the brand**: the layout, the order of steps, what is asked when, how empty and error states read. No logos, copy, colours, illustrations or icons from another app.
- **The project's design system wins** (see `design-system`): its components, tokens and spacing, even when a reference does it differently.
- **iOS habits stay on iOS.** Large titles, swipe back, SF Symbols and sheets with a grabber don't carry over to Android: take the structure and draw it with the project's components.
- **Never save Mobbin images in the repo.** They are other companies' screenshots, under Mobbin's licence, and their URLs expire.

## Say what you used

In the PR description, under "Inspired by", list each reference: the app, the screen or flow, its `mobbin_url`, and in one line what you took from it. Reviewers then see why the screen looks the way it does.
