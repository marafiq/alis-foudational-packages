# Typed .NET SSR Protocol Framework — Canonical Forms Specification

Status: Draft v0.3
Mode: Canonical reference. Every primitive is specified across four surfaces: (a) the C# authoring surface a developer writes, (b) the source-generator augmentation added to that C# at compile time, (c) the wire envelope on the network, (d) the TypeScript surface the client consumes. When these four agree by construction, the framework is correct. The rest of this document defines that agreement.

---

## 0. Reading model

This document is not a tutorial. It is the contract.

The framework has exactly four expressive surfaces, in this order of authority:

1. **C# authoring surface.** What a developer types. The closed type universe in §1.
2. **Compile-time graph.** What the Roslyn generator extracts and embeds as `alis.graph.json`. §5 and §41.
3. **Wire envelope.** What flows between server and client at runtime. §6–§39.
4. **Generated TypeScript surface.** What the client uses. §50.

A name in surface (1) is the same name in (2), (3), and (4). Identity threads through. If a developer renames `SaveResident` to `UpdateResident`, the wire identity changes, the generated TS changes, the graph node id changes, and every reference in props (still typed against `ActionRef<SaveResident>`) fails to compile. There is no soft seam.

Where the spec uses MUST, MUST NOT, SHOULD, MAY: RFC 2119 meanings.

Where a C# block appears: it is the canonical authoring form. Identifier names are normative; the source generator depends on them.

Where a TS block appears: it is the generated output. Hand-written equivalents are forbidden.

Where a JSON block appears: it is the wire shape. Field names, casing, and the `$kind` discriminator are normative.

---

# PART I — The Typed Layer in C#

The typed layer is the entire authoring surface. Section 1 specifies it in full.

## 1. The closed type universe

The framework defines a **closed** set of base classes, marker interfaces, typed reference primitives, attributes, and context objects. Application code derives from these and uses them. It introduces no new top-level framework abstractions; if a need cannot be expressed in this set, it is either out of scope or it requires a framework change with its own ADR.

### 1.1 Base classes — full surface

Every base class lives in `Alis.Ssr.Abstractions`. All members are declared `abstract` or `virtual` deliberately; the documented surface here is normative.

```csharp
// ----- AppRoute<TParams, TProps> ------------------------------------------------

public abstract class AppRoute<TParams, TProps> : IAppGraphNode
    where TParams : class, new()
    where TProps  : class
{
    public string Id              => GetType().FullName!;
    public abstract string Pattern { get; }                  // ASP.NET route template
    public virtual  string? Name              => null;       // TS key override
    public virtual  string? Renderer          => null;       // pin renderer
    public virtual  Type?   LayoutType        => null;       // pin layout
    public virtual  Type?   FallbackType      => null;       // SSR fallback component
    public virtual  string? Policy            => null;       // attach policy by name
    public virtual  CachePolicy Cache         => CachePolicy.None;
    public virtual  PrefetchPolicy Prefetch   => PrefetchPolicy.OnViewport;
    public virtual  IReadOnlyList<string> Vary => Array.Empty<string>();
    public virtual  ChunkHint? Chunk          => null;       // hint for bundler

    // Hook points (rarely overridden; defaults handle the common case)
    public virtual Task<TParams> BindParams(HttpRequest req, IAppParamBinder binder)
        => binder.Bind<TParams>(req);
    public virtual Task<RouteAuthorization> Authorize(TParams p, AuthContext ctx)
        => Task.FromResult(RouteAuthorization.Allow);
}

// ----- AppPage<TParams, TProps> -------------------------------------------------

public abstract class AppPage<TParams, TProps> : IAppPage
    where TParams : class, new()
    where TProps  : class
{
    public abstract Task<AppView<TProps>> Render(TParams p, AppPageContext ctx);

    public virtual Task<AppView<object>> RenderPartial(
        TParams p, string sectionId, AppPageContext ctx)
        => ctx.Sections.RenderById(sectionId, p);

    public virtual Task<AppView<TProps>> RenderFallback(
        TParams p, AppError error, AppPageContext ctx)
        => Task.FromResult(App.Render<TProps>(default!).Status(error.HttpStatus));
}

// ----- AppLayout<TLayoutProps> --------------------------------------------------

public abstract class AppLayout<TLayoutProps> : IAppLayout
    where TLayoutProps : class
{
    public abstract LayoutSlots DeclareSlots();
    public abstract Task<TLayoutProps> Project(AppLayoutContext ctx);
    public virtual  string? Renderer => null;
}

// ----- AppSection<TParams, TProps> ----------------------------------------------

public abstract class AppSection<TParams, TProps> : IAppGraphNode
    where TParams : class, new()
    where TProps  : class
{
    public string Id => GetType().FullName!;
    public virtual  string? Name      => null;
    public virtual  string? Renderer  => null;
    public virtual  string? Policy    => null;
    public virtual  CachePolicy Cache => CachePolicy.None;
}

public abstract class AppSectionHandler<TParams, TProps> : IAppSectionHandler
    where TParams : class, new()
    where TProps  : class
{
    public abstract Task<TProps> Render(TParams p, AppSectionContext ctx);
}

// ----- AppAction<TRequest, TResponse> -------------------------------------------

public abstract class AppAction<TRequest, TResponse> : IAppGraphNode
    where TRequest  : class, new()
    where TResponse : class
{
    public string Id => GetType().FullName!;     // overridden by [Action(id)]
    public virtual string? Policy             => null;
    public virtual IdempotencyMode Idempotency => IdempotencyMode.None;
    public virtual ActionTransport Transport   => ActionTransport.Json;
    public virtual TimeSpan        Timeout     => TimeSpan.FromSeconds(30);
    public virtual int             MaxRetries  => 0;
}

public abstract class AppActionHandler<TRequest, TResponse> : IAppActionHandler
    where TRequest  : class, new()
    where TResponse : class
{
    public abstract Task<AppActionResult<TResponse>> Handle(
        TRequest req, AppActionContext ctx);
}

// ----- AppValidator<TRequest> ---------------------------------------------------

public abstract class AppValidator<TRequest> : IAppValidator
    where TRequest : class
{
    protected RuleBuilder<TRequest, TField> RuleFor<TField>(
        Expression<Func<TRequest, TField>> selector) => ...;
    protected void When(Expression<Predicate<TRequest>> when, Action body) => ...;
    protected void Unless(Expression<Predicate<TRequest>> unless, Action body) => ...;
    protected void RuleSet(string name, Action body) => ...;
}

// ----- AppOptimism<TRequest, TTarget> -------------------------------------------

public abstract class AppOptimism<TRequest, TTarget> : IAppOptimism
    where TRequest : class
    where TTarget  : class
{
    public abstract TTarget Project(TTarget current, TRequest request);
    public virtual  TTarget Rollback(TTarget current, TRequest request) => current;
}

// ----- AppResource<TIdentity> ---------------------------------------------------

public abstract record AppResource<TIdentity> : IAppResource
    where TIdentity : IResourceIdentity
{
    public abstract TIdentity Id { get; init; }
}

// ----- AppIsland<TProps> --------------------------------------------------------

public abstract class AppIsland<TProps> : IAppIsland
    where TProps : class
{
    public virtual IslandManifest Manifest => new();
}

// ----- AppLocalization<TKeys> ---------------------------------------------------

public abstract class AppLocalization<TKeys> : IAppLocalization
    where TKeys : IAppLocalizationKeys
{
    public abstract Task<IReadOnlyDictionary<string,string>> Resolve(
        IReadOnlyCollection<LocalizationKey> keys,
        AppLocalizationContext ctx);
}
```

Every public member shown above is part of the v1 ABI. Adding members is non-breaking; removing or renaming is.

### 1.2 Marker interfaces

Marker interfaces are how the generator distinguishes structural roles. They have no methods; their presence is the contract.

```csharp
public interface IAppGraphNode    { string Id { get; } }
public interface IAppResource     { }
public interface IAppPage         { }
public interface IAppLayout       { }
public interface IAppIsland       { }
public interface IResourceIdentity{ }
public interface IAppLinks        { }       // record marked as link bundle
public interface IAppActions      { }       // record marked as action bundle
public interface IAppSections     { }       // record marked as section bundle
public interface IAppPermissions  { }       // record marked as capability bundle
public interface IAppPolicy       { Task<AuthDecision> Authorize(AuthContext ctx); string Name { get; } }
public interface IAppLocalizationKeys { }   // static class of LocalizationKey constants
public interface IInvalidationRef { }       // marker for typed refs that can invalidate caches
```

A nested record that implements one of `IAppLinks`/`IAppActions`/`IAppSections`/`IAppPermissions` MUST contain **only** typed refs of the matching kind. Mixing kinds is `ALIS0020`.

### 1.3 Typed reference primitives

These are the only types that may cross the wire as live affordances. They are deliberately `readonly record struct` for value semantics.

```csharp
public readonly record struct RouteRef<TParams>(string RouteId, TParams Params)
    : IInvalidationRef where TParams : class;

public readonly record struct ActionRef<TAction>(string ActionId, object? Context)
    where TAction : IAppGraphNode;

public readonly record struct SectionRef<TProps>(
    string SectionId, object Params, TProps? Inline = null, string? Etag = null)
    : IInvalidationRef where TProps : class;

public readonly record struct ResourceRef<TResource>(
    string ResourceTypeId, IResourceIdentity Id)
    : IInvalidationRef where TResource : IAppResource;

public readonly record struct IslandRef(
    string IslandId, string Renderer, ChunkRef Chunk, object Props);

public readonly record struct ChunkRef(string ChunkId);

public readonly record struct MediaRef(
    string Id, string Url, IReadOnlyList<MediaVariant>? Variants = null);

public readonly record struct OperationRef(
    string OperationId, RouteRef<OperationStatusParams> Status, ActionRef<CancelOperation>? Cancel);

public readonly record struct FormRef<TRequest>(
    ActionRef<IAppGraphNode> Submit,
    TRequest InitialValues,
    string ValidatorId,
    IReadOnlyList<string>? RuleSets = null) where TRequest : class;
```

Application code MUST NOT construct these directly. The framework exposes typed factories on context objects (`ctx.Refs.Route<T>(...)`, `ctx.Refs.Action<T>(...)`, `ctx.Sections.Ref<T>(...)`, `ctx.Resources.RefTo<T>(...)`, `ctx.Forms.For<T>(...)`). Bypassing the factories is `ALIS0010`.

### 1.4 Identity types and branded values

Resource identity is **always** a typed record; it is never a raw `Guid` or `string` on the wire.

```csharp
public sealed record ResidentId(Guid Value) : IResourceIdentity;
public sealed record TenantId  (string Value) : IResourceIdentity;
public sealed record CompositeKey(string Scope, Guid Item) : IResourceIdentity;
```

The generator emits branded types in TS:

```ts
type ResidentId    = string & { readonly __brand: "ResidentId" };
type TenantId      = string & { readonly __brand: "TenantId" };
type CompositeKey  = { scope: string; item: string } & { readonly __brand: "CompositeKey" };
```

Branded types prevent accidental cross-resource id mixing in TS without runtime cost. A function that requires `ResidentId` cannot be called with a `TenantId` or a bare `string`. The branding rule for the generator is in §45.

### 1.5 Attributes — full surface

Attributes carry generator metadata. They MAY override convention; they MUST NOT introduce behavior the generator can't see.

```csharp
[AttributeUsage(AttributeTargets.Class)]                  public sealed class RouteAttribute(string pattern, string? name = null) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class ActionAttribute(string id) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class ResourceAttribute(string typeId) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class SectionAttribute(string name) : Attribute;
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Property)] public sealed class PolicyAttribute(string name) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class RendererAttribute(string id) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class ChunkAttribute(string hint) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class IslandAttribute(string id) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class LayoutAttribute(Type type) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class ComponentIdentityAttribute(string id) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class IdempotentAttribute(IdempotencyMode mode) : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class OptimisticAttribute : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class MultipartRequestAttribute : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class LongRunningAttribute : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class PolymorphicResponseAttribute : Attribute;
[AttributeUsage(AttributeTargets.Class)]                  public sealed class AllowDtoPropsAttribute : Attribute;

[AttributeUsage(AttributeTargets.Property)] public sealed class CapabilityAttribute(string name) : Attribute;
[AttributeUsage(AttributeTargets.Property)] public sealed class FromRouteAttribute : Attribute;
[AttributeUsage(AttributeTargets.Property)] public sealed class FromQueryAttribute(string? name = null) : Attribute;
[AttributeUsage(AttributeTargets.Property)] public sealed class FromHeaderAttribute(string name) : Attribute;
[AttributeUsage(AttributeTargets.Property)] public sealed class FromTenantAttribute : Attribute;
[AttributeUsage(AttributeTargets.Property)] public sealed class FromUserAttribute(string claim) : Attribute;
```

### 1.6 Context objects — full surface

A context object is the framework's handshake with handler code. It is the only way to obtain typed refs, project resources, evaluate policies, schedule sections, emit telemetry, or talk to DI. Handler code that captures state outside the context — static fields, ambient HTTP context, manual `IHttpContextAccessor` — is incorrect.

#### 1.6.1 `AppPageContext`

```csharp
public sealed class AppPageContext
{
    public IServiceProvider          Services        { get; }
    public ClaimsPrincipal           User            { get; }
    public ITenantInfo               Tenant          { get; }
    public string                    Locale          { get; }
    public CancellationToken         Cancellation    { get; }

    public NavigationStrategy        Strategy        { get; }     // full | partial | stream
    public string?                   RequestedSection{ get; }     // when partial
    public string                    AssetVersion    { get; }
    public string                    TraceId         { get; }

    public IRefFactory               Refs            { get; }     // §1.6.5
    public ISectionFactory           Sections        { get; }
    public IResourceFactory          Resources       { get; }
    public ICapabilityFactory        Capabilities    { get; }
    public ILinkFactory              Links           { get; }
    public IFormFactory              Forms           { get; }
    public IPageMetaFactory          Meta            { get; }
    public IIslandFactory            Islands         { get; }
    public IPolicyEvaluator          Policies        { get; }
    public IObservability            Observability   { get; }
    public IInvalidationBuilder      Invalidate      { get; }
    public IRedirectFactory          Redirect        { get; }
    public IClock                    Clock           { get; }
    public IRandom                   Random          { get; }     // deterministic for tests
}
```

#### 1.6.2 `AppActionContext`

```csharp
public sealed class AppActionContext
{
    public IServiceProvider          Services        { get; }
    public ClaimsPrincipal           User            { get; }
    public ITenantInfo               Tenant          { get; }
    public string                    Locale          { get; }
    public CancellationToken         Cancellation    { get; }

    public string                    ActionId        { get; }
    public string?                   IdempotencyKey  { get; }
    public string?                   RouteName       { get; }     // calling route
    public string?                   SectionId       { get; }     // calling section, if any
    public string                    TraceId         { get; }

    public IRefFactory               Refs            { get; }
    public IResourceFactory          Resources       { get; }
    public ICapabilityFactory        Capabilities    { get; }
    public IInvalidationBuilder      Invalidate      { get; }     // accumulates refs to return
    public IRedirectFactory          Redirect        { get; }
    public IObservability            Observability   { get; }
    public IOperationFactory         Operations      { get; }     // for [LongRunning]
    public IClock                    Clock           { get; }

    // Idempotency helpers
    public Task<AppActionResult<TResponse>?> TryReplay<TResponse>() where TResponse : class;
    public Task                            Record<TResponse>(AppActionResult<TResponse> result) where TResponse : class;
}
```

