---
id: route-match-loading-phase-3
title: Route Match Loading Phase 3 Guide
---

# Route Match Loading Phase 3 Guide

Status: phase 3 implementation handoff

This document is the standalone handoff for the phase 3 first-principles rewrite of route-match loading and rendering.

An agent should be able to start implementation from this document alone.

## Mission

Replace the current implicit, pool-and-promise-driven route-match loading pipeline with an explicit generation-based lifecycle that:

- preserves the intended observable behavior
- eliminates known race conditions and lifecycle bugs
- maps directly to the phase-2 FizzBee models
- keeps adapter-specific mechanics out of router-core

## Source Of Truth

If current code, existing tests, and old implementation habits disagree, use this priority order:

1. This document
2. `docs/router/route-match-loading-rfc.md`
3. The phase-2 `.fizz` models and traces
4. Current implementation details

This rewrite is spec-first, not legacy-first.

## Phase 3 Goal

Build a new internal route-match lifecycle that makes these semantic concepts explicit:

- route presence identity
- load resource identity
- load generations
- committed tree vs candidate tree
- semantic render gates
- explicit reuse policy
- explicit failure precedence
- explicit hydration boundary semantics

The rewrite should preserve observable semantics, not internal representation.

## What We Learned

### 1. Generations must be first-class

This is the single most important lesson from phase 2.

Old work must be explicitly obsolete-able. Newer generations must not be clobbered by stale completions from:

- old navigations
- preloads
- background stale reloads
- hydration follow-up work

Do not infer obsolescence from pool transitions, promise replacement, or store mutation order.

### 2. Identity must be split into separate concepts

Do not use one implementation key to mean all of the following at once:

- route presence in the visible tree
- reusable loader artifact identity
- storage record identity

Treat these as distinct semantic concepts:

- Route presence identity
  The logical continuity of a route in the committed or candidate tree.
- Load resource identity
  The identity used to decide whether route-loading artifacts may be reused.
- Storage identity
  The implementation-level record/store used to hold runtime state.

### 3. Router-core should emit semantic render facts, not framework mechanics

Router-core should decide things like:

- blocked
- pending
- ready
- error
- notFound
- obsolete/discardable generation

Adapters should decide how to realize those facts:

- React promise throwing and boundary placement
- Solid suspense/resource wiring
- Vue DOM-safe rendering behavior

### 4. Reuse must be policy, not accidental object survival

The rewrite should encode reuse rules explicitly.

Current intended policy:

- `beforeLoad` is conservative preflight/context/authorization work
- `loader` is reusable data acquisition work
- fresh successful loader artifacts may be reused
- in-flight preload loader work may be adopted by navigation
- terminal failure artifacts are not reusable as success data
- later artifact failure must not retroactively poison a navigation that already reused fresh data

### 5. Slice models worked better than one monolithic model

Phase 2 succeeded by isolating the hard seams instead of forcing them into one giant exhaustive graph.

Phase 3 should keep that same structure in code, even if the implementation lives in fewer files.

That means the rewrite should keep explicit sub-lifecycles for:

- preflight / `beforeLoad`
- loader / reusable artifact acquisition
- final outcome resolution
- committed-tree update
- hydration boundary handling
- pending timing
- background stale reload handling

### 6. Context readiness is modeled; full context-value coherence is still the biggest open implementation risk

Phase 2 proved the importance of context completeness before render, but it models readiness more strongly than concrete merged values.

Phase 3 should be especially careful about:

- parent-to-child context visibility
- `beforeLoad` updates becoming render-visible at the right time
- redirecting into a new generation with fresh context
- invalidation and rerun behavior for context-producing routes

If implementation work exposes uncertainty about actual context-value coherence, pause and extend phase 2 rather than guessing.

## What Must Be Preserved

### Committed Tree Coherence

The user-visible tree must update atomically.

Do not expose a partially committed mixture of:

- old and new generations
- old and new route contexts
- old and new terminal outcomes

### Serial `beforeLoad`, Parallel `loader`

- `beforeLoad` runs serially from root to leaf
- loader work begins only after the serial preflight prefix is established
- loader work may run in parallel after that

### Failure Precedence

The semantic rules are:

1. Redirect dominates all non-redirect outcomes for the same generation.
2. Winner selection must be deterministic and must not depend on completion timing.
3. notFound and error rendering are resolved against the boundary that actually renders them.
4. Root notFound must preserve the root shell.
5. Child failure may still require ancestor work so the final boundary can render coherently.

