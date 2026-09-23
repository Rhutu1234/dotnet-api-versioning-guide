# API Versioning

*A deep-dive walkthrough of API versioning — covering what actually constitutes a breaking vs. non-breaking change, the four major versioning strategies (URL path, query string, header, media type) in depth with their genuine trade-offs, implementing versioning in ASP.NET Core, deprecation as a first-class, communicated process rather than a silent removal, backward and forward compatibility as distinct goals, and the case for never needing to break a client at all if a change is designed carefully enough.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Versioning Exists: The Core Tension](#1-why-versioning-exists-the-core-tension)
3. [Breaking vs. Non-Breaking Changes, Defined Precisely](#2-breaking-vs-non-breaking-changes-defined-precisely)
4. [The Tolerant Reader Pattern: Avoiding Some Breaks Entirely](#3-the-tolerant-reader-pattern-avoiding-some-breaks-entirely)
5. [Strategy 1: URL Path Versioning](#4-strategy-1-url-path-versioning)
6. [Strategy 2: Query String Versioning](#5-strategy-2-query-string-versioning)
7. [Strategy 3: Header Versioning](#6-strategy-3-header-versioning)
8. [Strategy 4: Media Type Versioning](#7-strategy-4-media-type-versioning)
9. [Comparing the Four Strategies Directly](#8-comparing-the-four-strategies-directly)
10. [Implementing Versioning in ASP.NET Core](#9-implementing-versioning-in-aspnet-core)
11. [Versioning at the Right Granularity](#10-versioning-at-the-right-granularity)
12. [Deprecation: A Process, Not an Event](#11-deprecation-a-process-not-an-event)
13. [Semantic Versioning vs. API Versioning: Related, Not the Same](#12-semantic-versioning-vs-api-versioning-related-not-the-same)
14. [Versioning the Data Contract, Not Just the Route](#13-versioning-the-data-contract-not-just-the-route)
15. [Common Pitfalls](#14-common-pitfalls)
16. [Quick Reference Table](#quick-reference-table)
17. [Conclusion](#conclusion)

---

## Introduction

An API is a contract, and every contract eventually needs to change — but unlike code you control entirely within one deployment, an API's consumers are often outside your control, on their own release schedules, sometimes maintained by teams or companies you don't even know exist. This series' REST guide's Section 14 introduces the major versioning strategies briefly; this guide goes deep on the actual engineering discipline underneath versioning — precisely what counts as a breaking change (and, just as importantly, what doesn't, even though it might feel risky), how to implement each strategy concretely, and why a thoughtful deprecation process matters just as much as the versioning scheme itself, since a version number alone doesn't protect a client that never finds out an old version is going away.

```plaintext
GET /api/v1/products  → the OLD contract, still honored for existing clients
GET /api/v2/products  → the NEW contract, for clients that have migrated

Both can run SIMULTANEOUSLY, on the same server, for as long as v1
  still has real consumers — versioning exists specifically to make
  THAT coexistence possible, safely.
```

---

## 1. Why Versioning Exists: The Core Tension

### You cannot control when, or whether, every consumer updates

```plaintext
A mobile app's users update on THEIR OWN schedule, sometimes never at
  all — a partner integration might be maintained by a team that
  updates once a year — an internal service might be one your own
  organization hasn't gotten around to migrating yet. Every one of these
  is a CLIENT depending on your API's CURRENT contract, and "just tell
  everyone to update" is rarely a realistic, immediate option.
```

This is the fundamental, unavoidable reality versioning exists to manage — an API server is, from a deployment perspective, entirely within your control; the population of things calling it, generally, is not, and any change that breaks that population without warning is a real, often costly failure, not a hypothetical risk.

### The goal isn't "never change the API" — it's "never change it out from under someone without their knowledge and consent"

```plaintext
Versioning isn't about FREEZING an API forever — it's about giving
  consumers a genuine choice about WHEN to adopt a breaking change,
  rather than having it forced on them by a deployment they didn't
  even know was happening.
```

This framing matters because it clarifies what versioning is actually solving — it's not primarily a technical problem (multiple code paths running simultaneously is easy enough to implement, per Section 9) so much as a *trust and coordination* problem between an API provider and its consumers.

---

## 2. Breaking vs. Non-Breaking Changes, Defined Precisely

### The precise test: does an EXISTING, well-behaved client's code stop working correctly?

```plaintext
"Well-behaved" matters here — a client that made genuinely unreasonable
  assumptions about the contract (relying on undocumented field ORDER
  in a JSON object, say) breaking isn't the API's fault in the same way
  — but a client relying on anything the contract actually, reasonably
  promised is the bar this test applies to.
```

### Changes that ARE breaking

```plaintext
- Removing a field a client might be reading
- Renaming a field (functionally the same as removing the old name)
- Changing a field's DATA TYPE (a string that used to be a number)
- Changing a field's MEANING without changing its name or type
  (e.g., "total" used to be pre-tax, now it's post-tax)
- Adding a NEW REQUIRED field to a REQUEST body (an existing client's
  requests, which don't include it, now fail validation)
- Changing a URL's path or an endpoint's HTTP method
- Changing error response STRUCTURE a client might be parsing
- Tightening validation rules that previously-valid requests now fail
```

### Changes that are NOT breaking (generally safe to ship without a new version)

```plaintext
- Adding a NEW, OPTIONAL field to a RESPONSE body (a well-behaved
  client ignores fields it doesn't recognize — Section 3 covers exactly
  why this assumption is worth designing FOR, not just hoping for)
- Adding a NEW, OPTIONAL field to a REQUEST body, with a sensible default
  if omitted
- Adding an entirely NEW endpoint
- Adding a new, ADDITIONAL value to an enum-like field, PROVIDED clients
  are expected to handle unknown values gracefully (a real, nontrivial
  caveat worth designing for explicitly)
- Relaxing a previously-strict validation rule (something that used to
  be rejected is now accepted)
- Performance improvements, bug fixes that bring behavior in line with
  the DOCUMENTED contract (arguably these were always "broken" from the
  contract's perspective, even if some client had come to depend on the
  buggy behavior — a genuinely nuanced case worth its own judgment call)
```

This list is worth treating as the practical, working definition underneath every strategy this guide covers — the entire discipline of API evolution is, in large part, the discipline of maximizing how much falls into the second list and minimizing how often you're forced into the first.

---

## 3. The Tolerant Reader Pattern: Avoiding Some Breaks Entirely

### Designing clients (and encouraging consumers) to ignore what they don't recognize

```csharp
// A client deserializing a response should NOT fail if an UNEXPECTED field appears —
// most JSON deserializers do this correctly by DEFAULT, but it's worth confirming, not assuming
public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    // if the server later adds a "category" field, THIS client simply ignores it —
    // no exception, no failure, as long as the deserializer isn't configured to reject unknown fields
}
```

This is the client-side half of Section 2's "adding a field isn't breaking" claim — it's only genuinely non-breaking if clients are actually built to tolerate additions, which isn't automatic in every language/framework/configuration (some strict deserializers reject unrecognized fields by default) — worth explicitly confirming, and documenting as an expectation for your API's consumers, rather than assuming it.

### Tolerant readers for enums specifically: designing for a value you haven't invented yet

```csharp
public enum OrderStatus { Pending, Shipped, Delivered, Unknown = -1 } // an explicit fallback

// Client-side deserialization logic maps any UNRECOGNIZED string value to Unknown,
// rather than throwing — letting the client keep functioning (perhaps degraded) when
// the server introduces a NEW status the client wasn't built to know about yet
```

This is a genuinely valuable, if easy-to-overlook pattern — a client that throws on an unrecognized enum value turns "the server added a legitimate new status" into a breaking change for that client, purely because of how the client happened to be written; designing an explicit "unknown/unhandled" fallback is what actually makes Section 2's "adding an enum value is non-breaking" claim true in practice, not just in principle.

---

## 4. Strategy 1: URL Path Versioning

### The version is embedded directly, visibly, in the URL itself

```plaintext
GET /api/v1/products/42
GET /api/v2/products/42
```

```csharp
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV1Controller : ControllerBase { /* ... */ }

[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase { /* ... */ }
```

This is, by a wide margin, the most common versioning strategy in real-world practice — its dominant advantage is *visibility*: anyone reading a URL, a log entry, or a piece of documentation immediately sees which version they're dealing with, with zero additional context needed.

### The genuine cost: it treats "version" as if it were part of the resource's identity

```plaintext
Per this series' REST guide's Section 2: a URL is supposed to identify
  a RESOURCE — /products/42 conceptually names "product 42," a single,
  persistent thing. /v1/products/42 and /v2/products/42 arguably name
  TWO DIFFERENT URLs for what's really the SAME underlying resource,
  just represented differently — a genuine, if largely theoretical,
  tension with REST's own resource-identity philosophy.
```

This is worth knowing as the real, if often practically unimportant, philosophical cost of this otherwise overwhelmingly convenient approach — most real-world API teams accept this trade-off deliberately, valuing visibility and simplicity over strict adherence to REST's resource-identity principle.

---

## 5. Strategy 2: Query String Versioning

### The version travels as a query parameter, alongside the URL rather than embedded within its path

```plaintext
GET /api/products/42?api-version=1.0
GET /api/products/42?api-version=2.0
```

```csharp
[ApiVersion("1.0")]
[Route("api/products")]
public class ProductsController : ControllerBase { /* ... */ } // the SAME route works for BOTH versions
```

This keeps the URL's *path* stable — arguably a cleaner separation of "which resource" (the path) from "which contract version" (the query parameter) than URL path versioning provides — while still remaining highly visible and easy to test manually (just append `?api-version=2.0` to any request).

### The genuine cost: query strings are less semantically "sticky" and more easily dropped or lost

```plaintext
Query parameters are more easily stripped by intermediate caching
  layers, accidentally omitted by developers copying a URL without its
  full query string, or considered "optional-feeling" in a way a URL
  PATH segment isn't — a real, if largely practical rather than
  theoretical, downside worth weighing against this strategy's cleaner path structure.
```

---

## 6. Strategy 3: Header Versioning

### The version travels in a custom HTTP header, entirely separate from the URL

```plaintext
GET /api/products/42
X-Api-Version: 2.0
```

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new HeaderApiVersionReader("X-Api-Version");
});
```

This keeps the URL entirely clean and stable across every version — the same URL, `/api/products/42`, serves every version of the contract, with the header alone determining which one a specific request receives — a genuinely appealing property for teams that want URLs to be pure, permanent resource identifiers, fully separate from any versioning concern.

### The genuine cost: much lower visibility, and genuinely harder to test/explore manually

```plaintext
A URL alone, pasted into a browser or shared in a bug report, no longer
  tells you which version was actually being used — you need the
  request's HEADERS too, which are invisible in a browser address bar
  and easy to forget when manually testing with tools like curl or
  Postman unless you're deliberately including them every time.
```

---

## 7. Strategy 4: Media Type Versioning

### The version is expressed through content negotiation itself — the `Accept` header names a versioned media type

```plaintext
GET /api/products/42
Accept: application/vnd.myapi.v2+json
```

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new MediaTypeApiVersionReader("v");
});
```

This is, per this series' REST guide's Section 13 content negotiation discussion, arguably the most "correct" approach by REST's own philosophy — Fielding's model treats a resource's *representation format* (what content negotiation is fundamentally about) and its *version* as genuinely the same kind of concern: both describe "which specific shape of data do you want for this resource," which is precisely what the `Accept` header already exists to negotiate.

### The genuine cost: the least common, least immediately intuitive approach for most developers

```plaintext
Vendor-specific media types (application/vnd.COMPANYNAME.vN+json) are
  unfamiliar to many developers encountering an API for the first time,
  compared to an obviously version-numbered URL — this strategy's
  philosophical correctness comes at a real cost in DISCOVERABILITY and
  ease of first-time use, echoing this series' REST guide's Section 11
  HATEOAS discussion's own correctness-vs-practicality trade-off.
```

---

## 8. Comparing the Four Strategies Directly

### A side-by-side summary of the genuine trade-offs

```plaintext
                  Visibility   URL Stability   REST-Purity   Ease of Manual Testing
URL Path           Highest      Lowest          Lower          Highest
Query String        High         Medium          Medium         High
Header               Low          Highest         Higher         Lower (headers needed)
Media Type            Lowest       Highest         Highest        Lowest
```

### Why URL path versioning remains the dominant, pragmatic default despite not "winning" on REST-purity

```plaintext
Per this series' REST guide's Section 12 Richardson Maturity Model
  discussion: most real-world APIs deliberately trade some theoretical
  REST purity for practical developer experience — URL path versioning's
  visibility and ease of use are worth more, in practice, to the vast
  majority of API consumers than media-type versioning's philosophical
  correctness, which is exactly why it's the strategy you'll encounter
  most often "in the wild," including from major API providers.
```

This mirrors the honest trade-off framing this series' REST guide applies to HATEOAS directly — there's no single objectively "correct" strategy; there's a genuine, deliberate trade-off between visibility/simplicity and strict resource-identity purity, and different teams reasonably land in different places depending on who their actual API consumers are and how they'll interact with the API.

---

## 9. Implementing Versioning in ASP.NET Core

### The `Asp.Versioning` package: the standard, maintained library for this

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true; // requests with NO version specified get v1
    options.ReportApiVersions = true; // adds an api-supported-versions RESPONSE header, listing what's available
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV"; // integrates with Swagger/OpenAPI documentation per version
});
```

This is the current, actively maintained library for API versioning in ASP.NET Core (the earlier `Microsoft.AspNetCore.Mvc.Versioning` package is deprecated in favor of it) — `ReportApiVersions` is a genuinely useful, easy-to-enable detail worth highlighting: it tells CLIENTS, via a response header, exactly which versions the server currently supports, which is directly useful for Section 11's deprecation communication.

### Deprecating a specific version explicitly, while it's still supported

```csharp
[ApiVersion("1.0", Deprecated = true)] // still WORKS, but marked deprecated — surfaces in Swagger docs
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsController : ControllerBase { /* ... */ }
```

Marking a version `Deprecated = true` doesn't disable it — it continues functioning exactly as before, but this metadata is surfaced through the API's documentation tooling and the `ReportApiVersions` response header, giving consumers a genuine, visible signal that this version's days are numbered, well before Section 11's actual removal.

### Routing multiple versions from the SAME controller, when the difference is small

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/products")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1(int id) { /* the OLD shape */ return Ok(); }

    [HttpGet("{id}")]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2(int id) { /* the NEW shape */ return Ok(); }
}
```

For a small, contained difference between versions, keeping both action methods in the same controller (distinguished by `[MapToApiVersion]`) is often cleaner than Section 10's full-controller-duplication approach — worth choosing deliberately based on how much genuinely differs between the two versions, which is precisely Section 10's subject.

---

## 10. Versioning at the Right Granularity

### Whole-API versioning: bump EVERY endpoint's version together, even for a change touching just one

```plaintext
A change to a SINGLE endpoint (/products) triggers a new version number
  for the ENTIRE API (v1 → v2), even though every OTHER endpoint
  (/orders, /customers) is completely unaffected.
```

This is simpler to communicate and reason about ("v2 of the API" is one clear, singular concept) but genuinely coarser than necessary — a consumer only using `/orders` still has to care about a version bump that was entirely about `/products`, and potentially needs to migrate endpoints they never actually changed anything about.

### Per-endpoint (or per-resource) versioning: only the specific, actually-changed endpoint gets a new version

```plaintext
GET /api/v1/orders    — unaffected by the products change, stays at v1
GET /api/v2/products   — the ONLY endpoint that actually changed
```

This is more precise and reduces unnecessary migration churn for consumers of unrelated endpoints, but genuinely more complex to track, document, and communicate — "what version is the API at" no longer has one single, clean answer; it depends on which specific resource you're asking about.

### Which granularity to choose: a real, deliberate trade-off, not a universal answer

```plaintext
Smaller, more numerous APIs (a handful of tightly-related endpoints)
  often favor WHOLE-API versioning for its simplicity. Larger, more
  loosely-coupled APIs (many independent resource types, potentially
  owned by different internal teams) often favor PER-RESOURCE versioning,
  since forcing every team's endpoint to bump in lockstep with every
  OTHER team's changes becomes genuinely unworkable at scale.
```

---

## 11. Deprecation: A Process, Not an Event

### A version number alone doesn't protect anyone if consumers never find out an old one is going away

```plaintext
Per Section 1's core framing: versioning solves "give consumers a
  CHOICE about when to adopt a change" — but that choice is meaningless
  if consumers have no visibility into WHEN an old version will actually
  stop being supported. Deprecation needs its OWN explicit, communicated
  process, distinct from the versioning mechanism itself.
```

### The `Sunset` HTTP header (RFC 8594): a standardized way to signal an upcoming removal date

```http
HTTP/1.1 200 OK
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: <https://api.example.com/docs/migration-v1-to-v2>; rel="sunset"
```

```csharp
app.Use(async (context, next) =>
{
    if (context.GetRequestedApiVersion()?.ToString() == "1.0")
    {
        context.Response.Headers.Append("Sunset", "Sat, 31 Dec 2026 23:59:59 GMT");
    }
    await next(context);
});
```

The `Sunset` header is a standardized, machine-readable way to include a deprecation removal date directly in every response from a deprecated version — genuinely useful because it lets automated tooling (not just a human reading documentation) detect and alert on approaching deprecation, giving consumers a fighting chance to notice before it's a genuine emergency.

### A real deprecation timeline, worth treating as a first-class engineering deliverable

```plaintext
1. Announce: publish the deprecation, the removal date, and a migration
   guide, WELL before any actual behavior changes.
2. Warn actively: add the Sunset header (and ideally proactive outreach —
   email, dashboard notices) to every response from the deprecated version.
3. Monitor usage: track WHO is still calling the deprecated version, and
   for API keys/identifiable consumers, consider direct outreach to
   laggards as the removal date approaches.
4. Remove: only after the announced date has genuinely passed, AND
   usage has genuinely dropped to an acceptable level (or the business
   has made a deliberate decision to force the remaining consumers off).
```

This is worth treating with the same rigor as any other engineering process, not an afterthought tacked onto "we shipped v2" — the actual harm from bad versioning practice almost always comes from a poorly-communicated *removal*, not from the existence of multiple versions running simultaneously, which (per Section 9) is technically straightforward.

---

## 12. Semantic Versioning vs. API Versioning: Related, Not the Same

### Semantic Versioning (SemVer): MAJOR.MINOR.PATCH, describing a PACKAGE's compatibility promise

```plaintext
MAJOR version: incremented for BREAKING changes.
MINOR version: incremented for backward-COMPATIBLE new functionality.
PATCH version: incremented for backward-COMPATIBLE bug fixes.
```

SemVer is a convention primarily associated with versioned software *packages/libraries* (an npm package, a NuGet package) — worth knowing it exists as a related, but genuinely distinct, concept from the API versioning this entire guide covers, since the two are frequently, and understandably, confused.

### Why REST APIs typically only expose the MAJOR version, not a full SemVer number

```plaintext
Per Section 2's breaking/non-breaking distinction: a NON-breaking
  change (adding an optional field) shouldn't require ANY version bump
  at all for an already-tolerant client (Section 3) — there's no
  meaningful equivalent of SemVer's "minor" version for a live API
  endpoint the way there is for a versioned, installed PACKAGE, where
  every consumer explicitly, deliberately chooses when to pull in a new
  minor version.
```

This is worth understanding as the genuine, structural reason `/api/v2/products` (just a major version number) is the near-universal convention, rather than something like `/api/v2.3.1/products` — an API's consumers are, in the vast majority of cases, always calling whatever the *current* deployed state of a given major version is; there's no equivalent of a package manager letting them "pin" to a specific minor/patch version the way a library's consumers can.

---

## 13. Versioning the Data Contract, Not Just the Route

### The URL/header carries the version signal — but the DTOs are what actually define the contract

```csharp
namespace Api.V1
{
    public class ProductDto { public int Id { get; set; } public string Name { get; set; } = ""; public decimal Price { get; set; } }
}

namespace Api.V2
{
    public class ProductDto { public int Id { get; set; } public string Name { get; set; } = ""; public Money Price { get; set; } = new(); } // Price is now a structured object, not a plain decimal
}
```

Worth stating explicitly, since it's easy to treat "versioning" as purely a routing concern: the actual, meaningful contract a client depends on is the *shape of the data* going back and forth — maintaining genuinely separate DTO types per version (as above), rather than one shared type with awkward conditional logic trying to serve both shapes, is what keeps each version's contract honest, stable, and independently testable.

### Mapping between an internal domain model and multiple, version-specific DTOs

```csharp
public class ProductV1Mapper { public ProductDto Map(Product domain) => new() { Id = domain.Id, Name = domain.Name, Price = domain.Price.Amount }; }
public class ProductV2Mapper { public ProductDto Map(Product domain) => new() { Id = domain.Id, Name = domain.Name, Price = new Money(domain.Price.Amount, domain.Price.Currency) }; }
```

This is the practical shape versioning takes once you look past the routing layer — the underlying domain model (`Product`) stays singular and unversioned, while a distinct mapper per API version translates it into that version's specific, stable DTO shape — keeping the internal model free to evolve independently of any specific API contract's frozen shape, echoing this series' Order Management guide's own separation between an internal aggregate and its external representation.

---

## 14. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Treating any field addition as automatically safe, without confirming clients tolerate unknown fields | Some deserializers reject unrecognized fields by default, silently turning a "safe" addition into a real break for some clients | Explicitly design and document a Tolerant Reader expectation (Section 3); confirm client tooling actually behaves this way |
| Removing an old API version on the announced date, regardless of remaining usage | Consumers who missed the announcement, or whose migration slipped, experience a genuine, unannounced-feeling outage | Monitor actual usage before removal; treat the announced date as a target, not an unconditional trigger (Section 11) |
| Bumping the whole API's version for a change affecting only one endpoint | Forces unrelated consumers to care about and potentially migrate for changes that never affected them | Consider per-resource/per-endpoint versioning granularity for larger, more loosely-coupled APIs (Section 10) |
| Sharing one DTO type across multiple API versions, with conditional logic to serve both shapes | Entangles two independently-evolving contracts into one fragile, harder-to-reason-about type | Maintain genuinely separate, version-specific DTOs, mapped from a shared internal domain model (Section 13) |
| Confusing SemVer's minor/patch granularity with what a live API needs to expose | A REST API's consumers can't "pin" to a specific minor version the way a package manager's consumers can | Expose only a major version number for the API contract itself; reserve full SemVer for versioned client SDKs/packages (Section 12) |
| Choosing a versioning strategy purely on REST-purity grounds, ignoring actual consumer experience | Header/media-type versioning's theoretical correctness can come at a real cost to discoverability for typical API consumers | Weigh visibility and ease of use alongside purity (Section 8); URL path versioning remains a reasonable, common default for most audiences |
| Deprecating a version with no machine-readable signal, relying solely on documentation | Automated tooling and less-attentive consumers may never notice a deprecation notice buried in a docs page | Use the standardized `Sunset` header (Section 11) alongside documentation, so tooling can detect and alert on it |
| Assuming enum additions are automatically non-breaking | A client that throws on an unrecognized enum value turns a legitimate new value into a break for that specific client | Design clients with an explicit "unknown" fallback for enum-like fields (Section 3) |

---

## Quick Reference Table

| Strategy | Example | Key Trade-off |
|---|---|---|
| URL Path | `/api/v2/products` | Highest visibility; lowest URL stability/REST purity |
| Query String | `/api/products?api-version=2.0` | Clean path, but query strings are easily dropped |
| Header | `X-Api-Version: 2.0` | Stable, clean URLs; low visibility, harder to test manually |
| Media Type | `Accept: application/vnd.api.v2+json` | Most REST-pure; least discoverable/intuitive |
| `Sunset` header | `Sunset: Sat, 31 Dec 2026 23:59:59 GMT` | Machine-readable deprecation signal, alongside human-readable docs |
| SemVer (packages, not live APIs) | `MAJOR.MINOR.PATCH` | A related but distinct convention — live APIs typically expose only major version |

---

## Conclusion

API versioning is, at its technical core, a genuinely simple problem — running multiple, distinguishable code paths simultaneously — but the real discipline lives in two other places this guide spends the most effort on: precisely knowing what actually constitutes a breaking change (and designing, via the Tolerant Reader pattern, to minimize how often you're forced into one), and treating deprecation as a communicated, monitored process rather than a silent event that happens to coincide with a version bump. The four strategies this guide covers — URL path, query string, header, and media type — all solve the same underlying problem with genuinely different trade-offs between visibility and REST-purity, and there's no universally correct choice; URL path versioning's dominance in practice reflects a real, common preference for discoverability over theoretical resource-identity purity, exactly the kind of pragmatic trade-off this series' REST guide identifies throughout its own discussion of Richardson Maturity Levels.

The version number itself is the least interesting part of this whole discipline — what actually protects a client from a breaking change is the announcement, the migration guide, the `Sunset` header, and the monitoring that confirms it's actually safe to remove an old version, none of which the versioning mechanism provides automatically just by existing. Getting the mechanism right (Sections 4-10) is necessary but not sufficient; getting the surrounding process right (Sections 2-3, 11) is what actually keeps existing clients from breaking, which is the entire reason any of this exists in the first place.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the removed-a-deprecated-version-and-broke-a-partner-integration-nobody-remembered-existed incident that made a monitored, communicated deprecation process feel less like process overhead and more like a genuine necessity.*