#### 1.6.3 `AppSectionContext`

```csharp
public sealed class AppSectionContext
{
    public IServiceProvider          Services        { get; }
    public ClaimsPrincipal           User            { get; }
    public ITenantInfo               Tenant          { get; }
    public string                    Locale          { get; }
    public CancellationToken         Cancellation    { get; }

    public string                    SectionId       { get; }
    public NavigationStrategy        Strategy        { get; }     // partial when invoked alone
    public string                    AssetVersion    { get; }
    public string                    TraceId         { get; }

    public IRefFactory               Refs            { get; }
    public IResourceFactory          Resources       { get; }
    public ICapabilityFactory        Capabilities    { get; }
    public IObservability            Observability   { get; }
    public IClock                    Clock           { get; }
}
```

#### 1.6.4 `AppLayoutContext`

```csharp
public sealed class AppLayoutContext
{
    public IServiceProvider          Services        { get; }
    public ClaimsPrincipal           User            { get; }
    public ITenantInfo               Tenant          { get; }
    public string                    Locale          { get; }
    public CancellationToken         Cancellation    { get; }
    public IRefFactory               Refs            { get; }
    public IResourceFactory          Resources       { get; }
}
```

#### 1.6.5 Factories on contexts

```csharp
public interface IRefFactory
{
    RouteRef<TParams>     Route<TRoute>(TParams p)   where TRoute  : AppRoute<TParams,_>;
    ActionRef<TAction>    Action<TAction>(object? c = null) where TAction : IAppGraphNode;
    ChunkRef              Chunk(string id);
    MediaRef              Media(string id);
}

public interface ISectionFactory
{
    SectionRef<TProps>    Ref<TSection>(TParams p)
        where TSection : AppSection<TParams, TProps>
        where TParams  : class, new()
        where TProps   : class;

    Task<TProps>          Render<TSection>(TParams p);
    Task<AppView<object>> RenderById(string sectionId, object parentParams);
}

public interface IResourceFactory
{
    Task<TResource>       Project<TResource>(IResourceIdentity id)
        where TResource : IAppResource;
    ResourceRef<TResource> RefTo<TResource>(IResourceIdentity id)
        where TResource : IAppResource;
    Task<IReadOnlyList<TResource>> ProjectMany<TResource>(IEnumerable<IResourceIdentity> ids)
        where TResource : IAppResource;
}

public interface ICapabilityFactory
{
    TPermissions          Project<TPermissions>(object subject) where TPermissions : IAppPermissions;
    Task<bool>            Evaluate(string policyName, object? subject = null);
}

public interface ILinkFactory
{
    TLinks                For<TLinks>(object scope) where TLinks : IAppLinks;
}

public interface IFormFactory
{
    FormRef<TRequest>     For<TRequest>() where TRequest : class, new();
    FormRef<TRequest>     For<TRequest>(TRequest initial) where TRequest : class;
}

public interface IInvalidationBuilder
{
    IInvalidationBuilder  Section<TSection>(object @params) where TSection : IAppGraphNode;
    IInvalidationBuilder  Resource<TResource>(IResourceIdentity id) where TResource : IAppResource;
    IInvalidationBuilder  Route<TRoute>(object @params) where TRoute : IAppGraphNode;
    IReadOnlyList<IInvalidationRef> Build();
}

public interface IRedirectFactory
{
    AppView<object>       To<TRoute>(object @params) where TRoute : IAppGraphNode;
    AppView<object>       External(string url);
}

public interface IOperationFactory
{
    Task<OperationRef>    Start<TOp>(object input) where TOp : IAppGraphNode;
    Task                  Progress(string opId, double percent, string? message = null);
    Task                  Complete<TResponse>(string opId, TResponse value);
    Task                  Fail(string opId, AppError error);
}

public interface IPageMetaFactory
{
    PageMeta              Default(string? title = null);
    PageMetaBuilder       Build();
}
```

### 1.7 The static `App` surface

Exactly one entry point. App code uses `App`; the framework uses internal builders.

```csharp
public static class App
{
    public static AppView<TProps>  Render<TProps>(TProps props)        where TProps : class;
    public static AppView<TProps>  RenderPartial<TProps>(TProps props) where TProps : class;
    public static AppView<TProps>  RenderStream<TProps>(Action<IStreamBuilder<TProps>> build) where TProps : class;

    public static AppActionResult<TResponse> Ok<TResponse>(TResponse value) where TResponse : class;
    public static AppActionResult<TResponse> Ok<TResponse>(TResponse value, params IInvalidationRef[] invalidations) where TResponse : class;
    public static AppActionResult<TResponse> Fail<TResponse>(ErrorCode code, string? message = null, FieldErrors? fields = null, TimeSpan? retryAfter = null) where TResponse : class;
    public static AppActionResult<TResponse> RedirectTo<TResponse, TRoute>(object @params) where TResponse : class where TRoute : IAppGraphNode;

    public static AppView<object>  Redirect<TRoute>(object @params)    where TRoute : IAppGraphNode;
    public static AppView<object>  Redirect(string externalUrl);

    public static AppView<TProps>  Partial<TSection, TProps>(object @params) where TProps : class;

    public static AppStream        Stream();
}
```

### 1.8 The `AppView<TProps>` builder — full surface

`AppView<TProps>` is the immutable description of a render. Every method returns a new view; the original is unchanged.

```csharp
public sealed class AppView<TProps> where TProps : class
{
    public TProps                  Props        { get; }
    public string?                 LayoutId     { get; }
    public string?                 RendererId   { get; }
    public int                     Status       { get; }
    public CachePolicy             CachePolicy  { get; }
    public IReadOnlyList<string>   StreamPaths  { get; }
    public IReadOnlyList<string>   DeferPaths   { get; }
    public IReadOnlyList<IslandRef> Islands     { get; }
    public IReadOnlyList<ChunkRef> Preload      { get; }
    public IReadOnlyList<KeyValuePair<string,string>> Headers { get; }
    public IReadOnlyList<HttpCookie> Cookies    { get; }
    public PageMeta?               Meta         { get; }
    public IReadOnlyList<IInvalidationRef> Invalidations { get; }

    public AppView<TProps> WithLayout<TLayout>() where TLayout : IAppLayout;
    public AppView<TProps> WithRenderer(string renderer);
    public AppView<TProps> WithIsland<TIsland>(string slot, object props) where TIsland : IAppIsland;
    public AppView<TProps> Stream(Expression<Func<TProps, SectionRef>> selector);
    public AppView<TProps> Stream<TSel>(Expression<Func<TProps, SectionRef<TSel>>> selector) where TSel : class;
    public AppView<TProps> Defer(Expression<Func<TProps, SectionRef>> selector);
    public AppView<TProps> Cache(TimeSpan ttl);
    public AppView<TProps> Cache(CachePolicy policy);
    public AppView<TProps> Vary(params string[] keys);
    public AppView<TProps> Header(string name, string value);
    public AppView<TProps> Status(int code);
    public AppView<TProps> Cookie(string name, string value, CookieOptions? options = null);
    public AppView<TProps> Meta(Action<PageMetaBuilder> build);
    public AppView<TProps> Title(string title);
    public AppView<TProps> Title(LocalizationKey key, object? args = null);
    public AppView<TProps> Preload(params ChunkRef[] chunks);
    public AppView<TProps> Invalidate(params IInvalidationRef[] refs);
    public AppView<TProps> Trace(string name, object? attrs = null);
}
```

`AppView<object>` is the type returned for redirects and certain framework-internal views; it carries `Status` and `Headers` but no `Props`.

### 1.9 `AppActionResult<TResponse>` hierarchy

```csharp
public abstract record AppActionResult<TResponse> where TResponse : class
{
    public sealed record Ok(
        TResponse Value,
        IReadOnlyList<IInvalidationRef> Invalidations,
        AppRedirect? Redirect = null) : AppActionResult<TResponse>;

    public sealed record Fail(AppError Error) : AppActionResult<TResponse>;
}

public sealed record AppRedirect(string RouteId, object Params);
```

There is no exception path for expected outcomes. A handler that throws an unhandled exception is treated as `Fail(AppError.Internal)` and emits a `traceId` for diagnosis.

### 1.10 `AppStream` builder

```csharp
public sealed class AppStream
{
    public AppStream Shell<TProps>(TProps props) where TProps : class;
    public AppStream Stream<TProps>(string section, Func<Task<TProps>> producer);
    public AppStream Stream<TProps>(string section, Func<IAsyncEnumerable<TProps>> producer); // progressive
    public AppStream Defer<TProps>(string section, Func<Task<TProps>> producer);
    public AppStream Timeout(TimeSpan total, TimeSpan? perSection = null);
    public AppStream OnError(StreamErrorPolicy policy);
}
```

### 1.11 `AppError` type

```csharp
public sealed record AppError(
    ErrorCode               Code,
    string?                 Message       = null,
    FieldErrors?            Fields        = null,
    TimeSpan?               RetryAfter    = null,
    string?                 TraceId       = null,
    IReadOnlyDictionary<string,string>? Meta = null)
{
    public int HttpStatus => Code switch
    {
        ErrorCode.ValidationFailed       => 422,
        ErrorCode.NotAuthenticated       => 401,
        ErrorCode.NotAuthorized          => 403,
        ErrorCode.NotFound               => 404,
        ErrorCode.Conflict               => 409,
        ErrorCode.AssetVersionMismatch   => 409,
        ErrorCode.RateLimited            => 429,
        ErrorCode.PreconditionFailed     => 412,
        ErrorCode.UnprocessableEntity    => 422,
        ErrorCode.Internal               => 500,
        ErrorCode.Unavailable            => 503,
        _                                => 500,
    };
}

public sealed record FieldErrors(IReadOnlyDictionary<string, IReadOnlyList<string>> Map);

public enum ErrorCode
{
    ValidationFailed, NotAuthenticated, NotAuthorized, NotFound,
    Conflict, AssetVersionMismatch, RateLimited, PreconditionFailed,
    UnprocessableEntity, Internal, Unavailable
}
```

### 1.12 Generator-added partial augmentation

The Roslyn generator augments user `partial` classes. The augmentation is purely additive: it never changes existing members. The augmented members are normative — generator output across implementations MUST agree on them.

For an `AppRoute<,>` declared as `partial`:

```csharp
// User wrote:
[Route("/residents/{id:guid}")]
public sealed partial class ResidentDetailsRoute
    : AppRoute<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props> { ... }

// Generator emits (compile-time):
public sealed partial class ResidentDetailsRoute
{
    public const string                TypedRouteId   = "App.Features.Residents.ResidentDetailsRoute";
    public const string                TypedRouteName = "residentDetails";
    public const string                TypedPattern   = "/residents/{id:guid}";
    public static RouteRef<Params>     Ref(Params p) => new(TypedRouteId, p);
    public static RouteRef<Params>     Ref(Guid id, string? tab = null) => new(TypedRouteId, new Params { Id = id, Tab = tab });
}
```

For an `AppAction<,>` declared as `partial`:

```csharp
public sealed partial class SaveResident
{
    public const string                 TypedActionId = "residents.save";
    public static ActionRef<SaveResident> Ref(Request? ctx = null) => new(TypedActionId, ctx);
}
```

For an `AppSection<,>` declared as `partial`:

```csharp
public sealed partial class ResidentActivitySection
{
    public const string TypedSectionId = "ResidentActivity";
    public static SectionRef<Props> Ref(Params p) => new(TypedSectionId, p);
}
```

For an `AppResource<>` declared as `partial`:

```csharp
public sealed partial record ResidentResource
{
    public const string                       TypedResourceId = "Resident";
    public static ResourceRef<ResidentResource> RefTo(Identity id) => new(TypedResourceId, id);
}
```

Application code MAY call these augmented members. The framework MUST tolerate authors who omit `partial`, in which case the augmentation is emitted under a sibling `ResidentDetailsRoute_Typed` static class with identical members.

---

## 2. Generic constraints

The closed type universe relies on these constraints. Implementations MUST enforce them via Roslyn diagnostics, not at runtime.

| Constraint | Required by |
|------------|-------------|
| `where TParams : class, new()` | `AppRoute`, `AppSection`, `AppPage`, `AppSectionHandler` |
| `where TProps  : class` | as above, plus `AppLayout`, `AppIsland`, `AppView` |
| `where TRequest : class, new()` | `AppAction`, `AppActionHandler`, `AppValidator`, `AppOptimism` |
| `where TResponse : class` | `AppAction`, `AppActionHandler`, `AppActionResult` |
| `where TIdentity : IResourceIdentity` | `AppResource` |
| `where TResource : IAppResource` | `ResourceRef<>`, factories |
| `where TLayout : IAppLayout` | `AppView.WithLayout<T>` |
| `where TIsland : IAppIsland` | `AppView.WithIsland<T>` |
| `where TRoute : IAppGraphNode` | `App.Redirect<>`, navigation refs |
| `where TAction : IAppGraphNode` | `ActionRef<>` |
| `where TLinks : IAppLinks` / `TActions : IAppActions` / `TSections : IAppSections` / `TPermissions : IAppPermissions` | resource bundles |

A type parameter declared in user code MUST satisfy these or fail with the appropriate `ALIS00xx` diagnostic.

---

## 3. Naming, file, and assembly conventions

These conventions are load-bearing. The generator derives identity from them when no attribute overrides.

### 3.1 Type names

| Concept | Convention | Example |
|---------|-----------|---------|
| Route | `<Resource><Purpose>Route` | `ResidentDetailsRoute` |
| Page | `<Resource><Purpose>Page` | `ResidentDetailsPage` |
| Action | `<Verb><Resource>` | `SaveResident`, `ArchiveResident` |
| Resource | `<Singular>Resource` | `ResidentResource` |
| Section | `<Resource><Name>Section` | `ResidentActivitySection` |
| Layout | `<Name>Layout` | `TenantShellLayout` |
| Island | `<Name>Island` or `<Name>` with `[Island]` | `ResidentSchedulerIsland` |
| Policy | `<Name>Policy` | `EditResidentPolicy` |
| Localization keys | `<Resource>Keys` | `ResidentKeys` |

### 3.2 Nested record names

A class containing the framework primitives MUST use these nested-record names:

- `Params`, `Props`, `Request`, `Response`, `Validator`, `Handler`, `Optimism`,
  `Identity`, `Data`, `Links`, `Actions`, `Sections`, `Permissions`.

The generator infers ownership from nesting. A `Validator` nested inside a non-Action class is `ALIS0006`.

### 3.3 File layout

One Resource per folder. Folders are the unit of feature ownership.

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
    ResidentSchedulerIsland.cs
  Policies/
    EditResidentPolicy.cs
  Localization/
    ResidentKeys.cs
  Validators/        (optional; nested Validator preferred)
