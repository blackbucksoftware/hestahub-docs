---
title: HestaHub Public Tenant API v1
---

# HestaHub Public Tenant API — v1

**Status: draft.** This contract does not bind anyone yet. It binds from the first Revision installed on a server we do not control (see [Compatibility and Revisions](#compatibility-and-revisions)). Until then, Revisions may break each other, and the [revision history](#revision-history) records it when they do.

**Current Revision:** `2026-11-23`. Its shapes are in [`v1/2026-11-23/`](2026-11-23/).

This document binds two parties: **HestaHub** (the server) and any **client** that reads a Tenant's data through this API. The first client is the HestaHub WordPress plugin. The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119, and each sentence names who it binds. Where a sentence binds only the WordPress plugin, it says "the plugin".

## How this document works

- **The prose is normative for rules. The JSON Schemas are normative for shape.** This document never lists a resource's fields. Field names, types, nullability and which keys are required live only in `v1/<revision>/<endpoint>.schema.json`. Every `<endpoint>.example.json` beside a schema is an illustration and promises nothing.
- A schema names exactly the keys we promise in `required` and allows anything else (`additionalProperties: true`). That is the [tolerant reader](#the-client-is-a-tolerant-reader) rule expressed as a schema.
- Where a rule below names a field, the schema is still what says the field exists.

## The API at a glance

v1 is read-only. It has four endpoints. Each is a `GET`, and each is served from the Tenant's own subdomain:

| Endpoint | What it serves | Schema |
|---|---|---|
| `GET /api/v1/meta` | the deployment's Revision, the Tenant's name and resources, version guidance | `meta` |
| `GET /api/v1/apartments` | every public Apartment, with gallery, Features, Blueprint and Rate structure | `apartments` |
| `GET /api/v1/availability` | blocked date ranges for every public Apartment | `availability` |
| `GET /api/v1/clinics` | every public Clinic that has not finished | `clinics` |

The error envelope used by all four is `error`.

There is no per-item endpoint, no pagination and no write method. Bookings, Guests and other Persons never leave HestaHub: availability is anonymous date ranges, and no Clinic names a Trainer or participant.

## Requests

### Host and scheme

A Tenant's API lives at `https://<tenant>.hestahub.com/api/v1/…`. The subdomain chooses the Tenant.

A client MUST call `https://` literally. Plain `http://` is redirected to HTTPS, and a client that follows that redirect has already sent its key once in cleartext. The redirect does not make that request fail, so the leak would go unnoticed.

### Authentication

Every request, `meta` included, MUST carry the Tenant's API key:

```
Authorization: Bearer <api key>
```

An API key is `hestahub_` followed by 64 lowercase hex characters. A client MUST store and send it unchanged. The alphabet is hex so that WordPress's `sanitize_text_field()` cannot corrupt it. A client MUST NOT put the key in a URL.

A key belongs to exactly one Tenant. A key that is valid for another Tenant's subdomain is refused exactly like an unknown key. A Tenant may hold several keys at once, so rotating one means minting a new key, putting it in the client, and then revoking the old one. Keys do not expire.

### Identification

Every request MUST carry a `User-Agent` naming the client, its host platform and its runtime, and no site URL. The WordPress plugin sends exactly:

```
User-Agent: HestaHub-WP/<plugin version> (WordPress <wp version>; PHP <php version>)
```

This is a requirement, not a courtesy. HestaHub stores it against the key, and it is the evidence that later decides when the plugin's PHP floor may rise. A client that omits it takes itself out of that evidence.

## Responses

All responses are `application/json; charset=utf-8`.

### The success envelope

Every success response is an object with `data`, `revision` and `cache`. It is never a bare array and never a bare object. Resources that carry money (`apartments` and `clinics`) also carry `currency` and `prices_include_tax` in the envelope, once.

- `revision` is the Revision the responding deployment implements. A cached body therefore states its own provenance.
- `cache` holds the freshness rules for this body. See [Caching](#caching).
- `currency` is an ISO 4217 code. When `prices_include_tax` is `true`, every amount in the body is gross, including VAT. A client MUST NOT assume either value.

HestaHub MAY also send `Cache-Control`, `Deprecation` and `Sunset` headers, for any caller that reads headers. Nothing in this contract depends on them, because the plugin caches bodies and discards headers.

### The error envelope

Every error response is an object with `error` (holding `code` and `message`), `revision` and `docs_url`.

- **`code` is contractual.** It is a stable machine token, and clients switch on it.
- **`message` is not contractual.** It is English, written for developers, and may be reworded in any Revision. A client MUST NOT show it to a site visitor. The plugin renders its own localised text keyed on `code`. **The wire never carries display text for errors.**
- `revision` and `docs_url` are present on every error, including every `401`. A client with a broken key still learns which Revision answered and where this document lives. Both values are identical for every failure, so they reveal nothing about why a key failed.

The v1 codes:

| `code` | Status | Condition |
|---|---|---|
| `unauthorized` | `401` | no key, an unknown key, or a key belonging to another Tenant |
| `key_revoked` | `401` | the key existed for this Tenant and was revoked |
| `tenant_inactive` | `404` | the subdomain names no active Tenant (unknown and switched-off look alike) |
| `module_disabled` | `404` | the Tenant does not have the Module this endpoint serves |
| `not_found` | `404` | no such endpoint in v1 |
| `version_gone` | `410` | v1 has been retired (see [Retirement](#retirement)) |
| `server_error` | `500` | anything HestaHub failed to do |

Adding a code for a condition that previously had no code of its own is non-breaking. A client therefore MUST handle a `code` it does not recognise. It MUST classify the response by its HTTP status, and it MAY show the raw `code` and `message` to its own administrator. It MUST NOT show them to a visitor.

A `404` for `module_disabled` means "this Tenant does not have this feature". It does not mean "this Tenant has none of these". An empty `data` list means the second.

### Values

- **Dates** are `YYYY-MM-DD` in the Tenant's local calendar.
- **Instants** are ISO 8601 date-times with offset, in the Tenant's local time: `2027-03-14T14:00:00+01:00`.
- **Amounts** are decimal strings with exactly two places (`"45.00"`), never JSON numbers, so no client ever meets a binary rounding error in a price. A client MUST NOT do arithmetic on an amount. Every amount it needs is already computed.
- **Asset URLs** (photos, thumbnails, Blueprints) are absolute and **opaque**. A client MUST NOT parse one, construct one, or derive one from another by pattern. The URL layout is not uniform and may change in any Revision. Ceasing to return an absolute, publicly fetchable URL is breaking. **A URL resolves at the moment we return it. Nothing is promised about how long it keeps resolving.** HestaHub keeps a replaced image's old file for 30 days, so a URL in a cached body normally outlives the cache.
- **Slugs** are opaque public identifiers minted by HestaHub. A client MUST use a slug exactly as received and MUST NOT derive one from a name. An admin can change a slug, with a warning that it is a public address. A client therefore treats a slug it no longer receives as absent, never as renamed.
- **Keys** (a Building's `key`, a Season's `key`) are stable identifiers that never change once minted. Names may change at any time. **A client MUST group, match and compare on keys and print names.** It MUST NOT match on a name.
- **`null` is explicit.** A nullable field is present with `null`, never omitted. Where a schema allows `null`, a client MUST render the absence (for example, omit the "m²" token). It MUST NOT print `0` or an empty unit.
- **Enum-like fields may gain values in any Revision.** Each rule below states what a client does with a value it does not recognise.
- **Order is meaningful wherever this document states it.** A client MUST render lists in the order received and MUST NOT re-sort them.

## Resources

### `GET /api/v1/meta`

The diagnostic channel, and the one endpoint guaranteed to answer for the whole life of v1. It reports:

- the Version and its status
- the deployed Revision
- the sunset date, if any
- the resolved Tenant's `name`
- the Tenant's `resources`
- the endpoints v1 currently serves
- anything deprecated
- `minimum_plugin_version` and `recommended_plugin_version`
- `docs_url`

Rules:

- **`tenant.name` is how a client proves its key belongs to the right Tenant.** A `200` proves only that the key authenticates. A settings screen that tests a connection SHOULD show this name.
- **`resources` states which Modules the Tenant has.** It names `apartments`, `clinics`, or both. A client MUST serve a resource's routes only while `resources` lists it. Otherwise "no Module" and "nothing listed" look identical.
- `meta` requires the key like every other endpoint.
- A client MUST check `meta` at least once every 24 hours.
- **`meta` is additive-only forever, not merely within v1.** It must stay readable by the oldest client that ever shipped.

### `GET /api/v1/apartments`

Every Apartment the Tenant has switched on, in one list, sorted by `name`. Each Apartment carries everything both the listing page and its own page need: description, Building, size, occupancy, Features, gallery, Blueprint, Rate structure and portal link. A client SHOULD render both pages from the same cached copy. Two copies expire separately and can show two different prices.

Rules:

- **An Apartment that is switched off or deleted is absent, not flagged.** A client MUST treat an Apartment that has disappeared from a successful response as not found. The plugin answers `404`, never `410`: a sold Apartment and one closed for the winter look identical, and `410` permanently removes a page from search indexes.
- **Building.** An Apartment's `building` is `null` or an object with a stable `key`, a display `name` and a `position`. `position` is the Tenant's own order of its Buildings, which cannot be derived from the Apartments. A client groups on `key`, orders groups by `position`, and prints `name`.
- **`qm` and `max_persons` may be `null`.** Occupancy filters SHOULD treat a `null` `max_persons` as matching every filter value, so that a missing number never hides an Apartment.
- **Features** are free-text lines in the Tenant's order. A client SHOULD render them as a generic list. A new Feature then appears without any change to the client, and that is intended.
- **Gallery.** Images come in the Tenant's order, with exactly one or zero flagged primary. Each image MAY carry a `caption`. When a caption is rendered next to its image, the image's `alt` SHOULD be empty (`<figure><img alt=""><figcaption>`), so that a screen reader does not announce it twice. Only originals are served. Size them in CSS.
- **Blueprint** is its own field (`null` or one image URL), not a member of the gallery.
- **`portal_url`** is `null` or an absolute URL of the Apartment's page on the HestaHub customer portal. It is `null` for every Apartment and Clinic while the Tenant has its Portal links setting off. A client SHOULD render a booking link only when it is set. A client MUST NOT add dates or parameters to it. It names a page and nothing else.

#### The Rate structure

`pricing` is the Apartment's Rate structure: what a stay costs, published to be **printed, never calculated from**. Every amount in it is final. A client that multiplies, sums or chooses between rates is building the pricing engine this API exists not to need.

- `pricing.seasons` lists the Seasons for which the Apartment has at least one public LOS tier or Bundle. The default Season comes first and is flagged `is_default`. The others follow in order of their earliest listed Period.
- A Season's `periods` are its **inclusive** date ranges (`first_day`–`last_day`). Periods that ended before today are left out. The default Season has no Periods: it covers every date no other Season claims. The field names differ from availability's `from`/`to` on purpose. These ranges are inclusive and availability's are half-open, and a reader must not confuse them.
- A Season's `tiers` are its public LOS tiers in the Tenant's order. That order is meaningful: where tiers overlap, the first one listed prices the stay. A tier has a `name` or `null`, `min_nights`, and `max_nights`, where `null` means open-ended ("12+"). Without a name, a client labels a tier from its Season name and night range, in its own words.
- A Season's `bundles` are its public Bundles in the Tenant's order. Each has a `name`, an exact number of `nights`, a flat `total_price` and an optional `description` stating its terms ("Freitag, 15 Uhr, bis Sonntag, 17 Uhr").
- **Every tier and every Bundle carries `final_cleaning_fee`** (Endreinigung), always present. `"0.00"` means none, and a client MUST then leave the line out, never print "0,00 €". What the fee means: it is charged **once per stay**, on top of what the nights cost. A stay priced by tiers takes the fee of the tier that prices its **last** night. A stay priced by a Bundle takes the Bundle's fee, added to the Bundle's total. A client states this; it never applies it.
- **The listing's per-Apartment Endreinigung column.** A client that shows one cleaning figure per Apartment MUST derive it from the distinct non-zero `final_cleaning_fee` values across that Apartment's tiers and Bundles. It prints the single value when they agree, a range ("45,00–60,00 €") when they differ, and nothing when there are none. Comparing values is not pricing.
- Not published: a "from" price, any quote for specific dates, a Dog fee, a Single traveller discount, In-house prices, Check-in and Check-out times, Arrival days. Charges and discounts that depend on who travels or what they bring stay in the Tenant's own words on the Tenant's own site.

### `GET /api/v1/availability`

When each public Apartment is blocked, from today forward. This endpoint is separate from `apartments` because it goes stale on a different clock.

Rules:

- **The payload states the bounds of its own knowledge.** `data.from` and `data.to` are the half-open window the ranges describe: today to twelve months ahead. Outside that window nothing is known. A client MUST NOT render a date outside the window as free. Render it as unknown, or don't render it.
- `data.apartments` lists every Apartment that `apartments` would list, each by `slug`, with its `blocked` ranges. An Apartment with no blocked ranges is present with an empty list.
- **Ranges are half-open: `from` is the first blocked night, `to` is the morning the stay ends.** `{from: 2027-07-03, to: 2027-07-10}` blocks the nights of 3 to 9 July. 10 July is free to arrive on. Changeover days (one stay leaves and another arrives) are why the ranges are half-open. With inclusive ranges, every changeover Saturday would vanish.
- One range per blocking stay, sorted by `from`, **never merged**. Two adjacent ranges (`to` of one equals `from` of the next) are a changeover, not one long stay. A client derives the four day states from adjacent ranges: booked, arrival, departure, changeover. Nothing else on the wire says which Booking, of what kind, for whom, or for how long.
- A range may start before `data.from` when a stay is already under way. A client clips it.

### `GET /api/v1/clinics`

Every Clinic the Tenant has switched on that has a start and has not yet finished, sorted by `starts_at`, ties by `name`. That order is contractual. A Clinic leaves the list once it has finished, so a vanished Clinic slug is an everyday event, not an error.

Rules:

- `starts_at` and `ends_at` are **instants**, not dates, so they have no inclusive or exclusive end. `ends_at` may be `null` (no set end).
- `min_participants` is `null` when the Tenant set none. It is a guide, not a promise that the Clinic runs.
- `horses` is one of `own_only`, `own_or_loan` or `none`. A client that does not recognise a value prints nothing about horses.
- `fee` is per rider and `loan_horse_fee` per loan horse. Both are regular prices, gross under the envelope's `currency` and `prices_include_tax`. `loan_horse_fee` is `null` unless the Clinic takes loan horses. In-house prices are not published.
- **`signup` is a state, never a count.** It is one of `not_yet_open`, `open`, `waitlist`, `full` or `closed`. HestaHub computes it from the signup window, the Waitlist and the places taken. A Clinic that has begun may still be `open`. A client that does not recognise a value treats it as `closed` for any call to action and prints no state. Places left and the maximum are not published.
- `signup_opens_at` is `null` when signups have no set opening. `signup_closes_at` is the **effective** close, already resolved. A client never computes it.
- `apartment_guests_only` says that only people staying in one of the Tenant's Apartments may sign up.
- `portal_url` follows the Apartment rule above.

## Caching

**The server owns the clock.** Every success body carries `cache.ttl` (soft expiry, in seconds) and `cache.stale_ttl` (hard expiry, in seconds). A client MUST take both from the body, MUST NOT substitute its own numbers, and MUST NOT offer a setting to override them. HestaHub may change them in any Revision. Today they are:

| Endpoint | `ttl` | `stale_ttl` |
|---|---|---|
| `meta` | 12 hours | 7 days |
| `apartments`, `clinics` | 6 hours | 7 days |
| `availability` | 15 minutes | 7 days |

A client:

- MUST keep a body until `stale_ttl` and refetch once `ttl` has passed.
- **MUST keep serving the body it holds when a refetch fails, until `stale_ttl`, whatever the failure**, including `401`, `410` and every `5xx`. Stale beats blank. A deploy is a few seconds of downtime for every Tenant, and without this every Tenant's site would flicker on every release.
- MUST back off on outages (see [escalation](#telling-someone)).
- SHOULD offer its administrator a "refresh now" action that discards its cached bodies.

There are no conditional requests in v1: no `ETag`, no `304`. HestaHub caches nothing server-side, and nothing pushes invalidations. Freshness is time plus the refresh action.

**Revoking a key does not take a website down.** HestaHub refuses the next call at once, but a client serving stale keeps rendering for up to `stale_ttl`. Revoking a key does not remove data from the internet.

## The client is a tolerant reader

A client MUST read field by field and MUST NOT validate a whole payload against a bundled schema.

- An unrecognised object key MUST be ignored silently.
- An unrecognised value in a field the client branches on MUST fall back to a defined default. Where no default is sensible, the client skips that item, not the page.
- A missing expected field degrades that field alone.

A client that does not do this forfeits the promise below, because every guarantee in it depends on this behaviour.

## Requirements on the WordPress plugin

These rules bind the plugin. Some exist because the statuses it emits are themselves contract. Others exist because the plugin's templates are.

### Routes

- The plugin owns two base paths, one for Apartments and one for Clinics, each set by the site's administrator. It serves `<base>` and `<base>/<slug>` under each. A base path is registered only while `meta.resources` lists its resource.
- **Both base paths ship empty.** The plugin owns no URL until an administrator sets one. While permalinks are plain, it MUST refuse to accept a base path, and say why: no rewrite rule can work then.
- On taking a base path, the plugin MUST raise a persistent admin notice listing what it now shadows. That means the existing page at the base and each of its direct children, each linked; grandchildren, which survive orphaned under a base that no longer resolves; and the core endpoints `<base>/feed`, `/embed` and `/trackback`, shadowed as a side effect.
- **Routes come from configuration, never from data.** A base path resolves whether or not the API answers.
- The calendar renders three months server-side and pages by query parameter. No JavaScript is required.
- The plugin echoes its own `<link rel="canonical">` on every owned route. It sets the document title where the theme allows it. It emits no other metadata.

### What a route returns when there is no data

The render is a ladder on a clock, counted from the last successful fetch. A fresh install that has never fetched starts at the top:

1. **The cached copy**, through soft expiry and on until `stale_ttl`.
2. **`503` with `Retry-After`** (delta-seconds, matching the real retry interval) and a themed "temporarily unavailable" body, for at most 24 hours.
3. **`200` with an indexable placeholder** page, indefinitely.

Exception: a `<base>/<slug>` the plugin has **never** resolved answers `404`. These statuses are contract. The markup behind them is not.

Before emitting a `503`, the plugin MUST do all of the following, as a unit, before any output:

1. Define `DONOTCACHEPAGE` as boolean `true`.
2. Fire `litespeed_control_set_nocache`.
3. Call `nocache_headers()`.
4. Set the status with `status_header( 503 )`, never `http_response_code()`.

The body MUST be larger than 512 bytes. Page caches otherwise store the outage and replay it as `200 OK`.

**Nothing operational reaches a visitor.** No error text, no diagnostic, no version or sunset warning is ever rendered publicly. The availability calendar always shows a "Stand: <date>" line, fresh or stale, taken from the plugin's last successful fetch.

### Telling someone

Escalation depends on whether retrying can help:

- **Verdicts** (`401`, both kinds of `404`, `410`): raise an admin notice, a Site Health entry and an email on the **first** occurrence, then retry hourly.
- **Outages** (`5xx`, a timeout, a connection failure, an unparseable body): back off exponentially to a one-hour cap. Raise a notice after **24 continuous hours** and send an email on **day 5**.

Email goes by WP-Cron to the WordPress admin address. It is sent on a transition into failure and is never repeated. It is never sent by a plugin that has never fetched successfully. A notice names both the diagnosis and who can fix it: for `tenant_inactive` and `module_disabled`, that is whoever administers `<tenant>.hestahub.com`, not the webmaster. An empty base path gets a dismissible setup prompt and is not a health issue. An Apartment that disappears from a successful response raises nothing. The settings screen lists the slugs the plugin currently holds.

A **test connection** action calls `meta` live, bypassing the cache. It reports `tenant.name`, the Revision, `minimum_plugin_version` (or the failure's `code` and `message`), and probes once whether the host would rewrite the plugin's `503`. It is a separate action from "refresh now", because "refresh now" discards the stale copy that may be holding the site up.

### Being out of date

**HestaHub never refuses a caller on version grounds.** A plugin below `meta.minimum_plugin_version` shows an admin notice to the webmaster, never to a visitor. A site on PHP below 8.0 gets an admin notice that says "PHP 7". It never names a supported branch and never blocks.

### Templates and hooks

The plugin's templates are as public as the wire. A theme may override any of them by copying it into `<theme>/hestahub/`.

- The plugin ships six templates: Apartment listing, Apartment single, a rate-table partial, a calendar partial, Clinic listing and Clinic single.
- **A template's variables are contract. Its markup is not.** Each template's variables are the wire objects it renders, passed as associative arrays exactly as received: `$apartments`/`$apartment`, `$apartment['pricing']` for the rate table, the Apartment's availability entry plus the window for the calendar, and `$clinics`/`$clinic`. That is why this document has no separate template field list: the schema is the list, and any change to the wire's promise is a change to the templates' promise under the same Revision.
- Named action hooks, with their names and arguments, are contract on the same terms. The plugin fires each hook after the main content of the template named in it, passing the object(s) shown:
  - `hestahub_after_apartment_listing` (`$apartments`)
  - `hestahub_after_apartment` (`$apartment`)
  - `hestahub_after_clinic_listing` (`$clinics`)
  - `hestahub_after_clinic` (`$clinic`)

  Tenant content the API does not carry belongs on these hooks: enquiry forms, rental policies, the dog fee, discounts.

## Compatibility and Revisions

### What `v1` means

`v1` in the path is a **Version**. It changes only for a breaking change. Inside it, every non-breaking change bumps a date-stamped **Revision** (`2026-11-23`), which sorts without a parser. Each Revision:

- is a directory under `v1/` holding a full snapshot of every endpoint's schema
- is a git tag
- appears in the [revision history](#revision-history)
- is reported by `meta` and in every response body

This document says what was promised as of a date. The live API says what is true now. Those are different facts, and comparing them is how you see drift.

**This promise binds from the first Revision installed on a server we do not control.** It does not bind from the first commit or the first tag. The baseline Revision date will be recorded here when that happens.

### We may do these things at any time, in any Revision

- Add a field to a response object
- Add a value to an enum-like field
- Add an endpoint, or an optional query parameter
- Relax validation, accepting input previously rejected
- Add a **new** error `code` for a condition that previously had no code of its own
- Change data: a new Apartment, a new Feature, a changed price. Data is not schema
- Reword any `message`. Error `code` is contractual; `message` is not
- Change the layout of asset URLs
- Change the numbers in `cache`
- Correct a computed value that was previously wrong. This is a bug fix, but it appears in the revision history with the words "changed output", because "we fixed it" and "your prices changed" are the same event seen from the two ends

### These require a new Version

- Remove or rename a field
- Remove a value from an enum-like field
- Remove an endpoint
- Change a field's type or format
- Make an optional parameter required, or otherwise tighten validation
- Change the HTTP status code of a condition that already had one
- Change the shape of the error envelope
- Change a collection's sort order
- **Introduce `null` into a field that was never null.** It is not a type change in the loose sense, but it is fatal to a reader that never had to check for null
- **Change the meaning of a field while keeping its name and type.** Tolerance protects against nothing here: everything parses, everything renders, and the page is quietly wrong

Template variables and hook names and arguments follow the same list.

### What enforces this

Each Revision's schemas are hand-written as the promise; examples are generated. CI asserts:

- that every example validates against its schema
- that a live response validates against the current Revision's schema
- that no `required` key disappears and no type changes between one Revision's schemas and the next
- that the app's Revision constant, the top revision-history entry and the newest tag agree with the newest directory

The plugin's suite vendors the schemas, and fails if its vendored copy differs from the published one for the Revision it claims.

### Retirement

v1 has no scheduled death.

- Minimum **12 months** of overlap after v2 reaches general availability.
- After that, v1 is retired only once no key has called it in **90 days**. The decision rests on evidence, not on a calendar.
- Once v2 ships, v1 is **frozen at its final Revision** and receives correctness and security fixes only. New capability lands in v2 alone.
- On retirement, every endpoint answers **`410 Gone`** with `version_gone`. It does not answer `404`, which already means "this Tenant does not have this Module". The two call for opposite actions.
- **`meta` is the single exemption.** It keeps answering `200` and reports the retirement and its sunset date. Silencing the endpoint that explains retirement would be a failure shaped like an answer.

## Revision history

| Revision | Changes |
|---|---|
| `2026-11-23` | First draft: `meta`, `apartments`, `availability`, `clinics`. Not yet binding. |
