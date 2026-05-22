# Typed .NET SSR Protocol Framework — Canonical Forms Specification

Status: Draft v0.2
Mode: Canonical reference. For each primitive, this document gives the C# authoring form, the generated TypeScript, the wire shape, and the binding rules. There is no separate tutorial: the canonical forms *are* the contract.

---

## 0. Reading model

The framework is a **typed application protocol**. It has four expressive surfaces:

1. **C# authoring surface** — what a developer types.
2. **Compile-time graph** — what the Roslyn generator extracts.
3. **Wire envelope** — what flows between server and client at runtime.
4. **Generated TypeScript surface** — what the client uses.

The spec is canonical when, for every concept, the same identity threads through all four surfaces with no manual glue. If a developer writes `SaveResident` once in C#, then:

- the graph has a node `residents.save`,
- the wire encodes references to it as `{ "$kind": "action", "id": "residents.save", ... }`,
- the client uses it as `resident.actions.save.execute({...})`.

That thread is the whole point of the framework. Every section below describes one strand of it.

---

## 1. The Typed Layer in C#

### 1.1 The type system spine

The framework is anchored by a small, closed set of base types and marker interfaces. Application code derives from these and nothing else.

#### Base classes

| Base | Purpose |
|------|---------|
| `AppRoute<TParams, TProps>` | A typed endpoint. Declares URL pattern, params, and props. |
| `AppPage<TParams, TProps>` | The server handler that produces props for a route. |
| `AppLayout<TProps>` | A composable server layout with named slots. |
| `AppSection<TParams, TProps>` | An independently reloadable region within a page. |
| `AppSectionHandler<TParams, TProps>` | The handler that produces section props. |
| `AppAction<TRequest, TResponse>` | A typed mutation, identified by a stable action id. |
| `AppActionHandler<TRequest, TResponse>` | The handler that executes the mutation. |
| `AppResource<TIdentity>` | The capability graph of a domain entity. |
| `AppValidator<TRequest>` | Server-canonical validation for an action request. |
| `AppView<TProps>` | The server's return value from a page (or a partial). |
| `AppStream` | A streaming response builder. |
| `AppLocalization<TKeys>` | A localization catalog. |

#### Typed reference primitives

These exist purely to carry identity across boundaries. They are not DTOs; they are pointers the client materializes into live affordances.

| Primitive | Carries | Generated TS shape |
|-----------|---------|--------------------|
| `RouteRef<TParams>` | route identity + params | `{ navigate(), href(), prefetch() }` |
| `ActionRef<TAction>` | action identity + context | `{ execute(req), optimistic(req) }` |
| `SectionRef<TProps>` | section identity + params | `{ reload(), invalidate(), props }` |
| `ResourceRef<TResource>` | resource type + identity | live resource proxy (lazy-fetched) |
| `IslandRef` | island identity + chunk hint | hydration descriptor |
| `ChunkRef` | chunk identity | preload/dynamic-import handle |
| `MediaRef` | asset identity (image, font, etc.) | URL + variants |

#### Marker interfaces

| Interface | Meaning |
|-----------|---------|
| `IResourceIdentity` | Identifies a `ResourceRef`'s key. |
| `IAppLinks` | Marks a record as a link bundle on a Resource. |
| `IAppActions` | Marks a record as the action bundle on a Resource. |
| `IAppSections` | Marks a record as the section bundle on a Props or Resource. |
| `IAppPermissions` | Marks a record as the capability bundle on a Resource or Props. |
| `IAppPolicy` | Marker for a policy descriptor type. |
| `IAppLocalizationKeys` | Marker for a string-key catalog. |

#### Attributes

```csharp
[Route("/pattern")]            // optional override; default = convention
[Action("residents.save")]     // optional override of action id
[Resource("Resident")]         // optional override of resource type id
[Section("Activity")]          // optional override of section id
[Policy("EditResident")]       // attach a policy to a route/section/action
[Renderer("vue")]              // pin a renderer for a page/island
[Chunk("scheduler")]           // chunk hint for the bundler
[Island("ResidentScheduler")]  // declare an island component
[Layout(typeof(TenantShell))]  // pin a layout to a route
[Idempotent("clientKey")]      // declare idempotency mode for an action
[Optimistic]                   // mark an action as supporting optimism
[Capability("canEdit")]        // expose a capability on a Resource
[FromRoute] [FromQuery("tab")] [FromHeader("X-Tenant")]
```

### 1.2 Naming conventions

These are normative; the generator relies on them when no attribute is supplied.

- **Route type**: `<Resource><Purpose>Route`, e.g. `ResidentDetailsRoute`, `ResidentEditRoute`.
- **Page type**: `<Resource><Purpose>Page`, paired with the route by suffix swap.
- **Action type**: `<Verb><Resource>`, e.g. `SaveResident`, `ArchiveResident`. The action id is `<resourceLowercase>.<verbLowercase>` unless `[Action(...)]` overrides.
- **Resource type**: `<Singular>Resource`, e.g. `ResidentResource`.
- **Section type**: `<Resource><Name>Section`, e.g. `ResidentActivitySection`.
- **Layout type**: `<Name>Layout`, e.g. `TenantShellLayout`.
- **Island type**: `<Name>Island` or `<Name>` when declared with `[Island]`.
- **Nested records**: `Params`, `Props`, `Links`, `Actions`, `Sections`, `Permissions`, `Identity`, `Request`, `Response`, `Validator`, `Handler` — all expected as nested types of their parent.

The nesting convention is load-bearing: the generator infers ownership and the wire shape from it.

### 1.3 File conventions

One Resource per folder. The folder is the unit of feature ownership.

```
/Features/Residents/
  ResidentResource.cs
  ResidentDetailsRoute.cs
  ResidentDetailsPage.cs
  ResidentEditRoute.cs
  ResidentEditPage.cs
  Sections/
    ResidentActivitySection.cs
    ResidentCarePlanSection.cs
  Actions/
    SaveResident.cs
    ArchiveResident.cs
  Islands/
    ResidentScheduler.cs
  Policies/
    EditResidentPolicy.cs
  Localization/
    ResidentKeys.cs
```

Tests live in a parallel `/Features/Residents.Tests/` tree; the generator treats anything in a `*.Tests` assembly as out of the graph.

### 1.4 Composition root

A single registration call wires the framework into ASP.NET Core. Application code does not register individual routes, actions, or sections — the graph is discovered.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAlisSsr(opt =>
{
    opt.AssetVersion       = AppBuild.AssetVersion;
    opt.AddAssembly<AppMarker>();              // discovery root
    opt.AddRenderer<ReactRenderer>("react");
    opt.AddRenderer<VueRenderer>("vue");
    opt.AddLayout<TenantShellLayout>().Default();
    opt.AddLocalization<AppLocalization>();
    opt.AddPolicy<EditResidentPolicy>();
});

var app = builder.Build();
app.UseAlisSsr();                               // single middleware
app.Run();
```

`AddAlisSsr` is the only public hosting entry point. Anything more granular is an internal extension surface.

---

## 2. Route — canonical form

### Purpose
A `Route` declares a URL pattern, a typed params record, and a typed props record. It is the only place URL syntax is allowed in user code.

### C# canonical form

```csharp
namespace App.Features.Residents;