```

### 3.4 Assembly conventions

- The application assembly MUST carry `[assembly: AlisAssemblyMarker]` to be discovered.
- Test assemblies (`*.Tests`) are excluded from the graph regardless of contents.
- Generator output is written to `obj/.alis/` (assembly resources) and `client/.generated/` (TS).

---

## 4. Composition root and DI

The framework attaches via exactly one builder call. There is no per-route registration.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAlisSsr(opt =>
{
    opt.AssetVersion = AppBuild.AssetVersion;
    opt.ProcessId    = Environment.MachineName + ":" + AppBuild.Commit;

    opt.AddAssembly<AppMarker>();                             // discovery root(s)

    opt.AddRenderer<ReactRenderer>("react").Default();
    opt.AddRenderer<VueRenderer>("vue");
    opt.AddRenderer<SolidRenderer>("solid");
    opt.AddRenderer<PreactRenderer>("preact");

    opt.AddLayout<TenantShellLayout>().Default();
    opt.AddLayout<PublicLayout>();

    opt.AddLocalization<AppLocalization>();
    opt.AddPolicy<EditResidentPolicy>();
    opt.AddPolicy<ArchiveResidentPolicy>();

    opt.AddTenants(t =>
    {
        t.ResolveFromSubdomain();
        t.AddOverride("acme",   o => o.AssetVersion = "2026.05.22.1-acme");
    });

    opt.AddObservability(o =>
    {
        o.UseOpenTelemetry();
        o.WithClientTelemetry(transport: ClientTelemetryTransport.OtlpHttp);
    });

    opt.AddStreaming(s => s.MaxConcurrentSections = 16);
    opt.AddCache(c => c.MaxBytes = 256 * 1024 * 1024);

    opt.AddInvalidationBus(i => i.UseInProcess()); // or .UseSse() / .UseRedis()
});

var app = builder.Build();
app.UseAlisSsr();
app.Run();
```

`AddAlisSsr` is the only sanctioned hosting entry. Lower-level extension points exist but are not in the public surface.

The framework auto-registers, by reflection over discovered assemblies:

- every `AppRoute<,>` as singleton,
- every `AppPage<,>` as scoped,
- every `AppAction<,>` as singleton (metadata) plus its handler as scoped,
- every `AppSection<,>` as singleton plus its handler as scoped,
- every `AppValidator<>` as singleton,
- every `AppOptimism<,>` as singleton,
- every `AppLayout<>` as singleton,
- every `AppIsland<>` as singleton,
- every `IAppPolicy` as singleton.

User-supplied `services.Add*<T>()` calls MAY replace a default registration; they MUST NOT collide on the same identity.

---

## 5. Source generator outputs — full inventory

The generator runs as a Roslyn incremental generator and as a build task. For each successful compilation, it produces the artifacts in this table. The names are normative; downstream tools depend on them.

| Artifact | Path | Purpose |
|----------|------|---------|
| Graph manifest | `obj/.alis/alis.graph.json` (and assembly resource `Alis.Graph`) | The compile-time graph; consumed at startup. |
| Routes TS | `client/.generated/routes.ts` | Typed navigation API. |
| Actions TS | `client/.generated/actions.ts` | Typed action API. |
| Resources TS | `client/.generated/resources.ts` | Resource interfaces and proxy types. |
| Sections TS | `client/.generated/sections.ts` | Section refs and handles. |
| Forms TS | `client/.generated/forms.ts` | Form refs and binding helpers. |
| Validators TS | `client/.generated/validators.ts` | Validation metadata trees. |
| Models TS | `client/.generated/models.ts` | All `Params`/`Props`/`Request`/`Response`/`Data` interfaces. |
| Identity TS | `client/.generated/identity.ts` | Branded resource identity types. |
| Localization TS | `client/.generated/keys.ts` | Static key tree. |
| Islands manifest | `client/.generated/islands.ts` | Renderer + chunk per island. |
| Protocol JSON Schema | `client/.generated/protocol/*.schema.json` | Envelope shapes for runtime validation. |
| C# augmentation | `obj/.alis/Gen.*.cs` | Partial augmentations (§1.12). |
| Build manifest | `obj/.alis/alis.build.json` | Build inputs, hash, asset version pin. |

Determinism: identical source MUST produce byte-identical outputs (timestamps and machine names excluded). Map ordering is alphabetical by id.

---

# PART II — Canonical forms per primitive

