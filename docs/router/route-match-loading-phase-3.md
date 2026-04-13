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

This is a first-principles rewrite.

It is not a refactor that incrementally preserves the structure of the current implementation.

The current code should be treated as an observable legacy implementation to learn from, not as the scaffold for the new design.

In particular:

- `packages/router-core/src/load-matches.ts` should probably be replaced in its entirety, not patched into compliance
- `packages/react-router/src/Match.tsx` should be re-expressed around semantic render gates, not preserved as a throw-heavy decision tree
- old helper structure, promise naming, and control-flow shape should not be carried forward just because they already exist

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

The target outcome is a clean and readable implementation that looks like the models and the semantics we now understand, not a compatibility layer spread across the old architecture.

## Rewrite Posture

Use the existing code to answer questions like:

- what observable behavior exists today?
- which tests and edge cases need to be preserved intentionally?
- where are the current bugs and accidental behaviors coming from?

Do not use the existing code as the structural blueprint for the rewrite.

Phase 3 should prefer:

- new explicit lifecycle/state modules over reusing `load-matches.ts` helper clusters
- direct semantic state over implicit promise choreography
- small readable adapter mappings over large nested runtime branching

Phase 3 should avoid:

- duct-taping generation semantics on top of the current `load-matches.ts` shape
- preserving legacy pool/promise coupling as the organizing principle
- keeping React rendering control flow in its current many-throw form if semantic gates make a simpler structure possible

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

The React consequence should be explicit: once router-core owns render gating semantically, `packages/react-router/src/Match.tsx` should not need dozens of distinct `throw`-driven control-flow paths.

Some throws will still be appropriate because React Suspense and error boundaries require them, but the rewrite should aim for a small number of clearly justified throws that directly correspond to semantic gate transitions.

### 4. Reuse must be policy, not accidental object survival

The rewrite should encode reuse rules explicitly.

Current intended policy:

- `beforeLoad` is conservative preflight/context/authorization work
- `loader` is reusable data acquisition work
- fresh successful loader artifacts may be reused
- fresh cached terminal outcomes may be reused as the same terminal outcome when policy allows
- in-flight preload loader work may be adopted by navigation
- terminal outcomes are cacheable under the same cache/eviction rules as successful matches
- terminal outcomes are not reusable as success data
- later artifact failure must not retroactively poison a navigation that already reused fresh data

This distinction is important:

- cacheable terminal outcome != reusable success payload
- a cached `redirect`, `notFound`, or regular `error` may be reused as that same terminal outcome
- it must never be reinterpreted as a successful data result

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

## Phase 3 Retrospective

This section is a direct implementation retrospective from doing phase 3 work for real.

Keep it in mind while implementing. It is the shortest path away from rebuilding the old architecture in a different shape.

### The problem is really about committed-tree semantics

The hard problem is not "run loaders correctly".

The hard problem is:

- build candidate trees coherently
- keep preflight, loader reuse, and failure resolution separate
- commit the visible tree atomically
- ensure render-visible state never observes a half-old/half-new mixture

If the implementation is centered on match mutation, promise choreography, or cache pool transitions, it is already drifting back toward the old design.

### `RouteMatch` should be treated as a visible-tree projection, not the runtime

This is the single most important practical lesson from phase-3 implementation work.

`RouteMatch` is a reasonable adapter-facing representation of the committed tree.

It is a bad place to store:

- generation state
- resource cache state
- pending timers
- suspense bookkeeping
- hydration ownership bookkeeping
- shell-level notFound semantics

When `RouteMatch` becomes the runtime, code volume grows, state duplication grows, and reuse/obsolescence bugs become much harder to reason about.

### Preserve the two param semantics explicitly

Implementation work exposed a crucial distinction that should stay explicit:

- `params`
  The cumulative parsed param bag for the active branch.
- `_strictParams`
  The route-local parsed params for the current route only.

This distinction matters for:

- `useParams({ strict: false })`
- remount stability
- child-route navigation where only descendant params change