[Route("/residents/{id:guid}")]
public sealed partial class ResidentDetailsRoute
    : AppRoute<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public sealed record Params
    {
        [FromRoute]               public required Guid    Id  { get; init; }
        [FromQuery("tab")]        public          string? Tab { get; init; }
    }

    public sealed record Props
    {
        public required ResidentResource     Resident { get; init; }
        public required Sections             Sections { get; init; }
        public required PageMeta             Meta     { get; init; }
    }

    public sealed record Sections : IAppSections
    {
        public required SectionRef<ResidentActivitySection.Props> Activity { get; init; }
        public required SectionRef<ResidentCarePlanSection.Props> CarePlan { get; init; }
    }
}
```

### Generated TypeScript

```ts
// routes.ts
export interface ResidentDetailsParams { id: string; tab?: string }
export interface ResidentDetailsProps  { resident: ResidentResource; sections: ResidentDetailsSections; meta: PageMeta }

export const routes = {
  residentDetails: {
    pattern: "/residents/:id",
    navigate(params: ResidentDetailsParams, opts?: NavigateOptions): Promise<NavigationResult>;
    href(params: ResidentDetailsParams): string;
    prefetch(params: ResidentDetailsParams): Promise<void>;
  },
  // ...
} as const;
```

### Wire form (envelope `route` field)

```json
{ "name": "ResidentDetailsRoute", "params": { "id": "...", "tab": null } }
```

### Rules
- Exactly one `[Route]` per `AppRoute<,>`. The pattern follows ASP.NET route template syntax.
- `Params` and `Props` MUST be `sealed record` and MUST be nested types.
- URL strings never appear in app code beyond the `[Route]` attribute and never appear in TS app code at all.
- The TS key is the route type name with the `Route` suffix stripped and camelCased (`ResidentDetailsRoute` → `residentDetails`). Override with `[Route("/...", Name = "residentDetails")]`.

---

## 3. Params — canonical form

### Purpose
A `Params` record binds the URL/query/header inputs into a typed value before the page handler runs.

### C# canonical form

```csharp
public sealed record Params
{
    [FromRoute]                      public required Guid     Id      { get; init; }
    [FromQuery("tab")]               public          string?  Tab     { get; init; }
    [FromQuery("page")]              public          int      Page    { get; init; } = 1;
    [FromHeader("X-Locale")]         public          string?  Locale  { get; init; }
}
```

### Generated TypeScript

```ts
export interface ResidentDetailsParams {
  id: string;             // Guid -> string (canonical form)
  tab?: string;
  page?: number;
  locale?: string;
}
```

### Rules
- All members are `required` or have a default; nullable means "optional in the URL".
- Allowed source attributes: `[FromRoute]`, `[FromQuery]`, `[FromHeader]`. No `[FromBody]` (that's for Actions).
- Type mapping (canonical):
  - `Guid`, `DateOnly`, `DateTimeOffset`, `TimeSpan` → `string` (ISO).
  - `decimal`, `long` → `string` (precision-safe).
  - `int`, `short`, `byte`, `double`, `float`, `bool` → native.
  - `enum` → string literal union.
  - `T[]`, `IReadOnlyList<T>` → `T[]`.
- Validation on params is structural (parse failure → `400`); semantic validation belongs on Actions.

---

## 4. Props — canonical form (capability graphs)

### Purpose
A `Props` record is **not a DTO**. It is the executable surface a page exposes to the client: data, navigation, actions, sections, capabilities, links, and metadata, all typed.

### C# canonical form

```csharp
public sealed record Props
{
    // Data
    public required ResidentResource            Resident { get; init; }

    // Subsections (independently reloadable)
    public required Sections                    Sections { get; init; }

    // Page-scoped capabilities (project, don't recompute on the client)
    public required Permissions                 Permissions { get; init; }

    // Page-scoped routes
    public required Links                       Links { get; init; }

    // Page-scoped metadata (cache, locale, title)
    public required PageMeta                    Meta { get; init; }
}

public sealed record Permissions : IAppPermissions
{
    public required bool CanEditResident   { get; init; }
    public required bool CanArchiveResident{ get; init; }
}

public sealed record Links : IAppLinks
{
    public required RouteRef<ResidentEditRoute.Params>     Edit     { get; init; }
    public required RouteRef<ResidentsListRoute.Params>    Back     { get; init; }
}
```

### Generated TypeScript

```ts
export interface ResidentDetailsProps {
  resident:    ResidentResource;
  sections:    {
    activity: SectionHandle<ResidentActivityProps>;
    carePlan: SectionHandle<ResidentCarePlanProps>;
  };
  permissions: { canEditResident: boolean; canArchiveResident: boolean };
  links:       { edit: RouteHandle<ResidentEditParams>; back: RouteHandle<ResidentsListParams> };
  meta:        PageMeta;
}
```

`SectionHandle` and `RouteHandle` are live objects (see §6, §7).

### What Props MUST NOT contain
- Raw URL strings for navigation. Use `RouteRef`.
- Client-only mutation state.
- Methods that are not modeled as Actions.
- Recomputed permissions; capabilities are server-projected.

### Generator lint
A page whose Props has only `data` and no capability surface raises `ALIS0001 Props is degenerate`. The escape hatch is `[AllowDtoProps]` on the route, intended only for read-only export endpoints.

---

## 5. Resource — canonical form

### Purpose
A `Resource` is the canonical capability graph of a domain entity. It is the unit of cross-cutting reuse: a `ResidentResource` carries its identity, data, links, actions, and permissions wherever it appears in props.

### C# canonical form

```csharp
[Resource("Resident")]
public sealed record ResidentResource : AppResource<ResidentResource.Identity>
{
    public required Identity      Id          { get; init; }
    public required Data          Resident    { get; init; }
    public required Links         Links       { get; init; }
    public required Actions       Actions     { get; init; }
    public required Permissions   Permissions { get; init; }

    public sealed record Identity(Guid Value) : IResourceIdentity;

    public sealed record Data
    {
        public required string     FirstName  { get; init; }
        public required string     LastName   { get; init; }
        public          DateOnly?  Dob        { get; init; }
        public required string     Status     { get; init; }
    }

    public sealed record Links : IAppLinks
    {
        public required RouteRef<ResidentDetailsRoute.Params>     Details   { get; init; }
        public required RouteRef<ResidentEditRoute.Params>        Edit      { get; init; }
        public required RouteRef<ResidentCarePlanRoute.Params>    CarePlan  { get; init; }
    }

    public sealed record Actions : IAppActions
    {
        public required ActionRef<SaveResident>     Save     { get; init; }
        public required ActionRef<ArchiveResident>  Archive  { get; init; }
    }

    public sealed record Permissions : IAppPermissions
    {
        public required bool CanEdit    { get; init; }
        public required bool CanArchive { get; init; }
    }
}
```

### Generated TypeScript

```ts
export interface ResidentResource {
  // Identity is a branded type, never a raw string
  id: ResourceId<"Resident">;
  resident:    { firstName: string; lastName: string; dob?: string; status: string };
  links:       { details: RouteHandle<ResidentDetailsParams>;
                 edit:    RouteHandle<ResidentEditParams>;
                 carePlan:RouteHandle<ResidentCarePlanParams> };
  actions:     { save:    ActionHandle<SaveResidentRequest, SaveResidentResponse>;
                 archive: ActionHandle<ArchiveResidentRequest, ArchiveResidentResponse> };
  permissions: { canEdit: boolean; canArchive: boolean };
}