Compatibility baseline already captured in phase 2:

- loader error beats serial notFound when no redirect is present
- loader notFound beats serial error when no redirect or loader error is present
- leaf notFound resolves to parent boundary
- leaf error resolves to parent boundary

### Pending Timing Contract

Pending UI is an observable timing contract, not just a React promise shape.

Must preserve:

- pending is not shown before `pendingMs`
- if the load finishes before pending shows, pending never flashes
- once pending becomes visible, it remains visible for at least `pendingMinMs`

### Hydration Boundary Semantics

The rewrite must preserve the semantic distinction between:

- `SERVER_DOM`
- `DATA_ONLY`
- `CLIENT_ONLY`

Meaning:

- fully SSR-backed hydration should not trigger an unnecessary extra client load
- a `data-only` boundary may start with usable data while still being client-owned for first meaningful DOM
- a `client-only` boundary must not become render-ready early
- hydrated server-owned prefix must remain coherent even if the client-owned suffix fails

### Background Stale Reload Semantics

While stale background work is running:

- the currently committed view must remain renderable
- a newer committed navigation must not be clobbered by late background completion
- background failure must not force visible error UI for the already committed view by itself

### Reuse / Adoption Semantics

Must preserve:

- fresh artifact reuse on navigation
- in-flight preload adoption by navigation
- obsolete older generations never becoming the committed winner
- failure artifacts not being reused as success data

## What Must Not Be Copied From The Old Implementation

These are implementation accidents or legacy shapes, not rewrite goals.

- active/pending/cached pools as semantic truth
- `_nonReactive.*Promise` names as part of the design
- `match.id` string construction as the semantic definition of reuse identity
- `globalNotFound` as a required representation
- hydration heuristics expressed as index-based hacks rather than boundary semantics
- adapter behavior being driven directly by router-core promise lifetime accidents

These may still exist internally if useful, but only as implementation details under clearer semantic state.

## Recommended Internal Architecture

### Core Runtime Concepts

Implement explicit runtime records for these concepts.

1. Route presence record
   Represents a route's presence in a committed or candidate tree.

2. Load resource record
   Represents reusable route-loading artifacts and their freshness.

3. Generation record
   Represents one causally ordered attempt to make a candidate tree render-ready.

4. Render gate record
   Represents semantic visibility state for a route presence in a generation.

### Suggested Generation Record

The implementation does not need these exact names, but it should expose these concepts clearly.

- generation id
- entry type
  Initial load, navigation, preload, hydration follow-up, background reload
- obsolete / live status
- candidate tree
- per-route preflight status
- per-resource loader status
- final resolved outcome
- winning boundary owner, if any
- commit eligibility

### Suggested Resource Record

- resource identity
- acquisition status
- freshness
- reusable success payload
- non-reusable terminal failure marker
- which generations are currently observing or adopting it

### Suggested Render-Gate API

Router-core should produce a small semantic gate surface, for example:

- `blocked`
- `pending`
- `ready`
- `error`
- `notFound`
- `obsolete`

The exact names can differ, but phase 3 should keep the surface small and semantic.

### Suggested Lifecycle Split

Keep the implementation factored around these responsibilities:

1. Candidate-tree construction
2. Preflight runner
3. Resource acquisition / reuse / adoption
4. Final outcome resolver
5. Commit coordinator
6. Hydration coordinator
7. Adapter gate mapping

This is preferable to keeping all logic in one mega-pipeline, even if the old code centered around `loadMatches()`.

## Recommended Rewrite Strategy

### 1. Keep the public API stable while replacing internals

Prefer replacing internal orchestration behind existing public router behavior.

Do not start by redesigning public route APIs.

### 2. Build the new core semantics before rewriting adapter mechanics

Recommended order:

1. introduce generation/resource/presence concepts in router-core
2. reimplement finalization and commit logic around them
3. reimplement preflight and loader orchestration
4. expose semantic render gates to adapters
5. port React adapter to semantic gates
6. align Solid and Vue afterward

### 3. Port React first

React remains the main focus.

The React adapter should consume semantic gates and preserve:

- pending continuity
- redirect discard safety
- root-shell notFound behavior
- hydration boundary behavior