If these are accidentally collapsed into one representation, regressions appear quickly.

### `beforeLoad` and `loader` are different phases, but error handling must still feel uniform

`beforeLoad` is serial preflight work and `loader` is reusable resource work.

But observable route error behavior should still remain coherent.

In particular, implementation work showed that these expectations must be preserved explicitly:

- a route `beforeLoad` error still renders through that route's `errorComponent`
- search/param validation failures still participate in route `onError`
- `beforeLoad` errors should not silently degrade into generic root-boundary behavior

Do not let the internal phase distinction leak into user-visible boundary behavior by accident.

### Root-shell notFound is a render semantic, not a match field

The old `globalNotFound` style of representation is a smell.

Root-shell notFound really means:

- the root shell remains mounted
- the next outlet renders notFound content

That should be modeled as runtime or committed-tree semantic state, not as a boolean permanently attached to a match record.

### Start with an acceptance matrix, not code

Phase 3 goes much better when implementation starts from an explicit matrix of acceptance criteria and focused tests.

Do this before writing substantial code:

1. map each phase-3 acceptance criterion to at least one focused test file
2. identify which current tests are observable-intent tests and which only encode deleted internals
3. run the full `router-core` and `react-router` unit suites early, not only targeted slices

Without this, it is too easy to "pass the focused rewrite tests" while regressing real observable behavior elsewhere.

### Distinguish intent tests from internal-shape tests

Implementation work exposed two very different classes of failures.

Tests that still matter because they assert real behavior:

- route error rendering
- invalid search / `onError` behavior
- same-route remount stability
- parsed param visibility
- pending timing

Tests that should be rewritten or deleted when they only assert deleted architecture:

- match-pool presence
- cached/pending id arrays
- pool-specific store behavior
- exact compatibility helpers that phase 3 intentionally removed

If a failing test is about observable behavior, fix the implementation.

If a failing test is about deleted architecture, rewrite the test.

### Exact update counts are fragile; upper bounds are better

Store-update count tests can still be useful, but exact-count assertions are brittle across a rewrite like phase 3.

If update counts go down, that is usually a good outcome, not a regression.

### Practical anti-patterns to avoid immediately

If you see yourself doing any of the following, stop and reset:

- recreating `pending`/`cached` match pools under new names
- adding lifecycle promises/timers back onto `RouteMatch`
- introducing helper methods whose only job is to preserve deleted architecture for tests or devtools
- adding wrappers like `cloneMatch` that do not reduce semantic complexity
- splitting one large file into several files without reducing state duplication or branching
- keeping both runtime truth and match truth for the same concept unless there is a very strong reason

### Good implementation direction

The best direction observed in phase-3 work was:

- runtime records own lifecycle truth
- committed visible tree is projected from runtime state
- adapters consume semantic gates and committed-tree state only
- tests are organized by model slice, not by helper cluster

The more the code looks like that, the better phase 3 goes.

### React should consume render plans, not infer lifecycle from raw state

One of the clearest implementation lessons from phase 3 is that `packages/react-router/src/Match.tsx` stays complicated if router-core only exposes low-level ingredients and expects React to reconstruct the lifecycle from them.

If React still has to infer behavior from combinations of:

- `match.status`
- `match.error`
- `match.ssr`
- shell-level notFound state
- suspense promise lookup
- route option branching

then the adapter still contains too much lifecycle intelligence.

That is a smell in the state model, not just in the adapter.

The desired direction is:

- router-core decides the semantic render situation
- adapters only map that semantic render situation to framework primitives

For React specifically, the best target is something conceptually closer to a render plan such as:

- ready
- pending with suspense promise
- error with boundary owner
- notFound with boundary owner and shell-preservation fact
- obsolete / discardable

React will still need some complexity for:

- Suspense
- error boundaries
- client-only / data-only / server-owned hydration behavior
- remount key calculation

But it should not have to reverse-engineer lifecycle semantics from raw router state.