// usage
resident.links.details.navigate();
resident.actions.save.execute({ firstName: "..." });
resident.permissions.canEdit;
```

### Wire form

```json
{
  "$kind": "resource",
  "type":  "Resident",
  "id":    { "value": "5f8e..." },
  "resident":    { "firstName": "...", "lastName": "...", "dob": null, "status": "Active" },
  "links": {
    "details":  { "$kind": "route",  "id": "ResidentDetailsRoute", "params": { "id": "5f8e..." } },
    "edit":     { "$kind": "route",  "id": "ResidentEditRoute",    "params": { "id": "5f8e..." } }
  },
  "actions": {
    "save":     { "$kind": "action", "id": "residents.save",       "context": { "id": "5f8e..." } },
    "archive":  { "$kind": "action", "id": "residents.archive",    "context": { "id": "5f8e..." } }
  },
  "permissions": { "canEdit": true, "canArchive": false }
}
```

### Rules
- A `Resource` MUST have an `Identity` nested record implementing `IResourceIdentity`.
- A `Resource` MAY appear nested inside other Resources or Props; the wire form is the same.
- Permissions are projected, not derived client-side.
- Links use `RouteRef`; the params are filled by the server, not constructed on the client.

---

## 6. Section — canonical form

### Purpose
A `Section` is a named, independently reloadable, optionally streamable region inside a page. Sections decouple data freshness within a page from full navigations.

### C# canonical form

```csharp
[Section("Activity")]
public sealed partial class ResidentActivitySection
    : AppSection<ResidentActivitySection.Params, ResidentActivitySection.Props>
{
    public sealed record Params { public required Guid ResidentId { get; init; } }

    public sealed record Props
    {
        public required IReadOnlyList<ActivityItem> Items { get; init; }
        public required RouteRef<ActivityDetailsRoute.Params> Open { get; init; }
    }

    public sealed class Handler : AppSectionHandler<Params, Props>
    {
        public Handler(IActivityRepository repo) { ... }

        public override async Task<Props> Render(Params p, AppSectionContext ctx)
        {
            var items = await repo.For(p.ResidentId);
            return new Props { Items = items, Open = ctx.Routes.For<ActivityDetailsRoute>() };
        }
    }
}
```

A page declares its sections in its `Sections` record (see §2). The page handler decides which to render eagerly, which to stream, and which to defer.

### Generated TypeScript

```ts
// from props.sections.activity
const handle: SectionHandle<ResidentActivityProps> = props.sections.activity;

handle.props;            // the current section props (typed)
handle.reload();         // re-fetch only this section
handle.invalidate();     // mark dirty; refetch on next focus / navigation
handle.subscribe(fn);    // observe prop changes
```

### Wire form

When carried inside Props:

```json
{
  "activity": {
    "$kind":  "section",
    "id":     "ResidentActivity",
    "params": { "residentId": "5f8e..." },
    "props":  { "items": [...], "open": { "$kind": "route", "id": "ActivityDetailsRoute", "params": {...} } },
    "etag":   "W/\"af71...\""
  }
}
```

When reloaded standalone, the response is a partial envelope (§13).

### Rules
- A section's `Params` MUST be derivable from the parent page's `Params` and `Resource` identities (no out-of-band data).
- A section MAY be streamed by returning `AppView.Stream(...)` from its handler.
- A section's `etag` is computed by the framework from the canonicalized props; the client uses it to avoid redundant reloads.

---

## 7. Page handler — canonical form

### Purpose
A `Page` is the server handler that takes route params and produces an `AppView<TProps>`. It is where the typed graph is composed for a single navigation.

### C# canonical form

```csharp
public sealed class ResidentDetailsPage
    : AppPage<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public ResidentDetailsPage(
        IResidentService residents,
        IAppLinks links,
        IAppCapabilities caps) { ... }

    public override async Task<AppView<ResidentDetailsRoute.Props>> Render(
        ResidentDetailsRoute.Params @params,
        AppPageContext ctx)
    {
        var resident = await residents.Project<ResidentResource>(@params.Id, ctx);

        var props = new ResidentDetailsRoute.Props
        {
            Resident    = resident,
            Permissions = ctx.Capabilities.Project<ResidentDetailsRoute.Props.Permissions>(resident),
            Links       = ctx.Links.For<ResidentDetailsRoute.Props.Links>(@params),
            Sections    = new()
            {
                Activity = ctx.Sections.Ref<ResidentActivitySection>(new() { ResidentId = @params.Id }),
                CarePlan = ctx.Sections.Ref<ResidentCarePlanSection>(new() { ResidentId = @params.Id }),
            },
            Meta        = ctx.Meta.Default(title: $"{resident.Resident.FirstName} {resident.Resident.LastName}"),
        };

        return App.View(props)
            .WithLayout<TenantShellLayout>()
            .Stream(p => p.Sections.Activity)
            .Stream(p => p.Sections.CarePlan)
            .Cache(TimeSpan.FromSeconds(15));
    }
}
```

### `AppView<TProps>` canonical surface

```csharp
AppView<TProps> WithLayout<TLayout>() where TLayout : AppLayout<TProps>;
AppView<TProps> WithRenderer(string renderer);
AppView<TProps> WithIsland<TIsland>(string slot, object props);
AppView<TProps> Stream(Expression<Func<TProps, SectionRef>> selector);
AppView<TProps> Defer(Expression<Func<TProps, SectionRef>> selector);   // load on visible
AppView<TProps> Cache(TimeSpan ttl);
AppView<TProps> Header(string name, string value);
AppView<TProps> Status(int code);
AppView<TProps> Redirect<TRoute>(object @params);
```

### Rules
- A `Page` is the only place in app code where Resources are *constructed*; everywhere else they are *passed*.
- `AppView` is immutable; each builder method returns a new view.
- A page returning `Redirect<TRoute>(...)` short-circuits all rendering; no props are emitted.

---

## 8. Layout — canonical form

### Purpose
A `Layout` is a composable server component with named slots and per-slot persistence policy. Persistent slots survive partial reloads; transient slots are replaced.

### C# canonical form

```csharp
public sealed class TenantShellLayout : AppLayout<TenantShellLayout.Props>
{
    public sealed record Props
    {
        public required TenantHeaderProps Header { get; init; }
        public required NavTreeProps      Nav    { get; init; }
    }

    public override LayoutSlots DeclareSlots() => new()
    {
        { "header", SlotPolicy.Persistent },
        { "nav",    SlotPolicy.Persistent },
        { "main",   SlotPolicy.Transient  },
        { "aside",  SlotPolicy.Transient  },
    };

    public override async Task<LayoutRender> Render(Props p, AppLayoutContext ctx) { ... }
}
```

### Wire form (envelope `layout` field)

```json
{
  "layout": {
    "id":   "TenantShellLayout",
    "slots":  { "header": "persistent", "nav": "persistent", "main": "transient", "aside": "transient" }
  }
}
```

### Rules
- A page binds its layout via `WithLayout<T>()` on `AppView`.
- A partial reload that targets only transient slots MUST NOT re-render persistent slots.
- A layout switch (different layout id) downgrades the navigation to a `full` strategy.

---

## 9. Component identity — canonical form

### Purpose
Every server-renderable thing has a stable string identity. Identity is the join key between server emit, client manifest, and AI tooling.

### C# canonical form

```csharp
// default identity = "App.Features.Residents.ResidentDetailsPage"
public sealed class ResidentDetailsPage : AppPage<...> { }