For every primitive, the canonical form spans four surfaces (C# author, generator augmentation if any, wire, generated TS). All four are normative; mismatch is a framework bug.

## 6. Route — canonical form

### Purpose
A `Route` declares a URL pattern, a typed `Params`, and a typed `Props`. It is the only place URL syntax appears in app code.

### C# author surface

```csharp
namespace App.Features.Residents;

[Route("/residents/{id:guid}", Name = "residentDetails")]
[Policy("ViewResident")]
public sealed partial class ResidentDetailsRoute
    : AppRoute<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public override CachePolicy Cache => CachePolicy.Private(TimeSpan.FromSeconds(15));
    public override PrefetchPolicy Prefetch => PrefetchPolicy.OnViewport;

    public sealed record Params
    {
        [FromRoute]              public required Guid     Id  { get; init; }
        [FromQuery("tab")]       public          string?  Tab { get; init; }
        [FromHeader("X-Locale")] public          string?  Locale { get; init; }
    }

    public sealed record Props
    {
        public required ResidentResource Resident { get; init; }
        public required Sections         Sections { get; init; }
        public required Permissions      Permissions { get; init; }
        public required Links            Links { get; init; }
        public required PageMeta         Meta { get; init; }
    }

    public sealed record Sections : IAppSections
    {
        public required SectionRef<ResidentActivitySection.Props>  Activity  { get; init; }
        public required SectionRef<ResidentCarePlanSection.Props>  CarePlan  { get; init; }
        public required SectionRef<ResidentAuditSection.Props>     Audit     { get; init; }
    }

    public sealed record Permissions : IAppPermissions
    {
        [Capability("canEditResident")]    public required bool CanEdit    { get; init; }
        [Capability("canArchiveResident")] public required bool CanArchive { get; init; }
    }

    public sealed record Links : IAppLinks
    {
        public required RouteRef<ResidentEditRoute.Params>  Edit { get; init; }
        public required RouteRef<ResidentsListRoute.Params> Back { get; init; }
    }
}
```

### Generator augmentation
See §1.12. Adds `TypedRouteId`, `TypedRouteName`, `TypedPattern`, `Ref(...)` overloads.

### Wire — request

```http
GET /residents/5f8e...?tab=overview HTTP/1.1
X-Alis-Strategy:       full
X-Alis-Asset-Version:  2026.05.22.1
X-Alis-Tenant:         acme
X-Alis-Locale:         en-US
X-Alis-Trace:          00-...-01
Accept:                application/json
```

### Wire — response envelope (full navigation)

```json
{
  "protocol":      { "version": "1" },
  "route":         { "name": "ResidentDetailsRoute", "params": { "id": "5f8e...", "tab": "overview", "locale": null } },
  "component":     "App.Features.Residents.ResidentDetailsPage",
  "renderer":      "react",
  "layout":        { "id": "TenantShellLayout", "slots": { "header": "persistent", "main": "transient" } },
  "props":         { /* §8 */ },
  "assetVersion":  "2026.05.22.1",
  "processId":     "host-01:8f4c",
  "chunks":        ["residents-details.ef561234.js"],
  "preload":       ["scheduler.abcd1234.js"],
  "stream":        false,
  "status":        200,
  "headers":       { "Cache-Control": "private, max-age=15" },
  "meta":          { "trace": "00-...-01", "tenant": "acme", "locale": "en-US",
                     "i18n": { "residents.details.title": "Resident - {name}" } }
}
```

### Generated TypeScript

```ts
// routes.ts (excerpt)
export interface ResidentDetailsParams {
  id:     ResidentId;
  tab?:   string;
  locale?:string;
}

export const routes = {
  residentDetails: {
    id:       "App.Features.Residents.ResidentDetailsRoute" as const,
    pattern:  "/residents/:id",
    cache:    { kind: "private", maxAge: 15 } as const,
    prefetch: "onViewport" as const,

    navigate(params: ResidentDetailsParams, opts?: NavigateOptions): Promise<NavigationResult>;
    href(params: ResidentDetailsParams): string;
    prefetch(params: ResidentDetailsParams): Promise<void>;
    match(url: string): ResidentDetailsParams | null;
  },
  // ...
} as const;
```

### Rules
- Exactly one `[Route]` per `AppRoute<,>`.
- `Params` and `Props` MUST be `sealed record` and nested.
- URL strings appear in user code only inside `[Route(...)]` attributes.
- `cache` and `prefetch` on the route propagate into the generated TS — they are part of the route's contract, not policy applied later.
- Two routes with the same canonical pattern raise `ALIS0013`.

---

## 7. Params — canonical form

### Purpose
Bind URL/query/header inputs into a typed value before the page handler runs.

### Source attributes (full set)

| Attribute | Meaning |
|-----------|---------|
| `[FromRoute]` | route template segment |
| `[FromQuery(name?)]` | query string parameter |
| `[FromHeader(name)]` | request header |
| `[FromTenant]` | resolved tenant id |
| `[FromUser(claim)]` | claim value from `User` |

No `[FromBody]` on Params — that surface belongs to Actions only.

### Type mapping (canonical)

| C# | TS | Wire | Notes |
|----|-----|------|-------|
| `Guid` | branded `string` | string (D-form) | branded if used as Identity field |
| `DateOnly` | `string` | ISO-8601 date | `2026-05-22` |
| `DateTimeOffset` | `string` | ISO-8601 instant | `2026-05-22T00:00:00Z` |
| `TimeSpan` | `string` | ISO-8601 duration | `PT15M` |
| `decimal`, `long` | `string` | string | precision-safe |
| `int`/`short`/`byte`/`double`/`float`/`bool` | native | native | |
| enum | string literal union | string | values are member names |
| `T[]` / `IReadOnlyList<T>` | `T[]` | array | query repetition `?tag=a&tag=b` |
| nullable reference / `T?` | `T \| null` | nullable | |
| nullable in Params | `T?` optional | absent | for query parameters |

Parse failure → `400` with `ErrorCode.UnprocessableEntity` carrying the offending field.

---

## 8. Props — capability graphs (canonical form)

### Purpose
A `Props` record is the page's executable surface: data, sections, permissions, links, actions, page metadata. It is **not a DTO**.

### Required structural shape

A `Props` record SHOULD declare:

- **Resource(s)** — one or more `AppResource` typed values (each carries its own links, actions, permissions).
- **Sections** — a record implementing `IAppSections`, populated with `SectionRef`s.
- **Page-scoped Permissions** — a record implementing `IAppPermissions` (only for capabilities not naturally on a resource).
- **Page-scoped Links** — a record implementing `IAppLinks` (cross-resource navigation).
- **PageMeta** — title, cache hints, localization keys actually used.

### What Props MUST NOT contain
- Raw URL strings (use `RouteRef`).
- Client-only mutation state.
- Methods that aren't Actions.
- Capabilities that duplicate what is already on a contained Resource.

### Lint
A page Props with only `data` and no capability surface raises `ALIS0001`. The escape hatch is `[AllowDtoProps]` on the route, intended only for read-only export endpoints (CSV download, sitemap, etc.).

### Wire — `props` field shape

The wire `props` is a JSON object whose nested values may be **resolvable references** identified by a `$kind` discriminator. The reference-resolution algorithm in §40 walks the tree on the client and replaces references with live handles.

```json
{
  "resident":   { "$kind": "resource", "type": "Resident", "id": { "value": "5f8e..." }, "...": "..." },
  "sections":   { "activity": { "$kind": "section", "id": "ResidentActivity", "params": {...}, "deferred": true } },
  "permissions":{ "canEditResident": true, "canArchiveResident": false },
  "links":      { "edit": { "$kind": "route", "id": "ResidentEditRoute", "params": {...} } },
  "meta":       { "title": "Resident - Ada Lovelace", "cache": { "kind": "private", "maxAge": 15 } }
}
```

---

## 9. Resource — canonical form

### Purpose
A `Resource` is the canonical capability graph of a domain entity: identity + data + links + actions + permissions. Resources are the unit of cross-cutting reuse — they appear in multiple pages with the same shape.

### C# author surface

```csharp
[Resource("Resident")]
public sealed partial record ResidentResource : AppResource<ResidentResource.Identity>
{
    public required Identity      Id          { get; init; }
    public required Data          Resident    { get; init; }
    public required Links         Links       { get; init; }
    public required Actions       Actions     { get; init; }
    public required Permissions   Permissions { get; init; }

    public sealed record Identity(Guid Value) : IResourceIdentity;

    public sealed record Data
    {
        public required string    FirstName  { get; init; }
        public required string    LastName   { get; init; }
        public          DateOnly? Dob        { get; init; }
        public required ResidentStatus Status{ get; init; }
        public          MediaRef? Avatar     { get; init; }
    }

    public sealed record Links : IAppLinks
    {
        public required RouteRef<ResidentDetailsRoute.Params> Details  { get; init; }
        public required RouteRef<ResidentEditRoute.Params>    Edit     { get; init; }
        public required RouteRef<ResidentCarePlanRoute.Params>CarePlan { get; init; }
    }

    public sealed record Actions : IAppActions
    {
        public required ActionRef<SaveResident>      Save     { get; init; }
        public required ActionRef<ArchiveResident>   Archive  { get; init; }
        public required ActionRef<RestoreResident>   Restore  { get; init; }
    }

    public sealed record Permissions : IAppPermissions
    {
        [Capability("canEdit")]    public required bool CanEdit    { get; init; }
        [Capability("canArchive")] public required bool CanArchive { get; init; }
        [Capability("canRestore")] public required bool CanRestore { get; init; }
    }
}

public enum ResidentStatus { Active, Archived }
```

### Resource projection (server)

A resource is constructed exactly once per request, by an `IResourceProjector<TResource>` registered with DI. The page handler obtains it via `ctx.Resources.Project<TResource>(id)`.

```csharp
public sealed class ResidentResourceProjector : IResourceProjector<ResidentResource>
{
    public ResidentResourceProjector(IResidentService residents) { ... }

    public async Task<ResidentResource> Project(IResourceIdentity id, AppResourceContext ctx)
    {
        var ident = (ResidentResource.Identity)id;
        var data  = await residents.Get(ident.Value, ctx.Cancellation);

        return new ResidentResource
        {
            Id          = ident,
            Resident    = new() { FirstName = data.FirstName, LastName = data.LastName, Dob = data.Dob, Status = data.Status, Avatar = ctx.Refs.Media(data.AvatarId) },
            Links       = new() { Details  = ResidentDetailsRoute.Ref(ident.Value),
                                  Edit     = ResidentEditRoute.Ref(ident.Value),
                                  CarePlan = ResidentCarePlanRoute.Ref(ident.Value) },
            Actions     = new() { Save     = SaveResident.Ref(),
                                  Archive  = ArchiveResident.Ref(),
                                  Restore  = RestoreResident.Ref() },
            Permissions = ctx.Capabilities.Project<ResidentResource.Permissions>(data),
        };
    }
}
```

### Wire form

```json
{
  "$kind": "resource",
  "type":  "Resident",
  "id":    { "value": "5f8e..." },
  "resident":    {
    "firstName": "Ada",
    "lastName":  "Lovelace",
    "dob":       "1815-12-10",
    "status":    "Active",
    "avatar":    { "$kind": "media", "id": "avatar/5f8e", "url": "/m/avatar/5f8e.jpg" }
  },
  "links": {
    "details":  { "$kind": "route", "id": "ResidentDetailsRoute", "params": { "id": "5f8e..." } },
    "edit":     { "$kind": "route", "id": "ResidentEditRoute",    "params": { "id": "5f8e..." } },
    "carePlan": { "$kind": "route", "id": "ResidentCarePlanRoute","params": { "id": "5f8e..." } }
  },
  "actions": {
    "save":     { "$kind": "action", "id": "residents.save",    "context": { "id": "5f8e..." } },
    "archive":  { "$kind": "action", "id": "residents.archive", "context": { "id": "5f8e..." } },
    "restore":  { "$kind": "action", "id": "residents.restore", "context": { "id": "5f8e..." } }
  },
  "permissions": { "canEdit": true, "canArchive": false, "canRestore": false }
}
```

### Generated TypeScript

```ts
export interface ResidentResource {
  readonly $kind: "resource";
  readonly type:  "Resident";
  readonly id:    ResidentId;                    // branded
  readonly resident: {
    firstName: string;
    lastName:  string;
    dob?:      string;                            // date-only
    status:    "Active" | "Archived";
    avatar?:   MediaHandle;
  };
  readonly links: {
    details:  RouteHandle<ResidentDetailsParams>;
    edit:     RouteHandle<ResidentEditParams>;
    carePlan: RouteHandle<ResidentCarePlanParams>;
  };
  readonly actions: {
    save:     ActionHandle<SaveResidentRequest,    SaveResidentResponse>;
    archive:  ActionHandle<ArchiveResidentRequest, ArchiveResidentResponse>;
    restore:  ActionHandle<RestoreResidentRequest, RestoreResidentResponse>;
  };
  readonly permissions: {
    canEdit:    boolean;
    canArchive: boolean;
    canRestore: boolean;
  };
}
```

### Rules
- A `Resource` MUST have an `Identity` nested record implementing `IResourceIdentity`.
- A `Resource` MAY appear nested inside other Resources or Props; the wire form is the same and the proxy is the same.
- Permissions are **projected** from server policies (§32), never recomputed on the client.
- When the same Resource appears multiple times in a single envelope (e.g. a list page containing 20 residents), the wire payload MAY be **deduplicated** — see §10.

---

## 10. Collection / List resource — canonical form

### Purpose
A typed list of a Resource type, with pagination, list-scoped actions, and on-wire deduplication of repeated identities.

### C# author surface

```csharp
[Resource("Residents")]
public sealed partial record ResidentsListResource
    : AppResource<ResidentsListResource.Identity>
{
    public required Identity                       Id         { get; init; }
    public required IReadOnlyList<ResidentResource>Items      { get; init; }
    public required PaginationCursor?              Next       { get; init; }
    public required Actions                        Actions    { get; init; }
    public required ListPermissions                Permissions{ get; init; }

    public sealed record Identity : IResourceIdentity
    {
        public required string Scope { get; init; }      // e.g. tenant/site
        public required string Filter{ get; init; }       // canonical filter hash
    }

    public sealed record Actions : IAppActions
    {
        public required ActionRef<CreateResident>    Create   { get; init; }
        public required ActionRef<ExportResidents>   Export   { get; init; }   // [LongRunning]
    }

    public sealed record ListPermissions : IAppPermissions
    {
        [Capability("canCreate")] public required bool CanCreate { get; init; }
        [Capability("canExport")] public required bool CanExport { get; init; }
    }
}

public sealed record PaginationCursor
{
    public required string                                 Token { get; init; }
    public required RouteRef<ResidentsListRoute.Params>    Next  { get; init; }
}
```

### Wire — deduplication

A response containing many copies of the same Resource identity MAY use a `$resources` table at envelope root:

```json
{
  "$resources": {
    "Resident:5f8e...": { "$kind": "resource", "type": "Resident", "id": {...}, "resident": {...}, "links": {...}, "actions": {...}, "permissions": {...} },
    "Resident:a8b2...": { ... }
  },
  "props": {
    "list": {
      "$kind": "resource", "type": "Residents", "id": {...},
      "items": [
        { "$ref": "Resident:5f8e..." },
        { "$ref": "Resident:a8b2..." }
      ],
      "next": { "token": "eyJsdGUi...", "next": { "$kind": "route", "id": "ResidentsListRoute", "params": { "cursor": "eyJsdGUi..." } } },
      "actions": { "create": {...}, "export": {...} },
      "permissions": { "canCreate": true, "canExport": false }
    }
  }
}
```

`$ref` is the dereference sentinel; see §40.

### Generated TypeScript

```ts
export interface ResidentsListResource {
  readonly $kind: "resource";
  readonly type:  "Residents";
  readonly id:    CompositeKey;
  readonly items: ResidentResource[];                // dereferenced lazily
  readonly next?: { token: string; next: RouteHandle<ResidentsListParams> };
  readonly actions:     { create: ActionHandle<CreateResidentRequest, CreateResidentResponse>; export: ActionHandle<ExportResidentsRequest, ExportResidentsResponse> };
  readonly permissions: { canCreate: boolean; canExport: boolean };
}
```

---

## 11. Pagination — canonical form

The framework supports three pagination styles. Each has a canonical type the developer reaches for; they are not mixed.

### 11.1 Cursor (default)

```csharp
public sealed record CursorPage<T>
{
    public required IReadOnlyList<T> Items { get; init; }
    public required string?          Next  { get; init; }   // opaque
    public required RouteRef<object>?NextRoute { get; init; }
}
```

### 11.2 Offset

```csharp
public sealed record OffsetPage<T>
{
    public required IReadOnlyList<T> Items     { get; init; }
    public required int              Page      { get; init; }
    public required int              PageSize  { get; init; }
    public required long             TotalItems{ get; init; }
    public required RouteRef<object>?Previous  { get; init; }
    public required RouteRef<object>?Next      { get; init; }
}
```

### 11.3 Infinite

```csharp
public sealed record InfiniteFeed<T>
{
    public required IReadOnlyList<T>            Items { get; init; }
    public required ActionRef<LoadMoreFeed<T>>  More  { get; init; }
}
```

### Rules
- A Resource exposing pagination MUST pick one style and stick to it.
- The route handler MUST emit `Cache-Control: no-store` on offset pagination beyond page 1 when totals are sensitive to live writes (configurable).

---

## 12. Section — canonical form

### Purpose
A `Section` is a named, independently reloadable, optionally streamable region of a page.

### C# author surface

```csharp
[Section("ResidentActivity")]
[Policy("ViewResident")]
public sealed partial class ResidentActivitySection
    : AppSection<ResidentActivitySection.Params, ResidentActivitySection.Props>
{
    public override CachePolicy Cache => CachePolicy.Private(TimeSpan.FromSeconds(10));

    public sealed record Params
    {
        public required Guid ResidentId { get; init; }
        public          int  Page       { get; init; } = 1;
    }

    public sealed record Props
    {
        public required IReadOnlyList<ActivityItem>             Items { get; init; }
        public required RouteRef<ActivityDetailsRoute.Params>   Open  { get; init; }
        public required PaginationCursor?                       Next  { get; init; }
    }

    public sealed class Handler : AppSectionHandler<Params, Props>
    {
        public Handler(IActivityRepository repo) { ... }

        public override async Task<Props> Render(Params p, AppSectionContext ctx)
        {
            var page = await repo.Page(p.ResidentId, p.Page, ctx.Cancellation);
            return new Props
            {
                Items = page.Items,
                Open  = ActivityDetailsRoute.Ref(default),       // dummy until row
                Next  = page.NextCursor is null ? null : new()
                    { Token = page.NextCursor, Next = ResidentDetailsRoute.Ref(p.ResidentId) }
            };
        }
    }
}
```

### Wire — inline in parent props

```json
{ "activity": { "$kind": "section", "id": "ResidentActivity", "params": { "residentId": "5f8e...", "page": 1 }, "deferred": false, "props": { "items": [...], "open": {...} }, "etag": "W/\"af71\"" } }
```

### Wire — partial reload response (§23)

```json
{
  "protocol": { "version": "1" },
  "partials": [{
    "section": "ResidentActivity",
    "component": "App.Features.Residents.Sections.ResidentActivitySection",
    "renderer": "react",
    "props":    { "items": [...], "open": {...} },
    "chunks":   ["activity-section.7c12.js"],
    "etag":     "W/\"af71\""
  }],
  "props":    null,
  "status":   200
}
```

### Generated TypeScript — `SectionHandle<TProps>`

```ts
export interface SectionHandle<TProps> {
  readonly id:        string;
  readonly props:     TProps;
  readonly etag?:     string;
  reload(): Promise<TProps>;
  invalidate(): void;                  // mark stale; refetch on next view
  subscribe(fn: (p: TProps) => void): () => void;
  prefetch(): Promise<void>;
}
```

### Rules
- A section's `Params` MUST be derivable from the parent page's `Params` and any resource identities present (`ALIS0008`).
- A section MAY be streamed (§24) or deferred (§40).
- A section's `etag` is computed by the framework from canonicalized props; the client uses it to short-circuit identical reloads.

---

## 13. Page handler — canonical form

### Purpose
The server handler that takes route params and produces an `AppView<TProps>`. It is where the typed graph is composed for a single navigation.

### C# author surface

```csharp
public sealed class ResidentDetailsPage
    : AppPage<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public override async Task<AppView<ResidentDetailsRoute.Props>> Render(
        ResidentDetailsRoute.Params p, AppPageContext ctx)
    {
        var resident = await ctx.Resources.Project<ResidentResource>(new ResidentResource.Identity(p.Id));

        var props = new ResidentDetailsRoute.Props
        {
            Resident    = resident,
            Permissions = ctx.Capabilities.Project<ResidentDetailsRoute.Props.Permissions>(resident),
            Links       = ctx.Links.For<ResidentDetailsRoute.Props.Links>(p),
            Sections    = new()
            {
                Activity = ctx.Sections.Ref<ResidentActivitySection>(new() { ResidentId = p.Id }),
                CarePlan = ctx.Sections.Ref<ResidentCarePlanSection>(new() { ResidentId = p.Id }),
                Audit    = ctx.Sections.Ref<ResidentAuditSection>   (new() { ResidentId = p.Id }),
            },
            Meta        = ctx.Meta.Default(title: $"{resident.Resident.FirstName} {resident.Resident.LastName}"),
        };

        return App.Render(props)
            .WithLayout<TenantShellLayout>()
            .Stream(x => x.Sections.Activity)
            .Stream(x => x.Sections.CarePlan)
            .Defer (x => x.Sections.Audit)
            .Cache (TimeSpan.FromSeconds(15))
            .Title ("residents.details.title", new { name = resident.Resident.FirstName })
            .Trace ("residents.details");
    }

    public override Task<AppView<ResidentDetailsRoute.Props>> RenderFallback(
        ResidentDetailsRoute.Params p, AppError error, AppPageContext ctx)
        => error.Code switch
        {
            ErrorCode.NotFound => Task.FromResult(App.Render(new ResidentDetailsRoute.Props { /* skeleton */ }).Status(404)),
            _                  => base.RenderFallback(p, error, ctx),
        };
}
```

### Rules
- Resources are **constructed** only in `IResourceProjector<T>` implementations and **composed** only in pages.
- `AppView` is immutable; every builder method returns a new view.
- A page returning `Redirect<TRoute>(...)` short-circuits rendering.
- A page MAY throw `AppCancelException` to indicate cancellation; the framework MUST translate it into a `499` and abort streams.

---

## 14. Layout — canonical form

### Purpose
A composable server component with named slots and per-slot persistence policy. Persistent slots survive partial reloads; transient slots are replaced.

### C# author surface

```csharp
public sealed class TenantShellLayout : AppLayout<TenantShellLayout.Props>
{
    public sealed record Props
    {
        public required TenantHeaderProps Header { get; init; }
        public required NavTreeProps      Nav    { get; init; }
        public required FooterProps       Footer { get; init; }
    }

    public override LayoutSlots DeclareSlots() => new()
    {
        { "header",  SlotPolicy.Persistent },
        { "nav",     SlotPolicy.Persistent },
        { "main",    SlotPolicy.Transient  },
        { "aside",   SlotPolicy.Transient  },
        { "footer",  SlotPolicy.Persistent },
    };

    public override async Task<Props> Project(AppLayoutContext ctx)
    {
        var tenant = await ctx.Services.GetRequiredService<ITenantService>().Get(ctx.Tenant.Id);
        var nav    = await ctx.Services.GetRequiredService<INavService>().Tree(ctx.User);
        return new Props
        {
            Header = new() { TenantName = tenant.Name, User = ctx.User.Identity?.Name ?? "" },
            Nav    = new() { Items = nav.Items },
            Footer = new() { Year = ctx.Services.GetRequiredService<IClock>().Now.Year },
        };
    }
}
```

### Wire — layout field

```json
{ "layout": { "id": "TenantShellLayout", "slots": { "header":"persistent", "nav":"persistent", "main":"transient", "aside":"transient", "footer":"persistent" }, "props": { "header":{...}, "nav":{...}, "footer":{...} } } }
```

### Rules
- A page binds its layout via `WithLayout<T>()` on `AppView`. The framework supplies a default layout if none is bound (configurable in `AddAlisSsr`).
- A partial reload targeting only transient slots MUST NOT re-render persistent slots.
- A layout switch (different layout id between current and target) downgrades the navigation to `full`.

---

## 15. Component identity — canonical form

Every server-renderable thing has a stable string identity. Identity is the join key between server emit, client manifest, AI tooling, and the chunk graph.

```csharp
// default identity = "App.Features.Residents.ResidentDetailsPage"
public sealed class ResidentDetailsPage : AppPage<...> { }

// override
[ComponentIdentity("residents.details")]
public sealed class ResidentDetailsPage : AppPage<...> { }
```

Two components ending up with the same identity raise `ALIS0014`. The framework MUST refuse to start.

---

## 16. Island — canonical form

### Purpose
An island is a client-hydrated component embedded in SSR output. Each island is a separate hydration unit, renderer, chunk, and lifecycle.

### C# author surface — declaration

```csharp
[Island("ResidentScheduler")]
[Renderer("vue")]
[Chunk("scheduler")]
public sealed class ResidentSchedulerIsland : AppIsland<ResidentSchedulerIsland.Props>
{
    public sealed record Props
    {
        public required ResourceRef<ResidentResource>     Resident      { get; init; }
        public required IReadOnlyList<Appointment>        Appointments  { get; init; }
        public required ActionRef<RescheduleAppointment>  Reschedule    { get; init; }
        public required SectionRef<SchedulerOptionsProps> Options       { get; init; }
    }

    public override IslandManifest Manifest => new()
    {
        Mode               = HydrationMode.Visible,
        FallbackComponent  = typeof(ResidentSchedulerSkeleton),
        InvalidatesOnFocus = false,
    };
}
```

### Usage from a page handler

```csharp
return App.Render(props)
    .WithIsland<ResidentSchedulerIsland>(
        slot: "main",
        props: new ResidentSchedulerIsland.Props
        {
            Resident     = ctx.Resources.RefTo<ResidentResource>(new ResidentResource.Identity(p.Id)),
            Appointments = appointments,
            Reschedule   = RescheduleAppointment.Ref(new() { ResidentId = p.Id }),
            Options      = ctx.Sections.Ref<SchedulerOptionsSection>(new() { ResidentId = p.Id }),
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

Why a sibling `<script>` and not inline JSON in an attribute: large JSON in `data-` attributes blocks HTML streaming and breaks attribute-quoting for arbitrary user content. The `data-props-ref` sentinel allows lazy materialization of props.

### Hydration modes

| Mode | Trigger |
|------|---------|
| `eager` | Runtime boot, document order. |
| `visible` | IntersectionObserver intersection. |
| `idle` | `requestIdleCallback`. |
| `interaction` | First user interaction on placeholder. |
| `media` | Media query match. |

### Hydration lifecycle

```
discover  (server-emitted placeholders, JSON props sibling)
  -> schedule (per data-mode)
  -> resolveRenderer
  -> resolveChunk        (dynamic import via islands manifest)
  -> materializeRefs     (§40)
  -> renderer.mount(target, component, props, ctx)
  -> bindInvalidation    (subscribe to section/resource events)
```

Hydration is cancelable. If the placeholder is removed before mount completes, the renderer's `MountHandle.aborted` MUST become `true` and any `dispose()` callbacks MUST be invoked.

---

## 17. Renderer — canonical form

### Purpose
A renderer is a client framework adapter (React/Vue/Solid/Preact) for mount, update, and unmount. The protocol does not know what React is.

### TypeScript contract

```ts
export interface ClientRenderer {
  readonly id: "react" | "vue" | "solid" | "preact" | string;
  mount(target: Element, component: ComponentRef, props: unknown, ctx: RenderCtx): MountHandle;
  update(handle: MountHandle, props: unknown): void;
  unmount(handle: MountHandle): void;
  ssr?(component: ComponentRef, props: unknown, ctx: SsrCtx): Promise<SsrResult>;   // server-side via Node worker
}

export interface ComponentRef {
  readonly id:   string;                  // component identity
  readonly load: () => Promise<unknown>;  // dynamic import via manifest
}

export interface MountHandle {
  readonly dispose: () => void;
  readonly aborted: boolean;
}

export interface RenderCtx {
  readonly bus:    EventBus;
  readonly cache:  ClientCache;
  readonly tenant: TenantInfo;
  readonly locale: string;
  readonly trace:  string;
}

export interface SsrCtx {
  readonly tenant: TenantInfo;
  readonly locale: string;
  readonly cookies: ReadonlyMap<string,string>;
  readonly trace:  string;
}

export interface SsrResult {
  readonly html:    string;
  readonly chunks:  string[];
  readonly preload: string[];
}
```

### Server-side registration

```csharp
opt.AddRenderer<ReactRenderer>("react").Default();
opt.AddRenderer<VueRenderer>("vue");
```

The server-side `ReactRenderer` etc. are .NET types implementing `IServerRenderer`:

```csharp
public interface IServerRenderer
{
    string Id { get; }
    Task<SsrResult> Render(SsrRequest req, CancellationToken ct);   // calls Node worker
}
```

### Rules
- Multiple renderers may participate in one page (e.g. React page + Vue scheduler island).
- A component pinned to an unregistered renderer raises a startup error (`ALIS0023`).
- The shared protocol runtime is loaded exactly once regardless of how many renderers are in use (chunk dedup).

---

## 18. Action — canonical form

### Purpose
A typed server mutation identified by a stable action id, with typed request, response, validator, optional optimistic projection, idempotency mode, policy, and invalidation.

### C# author surface

```csharp
namespace App.Features.Residents.Actions;

[Action("residents.save")]
[Idempotent(IdempotencyMode.ClientKey)]
[Policy("EditResident")]
public sealed partial class SaveResident : AppAction<SaveResident.Request, SaveResident.Response>
{
    public override TimeSpan Timeout => TimeSpan.FromSeconds(10);
    public override int      MaxRetries => 2;

    public sealed record Request
    {
        public required Guid      Id        { get; init; }
        public required string    FirstName { get; init; }
        public required string    LastName  { get; init; }
        public          DateOnly? Dob       { get; init; }
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
            When(x => x.Dob is not null, () =>
            {
                RuleFor(x => x.Dob).Before(_ => DateOnly.FromDateTime(DateTime.UtcNow));
            });
        }
    }

    [Optimistic]
    public sealed class Optimism : AppOptimism<Request, ResidentResource>
    {
        public override ResidentResource Project(ResidentResource current, Request req)
            => current with
            {
                Resident = current.Resident with
                {
                    FirstName = req.FirstName,
                    LastName  = req.LastName,
                    Dob       = req.Dob,
                },
            };
    }

    public sealed class Handler : AppActionHandler<Request, Response>
    {
        public Handler(IResidentService residents) { ... }

        public override async Task<AppActionResult<Response>> Handle(
            Request req, AppActionContext ctx)
        {
            if (await ctx.TryReplay<Response>() is { } cached) return cached;

            var updated = await residents.Save(req, ctx.Cancellation);

            ctx.Invalidate
               .Section<ResidentActivitySection>(new() { ResidentId = req.Id })
               .Resource<ResidentResource>(new ResidentResource.Identity(req.Id));

            var result = App.Ok(new Response { Resident = updated }, ctx.Invalidate.Build().ToArray());
            await ctx.Record(result);
            return result;
        }
    }
}
```

### Wire — request

```http
POST /__alis/action HTTP/1.1
Content-Type:            application/json
X-Alis-Action:           residents.save
X-Alis-Idempotency-Key:  9b2c-...
X-Alis-Route:            ResidentDetailsRoute
X-Alis-Section:
X-Alis-Asset-Version:    2026.05.22.1
X-Alis-Trace:            00-...-01

