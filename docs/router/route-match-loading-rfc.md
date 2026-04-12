---
id: route-match-loading-rfc
title: Route Match Loading RFC
---

# Route Match Loading And Rendering RFC

Status: phase 1 complete, phase 2 complete, phase 3 ready to start

Owner: multi-session effort, React-first

Decision defaults:

- Source of truth: the new spec wins over current code/tests when they disagree
- Phase 1 artifact: this checked-in RFC/doc is the coordination hub
- Phase 3 rollout: rework router-core plus React first, then align Solid and Vue

## Purpose

This document is the working coordination hub for redesigning how TanStack Router loads and renders route matches.

The current implementation works, but the behavior is spread across several files, pools, framework adapters, timers, and promise lifetimes. We have repeatedly hit bugs and races around match context visibility, pending UI continuity, redirects, not-found handling, hydration, background reloads, and cache promotion/eviction.

This effort is split into three phases:

1. Specify how route-match loading and rendering should work.
2. Model the required semantics in FizzBee.
3. Re-implement from first principles against the model.

This RFC captures the plan and also starts phase 1.

## Problem Statement

Representative failure modes already seen in the current system:

- a route match renders before the effective route context is complete, so data from `beforeLoad` is initially `undefined`
- a promise consumed by a framework adapter disappears while the match is still mounted
- a match is evicted or replaced before navigation/rendering semantics are actually finished with it
- a redirect/not-found/error from an older or background load leaks into a newer render generation
- hydration-only flags and render-driven timers leak framework details into router-core state

The core challenge is that the router must coordinate all of the following at once:

- initial client load
- SSR hydration
- client-only hydration gaps (`ssr: false`, `ssr: 'data-only'`)
- navigation
- preload
- invalidation and stale reloads
- redirect and not-found interruption
- lazy route chunk loading
- framework-specific suspense/error-boundary behavior

## Primary Code Map

Current behavior is spread across these files.

Core loading and state:

- `packages/router-core/src/router.ts:1389-1668`
  Match creation, reuse, initial context assembly.
- `packages/router-core/src/router.ts:1758-1787`
  Match cancellation.
- `packages/router-core/src/router.ts:2327-2860`
  `beforeLoad`, `load`, pending-to-active commit, invalidation, preloading, cache cleanup.
- `packages/router-core/src/load-matches.ts:23-1278`
  The main loading pipeline: serial `beforeLoad`, parallel loaders, pending timers, redirect/notFound/error handling, head execution, lazy chunk loading.
- `packages/router-core/src/Matches.ts:118-175`
  Match shape and `_nonReactive` promise/timer state.
- `packages/router-core/src/stores.ts:72-340`
  Active, pending, and cached match pools and their store reconciliation rules.
- `packages/router-core/src/ssr/ssr-client.ts:19-309`
  Client hydration, forced pending, display-pending, rehydrated context/head reconstruction.

React rendering:

- `packages/react-router/src/Matches.tsx:46-113`
  Root-level suspense/catch setup.
- `packages/react-router/src/Match.tsx:25-568`
  Per-match suspense, catch, not-found routing, client-only rendering, `MatchInner`, and `Outlet` behavior.
- `packages/react-router/src/CatchBoundary.tsx:7-116`
  Error-boundary reset semantics.
- `packages/react-router/src/Transitioner.tsx:13-139`
  Transition integration and `onLoad`/`onResolved`/`onRendered` timing.

Other adapters:

- `packages/solid-router/src/Match.tsx:21-507`
- `packages/vue-router/src/Match.tsx:26-564`

Tests worth using as anchors while specifying behavior:

- `packages/router-core/tests/load.test.ts:574-650`
  `beforeLoad` reruns on stay, loaders may not.
- `packages/router-core/tests/load.test.ts:1170-1381`
  beforeLoad/notFound hierarchy, ancestor loader/head behavior, serial error behavior.
- `packages/router-core/tests/hydrate.test.ts:222-350`
  hydration of `globalNotFound`.
- `packages/react-router/tests/routeContext.test.tsx:447-574`
  expected context visible in `beforeLoad`.
- `packages/react-router/tests/loaders.test.tsx:468-648`
  route context and loader data coherence through navigation and invalidation.
- `packages/react-router/tests/router.test.tsx:2337-2537`
  beforeLoad notFound with and without pending components.
- `packages/react-router/tests/redirect.test.tsx:116-125`
  redirect while pending UI is already showing.

## Current Lifecycle Map

This section describes the current implementation, not the target design.

### 1. Match creation

- `router.beforeLoad()` cancels pending work and computes the next pending matches.
- `matchRoutesInternal()` creates or reuses a `match.id` from `route.id + interpolatedPath + loaderDepsHash`.
- New matches start in `status: 'pending'` if they have `beforeLoad`, `loader`, lazy route options, or preloadable components.
- Reused matches carry forward substantial state, including cached data and `_nonReactive` state.

References:

- `packages/router-core/src/router.ts:1502-1600`
- `packages/router-core/src/router.ts:2327-2371`

### 2. Loading

- `loadMatches()` runs `beforeLoad` serially from root to leaf.
- `loadRouteMatch()` then runs loaders in parallel for the surviving prefix.
- `beforeLoad` may set `__beforeLoadContext`, redirect, notFound, or error.
- Loaders may reuse fresh data, reload in the foreground, or stale-reload in the background.
- Heads run after loading/error/notFound resolution.

References:

- `packages/router-core/src/load-matches.ts:355-558`
- `packages/router-core/src/load-matches.ts:641-959`
- `packages/router-core/src/load-matches.ts:961-1185`

### 3. Commit

- `router.load()` promotes `pendingMatches` to active matches in `onReady`.
- Exiting active matches may be moved into the cache if they are not `error`, `notFound`, or `redirected`.
- Lifecycle hooks use `routeId` identity, while storage pools use `match.id` identity.

References:

- `packages/router-core/src/router.ts:2402-2505`

### 4. Hydration

- hydration copies server data into client matches
- hydration may set `_forcePending` and `_displayPending`
- hydration may skip a normal `router.load()` if everything is already SSR-backed

References:

- `packages/router-core/src/ssr/ssr-client.ts:107-163`
- `packages/router-core/src/ssr/ssr-client.ts:249-308`

### 5. React rendering

- `MatchInner` throws promises for `_displayPending`, `_forcePending`, `status: 'pending'`, and `status: 'redirected'`
- `status: 'error'` throws on the client but renders directly on the server
- non-root not-found uses `status: 'notFound'`
- root not-found is represented as `status: 'success'` plus `globalNotFound: true`

