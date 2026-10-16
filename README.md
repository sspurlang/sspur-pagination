# pagination

Pagination for SSPUR list endpoints: offset and page-number pagination with clamped (or strictly
checked) requests, keyset pagination with opaque, tamper-evident cursors signed with HMAC-SHA-256,
page facts (total pages, has next/previous, first/last), page-number ranges for UIs with elided
middles, slicing of in-memory lists, RFC 8288 Link headers (written and parsed) and JSON envelopes.
Everything is pure: no effects, no clock, no randomness.

## Example

```
use pagination.{request_str, paginate, link_header, range_label, page_numbers_for, page_numbers_text, after, keyset, seek, sign_page, decode_cursor}

fn by_id(id: Int, keys: List[Str]) -> Int
= id - keys.first.map(k => k.to_int.or(0)).or(0)

fn main() -> Unit ! log
= do
  ids = (1..96).map(i => i)
  p = paginate(ids, request_str(some("20"), some("40")))
  log(range_label(p))
  log(page_numbers_text(page_numbers_for(p, 1, 1)))
  log(link_header("/items", p))
  key = secret("change me: 32 random bytes".to_bytes)
  rows = seek(ids, some(after(["90"])), 3, by_id)
  cp = sign_page(key, "items?sort=id", keyset(rows, 3, some(after(["90"])), k => [k.str]), none)
  log("{cp.items} next={cp.next_cursor.is_some} prev={cp.prev_cursor.is_some}")
  log(cp.prev_cursor.map(t => decode_cursor(key, "items?sort=id", t)).str)
  log(cp.prev_cursor.map(t => decode_cursor(key, "items?sort=name", t)).str)
```

prints (natively and with `--interp`):

```
41-60 of 95
1 2 [3] 4 5
</items?limit=20&offset=0>; rel="first", </items?limit=20&offset=20>; rel="prev", </items?limit=20&offset=60>; rel="next", </items?limit=20&offset=80>; rel="last"
[91, 92, 93] next=true prev=true
some(ok(Cursor{keys: ["91"], dir: Backward, expires_ms: none}))
some(err(BadSignature))
```

In a real service the key comes from configuration (`crypto.key()` once, stored as a secret), and
`seek` is your SQL: `WHERE id > ? ORDER BY id LIMIT ?` with `fetch_limit(limit)` rows.

## Clamping rules

| Input | Rule |
|---|---|
| limit / page size | default 20 (or `Limits.default_limit`), kept within 1 to 100 (or `Limits.max_limit`) |
| offset | default 0, never negative, no upper bound (a page past the end is empty) |
| page number | default 1, never below 1 |
| query text | trimmed, read as an integer; anything else counts as missing (`request_str`), or is refused (`strict_request`, `strict_numbered`) |
| a `Page` built by hand | a limit below 1 counts as 1, a negative offset or total as 0 |
| arithmetic | offsets saturate at the largest `Int` instead of overflowing |

Refinements guard the rest: `count_pages`, `last_offset` and `bounds` need `total >= 0` and
`limit >= 1`; `page_numbers` needs `pages >= 0` and a window and edge count of 0 to 1000; keyset
limits are 1 to a million. A violation stops the program with the clause that failed.

## API

**Offset requests**

- `PageReq` a checked request: limit 1 to 100, offset at least 0
- `Limits` your own default and maximum page size (within 1 to 100)
- `request(limit, offset)` clamped request from optional integers
- `request_in(lim, limit, offset)` the same with your limits
- `request_str(limit, offset)` the same from raw query strings
- `strict_request(lim, limit, offset)` refuses bad values with a `ParamErr` instead of clamping
- `ParamErr` `NotANumber{param, text}` or `OutOfRange{param, value, low, high}`

**Pages**

- `Page[T]` items, total, limit, offset and the offset of the next page
- `page(items, total, r)` the page for rows your query returned
- `paginate(xs, r)` the page of an in-memory list
- `map_items(p, f)` transform the items, keep the rest
- `bounds(r, total)` the `[start, end)` rows a request covers, clamped
- `clamp_req(r, total)` move a request past the end back to the last page
- `last_offset(total, limit)`, `count_pages(total, limit)`

**Page facts and neighbours**