React-specific promise handling should become an adapter implementation of semantic states, not the core lifecycle itself.

### 4. Keep slice-based testability

Every focused `.fizz` model should correspond to focused implementation tests.

Suggested mapping:

- `route-match-hydration.fizz` -> hydration adapter/core tests
- `route-match-overlap.fizz` -> preload/navigation reuse tests
- `route-match-background.fizz` -> stale reload / newer navigation tests
- `route-match-pending.fizz` -> pending timing tests
- `route-match-failures.fizz` -> failure precedence and boundary tests

## Minimum Acceptance Criteria For Phase 3

Before considering the rewrite correct, the implementation should satisfy all of the following:

1. The new internal state machine is generation-based, not promise-lifetime-based.
2. Render-visible route components never observe incomplete effective context.
3. Redirects, notFound, and errors follow the modeled precedence and boundary rules.
4. Fresh reuse and in-flight adoption match the overlap model.
5. Background stale reloads cannot clobber newer committed state.
6. Pending timing matches the pending model.
7. Hydration semantics match the hydration model.
8. React adapter behavior matches semantic render gates instead of inventing its own lifecycle.
9. Existing tests are updated or replaced where they encode accidental legacy behavior.
10. New focused tests exist for each model slice.

## Remaining Doubts

These are the few remaining uncertainties that are real enough to justify more phase-2 work if phase 3 runs into them.

### Doubt 1. Concrete context-value coherence

We modeled context readiness, but not full merged context values.

If phase 3 needs to answer questions like:

- exactly when a child should observe a parent's `beforeLoad` return
- whether route `context()` should rerun in a corner case
- how invalidation changes context value reuse

then extend phase 2 before finalizing the implementation.

### Doubt 2. Invalidation and eviction semantics are weaker than the other slices

The suite covers stale background reload and explicit resource freshness, but it does not yet have a dedicated invalidation/eviction model slice.

If phase 3 work reaches GC, cache eviction, or invalidation corner cases that affect visible lifecycle guarantees, pause and model that slice first.

### Doubt 3. One shared exhaustive model does not exist

The phase-2 suite is intentionally slice-based.

If a phase-3 design depends on subtle interactions across:

- hydration
- pending timing
- background reload
- failure precedence
- reuse/adoption

then consider adding a targeted combined model for that exact interaction, rather than assuming the cross-product is already proven.

### Doubt 4. Adapter-specific continuity is still semantic, not mechanical

The suite proves adapter obligations semantically, not the exact promise choreography inside React/Solid/Vue.

If an adapter rewrite runs into a framework-specific lifecycle puzzle, model or test that mechanism directly instead of forcing router-core to own it.

## Exact Phase 2 Assets

Primary RFC and status:

- `docs/router/route-match-loading-rfc.md`

Core and focused models:

- `docs/router/route-match-loading.fizz`
- `docs/router/route-match-hydration.fizz`
- `docs/router/route-match-overlap.fizz`
- `docs/router/route-match-background.fizz`
- `docs/router/route-match-pending.fizz`
- `docs/router/route-match-failures.fizz`

Useful trace files:

- `docs/router/route-match-overlap-fresh.trace`
- `docs/router/route-match-overlap-adopt.trace`
- `docs/router/route-match-background-late-success.trace`
- `docs/router/route-match-background-failure.trace`
- `docs/router/route-match-pending-fast.trace`
- `docs/router/route-match-pending-slow.trace`
- `docs/router/route-match-failures-root-notfound.trace`
- `docs/router/route-match-failures-leaf-notfound.trace`
- `docs/router/route-match-failures-notfound-redirect.trace`
- `docs/router/route-match-failures-error-vs-loader-notfound.trace`
- `docs/router/route-match-failures-leaf-loader-error.trace`

Current code hotspots to replace or absorb:

- `packages/router-core/src/load-matches.ts`
- `packages/router-core/src/router.ts`
- `packages/router-core/src/ssr/ssr-client.ts`
- `packages/react-router/src/Match.tsx`

## Final Instruction To A Phase 3 Agent

Do not start by reshuffling the current code.

Start by implementing explicit semantic state in router-core that matches the model suite. Then make React consume that state. Only after the semantic core is stable should you align the other adapters and remove obsolete legacy machinery.

If implementation work exposes uncertainty about semantics rather than code mechanics, stop and iterate on phase 2 before shipping the rewrite.