If `Match.tsx` still feels like a mini state machine, phase 3 has not pushed enough semantic ownership into router-core.

### Route-match code should read like a procedure, not a helper forest

Another major implementation lesson is that simply splitting a large file into many `route-match-*` files does not automatically count as simplification.

Formal models look simple because they express a small number of state transitions and omit many implementation details.

A good phase-3 implementation should still aim to read like a step-by-step procedure:

1. build candidate tree
2. run serial preflight
3. decide resource mode for each route
4. await required work
5. resolve terminal winner and boundary owner
6. commit visible tree
7. let adapters realize semantic gates

If the implementation instead becomes a large collection of helper functions and exports with state spread across many files, that is usually a sign of one of these problems:

- duplicated semantic state
- too many side channels for the same concept
- core steps not being modeled explicitly enough
- file splitting used as an organizational tactic without actually reducing branching or duplication

The route-match problem is genuinely subtle, but it should still read as a small lifecycle with a few explicit phases.

Prefer:

- a small number of procedural phase functions
- one explicit resource mode decision per route/resource
- one explicit final outcome / boundary-owner decision
- one explicit commit step

Avoid:

- many exported helpers that only move code around without deleting state
- repeated "patch match and return" logic across multiple files
- multiple representations of the same fact unless the distinction is semantically necessary

If a route-match subsystem grows across many files and still does not feel like a readable lifecycle, treat that as a design warning, not as an acceptable cost of the problem.

### Keep one source of truth per concept

A repeated implementation failure mode was keeping the same semantic fact in multiple places at once.

Examples of facts that should have one clear owner:

- route context for the current generation
- shell-level notFound ownership
- whether a reusable resource is currently cached or in flight
- which generation currently owns visible state

If phase 3 introduces both:

- a runtime-level truth, and
- a match/store/helper-level mirror of the same truth

then the burden of keeping them synchronized usually outweighs the convenience.

Prefer one owning representation and derive everything else as late as possible.

### If React still feels like a mini runtime, core modeling is not finished

The React adapter is an excellent smell detector.

If `packages/react-router/src/Match.tsx` still has to do substantial work to:

- discover the correct thing to suspend on
- decide whether to throw an error or render one
- decide whether shell notFound applies
- infer boundary ownership from raw match state

then router-core is still exporting raw ingredients instead of semantic render facts.

Do not accept a complicated adapter as inevitable too early.

The right question is not "how do we simplify `Match.tsx` by reorganizing it?"

The right question is "what semantic render plan is router-core still failing to provide?"

### Full-package tests catch important intent that slice tests will miss

Focused slice tests are necessary but not sufficient.

During implementation, several important regressions only showed up when running the full package unit suites.

In particular, these tests caught real observable regressions and should be treated as phase-3 guardrails, not noise:

- `packages/react-router/tests/errorComponent.test.tsx`
  Caught incorrect `beforeLoad` error rendering behavior.
- `packages/react-router/tests/link.test.tsx`
  Caught invalid search / `onError` regression.
- `packages/react-router/tests/router.test.tsx`
  Caught same-route remount stability regressions.
- `packages/react-router/tests/useParams.test.tsx`
  Caught accidental collapse of cumulative params vs route-local strict params.
- `packages/react-router/tests/store-updates-during-navigation.test.tsx`
  Helped distinguish good reductions in update count from brittle exact-count assertions.

Phase 3 should run full `router-core` and `react-router` unit suites early and repeatedly, not only after the rewrite feels complete.

### Preserve observable behavior even when internal phase distinctions change

Implementation work repeatedly showed that internal distinctions can be semantically correct but still produce wrong user-visible behavior if boundary rules are not carried through.

Examples that must stay explicit:

- `beforeLoad` and `loader` are different phases internally, but route-level error rendering should still feel coherent
- route-local parsed params and cumulative params are different internally, but hooks must still expose the right public shape
- root-shell notFound is not a match field, but the adapter must still preserve the same shell behavior