// override
[ComponentIdentity("residents.details")]
public sealed class ResidentDetailsPage : AppPage<...> { }
```

### Rules
- Identity is generated from namespace + type name when no attribute is given.
- Renames MUST be accompanied by a temporary `[ComponentIdentity(...)]` retaining the old id, or by a graph migration entry (§24).

---

## 10. Island — canonical form

### Purpose
An `Island` is a client-hydrated component embedded in SSR output. Each island is a separate hydration unit, renderer, chunk, and lifecycle.

### C# canonical form (declaration)

```csharp
[Island("ResidentScheduler")]
[Renderer("vue")]
[Chunk("scheduler")]
public sealed class ResidentSchedulerIsland : AppIsland<ResidentSchedulerIsland.Props>
{
    public sealed record Props
    {
        public required ResourceRef<ResidentResource>   Resident      { get; init; }
        public required IReadOnlyList<Appointment>      Appointments  { get; init; }
        public required ActionRef<RescheduleAppointment>Reschedule    { get; init; }
    }

    public override IslandManifest Manifest => new()
    {
        Mode = HydrationMode.Visible,
        FallbackComponent = typeof(ResidentSchedulerSkeleton),
    };
}
```

### C# canonical form (usage from a page)

```csharp
return App.View(props)
    .WithIsland<ResidentSchedulerIsland>(
        slot: "main",
        props: new ResidentSchedulerIsland.Props
        {
            Resident     = ctx.Refs.Resource<ResidentResource>(props.Resident.Id),
            Appointments = appointments,
            Reschedule   = ctx.Refs.Action<RescheduleAppointment>(),
        });
```

### Markup contract (server-emitted)

```html
<alis-island
  data-name="ResidentScheduler"
  data-renderer="vue"
  data-chunk="scheduler.abcd1234"
  data-mode="visible"
  data-props-ref="island-3">
  <!-- SSR-rendered fallback HTML -->
</alis-island>
<script type="application/json" data-island-props="island-3">{ "...": "..." }</script>
```

### Generated TypeScript (renderer-agnostic mount)

```ts
// generated islands manifest
export const islands = {
  ResidentScheduler: {
    renderer: "vue",
    chunk:    () => import("./chunks/scheduler.abcd1234.js"),
    mode:     "visible" as HydrationMode,
  },
  // ...
} as const;
```

### Hydration modes

| Mode | Triggered by |
|------|--------------|
| `eager` | Runtime boot, document order. |
| `visible` | IntersectionObserver intersection. |
| `idle` | `requestIdleCallback`. |
| `interaction` | First user interaction on placeholder. |
| `media` | Media query match. |

Hydration is cancelable; if the placeholder is removed before mount completes, the renderer adapter MUST observe an `aborted` mount handle and free the chunk reference.

---

## 11. Renderer — canonical form

### Purpose
A `Renderer` is a client-side mount/update/unmount adapter for a specific framework (React, Vue, Solid, Preact). The protocol is renderer-agnostic; renderers register against the same contract.

### Generated TypeScript contract

```ts
export interface ClientRenderer {
  readonly id: "react" | "vue" | "solid" | "preact" | string;
  mount(target: Element, component: ComponentRef, props: unknown, ctx: RenderCtx): MountHandle;
  update(handle: MountHandle, props: unknown): void;
  unmount(handle: MountHandle): void;
}

export interface MountHandle { readonly dispose: () => void; readonly aborted: boolean }
export interface ComponentRef { readonly id: string; readonly load: () => Promise<unknown> }
export interface RenderCtx    { readonly bus: EventBus; readonly cache: Cache; readonly tenant: TenantInfo }
```

### Server-side registration

```csharp
opt.AddRenderer<ReactRenderer>("react");
opt.AddRenderer<VueRenderer>("vue").Default();   // default for islands with no [Renderer]
```

### Rules
- A page or island pinned to a renderer that is not registered MUST cause startup to fail loudly (no silent fallback).
- Mixed renderers on a single page are supported and MUST NOT cause duplicate runtime chunks.

---

## 12. Action — canonical form

### Purpose
An `Action` is a typed server mutation, identified by a stable action id, with a typed request, response, optional validator, optional optimistic projection, idempotency mode, and a policy.

### C# canonical form

```csharp
[Action("residents.save")]
[Idempotent(IdempotencyMode.ClientKey)]
public sealed partial class SaveResident
    : AppAction<SaveResident.Request, SaveResident.Response>
{
    public override AuthPolicy Policy => Policies.EditResident;

    public sealed record Request
    {
        public required Guid     Id          { get; init; }
        public required string   FirstName   { get; init; }
        public required string   LastName    { get; init; }
        public          DateOnly?Dob         { get; init; }
    }

    public sealed record Response
    {
        public required ResidentResource Resident { get; init; }
    }

    public sealed class Validator : AppValidator<Request>
    {
        public Validator()
        {
            RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
            RuleFor(x => x.LastName ).NotEmpty().MaximumLength(100);
            RuleFor(x => x.Dob      ).Before(_ => DateOnly.FromDateTime(DateTime.UtcNow));
        }
    }

    [Optimistic]
    public sealed class Optimism : AppOptimism<Request, ResidentResource>
    {
        public override ResidentResource Project(ResidentResource current, Request req) => current with
        {
            Resident = current.Resident with
            {
                FirstName = req.FirstName,
                LastName  = req.LastName,
                Dob       = req.Dob,
            }
        };
    }

    public sealed class Handler : AppActionHandler<Request, Response>
    {
        public Handler(IResidentService residents) { ... }

        public override async Task<AppActionResult<Response>> Handle(Request req, AppActionContext ctx)
        {
            var updated = await residents.Save(req, ctx.Cancellation);

            ctx.Invalidate<ResidentActivitySection>(new() { ResidentId = req.Id });
            ctx.Invalidate<ResidentResource>(new() { Value = req.Id });

            return AppActionResult.Ok(new Response { Resident = updated });
        }
    }
}
```

### Generated TypeScript

```ts
// either standalone:
import { actions } from "@app/generated";
await actions.residents.save.execute({ id, firstName, lastName });

// or via a bound ActionRef on a Resource:
await resident.actions.save.execute({ firstName, lastName });
resident.actions.save.optimistic({ firstName: "Pending..." });    // local projection
```

### Wire form (request)

```http
POST /__alis/action HTTP/1.1
X-Alis-Action:           residents.save
X-Alis-Idempotency-Key:  9b2c-...
X-Alis-Route:            ResidentDetailsRoute
X-Alis-Section:          (none)
X-Alis-Asset-Version:    2026.05.22.1
Content-Type:            application/json

