# Typed .NET SSR Protocol Framework — Specification

Status: Draft v0.1
Owner: Platform / Foundational Packages
Audience: Framework implementers, application teams adopting the framework, AI agents generating or extending the runtime.

---

## 0. How to read this document

This is a specification, not a tutorial and not source code. Every section defines:

1. The **contract** (what must exist).
2. The **shape** (types, fields, lifecycle, ordering).
3. The **acceptance criteria** (how an implementer knows they are done).

Code-like fragments are *signatures and shapes*, not implementations. They are normative when they describe contracts, illustrative when they describe usage.

The framework is built from independently shippable packages. Each layer in this spec corresponds to one or more packages. The package list is in §3.

Wherever this spec uses MUST / MUST NOT / SHOULD / MAY, the meaning is RFC 2119.

---

## 1. Goals and non-goals

### 1.1 Goals

The framework MUST deliver:

- A canonical **.NET-owned application graph** (routes, resources, layouts, sections, actions, capabilities, policies).
- **SSR-first** rendering: the server always produces the first useful HTML.
- **Generated TypeScript runtime** derived from the .NET graph. The TS surface is not hand-written.
- **Island-based selective hydration** with renderer adapters (React, Vue, Solid, Preact) behind a single contract.
- **Vite-driven** client build with a chunk manifest the SSR runtime consumes at request time.
- **Partial reloads** at section granularity (no forced full-page refresh).
- **Streaming SSR** with named async boundaries.
- **Typed mutations (Actions)** with server-canonical validation, generated client metadata.
- **Bundle/asset awareness** as a first-class primitive — chunk identity, version, preload, invalidation.
- **Deterministic** behavior across deployments (version skew is handled, not avoided by luck).
- **AI-readable** application graph for tooling and agentic edits.

### 1.2 Non-goals (anti-goals)

The framework MUST NOT become:

- An MVC successor with view-bag conventions.
- A REST framework. Actions are not REST controllers.
- A SignalR-coupled live framework.
- A Blazor WASM / CLR-on-client framework.
- A client-state-first framework (canonical state lives on the server).
- A DTO generator. Props are not DTOs; see §8.

### 1.3 Out of scope for v1

- Native mobile renderer adapters.
- Offline-first sync (CRDT, replication).
- Edge SSR / V8 isolate runtime. (The SSR host is the ASP.NET process in v1.)
- Visual designer / no-code authoring.

---

## 2. Glossary

| Term | Meaning |
|------|---------|
| **Application Graph** | The full, server-discovered model of routes, resources, layouts, sections, actions, validators, capabilities, and policies. |
| **Route** | A typed endpoint that resolves to a page component and produces props. |
| **Params** | Strongly-typed route parameter contract. |
| **Props** | Capability graph passed to a component (data + routes + actions + sections + permissions + links + metadata). |
| **Component** | A server-rendered unit (page, layout, or partial). |
| **Island** | A client-hydrated component embedded inside SSR output. |
| **Section** | A named, independently reloadable region of a page. |
| **Action** | A typed server mutation invokable from the client runtime. |
| **Renderer** | A client-side framework adapter (react, vue, solid, preact) implementing the renderer contract. |
| **Chunk** | A Vite-emitted JS/CSS asset identified by content hash. |
| **Asset version** | A monotonic identifier tying a chunk manifest to a deployed server build. |
| **Protocol envelope** | The JSON shape returned by the SSR runtime for a navigation or partial. |
| **Capability** | A server-computed boolean/affordance the client must not recompute. |
| **Affordance** | A generated method on the client runtime (e.g. `resident.actions.save.execute(...)`). |

---

## 3. Solution topology

### 3.1 Repository layout (target)

```
/src
  /Alis.Ssr.Abstractions          // graph & protocol contracts (no impl)
  /Alis.Ssr.Graph                 // graph discovery + registry
  /Alis.Ssr.Protocol              // wire format, serializers
  /Alis.Ssr.Hosting               // ASP.NET Core integration, request pipeline
  /Alis.Ssr.Streaming             // streaming SSR primitives
  /Alis.Ssr.Validation            // validation metadata bridge
  /Alis.Ssr.Generator             // Roslyn source/incremental generator
  /Alis.Ssr.Vite                  // Vite manifest consumer, dev middleware
  /Alis.Ssr.Renderers             // renderer registry (no client code)
  /Alis.Ssr.Observability         // tracing, metrics
/client
  /packages
    /runtime                      // generated-target client runtime (TS)
    /renderer-react
    /renderer-vue
    /renderer-solid
    /renderer-preact
    /vite-plugin                  // dev + build integration
/docs
  /spec
    typed-ssr-protocol-framework.md   // this file
/samples
  /Sample.Residents               // reference app
```