Do not let internal refactoring leak through as a changed public behavior unless the change is intentional and specified.

### Start from a short procedural sketch before writing code

Before implementing phase 3, write a small pseudocode outline for the lifecycle.

For example:

1. build candidate tree
2. derive route-local context inputs
3. run serial preflight
4. decide resource mode per route
5. run/adopt/reuse required resource work
6. resolve winner and boundary owner
7. commit visible tree and semantic gates

If the code stops matching a sketch like this, it is usually a sign that too much logic has leaked into helpers or side channels.

### Prefer deleting state over renaming state

A common trap is replacing old state with new state that means the same thing under a different name.

Good phase-3 progress is measured by deleting categories of state entirely, for example:

- deleting pools rather than renaming pools
- deleting match-level lifecycle promises rather than renaming them
- deleting shell-level match flags rather than moving them to another match field

If the number of semantic concepts is not going down, the rewrite is probably not simplifying enough.

### Concrete startup checklist for the next implementation attempt

Before doing substantial code changes:

1. Write the acceptance matrix from `## Minimum Acceptance Criteria For Phase 3` into concrete test files.
2. Run full `router-core` and `react-router` unit suites once to establish baseline failures.
3. List which current failures are intent regressions vs deleted-internal tests.
4. Write a short procedural lifecycle sketch.
5. Decide the single owning representation for:
   - route context
   - shell-level notFound
   - cached resource state
   - visible committed tree

If any of those are undecided, phase 3 is not ready to be implemented cleanly.

## Must-Save Decisions

If implementation is restarted from scratch, preserve these conclusions.

- `RouteMatch` should remain a visible-tree projection, not the runtime.
- Do not restore match pools.
- Do not restore promise/timer state on matches.
- Root-shell notFound should not be modeled as a match field.
- `params` and `_strictParams` must remain distinct concepts.
- `beforeLoad` errors still need route-level `errorComponent` behavior.
- search/param validation failures still need route `onError` behavior.
- terminal cache entries must remain terminal and never become success data.
- invalidation-triggered reloads should remain blocking, not background reuse.

## Focused Test Matrix

These are the focused test categories that most directly cover the phase-3 acceptance criteria.

1. Overlap / adoption
   Fresh success reuse, in-flight preload adoption, obsolete preload discard.
2. Terminal cache
   Cached `redirect`, `notFound`, and `error` reuse as the same terminal kind.
3. Background reload
   Late background completion never clobbers newer committed state.
4. Pending timing
   `pendingMs`, no flash, `pendingMinMs` continuity.
5. Failures / boundaries
   Redirect precedence, loader error vs notFound precedence, boundary ownership.
6. Hydration modes
   `SERVER_DOM`, `DATA_ONLY`, `CLIENT_ONLY` semantics.
7. Context coherence
   Child visibility of parent `beforeLoad` context and invalidation behavior.
8. Adapter gate behavior
   React/Solid/Vue consume semantic render facts rather than inventing lifecycle.

## Remaining Simplification Opportunities

Even with a correct phase-3 implementation, these areas are worth simplifying further if they still grow too large.

- resource acquisition / reuse logic
- shell-level notFound representation
- route context runtime plumbing
- duplicated freshness bookkeeping that could be derived from age plus invalidation

But simplification should still follow the same rule:

reduce semantic duplication first, not just file size.

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
- fresh terminal-outcome reuse on navigation
- in-flight preload adoption by navigation
- obsolete older generations never becoming the committed winner
- terminal outcomes remaining cacheable under the same rules as non-terminal matches
- terminal outcomes not being reused as success data

## What Must Not Be Copied From The Old Implementation

These are implementation accidents or legacy shapes, not rewrite goals.

- active/pending/cached pools as semantic truth
- `_nonReactive.*Promise` names as part of the design
- `match.id` string construction as the semantic definition of reuse identity
- `globalNotFound` as a required representation
- hydration heuristics expressed as index-based hacks rather than boundary semantics
- adapter behavior being driven directly by router-core promise lifetime accidents
- the overall control-flow structure of `packages/router-core/src/load-matches.ts`
- the current throw-heavy branching structure in `packages/react-router/src/Match.tsx`