{ "id":"5f8e...", "firstName":"Augusta", "lastName":"Lovelace", "dob":null }
```

### Wire — response (success)

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

### Wire — response (validation failure)

```json
{
  "protocol": { "version": "1" },
  "error": {
    "code":   "ValidationFailed",
    "fields": { "firstName": ["required"], "lastName": ["maxLength"] },
    "traceId":"00-...-01"
  }
}
```

### Idempotency modes (full table)

| Mode | Key source | Replay window | Semantics |
|------|-----------|---------------|-----------|
| `None` | none | n/a | Every call executes. |
| `ClientKey` | `X-Alis-Idempotency-Key` | TTL in cache (default 1h) | Replay returns cached result; new key forces re-execute. |
| `ServerHash` | `sha256(actionId + principal + canonical(request))` | TTL in cache | Identical requests within window dedupe. |
| `RequestScope` | per HTTP request | n/a | Multiple invokes in the same envelope dedupe. |

### Generated TypeScript

```ts
// standalone
await actions.residents.save.execute({ id, firstName, lastName });

// resource-bound
await resident.actions.save.execute({ firstName, lastName });
resident.actions.save.optimistic({ firstName: "Pending..." });
```

`ActionHandle<TReq, TResp>`:

```ts
export interface ActionHandle<TReq, TResp> {
  readonly id: string;
  execute(req: TReq, opts?: ExecuteOptions): Promise<ActionResult<TResp>>;
  optimistic(req: TReq): OptimisticHandle;
  prefetchPolicy(): Promise<void>;       // no-op for unauthorized
}

export type ActionResult<TResp> = { result: TResp } | { error: AppErrorJson };

export interface OptimisticHandle { commit(): void; rollback(): void; }
```

### Rules
- The action id is the wire identity. The C# type name is irrelevant to the wire (good — renaming the type doesn't break wire compatibility *if* the `[Action]` id stays).
- Policy enforcement runs before validation; policy failure yields `403`, validation failure yields `422`.
- `Invalidate` calls are aggregated; the client applies them after a successful commit. Optimistic rollback un-invalidates speculative state.
- `[LongRunning]` actions return immediately with an `OperationRef` (§22).

---

## 19. Validation — canonical form

Validation rules are server-canonical; the client receives **metadata**, never executable code.

### Rule kinds (v1)

`required`, `minLength`, `maxLength`, `pattern`, `min`, `max`, `email`, `url`, `oneOf`, `date`, `before`, `after`, `equalsField`, `differsField`, `requiredWhen`, `forbiddenWhen`, `custom`.

### C# author surface

```csharp
public sealed class Validator : AppValidator<SaveResident.Request>
{
    public Validator()
    {
        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.LastName ).NotEmpty().MaximumLength(100);

        When(x => x.Dob is not null, () =>
        {
            RuleFor(x => x.Dob).Before(_ => DateOnly.FromDateTime(DateTime.UtcNow));
        });

        RuleFor(x => x.FirstName)
            .Custom((value, ctx) =>
            {
                if (value.Contains("..")) ctx.Fail("firstName.invalid");
            });
    }
}
```

### Generated TypeScript

```ts
export const validators = {
  saveResidentRequest: {
    fields: {
      firstName: [{ kind: "required" }, { kind: "maxLength", value: 100 }, { kind: "custom", id: "firstName.invalid" }],
      lastName:  [{ kind: "required" }, { kind: "maxLength", value: 100 }],
      dob:       [{ kind: "date" }, { kind: "before", value: "today", when: "dob != null" }]
    }
  }
} as const satisfies ValidatorMetadata;
```

### Predicate language (for `When`/`Unless`)

A tiny canonical predicate language, parseable on the client:

```
predicate := orExpr
orExpr    := andExpr ('||' andExpr)*
andExpr   := unaryExpr ('&&' unaryExpr)*
unaryExpr := '!' atom | atom
atom      := comparison | '(' predicate ')' | boolLit
comparison:= fieldPath op valueExpr
op        := '==' | '!=' | '<' | '<=' | '>' | '>='
fieldPath := ident ('.' ident)*
valueExpr := literal | fieldPath | 'null'
literal   := number | string | 'true' | 'false' | 'today'
```

Anything not expressible in this grammar collapses to a `custom` rule and is server-only.

### Rules
- Validators are nested inside their owning Action; one validator per request type (`ALIS0006`).
- The metadata tree is keyed by request type **id**, not C# name.
- Conditional rules emit a `when` predicate against the request shape.

---

## 20. Form binding — canonical form

### Purpose
Bind a typed action request to a server-rendered form, with validation metadata and submit ergonomics, in a renderer-agnostic way.

### C# author surface (page side)

```csharp
public sealed class ResidentEditPage : AppPage<ResidentEditRoute.Params, ResidentEditRoute.Props>
{
    public override async Task<AppView<ResidentEditRoute.Props>> Render(
        ResidentEditRoute.Params p, AppPageContext ctx)
    {
        var resident = await ctx.Resources.Project<ResidentResource>(new ResidentResource.Identity(p.Id));

        var form = ctx.Forms.For(new SaveResident.Request
        {
            Id        = p.Id,
            FirstName = resident.Resident.FirstName,
            LastName  = resident.Resident.LastName,
            Dob       = resident.Resident.Dob,
        });

        return App.Render(new ResidentEditRoute.Props { Resident = resident, Form = form });
    }
}
```

### Wire — form ref

```json
{
  "form": {
    "$kind":      "form",
    "submit":     { "$kind": "action", "id": "residents.save" },
    "initial":    { "id": "5f8e...", "firstName": "Ada", "lastName": "Lovelace", "dob": null },
    "validator":  "saveResidentRequest",
    "ruleSets":   []
  }
}
```

### Generated TypeScript — `FormHandle<TReq>`

```ts
export interface FormHandle<TReq> {
  readonly initial: TReq;
  readonly validator: ValidatorMetadata;
  submit(req: TReq): Promise<ActionResult<unknown>>;
  validate(req: TReq): FormValidationResult;        // local validation against metadata
  bindField<K extends keyof TReq>(key: K): FieldBinding<TReq[K]>;
}

export interface FormValidationResult {
  readonly valid: boolean;
  readonly errors: Record<string, string[]>;
}

export interface FieldBinding<TValue> {
  readonly value: TValue;
  setValue(v: TValue): void;
  readonly errors: string[];
  readonly touched: boolean;
  readonly required: boolean;
}
```

### Rules
- Forms are renderer-agnostic. React/Vue/Solid/Preact form libraries consume `FormHandle` via small adapter packages.
- A `FormHandle` MAY be reused across renders; `submit` carries the latest values and applies the latest validator metadata.
- Submission failure carrying `ValidationFailed` MUST surface field errors into the form binding without throwing.

---

## 21. File upload (multipart action) — canonical form

### Purpose
An action whose request is multipart, carrying both typed fields and one or more files. The wire is `multipart/form-data` instead of JSON; everything else is the same.

### C# author surface

```csharp
[Action("residents.uploadPhoto")]
[MultipartRequest]
[Policy("EditResident")]
public sealed partial class UploadResidentPhoto
    : AppAction<UploadResidentPhoto.Request, UploadResidentPhoto.Response>
{
    public sealed record Request
    {
        public required Guid       Id    { get; init; }
        public required FileUpload Photo { get; init; }
    }

    public sealed record Response
    {
        public required MediaRef Avatar { get; init; }
    }

    public sealed class Validator : AppValidator<Request>
    {
        public Validator()
        {
            RuleFor(x => x.Photo).Required().MaxSize(8 * 1024 * 1024).MimeIn("image/jpeg", "image/png", "image/webp");
        }
    }

    public sealed class Handler : AppActionHandler<Request, Response>
    {
        public Handler(IMediaService media) { ... }

        public override async Task<AppActionResult<Response>> Handle(Request req, AppActionContext ctx)
        {
            var stored = await media.Store(req.Photo, ctx.Cancellation);
            ctx.Invalidate.Resource<ResidentResource>(new ResidentResource.Identity(req.Id));
            return App.Ok(new Response { Avatar = ctx.Refs.Media(stored.Id) }, ctx.Invalidate.Build().ToArray());
        }
    }
}

public sealed class FileUpload
{
    public required string       FileName    { get; init; }
    public required string       ContentType { get; init; }
    public required long         Length      { get; init; }
    public required Stream       Content     { get; init; }
}
```

### Wire — request

```
POST /__alis/action HTTP/1.1
Content-Type: multipart/form-data; boundary=----alis
X-Alis-Action:           residents.uploadPhoto
X-Alis-Idempotency-Key:  9b2c-...

------alis
Content-Disposition: form-data; name="payload"
Content-Type: application/json

{ "id": "5f8e..." }
------alis
Content-Disposition: form-data; name="Photo"; filename="ada.jpg"
Content-Type: image/jpeg

<binary>
------alis--
```

The `payload` part carries the JSON; each `FileUpload` field is a sibling part keyed by the C# property name.

### Generated TypeScript

```ts
await resident.actions.uploadPhoto.execute({
  id: resident.id,
  photo: file,                       // browser File / Blob
});
```

The generated client serializes `File`/`Blob` fields as multipart parts automatically; non-file fields go into the `payload` part.

### Rules
- A multipart action MUST declare exactly one `payload` part for JSON.
- Field validators (`Required`, `MaxSize`, `MimeIn`, etc.) emit metadata kinds visible to the client for pre-flight checks.
- Streaming uploads (HTTP/1.1 chunked, HTTP/2 streams) MUST be supported end to end.

---

## 22. Long-running operation — canonical form

### Purpose
An action that may exceed a normal request timeout. The action returns immediately with an `OperationRef`; the client polls or subscribes to operation status.

### C# author surface

```csharp
[Action("residents.export")]
[LongRunning]
[Policy("ExportResidents")]
public sealed partial class ExportResidents : AppAction<ExportResidents.Request, ExportResidents.Response>
{
    public sealed record Request
    {
        public required string Format { get; init; }       // "csv" | "xlsx"
        public required string Filter { get; init; }
    }

