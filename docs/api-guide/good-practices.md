# Good Practices

How to write API code that performs and ages well. The contract in [contract.md](contract.md) tells you *where* things go; this page tells you *how* to write the code that goes there.

Almost every entry on this page is about not making more database round-trips than you have to.

---

## 1. Eager-load related rows with `select_related` and `prefetch_related`

The Django ORM is lazy. Every `obj.foreign_key` access on a fetched row that didn't pre-load that FK fires a fresh query. That's the **N+1** problem — list 100 documents and naïvely render `doc.owner.name` for each, you get 101 queries.

### When to reach for which

| Method | Use when | What it does |
|---|---|---|
| `select_related("owner")` | `ForeignKey`, `OneToOneField` | Single SQL JOIN; the FK row is hydrated in one query. |
| `prefetch_related("tags")` | `ManyToManyField`, reverse FK (`document.attachments_set`) | Two queries — main + IN-clause for the related — joined in Python. |
| `Prefetch("tags", queryset=...)` | Filtered / sorted / `select_related`'d prefetch | Same as `prefetch_related` but you control the inner queryset. |

### How this lands in NimbusVault

Because of the `service.py` rule, callers cannot reach into the ORM to add a `select_related`. The service exposes optimisation hints as kwargs:

```python
# bll/<Domain>/service.py
class ThingService:
    _DEFAULT_SELECT_RELATED = ("owner", "category")

    @staticmethod
    def filter(**filters):
        select_related = filters.pop("_select_related", ThingService._DEFAULT_SELECT_RELATED)
        prefetch_related = filters.pop("_prefetch_related", ())
        qs = Thing.objects.filter(**filters)
        if select_related:
            qs = qs.select_related(*select_related)
        if prefetch_related:
            qs = qs.prefetch_related(*prefetch_related)
        return qs
```

Set a sensible default on the service. Callers override for specialised read paths:

```python
# bll/<Domain>/GetApisbll.py
things = thing_service.filter(
    owner=user,
    _select_related=("owner", "category", "category__group"),
    _prefetch_related=("tags",),
)
```

### Anti-pattern

```python
# BAD — N+1
for thing in thing_service.filter(owner=user):
    serialize_with(thing.category.name, thing.owner.email)   # two queries per row

# GOOD
qs = thing_service.filter(
    owner=user,
    _select_related=("category", "owner"),
)
for thing in qs:
    serialize_with(thing.category.name, thing.owner.email)
```

---

## 2. Bulk operations beat loops

`for x in items: Model.objects.create(...)` is one INSERT per row. `bulk_create(items)` is one INSERT for the whole batch. Same story for updates and deletes.

| Need | Do this | Not this |
|---|---|---|
| Insert N rows | `service.bulk_create([...])` | `for x in xs: service.create(...)` |
| Update N rows in place | `service.bulk_update(rows, fields=["status"])` | `for r in rows: service.update(r, status=...)` |
| Set the same value on many rows | `Model.objects.filter(...).update(status=...)` (in `service.py`) | Fetch → mutate → save in a loop |
| Delete N rows | `service.bulk_delete({"id__in": ids})` | `for id in ids: service.delete(id)` |

### Important caveats