References:

- `packages/react-router/src/Match.tsx:141-207`
- `packages/react-router/src/Match.tsx:404-489`
- `packages/react-router/src/Match.tsx:542-567`

## Initial Findings

These are phase-1 observations, not final conclusions.

### 1. Match identity and lifecycle identity are mixed

Storage and reuse are keyed by `match.id`, but enter/stay/leave semantics are keyed by `routeId`.

References:

- `packages/router-core/src/router.ts:1502-1514`
- `packages/router-core/src/router.ts:2419-2467`

This is likely necessary, but the current implementation makes the distinction implicit instead of explicit.

### 2. Pool lookup order is surprising

`getMatch()` reads `cached -> pending -> active`.

Reference:

- `packages/router-core/src/router.ts:2653-2658`

That ordering makes sense only if the same `match.id` can never exist in multiple pools at once. That appears to be intended, but it is not enforced as an invariant.

### 3. Rendering semantics depend on promises with implicit ownership

The adapters depend on promises in `_nonReactive`, but ownership and lifetime are spread across many branches.

References:

- `packages/router-core/src/load-matches.ts:396-401`
- `packages/router-core/src/load-matches.ts:451`
- `packages/router-core/src/load-matches.ts:925`
- `packages/router-core/src/load-matches.ts:938-945`
- `packages/react-router/src/Match.tsx:264-277`

This makes it easy to end up with a mounted match still trying to suspend on a promise that has been resolved, removed, or replaced unexpectedly.

### 4. React hydration concerns leak into core match state

`_forcePending` and `_displayPending` are currently router-core state even though they are primarily adapter/rendering concerns.

References:

- `packages/router-core/src/ssr/ssr-client.ts:107-129`
- `packages/router-core/src/ssr/ssr-client.ts:272-306`

### 5. Reuse/reset semantics are not explicit enough

The HMR path already has to manually clear stale `loaderData` and `__beforeLoadContext` before invalidation.

Reference:

- `packages/router-plugin/src/core/route-hmr-statement.ts:99-138`

That is a strong sign that “what survives match reuse” is currently emergent instead of specified.

## Phase Plan

### Phase 1. Specify intended behavior

Deliverables:

- glossary and explicit terminology
- normative invariants
- scenario matrix across all entry points
- clear split between router-core guarantees and adapter guarantees
- documented divergences from the current implementation

### Phase 2. Model behavior in FizzBee

Deliverables:

- a compact state model for match lifecycle, cache membership, generation, and render gating
- assertions for exclusivity, precedence, continuity, and generation safety
- guided traces for key scenarios: redirects, notFound, preload promotion, hydration, background reloads

### Phase 3. Re-implement from first principles

Deliverables:

- explicit lifecycle/generation model in router-core
- React-first adapter integration
- follow-up alignment of Solid and Vue

## Phase 1: Initial Spec Draft

This section is the start of phase 1. It should be treated as normative unless superseded by later RFC edits.

### Terminology

- Match instance: an implementation-level record representing one matched route occurrence.
- Route presence identity: the logical continuity of a route in the visible tree.
- Load resource identity: the key used to determine whether previously acquired route-loading artifacts may be reused for a route.
  Today the implementation approximates this with `route.id + interpolatedPath + loaderDepsHash`, but phase 2 should model the semantic concept, not the current string construction.
- Committed tree: the currently visible route tree.
- Candidate tree: the next route tree being evaluated for possible commit.
- Reusable artifacts: any previously acquired route-loading artifacts that a later generation may be allowed to reuse.
- Load generation: a single causally ordered attempt to make a match render-ready.
- Effective context: router context plus accumulated parent `context()` values plus accumulated `beforeLoad` returns up to the current match.
- Render gate: a semantic visibility state emitted by router-core, such as blocked, pending, ready, error, or notFound.
- Generation obsolescence: the fact that an older generation may no longer affect the visible tree.

### Normative Goals

1. A render-visible match must never expose incomplete effective context for its own generation.
2. The router must model match loading as explicit generations, so stale completions cannot overwrite newer state.
3. The committed tree must update atomically and coherently, never as an observably inconsistent mix of generations.
4. If an implementation uses multiple pools or stores, their aliasing semantics must be explicit and must not affect correctness.
5. A render-visible pending or redirected match must retain a valid render contract for the entire period it is visible.
6. `beforeLoad` execution is serial from root to leaf.
7. Loader execution begins only after the serial `beforeLoad` prefix is established, and loaders may run in parallel after that.
8. The winning failure for a generation must be determined by semantic precedence and boundary ownership, not by completion timing accidents.
9. Cache promotion and reuse must be explicit. Reused data and rerun work must be determined by spec, not by incidental object reuse.
10. Eviction and garbage collection must not invalidate currently rendered or still-needed render gates.
11. Router-core should model routing facts; framework adapters should model framework-specific rendering mechanics.

### Normative Invariants

These invariants should later become FizzBee assertions.

1. Generation monotonicity
   A completion from generation `N` must not mutate the visible state for generation `N + 1`.
2. Context completeness before render
   If a match may render its route component, then its effective context for that generation is complete.
3. Commit atomicity/coherence
   The committed tree must never expose a mix of route-loading facts that could not have come from one coherent generation.
4. Pending continuity
   If a match is intentionally rendered as pending, the adapter must always have a valid render gate for it until the match leaves pending display.
5. Redirect continuity
   If a match is observed as redirected during a transition, the stale generation must not later become render-ready again.
6. Root not-found shell preservation
   Root not-found must preserve the root shell/layout contract.
7. Ancestor visibility
   If a child fails in a way that still requires ancestor UI/head/data to render, the required ancestor work must be preserved.
8. Explicit reuse rules
   Reusing a cached or active match must not implicitly reuse stale `beforeLoad` context or loader data unless the spec says it does.
9. Implementation alias safety
   If the implementation stores equivalent route information in multiple pools or stores, that duplication must not create ambiguous lookups or divergent visible semantics.

### Router-Core Versus Adapter Contract

Router-core should guarantee:

- committed-tree and candidate-tree semantics
- load resource reuse decisions
- generation identity and completion rules
- the final semantic outcome of each route for a generation
- when context is complete enough for each phase
- semantic render-gate facts and generation obsolescence

Adapters should guarantee:

- how a render gate maps to the framework primitive
- continuity between router-led pending and userland suspense where the framework requires it
- framework-specific shell/error/not-found presentation details
- framework-specific on-mount timing hooks such as React view transitions