### 3.2 Package boundaries (rules)

- `Alis.Ssr.Abstractions` MUST have **zero** dependencies on ASP.NET, Vite, or any renderer.
- `Alis.Ssr.Hosting` MUST be the only assembly referencing `Microsoft.AspNetCore.*`.
- `Alis.Ssr.Generator` MUST NOT reference runtime assemblies; it consumes the abstractions via metadata only.
- Client `/runtime` MUST NOT import any renderer package directly. Renderers register themselves.

### 3.3 Versioning

- Server packages use SemVer. Breaking protocol changes MUST bump major.
- The wire protocol is versioned independently (`protocol.version`, see §11).
- Generated TS runtime versions track the server build that produced them; see §16.

---

## 4. Layered architecture

```
.NET application
   |
   v
[L1] Application Graph Discovery   (Alis.Ssr.Graph)
   |
   v
[L2] Protocol Layer                (Alis.Ssr.Protocol)
   |
   v
[L3] Type Generation               (Alis.Ssr.Generator)
   |
   v
[L4] Client Runtime                (client/runtime)
   |
   v
[L5] Renderer Adapter              (client/renderer-*)
```

Each layer is replaceable only at its own seam. A layer MUST NOT reach across a non-adjacent boundary (e.g. the generator MUST NOT depend on the hosting layer).

---

## 5. Layer 1 — Application Graph

### 5.1 Purpose

The application graph is the canonical model of the running application. Every other layer is derived from it.

### 5.2 Graph nodes

The graph MUST expose, at minimum, the following node kinds:

- **Route** — pattern + params type + props type + page component identity + renderer hint + chunk hint.
- **Layout** — component identity + child slot policy.
- **Section** — named partial within a page; reloadable independently.
- **Component** — server-rendered or island; renderer; chunk binding; prop contract.
- **Action** — request type + response type + policy reference + idempotency key strategy.
- **Validator** — owning type + rules expressed as a portable metadata tree (see §13).
- **Capability** — boolean or scalar derived server-side; appears in props.
- **Policy** — named auth requirement attached to routes/actions/sections.
- **Resource** — a logical entity (e.g. `Resident`) that owns links + actions; the bridge for AI-readable affordances.

### 5.3 Discovery

Discovery is performed at:

1. **Compile time** — Roslyn source generator emits a static graph manifest into the assembly (see §15).
2. **Startup time** — the hosting layer hydrates the runtime graph from the manifest plus reflection over DI-registered handlers.

The graph MUST be available before the first request is served. There is no lazy first-request discovery.

### 5.4 Authoring contracts (illustrative)

```csharp
public abstract class AppRoute<TParams, TProps>
{
    public abstract string Pattern { get; }
    public virtual string? Renderer => null;     // null = inherit page default
    public virtual string? ChunkHint => null;    // null = generator decides
}

public abstract class AppAction<TRequest, TResponse>
{
    public virtual string? Policy => null;
    public virtual string? IdempotencyKey => null;
}

public sealed class SectionAttribute : Attribute
{
    public SectionAttribute(string name) { Name = name; }
    public string Name { get; }
}
```

### 5.5 Acceptance criteria

- A scan of the sample app MUST produce a graph snapshot containing all routes, actions, sections, validators present in source.
- The graph snapshot MUST be serializable to JSON for tooling (`alis-ssr graph dump`).
- Two builds of identical source MUST produce byte-identical graph snapshots (determinism).

---

## 6. Layer 2 — Protocol

### 6.1 Purpose

The protocol layer defines the **wire format** between the SSR runtime and the client runtime. It is renderer-agnostic and component-agnostic.

### 6.2 Envelope (full navigation response)

```
{
  "protocol":      { "version": "1" },
  "route":         { "name": "<RouteName>", "params": { ... } },
  "component":     "<ComponentIdentity>",
  "renderer":      "<react|vue|solid|preact>",
  "props":         { ... },                  // see §8
  "layout":        "<LayoutIdentity|null>",
  "assetVersion":  "<string>",
  "chunks":        [ "<chunkId>", ... ],     // chunks required to hydrate
  "preload":       [ "<chunkId>", ... ],     // chunks to preload, not block on
  "partials":      [ ],                      // see §6.4
  "stream":        false,                    // true => see §14
  "status":        200,
  "headers":       { ... },
  "meta": {
    "trace":  "<traceparent>",
    "tenant": "<tenantId|null>",
    "locale": "<bcp47>"
  }
}
```