{ "id": "5f8e...", "firstName": "...", "lastName": "...", "dob": null }
```

### Wire form (response)

```json
{
  "protocol":   { "version": "1" },
  "result":     { "resident": { "$kind": "resource", "type": "Resident", "id": {...}, "...": "..." } },
  "invalidate": [
    { "$kind": "section",  "id": "ResidentActivity", "params": { "residentId": "5f8e..." } },
    { "$kind": "resource", "type": "Resident",       "id": { "value": "5f8e..." } }
  ],
  "redirect":   null
}
```

### Idempotency modes
- `None` — duplicate calls execute again.
- `ClientKey` — the client sends `X-Alis-Idempotency-Key`; the server deduplicates within a TTL.
- `ServerHash` — the server derives a key from `(actionId, principal, canonical(request))`.

### Rules
- The action id is the wire identity. The C# type name is irrelevant to the wire.
- `Policy` is enforced before `Validator`. Validator failures yield `422`; policy failures yield `403`.
- `Invalidate` calls are aggregated into the response envelope; the client applies them after a successful commit.
- Optimism is opt-in via a nested `[Optimistic] AppOptimism<TRequest, TTarget>`.

---

## 13. Validation — canonical form

### Purpose
Validation rules are server-canonical, expressed once, and projected to the client as **metadata** (never executable code).

### C# canonical form (rule expression)

```csharp
public sealed class Validator : AppValidator<SaveResident.Request>
{
    public Validator()
    {
        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.LastName ).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Dob)
            .Before(_ => DateOnly.FromDateTime(DateTime.UtcNow))
            .When(x => x.Dob is not null);

        RuleFor(x => x.FirstName)
            .Custom((value, ctx) =>
            {
                if (value.Contains("..")) ctx.Fail("firstName.invalid");
            });
    }
}
```

### Generated TypeScript (metadata tree)

```ts
export const validators = {
  saveResidentRequest: {
    firstName: [
      { kind: "required" },
      { kind: "maxLength", value: 100 },
      { kind: "custom", id: "firstName.invalid" }   // opaque; server round-trip required
    ],
    lastName: [
      { kind: "required" },
      { kind: "maxLength", value: 100 }
    ],
    dob: [
      { kind: "date" },
      { kind: "before", value: "today", when: "dob != null" }
    ]
  }
} as const;
```

### Supported rule kinds (v1)

`required`, `minLength`, `maxLength`, `pattern`, `min`, `max`, `email`, `url`, `oneOf`, `date`, `before`, `after`, `equalsField`, `differsField`, `requiredWhen`, `forbiddenWhen`, `custom`.

`custom` rules are opaque — the client surfaces them only as a server round-trip; never as a local pass/fail.

### Rules
- Validators are nested inside their owning Action; one validator per request type.
- The metadata tree is keyed by request type id, not the C# name.
- Conditional rules (`When`, `Unless`) are emitted as a `when` predicate expressed against the request shape, in a tiny canonical predicate language (`==`, `!=`, `&&`, `||`, `null`, field paths). Anything more complex degrades to `custom`.

---

## 14. Partial reload — canonical form

### Purpose
Update a named section without re-running the entire page.

### C# canonical form (server)

```csharp
public override async Task<AppView<...>> Render(...)
{
    if (ctx.Strategy == NavigationStrategy.Partial && ctx.RequestedSection == "Activity")
        return App.Partial<ResidentActivitySection>(new() { ResidentId = @params.Id });

    return App.View(props).Stream(p => p.Sections.Activity);
}
```

Or, more idiomatically, the section handler is called directly by the framework when the client requests `?section=Activity`; the page does not need explicit branching.

### Generated TypeScript

```ts
await resident.sections.activity.reload();        // re-fetch only Activity
resident.sections.activity.invalidate();          // mark dirty
resident.sections.activity.subscribe(p => ...);   // observe
```

### Wire form (request)

```http
GET /residents/5f8e...?section=Activity HTTP/1.1
X-Alis-Strategy:       partial
X-Alis-Asset-Version:  2026.05.22.1
```

### Wire form (response)

```json
{
  "protocol":   { "version": "1" },
  "partials": [
    {
      "section": "ResidentActivity",
      "component": "App.Features.Residents.Sections.ResidentActivitySection",
      "renderer": "react",
      "props":    { "items": [...], "open": { "$kind": "route", "id": "...", "params": {...} } },
      "chunks":   ["activity-section.7c12.js"],
      "etag":     "W/\"af71...\""
    }
  ],
  "props":      null,
  "stream":     false,
  "status":     200
}
```

### Rules
- A partial response MUST set `props` to `null` and MUST populate `partials`.
- A partial response MAY carry multiple section updates.
- A section's `etag` MAY be returned with `304 Not Modified` headers when unchanged.

---

## 15. Streaming SSR — canonical form

### Purpose
Render a shell immediately and stream named section payloads as they complete, in any order, with per-section error isolation.

### C# canonical form

```csharp
return App.View(props)
    .WithLayout<TenantShellLayout>()
    .Stream(p => p.Sections.Activity)
    .Stream(p => p.Sections.CarePlan)
    .Defer(p => p.Sections.Audit);   // load lazily, not part of stream
```

### Wire form (NDJSON frames)

```
{"$frame":"shell","layout":"TenantShellLayout","renderer":"react","props":{...},"chunks":[...],"preload":[...]}
{"$frame":"chunk","section":"ResidentCarePlan","props":{...},"chunks":[...]}
{"$frame":"chunk","section":"ResidentActivity","props":{...},"chunks":[...]}
{"$frame":"error","section":"ResidentAudit","error":{"code":"Timeout"}}
{"$frame":"end"}
```

- `Content-Type: application/x-ndjson`
- `Transfer-Encoding: chunked`
- One JSON object per line.

### Rules
- The `shell` frame MUST be flushed before any section starts work that depends on I/O.
- Each section is a `Task`; failures are isolated as `error` frames.
- The stream is canceled atomically if the client disconnects; the framework MUST propagate cancellation to each section's handler.

---

## 16. Navigation — canonical form

### Purpose
Typed navigation, with cache, prefetch, and strategy selection. No string URLs in app code.

### Generated TypeScript

```ts
import { routes } from "@app/generated";

await routes.residentDetails.navigate({ id });
await routes.residentDetails.navigate({ id }, { replace: true });
await routes.residentDetails.prefetch({ id });
const href = routes.residentDetails.href({ id });   // when an <a> truly needs a string
```

### Lifecycle (canonical)

```
navigate(req)
  -> resolveRoute(req)
  -> selectStrategy(current, target)             // full | partial | stream
  -> emit "navigation.start"
  -> loadChunks(target.chunks)                   // parallel, honors preload
  -> fetchEnvelope(target, strategy)
  -> applyEnvelope(envelope)                     // mount or patch islands
  -> commit(target)                              // history, scroll, focus
  -> emit "navigation.commit"
```

### Strategy headers (client → server)

| Header | Meaning |
|--------|---------|
| `X-Alis-Strategy` | `full` \| `partial` \| `stream` |
| `X-Alis-Asset-Version` | The client's loaded asset version (for skew detection) |
| `X-Alis-Section` | When `partial`, the requested section id |
| `X-Alis-Trace` | Traceparent |
| `X-Alis-Tenant` | Resolved tenant id (when client-side known) |

### Rules
- A newer `navigate` MUST cancel any in-flight older navigation atomically.
- Cancellation MUST NOT mutate `history` or scroll position.
- Back/forward MUST replay cached envelopes when valid; otherwise re-fetch with `X-Alis-Strategy: full`.

---

## 17. Cache and invalidation — canonical form

### Purpose
Reduce redundant fetches without making the client the source of truth.

### Generated TypeScript

```ts
// envelope cache
client.cache.get(routes.residentDetails, { id });
client.cache.put(routes.residentDetails, { id }, envelope);
client.cache.invalidate(routes.residentDetails, { id });