    public sealed record Response
    {
        public required MediaRef File { get; init; }
    }

    public sealed class Handler : AppActionHandler<Request, Response>
    {
        public Handler(IExportService exports) { ... }

        public override async Task<AppActionResult<Response>> Handle(Request req, AppActionContext ctx)
        {
            var op = await ctx.Operations.Start<ExportResidents>(req);
            _ = Task.Run(async () =>
            {
                try
                {
                    await foreach (var pct in exports.Run(req, ctx.Cancellation))
                        await ctx.Operations.Progress(op.OperationId, pct);

                    var file = await exports.Finalize(req, ctx.Cancellation);
                    await ctx.Operations.Complete(op.OperationId, new Response { File = ctx.Refs.Media(file.Id) });
                }
                catch (Exception ex)
                {
                    await ctx.Operations.Fail(op.OperationId, AppError.From(ex));
                }
            });
            return App.Ok(new Response { File = default! }, Array.Empty<IInvalidationRef>());   // placeholder; client uses OperationRef
        }
    }
}
```

### Wire — initial response

```json
{
  "protocol":  { "version": "1" },
  "operation": { "$kind": "operation", "id": "op_5f8e...",
                 "status": { "$kind": "route", "id": "OperationStatusRoute", "params": { "id": "op_5f8e..." } },
                 "cancel": { "$kind": "action", "id": "operations.cancel" } },
  "result":    null,
  "error":     null
}
```

### Wire — status route response

```json
{
  "protocol": { "version": "1" },
  "props": {
    "operation": {
      "id":       "op_5f8e...",
      "state":    "Running",                           // Pending|Running|Completed|Failed|Cancelled
      "progress": 0.42,
      "message":  "Exporting 4200/10000",
      "result":   null,
      "error":    null,
      "startedAt":"2026-05-22T08:00:00Z",
      "updatedAt":"2026-05-22T08:00:21Z"
    }
  }
}
```

When complete, `result` carries the typed response (e.g. `{ "file": { "$kind": "media", "id": "exports/...", "url": "..." } }`).

### Generated TypeScript

```ts
const op = await actions.residents.export.execute({ format: "csv", filter: "active" });
if ("result" in op) {
  const ref: OperationHandle<ExportResidentsResponse> = op.result.operation;
  ref.subscribe(state => /* update progress UI */ );
  ref.onComplete(r => downloadMedia(r.file));
  // or: const final = await ref.wait();
}
```

`OperationHandle<TResp>`:

```ts
export interface OperationHandle<TResp> {
  readonly id: string;
  readonly state: "Pending"|"Running"|"Completed"|"Failed"|"Cancelled";
  readonly progress: number;        // 0..1
  readonly message?: string;
  subscribe(fn: (h: OperationHandle<TResp>) => void): () => void;
  onComplete(fn: (r: TResp) => void): void;
  onError(fn: (e: AppErrorJson) => void): void;
  wait(): Promise<TResp>;
  cancel(): Promise<void>;
}
```

### Rules
- The client polls `OperationStatusRoute` by default; if SSE is enabled (`AddInvalidationBus(.UseSse())`), operations push updates instead of polling.
- Default poll interval doubles with backoff; minimum 1s, maximum 30s; configurable.
- Cancellation propagates to the server handler's `ctx.Cancellation`.

---

## 23. Partial reload — canonical form

### Purpose
Update a named section without re-running the entire page.

### C# canonical form

```csharp
return App.Render(props)
    .Stream(p => p.Sections.Activity);  // streamed sections are also independently reloadable
```

The page handler does not branch on `Strategy == Partial` for the common case; the framework calls the section handler directly when the client requests `?section=...`. A page MAY override `RenderPartial` for cross-section coordination.

### Wire — request

```http
GET /residents/5f8e...?section=ResidentActivity HTTP/1.1
X-Alis-Strategy:       partial
X-Alis-Section:        ResidentActivity
X-Alis-Asset-Version:  2026.05.22.1
If-None-Match:         W/"af71"
```

### Wire — response (200)

See §12.

### Wire — response (304)

```http
HTTP/1.1 304 Not Modified
ETag: W/"af71"
```

The client retains its existing section state; the envelope cache MAY extend TTL on a 304.

### Generated TypeScript

```ts
await resident.sections.activity.reload();
resident.sections.activity.invalidate();
```

---

## 24. Streaming SSR — canonical form

### Purpose
Render a shell immediately and stream named section payloads as they complete, in any order, with per-section error isolation.

### C# canonical form

```csharp
return App.Render(props)
    .WithLayout<TenantShellLayout>()
    .Stream(p => p.Sections.Activity)
    .Stream(p => p.Sections.CarePlan)
    .Defer (p => p.Sections.Audit);
```

### Wire — frames (NDJSON)

```
{"$frame":"shell","route":{...},"component":"...","renderer":"react","layout":{...},"props":{...},"chunks":[...],"preload":[...]}
{"$frame":"chunk","section":"ResidentCarePlan","props":{...},"chunks":["careplan.7c12.js"]}
{"$frame":"chunk","section":"ResidentActivity","props":{...},"chunks":["activity.7c12.js"]}
{"$frame":"error","section":"ResidentAudit","error":{"code":"Timeout","traceId":"00-..."}}
{"$frame":"end"}
```

- `Content-Type: application/x-ndjson`
- `Transfer-Encoding: chunked`
- One JSON object per line.

### Backpressure and timeouts

- Per-stream wall-clock timeout: configurable; default 30s.
- Per-section timeout: configurable; default 15s.
- Slow-client write timeout: configurable; default 5s. Exceeding it aborts the stream and cancels outstanding sections.

### Lifecycle (server)

```
open ndjson
  emit shell frame                       // before any I/O on sections
  start each Stream() section in parallel
  on each completion -> emit chunk frame (in completion order)
  on each timeout/exception -> emit error frame
  on all sections complete -> emit end frame
  close
```

If the client disconnects, the framework MUST propagate cancellation to all section handlers within 1 second.

---

## 25. Navigation — canonical form

### Purpose
Typed navigation with cache, prefetch, and strategy selection. No string URLs in app code.

### Generated TypeScript

```ts
import { routes } from "@app/generated";

await routes.residentDetails.navigate({ id });
await routes.residentDetails.navigate({ id }, { replace: true, scroll: "preserve", focus: "main" });
await routes.residentDetails.prefetch({ id });
const href = routes.residentDetails.href({ id });    // string only when an <a href> truly needs it
```

`NavigateOptions`:

```ts
export interface NavigateOptions {
  replace?: boolean;
  scroll?: "top" | "preserve" | { x: number; y: number };
  focus?: "main" | "skip-link" | string;       // CSS selector
  signal?: AbortSignal;
  strategy?: "auto" | "full" | "partial" | "stream";   // default: auto
}
```

### Navigation lifecycle

```
navigate(req)
  -> resolveRoute(req)
  -> selectStrategy(current, target, opts)
  -> emit "navigation.start"
  -> loadChunks(target.chunks)              // parallel; honors preload list
  -> fetchEnvelope(target, strategy)
  -> if envelope.assetVersion != client.assetVersion -> forceReload()
  -> materializeRefs(envelope.props)        // §40
  -> applyEnvelope(envelope)                // mount or patch islands
  -> commit(target)                         // history, scroll, focus
  -> emit "navigation.commit"
```

### Strategy headers

| Header | Meaning |
|--------|---------|
| `X-Alis-Strategy` | `full` \| `partial` \| `stream` |
| `X-Alis-Asset-Version` | client's loaded asset version |
| `X-Alis-Section` | when `partial`, the requested section id |
| `X-Alis-Trace` | traceparent |
| `X-Alis-Tenant` | resolved tenant id (when client-side known) |
| `X-Alis-Process-Id` | server process id (echoed for HMR detection) |

### Rules
- A newer `navigate` MUST cancel any in-flight older navigation atomically.
- Cancellation MUST NOT mutate `history` or scroll position.
- Back/forward MUST replay cached envelopes when valid; otherwise re-fetch with `X-Alis-Strategy: full`.

---

## 26. Prefetch — canonical form

### Purpose
Warm the cache for a likely-next navigation: load chunks, fetch envelope, materialize refs — without committing.

### Trigger policies (route-declared)

| Policy | Meaning |
|--------|---------|
| `Never` | no automatic prefetch |
| `OnViewport` | when a `<RouteLink>` enters the viewport |
| `OnHover` | when the user hovers a `<RouteLink>` |
| `OnFocus` | when a `<RouteLink>` receives keyboard focus |
| `Eager` | as soon as the page mounts (use sparingly) |

### Cache lifetime
A prefetched envelope is treated as a normal cache entry; it expires per `CachePolicy`. The framework MUST NOT count prefetch traffic against rate-limit policies meant for explicit user actions.

---

## 27. Cache and invalidation — canonical form

### Cache key

`(routeId, canonical(params))`. Canonical params sort keys lexicographically, drop nulls/defaults, and lowercase enum literals.

### Cache policies

```csharp
public abstract record CachePolicy
{
    public sealed record None    : CachePolicy;
    public sealed record Public  (TimeSpan MaxAge) : CachePolicy;
    public sealed record Private (TimeSpan MaxAge) : CachePolicy;
    public sealed record SwrPolicy(TimeSpan MaxAge, TimeSpan Stale) : CachePolicy;
    public sealed record MustRevalidate(TimeSpan MaxAge) : CachePolicy;
}
```

### Generated TypeScript

```ts
client.cache.get(routes.residentDetails, { id });
client.cache.put(routes.residentDetails, { id }, envelope);
client.cache.invalidate(routes.residentDetails, { id });

// section-scoped
client.cache.invalidate(sections.residentActivity, { residentId });

// resource-scoped
client.cache.invalidate(resources.resident, { value });
```

### Server-driven invalidation
Action responses (§18) and stream `end` frames carry an `invalidate` list; the client applies them after a successful commit.

### Cross-tab invalidation
See §39.

### Rules
- Cache entries are pinned to an `assetVersion`; stale-version entries are discarded on startup.
- `props.meta.cache.maxAge` mirrors the route's `CachePolicy`.
- `invalidate(...)` cascades to all section and resource refs nested inside cached envelopes.

---

## 28. Optimistic mutation — canonical form

### Purpose
Apply a local projection of a mutation immediately, with deterministic rollback on failure.

### C# author surface

See §18 (the `[Optimistic]` nested class).

### Generated TypeScript

```ts
const projection = resident.actions.save.optimistic({ firstName, lastName });
const result = await resident.actions.save.execute({ firstName, lastName });
if ("error" in result) projection.rollback();
```

### Rules
- Optimism is opt-in per action.
- A failed `execute` MUST roll back exactly the projection created with the same call id.
- Multiple optimistic calls on the same resource stack; rollback unwinds them in reverse.
- The framework MUST NOT silently drop a stacked rollback; if a parent is rolled back, descendants are also rolled back.

---

## 29. Error envelope — canonical form

### Wire form

```json
{
  "error": {
    "code":       "ValidationFailed",
    "message":    null,
    "fields":     { "firstName": ["required"] },
    "retryAfter": null,
    "traceId":    "00-af7c-..."
  }
}
```

### Categories (HTTP status mapping)

See §1.11 `ErrorCode.HttpStatus`.

### Client behavior

| Code | Client behavior |
|------|----------------|
| `ValidationFailed` | surfaced into the calling `FormHandle` / returned to caller. |
| `NotAuthenticated` | triggers re-auth flow exactly once per navigation. |
| `NotAuthorized` | typed return; capability projection should have prevented this. |
| `NotFound` | typed return; pages may swap into a 404 view via `RenderFallback`. |
| `Conflict` | typed return; UI surfaces retry. |
| `AssetVersionMismatch` | full reload via `force-reload` directive. |
| `RateLimited` | typed return; UI surfaces retry with `retryAfter`. |
| `Internal` / `Unavailable` | typed return; UI surfaces global error. |

### Rules
- Actions return `{ result } | { error }`; they never throw for expected errors.
- `ValidationFailed.fields` is a map of field path → array of message keys (not human strings).
- `AssetVersionMismatch` is framework-handled; app code does not see it.

---

## 30. Polymorphic response / discriminated union — canonical form

### Purpose
Allow an action response (or a resource field) to be one of several typed shapes, with safe pattern-matching on the client.

### C# author surface

```csharp
[PolymorphicResponse]
public abstract record EnrollmentOutcome
{
    public sealed record Created(Guid Id)                             : EnrollmentOutcome;
    public sealed record AlreadyExists(Guid Id)                       : EnrollmentOutcome;
    public sealed record Throttled(TimeSpan RetryAfter)               : EnrollmentOutcome;
    public sealed record Rejected(string Reason)                      : EnrollmentOutcome;
}
```

### Wire form

```json
{ "outcome": { "$type": "Created", "id": "5f8e..." } }
```

The `$type` discriminator is the **simple type name** of the case. Renaming the case is a wire-breaking change.

### Generated TypeScript

```ts
export type EnrollmentOutcome =
  | { $type: "Created";       id: string }
  | { $type: "AlreadyExists"; id: string }
  | { $type: "Throttled";     retryAfter: number }    // TimeSpan -> seconds (number)
  | { $type: "Rejected";      reason: string };
```

### Rules
- All case names within a union MUST be unique.
- A union member without `[PolymorphicResponse]` on the base is treated as a regular nested record (not a union) — the marker is required.
- The TS form uses `$type` (not `kind`) to avoid clashing with the `$kind` reference sentinel.

---

## 31. Policy / Auth — canonical form

### C# author surface

```csharp
public static class Policies
{
    public static readonly AuthPolicy ViewResident    = new("ViewResident");
    public static readonly AuthPolicy EditResident    = new("EditResident");
    public static readonly AuthPolicy ArchiveResident = new("ArchiveResident");
    public static readonly AuthPolicy ExportResidents = new("ExportResidents");
}

public sealed class EditResidentPolicy : IAppPolicy
{
    public string Name => Policies.EditResident.Name;

    public Task<AuthDecision> Authorize(AuthContext ctx)
        => Task.FromResult(ctx.User.HasClaim("perm", "resident.edit")
            ? AuthDecision.Allow
            : AuthDecision.Deny("missing perm:resident.edit"));
}
```

### Attaching policies

```csharp
[Policy("EditResident")] public sealed partial class SaveResident       : AppAction<...> { }
[Policy("EditResident")] public sealed class       ResidentEditRoute   : AppRoute<...>  { }
[Policy("EditResident")] public sealed class       ResidentEditSection : AppSection<...>{ }
```

### Enforcement order

1. Tenant resolution.
2. Authentication.
3. Route/section/action policy evaluation.
4. Param/request binding.
5. Validation (Actions only).
6. Handler invocation.

A failure at any step short-circuits with the appropriate `ErrorCode`.

### Rules
- Policy names form a flat global namespace.
- A policy referenced but not registered raises `ALIS0024` at startup.

---

## 32. Capability projection — canonical form

### Purpose
Server-evaluated capabilities are projected into Resource and Props records; the client treats them as truth.

### C# author surface

```csharp
public sealed record Permissions : IAppPermissions
{
    [Capability("canEdit")]    public required bool CanEdit    { get; init; }
    [Capability("canArchive")] public required bool CanArchive { get; init; }
}