- `bulk_create` does **not** call `save()` or fire `pre_save` / `post_save` signals on PostgreSQL by default. If your model relies on a signal (audit, search-index sync), either drop the signal in favour of explicit calls or batch-and-then-notify.
- `bulk_update` requires existing PKs and the explicit `fields=[...]` list.
- `QuerySet.update()` bypasses `save()` entirely — the rows change in-DB without instances ever being hydrated. Use this when you want a SET on N rows with no Python in the loop.
- Pick a sane `batch_size` for huge inputs (PostgreSQL's parameter limit is 65k):

  ```python
  Thing.objects.bulk_create(instances, batch_size=1000)
  ```

---

## 3. Don't loop the database

The most common slow endpoint shape:

```python
# BAD — one query per id
results = []
for id in ids:
    thing = thing_service.get(id=id)
    if thing.is_active:
        results.append(thing)
```

Pull the whole set with one IN-clause, filter in Python (or in the queryset):

```python
# GOOD — one query
qs = thing_service.filter(id__in=ids, is_active=True)
results = list(qs)
```

### Build lookup dicts once

If you're going to join two collections in Python:

```python
# BAD — O(N²)
for doc in docs:
    matching_owner = next(u for u in users if u.id == doc.owner_id)

# GOOD — O(N)
users_by_id = {u.id: u for u in users}
for doc in docs:
    matching_owner = users_by_id[doc.owner_id]
```

### Use `__in` for set membership

```python
# BAD
existing = set()
for slug in incoming_slugs:
    if thing_service.get(slug=slug) is not None:
        existing.add(slug)

# GOOD
existing = set(
    thing_service.filter(slug__in=incoming_slugs).values_list("slug", flat=True)
)
```

### Use `exists()` for "is there any row?"

```python
# BAD — fetches the row, materialises Python object, throws it away
if thing_service.get(id=thing_id) is not None:
    ...

# Acceptable when you also use the row right after.
# But for a pure existence check, expose a tiny helper on service.py:
class ThingService:
    @staticmethod
    def exists(**filters) -> bool:
        return Thing.objects.filter(**filters).exists()
```

`.exists()` short-circuits at the first row in SQL. Cheaper than `count() > 0` and cheaper than fetching the row.

---

## 4. Compute once, reuse — memoize inside a function

If the same value is computed twice in one request, that's almost always a bug. The fix is usually a one-line variable. The pattern is so common it deserves a name.

### Local variable for repeated reads

```python
# BAD — settings dict is parsed every iteration
for thing in things:
    if vault_settings["FeatureFlags"]["EnableNewThing"]:
        process_new(thing)
    else:
        process_old(thing)

# GOOD — read once, branch outside the loop ideally
enable_new = vault_settings.get("FeatureFlags", {}).get("EnableNewThing", False)
processor = process_new if enable_new else process_old
for thing in things:
    processor(thing)
```

### `functools.lru_cache` for pure, frequently-called helpers

Only when the function is **pure** (same input → same output, no side effects, no DB reads):

```python
from functools import lru_cache

@lru_cache(maxsize=256)
def parse_attribute_path(path: str) -> tuple[str, ...]:
    return tuple(p.strip() for p in path.split("."))
```

Don't `lru_cache` a function that reads the DB — you'll cache stale tenant data across requests.

### `cached_property` for class-level memoization within one instance lifetime

```python
from django.utils.functional import cached_property

class ReportBuilder:
    def __init__(self, user, params):
        self.user = user
        self.params = params

    @cached_property
    def permission_set(self):
        # computed once per builder instance
        return permission_service.list_for_user(self.user)
```

### Django cache framework for cross-request memoization

When the computation is expensive and the result is shared across requests / tenants — and you have a sound invalidation story:

```python
from django.core.cache import cache
from NimbusVaultConstants.CacheKeys import CacheKeys   # use named keys, not strings

def list_categories_for_tenant(org_slug):
    key = CacheKeys.tenant_categories(org_slug)
    cached = cache.get(key)
    if cached is not None:
        return cached
    rows = list(category_service.filter())
    cache.set(key, rows, timeout=300)
    return rows
```

Rules of thumb for Django cache use:

- **Always namespace by tenant.** Caching tenant-scoped data without the org slug in the key leaks data across tenants.
- **Invalidate on write.** Every `PostApisbll` function that changes the underlying data must `cache.delete(...)` the same key.
- **Set a TTL.** Never cache forever unless you have invalidation that you trust completely.
- **Use named cache keys** from `NimbusVaultConstants/` — string-literal keys rot.
- **Don't cache `User`, `request`, or any per-request object.**

---

## 5. Fetch only what you need

| You need | Use |
|---|---|
| One field across many rows | `.values_list("name", flat=True)` |
| A few fields as dicts | `.values("id", "name", "status")` |
| Model instances minus heavy fields | `.only("id", "name", "status")` (be careful — accessing a deferred field triggers a query) |
| Model instances, never the giant TextField | `.defer("body")` |

`.values()` and `.values_list()` skip ORM hydration entirely. Use them for `format_*` helpers that only project a few fields.

```python
# Reporting endpoint, only needs id + name
ids_and_names = thing_service.filter(owner=user).values_list("id", "name")
return [{"id": str(i), "name": n} for i, n in ids_and_names]
```

---

## 6. Annotate at the database, not in Python

`F`, `Count`, `Sum`, `Avg`, `Exists`, `Subquery` push computation into SQL. The DB is much faster at this than Python.

```python
from django.db.models import Count, F, Q

# BAD — fetch all docs, count in Python
docs = list(document_service.filter(owner=user))
counts = {}
for d in docs:
    counts.setdefault(d.category_id, 0)
    counts[d.category_id] += 1

# GOOD — one query, DB does the math
counts = dict(
    document_service
        .filter(owner=user)
        .values_list("category_id")
        .annotate(c=Count("id"))
        .values_list("category_id", "c")
)
```

`F()` is for in-DB updates that depend on the existing column value (e.g., increment a counter atomically):

```python
# In service.py
@staticmethod
def increment_view_count(thing_id: int) -> int:
    return Thing.objects.filter(pk=thing_id).update(view_count=F("view_count") + 1)
```

---

## 7. Wrap multi-row writes in a transaction

If you're writing to multiple tables and any one of them failing should roll back all of them — use a transaction. Combine this with bulk operations.

```python
from django.db import transaction

def import_things(user, rows):
    with transaction.atomic():
        things = thing_service.bulk_create([{"owner": user, **r} for r in rows])
        audit_service.bulk_create([{"thing_id": t.id, "user": user, "action": "import"} for t in things])
    return things
```

Orchestrator steps already give you rollback semantics at the workflow level — `transaction.atomic()` is the SQL-level analogue for a single step that writes to many tables at once.

---

## 8. Existence checks before unique-violation INSERTs

If you can avoid an IntegrityError round-trip, do.

```python
# Bad in a hot loop — one IntegrityError per dup
try:
    thing_service.create(...)
except IntegrityError:
    ...

# Good — one query upfront
existing_slugs = set(
    thing_service.filter(slug__in=new_slugs).values_list("slug", flat=True)
)
to_create = [r for r in rows if r["slug"] not in existing_slugs]
thing_service.bulk_create(to_create)
```

For genuine upsert semantics (PostgreSQL):

```python
Thing.objects.bulk_create(
    instances,
    update_conflicts=True,
    unique_fields=["slug"],
    update_fields=["title", "updated_at"],
)
```

---

## 9. Pagination is not optional

Every list endpoint takes `limit` + `offset` (or a cursor). Cap `limit` in the serializer:

```python
class ListThingsQuerySerializer(serializers.Serializer):
    limit = serializers.IntegerField(min_value=1, max_value=200, default=50)
    offset = serializers.IntegerField(min_value=0, default=0)
```

Slice the queryset, don't fetch and then `[:limit]` in Python:

```python
# BAD — fetches everything
rows = list(qs)[offset : offset + limit]

# GOOD — DB LIMIT/OFFSET
rows = list(qs[offset : offset + limit])
```

For very large tables, prefer cursor pagination (`id__gt=last_id`) over OFFSET — OFFSET still scans all skipped rows.

---

## 10. Stream large reads with `.iterator()`

Default queryset evaluation fetches every matching row into memory. For exports / batch jobs that touch millions of rows:

```python
# In service.py
@staticmethod
def stream(**filters):
    return Thing.objects.filter(**filters).iterator(chunk_size=2000)

# In bll/PostApisbll.py
for thing in thing_service.stream(owner=user):
    write_to_export(thing)
```

`iterator()` opens a server-side cursor and streams rows. Memory stays flat. **Don't** combine it with `prefetch_related` on the same queryset — it disables prefetching.

---

## 11. Background work: pass IDs, not model instances

Celery serialises arguments. Passing a Django model instance pickles a snapshot — stale and brittle. Pass the primary key + the tenant; the task re-fetches with fresh data.

```python
# BAD
do_thing.apply_async(args=[thing, request.user])

# GOOD
do_thing.apply_async(args=[org_slug, request.user.id, thing.id])
```

Inside the task:

```python
@app.task(queue="medium", bind=True)
def do_thing(self, org_slug, user_id, thing_id):
    # 1. switch DB connection to tenant org_slug
    # 2. user = user_service.get(id=user_id)
    # 3. thing = thing_service.get(id=thing_id)
    # 4. orchestrator.run(...)
```

This also dodges a class of bug where a model fields changes between dispatch and execution.

---

## 12. External integrations — HTTP, Git, Elasticsearch, SSE

Anything that crosses a network or process boundary is slower, less reliable, and more failure-prone than a local DB call. The rules below are not optional.

### General rules for every external call

- **Always set timeouts.** Connect + read. A missing `timeout=` on `requests` will hang forever the day the upstream hiccups.

  ```python
  resp = self.session.get(url, headers=..., timeout=(5, 30))   # (connect, read) in seconds
  ```

- **Reuse a session.** A fresh `requests.get()` opens a new TCP+TLS handshake every call. Provider classes under `ExternalTools/` should hold a `requests.Session` (or library equivalent) on `self.session` and reuse it.

- **Retry transient failures with exponential backoff** (5xx, network errors, 429). Don't retry 4xx. Cap total attempts at 3-5, jitter the backoff:

  ```python
  from urllib3.util import Retry
  from requests.adapters import HTTPAdapter

  retry = Retry(total=3, backoff_factor=0.5,
                status_forcelist=(500, 502, 503, 504),
                allowed_methods=frozenset(["GET", "PUT", "DELETE", "HEAD", "OPTIONS"]))
  session.mount("https://", HTTPAdapter(max_retries=retry))
  ```

- **Never block the request thread on a slow external call.** Anything over ~500ms or anything you've seen go to 30s+ in practice belongs in a Celery task. Pass IDs in, fetch in the worker, publish progress via SSE or polling.

- **Never make external calls inside `transaction.atomic()`.** The transaction holds row locks while you wait on the network. Do external work first, capture identifiers, then open the transaction for the local write.

- **Never call externals inside a loop over DB rows.** If you need to enrich 100 records from an external API, look for a batched endpoint; if there isn't one, fan out to Celery.

- **Use the providers under `ExternalTools/`, not ad-hoc `requests.get()` calls.** They centralise auth, retries, logging, and tenant context. Adding a new integration means a new provider class in `ExternalTools/<Capability>/Providers/<Vendor>/`, not a one-off helper.

- **Log every external call** with target URL, HTTP status, latency, and a correlation/request id — at INFO for normal, WARNING for retries, ERROR for definitive failures.

- **Never log secrets, tokens, or full Authorization headers.** Redact in the provider's logging layer once and trust it.

- **Idempotency keys** on any retryable write so the upstream can dedupe replays. Most providers honour `Idempotency-Key`.

- **Circuit-breaker** repeated failures (e.g., 5 failures in 60s) to fail fast and not pile up worker time during an outage. Open the breaker, return a typed error from `errors.*`, and let the caller decide.

### Gitea — `ExternalTools/VaultFileManagement/Providers/Gitea/`

- **Use the provider class** (`Gitea` in `ExternalTools/VaultFileManagement/Providers/Gitea/Main.py`), not direct `requests` against the Gitea host.
- **Auth lives in `VaultSettings.json`** under `GITEA_SETTINGS`. Token rotation is a config change, not a code change.
- **Don't pull file contents when you only need metadata.** Gitea exposes file-info / tree endpoints that don't ship the blob.
- **Cache repo + branch metadata aggressively.** Repo names, default branches, and webhook URLs change rarely. Cache key namespaced by tenant; invalidate on the matching write path.
- **Use the batched endpoints** Gitea offers (commits list, contents at ref) instead of N single-file calls. If you find yourself iterating commits to count them, ask Gitea once.
- **Webhooks for change notifications, not polling.** When NimbusVault needs to react to a Gitea event, register a webhook and dispatch into the orchestrator from the webhook view. Polling Gitea every 30 seconds is a config smell.

### PyGit — `ExternalTools/VaultFileManagement/Providers/PyGit/`

- **Reuse the repository handle.** Opening a `pygit2.Repository` is non-trivial; keep one per request (or per task), not one per file read.
- **Never `git clone` a full history when you need a single file at a single ref.** Use a partial / shallow / sparse clone, or read the blob directly via the API on the remote when possible.
- **Walk by pathspec, not by enumerating every commit.** `repo.walk(...)` with the right filters runs in seconds; iterating every commit to inspect each one runs in minutes.
- **Close / dispose handles in `try/finally`.** Leaked handles in a long-running celery worker exhaust file descriptors over hours, not seconds — perfect for a Friday-evening incident.
- **Don't hold a working tree open across an external HTTP call.** Read what you need, close, then do the network call.

### Elasticsearch — `ExternalTools/VaultCacheStorage/Providers/ElasticProvider/`

Elasticsearch is fast when used correctly and the slowest thing in your trace when used wrong.

- **Tenant-namespace every index.** Indices follow `{tenant}_{logical_index_name}`. Never query a bare logical index name; the request will leak across tenants or fail outright.
- **Use the bulk API for ingest.** One `_bulk` request with N docs beats N single `_doc` requests by orders of magnitude — and Elasticsearch will rate-limit you for the latter anyway.
- **Filter context > query context** for matches that don't need a score. Filters are cached and faster:

  ```python
  # Good — bool/filter
  {"query": {"bool": {"filter": [{"term": {"status": "active"}},
                                  {"term": {"owner_id": user_id}}]}}}

  # Bad — bool/must on terms with no scoring value
  {"query": {"bool": {"must":   [{"term": {"status": "active"}},
                                  {"term": {"owner_id": user_id}}]}}}
  ```

- **Use `search_after` (cursor) for deep pagination.** `from + size` past ~10k entries is a hard error. The query-builder helpers under `ExternalTools/.../ElasticProvider/Queries/` already support cursors — use them.
- **Avoid leading wildcards** (`*foo`). They bypass the inverted index and scan every term. If you need substring search, model the field as `ngram` at index time and query exact tokens at search time.
- **Set a sensible refresh policy.** Default refresh of one second is fine for read-heavy paths; for high-volume ingest, set `refresh=False` (or `wait_for` only when the caller really needs read-your-writes).
- **Cap `size` in the query builder.** Never pass user-supplied `size` straight through; clamp it to the same bound as `limit` in the serializer.
- **Aggregate in Elasticsearch, not in Python.** `aggs` runs on the cluster across shards; fetching 10k docs and counting in Python ships 10k JSON objects over the wire to do something the cluster could finish in 50ms.
- **Don't index PII into searchable fields** unless the index has the right access controls. Use `index: false` for fields you only need to retrieve.

### SSE — `ExternalTools/SSE/`

Server-Sent Events are a streaming response held open for minutes or hours. They are nothing like a normal request, and most of the standard practices invert.

- **Don't hold a DB connection open for the lifetime of the stream.** Open per heartbeat, fetch, close. Otherwise one user with a flaky network ties up a connection in the pool indefinitely.
- **Heartbeat every 15-30 seconds.** Middleboxes (proxies, load balancers, mobile carrier NATs) silently drop idle TCP connections. Send a comment line:

  ```
  :ping\n\n
  ```

- **Required response headers** on the streaming view:

  ```
  Content-Type:        text/event-stream
  Cache-Control:       no-cache
  X-Accel-Buffering:   no       # disable nginx proxy buffering
  Connection:          keep-alive
  ```

- **Use `Last-Event-ID`.** Tag each event with a monotonic `id:` line; on reconnect the client sends `Last-Event-ID` and the server resumes from there. Skip this and you lose events on every network blip.
- **Push, don't poll.** The SSE view subscribes to a Redis pub/sub channel (or equivalent); the actual work runs in a Celery task that publishes to that channel. The view loops over `subscriber.listen()` and writes each event to the response. This keeps the view thin and the worker bounded.
- **Bound the per-user stream count.** A misbehaving client that opens 100 connections must not exhaust your worker pool — track open streams per user and reject above a threshold.
- **Never embed secrets, tokens, or PII in event payloads.** SSE events are plaintext over HTTP for the wire (HTTPS) and plaintext in browser memory; minimise what you ship.
- **Tenant-route inside the streaming view** the same way as a normal request. SSE responses live under the same middleware chain — the `Org:` header still determines DB routing.
- **Close the stream on tenant/permission change.** If the user's session is revoked mid-stream, the next heartbeat should send a terminal event and close. Don't wait for the client to notice.

---

## 13. Indexes

Add indexes for the columns you `.filter()` on most. Common patterns:

```python
class Thing(models.Model):
    owner = models.ForeignKey(User, on_delete=models.CASCADE)
    slug = models.SlugField(max_length=255, db_index=True)
    status = models.CharField(max_length=32)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["owner", "-created_at"]),   # list-by-owner ordered by recency
            models.Index(fields=["status", "owner"]),        # workflow dashboards
        ]
        unique_together = [("owner", "slug")]
```

FK columns get an index by default; the unique-together pair gets a composite index. Add explicit composite indexes when a list endpoint filters by two columns together.

Don't over-index — every index slows writes and bloats storage. Add the obvious ones, then add more based on slow-query logs.

---

## 14. Verify with `assertNumQueries`

The only way to be sure a read path doesn't have N+1 is to count queries in a test.

```python
from django.test.utils import CaptureQueriesContext
from django.db import connection

def test_list_things_is_constant_query_count(authed_client):
    # Seed 50 things
    seed_things(50)

    with CaptureQueriesContext(connection) as ctx:
        resp = authed_client.get("/vault/things/getAll?limit=50")

    assert resp.status_code == 200
    # 1 count query + 1 list query + 1 select_related JOIN = 3.
    # If we add a row, query count must NOT grow.
    assert len(ctx.captured_queries) <= 4, [q["sql"] for q in ctx.captured_queries]
```

Run this test once and pin the upper bound. It's the cheapest insurance against future N+1 regressions.

---

## 15. Small Python ergonomics that add up

| Don't | Do |
|---|---|
| `if len(items) > 0:` | `if items:` |
| `for i in range(len(xs)): xs[i].x = ...` | `for x in xs: x.x = ...` |
| Mutating a list while iterating it | Build a new list |
| `dict(zip(keys, [None] * len(keys)))` | `dict.fromkeys(keys)` |
| Concatenating large strings with `+=` in a loop | `"".join(parts)` |
| Catching `Exception` for control flow | Catch the specific class; let unknown errors propagate |
| `pickle` of untrusted data | Never |
| `eval` / `exec` on request data | Never |

---

## 16. Quick-reference checklist

Before merging a new read endpoint:

- [ ] `select_related` for every FK you touch in serialisation.
- [ ] `prefetch_related` for every reverse FK / M2M you touch.
- [ ] List endpoint has `limit` + `offset` (or cursor) capped in the serializer.
- [ ] `exists()` used for existence checks; `values_list()` used for one-field reads.
- [ ] `assertNumQueries` test pinned for the happy path.

Before merging a new write endpoint:

- [ ] Multi-row inserts use `bulk_create`.
- [ ] Multi-row updates use `bulk_update` or `QuerySet.update()`.
- [ ] Multi-table writes wrapped in `transaction.atomic()` (or a single orchestrator step that rolls back).
- [ ] Celery tasks receive IDs + tenant, not model instances.
- [ ] Cache invalidation `cache.delete(...)` matches every cache key the read paths write.

Before merging any code that talks to an external system:

- [ ] All external calls go through a provider under `ExternalTools/`, never raw `requests.*` in the BLL.
- [ ] Explicit `timeout=(connect, read)` on every HTTP call.
- [ ] Retries with backoff for 5xx / network errors only; 4xx never retried.
- [ ] No external call inside a `transaction.atomic()` block.
- [ ] No external call inside a `for` loop over DB rows.
- [ ] Slow / unbounded calls live in a Celery task, not the request thread.
- [ ] Elasticsearch queries are tenant-namespaced, use the bulk API for ingest, and use `search_after` for deep paging.
- [ ] SSE views send heartbeats, set `Cache-Control: no-cache` + `X-Accel-Buffering: no`, and back the stream by Redis pub/sub rather than polling.
- [ ] Logs include URL + status + latency + correlation id; logs do not include tokens or secrets.

If any of these don't apply, say so in the PR description — silence reads as "I forgot."