### 6.3 Partial reload response

A partial response MUST set `partials` to a non-empty list and `props` to `null`. Each partial entry:

```
{
  "section":     "<SectionName>",
  "component":   "<ComponentIdentity>",
  "renderer":    "<rendererId>",
  "props":       { ... },
  "chunks":      [ ... ]
}
```

### 6.4 Streaming response

For streaming, the response is **NDJSON** (one JSON object per line) with the following frame kinds:

- `shell` — first frame; carries layout + non-streamed props + chunk preloads.
- `chunk` — a streamed section payload (`section`, `props`).
- `error` — a frame-scoped error (does not abort the stream).
- `end` — final frame.

The HTTP response is `Content-Type: application/x-ndjson` with `Transfer-Encoding: chunked`.

### 6.5 Action response

```
{
  "protocol":   { "version": "1" },
  "result":     { ... }              // typed response, see §12
    OR
  "error":      { ... },             // see §17
  "invalidate": [ "<SectionRef>" ],  // sections to reload after success
  "redirect":   { "route": "...", "params": {...} } | null
}
```

### 6.6 Acceptance criteria

- The protocol MUST be expressible as a JSON Schema (committed under `/docs/spec/protocol/`).
- A round-trip serializer (server → client → server) MUST preserve all fields without lossy coercion.
- Unknown fields MUST be ignored by the client runtime (forward compatibility).

---

## 7. Routing

### 7.1 Route definition

Routes are .NET classes inheriting `AppRoute<TParams, TProps>`. The framework MUST:

- Bind `TParams` from the URL using the route `Pattern`.
- Validate params before calling the page handler.
- Produce `TProps` via a typed handler bound to the route.

### 7.2 No string URLs

The generated TS surface MUST NOT expose raw URL string construction for in-app navigation. Instead:

```
routes.residents.navigate({ id })            // navigate
routes.residents.href({ id })                // typed URL string when needed (e.g. <a href>)
routes.residents.prefetch({ id })            // preload chunks + props
```

`href` is the only sanctioned way to produce a URL string; it is generated, not user-built.

### 7.3 Navigation lifecycle

```
navigate(request)
  -> resolveRoute(request)              // pattern match
  -> selectStrategy(current, target)    // full | partial | stream
  -> loadChunks(target.chunks)          // parallel; honors preload
  -> fetchEnvelope(target)              // with strategy header
  -> applyEnvelope(envelope)            // mount or patch islands
  -> commit(target)                     // history, scroll, focus
```

Each step MUST be cancelable. A newer navigation MUST abort an in-flight older one without leaking listeners.

### 7.4 Strategy selection

The client runtime MUST select one of:

- **full** — first navigation, hard refresh, or layout change.
- **partial** — same layout, same page component, only sections changing.
- **stream** — server-declared streaming for this route.

The selection MUST be communicated to the server via the `X-Alis-Strategy` header so the server can short-circuit work.

### 7.5 Acceptance criteria

- Navigating between two sibling routes that share a layout MUST result in a partial response when both opt in.
- Cancelled navigations MUST NOT mutate URL or history.
- Browser back/forward MUST replay envelopes from cache when valid (see §10.5).

---

## 8. Params and Props

### 8.1 Params

Params:

- MUST be plain DTO-like records with `init` setters.
- MUST be JSON-serializable and URL-encodable.
- MAY use `[FromQuery]`, `[FromRoute]` markers to disambiguate.

```csharp
public sealed class ResidentsParams
{
    public required Guid Id { get; init; }
    public string? Tab { get; init; }
}
```

### 8.2 Props are capability graphs

Props are **not** DTOs. They MUST be structured to expose:

- **data** — the entity payload(s).
- **routes** — typed navigation entries scoped to this resource.
- **actions** — typed action affordances scoped to this resource.
- **sections** — handles for partial reloads.
- **permissions / capabilities** — server-computed booleans/affordances.
- **links** — related resources (HATEOAS-style, but typed).
- **invalidation metadata** — keys for the client cache.

The generator MUST shape the TS prop type so the following usage is idiomatic:

```ts
resident.routes.details()
resident.routes.carePlan()
resident.actions.archive.execute()
resident.permissions.canEdit
resident.sections.activity.reload()
```

### 8.3 Bad vs good shape

Disallowed (will fail generator lint):

- A prop containing only `entity: TDto` with no capability surface.
- A prop exposing raw URLs as strings.
- A prop exposing arbitrary methods on the .NET side that cannot be modeled as actions.

### 8.4 Acceptance criteria

