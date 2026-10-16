# pagination

Offset pagination for SSPUR list endpoints: page requests clamped to sane limits, and pages that carry the total and the offset of the next page.

```
import pagination

fn list(limit: Opt[Int], offset: Opt[Int]) -> pagination.Page[Str]
= do
  r = pagination.request(limit, offset)
  rows = ["a", "b", "c"].drop(r.offset).take(r.limit)
  pagination.page(rows, 3, r)
```

`request` defaults `limit` to 20 and keeps it within 1 to 100, and never lets `offset` go below 0. `page` sets `next` while rows remain, and `prev` gives the offset of the page before.

## Install

```
sspur add pagination
```

MIT licensed.
