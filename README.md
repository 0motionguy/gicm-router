# @gicm/router

A four-tier model router for LLM agents: regex intent rules, fallback chains, health tracking and session pinning. It returns a model ID and leaves the API call to you.

`SmartRouter` classifies a message into one of four tiers (0 no LLM, 1 cheap, 2 balanced, 3 premium) and returns the model configured for that tier, skipping models you have marked unhealthy. It is for developers who want cheap requests to stay on cheap models without a classifier call. It does not talk to any provider: you take the returned ID and call OpenRouter, Anthropic, OpenAI, Ollama or whatever you map it to.

Part of [gICM](https://github.com/0motionguy/gICM), a family of OpenClaw agent packages. API reference and examples: [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md).

## What it does

- **Classify:** `classifyIntent(message, rules)` checks tier 0 regexes first (time and date questions, greetings, bare arithmetic), then tier 3 keywords (architect, security audit, cryptography and similar), then tier 2 keywords (refactor, debug, review and similar), and defaults to tier 1.
- **Route:** `route({ message, sessionId?, forceTier? })` returns `{ model, tier, estimatedCostPer1k, reason, fromCache }`. The default tier table lists a primary model, fallbacks and per-1k prices, and you can replace it through the `tiers` option.
- **Fail over:** `setModelHealth(model, false)` called three times marks a model unhealthy; the router then walks the tier's fallback chain and emits `route:fallback` and `health:changed`.
- **Pin sessions:** a session keeps its first model while that model stays healthy (`fromCache: true`).
- **Report:** `getStats()` returns request counts by tier and model, cache hits and a rolling average routing time over the last 100 calls.
- **Shortcut:** a shared `router` instance and a `route()` function for quick use.

## Example

```ts
import { SmartRouter } from "@gicm/router";

const router = new SmartRouter();
const pick = router.route({ message: "Refactor this function", sessionId: "s-1" });
// pick.model is the tier-2 primary; call your provider with it.
router.setModelHealth(pick.model, false); // report a failed call; three strikes mark it unhealthy
```

## How it's built

TypeScript (ESM) bundled with tsup and tested with vitest; zod is the only dependency and the router extends `node:events`. Rules and tier tables are plain data in `src/tiers.ts`, so the default model IDs and prices are a snapshot that will age and are meant to be edited.

## Repository layout

- `src/`: `router`, `intent`, `tiers`, `types`
- `src/__tests__/`: two vitest files (54 test cases)
- `skill/`: `SKILL.md` (the OpenClaw skill definition) and `references/models.md`

## Run it locally

```bash
pnpm install
pnpm build   # tsup -> dist/
pnpm test    # vitest (add --run for a single pass)
```

## Status

Beta. Version 1.0.0, a single extracted release (February 2026) with a test suite. `route()` always uses the default rules (custom rules go through `classifyIntent`), the `ollamaUrl` and `openrouterKey` options are accepted but unused, and no measured cost-saving figures ship with the repo.