- The generator MUST emit a lint warning when a route's `TProps` contains no capability shape (only data).
- Generated TS MUST be tree-shakeable at the section/action level (unused affordances drop out).

---

## 9. Components

### 9.1 Server components

Server components:

- Are layout-aware (composable via slot props).
- MAY be async.
- MAY emit streaming boundaries (see §14).
- MAY embed islands.

The component authoring surface MUST be Razor-compatible at minimum; alternative authoring (`.alis` files) is OPTIONAL for v1.

### 9.2 Component identity

Every server component MUST resolve to a stable string identity (e.g. `Residents.ResidentsPage`). Identity is used in:

- the protocol envelope (`component`),
- the chunk manifest (renderer-side islands keyed by component identity),
- AI-graph references.

Identity is generated from namespace + type name unless overridden via `[ComponentIdentity("...")]`.

### 9.3 Layouts

A layout is a server component that:

- declares **named slots** (e.g. `header`, `body`, `aside`).
- declares **persistence policy** per slot (`persistent` survives partial reloads; `transient` is replaced).

The framework MUST avoid re-rendering persistent slots on a partial transition.

---

## 10. Islands

### 10.1 Definition

An island is a client-hydrated region embedded in SSR output. Each island MUST carry:

- a stable **name** (component identity, see §9.2),
- a **renderer** id,
- a serialized **props** payload,
- a **chunk** id,
- a **hydration mode** (`eager`, `visible`, `idle`, `interaction`, `media`).

### 10.2 Markup contract

The server MUST emit islands as a stable placeholder element. The runtime MUST locate islands without a DOM walk over arbitrary elements.

```
<alis-island
  data-name="ResidentScheduler"
  data-renderer="vue"
  data-chunk="scheduler.abcd1234"
  data-mode="visible"
  data-props-ref="island-3">
  <!-- SSR fallback HTML (skeleton or first paint) -->
</alis-island>
<script type="application/json" data-island-props="island-3">{...}</script>
```

The props payload MUST be in a sibling `<script type="application/json">` so the placeholder content remains valid markup and HTML streaming is not blocked by large JSON.

### 10.3 Hydration lifecycle

```
discover (server emitted placeholders)
  -> schedule (per data-mode)
  -> resolveRenderer
  -> resolveChunk        // dynamic import via manifest
  -> instantiate         // renderer.mount(...)
  -> bind invalidation   // subscribe to section events
```

Hydration MUST be cancelable if the island is removed before mount completes.

### 10.4 Hydration modes

| Mode | Trigger |
|------|---------|
| `eager` | At runtime boot, in document order. |
| `visible` | When the element enters the viewport (IntersectionObserver). |
| `idle` | `requestIdleCallback` (or setTimeout fallback). |
| `interaction` | First user interaction on the placeholder. |
| `media` | A media query matches. |

### 10.5 Client cache

The runtime MUST maintain a route-keyed envelope cache with:

- LRU eviction (size configurable).
- TTL per entry, derived from `props.cache.maxAge` if present.
- Invalidation by `section` reference (see §14).
- Asset-version-pinned: entries from a stale `assetVersion` MUST be discarded.

---

## 11. Renderer adapters

### 11.1 Renderer contract

Every renderer adapter MUST implement:

```ts
interface ClientRenderer {
  readonly id: string;          // "react" | "vue" | "solid" | "preact"
  mount(target: Element, component: ComponentRef, props: unknown): MountHandle;
  update(handle: MountHandle, props: unknown): void;
  unmount(handle: MountHandle): void;
}

interface MountHandle { readonly dispose: () => void }
```

### 11.2 Registration

Renderers self-register at runtime boot:

```ts
import { registerRenderer } from "@alis/ssr-runtime";
import { renderer } from "@alis/ssr-renderer-react";
registerRenderer(renderer);
```

The runtime MUST refuse to start if a referenced renderer id is missing from the registry.

### 11.3 SSR equivalence

For each renderer that supports SSR, the server MUST be able to produce the initial HTML using that renderer's SSR API (in v1 via a Node-side SSR worker). The Node-side SSR worker is invoked by `Alis.Ssr.Hosting` over a stable IPC contract; see §15.

### 11.4 Acceptance criteria

- Mixing two renderers on the same page (e.g. a React page with a Vue scheduler island) MUST work without bundle duplication of the shared protocol runtime.
- Switching renderer for the same component identity between deployments MUST not break navigation (envelope carries `renderer`).

---

## 12. Actions

### 12.1 Concept

Actions are typed server mutations addressed by **identity**, not by HTTP path. The HTTP endpoint is an implementation detail of the action transport.

### 12.2 Definition