### Phase 1 Decisions That Close The Original Gaps

The following phase-1 decisions are now treated as settled enough to begin phase 2.

#### D1. Preload reuse rules

This remains a compatibility policy, not a sacred first-principles law.

The current working principle is:

- `beforeLoad` is conservative preflight/context/authorization work
- `loader` is reusable data acquisition work

Unless phase 2 disproves that split, when a preload later becomes active for the same load resource identity:

- `beforeLoad` reruns on navigation
- a fresh successful loader result may be reused
- an in-flight preload loader may be adopted by navigation
- a resolved preload redirect/notFound/error is not reusable as success data

See rules `P4`, `P9`, and `P10`.

#### D2. Generation-scoped versus match-instance-scoped state

The phase-2 model should separate three concepts that are blurred together in the current implementation:

- route presence identity
- load resource identity
- implementation-level match instance

The phase-2 model should treat the following as stable route/presence facts:

- `routeId`
- route boundary capabilities
- SSR inheritance facts

The phase-2 model should treat the following as reusable-resource identity inputs:

- the route identity
- the relevant path/input key used for reusable artifacts
- loader-relevant dependency inputs

The phase-2 model should treat the following as generation-scoped:

- load status
- fetch phase
- effective context
- loader result data
- terminal error/redirect/notFound outcome
- render gate
- generation obsolescence / cancellation state
- freshness result for the current generation

The implementation may store some of these together, but the semantics should remain separated.

#### D3. Route `context()` semantics

For spec purposes, route `context()` is recomputed for every load generation after the parent effective context is known and before the current match's `beforeLoad` consumes it.

An implementation may memoize or structurally share equivalent results, but the observable semantics must match recomputation-per-generation.

This decision is intentionally spec-first because stale route-context reuse is a likely source of correctness bugs and the current implementation already has to compensate for this during hydration.

#### D4. Failure precedence

Failure precedence has two layers:

- semantic principles, which are normative for the rewrite
- a current-compatibility baseline, which phase 2 can test against and then challenge if a cleaner semantic formulation proves equivalent or better

#### D5. Render-gate ownership split

Router-core owns semantic render-gate facts such as:

- not renderable yet
- render pending
- render ready
- render notFound
- render error

Router-core also owns generation-obsolescence facts such as “this stale generation may no longer affect the committed tree.”

Adapters own the framework mechanism used to realize those facts, such as:

- thrown promises
- suspense/resource wiring
- client-only DOM gating
- framework-specific boundary placement

#### D6. Acceptable adapter divergence

Adapters may diverge in rendering mechanics, but not in observable route-loading semantics.

Allowed divergence:

- React throws promises
- Solid uses resources plus suspense
- Vue renders pending directly and avoids unsupported nested suspense assumptions

Required shared semantics:

- routing/lifecycle semantics
- cache/reuse/eviction semantics
- failure precedence
- hydration boundary semantics
- boundary ownership and root-shell preservation

#### D7. Residual non-blocking note

The tentative exclusive-pool-membership invariant is no longer treated as normative core semantics.

If phase 2 finds that strict exclusivity improves the model, it can be reintroduced as an implementation simplifier. If not, the model must instead define explicit alias-safety semantics.

### Draft Rule Set: Preload, Cache, Reuse, And Eviction

This is the third concrete semantic slice of phase 1.

#### Rule P1. Current implementation approximates load resource identity with route id plus concrete path plus loader-deps identity

For current-implementation compatibility, reusable loader artifacts are keyed by route id plus concrete path plus loader-deps identity.

For the rewrite, this should be treated as the current approximation of load resource identity, not as an untouchable semantic law.

Search params only affect resource reuse through the derived loader-relevant inputs.

References:

- `packages/router-core/src/router.ts:1482-1510`

#### Rule P2. Preload creates reusable artifacts before success

Semantically, preload may create reusable route-loading artifacts before success.

Current implementation note: `preloadRoute()` materializes those artifacts as cached entries at preload start, not only after success.

References:

- `packages/router-core/src/router.ts:2799-2824`
- `packages/router-core/tests/load.test.ts:189-200`
- `packages/router-core/tests/load.test.ts:434-445`

#### Rule P3. Navigation evaluates a candidate tree before visible commit

Semantically, navigation evaluates a candidate tree before updating the committed tree.

Current implementation note: matching first produces a pending tree, then readiness promotes pending to active.

References:

- `packages/router-core/src/router.ts:2355-2371`
- `packages/router-core/src/router.ts:2410-2488`

#### Rule P4. `beforeLoad` and `loader` reuse are intentionally asymmetric

The target spec keeps the current user-visible asymmetry unless phase-1 work finds a stronger reason to change it:

- `beforeLoad` reruns on navigation even if a preload already exists for the same match
- loader results may be reused if they are still reusable under freshness and invalidation rules

References:

- `packages/router-core/tests/load.test.ts:176-339`
- `packages/router-core/tests/load.test.ts:408-571`
- `packages/router-core/tests/load.test.ts:574-650`

#### Rule P5. Freshness is mode-specific

- preload freshness uses `preloadStaleTime`
- navigation freshness uses `staleTime`

References:

- `packages/router-core/src/load-matches.ts:796-803`

#### Rule P6. Staleness alone does not always force reload

A stale successful match should reload only when at least one of the following is true:

- explicit same-location reload
- match cause is `enter`
- the route reappears with a different load resource identity
- the match is invalidated
- `shouldReload` says to reload

References:

- `packages/router-core/src/load-matches.ts:816-827`
- `packages/router-core/src/router.ts:2402-2406`
- `packages/router-core/tests/load.test.ts:809-865`
- `packages/router-core/tests/load.test.ts:929-1044`

#### Rule P7. Background stale reload preserves the current renderable match

If stale reload mode is background, navigation may commit with stale data immediately while the reload continues in the background.

References:

- `packages/router-core/src/load-matches.ts:829-850`
- `packages/router-core/src/load-matches.ts:936-955`
- `packages/router-core/tests/load.test.ts:773-807`
- `packages/router-core/tests/load.test.ts:1067-1117`

#### Rule P8. Blocking stale reload delays activation

If stale reload mode is blocking, navigation waits for the stale reload before activation.

References:

- `packages/router-core/tests/load.test.ts:735-771`
- `packages/router-core/tests/load.test.ts:1078-1100`

#### Rule P9. In-flight preload loaders are reused by navigation

