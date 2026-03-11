# CitySpark Portal API Reference

This document describes the CitySpark Portal API — how to authenticate, which endpoints are available, what to send, and what to expect back.

---

## Authentication

### API Key

All protected endpoints accept an API key via the `X-API-Key` header:

```
X-API-Key: cspk_<your-key>
```

API keys are prefixed with `cspk_`. Keys are SHA-hashed before database lookup and are validated against a 15-minute in-memory cache. Each key has configurable rate limits:

- **Short window:** max calls per minute
- **Long window:** max calls per hour

### JWT Bearer Token (Fallback)

An RSA-signed JWT is also accepted as a fallback:

```
Authorization: Bearer <token>
```

> **Important:** If a request originates from a browser cross-origin context (i.e., the `Sec-Fetch-Site` header is present and not `same-origin`), the `X-API-Key` header is **rejected**. In that case, use a Bearer token instead.

### Claims Set on Successful Auth

When a key is validated, the following claims are attached to the request principal:

| Claim | Description |
|---|---|
| `Acct` | Materialized path / account hierarchy |
| `Name` | The API key value itself |
| `Id_Sub` | User ID to act as |
| `Access` | Rights string |
| Roles | One or more role claims |

---

## Services & Base URLs

There are two backend services that expose event/portal-related APIs:

| Service | Description | OpenAPI Spec |
|---|---|---|
| **CitySpark.Portal** | Event portal — the primary API for fetching events | `/openapi` |
| **CitySpark.Biz.Api** | Business/BestOf API | `/openapi/v1.json` · Scalar UI at `/docs` |

This document focuses on **CitySpark.Portal** and its `/api/events` and `/api/auth` routes.

---

## Response Envelope

Every endpoint returns a JSON envelope. There are three variants:

### `ResultSuccess` (base — no value)
```json
{
  "Success": true,
  "ErrorMessage": null,
  "UserOkErrorMsg": false
}
```

### `ResultSuccessValue<T>` — single value
```json
{
  "Success": true,
  "Value": { ... }
}
```

### `ResultSuccessPossible<T>` — paginated list
```json
{
  "Success": true,
  "Value": [ ... ],
  "Possible": 142
}
```
`Possible` is the **total matching record count** — use it to drive "load more" / pagination UI.

### Error response
```json
{
  "Success": false,
  "ErrorMessage": "Event not found",
  "UserOkErrorMsg": true
}
```
`UserOkErrorMsg: true` means the error message is safe to display directly to end users.

---

## `/api/events` Endpoints

All event endpoints are `[AllowAnonymous]` — no API key is required to call them, though some portal configurations may restrict behavior.

Legacy aliases exist at `/v1/events/{slug}`, `/v1/event/{slug}`, and `/sitemap/{slug}` and behave identically.

---

### `POST /api/events/GetEvents/{slug}`

Returns a paginated list of events for a portal.

**Route param:** `slug` — the portal's URL slug identifier.

#### Request Body: `GetEventsRequest`