```csharp
public sealed class SaveResidentAction
    : AppAction<SaveResidentRequest, SaveResidentResponse>
{
    public override string Policy => Policies.EditResident;
}
```

### 12.3 Generated client surface

```ts
await resident.actions.save.execute({ firstName });
```

The generated `execute` method MUST:

- attach an idempotency key when the action declares one,
- carry the current route + section context as headers,
- return a typed `{ result } | { error }` (no exceptions for expected errors),
- honor `invalidate` returned by the server.

### 12.4 Optimistic updates

An action MAY declare an optimistic projection. The generated client surface exposes:

```ts
resident.actions.save.optimistic({ firstName });  // applies locally
await resident.actions.save.execute({ firstName });  // resolves or rolls back
```

Optimistic projections MUST be opt-in per action. The framework provides no implicit optimism.

### 12.5 Acceptance criteria

- An action invocation MUST appear in observability as a single span with the action identity.
- A failed `execute` MUST roll back any optimistic projection associated with the same call id.

---

## 13. Validation

### 13.1 Canonical location

Validation rules MUST exist only on the server. The client MUST NOT receive executable validation code; it receives **metadata**.

### 13.2 Metadata tree

The metadata emitted per request type:

```
{
  "type": "SaveResidentRequest",
  "fields": {
    "firstName": [
      { "kind": "required" },
      { "kind": "maxLength", "value": 100 }
    ],
    "dob": [
      { "kind": "date" },
      { "kind": "before", "value": "today" }
    ]
  },
  "groups": { ... }
}
```

The metadata kinds in v1 MUST cover: `required`, `minLength`, `maxLength`, `pattern`, `min`, `max`, `email`, `url`, `oneOf`, `date`, `before`, `after`, `equalsField`, `custom`.

`custom` rules MUST be opaque to the client and surface only as a server round-trip.

### 13.3 Bridge for FluentValidation

Where FluentValidation is in use, the framework MUST ship a bridge that converts supported validators to the metadata kinds above. Unsupported rules degrade to `custom`.

### 13.4 Acceptance criteria

- Rules expressed with FluentValidation's `NotEmpty`, `MaximumLength`, `EmailAddress`, `Matches`, `GreaterThan`, etc. MUST round-trip into client metadata without authoring duplication.
- The generator MUST emit `validators.<request>.<field>.<kind>` as a tree the renderer's form layer can consume.

---

## 14. Partial reloads and Streaming SSR

### 14.1 Sections

A page declares one or more named sections. Each section is:

- a server-renderable subtree,
- addressable by name,
- independently reloadable,
- independently streamable.

### 14.2 Partial API

Server:

```csharp
return AppView.Partial(ResidentsSections.Grid, gridProps);
```

Client:

```ts
resident.sections.grid.reload();        // re-fetch this section
resident.sections.grid.invalidate();    // mark dirty; reload on next focus
```

### 14.3 Streaming API

```csharp
return AppStream.Begin()
    .Shell(shellProps)
    .Stream(ResidentsSections.Grid,     gridTask)
    .Stream(ResidentsSections.Activity, activityTask);
```

The stream:

- emits `shell` first (carrying chunk preloads),
- emits `chunk` frames as each task completes (out-of-order allowed),
- emits `end` last,
- emits an `error` frame for a single failed task without aborting siblings.

### 14.4 Backpressure & timeouts

- Per-stream wall-clock timeout MUST be configurable; default 30s.
- A slow client MUST not block the server beyond a configurable write timeout.
- Cancellation tokens MUST propagate from the HTTP response to each task.

### 14.5 Acceptance criteria

- A page with three streamed sections MUST render the shell before any section completes.
- An error in one section MUST NOT prevent the other two from rendering.
- Network disconnect mid-stream MUST cancel outstanding tasks on the server within one second.

---

## 15. Type generation and build pipeline

### 15.1 Generator inputs

The Roslyn generator consumes:

- All types deriving from `AppRoute<,>` and `AppAction<,>`.
- Types annotated with `[Section]`, `[ComponentIdentity]`, `[Capability]`, `[Resource]`.
- FluentValidation validators bound to action request types.
- Component identity registrations.

### 15.2 Generator outputs

For each compilation, the generator emits:

1. A **graph manifest** (`alis.graph.json`) embedded as an assembly resource.
2. A **TypeScript runtime SDK** written to `client/.generated/`:
   - `routes.ts` — all routes with `navigate`, `href`, `prefetch`.
   - `actions.ts` — all actions with `execute`, `optimistic`.
   - `models.ts` — params, props, request, response, capability types.
   - `validators.ts` — validation metadata trees.
   - `sections.ts` — section references.
   - `manifest.ts` — component identity → chunk hint mapping (resolved at build).