If navigation begins while the same preload loader is still in flight, navigation should reuse that in-flight loader rather than start a second one.

References:

- `packages/router-core/src/load-matches.ts:892-920`
- `packages/router-core/tests/load.test.ts:434-571`

#### Rule P10. Resolved preload failures are not reusable success data

A completed preload whose terminal result is redirect, notFound, or error is not reusable as success data for later navigation.

References:

- `packages/router-core/tests/load.test.ts:447-556`

#### Rule P11. Eviction timing is an implementation choice subject to safety constraints

Semantically, eviction must not break visible or still-needed route-loading state.

Current implementation note: GC eligibility is time-based, but actual removal happens on later GC passes, not on a live timer.

References:

- `packages/router-core/src/router.ts:2476-2488`
- `packages/router-core/src/router.ts:2764-2788`

#### Rule P12. Error-like terminal states do not remain reusable cache entries

- exiting active matches in `error`, `notFound`, or `redirected` do not enter reusable cache on commit
- cached matches that become `redirected` are removed from cache

References:

- `packages/router-core/src/router.ts:2479-2488`
- `packages/router-core/src/router.ts:2636-2648`

#### Rule P13. Invalidation applies uniformly across active, pending, and cached matches

Invalidation marks selected matches stale everywhere, and `error`/`notFound` matches reset to `pending` so they rerun.

References:

- `packages/router-core/src/router.ts:2661-2701`

### Draft Rule Set: Context Visibility

This is the first concrete semantic slice of phase 1.

#### Rule C1. Parent route context is available to child `beforeLoad`

If a parent route modifies context via `context()` or `beforeLoad`, a child `beforeLoad` must observe the updated value.

Evidence and anchors:

- `packages/react-router/tests/routeContext.test.tsx:532-574`
- `packages/react-router/tests/routeContext.test.tsx:952-984`
- `packages/router-core/src/load-matches.ts:453-458`

#### Rule C2. A match's own `beforeLoad` output is not render-visible until that `beforeLoad` resolves

Current code explicitly avoids updating `match.context` during async `beforeLoad` execution to avoid rendering incomplete context.

Reference:

- `packages/router-core/src/load-matches.ts:426-429`

The target spec should keep the safety property but express it positively:

- a match may not render its route component until its effective context for that generation is complete
- the match's loader must observe the same effective context that the route component will later observe for that generation

#### Rule C3. Redirects start a fresh context generation

If an older destination redirects to a new destination, the new destination's `beforeLoad` must observe the correct effective context for the new destination, independent of the abandoned one.

Evidence and anchors:

- `packages/react-router/tests/routeContext.test.tsx:987-1057`
- `packages/router-core/src/load-matches.ts:159-166`

#### Rule C4. Context coherence must hold across invalidation and sibling navigation

If `beforeLoad` reruns and loader data reruns or reuses according to policy, the route component and the loader for that generation must remain coherent.

Evidence and anchors:

- `packages/router-core/tests/load.test.ts:574-650`
- `packages/react-router/tests/loaders.test.tsx:468-648`

### Draft Rule Set: Render-Gate Continuity

This is the second concrete semantic slice of phase 1.

#### Rule R1. Pending display needs a stable render gate

If a match is intentionally displayed as pending, the adapter must have a stable render gate for the entire visible pending interval.

Current React implementation suspends on one of:

- `displayPendingPromise`
- `minPendingPromise`
- `loadPromise`

References:

- `packages/react-router/src/Match.tsx:404-435`
- `packages/router-core/src/ssr/ssr-client.ts:107-129`
- `packages/router-core/src/ssr/ssr-client.ts:272-306`

The target spec should not depend on these exact promise names, but it must preserve the continuity guarantee they currently try to provide.

#### Rule R2. Redirected stale generations must remain safely discardable

If a stale generation has already become visible enough for an adapter to be working with it, a redirect must not leave that adapter with an invalid intermediate state.

Different adapters may realize safe discard differently, but phase 2 should model the semantic guarantee rather than React's thrown-promise mechanism.

References:

- `packages/react-router/src/Match.tsx:448-461`
- `packages/react-router/tests/redirect.test.tsx:116-160`

#### Rule R3. Hydration-only pending behavior is still part of the render contract

Hydration may need to preserve pending UI even when the normal client-side render path would create timers too late.

References:

- `packages/router-core/src/ssr/ssr-client.ts:107-129`
- `packages/router-core/src/ssr/ssr-client.ts:272-306`

The target design should describe this in terms of render gates, not ad hoc React-specific flags in the core model.

#### Rule R4. Render gates must outlive visible implementation transitions

Implementation transitions such as cache promotion, candidate-to-committed promotion, redirect finalization, and stale background completion must not destroy the active render gate while the adapter can still read it.

Relevant implementation anchors:

- `packages/router-core/src/load-matches.ts:115-167`
- `packages/router-core/src/load-matches.ts:925-945`
- `packages/router-core/src/router.ts:2476-2488`

#### Rule R5. Pending timing is an observable contract, not an adapter mechanism

The router's pending semantics include two observable timing commitments:

- pending is not shown until the configured optimistic threshold elapses (`pendingMs`)
- once pending becomes visible, it remains visible for at least the configured minimum duration (`pendingMinMs`)

Adapters may realize this with different primitives, but phase 2 should model the timing semantics explicitly rather than infer them from promise shapes.

References:

- `docs/router/guide/data-loading.md:509-523`
- `packages/react-router/src/Match.tsx:412-435`
- `packages/solid-router/src/Match.tsx:322-367`
- `packages/vue-router/src/Match.tsx:436-471`

### Draft Rule Set: Failure Precedence

This is still provisional, but it is specific enough to guide phase-2 modeling.

#### Rule F1. Serial `beforeLoad` failures cut off deeper serial work

If `beforeLoad` fails at match index `i`, deeper `beforeLoad` work does not run for that generation.

Reference:

- `packages/router-core/src/load-matches.ts:985-1005`

#### Rule F2. beforeLoad notFound may still require ancestor loader/head completion

If a child `beforeLoad` throws `notFound`, ancestor loaders and heads may still be required so the eventual boundary has the required parent data and document head state.

Evidence and anchors:

- `packages/router-core/tests/load.test.ts:1170-1307`
- `packages/router-core/tests/load.test.ts:1334-1381`
- `packages/router-core/src/load-matches.ts:1007-1119`

#### Rule F3. Redirect wins immediately for the current generation

A redirect dominates the current generation and starts resolution of the redirect target.

