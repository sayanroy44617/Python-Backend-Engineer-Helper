# Pagination, Filtering, and Sorting

## What

Techniques for exposing large collections through a list endpoint without
returning everything at once: **pagination** (splitting results into
pages), **filtering** (returning a subset matching criteria), and
**sorting** (controlling result order).

## Why

Returning an entire table in one response doesn't scale — it's slow,
wastes bandwidth, and risks timeouts or memory pressure as data grows.
Pagination, filtering, and sorting are what make list endpoints usable at
real data volumes, and are among the most frequently designed (and
frequently gotten wrong) parts of a REST API.

## How

### Offset-based pagination

```
GET /orders?skip=40&limit=20
```

```json
{
  "items": [ /* 20 orders */ ],
  "total": 483,
  "skip": 40,
  "limit": 20
}
```

Simple to implement (`OFFSET`/`LIMIT` in SQL), but has real drawbacks at
scale: `OFFSET` still has to scan/skip past all prior rows (slow for deep
pages), and results can shift if rows are inserted/deleted between page
requests (a "page 3" fetched after an insert may skip or repeat an item).

### Cursor-based pagination

```
GET /orders?limit=20&after=eyJpZCI6IDQyfQ==
```

```json
{
  "items": [ /* 20 orders */ ],
  "next_cursor": "eyJpZCI6IDYyfQ==",
  "has_more": true
}
```

The cursor encodes a position (commonly the last item's sort key/ID), and
the next page is fetched with `WHERE id > :cursor_id ORDER BY id LIMIT
:limit` — no `OFFSET` scan, and stable under concurrent inserts/deletes
since each page is anchored to a specific row, not a numeric position.
This is the standard choice for large, frequently-changing datasets and
infinite-scroll-style UIs.

| | Offset-based | Cursor-based |
|---|---|---|
| Implementation | Simple (`OFFSET`/`LIMIT`) | Slightly more complex (encode/decode cursor) |
| Performance at deep pages | Degrades (scans skipped rows) | Consistent |
| Stability under concurrent writes | Can skip/repeat items | Stable |
| Random access ("jump to page 5") | Yes | No (sequential only) |

### Filtering

```
GET /orders?status=shipped&created_after=2024-01-01
```

Each filterable field becomes a query parameter. For more complex
filtering needs (multiple operators per field), a small convention is
common:

```
GET /orders?price[gte]=10&price[lte]=100
```

Document exactly which fields are filterable and what operators are
supported — an unbounded, ad hoc filtering DSL is a common source of
inconsistent client expectations and unindexed-query performance
surprises.

### Sorting

```
GET /orders?sort=created_at        # ascending
GET /orders?sort=-created_at       # descending (convention: leading '-')
GET /orders?sort=status,-created_at  # multiple sort keys
```

Whatever convention is chosen (`-field` for descending, or `sort_by`/
`sort_order` as separate parameters), document it once and apply it
consistently across every list endpoint in the API.

### Response envelope for lists

```json
{
  "items": [ ... ],
  "total": 483,
  "limit": 20,
  "next_cursor": "eyJpZCI6IDYyfQ=="
}
```

Wrapping list results in an envelope (rather than returning a bare JSON
array) leaves room to add pagination metadata without a breaking change —
switching from a bare array to an object is a breaking change for
existing clients, so this decision is easiest to make correctly up front.

## When to use

- Offset-based pagination for smaller, relatively static datasets, or
  when clients need "jump to page N" behavior (e.g. an admin UI table).
- Cursor-based pagination for large, high-write-volume datasets or
  infinite-scroll UIs where stability under concurrent writes and
  consistent performance at any depth matter more than random page access.
- A small, well-documented set of filter parameters mapped directly to
  indexed database columns.
- A consistent sort parameter convention applied uniformly across every
  list endpoint.

## When NOT to use

- Don't return every row of a large/unbounded collection in one response —
  always paginate collections that can grow without bound.
- Don't expose filtering on unindexed columns without considering the
  query cost — an API contract that allows filtering on any field can turn
  into a full table scan under the hood.
- Don't design a fully generic, arbitrary query-language-style filtering
  API (e.g. allowing raw expressions) unless you specifically need that
  flexibility and have addressed the security/performance implications —
  it's easy to accidentally expose something close to unrestricted query
  execution.

## Common mistakes

- Using deep offset pagination on a large, frequently-changing table and
  being surprised by slow queries and inconsistent results (skipped/
  duplicated rows) as pages get returned out of sync with writes.
- Returning a bare array from a list endpoint, then later needing to add
  pagination metadata — a breaking change for every existing client.
- Allowing unrestricted sort/filter fields that don't have a database
  index, causing full table scans under real traffic.
- Inconsistent sorting/filtering conventions across different endpoints in
  the same API, forcing clients to special-case each one.

## Interview questions

- What's the difference between offset-based and cursor-based
    pagination? When would you choose one over the other?

    **Answer:** Offset pagination says "skip N, take M" (`?offset=100&
    limit=20`) — simple, but gets slow and inconsistent on large/changing
    tables. Cursor pagination says "give me items after this specific
    marker" (`?after=<last_id>&limit=20`) — stays fast and stable as data
    grows or changes, so prefer it for large or frequently-written tables.

- Why can offset pagination return duplicate or skipped items under
    concurrent writes?

    **Answer:** Offset is just "position N in the current result set." If
    a row is inserted/deleted before that position while you're paging,
    every row after it shifts, so page 2 can repeat or skip an item
    compared to page 1.

- Why is wrapping list responses in an envelope (`{"items": [...],
    "total": ...}`) generally preferred over returning a bare array?

    **Answer:** An envelope gives you room to add pagination metadata
    (`total`, `next_cursor`) without changing the response's top-level
    shape later — a bare array can only ever be a list, so adding metadata
    is a breaking change.

- What database-level risk does unrestricted, arbitrary filtering expose?

    **Answer:** Letting clients filter/sort on any column can force full
    table scans on unindexed columns, or open a path to SQL injection if
    filter values are concatenated into raw SQL instead of parameterized.

- How would you design a sort parameter that supports multiple sort keys
    and both directions?

    **Answer:** Accept a comma-separated list with an optional `-` prefix
    for descending, e.g. `?sort=-created_at,name`, and validate it against
    an explicit allow-list of sortable columns.

## Senior-level considerations

- Pagination strategy is a data-model decision, not just an API surface
  decision — cursor-based pagination usually requires a stable, indexed
  sort key (often the primary key or a composite key) chosen deliberately
  at schema design time. For example, cursoring on `created_at` alone
  breaks if two rows share the same timestamp; pairing it with `id` as a
  tiebreaker fixes that.
- At scale, "total count" in an envelope can itself be an expensive query
  (`COUNT(*)` on a huge, filtered table) — some APIs deliberately omit or
  approximate it rather than pay that cost on every paginated request. For
  example, returning `has_more: bool` instead of an exact `total` avoids a
  second expensive query on every page load.
- Filtering/sorting/pagination conventions should be established once as
  an API-wide standard (shared utility/dependency, not per-endpoint
  reinvention) — inconsistency here is one of the most visible signs of an
  API that grew without a coherent design review process. For example, a
  shared FastAPI dependency like `PaginationParams` used across every
  list endpoint keeps `limit`/`cursor` naming and defaults consistent.