// projector
public sealed class ResidentCapabilityProjector : ICapabilityProjector<ResidentResource.Permissions>
{
    public ResidentCapabilityProjector(IPolicyEvaluator policies) { ... }

    public async Task<ResidentResource.Permissions> Project(object subject, AuthContext ctx)
        => new()
        {
            CanEdit    = await policies.Evaluate(Policies.EditResident,    subject, ctx),
            CanArchive = await policies.Evaluate(Policies.ArchiveResident, subject, ctx),
        };
}
```

### Wire form

```json
{ "permissions": { "canEdit": true, "canArchive": false } }
```

### Generated TypeScript guards (dev only)

```ts
// emitted in dev builds; no-op in production
if (process.env.NODE_ENV !== "production") {
  if (!resident.permissions.canEdit) {
    console.warn("Action SaveResident invoked despite canEdit=false; server will reject.");
  }
}
```

Server enforcement (§31) remains the real boundary; capabilities are an affordance hint.

### Rules
- A capability flag referenced in props without a corresponding `[Policy]` declaration raises `ALIS0012`.
- Capabilities are projected per-render; they are not cached separately on the client.

---

## 33. Tenant — canonical form

### Resolution

```csharp
opt.AddTenants(t =>
{
    t.ResolveFromSubdomain();             // *.app.example.com
    t.ResolveFromHeader("X-Tenant");      // fallback
    t.ResolveFromClaim("tenant_id");      // fallback
    t.AddOverride("acme",   o => { o.AssetVersion = "2026.05.22.1-acme"; o.Locale = "en-US"; });
    t.AddOverride("globex", o => { o.AssetVersion = "2026.05.22.1-globex"; o.Locale = "en-GB"; });
});
```

### Envelope projection

```json
{ "meta": { "tenant": "acme", "locale": "en-US" } }
```

### Rules
- A tenant override pins to an `assetVersion` already published; pinning to an unpublished version raises a startup error.
- A locale switch is a partial reload when the layout permits; full otherwise.
- A `ITenantInfo` is available on every context object; the framework MUST scope DI by tenant.

---

## 34. Localization — canonical form

### C# author surface

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

### Resolution

The shell envelope carries the resolved catalog for keys used by the page:

```json
{ "meta": { "i18n": { "residents.details.title": "Resident — {name}" } } }
```

Lazy keys fetched via `GET /__alis/i18n?keys=...&locale=...`.

### Rules
- Keys are static `LocalizationKey` constants; runtime string construction is forbidden.
- A key referenced in props but missing in the active locale falls back to the key name and emits a warning span.
- Plurals and ICU MessageFormat are supported via key arguments (`{name, plural, ...}`).

---

## 35. Asset version / Chunk — canonical form

### Asset manifest (`alis.assets.json`)

```json
{
  "assetVersion": "2026.05.22.1",
  "renderers":    { "react": "react.abcd1234.js", "vue": "vue.7c12dead.js" },
  "components": {
    "App.Features.Residents.ResidentDetailsPage": {
      "chunk":   "residents-details.ef561234.js",
      "css":     ["residents-details.ef561234.css"],
      "preload": ["scheduler.abcd1234.js"]
    }
  },
  "chunks": { "residents-details.ef561234.js": { "file": "...", "imports": [...], "css": [...] } },
  "islands": {
    "ResidentScheduler": { "chunk": "scheduler.abcd1234.js", "renderer": "vue", "mode": "visible" }
  }
}
```

### Envelope projection

```json
{ "assetVersion": "2026.05.22.1", "chunks": ["residents-details.ef561234.js"], "preload": ["scheduler.abcd1234.js"] }
```

### Rules
- The SSR process pins one `assetVersion` for its lifetime.
- An envelope's `assetVersion` MUST equal the process's pinned value.
- A client envelope with stale `assetVersion` triggers a full reload via `AssetVersionMismatch` (§29).
- The server MUST keep the last N manifests addressable (default N=2) so an in-flight stale navigation can complete before the forced reload.

---

## 36. Observability — canonical form

### Required signals

| Signal | Producer | Where |
|--------|----------|-------|
| Trace span `alis.route` | route resolution | server |
| Trace span `alis.page.render` | page handler | server |
| Trace span `alis.section.render` | section handler | server |
| Trace span `alis.action.handle` | action handler | server |
| Trace span `alis.ssr.worker` | Node SSR worker | server |
| Trace span `alis.client.navigation` | client runtime | client |
| Trace span `alis.client.hydrate` | client runtime | client |
| Metric `alis.ssr.duration` (p50/p95) | server | per route |
| Metric `alis.action.duration` (p50/p95) | server | per action |
| Metric `alis.hydrate.duration` (p50/p95) | client | per island |
| Metric `alis.chunk.duration` (p50/p95) | client | per chunk |
| Log `alis.*` structured | both | includes `traceId`, `routeName`, `tenant`, `assetVersion` |

### Client telemetry transport

```csharp
opt.AddObservability(o => o.WithClientTelemetry(transport: ClientTelemetryTransport.OtlpHttp, endpoint: "/__alis/telemetry"));
```

The default endpoint accepts OTLP-over-HTTP JSON and forwards into the server's existing observability pipeline.

### Rules
- A single end-to-end trace MUST connect: browser navigation → server route → action → Node SSR worker → hydration.
- The framework MUST emit a `traceId` in every error envelope (§29).

---

## 37. Graph export — canonical form

`GET /__alis/graph` (admin-gated) returns a machine-readable representation of the entire graph:

```json
{
  "version":       "1",
  "assetVersion":  "2026.05.22.1",
  "routes":        [{ "id":"...", "pattern":"/...", "params":{...}, "props":{"$ref":"..."}, "policy":null, "layout":"...", "renderer":"react", "chunks":[...] }],
  "actions":       [{ "id":"residents.save", "request":{"$ref":"SaveResident.Request"}, "response":{"$ref":"..."}, "policy":"EditResident", "idempotency":"ClientKey", "optimistic":true, "invalidates":[...] }],
  "resources":     [{ "id":"Resident", "identity":{"value":"Guid"}, "links":{...}, "actions":[...], "permissions":[...] }],
  "sections":      [...],
  "components":    [...],
  "islands":       [...],
  "policies":      [...],
  "validators":    {...},
  "localization":  {...},
  "renderers":     ["react","vue"],
  "affordances":   ["navigate","href","prefetch","execute","optimistic","reload","invalidate","wait","cancel","submit"]
}
```

### Rules
- Every node carries a stable string id.
- The shape is versioned; a `version` bump is required for breaking changes.
- The endpoint is policy-gated; no anonymous access in any environment.

---

## 38. `App.Render` — canonical entry point

The single ergonomic developer surface for returning a typed page.

```csharp
return App.Render<ResidentDetailsRoute.Props>(props)
    .WithLayout<TenantShellLayout>()
    .Stream(p => p.Sections.Activity)
    .Defer (p => p.Sections.Audit);
```

```csharp
return App.RenderPartial<ResidentActivitySection.Props>(props);
```

```csharp
return App.RenderStream<ResidentDetailsRoute.Props>(b => b
    .Shell(shellProps)
    .Stream(p => p.Sections.Activity, () => activityTask)
    .Stream(p => p.Sections.CarePlan, () => carePlanTask));
```

Page handlers MUST NOT return raw HTTP responses or use any non-typed surface. `App.Render`, `App.RenderPartial`, `App.RenderStream` are the only sanctioned entry points (`ALIS0009`).

---

## 39. Cross-tab invalidation — canonical form

### Purpose
When user does X in tab A, tab B sees a fresh state on next view, without SignalR and without polling.

### Mechanism

The client runtime opens a `BroadcastChannel("alis-invalidation")` on boot. Every applied `invalidate` ref is broadcast to siblings on the same origin. Each tab applies received refs to its own cache.

Optional server push: `AddInvalidationBus(b => b.UseSse())` opens an SSE stream at `/__alis/events` per session. Frames:

```
event: invalidate
data: { "kind":"section",  "id":"ResidentActivity", "params": { "residentId": "..." } }

event: invalidate
data: { "kind":"resource", "type":"Resident",       "id": { "value": "..." } }
```

The server emits SSE frames after action handlers (or domain events) that explicitly request cross-session invalidation.

### Rules
- Broadcast is best-effort; the canonical state remains the server.
- A tab MUST NOT trust a broadcast to mutate its rendered state; it MUST refetch.
- SSE is opt-in and falls back gracefully to BroadcastChannel-only when unavailable.

---

# PART III — Protocol algorithms

## 40. Reference resolution algorithm (the `$kind` walker)

The wire `props` (and `result`, and section payloads) may contain typed references. The client materializes them into live handles during envelope application.

### Sentinels

| Sentinel | Meaning | Result type |
|----------|---------|------------|
| `{ "$kind": "route",    "id", "params" }` | route reference | `RouteHandle<TParams>` |
| `{ "$kind": "action",   "id", "context" }` | action reference | `ActionHandle<TReq, TResp>` |
| `{ "$kind": "section",  "id", "params", "props"?, "etag"?, "deferred"? }` | section reference | `SectionHandle<TProps>` |
| `{ "$kind": "resource", "type", "id", ... }` | resource | `ResourceProxy` |
| `{ "$kind": "media",    "id", "url", "variants"? }` | media | `MediaHandle` |
| `{ "$kind": "operation","id", "status", "cancel"? }` | long-running operation | `OperationHandle<TResp>` |
| `{ "$kind": "form",     "submit", "initial", "validator", "ruleSets"? }` | form | `FormHandle<TReq>` |
| `{ "$ref": "<table-id>" }` | dereference from `$resources` table | resolved value |

### Algorithm

```
materializeRefs(node, table)
  if node is null or primitive: return node
  if node is array:             return node.map(n => materializeRefs(n, table))
  if node has "$ref":           return materializeRefs(table[node["$ref"]], table)
  if node has "$kind":          return liveHandle(node, table)
  // plain object
  out = {}
  for (k, v) of node: out[k] = materializeRefs(v, table)
  return out

liveHandle(node, table) switch node.$kind
  case "route":     return new RouteHandle(node.id, materializeRefs(node.params, table))
  case "action":    return new ActionHandle(node.id, materializeRefs(node.context, table))
  case "section":   return new SectionHandle(node.id, materializeRefs(node.params, table),
                                              materializeRefs(node.props, table), node.etag, node.deferred)
  case "resource":  return new ResourceProxy(node.type, node.id, materializeRefs(rest(node), table))
  case "media":     return new MediaHandle(node.id, node.url, node.variants)
  case "operation": return new OperationHandle(node.id, materializeRefs(node.status, table), materializeRefs(node.cancel, table))
  case "form":      return new FormHandle(materializeRefs(node.submit, table), node.initial, node.validator, node.ruleSets)
```

### Determinism
- The walker visits properties in insertion order (JSON order).
- `$ref` cycles MUST be detected and raise a runtime error.
- The walker MUST be pure: no network I/O, no state mutation beyond constructing handles.

### Performance
- The walker MUST be incremental for partial responses: only the changed section's subtree is re-walked.
- The walker SHOULD share handle instances within an envelope when params and ids match (for `===` equality in renderers).

---

## 41. Chunk graph derivation

The chunk graph is computed from the union of (a) Vite's module graph and (b) the framework's component identity map.

### Inputs
- `vite manifest.json` — every emitted asset with its imports and CSS.
- `alis.graph.json` — component identities mapped to source files.
- `client/.generated/islands.ts` — island ids mapped to source files.
- Renderer registration list.

### Derivation steps

1. For each **page component identity**, locate its source TS module. The chunk that produces it is the page's primary chunk.
2. For each **island id**, locate its source TS module. Its chunk is one of: (a) a dedicated chunk if `[Chunk]` hints one, (b) merged with the page chunk if it is only used by that page, (c) a shared chunk if used by multiple pages.
3. For each **renderer adapter**, its chunk is shared and preloaded by any page using that renderer.
4. For each chunk, compute its **transitive imports** (closure of `imports`) and its **direct CSS**.
5. Emit `alis.assets.json` with `components`, `chunks`, `islands`, `renderers` sections.
6. For each component identity, compute the **page-critical preload set**: the union of (its chunk's imports) minus (the imports already loaded for the previous navigation). This is the `preload` list in the envelope.

### Rules
- The derivation MUST be deterministic given identical inputs.
- A component identity with no resolvable source file raises `ALIS0030`.
- A circular chunk import raises `ALIS0031` (Vite normally prevents this, but the framework checks).

---

## 42. Node SSR worker IPC contract

For renderers that produce HTML server-side via their JS runtime (React/Vue/Solid/Preact), `Alis.Ssr.Hosting` runs a Node worker pool. The IPC contract is stable across implementations.

### Transport
- **stdio JSON-RPC v2**, line-delimited.
- One worker process per renderer per host. Pool size configurable; default `Environment.ProcessorCount / 2`.
- Supervisor restarts crashed workers with exponential backoff (1s, 2s, 4s, max 30s).

### Methods

```
render(req: RenderRequest) -> RenderResult | RenderError
health() -> { ok: true, uptime: number, renderer: string }
ping() -> { pong: true }
shutdown() -> { ok: true }
```

### RenderRequest

```json
{
  "id":          "<correlation-id>",
  "component":   "App.Features.Residents.ResidentDetailsPage",
  "renderer":    "react",
  "props":       { ... },                  // already ref-materialized? no -- see Rules
  "assetVersion":"2026.05.22.1",
  "tenant":      { "id": "acme", "locale": "en-US" },
  "trace":       "00-...-01",
  "timeoutMs":   5000
}
```

### RenderResult

```json
{
  "id":      "<correlation-id>",
  "html":    "<div ...>...</div>",
  "chunks":  ["residents-details.ef561234.js"],
  "preload": ["scheduler.abcd1234.js"],
  "css":     ["residents-details.ef561234.css"]
}
```

### RenderError

```json
{ "id": "<correlation-id>", "error": { "code": "RenderFailed", "message": "...", "stack": "..." } }
```

### Notifications

```json
// log
{ "method": "log", "params": { "level": "warn", "trace": "00-...-01", "message": "..." } }
// metric
{ "method": "metric", "params": { "name": "alis.ssr.worker.duration", "value": 18.4, "trace": "..." } }
```

### Rules
- The worker MUST reject mismatched `assetVersion` loudly (no soft fallback).
- The worker MUST receive props as JSON; ref materialization is performed inside the worker against the same algorithm as the browser (§40).
- Shutdown is graceful: the worker stops accepting new requests, drains in-flight, then exits.

---

## 43. Dev mode HMR contract

### Dev orchestration

`alis ssr dev` runs three processes:

1. `dotnet watch run` — the .NET host with file-watch and incremental rebuild.
2. `vite` — the Vite dev server on a separate port, serving uncompiled TS with HMR.
3. Node SSR workers — managed by the .NET host, hot-restart on chunk changes.

### Process generation id

Every server start picks a fresh `X-Alis-Process-Id`. The client tracks it across navigations:

- If `X-Alis-Process-Id` changes mid-session, the client purges its envelope cache and reloads islands.
- If `X-Alis-Asset-Version` changes mid-session, the client purges cache and triggers `AssetVersionMismatch`.

### HMR events (Vite → client runtime)

The client runtime listens on Vite's HMR channel for the following custom events:

```ts
// when a TS island module changes
import.meta.hot.on("alis:island-changed", ({ islandId, chunk }) => {
  alis.runtime.replaceIslandChunk(islandId, chunk);
});