Phase 2 should model this as a semantic dominance rule, not as a claim about the exact timing of current implementation finalization.

Reference:

- `packages/router-core/src/load-matches.ts:115-167`

#### Rule F4. The winning notFound is the boundary-rendered notFound, not merely the throwing route

The not-found result that matters for rendering is the resolved boundary route, which may differ from the throwing route.

Reference:

- `packages/router-core/src/load-matches.ts:80-113`
- `packages/router-core/src/load-matches.ts:1066-1119`

#### Rule F5. Mixed failure precedence still needs a formal table

The exact precedence order for mixed redirect/notFound/error outcomes across multiple matches is defined below for phase-2 modeling.

### Semantic Failure Principles

These principles are more important than any current table or status encoding.

1. Redirect dominance
   A redirect dominates all non-redirect outcomes for the same generation.
2. Deterministic winner selection
   The winning outcome for a generation must be deterministic and must not depend on completion timing races.
3. Boundary ownership
   notFound and error outcomes are resolved relative to the boundary that will actually render them, not merely the throwing route.
4. Root-shell preservation
   A root-level notFound must preserve the root shell contract.
5. Ancestor work preservation
   Child failure may still require ancestor work so the winning boundary can render coherently.

### Current Compatibility Precedence Baseline

The table below captures the current compatibility baseline inferred from existing code and tests.

This baseline is non-normative. Phase 2 should first model the semantic failure principles and only then add this baseline as an optional compatibility target.

Phase 2 may replace it with a cleaner semantic formulation if the observable behavior remains correct or improves intentionally.

For the current baseline, a single load generation is resolved in this order.

| Rank | Outcome source        | Outcome type  | Notes                                                                                       |
| ---- | --------------------- | ------------- | ------------------------------------------------------------------------------------------- |
| 1    | serial phase          | redirect      | params/search validation and `beforeLoad` redirect abort the generation immediately         |
| 2    | parallel loader phase | redirect      | wins over all non-serial-redirect results                                                   |
| 3    | parallel loader phase | regular error | wins over any notFound and over deferred serial regular errors                              |
| 4    | parallel loader phase | notFound      | wins over serial beforeLoad notFound and serial regular errors                              |
| 5    | serial phase          | notFound      | provisional during serial phase, final only if no higher-ranked loader result supersedes it |
| 6    | serial phase          | regular error | final only if no higher-ranked loader result supersedes it                                  |

Tie-breakers:

- within launched loaders, the winner is chosen by lowest match index, not fastest completion time
- explicit `notFound({ routeId })` targets the exact matched boundary if present; otherwise notFound resolution falls back toward root
- root notFound renders via `globalNotFound`, not `status: 'notFound'`

References:

- `packages/router-core/src/load-matches.ts:985-1005`
- `packages/router-core/src/load-matches.ts:1007-1055`
- `packages/router-core/src/load-matches.ts:1066-1119`
- `packages/router-core/src/load-matches.ts:1176-1181`
- `packages/router-core/tests/load.test.ts:1170-1432`

### Failure Precedence Narrative

TanStack Router currently resolves failures in two stages: a serial preflight stage and a parallel loader stage.

- A redirect thrown during the serial stage terminates the generation immediately.
- A serial-stage `notFound` or regular error is provisional: it stops deeper serial work, but eligible ancestor loaders may still run.
- Once loaders settle, loader outcomes are resolved in route-match order with precedence `redirect > error > notFound`.
- If no loader outcome supersedes the provisional serial result, the router finalizes the serial `notFound` or regular error.
- notFound rendering is based on the resolved boundary route, not merely the throwing route.
- current implementation preserves root notFound via `globalNotFound`, but the rewrite should treat root-shell preservation as the semantic requirement rather than copy the exact flag.

### Draft Rule Set: Hydration And No-SSR Rendering

This is the fourth concrete semantic slice of phase 1.

#### Rule H1. Data participation and render participation are separate concepts

The spec distinguishes:

- did the server provide usable data for this match?
- did the server provide authoritative DOM for this match?

Current `ssr` modes map as follows:

- `ssr: true` => server data, server DOM
- `ssr: 'data-only'` => server data, client-owned first meaningful DOM
- `ssr: false` => client-owned data, client-owned first meaningful DOM

References:

- `packages/router-core/src/router.ts:154-155`
- `packages/router-core/src/load-matches.ts:263-290`

#### Rule H2. Hydration boundaries are defined by DOM ownership, not implementation index

Hydration introduces a client-render boundary wherever authoritative DOM transitions from server-owned to client-owned.

Current working simplification: the first active match with client-owned authoritative DOM is treated as the hydration boundary.

Phase 2 should model the semantic boundary, not copy today's implementation heuristics mechanically.

References:

- `packages/router-core/src/ssr/ssr-client.ts:140-163`

#### Rule H3. Full SSR hydration does not run an unnecessary client load

If all matches are fully SSR-backed and hydration is not in SPA mode, client hydration should not run a redundant `router.load()`.

References:

- `packages/router-core/src/ssr/ssr-client.ts:249-263`

#### Rule H4. SPA-mode hydration preserves pending without blocking the root unnecessarily

If hydration is effectively falling back to SPA-mode, the client may display pending below the root while the router finishes loading, without forcing the whole root tree through an unnecessary initial suspense cycle.

Current implementation note: the existing code special-cases the first non-root match during SPA-mode hydration. Phase 2 should preserve only the user-observable behavior, not the index-based heuristic.

References:

- `packages/router-core/src/ssr/ssr-client.ts:265-308`
- `packages/react-router/src/Matches.tsx:55-66`

#### Rule H5. Adapters may implement pending differently, but observable semantics must match

Allowed adapter divergence:

- React may suspend by throwing promises
- Solid may use resources plus suspense
- Vue may render pending directly and must avoid nested suspense assumptions that corrupt DOM

Required shared semantics:

- no duplicate initial load during SSR hydration
- hydration-safe fallback for client-rendered matches
- pending remains visible where configured
- SSR inheritance semantics stay intact
- committed-tree coherence is preserved during hydration and post-hydration loading

References:

- `packages/react-router/src/Match.tsx:404-462`
- `packages/solid-router/src/Match.tsx:300-413`
- `packages/vue-router/src/Match.tsx:436-472`
- `packages/vue-router/src/Match.tsx:555-561`

#### Rule H6. Optional hydration facts must preserve client-derived truth

Hydration should only overwrite optional facts like `globalNotFound` when the server payload explicitly provided them.

References:

- `packages/router-core/src/ssr/ssr-client.ts:30-37`
- `packages/router-core/tests/hydrate.test.ts:222-350`

### Scenario Rules

The matrix above is summarized here as explicit rules by entry point.

#### Initial page load

- the router computes the full candidate tree for the destination
- all `beforeLoad` run serially from root to leaf until cut off by failure
- loaders run for the surviving prefix according to the failure rules above
- the committed tree updates only after a coherent candidate result exists

#### Fully SSR-backed hydration

- dehydrated matches become the committed tree
- route context and head/script data are reconstructed client-side
- no additional client load runs if hydration is fully SSR-backed and not in SPA mode

#### Hydration with client-rendered boundary

- a semantic client-render boundary exists wherever authoritative DOM shifts from server-owned to client-owned
- the adapter may show a hydration-safe fallback for that boundary and below
- forced/display-pending behavior is allowed here to preserve continuity

#### Client navigation

- navigation always computes a fresh candidate tree
- `beforeLoad` reruns on navigation even when loader results are still reusable
- commit and reusable-artifact behavior follow the reuse/eviction rules above

#### Preload

- preload may create reusable artifacts before success
- preload never commits active UI directly
- in-flight preload loader work may be adopted by a later navigation for the same match

#### Redirect during navigation or preload

- redirect terminates the current generation according to the precedence table
- redirect starts target resolution as a fresh generation
- stale redirected renders must still remain safely discardable until the adapter releases them

#### beforeLoad notFound

- deeper serial work stops immediately
- ancestor loaders and heads may still need to run so the boundary can render correctly
- boundary selection is based on the resolved notFound boundary, not just the throwing route

#### Background stale reload

- the current committed route state remains renderable while background reload continues
- late completion from background reload must not clobber a newer generation

#### Invalidation

- invalidation applies uniformly to committed route state, candidate route state, and reusable artifacts selected by policy
- invalidation marks the selected route-loading facts stale and may reset terminal error/notFound outcomes so they can rerun

#### GC and eviction

- eviction removes only reusable artifacts that are no longer needed by the committed tree or an in-flight candidate tree
- eviction must not break any still-visible or still-needed render gate

### Phase 1 Coverage And Gaps

Coverage already anchoring the spec:

- route context visibility in `beforeLoad` and loader: `packages/react-router/tests/routeContext.test.tsx:447-1057`, `packages/react-router/tests/loaders.test.tsx:468-648`
- navigation/preload/cache/reload rules: `packages/router-core/tests/load.test.ts:176-1117`
- notFound hierarchy and head behavior: `packages/router-core/tests/load.test.ts:1170-1882`
- hydration and `globalNotFound`: `packages/router-core/tests/hydrate.test.ts:222-396`
- pending/redirect UI regressions: `packages/react-router/tests/router.test.tsx:2337-2576`, `packages/react-router/tests/redirect.test.tsx:116-160`

Important remaining gaps to close during implementation:

- explicit tests for mixed loader error vs serial beforeLoad notFound precedence
- explicit tests for generation safety during background reload plus fast subsequent navigation
- explicit adapter-level tests for hydration boundary semantics across React, Solid, and Vue
- explicit tests for promise/render-gate continuity across implementation transitions such as reuse, commit, and eviction

## Scenario Matrix

This is the first pass of the phase-1 inventory. It is intentionally broad so later sessions can drill into each cell.

| Scenario                                      | Match creation/reuse                                    | beforeLoad                           | loader                                    | cache/promotion                                            | render gate                                                    | failure handling                                            |
| --------------------------------------------- | ------------------------------------------------------- | ------------------------------------ | ----------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| Initial page load                             | all next matches created or reused                      | serial root to leaf                  | parallel after serial prefix              | may reuse fresh cached data                                | may show pending depending on route/router defaults            | redirect/notFound/error may prevent full tree mount         |
| SSR hydration, fully SSR-backed               | matches hydrated from server payload                    | may be skipped on first client pass  | may be skipped on first client pass       | hydrated matches become active immediately                 | avoid hydration mismatch; no extra pending UI unless needed    | hydration-time errors/notFound still need defined behavior  |
| SSR hydration with `ssr: false` / `data-only` | first client-render boundary becomes special            | may need rerun on client             | may need rerun on client                  | hydrated prefix may coexist with client-only suffix        | pending continuity must survive hydration                      | redirect/notFound/error cannot corrupt hydrated prefix      |
| Client navigation                             | pending tree derived from next location                 | serial root to leaf                  | parallel                                  | active matches may be preserved; exiting matches may cache | pending may appear late or immediately                         | winning failure must be deterministic                       |
| Preload                                       | target tree may be loaded into cache only               | serial root to leaf                  | parallel                                  | later navigation may promote or rerun                      | no active render, but render-gate facts may still matter later | preload redirect/notFound/error must not poison active tree |
| Redirect during navigation                    | next location may supersede older generation            | may come from `beforeLoad`           | may come from loader                      | stale work must not win later                              | stale render must abandon safely                               | redirect must preserve target navigation semantics          |
| beforeLoad notFound                           | partial tree may still need ancestors                   | serial failure truncates deeper work | ancestors may still need loader/head work | no accidental cache poisoning                              | boundary route must be known                                   | nearest valid boundary and root shell rules must hold       |
| Background stale reload                       | active match remains visible                            | usually reruns serial prefix first   | loader may continue in background         | cached/active data remains usable                          | current render gate stays stable                               | late error/redirect must not clobber newer generation       |
| Invalidation                                  | existing active/cached matches are marked stale/pending | reruns according to spec             | reruns according to spec                  | reuse rules must be explicit                               | active UI may remain while reload happens                      | previous error/notFound state must reset correctly          |
| GC/eviction                                   | inactive cached matches may drop                        | n/a                                  | n/a                                       | must not evict still-needed matches                        | no visible render gate may break                               | eviction cannot affect active/pending generations           |

## Phase 1 Evaluation Findings

An adversarial subagent review of this RFC produced the following conclusions, and this document has already been updated to reflect them.

What survived review as strong first-principles material:

- effective context must be complete before render
- generations must isolate stale completions from newer visible state
- serial preflight plus parallel data loading remains a sound core model
- boundary ownership and root-shell preservation are the right way to reason about notFound and error rendering
- router-core and adapter responsibilities must remain separate

What was corrected or reframed after review:

- `match.id` is no longer treated as the core semantic identity; the RFC now separates route presence identity, load resource identity, and implementation-level match instances
- active/pending/cached pools are no longer treated as normative core semantics; they are implementation choices subject to alias-safety requirements
- the failure-precedence table is now treated as a current compatibility baseline beneath a smaller set of semantic failure principles
- root notFound semantics are defined in terms of root-shell preservation, not `globalNotFound` as a required representation
- hydration is now defined in terms of semantic client-render boundaries, not just the current index-based heuristics
- pending timing is now called out as an observable contract independent of adapter mechanism

What remains intentionally provisional going into phase 2:

- whether the current `beforeLoad` versus `loader` reuse asymmetry is a fundamental product rule or a compatibility policy
- whether strict cross-pool exclusivity is worth preserving as an implementation simplifier
- whether the current compatibility precedence baseline should survive unchanged once modeled semantically

## Phase 2 Status

Phase 2 is complete.

The resulting artifact is a suite of focused executable FizzBee models, not one monolithic exhaustive model. The broad umbrella spec remains useful for exploration, but phase-2 correctness is now carried by the focused slice models below.

Artifacts added:

- core model: `docs/router/route-match-loading.fizz`
- core tiny exhaustive-check config: `docs/router/route-match-loading-tiny.cfg`
- core two-generation exploratory config: `docs/router/route-match-loading-small.cfg`
- hydration-focused model: `docs/router/route-match-hydration.fizz`
- overlap/reuse-focused model: `docs/router/route-match-overlap.fizz`
- background-reload-focused model: `docs/router/route-match-background.fizz`
- pending-timing-focused model: `docs/router/route-match-pending.fizz`
- failure-precedence-focused model: `docs/router/route-match-failures.fizz`
- overlap scenario traces: `docs/router/route-match-overlap-fresh.trace`, `docs/router/route-match-overlap-adopt.trace`
- background scenario traces: `docs/router/route-match-background-late-success.trace`, `docs/router/route-match-background-failure.trace`
- pending scenario traces: `docs/router/route-match-pending-fast.trace`, `docs/router/route-match-pending-slow.trace`
- failure scenario traces: `docs/router/route-match-failures-root-notfound.trace`, `docs/router/route-match-failures-leaf-notfound.trace`, `docs/router/route-match-failures-notfound-redirect.trace`, `docs/router/route-match-failures-error-vs-loader-notfound.trace`, `docs/router/route-match-failures-leaf-loader-error.trace`

Phase-1 semantic coverage now maps to the suite like this:

- committed tree vs candidate generation separation, effective context readiness, notFound root-shell preservation, and core finalization structure: `docs/router/route-match-loading.fizz`
- hydration boundary semantics for fully SSR-backed, `data-only`, and `client-only` leaf boundaries: `docs/router/route-match-hydration.fizz`
- fresh artifact reuse, preload adoption, obsolete-generation non-commit, and generation-local reuse safety: `docs/router/route-match-overlap.fizz`
- background stale reload and late-completion non-clobbering: `docs/router/route-match-background.fizz`
- pending timing contract (`pendingMs` / `pendingMinMs`) at the semantic level: `docs/router/route-match-pending.fizz`
- failure precedence, ancestor preservation, redirect dominance, and error/notFound boundary ownership: `docs/router/route-match-failures.fizz`

Items intentionally left out of the phase-2 suite because they are implementation refinements rather than blockers for phase 3:

- one shared exhaustive model covering every slice at once
- adapter-specific rendering mechanics such as React promise throwing or Vue DOM workarounds
- code-splitting, chunk loading, and head/script asset semantics beyond their effect on readiness and boundary availability

Current verification status:

- `fizz --preinit-hook-file docs/router/route-match-loading-tiny.cfg docs/router/route-match-loading.fizz`
  Passed with `Valid Nodes: 12954` and `Unique states: 1007`.
- `fizz --preinit-hook-file docs/router/route-match-loading-small.cfg docs/router/route-match-loading.fizz`
  Remains useful for exploration but currently exceeds the 2-minute local command budget; it is now treated as an exploratory umbrella model rather than the only overlap-checking path.
- `fizz docs/router/route-match-hydration.fizz`
  Passed with `Valid Nodes: 11` and `Unique states: 10`.
- `fizz docs/router/route-match-overlap.fizz`
  Passed with `Valid Nodes: 162` and `Unique states: 162`.
- `fizz docs/router/route-match-background.fizz`
  Passed with `Valid Nodes: 36` and `Unique states: 36`.
- `fizz docs/router/route-match-pending.fizz`
  Passed with `Valid Nodes: 7` and `Unique states: 7`.
- `fizz docs/router/route-match-failures.fizz`
  Passed with `Valid Nodes: 1740` and `Unique states: 348`.
- `fizz --trace-file docs/router/route-match-overlap-fresh.trace docs/router/route-match-overlap.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-overlap-adopt.trace docs/router/route-match-overlap.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-background-late-success.trace docs/router/route-match-background.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-background-failure.trace docs/router/route-match-background.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-pending-fast.trace docs/router/route-match-pending.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-pending-slow.trace docs/router/route-match-pending.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-failures-root-notfound.trace docs/router/route-match-failures.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-failures-leaf-notfound.trace docs/router/route-match-failures.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-failures-notfound-redirect.trace docs/router/route-match-failures.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-failures-error-vs-loader-notfound.trace docs/router/route-match-failures.fizz`
  Passed.
- `fizz --trace-file docs/router/route-match-failures-leaf-loader-error.trace docs/router/route-match-failures.fizz`
  Passed.

Initial model-driven corrections already made while starting phase 2:

- root-shell notFound rendering needed explicit context completion in the model to satisfy the context-ready invariant
- fresh-artifact reuse needed a generation-local assertion rather than one tied to the artifact's later global state
- the phase-2 model structure worked better when helper logic was inlined instead of relying on top-level function calls in FizzBee actions
- overlap/reuse reachability works better as explicit guided traces than as `exists assertion`s in the current FizzBee toolchain
- a later artifact failure must not retroactively poison a navigation that already reused fresh data; the overlap model now encodes that explicitly
- failure precedence assertions had to be conditioned on higher-ranked outcomes being absent, which helped sharpen the semantic precedence rules

Phase 2 exit criteria met:

- every remaining phase-1 semantic gap was given a focused executable model or trace-backed scenario
- all focused slice models pass exhaustive bounded checks
- all critical scenario traces pass
- remaining non-modeled details are now implementation refinements, not blockers for phase 3

## Phase 2 Bootstrap (Historical)

Phase 1's core semantic contract is complete enough to start modeling.