// section-scoped invalidation
client.cache.invalidate(sections.residentActivity, { residentId });

// resource-scoped invalidation
client.cache.invalidate(resources.resident, { value: residentId });
```

### Cache rules
- Cache entries are keyed by `(routeId, canonical(params))`.
- Entries are pinned to an `assetVersion`; entries from a stale version are discarded on startup.
- `props.meta.cache.maxAge` sets per-route TTL; default is 0 (no cache) unless the route opts in via `.Cache(...)`.
- `invalidate(...)` cascades to all section and resource refs nested in cached envelopes.

### Server-driven invalidation (within a single response)

The action response (§12) and the streaming `end` frame MAY carry an `invalidate` array of typed refs. The client applies them after commit.

---

## 18. Optimistic mutation — canonical form

### Purpose
Apply a local projection of a mutation immediately, with deterministic rollback on failure.

### C# canonical form

```csharp
[Optimistic]
public sealed class Optimism : AppOptimism<SaveResident.Request, ResidentResource>
{
    public override ResidentResource Project(ResidentResource current, SaveResident.Request req) =>
        current with { Resident = current.Resident with { FirstName = req.FirstName, LastName = req.LastName } };
}
```

### Generated TypeScript

```ts
const projection = resident.actions.save.optimistic({ firstName, lastName });
try {
    await resident.actions.save.execute({ firstName, lastName });
} catch {
    projection.rollback();          // automatic if execute returns { error }
}
```

### Rules
- Optimism is opt-in per action; the framework provides no implicit optimism.
- A failed `execute` MUST roll back exactly the projection created with the same call id.
- Multiple optimistic calls on the same resource are stacked; rollback unwinds them in reverse.

---

## 19. Error envelope — canonical form

### Wire form

```json
{
  "error": {
    "code":       "ValidationFailed",
    "message":    null,
    "fields":     { "firstName": ["required"] },
    "retryAfter": null,
    "traceId":    "00-af7c...-...-01"
  }
}
```

### Categories (v1)
`ValidationFailed (422)`, `NotAuthenticated (401)`, `NotAuthorized (403)`, `NotFound (404)`, `Conflict (409)`, `AssetVersionMismatch (409)`, `RateLimited (429)`, `ServerError (500)`.

### C# canonical form (raise from a handler)

```csharp
return AppActionResult.Fail(ErrorCode.Conflict, fields: null, retryAfter: TimeSpan.FromSeconds(2));
```

### Generated TypeScript

```ts
const result = await resident.actions.save.execute({...});
if ("error" in result) {
    switch (result.error.code) {
        case "ValidationFailed": form.applyFieldErrors(result.error.fields); break;
        case "Conflict":         showRetry(result.error.retryAfter); break;
        case "AssetVersionMismatch": location.reload(); break;
    }
}
```

### Rules
- Actions return `{ result } | { error }`; they do not throw for expected errors.
- `ValidationFailed.fields` is a map of field path → array of message keys (not human strings).
- `AssetVersionMismatch` is always client-handled via full reload; app code does not need to handle it explicitly.

---

## 20. Policy / Auth — canonical form

### Purpose
Declare server-enforced authorization, project capabilities into props, and prevent client invocation of forbidden actions at the generated-affordance level.

### C# canonical form

```csharp
public static class Policies
{
    public static readonly AuthPolicy EditResident    = new("EditResident");
    public static readonly AuthPolicy ArchiveResident = new("ArchiveResident");
}

public sealed class EditResidentPolicy : IAppPolicy
{
    public string Name => Policies.EditResident.Name;

    public Task<AuthDecision> Authorize(AuthContext ctx) =>
        Task.FromResult(ctx.User.HasClaim("perm", "resident.edit")
            ? AuthDecision.Allow
            : AuthDecision.Deny);
}
```

### Attaching policies

```csharp
[Policy("EditResident")]
public sealed partial class SaveResident : AppAction<...> { }

[Policy("EditResident")]
public sealed class ResidentEditRoute : AppRoute<...> { }

[Policy("EditResident")]
public sealed class ResidentEditSection : AppSection<...> { }
```

### Capability projection into Resources

```csharp
public sealed record Permissions : IAppPermissions
{
    public required bool CanEdit    { get; init; }     // projected from EditResident policy
    public required bool CanArchive { get; init; }     // projected from ArchiveResident policy
}
```

The page handler (or framework default) fills these from the active `AuthContext`. The client treats them as the source of truth for affordances.

### Generated TypeScript

```ts
if (resident.permissions.canEdit) {
    resident.actions.save.execute({...});
}
```

The generator additionally emits a development-time guard: invoking an action whose declared policy resolves to `false` in the current props logs a warning and (in dev) throws. The server enforcement remains the real boundary.

---

## 21. Tenant — canonical form

### Purpose
Resolve a tenant per request, scope assets and localization, and propagate the tenant through the envelope.

### C# canonical form

```csharp
opt.AddTenants(t =>
{
    t.ResolveFromSubdomain();
    t.AddOverride("acme",   o => { o.AssetVersion = "2026.05.22.1-acme"; o.Locale = "en-US"; });
    t.AddOverride("globex", o => { o.AssetVersion = "2026.05.22.1-globex"; o.Locale = "en-GB"; });
});
```

### Envelope projection

```json
{ "meta": { "tenant": "acme", "locale": "en-US" } }
```

### Rules
- A tenant override pins to a specific `assetVersion` already published; pinning to an unpublished version raises a startup error.
- A locale switch is a partial reload when the layout permits, full otherwise.

---

## 22. Localization — canonical form

### Purpose
Strongly typed string keys derived from .NET constants; runtime resolution is tenant + locale.

### C# canonical form

```csharp
public static class ResidentKeys : IAppLocalizationKeys
{
    public static readonly LocalizationKey Title    = "residents.details.title";
    public static readonly LocalizationKey SaveBtn  = "residents.details.save";
    public static readonly LocalizationKey Required = "validation.required";
}
```

### Generated TypeScript

```ts
export const keys = {
  residents: {
    details: { title: "residents.details.title", save: "residents.details.save" }
  },
  validation: { required: "validation.required" }
} as const;
```

### Wire (resolution catalog)

The shell envelope carries the resolved catalog for the keys actually used by the page:

```json
{ "meta": { "i18n": { "residents.details.title": "Resident — {name}" } } }
```

Lazy keys are fetched on demand via `GET /__alis/i18n?keys=...&locale=...`.

---

## 23. Asset version / Chunk — canonical form

### Purpose
A single, deterministic identifier ties a server build to a chunk manifest; deployment skew is detected and handled.

### Asset manifest (`alis.assets.json`)

```json
{
  "assetVersion": "2026.05.22.1",
  "renderers":    { "react": "react.abcd1234.js", "vue": "vue.7c12dead.js" },
  "components": {
    "App.Features.Residents.ResidentDetailsPage": {
      "chunk":   "residents-details.ef561234.js",
      "css":     ["residents-details.ef561234.css"],
      "preload": ["residents-grid.ab981234.js"]
    }
  },
  "chunks": { "residents-details.ef561234.js": { "file": "...", "imports": [...], "css": [...] } }
}
```

### Envelope projection

```json
{ "assetVersion": "2026.05.22.1", "chunks": ["residents-details.ef561234.js"], "preload": ["..."] }
```

### Rules
- The SSR process pins one `assetVersion` for its entire lifetime.
- An envelope's `assetVersion` MUST match the process's pinned value.
- A client envelope with a stale `assetVersion` MUST trigger a full reload via the `AssetVersionMismatch` flow (§19).
- The server MUST keep the last N manifests addressable (default N=2) so an in-flight stale navigation can complete before the forced reload.

---

## 24. Graph export — canonical form

### Purpose
A machine-readable representation of the entire application graph, suitable for AI tooling, codemods, and external clients.

### Wire form (`GET /__alis/graph`, admin-gated)

```json
{
  "version": "1",
  "assetVersion": "2026.05.22.1",
  "routes": [
    {
      "id": "ResidentDetailsRoute",
      "pattern": "/residents/{id:guid}",
      "params":  { "id": "Guid", "tab": "string?" },
      "props":   { "$ref": "ResidentDetailsRoute.Props" },
      "policy":  null,
      "layout":  "TenantShellLayout",
      "renderer":"react",
      "chunks":  ["residents-details.ef561234.js"]
    }
  ],
  "actions": [
    {
      "id": "residents.save",
      "request":     { "$ref": "SaveResident.Request" },
      "response":    { "$ref": "SaveResident.Response" },
      "policy":      "EditResident",
      "idempotency": "ClientKey",
      "optimistic":  true,
      "invalidates": [
        { "$kind": "section",  "id": "ResidentActivity" },
        { "$kind": "resource", "type": "Resident" }
      ]
    }
  ],
  "resources": [
    {
      "id": "Resident",
      "identity": { "value": "Guid" },
      "links":    { "details": "ResidentDetailsRoute", "edit": "ResidentEditRoute" },
      "actions":  ["residents.save", "residents.archive"],
      "permissions": ["canEdit", "canArchive"]
    }
  ],
  "sections":  [...],
  "components":[...],
  "policies":  [...],
  "validators":{...},
  "localization": {...},
  "renderers": ["react","vue"],
  "affordances": ["navigate","href","prefetch","execute","optimistic","reload","invalidate"]
}
```

### Rules
- Every node carries a stable string id.
- The shape is versioned; a `version` bump is required for breaking changes.
- The endpoint is policy-gated; no anonymous access in any environment.

---

## 25. `App.Render` — the headline entry point

### Purpose
The single ergonomic developer surface for returning a typed page from server code.

### C# canonical form (page handler)

```csharp
return App.Render<ResidentDetailsRoute.Props>(props)
    .WithLayout<TenantShellLayout>()
    .Stream(p => p.Sections.Activity)
    .Defer(p => p.Sections.Audit);