- `page_count(p)`, `page_number(p)` (1-based)
- `has_next(p)`, `has_prev(p)`, `is_first(p)`, `is_last(p)`, `is_empty_page(p)`, `out_of_range(p)`
- `prev(p)` the offset of the previous page
- `next_req(p)`, `prev_req(p)`, `first_req(p)`, `last_req(p)`
- `shown_range(p)` 1-based first and last item, `range_label(p)` such as `21-40 of 95`

**Page numbers**

- `NumReq` a checked page number and size
- `numbered(number, size)`, `numbered_in(lim, number, size)`, `numbered_str(number, size)`, `strict_numbered(lim, number, size)`
- `offset_req(n)` and `num_req(r)` convert between page numbers and offsets
- `PageItem` `PageNo{number, current}` or `Ellipsis`
- `page_numbers(current, pages, window, edges)` the elided range for a UI: page 8 of 20 with window 1 and edges 1 is `1 … 7 [8] 9 … 20`; a gap of one page shows that page instead of an ellipsis
- `page_numbers_for(p, window, edges)` the same for a page, `page_numbers_text(items)` as text

**Cursors (keyset pagination)**

- `Cursor` the sort keys of a row (as text), a `Dir` (`Forward` or `Backward`) and an optional expiry
- `after(keys)`, `before(keys)`, `is_expired(c, now_ms)`
- `encode_cursor(key, scope, c)` an opaque token: base64url JSON, `.`, and a 128-bit HMAC-SHA-256 tag over the payload and the scope
- `decode_cursor(key, scope, token)`, `decode_cursor_at(key, scope, token, now_ms)` (also checks expiry), `decode_cursor_any(keys, scope, token, now_ms)` (key rotation)
- `CursorErr` `Malformed{reason}`, `BadSignature`, `Expired{at_ms}`, `Unsupported{version}`
- `fetch_limit(limit)` rows to fetch: one more than the limit
- `seek(sorted, req, limit, cmp)` the rows a database would return, for in-memory lists
- `keyset(rows, limit, req, key_of)` the page and its neighbours' cursors (`KeysetPage[T]`), from rows fetched ascending after the cursor or descending before it
- `sign_page(key, scope, kp, expires_ms)` sign the cursors (`CursorPage[T]`), `map_cursor_items(cp, f)`

The scope binds a token to one query: sign with the endpoint, filters and sort order (say
`"orders?status=open&sort=date"`) and a cursor from another query is refused as `BadSignature`.
Tokens are tamper-evident, not encrypted: anyone holding one can read its keys.

**HTTP**

- `set_query(url, params)` set query parameters, percent-encoded, keeping the others and the fragment
- `offset_url(url, r)`, `page_url(url, number, size, page_param, size_param)`
- `link_header(url, p)`, `link_header_numbered(url, p, page_param, size_param)`, `cursor_link_header(url, param, cp)` RFC 8288 `Link` values with `first`, `prev`, `next` and `last`
- `parse_link_header(s)` the links of a `Link` value (`WebLink{uri, rel, params}`), `rel_uri(links, rel)`
- `total_count_header(p)` the `X-Total-Count` header

**JSON**

- `Envelope[T]` `{"data": [...], "meta": {...}, "links": {...}}` with `Meta` (total, limit, offset, number, pages, next and previous offsets) and `Links` (self, first, prev, next, last URLs)
- `envelope(url, p)`, `page_meta(p)`, `page_link_set(url, p)`
- `CursorEnvelope[T]` and `cursor_envelope(url, param, cp)` for cursor pages
- `Page[T]` itself encodes as `{"items": [...], "total": 95, "limit": 20, "offset": 40, "next": 60}`

## Changes in 0.2.0

Everything in 0.1 keeps its signature. Two fixes in behaviour: `page` no longer sets `next` on an
empty page (a client reading past the end would otherwise ask for the same offset forever), and
`prev` of a page past the end points at the last page instead of `limit` rows back.

## Out of scope

- Running queries: `seek` shows what to fetch, your SQL does the fetching (`db.query` with
  `WHERE (a, b) > (?, ?) ORDER BY a, b LIMIT ?`).
- Encrypted cursors: tokens are signed, not sealed. Seal them with `crypto.seal` yourself if the keys must stay private.
- Page sizes above 100: `PageReq`'s range is part of its type since 0.1; `Limits` can lower it.
- Total counts for keyset pages, and GraphQL Relay connection types.

## Install

```
sspur add pagination
```

MIT licensed.