The goal of phase 2 is not to encode React, Solid, or Vue internals directly. The goal is to model router-core semantics plus an abstract render-gate contract that all adapters must satisfy.

Recommended first model file: `docs/router/route-match-loading.fizz`

### Recommended first model scope

Start with a deliberately small model.

- at most 3 matched indices: root, parent, leaf
- start with 1 committed-tree generation and 1 candidate-tree generation, then expand quickly to allow at least one stale older generation to overlap a newer candidate generation
- support these entry points first: initial load, navigation, preload, redirect, beforeLoad notFound, background stale reload, hydration with one client-rendered boundary
- ignore code-splitting details at first except as a boolean “chunk needed before render-ready” fact
- model userland suspense only as an adapter-side opaque condition, not as part of router-core state

### Recommended model vocabulary

Use these terms in the first FizzBee model.

- committed tree
- candidate tree
- route presence identity
- load resource identity
- reusable artifacts
- load generation
- effective context
- freshness
- boundary owner
- hydration boundary
- render gate
- terminal outcome
- generation obsolescence

### Recommended abstract state

The first model should represent at least the following facts.

- `committed_tree`
  Ordered route presence chain currently visible to the user.
- `candidate_tree`
  Ordered route presence chain currently being evaluated for possible commit.
- `route_facts_by_presence_id`
  Stable route facts such as `routeId`, boundary capabilities, and SSR inheritance facts.
- `resource_key_by_presence_id`
  The load resource identity currently associated with each route presence.
- `reusable_artifacts_by_resource_id`
  Reusable route-loading artifacts and their freshness state.
- `generations`
  First-class generation records keyed by generation token, each carrying entry point, affected route presences/resources, provisional failure, final outcome, and obsolescence state.
- `preflight_state_by_generation_and_presence`
  `not_started`, `in_flight`, `success`, `redirect`, `notFound`, `error`.
- `data_state_by_generation_and_resource`
  `not_started`, `in_flight`, `success`, `redirect`, `notFound`, `error`.
- `context_state_by_generation_and_presence`
  `unknown`, `parent_ready`, `ready`.
- `render_gate_by_generation_and_presence`
  `blocked`, `pending`, `ready`, `error`, `notFound`.
- `obsolescence_by_generation`
  Whether a generation is still allowed to affect the committed tree.
- `hydration_mode_by_presence_id`
  `server_dom`, `data_only`, `client_only`.
- generation-level facts:
  - current entry point
  - serial cursor for `beforeLoad`
  - launched loader set
  - provisional serial failure, if any
  - winning final outcome, if any

### Recommended first actions

The first model should include actions equivalent to:

1. `InitInitialLoad`
2. `InitNavigation`
3. `InitPreload`
4. `InitHydration`
5. `RunBeforeLoadStep`
6. `ResolveBeforeLoadSuccess`
7. `ResolveBeforeLoadRedirect`
8. `ResolveBeforeLoadNotFound`
9. `ResolveBeforeLoadError`
10. `LaunchEligibleLoaders`
11. `ResolveLoaderSuccess`
12. `ResolveLoaderRedirect`
13. `ResolveLoaderNotFound`
14. `ResolveLoaderError`
15. `DetermineBoundaryOwner`
16. `FinalizeWinningOutcome`
17. `StartBackgroundReload`
18. `CompleteBackgroundReload`
19. `CommitCandidateTree`
20. `InvalidateResource`
21. `EvictReusableArtifacts`

Hydration-specific actions can be added immediately after the first model passes the core assertions:

- `HydrateServerFacts`
- `MarkHydrationBoundary`
- `EnterPendingFallback`
- `ExitPendingFallback`

### Recommended first assertions

The first FizzBee model should prove these assertions.

1. Generation safety
   A completion from an older generation never changes the winning visible outcome of a newer generation.
2. Context completeness
   A match with render gate `ready` always has effective context `ready` for that generation.
3. Redirect discard safety
   A redirected stale generation never becomes render gate `ready` again and never corrupts the committed tree.
4. Pending continuity
   A match with render gate `pending` never loses its ability to remain pending until it transitions to another valid gate.
5. Failure precedence
   For any generation, the final winning outcome satisfies the semantic failure principles in this RFC and, when running a compatibility-focused model, the current compatibility baseline.
6. Boundary correctness
   A finalized notFound always resolves to the correct rendered boundary owner.
7. Root shell preservation
   Root notFound never requires replacing the root shell with a non-root-style notFound state.
8. Reuse correctness
   Reused preload/cache data is only observed in cases permitted by rules `P4`, `P9`, and `P10`.
9. Commit atomicity
   Any visible commit corresponds to one coherent candidate-tree resolution.

### Recommended first trace checklist

The first guided traces should cover at least:

1. initial load success
2. initial load with parent beforeLoad modifying child context
3. preload success followed by navigation reuse
4. preload in flight followed by navigation adoption
5. beforeLoad notFound with ancestor loader/head preservation
6. redirect while pending is already visible
7. stale background reload followed by a faster next navigation
8. hydration with one `data-only` boundary

### Recommended first FizzBee config

The first model should stay intentionally small.

- `max_concurrent_actions: 2`
- small bounded route tree depth
- small bounded generation count
- no crash injection at first
- BFS exploration first, then guided traces for the interesting races

### Recommended first implementation boundary after the model exists

Once the first model passes, phase 3 should begin by implementing explicit generation tracking and semantic render gates in router-core before touching adapter-specific mechanics.

## Phase 1 Exit Status

Phase 1 is considered complete at the core semantic level because this RFC now contains:

- a shared vocabulary
- explicit normative goals and invariants
- explicit rules for context visibility
- explicit rules for render-gate continuity
- explicit rules for preload/cache/reuse/background reload/eviction
- an explicit failure-precedence table
- hydration and no-SSR semantics plus adapter constraints
- scenario rules by entry point
- a phase-2 modeling bootstrap with state, actions, assertions, and trace coverage

Several compatibility-oriented details remain intentionally provisional, but they are now narrow enough to be resolved inside phase 2 while modeling, rather than blocking phase 2 from starting.

## Notes For Later Agents

- Treat this document as the source of truth for coordination until a successor RFC replaces it.
- Prefer editing this RFC incrementally instead of scattering planning notes across new files.
- When current code and this RFC disagree, preserve the disagreement in the RFC and treat it as an explicit design decision to resolve, not as a reason to silently follow the current implementation.
- The most important unresolved seam is the implicit state machine spread across `router.ts`, `load-matches.ts`, hydration, and the framework adapters.