```

### Equivalent partial form

```csharp
return App.RenderPartial<ResidentActivitySection.Props>(props);
```

### Equivalent streaming form

```csharp
return App.RenderStream<ResidentDetailsRoute.Props>(propsBuilder => propsBuilder
    .Shell(shellProps)
    .Stream(p => p.Sections.Activity, () => activityTask)
    .Stream(p => p.Sections.CarePlan, () => carePlanTask));
```

### Rules
- `App.Render`, `App.RenderPartial`, `App.RenderStream` are the only sanctioned entry points.
- The page handler MUST return an `AppView<TProps>` (or `Redirect<TRoute>`); raw HTTP responses are forbidden in app code.

---

## 26. Worked example — `Resident` end-to-end

This is the canonical proof: one Resource expressed through every layer with no glue.

### 26.1 C# (the entire surface)

```csharp
namespace App.Features.Residents;

// Resource
[Resource("Resident")]
public sealed record ResidentResource : AppResource<ResidentResource.Identity>
{
    public required Identity      Id          { get; init; }
    public required Data          Resident    { get; init; }
    public required Links         Links       { get; init; }
    public required Actions       Actions     { get; init; }
    public required Permissions   Permissions { get; init; }

    public sealed record Identity(Guid Value) : IResourceIdentity;
    public sealed record Data { public required string FirstName { get; init; } public required string LastName { get; init; } }
    public sealed record Links : IAppLinks
    {
        public required RouteRef<ResidentDetailsRoute.Params> Details { get; init; }
        public required RouteRef<ResidentEditRoute.Params>    Edit    { get; init; }
    }
    public sealed record Actions : IAppActions
    {
        public required ActionRef<SaveResident>    Save    { get; init; }
        public required ActionRef<ArchiveResident> Archive { get; init; }
    }
    public sealed record Permissions : IAppPermissions
    {
        public required bool CanEdit    { get; init; }
        public required bool CanArchive { get; init; }
    }
}

// Route
[Route("/residents/{id:guid}")]
public sealed class ResidentDetailsRoute : AppRoute<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public sealed record Params { [FromRoute] public required Guid Id { get; init; } }
    public sealed record Props
    {
        public required ResidentResource Resident { get; init; }
        public required Sections         Sections{ get; init; }
    }
    public sealed record Sections : IAppSections
    {
        public required SectionRef<ResidentActivitySection.Props> Activity { get; init; }
    }
}

// Page
public sealed class ResidentDetailsPage : AppPage<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public override async Task<AppView<ResidentDetailsRoute.Props>> Render(
        ResidentDetailsRoute.Params p, AppPageContext ctx)
    {
        var resident = await ctx.Resources.Project<ResidentResource>(p.Id);
        return App.Render(new ResidentDetailsRoute.Props
        {
            Resident = resident,
            Sections = new()
            {
                Activity = ctx.Sections.Ref<ResidentActivitySection>(new() { ResidentId = p.Id }),
            },
        })
        .WithLayout<TenantShellLayout>()
        .Stream(x => x.Sections.Activity);
    }
}

// Section
[Section("ResidentActivity")]
public sealed class ResidentActivitySection : AppSection<ResidentActivitySection.Params, ResidentActivitySection.Props>
{
    public sealed record Params { public required Guid ResidentId { get; init; } }
    public sealed record Props  { public required IReadOnlyList<ActivityItem> Items { get; init; } }
    public sealed class Handler : AppSectionHandler<Params, Props> { /* ... */ }
}

// Action
[Action("residents.save"), Idempotent(IdempotencyMode.ClientKey), Policy("EditResident")]
public sealed class SaveResident : AppAction<SaveResident.Request, SaveResident.Response>
{
    public sealed record Request  { public required Guid Id { get; init; } public required string FirstName { get; init; } public required string LastName { get; init; } }
    public sealed record Response { public required ResidentResource Resident { get; init; } }
    public sealed class Validator : AppValidator<Request>
    {
        public Validator()
        {
            RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
            RuleFor(x => x.LastName ).NotEmpty().MaximumLength(100);
        }
    }
    [Optimistic] public sealed class Optimism : AppOptimism<Request, ResidentResource>
    {
        public override ResidentResource Project(ResidentResource cur, Request r) =>
            cur with { Resident = cur.Resident with { FirstName = r.FirstName, LastName = r.LastName } };
    }
    public sealed class Handler : AppActionHandler<Request, Response>
    {
        public override async Task<AppActionResult<Response>> Handle(Request r, AppActionContext ctx)
        {
            var saved = await ctx.Services.GetRequiredService<IResidentService>().Save(r, ctx.Cancellation);
            ctx.Invalidate<ResidentActivitySection>(new() { ResidentId = r.Id });
            ctx.Invalidate<ResidentResource>(new() { Value = r.Id });
            return AppActionResult.Ok(new Response { Resident = saved });
        }
    }
}