These may still exist internally if useful, but only as implementation details under clearer semantic state.

The default assumption should be replacement, not preservation.

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
- reusable terminal outcome payload, if the current cached artifact is terminal
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

If a proposed implementation still looks like a cleaned-up version of `load-matches.ts` rather than an explicit lifecycle built around generations, resources, outcomes, and render gates, it is probably too close to the old design.

## Recommended Rewrite Strategy

### 1. Keep the public API stable while replacing internals

Prefer replacing internal orchestration behind existing public router behavior.

Do not start by redesigning public route APIs.

But do replace internal loading/rendering orchestration aggressively if needed. The right bias is to write new internals that satisfy the models, then delete obsolete legacy machinery, not to incrementally massage the old machinery into shape.

### 2. Build the new core semantics before rewriting adapter mechanics

Recommended order:

1. introduce generation/resource/presence concepts in router-core
2. build a new lifecycle/orchestration path around them instead of extending `load-matches.ts`
3. reimplement finalization and commit logic around the new lifecycle
4. reimplement preflight and loader orchestration
5. expose semantic render gates to adapters
6. port React adapter to semantic gates with a much simpler throw surface
7. align Solid and Vue afterward

### 3. Port React first

React remains the main focus.

The React adapter should consume semantic gates and preserve:

- pending continuity
- redirect discard safety
- root-shell notFound behavior
- hydration boundary behavior

React-specific promise handling should become an adapter implementation of semantic states, not the core lifecycle itself.

The adapter target is readability and directness.

With 20-20 hindsight, the rewritten `packages/react-router/src/Match.tsx` should be dramatically simpler than the current version. It should map semantic gate states to a small number of React mechanisms instead of embedding a large distributed control-flow machine in render.

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
5. Cached terminal outcomes follow the same cache/reuse/eviction rules as non-terminal matches, while preserving their terminal kind.
6. Background stale reloads cannot clobber newer committed state.
7. Pending timing matches the pending model.
8. Hydration semantics match the hydration model.
9. React adapter behavior matches semantic render gates instead of inventing its own lifecycle.
10. Existing tests are updated or replaced where they encode accidental legacy behavior.
11. New focused tests exist for each model slice.

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
- `docs/router/route-match-terminal-cache.fizz`

Useful trace files:

- `docs/router/route-match-overlap-fresh.trace`
- `docs/router/route-match-overlap-adopt.trace`
- `docs/router/route-match-background-late-success.trace`
- `docs/router/route-match-background-failure.trace`
- `docs/router/route-match-pending-fast.trace`
- `docs/router/route-match-pending-slow.trace`
- `docs/router/route-match-terminal-cache-redirect.trace`
- `docs/router/route-match-terminal-cache-notfound.trace`
- `docs/router/route-match-terminal-cache-error.trace`
- `docs/router/route-match-terminal-cache-stale-reload.trace`
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

These files should be treated as legacy reference material, not as the skeleton of the new implementation.

`packages/router-core/src/load-matches.ts` in particular should be assumed replaceable in full unless a very small isolated fragment is clearly worth carrying over.

## Final Instruction To A Phase 3 Agent

Do not start by reshuffling the current code.

Never access anything through router.state or useRouterState for internal adapter/runtime work, these are low performance public API only.

Start by implementing explicit semantic state in router-core that matches the model suite. Then make React consume that state. Only after the semantic core is stable should you align the other adapters and remove obsolete legacy machinery.

If you find yourself preserving the broad structure of `load-matches.ts` or reproducing the current many-branch `Match.tsx` control flow, stop and reset toward a cleaner first-principles design.

If implementation work exposes uncertainty about semantics rather than code mechanics, stop and iterate on phase 2 before shipping the rewrite.