3. A **JSON Schema** for the protocol envelope under `client/.generated/protocol/`.

The generated TS MUST be deterministic: identical source MUST yield byte-identical output.

### 15.3 Build pipeline

```
1. dotnet build
     -> Roslyn generator runs
     -> alis.graph.json embedded
     -> client/.generated/* written

2. vite build (client)
     -> reads client/.generated
     -> produces chunk manifest (vite/manifest.json)
     -> chunk hashes assigned

3. alis ssr publish
     -> merges alis.graph.json + vite manifest
     -> produces alis.assets.json (the runtime asset manifest, see §16)
     -> pins assetVersion
```

### 15.4 Dev mode

In dev mode:

- The generator runs on file save (incremental).
- The Vite dev server runs alongside ASP.NET; `Alis.Ssr.Vite` proxies asset requests in development and rewrites chunk URLs in envelopes.
- HMR MUST work for renderer islands. HMR MUST NOT break SSR rendering of the same components.

### 15.5 Node-side SSR worker

For renderer adapters that need JS-side SSR (React/Vue/Solid/Preact), `Alis.Ssr.Hosting` MUST manage a Node worker pool:

- The worker exposes a request/response IPC contract with frames: `render`, `result`, `error`.
- Worker lifetime MUST be supervised (restart on crash, drain on shutdown).
- The IPC contract MUST carry: `component`, `renderer`, `props`, `assetVersion`. The worker MUST reject `assetVersion` mismatches loudly.

### 15.6 Acceptance criteria

- A clean build (`dotnet build && vite build && alis ssr publish`) MUST produce a deployable artifact set with no manual hand-off.
- The generator MUST be incremental: editing a single action MUST not regenerate unrelated files.
- The dev loop MUST surface a clear error when the generated TS is stale relative to .NET source.

---

## 16. Asset system

### 16.1 Asset manifest (`alis.assets.json`)

```
{
  "assetVersion": "2026.05.22.1",
  "renderers":    { "react": "react.abcd1234.js", ... },
  "components": {
    "Residents.ResidentsPage": {
      "chunk":    "residents-page.ef561234.js",
      "css":      [ "residents-page.ef561234.css" ],
      "preload":  [ "residents-grid.ab981234.js" ]
    }
  },
  "chunks": { "<id>": { "file": "...", "imports": [...], "css": [...] } }
}
```

### 16.2 Version pinning

- The SSR runtime MUST resolve a single `assetVersion` per process. It is selected at startup and stable for the process lifetime.
- An envelope's `assetVersion` MUST be the process's pinned value.
- The client MUST refuse to apply an envelope whose `assetVersion` differs from the currently-loaded one and MUST force a full reload instead.

### 16.3 Deployment skew

During rolling deployments:

- Old client + new server: server detects via `X-Alis-Asset-Version` request header and responds with a `409 Asset Version Mismatch` carrying a `force-reload` directive.
- New client + old server: same mechanism in reverse.
- The client runtime MUST handle this directive transparently (no white screen).

### 16.4 Preload

The server MUST attach `Link: <chunk>; rel=preload; as=script` headers for the page-critical chunks and use `modulepreload` for ESM where supported.

### 16.5 Acceptance criteria

- Two server instances on different builds behind a load balancer MUST not mix-and-match chunks within a single page render.
- An asset version mismatch MUST result in at most one extra full reload, never a navigation loop.

---

## 17. Error model

### 17.1 Error envelope

```
{
  "error": {
    "code":       "<MachineCode>",
    "message":    "<HumanReadable|null>",
    "fields":     { "<fieldPath>": [ "<MessageKey>" ] },
    "retryAfter": <seconds|null>,
    "traceId":    "<traceparent>"
  }
}
```

### 17.2 Error categories (v1)

- `ValidationFailed` — 422; carries `fields`.
- `NotAuthenticated` — 401.
- `NotAuthorized` — 403.
- `NotFound` — 404.
- `Conflict` — 409; carries `retryAfter` when applicable.
- `AssetVersionMismatch` — 409 with `force-reload` directive.
- `RateLimited` — 429; `retryAfter` REQUIRED.
- `ServerError` — 500.

Custom codes are permitted but MUST extend, not replace, the categories above.

### 17.3 Client behavior

- `ValidationFailed` MUST be surfaced to the relevant form without a thrown exception.
- `NotAuthenticated` MUST trigger the configured re-auth flow exactly once per navigation.
- `AssetVersionMismatch` MUST trigger a full reload (see §16.3).
- All other categories MUST be available to the calling code as a typed `{ error }` return.