// Island
[Island("ResidentScheduler"), Renderer("vue"), Chunk("scheduler")]
public sealed class ResidentSchedulerIsland : AppIsland<ResidentSchedulerIsland.Props>
{
    public sealed record Props
    {
        public required ResourceRef<ResidentResource> Resident { get; init; }
        public required ActionRef<RescheduleAppointment> Reschedule { get; init; }
    }
}
```

### 26.2 Wire (envelope, full navigation)

```json
{
  "protocol": { "version": "1" },
  "route":    { "name": "ResidentDetailsRoute", "params": { "id": "5f8e..." } },
  "component":"App.Features.Residents.ResidentDetailsPage",
  "renderer": "react",
  "layout":   { "id": "TenantShellLayout", "slots": { "header": "persistent", "main": "transient" } },
  "props": {
    "resident": {
      "$kind": "resource", "type": "Resident", "id": { "value": "5f8e..." },
      "resident": { "firstName": "Ada", "lastName": "Lovelace" },
      "links": {
        "details": { "$kind": "route", "id": "ResidentDetailsRoute", "params": { "id": "5f8e..." } },
        "edit":    { "$kind": "route", "id": "ResidentEditRoute",    "params": { "id": "5f8e..." } }
      },
      "actions": {
        "save":    { "$kind": "action", "id": "residents.save",    "context": { "id": "5f8e..." } },
        "archive": { "$kind": "action", "id": "residents.archive", "context": { "id": "5f8e..." } }
      },
      "permissions": { "canEdit": true, "canArchive": false }
    },
    "sections": {
      "activity": {
        "$kind": "section", "id": "ResidentActivity",
        "params": { "residentId": "5f8e..." },
        "deferred": true
      }
    }
  },
  "assetVersion": "2026.05.22.1",
  "chunks":  ["residents-details.ef561234.js"],
  "preload": ["scheduler.abcd1234.js"],
  "stream":  true,
  "status":  200,
  "meta":    { "tenant": "acme", "locale": "en-US", "trace": "00-..." }
}
```

### 26.3 Generated TypeScript usage (client)

```ts
import { routes, resources, actions, sections } from "@app/generated";

// navigation
await routes.residentDetails.navigate({ id: "5f8e..." });

// in a component, after hydration
const { resident, sections: secs } = useProps<ResidentDetailsProps>();

resident.links.edit.navigate();
await resident.actions.save.execute({ id: resident.id.value, firstName: "Augusta", lastName: "Lovelace" });
secs.activity.reload();
```

That is the entire trace: one source of truth in C#, one wire envelope, one ergonomic TS surface. No bridges hand-written.

---

## 27. Type generation rules — canonical mapping

This table is the contract between the Roslyn generator and the TS runtime. Each row is normative.

| C# construct | Generated TS |
|--------------|--------------|
| `AppRoute<TParams, TProps>` | `routes.<camel(name)>` with `navigate/href/prefetch`; `TParams` & `TProps` as interfaces. |
| `AppAction<TReq, TResp>` with id `x.y` | `actions.x.y` with `execute/optimistic`; `TReq` & `TResp` as interfaces. |
| `AppResource<TIdentity>` | `<Name>Resource` interface; nested `Identity`, `Links`, `Actions`, `Permissions` become typed sub-objects with live handles. |
| `AppSection<TParams, TProps>` | `sections.<camel(name)>` with `reload/invalidate/subscribe`. |
| `RouteRef<TParams>` | `RouteHandle<TParams>` = `{ navigate, href, prefetch }`. |
| `ActionRef<TAction>` | `ActionHandle<TReq, TResp>` = `{ execute, optimistic }`. |
| `SectionRef<TProps>` | `SectionHandle<TProps>` = `{ props, reload, invalidate, subscribe }`. |
| `ResourceRef<TResource>` | Lazy resource proxy. |
| `IslandRef` | Entry in `islands` manifest. |
| `record` | `interface`. |
| `required` member | non-optional TS field. |
| Nullable reference / `Nullable<T>` | optional / `T \| null` (canonical: `?` for query, `\| null` for body). |
| `Guid`, `DateOnly`, `DateTimeOffset`, `TimeSpan`, `decimal`, `long` | `string` (with branded types where applicable). |
| `enum` | string literal union. |
| `[Capability]` on a `bool` member | named permission flag. |
| `AppValidator<T>` | entry in `validators` metadata tree (§13). |
| `IAppLocalizationKeys` | nested `keys` tree (§22). |

### Determinism
- Same source → byte-identical generated TS.
- Ordering: alphabetical by id within each map.
- Diff-stability: adding a route MUST NOT reorder existing entries.

---

## 28. Constraints — what is illegal to author

These are generator-enforced; violations fail the build with a specific diagnostic code.

| Code | Constraint |
|------|------------|
| `ALIS0001` | Route Props is degenerate (data only, no capability surface). |
| `ALIS0002` | URL string literal outside a `[Route]` attribute. |
| `ALIS0003` | Action without `[Action(id)]` attribute and without convention-derivable id. |
| `ALIS0004` | Resource without `Identity` nested record implementing `IResourceIdentity`. |
| `ALIS0005` | `[FromBody]` on a Params record. |
| `ALIS0006` | Validator referencing a request type that is not nested in its Action. |
| `ALIS0007` | Island missing `[Renderer]` when more than one renderer is registered. |
| `ALIS0008` | Section's Params not derivable from parent page's Params and Resource ids. |
| `ALIS0009` | Action returning a raw HTTP response from a handler. |
| `ALIS0010` | Page constructing a `RouteRef`/`ActionRef`/`SectionRef` outside `ctx.Refs.*` / `ctx.Sections.*`. |
| `ALIS0011` | Asset version mismatch between embedded graph manifest and Vite chunk manifest. |
| `ALIS0012` | Capability flag referenced in props without a corresponding `[Policy]` declaration. |
| `ALIS0013` | Two routes resolving to the same canonical pattern. |
| `ALIS0014` | Component identity collision after attribute overrides. |
| `ALIS0015` | A `ResourceRef` projected without permissions when the resource declares any. |

Each diagnostic has a runbook at `/docs/spec/runbooks/<code>.md`.

---

## 29. Invariants (the spine that must hold)

1. **The graph is the source of truth.** Every TS construct exists because a C# construct generated it.
2. **Identity threads through all four surfaces.** A name in C# is the same name in the graph, on the wire, and in TS.
3. **Props are capability graphs.** Data without affordances is a smell; the generator flags it.
4. **No string URLs in app code.** Routes are typed end to end.
5. **Validation lives once.** On the server, as `AppValidator<T>`. The client gets metadata.
6. **Permissions are projected, never recomputed.** Server-canonical.
7. **Renderer is metadata.** The protocol doesn't know what React is.
8. **`assetVersion` is pinned per process.** Skew is detected and handled, not avoided by luck.
9. **Sections are the partial unit.** Pages don't reload as a whole unless they truly must.
10. **`App.Render(props)` is the only ergonomic the developer needs.** Everything else is the framework's job.

End of canonical reference.