// when generated TS changes (model/route/action signatures shifted)
import.meta.hot.on("alis:generated-changed", () => {
  alis.runtime.softReload();      // re-fetch envelope; keep history
});
```

### Server-side changes (`.cs` files)

`dotnet watch` restarts the host. The client detects via `X-Alis-Process-Id` change on next request and soft-reloads. Open envelopes from the previous process are discarded.

### Rules
- HMR MUST work for renderer islands without breaking SSR rendering of the same components.
- A failed dev build (compile error) MUST be surfaced as a structured error overlay, not a generic 500.

---

## 44. Predicate language

See §19 grammar. The reference parser is shipped in `@alis/ssr-runtime/predicate`.

### Examples

| C# `When` clause | Emitted predicate |
|------------------|-------------------|
| `x => x.Dob is not null` | `dob != null` |
| `x => x.Status == ResidentStatus.Archived` | `status == "Archived"` |
| `x => x.Page > 1 && x.Filter != null` | `page > 1 && filter != null` |
| `x => x.FirstName == x.LastName` | `firstName == lastName` |
| `x => x.CustomThing()` (call) | (no canonical form) → `custom` rule |

---

## 45. Branded type generation

For every `IResourceIdentity` implementation, the generator emits a branded TS type:

```ts
type ResidentId = string & { readonly __brand: "ResidentId" };
```

For composite identities (records with multiple properties), the brand is applied to the object:

```ts
type CompositeKey = { scope: string; item: string } & { readonly __brand: "CompositeKey" };
```

Branding rules:
- The brand string is the identity record's simple type name.
- Branding has zero runtime cost; it is a compile-time TS construct.
- A function generated from a C# signature taking `ResidentResource.Identity` accepts only `ResidentId` in TS; bare `string` is rejected.
- The runtime ID materializer (§40) returns the branded value without runtime checking.

---

# PART IV — Worked examples

## 46. Resident detail (end-to-end)

The full surface — Resource + Route + Page + Sections + Actions + Island — in C#:

```csharp
namespace App.Features.Residents;

// ----- Resource -----
[Resource("Resident")]
public sealed partial record ResidentResource : AppResource<ResidentResource.Identity>
{
    public required Identity      Id          { get; init; }
    public required Data          Resident    { get; init; }
    public required Links         Links       { get; init; }
    public required Actions       Actions     { get; init; }
    public required Permissions   Permissions { get; init; }

    public sealed record Identity(Guid Value) : IResourceIdentity;
    public sealed record Data    { public required string FirstName { get; init; } public required string LastName { get; init; } }
    public sealed record Links   : IAppLinks    { public required RouteRef<ResidentDetailsRoute.Params> Details { get; init; } public required RouteRef<ResidentEditRoute.Params> Edit { get; init; } }
    public sealed record Actions : IAppActions  { public required ActionRef<SaveResident> Save { get; init; } public required ActionRef<ArchiveResident> Archive { get; init; } }
    public sealed record Permissions : IAppPermissions { [Capability("canEdit")] public required bool CanEdit { get; init; } [Capability("canArchive")] public required bool CanArchive { get; init; } }
}

// ----- Route -----
[Route("/residents/{id:guid}")]
public sealed partial class ResidentDetailsRoute : AppRoute<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public sealed record Params { [FromRoute] public required Guid Id { get; init; } }
    public sealed record Props
    {
        public required ResidentResource Resident { get; init; }
        public required Sections         Sections{ get; init; }
    }
    public sealed record Sections : IAppSections { public required SectionRef<ResidentActivitySection.Props> Activity { get; init; } }
}

// ----- Page -----
public sealed class ResidentDetailsPage : AppPage<ResidentDetailsRoute.Params, ResidentDetailsRoute.Props>
{
    public override async Task<AppView<ResidentDetailsRoute.Props>> Render(
        ResidentDetailsRoute.Params p, AppPageContext ctx)
    {
        var resident = await ctx.Resources.Project<ResidentResource>(new ResidentResource.Identity(p.Id));
        return App.Render(new ResidentDetailsRoute.Props
        {
            Resident = resident,
            Sections = new() { Activity = ctx.Sections.Ref<ResidentActivitySection>(new() { ResidentId = p.Id }) },
        })
        .WithLayout<TenantShellLayout>()
        .Stream(x => x.Sections.Activity);
    }
}

// ----- Section -----
[Section("ResidentActivity")]
public sealed partial class ResidentActivitySection : AppSection<ResidentActivitySection.Params, ResidentActivitySection.Props>
{
    public sealed record Params { public required Guid ResidentId { get; init; } }
    public sealed record Props  { public required IReadOnlyList<ActivityItem> Items { get; init; } }
    public sealed class Handler : AppSectionHandler<Params, Props>
    {
        public override async Task<Props> Render(Params p, AppSectionContext ctx)
            => new() { Items = await ctx.Services.GetRequiredService<IActivityRepository>().For(p.ResidentId) };
    }
}

// ----- Action -----
[Action("residents.save"), Idempotent(IdempotencyMode.ClientKey), Policy("EditResident")]
public sealed partial class SaveResident : AppAction<SaveResident.Request, SaveResident.Response>
{
    public sealed record Request  { public required Guid Id { get; init; } public required string FirstName { get; init; } public required string LastName { get; init; } }
    public sealed record Response { public required ResidentResource Resident { get; init; } }
    public sealed class Validator : AppValidator<Request> { public Validator() { RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100); RuleFor(x => x.LastName).NotEmpty().MaximumLength(100); } }
    [Optimistic] public sealed class Optimism : AppOptimism<Request, ResidentResource>
    {
        public override ResidentResource Project(ResidentResource cur, Request r)
            => cur with { Resident = cur.Resident with { FirstName = r.FirstName, LastName = r.LastName } };
    }
    public sealed class Handler : AppActionHandler<Request, Response>
    {
        public override async Task<AppActionResult<Response>> Handle(Request r, AppActionContext ctx)
        {
            var saved = await ctx.Services.GetRequiredService<IResidentService>().Save(r, ctx.Cancellation);
            ctx.Invalidate.Section<ResidentActivitySection>(new() { ResidentId = r.Id }).Resource<ResidentResource>(new ResidentResource.Identity(r.Id));
            return App.Ok(new Response { Resident = saved }, ctx.Invalidate.Build().ToArray());
        }
    }
}

// ----- Island -----
[Island("ResidentScheduler"), Renderer("vue"), Chunk("scheduler")]
public sealed class ResidentSchedulerIsland : AppIsland<ResidentSchedulerIsland.Props>
{
    public sealed record Props
    {
        public required ResourceRef<ResidentResource>     Resident      { get; init; }
        public required ActionRef<RescheduleAppointment>  Reschedule    { get; init; }
    }
}
```

### Wire (envelope)

See §26 in v0.2 — the example here matches.

### Generated TypeScript usage

```ts
await routes.residentDetails.navigate({ id });
const { resident, sections } = useProps<ResidentDetailsProps>();
resident.links.edit.navigate();
await resident.actions.save.execute({ id: resident.id.value, firstName: "Augusta", lastName: "Lovelace" });
sections.activity.reload();
```

---

## 47. Resident list (pagination)

```csharp
[Route("/residents")]
public sealed partial class ResidentsListRoute : AppRoute<ResidentsListRoute.Params, ResidentsListRoute.Props>
{
    public sealed record Params { [FromQuery("cursor")] public string? Cursor { get; init; } [FromQuery("q")] public string? Query { get; init; } }
    public sealed record Props  { public required ResidentsListResource List { get; init; } }
}

public sealed class ResidentsListPage : AppPage<ResidentsListRoute.Params, ResidentsListRoute.Props>
{
    public override async Task<AppView<ResidentsListRoute.Props>> Render(ResidentsListRoute.Params p, AppPageContext ctx)
    {
        var list = await ctx.Resources.Project<ResidentsListResource>(new ResidentsListResource.Identity { Scope = ctx.Tenant.Id, Filter = p.Query ?? "" });
        return App.Render(new ResidentsListRoute.Props { List = list }).WithLayout<TenantShellLayout>();
    }
}
```

Wire (with deduplication): see §10.

Generated TS:

```ts
const { list } = useProps<ResidentsListProps>();
list.items.forEach(r => render(r));
if (list.next) await list.next.next.prefetch();
await list.actions.create.execute({ /* ... */ });
```

---

## 48. Resident photo upload (multipart)

See §21 in full. Client usage:

```ts
const result = await resident.actions.uploadPhoto.execute({ id: resident.id, photo: file });
if ("result" in result) updateAvatar(result.result.avatar);
else                    showError(result.error);
```

---

## 49. Resident export (long-running)

See §22 in full. Client usage:

```ts
const result = await list.actions.export.execute({ format: "csv", filter: "active" });
if ("result" in result) {
  result.result.operation.subscribe(s => setProgress(s.progress));
  const final = await result.result.operation.wait();
  downloadMedia(final.file);
}
```

---

# PART V — Type generation mapping table

## 50. C# → TypeScript canonical mapping

| C# construct | Generated TS |
|--------------|--------------|
| `AppRoute<TParams, TProps>` | `routes.<camel(name)>` with `navigate/href/prefetch/match`; `TParams` & `TProps` as interfaces. |
| `AppAction<TReq, TResp>` with id `x.y` | `actions.x.y` with `execute/optimistic`. |
| `AppResource<TIdentity>` | `<Name>Resource` interface; live `links/actions/permissions` substructures. |
| `AppSection<TParams, TProps>` | `sections.<camel(name)>` with `reload/invalidate/subscribe/prefetch`. |
| `RouteRef<TParams>` | `RouteHandle<TParams>` |
| `ActionRef<TAction>` | `ActionHandle<TReq, TResp>` |
| `SectionRef<TProps>` | `SectionHandle<TProps>` |
| `ResourceRef<TResource>` | lazy resource proxy |
| `IslandRef` | entry in `islands` manifest |
| `OperationRef` | `OperationHandle<TResp>` |
| `FormRef<TReq>` | `FormHandle<TReq>` |
| `MediaRef` | `MediaHandle` |
| `record` | `interface` |
| `required` member | non-optional TS field |
| nullable reference / `T?` | optional / `T \| null` (canonical: `?` in Params query, `\| null` in body) |
| `Guid`, `DateOnly`, `DateTimeOffset`, `TimeSpan`, `decimal`, `long` | `string` (branded where Identity) |
| `enum` | string literal union |
| `IResourceIdentity` implementor | branded type (§45) |
| `[Capability]` `bool` | named permission flag in `Permissions` |
| `AppValidator<T>` | entry in `validators` metadata tree |
| `IAppLocalizationKeys` | nested `keys` tree |
| `[PolymorphicResponse]` | discriminated union with `$type` |
| `[MultipartRequest]` | multipart-aware `execute` (auto Blob handling) |
| `[LongRunning]` | `execute` returns `{ result: { operation: OperationHandle<TResp> } }` |

### Determinism rules
- Same source → byte-identical generated TS.
- Map ordering: alphabetical by id within each map.
- Diff-stability: adding a route MUST NOT reorder existing entries.

---

# PART VI — Diagnostics

## 51. ALIS00xx diagnostics

| Code | Rule |
|------|------|
| `ALIS0001` | Route Props is degenerate (data only, no capability surface). |
| `ALIS0002` | URL string literal outside a `[Route]` attribute. |
| `ALIS0003` | Action without `[Action(id)]` and without convention-derivable id. |
| `ALIS0004` | Resource without `Identity` nested record implementing `IResourceIdentity`. |
| `ALIS0005` | `[FromBody]` on a Params record. |
| `ALIS0006` | Validator referencing a request type not nested in its Action. |
| `ALIS0007` | Island missing `[Renderer]` when more than one renderer is registered. |
| `ALIS0008` | Section's Params not derivable from parent page's Params and Resource ids. |
| `ALIS0009` | Action returning a raw HTTP response from a handler. |
| `ALIS0010` | Page constructing a typed ref outside `ctx.Refs.*` / `ctx.Sections.*` / `ctx.Resources.*`. |
| `ALIS0011` | Asset version mismatch between embedded graph manifest and Vite chunk manifest. |
| `ALIS0012` | Capability flag referenced in props without a corresponding `[Policy]` declaration. |
| `ALIS0013` | Two routes resolving to the same canonical pattern. |
| `ALIS0014` | Component identity collision after attribute overrides. |
| `ALIS0015` | A `ResourceRef` projected without permissions when the resource declares any. |
| `ALIS0016` | A nested `Validator` declared on a type that is not an `AppAction`. |
| `ALIS0017` | A `partial` augmentation member collides with a user-declared member. |
| `ALIS0018` | A streamed section selector that does not resolve to a `SectionRef`. |
| `ALIS0019` | A `Cache(...)` policy on a route whose action handler declares `Vary` incompatible with caching. |
| `ALIS0020` | A bundle record (`IAppLinks` / `IAppActions` / `IAppSections` / `IAppPermissions`) containing refs of the wrong kind. |
| `ALIS0021` | A `[MultipartRequest]` request without exactly one `FileUpload`-typed property. |
| `ALIS0022` | A `[LongRunning]` action without a typed `Response`. |
| `ALIS0023` | A page or island pinned to an unregistered renderer. |
| `ALIS0024` | A `[Policy]` referencing a policy name not registered with `AddPolicy<>`. |
| `ALIS0025` | A `[PolymorphicResponse]` base with case-name collision. |
| `ALIS0026` | A predicate inside `When`/`Unless` that does not parse to the canonical predicate language and is not marked `custom`. |
| `ALIS0027` | A localization key referenced in props but not declared in any `IAppLocalizationKeys` static class. |
| `ALIS0028` | A `ResourceRef` whose declared `TResource` does not implement `IAppResource`. |
| `ALIS0029` | A `RouteRef`/`ActionRef`/`SectionRef` whose params type does not match the declared route/section. |
| `ALIS0030` | Component identity has no resolvable TS source file. |
| `ALIS0031` | Circular chunk import detected. |
| `ALIS0032` | A `[LongRunning]` action used inside a streamed section without explicit `OperationRef` projection. |
| `ALIS0033` | A `[Tenant]`-scoped route used without an `AddTenants(...)` registration. |
| `ALIS0034` | A `FormRef` whose action `Request` type cannot be defaulted (missing `new()` constraint). |
| `ALIS0035` | A renderer registered twice with the same id. |

Every diagnostic has a runbook at `/docs/spec/runbooks/<code>.md`.

---

# PART VII — Invariants

## 52. The spine that must hold

These ten invariants are the framework's load-bearing properties. If a change to the implementation would break one, it requires an ADR.

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