### 17.4 Acceptance criteria

- Every error path in the framework MUST emit a `traceId` matching the request's trace.
- An unmatched route MUST return `NotFound` with no stack details in production.

---

## 18. Auth, policies, and capabilities

### 18.1 Policy attachment

Routes, sections, and actions declare a named policy. The framework MUST enforce the policy server-side **before** invoking any handler.

### 18.2 Capability projection

For each protected affordance, the server projects a capability into props:

```
{
  "permissions": {
    "canEdit":     true,
    "canArchive":  false
  }
}
```

The client MUST treat capabilities as the source of truth for UI affordances. The client MUST NOT compute permissions locally.

### 18.3 Generated guards

The generator MUST emit guards that prevent invoking an action whose declared policy resolves to a false capability in the current props. The guard is a developer-experience aid; the server enforcement is the real boundary.

### 18.4 Acceptance criteria

- Removing a user's permission MUST cause the next navigation to that page to render without the affordance.
- An action invoked despite a false capability MUST return `NotAuthorized` and MUST NOT trigger optimistic updates.

---

## 19. Multi-tenancy

### 19.1 Tenant resolution

Tenant resolution is performed at the host edge (subdomain, header, claim — configurable). The resolved tenant is attached to:

- the request scope,
- the response envelope (`meta.tenant`),
- observability spans.

### 19.2 Tenant-specific assets

`assetVersion` MAY be tenant-scoped. The framework MUST support:

- Single-asset-set deployments (default).
- Per-tenant asset overrides (a tenant pins a sub-version).
- A tenant cannot pin a version the server does not still host.

### 19.3 Tenant-specific localization

Each envelope carries `meta.locale`. The generator emits a string-key catalog; runtime resolution is tenant + locale.

### 19.4 Acceptance criteria

- A request to tenant A MUST never load chunks intended for tenant B.
- A locale switch MUST be a partial reload, not a full page refresh, where the layout permits.

---

## 20. Observability

### 20.1 Required signals

- **Tracing**: every request MUST emit a root span; navigation, partial, action, validation, render, and Node SSR worker work MUST be child spans.
- **Metrics**: per-route p50/p95 SSR latency; per-action error rate; chunk load time; hydration time per island.
- **Logs**: structured logs MUST include `traceId`, `routeName`, `tenant`, `assetVersion`.

### 20.2 Client telemetry

The client runtime MUST emit:

- `navigation.start`, `navigation.commit`, `navigation.error`.
- `island.hydrate.start`, `island.hydrate.end`.
- `chunk.load.start`, `chunk.load.end`.

Telemetry transport is pluggable; the framework ships a no-op default and a sample OTLP-over-HTTP exporter.

### 20.3 Acceptance criteria

- A single end-to-end trace MUST connect: browser navigation → server route → action → Node SSR worker → hydration.

---

## 21. Deployment safety

The framework MUST tolerate:

- **Rolling deployments** (mixed-version fleet behind a load balancer).
- **Stale clients** (open tabs from a previous build).
- **Asset CDN lag** (server build referencing a chunk the CDN hasn't propagated).

Strategies:

- `assetVersion` pinning (§16).
- `force-reload` directive on mismatch.
- Server MUST keep the **last N** asset manifests addressable so a stale client can complete an in-flight navigation before being forced to reload. N is configurable; default 2.

---

## 22. AI-first metadata

### 22.1 Graph export

The framework MUST expose `GET /__alis/graph` (gated by an admin policy) returning the full application graph as JSON. The shape MUST include:

- routes (with params, props shape, policies),
- resources and their affordances,
- actions (with request/response shapes and policies),
- sections and their owning components,
- chunk topology,
- localization keys,
- validation metadata.

### 22.2 Stable identifiers

Every node in the graph MUST have a stable string identifier suitable for AI tooling (e.g. agentic edits, search, code-mod). Identifiers MUST NOT depend on file paths.

### 22.3 Affordance vocabulary

The framework MUST publish a vocabulary describing affordance kinds (`navigate`, `execute`, `reload`, `invalidate`, `prefetch`, `optimistic`) so external tools can reason about them without inspecting source.

### 22.4 Acceptance criteria

- An LLM agent given only the graph export MUST be able to generate a correct call to any action without reading source.
- Renaming a route in source MUST surface in the graph export within the same build cycle.

---

## 23. Developer experience

### 23.1 The headline ergonomic

In application code, rendering a page MUST be:

```csharp
return App.Render<ResidentDetails>(props);
```

No serializers, no route registration, no hydration registration, no manifest plumbing on the developer's path.

### 23.2 Generated artifacts ownership

- `client/.generated/**` is owned by the generator. It MUST be added to `.gitignore` by the project template.
- Developers MUST NEVER edit generated files.
- The generator MUST fail loudly if generated files are detected as modified.

### 23.3 Errors

Errors surfaced to developers MUST identify:

- the offending .NET symbol (file:line),
- the layer (graph, generator, hosting, vite),
- a runbook link in `/docs/spec/runbooks/<code>.md`.

### 23.4 CLI

A single CLI `alis ssr` MUST provide:

- `alis ssr dev` — start dev loop (dotnet watch + vite dev + node ssr).
- `alis ssr build` — full production build.
- `alis ssr publish` — emit deployable artifact bundle.
- `alis ssr graph dump [--json|--md]` — inspect the application graph.
- `alis ssr doctor` — diagnose common misconfigurations.

---

## 24. Security considerations

- Generated TS MUST NOT embed any server-only secret, connection string, or policy expression body.
- The graph export endpoint MUST be policy-gated.
- All envelopes MUST be served with `Content-Type: application/json; charset=utf-8` and `X-Content-Type-Options: nosniff`.
- Action endpoints MUST be CSRF-protected (double-submit cookie or origin check; configurable).
- Streaming responses MUST close cleanly on auth invalidation; subsequent frames MUST NOT leak data after revocation.

---

## 25. Conformance

A framework implementation is conformant when, against the sample app in `/samples/Sample.Residents`:

1. A cold navigation to `/residents/{id}` returns a single envelope, renders SSR HTML, and hydrates the listed islands.
2. A sibling navigation to `/residents/{otherId}` results in a partial envelope, not a full one.
3. An action invocation from the client returns a typed result and triggers section invalidation.
4. A streamed page emits the shell within 100ms p95 on a baseline machine.
5. A rolling deployment between two builds produces zero white-screen errors for a stale client.
6. The generator and the dev loop are deterministic across two clean runs.
7. The graph export is consumable by an external tool to call every action correctly.

---

## 26. Milestones

| Milestone | Scope | Exit criteria |
|-----------|-------|---------------|
| M1 — Skeleton | Abstractions, Hosting (no streaming, no Vite), single React renderer, hand-written client | A page renders SSR, hydrates one island, action round-trips. |
| M2 — Generator | Roslyn graph + TS generation + validators bridge | `client/.generated` produced; sample app uses generated routes/actions. |
| M3 — Vite | Vite plugin, chunk manifest, asset version pinning | Cold + warm builds produce stable manifests; preload headers attached. |
| M4 — Partials & Streaming | Sections, partial envelopes, NDJSON streaming, section invalidation | Three-section sample page streams with independent errors. |
| M5 — Multi-renderer | Vue, Solid, Preact adapters; Node SSR worker pool | Mixed-renderer page renders and hydrates correctly. |
| M6 — Enterprise | Multi-tenancy, observability, deployment safety, AI graph export | Rolling deploy + tenant override + trace continuity verified. |

Each milestone MUST ship with: tests, runbook docs, and a sample-app demonstration of the new capability.

---

## 27. Open questions

The following are explicitly deferred for resolution before M2 starts:

1. **Component authoring surface beyond Razor** — do we ship a `.alis` first-party authoring format, or treat Razor as the only v1 server component format?
2. **Form runtime** — does the framework ship a renderer-agnostic form binding layer, or do we expose validation metadata and let renderer-specific form libraries consume it?
3. **Resource discovery** — do we infer `Resource` nodes from convention (e.g. one per props root) or require explicit `[Resource]` attribution?
4. **Generator hosting** — is the generator a Roslyn source generator only, or also a standalone CLI for non-Roslyn consumers (e.g. tools generating clients in other ecosystems)?
5. **Server-sent invalidation** — without SignalR, do we add a server-push channel (SSE) for cross-tab section invalidation, or remain pull-only in v1?
6. **Streaming over HTTP/3** — do we require HTTP/2 minimum, or also validate HTTP/3 behavior in v1?

Each open question MUST be closed via an ADR in `/docs/spec/adr/` before the milestone that depends on it begins.

---

## 28. Change control

- This document is the source of truth for the framework's behavior.
- Any change to the wire protocol (§6) MUST be accompanied by a protocol version bump and a migration note in `/docs/spec/protocol/CHANGELOG.md`.
- Any change to the generator output shape MUST be accompanied by a generator version bump.
- Breaking changes MUST be staged: deprecation warning for one minor version, then removal.

End of specification.