```json
{
  "ppid": 0,
  "start": "2026-03-11T00:00:00",
  "end": "2026-03-18T00:00:00",
  "skip": 0,
  "lat": 40.7128,
  "lng": -74.0060,
  "distance": 25.0,
  "search": "music festival",
  "sort": "start",
  "category": [10, 42],
  "labels": ["csVirtual"],
  "tps": "All",
  "defFilter": "all"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `ppid` | `int` | Yes | Portal ID |
| `start` | `DateTime` | Yes | Start of search window |
| `end` | `DateTime?` | No | End of search window — when set, page size increases to 100 |
| `skip` | `int` | No | Pagination offset (0-based). Max 2000 globally; max 155 for historical queries |
| `lat` | `double` | No | Center latitude for geo search |
| `lng` | `double` | No | Center longitude for geo search |
| `distance` | `double` | No | Search radius (miles or km per portal config) |
| `search` | `string` | No | Free-text keyword search |
| `sort` | `string` | No | Sort order. Valid values: `"name"`, `"A to Z"`, `"Z to A"`, `"start"`, `"Time"`, `"popularity"`, `"Popularity"`, `"total_popularity"`, `"interest"`, `"day"`, `"unique"` |
| `category` | `int[]` | No | Category IDs (additive with portal defaults) |
| `labels` | `string[]` | No | Label strings — intersected with portal-level label config |
| `tps` | `string` | No | Ticket Partner String — controls `ticketLight` on returned events. Values: `"All"`, `"Internal"`, `"External"`, or comma-separated partner IDs |
| `defFilter` | `string` | No | Special filter mode — see below |

#### `defFilter` Values

| Value | Behavior |
|---|---|
| `"virtual"` | Virtual events only |
| `"discount"` or `"deals"` | Discounted / deal events only |
| `"ourTix"` | Events with CitySpark-partner ticketing only |
| `"picks"` | Editor's Picks for this portal only |
| `"bookmarks"` | Logged-in user's saved/sparked events (returns empty list if not authenticated) |
| `"free"` | Free events only (max ticket price = $0) |
| `"all"` | No filter — portal defaults apply |
| *any other string* | Treated as a **label filter** (e.g. `"csNational"` to show national events) |

#### Pagination Rules

- Default page size: **25 events**
- Page size when `end` is provided: **100 events**
- `Possible` in the response = total matching count
- `skip` max: **2000** globally
- Historical queries (`start` > 15 days ago): `skip` max = **155**
- Historical `end` clamping: if `start` is > 15 days in the past, `end` is capped to `start + 1 month` (or `start + 10 days` if no `end` was provided)

#### Response: `ResultSuccessPossible<List<jsEvent>>`

```json
{
  "Success": true,
  "Value": [ /* array of jsEvent — see below */ ],
  "Possible": 87
}
```

#### Notable Error Responses

| Condition | `ErrorMessage` |
|---|---|
| Portal slug not found | `"No portal found with that slug"` |
| `skip > 2000` | `"Please refine your search"` |
| Historical scroll exceeded | `"Historical events can not be scrolled"` |
| Malformed/null request body | `"malformed request"` |

---

### `POST /api/events/GetEvent/{slug}`

Returns a single event by its persistent ID, with full occurrence details.

**Route param:** `slug` — the portal's URL slug.

#### Request Body: `GetEventRequest`

```json
{
  "pid": 123456789,
  "time": "19:30",
  "ppid": 0,
  "tps": "All",
  "hier": [2, 45, 99],
  "userId": 0,
  "lat": 40.7128,
  "lng": -74.0060
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `pid` | `long` | Yes | Persistent event ID (series-level) |
| `time` | `string` | No | Time hint in `"HH:mm"` format to narrow which occurrence is returned. The system tries ±48 hours, then ±14 days, then any occurrence. |
| `ppid` | `int` | No | Portal ID — used for `Picked`/`enhance`/ticket light checks |
| `tps` | `string` | No | Ticket Partner String — same as GetEvents |
| `hier` | `int[]` | No | Account hierarchy IDs for share/block scoping |
| `userId` | `int` | No | If > 0, logs a `ViewEvent` tracking action for this user |
| `lat` | `double?` | No | If provided with `lng`, full time zone adjustment is applied to all occurrences |
| `lng` | `double?` | No | See `lat` |

#### Lookup Logic

The API performs a **three-pass lookup** to find the best matching occurrence:
1. Match by `pid` + `time` within **±48 hours**
2. Match by `pid` + `time` within **±14 days**
3. Match by `pid` only (no date constraint)

If all three passes fail, a `"Event not found"` error is returned.

#### Response: `ResultSuccessValue<jsEvent>`

```json
{
  "Success": true,
  "Value": { /* jsEvent — see below */ }
}
```

> The single-event response includes the full `Occurances` list (all occurrences in the series) and is mapped with `loadForEdit: true`, which populates the `field1` custom field.

---

### `POST /api/events/SiteMap`

Returns an XML sitemap of up to **1799 events** for the portal. Primarily used for SEO.

**Response:** `text/xml` — standard sitemap format with `<loc>`, `<lastmod>`, `<changefreq>monthly`, `<priority>0.8` per event.

> Internally fetches events in batches of 301 with a 250ms delay between pages.

---

## The `jsEvent` Object

This is the primary event model returned by both `GetEvents` and `GetEvent`.

### Identity & Display

| Field | Type | Notes |
|---|---|---|
| `Id` | `string` | MongoDB `_id` — unique occurrence-level identifier |
| `PId` | `long` | Persistent/series-level ID — stable across all occurrences of the same event |
| `SubmitPID` | `long?` | Submit Persistent ID (from user-submitted events) |
| `Name` | `string` | Event name |
| `Description` | `string` | Full event description (markdown) |
| `Short` | `string` | Auto-generated SEO snippet (~160 chars): *"The event is held on {date} at {venue} in {city}. [Free.] [Cost is $X-Y.]"* |
| `field1` | `string` | Custom portal-defined field — only populated in `GetEvent` (`loadForEdit: true`) |

### Location

| Field | Type | Notes |
|---|---|---|
| `Venue` | `string` | Venue or location name |
| `VenuePhone` | `string` | Venue phone number |
| `CityState` | `string` | Formatted as `"City, ST"` (US) or `"City, Country"` (international) |
| `Address` | `string` | Street address |
| `Zip` | `string` | Postal/ZIP code |
| `latitude` | `double?` | Geo latitude |
| `longitude` | `double?` | Geo longitude |
| `Distance` | `double?` | Distance from the search center (in portal's configured unit) |

### Dates & Time

| Field | Type | Notes |
|---|---|---|
| `Date` | `DateTime` | Day-level start date (UTC-forced) — used for day grouping and sorting |
| `DateStart` | `DateTime` | Occurrence start datetime (UTC) |
| `DateEnd` | `DateTime?` | Occurrence end datetime (UTC), null if open-ended |
| `AllDay` | `bool` | `true` if the event has no specific time |
| `HasTime` | `bool` | `true` if the event has a specific start time |
| `StartLocal` | `DateTime?` | Original local start time — set when a national event is displayed in a different timezone than its origin |
| `EndLocal` | `DateTime?` | Original local end time — same as above |
| `StartUTC` | `DateTime?` | Start converted to UTC from the event's own geo timezone |
| `EndUTC` | `DateTime?` | End converted to UTC from the event's own geo timezone |
| `tzAbbrev` | `string` | Timezone abbreviation (e.g., `"EST"`) — only set for national events shown outside their home timezone |
| `Occurances` | `List<jsOccurance>` | All occurrences in the series — only populated on `GetEvent` responses |
| `multipleTimes` | `List<DateTime>` | Additional times for events with multiple daily time slots |
| `hasEarlierTimes` | `bool` | Indicates there are earlier time slots not included in this response |

### Categories, Format & Labels

| Field | Type | Notes |
|---|---|---|
| `Tags` | `List<int>` | Category IDs assigned to this event |
| `BiasTags` | `List<int>` | Bias/recommendation tag IDs |
| `Labels` | `string[]` | Raw label strings. Common values: `"csVirtual"`, `"csNational"`, `"csHybrid"` |
| `Format` | `int` | Event format enum: `0`=Unknown, `1`=InPerson, `2`=Virtual, `3`=Hybrid (VirtualWithInPerson), `4`=VirtualWithPickup, `5`=VirtualWithDriveUp |
| `isVirtual` | `bool` | `true` if `Labels` contains `"csVirtual"` |
| `IsPromotion` | `bool` | `true` if this is a promotional event |

### Pricing

| Field | Type | Notes |
|---|---|---|
| `Price` | `decimal?` | Low price (or single price if no range) |
| `PriceHigh` | `decimal?` | High price — only set when both low and high prices exist |
| `LowFullPrice` | `decimal?` | Full (pre-discount) low price |
| `HighFullPrice` | `decimal?` | Full (pre-discount) high price |
| `Free` | `bool` | `true` if the event is free |
| `PriceText` | `string` | Human-readable price description |
| `IsTicketed` | `bool` | `true` if `TicketInfo` is non-null |
| `TicketInfo` | `TicketInfo` | Structured ticket info object |
| `TicketUrl` | `string` | Primary ticket purchase URL |
| `Tickets` | `jsLink[]` | All ticketing source links (up to 4). Sorted: resellers last, then by revenue potential descending |
| `ticketLight` | `bool` | Whether the ticket partner spotlight should be illuminated — controlled by the `tps` (Ticket Partner String) field in the request |

### Images

| Field | Type | Notes |
|---|---|---|
| `SmallImg` | `string` | Small image URL |
| `MediumImg` | `string` | Medium image URL |
| `LargeImg` | `string` | Large/primary image URL |
| `Images` | `jsImg[]` | All large images — primary image first, then additional event images |

### Links & Sponsors

| Field | Type | Notes |
|---|---|---|
| `Links` | `jsLink[]` | Info/source links (up to 4; excludes hidden, sponsor, and ticket links) |
| `Sponsor` | `jsLink` | First sponsor link |
| `Sponsors` | `jsImg[]` | Sponsor logo images |
| `LightUp` | `jsLight[]` | External light-up source data for select partner source IDs |
| `PrimaryUrl` | `string` | Primary event info URL (falls back to first link if a dedicated event URL is not set) |

### Media & Portal Context

| Field | Type | Notes |
|---|---|---|
| `Media` | `jsMedia[]` | Embedded media (YouTube, Vimeo, etc.) — only shown if `enhance == true` or admin-added |
| `enhance` | `bool` | `true` if the event is enhanced for the current portal (`ppid`) |
| `featured` | `bool` | Featured flag |
| `Picked` | `bool` | `true` if this is an Editor's Pick for the requested portal (`ppid`) |
| `PopularityIndex` | `int` | Popularity score |

### Contact & Social

| Field | Type | Notes |
|---|---|---|
| `ct` | `jsContact` | Contact info — only present if the event has contact data |
| `social` | `jsSocial[]` | Social media links |

---

## Supporting Sub-Models

### `jsOccurance`
An individual occurrence of a recurring event series.
| Field | Type |
|---|---|
| `Start` | `DateTime` |
| `End` | `DateTime?` |

### `jsImg`
| Field | Type |
|---|---|
| `id` | `int` |
| `url` | `string` |

### `jsLink`
| Field | Type | Notes |
|---|---|---|
| `name` | `string` | Display name |
| `url` | `string` | URL |
| `sId` | `int` | Source ID |
| `revP` | `int` | Revenue potential (used for sorting) |
| `ourTix` | `bool` | `true` if this is a CitySpark-partner ticket link |
| `type` | `int` | Link type (`8` = reseller/ticket) |

### `jsLight`
External light-up partner data. Only populated for specific partner source IDs (3, 111, 22).
| Field | Type | Notes |
|---|---|---|
| `id` | `int` | Source ID |
| `extId` | `string` | Event ID on that external source |
| `url` | `string` | Event URL on that external source |

### `jsMedia`
| Field | Type |
|---|---|
| `embed` | `string` | HTML embed code |
| `url` | `string` |

### `jsContact`
| Field | Type |
|---|---|
| `name` | `string` |
| `phone` | `string` |
| `email` | `string` |
| `org` | `string` |

### `jsSocial`
| Field | Type | Notes |
|---|---|---|
| `url` | `string` | Full social profile/page URL |
| `platform` | `string` | Platform name: `"Twitter"`, `"Facebook"`, `"Instagram"`, `"Pinterest"`, `"TikTok"`, `"YouTube"`, `"LinkedIn"`, `"Snapchat"`, `"BlueSky"`, `"Other"` |
| `handle` | `string` | @handle |

---

## `/api/auth` Endpoints

These endpoints handle user session state, bookmarks, labels, and admin actions within a portal context.

### `POST /api/auth/LoginInfo/{slug?}/{ppid}` — Anonymous

Returns the current user's login state and portal-specific rights.

**Response: `ResultSuccessValue<GetLoginResult>`**

```json
{
  "Success": true,
  "Value": {
    "LoggedIn": true,
    "UserName": "jane@example.com",
    "Sparks": [123456, 789012],
    "Rights": {
      "ViewEvents": true,
      "BlockEvents": false
    },
    "Roles": ["editor"]
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `LoggedIn` | `bool` | Whether the user is authenticated |
| `UserName` | `string` | User's display name / email |
| `Sparks` | `IEnumerable<long>` | PIds of events the user has bookmarked ("sparked") |
| `Rights` | `Dictionary<string, bool>` | Portal-specific rights map |
| `Roles` | `string[]` | Role names |

---

### `POST /api/auth/UserAction` — Anonymous

Record a user interaction (bookmark, view, etc.).

**Request body: `UserActionModel`**

| Field | Type | Notes |
|---|---|---|
| `PId` | `long?` | Event persistent ID |
| `Id` | `string` | MongoDB event `_id` |
| `action` | `string` | Action type: `"Spark"`, `"ViewEvent"`, etc. |
| `remove` | `bool` | `true` to remove the action (e.g., un-bookmark) |
| `ppid` | `int` | Portal ID |

**Response:** `bool` (success/failure)

---

### `POST /api/auth/GetLabels` — Requires Auth

Returns the list of labels available for the authenticated user's context.

**Request body:** `{ "q": "search string" }`  
**Response: `ResultSuccessValue<List<string>>`**

---

### `POST /api/auth/InlineEdit` — Requires Auth + `ViewEvents` right

Inline-edit an event. Returns the event format enum value.

**Request body: `EditRequest`**

| Field | Type | Notes |
|---|---|---|
| `eventId` | `long?` | Existing event ID (null to create) |
| `submitId` | `long?` | Submit ID |
| `portalId` | `int` | Required |
| `eventName` | `string` | Required |
| `eventDesc` | `string` | |
| `labels` | `string[]` | |
| `categories` | `int[]` | |
| `makePrivate` | `bool` | |
| `field1` | `string` | Custom portal field |
| `price` | `decimal?` | |
| `priceHigh` | `decimal?` | |
| `lowFullPrice` | `decimal?` | |
| `highFullPrice` | `decimal?` | |
| `free` | `bool?` | |
| `priceText` | `string` | |
| `address` | `string` | |
| `venueName` | `string` | |

**Response: `ResultSuccessValue<int>`** — event format enum value

---

### `POST /api/auth/Block` — Requires Auth + `BlockEvents` right

Block an event from appearing on the portal.

**Request body: `BlockRequest`**

| Field | Type |
|---|---|
| `_id` | `long` |
| `ppid` | `int` |
| `blockAll` | `bool` |
| `reasons` | `List<string>` |

**Response:** `long` — the created block record ID

---

### `POST /api/auth/Pick` / `POST /api/auth/UnPick` — Requires Auth + `BlockEvents` right

Mark or unmark an event as an Editor's Pick for this portal.

**Request body:** `BlockRequest` (same as Block above)  
**Response:** `bool`

---

## Example: Fetch This Week's Events

```http
POST /api/events/GetEvents/your-portal-slug
Content-Type: application/json

{
  "ppid": 123,
  "start": "2026-03-11T00:00:00",
  "end": "2026-03-18T00:00:00",
  "lat": 40.7128,
  "lng": -74.0060,
  "distance": 25,
  "sort": "start",
  "defFilter": "all",
  "skip": 0
}
```

```json
{
  "Success": true,
  "Possible": 42,
  "Value": [
    {
      "Id": "64abc123...",
      "PId": 987654321,
      "Name": "Spring Jazz Night",
      "Short": "The event is held on March 14 at Blue Note in New York, NY. Cost is $20-35.",
      "Date": "2026-03-14T00:00:00Z",
      "DateStart": "2026-03-14T23:00:00Z",
      "DateEnd": "2026-03-15T02:00:00Z",
      "HasTime": true,
      "AllDay": false,
      "Venue": "Blue Note",
      "CityState": "New York, NY",
      "Address": "131 W 3rd St",
      "latitude": 40.7308,
      "longitude": -74.0005,
      "Distance": 3.2,
      "Price": 20.00,
      "PriceHigh": 35.00,
      "Free": false,
      "PriceText": "$20 - $35",
      "Tags": [10, 42],
      "Labels": [],
      "Format": 1,
      "isVirtual": false,
      "LargeImg": "https://cdn.cityspark.com/images/...",
      "PrimaryUrl": "https://www.bluenote.net/...",
      "Picked": false,
      "enhance": false,
      "PopularityIndex": 74
    }
  ]
}
```

---

## Example: Fetch a Single Event

```http
POST /api/events/GetEvent/your-portal-slug
Content-Type: application/json

{
  "pid": 987654321,
  "time": "19:00",
  "ppid": 123,
  "lat": 40.7128,
  "lng": -74.0060
}
```

The response includes the full `jsEvent` object plus the complete `Occurances` list for all upcoming dates in the series.
