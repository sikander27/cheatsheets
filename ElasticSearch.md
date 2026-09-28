# Elasticsearch Cheat Sheet

Commands are shown in Kibana Dev Tools format (`METHOD /path` + JSON body). For curl, prefix with `curl -X METHOD "http://localhost:9200/path" -H 'Content-Type: application/json' -d '...'`.

---

## 1. Core Concepts

| Term | Meaning |
|---|---|
| **Document** | One JSON record (like a row) |
| **Index** | A collection of similar documents (like a table) |
| **Shard (primary)** | A slice of an index; each document lives in exactly one primary |
| **Replica** | A copy of a primary shard, always on a different node |
| **Node** | One running Elasticsearch instance |
| **Cluster** | A group of nodes sharing one cluster name |
| **Mapping** | The schema: field names and types |
| **Analyzer** | Turns text into searchable tokens |
| **Inverted index** | Lookup structure inside each shard: term -> documents |

### Node roles

| Role | Purpose |
|---|---|
| `master` | Manages cluster state, decides shard placement |
| `data` (`data_hot`, `data_warm`, `data_cold`) | Stores shards, runs searches and indexing |
| `ingest` | Pre-processes documents with pipelines |
| `coordinating` (no roles) | Routes requests, merges results |
| `ml`, `transform` | Machine learning / transforms |

### Health colors

| Color | Meaning |
|---|---|
| Green | All primaries and replicas assigned |
| Yellow | All primaries assigned, some replicas unassigned |
| Red | At least one primary unassigned (data unavailable) |

---

## 2. Shard and Replica Math

| What | Formula |
|---|---|
| Total shards | `primaries x (1 + replicas)` |
| Docs per shard | `docs.count / primaries` |
| Real data size | `store.size / (1 + replicas)` |
| Disk needed | `data size x (1 + replicas)` + 20-30% headroom |
| Min nodes for green | `1 + replicas` |
| Shard for a doc | `hash(_routing or _id) % primaries` |
| Ideal shards per node | `total shards / nodes` |

Rules of thumb:

- Aim for **10-50 GB per shard**. Search-heavy workloads often prefer the lower end, log workloads the higher end.
- Keep shards under roughly **200 million documents**.
- Avoid too many small shards: each has overhead. Keep the shard count per node reasonable (a common guideline is up to about 20 shards per GB of heap).
- **Primary count is fixed** at index creation (change via reindex, `_split` or `_shrink`). **Replica count can change any time.**
- `docs.count` counts primaries only. `store.size` includes replicas (`pri.store.size` is primaries only).
- A search fans out to **every shard**, uses **one copy of each**, and is only as fast as the slowest shard.
- Lookup by `_id` (or custom routing) hits one shard. Search by content hits all shards.
- Replicas give both **high availability** and **extra read throughput**.

---

## 3. Cat APIs (quick cluster inspection)

Add `?v` for headers, `?help` to list columns, `&s=column:desc` to sort, `&h=col1,col2` to pick columns.

```
GET _cat/health?v
GET _cat/nodes?v&h=name,ip,node.role,cpu,load_1m,heap.percent,disk.used_percent
GET _cat/indices?v&s=store.size:desc
GET _cat/shards?v
GET _cat/shards/my-index?v&h=index,shard,prirep,state,docs,store,node
GET _cat/allocation?v
GET _cat/aliases?v
GET _cat/templates?v
GET _cat/pending_tasks?v
GET _cat/thread_pool?v&h=node_name,name,active,queue,rejected
GET _cat/recovery?v&active_only=true
GET _cat/segments/my-index?v
GET _cat/count/my-index?v
```

---

## 4. Cluster APIs

```
GET _cluster/health
GET _cluster/health?level=indices
GET _cluster/state
GET _cluster/stats
GET _cluster/settings?include_defaults=true
GET _nodes/stats
GET _nodes/hot_threads
GET _nodes/stats/jvm,os,fs
```

Update a persistent cluster setting:

```
PUT _cluster/settings
{
  "persistent": { "cluster.routing.allocation.enable": "all" }
}
```

---

## 5. Index Management

### Create an index with settings and mappings

```
PUT my-index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s"
  },
  "mappings": {
    "properties": {
      "title":      { "type": "text" },
      "status":     { "type": "keyword" },
      "salary":     { "type": "integer" },
      "created_at": { "type": "date" },
      "location":   { "type": "geo_point" },
      "skills":     { "type": "keyword" }
    }
  }
}
```

### Common operations

```
GET my-index
GET my-index/_settings
GET my-index/_mapping
GET my-index/_stats
DELETE my-index
POST my-index/_close
POST my-index/_open
```

### Change replicas (allowed any time)

```
PUT my-index/_settings
{ "number_of_replicas": 2 }
```

### Add a field to a mapping

```
PUT my-index/_mapping
{ "properties": { "email": { "type": "keyword" } } }
```

Existing fields cannot change type; create a new index and reindex.

### Aliases (zero-downtime swaps)

```
POST _aliases
{
  "actions": [
    { "remove": { "index": "my-index-v1", "alias": "my-index" } },
    { "add":    { "index": "my-index-v2", "alias": "my-index" } }
  ]
}
```

### Reindex

```
POST _reindex
{
  "source": { "index": "my-index-v1" },
  "dest":   { "index": "my-index-v2" }
}
```

Add `?wait_for_completion=false` for large jobs, then check `GET _tasks/<task_id>`.

### Split / shrink / rollover

```
POST my-index/_split/my-index-split   { "settings": { "index.number_of_shards": 6 } }
POST my-index/_shrink/my-index-small  { "settings": { "index.number_of_shards": 1 } }
POST my-alias/_rollover               { "conditions": { "max_size": "50gb", "max_age": "30d" } }
```

The source index must be read-only (`index.blocks.write: true`) for split and shrink; the target shard count must be a multiple/factor of the source.

---

## 6. Document CRUD

```
PUT  my-index/_doc/1          { "title": "Backend Engineer", "status": "open" }   # create/replace with ID
POST my-index/_doc            { "title": "Data Analyst" }                          # auto ID
GET  my-index/_doc/1
GET  my-index/_source/1
HEAD my-index/_doc/1                                                               # exists?
POST my-index/_update/1       { "doc": { "status": "closed" } }                    # partial update
DELETE my-index/_doc/1
POST my-index/_update_by_query { "query": { "term": { "status": "open" } }, "script": "ctx._source.status='closed'" }
POST my-index/_delete_by_query { "query": { "term": { "status": "closed" } } }
```

### Bulk API (newline-delimited JSON, each line ends with `\n`)

```
POST _bulk
{ "index":  { "_index": "my-index", "_id": "1" } }
{ "title": "Backend Engineer" }
{ "update": { "_index": "my-index", "_id": "1" } }
{ "doc": { "status": "closed" } }
{ "delete": { "_index": "my-index", "_id": "2" } }
```

### Multi-get

```
GET _mget
{ "docs": [ { "_index": "my-index", "_id": "1" }, { "_index": "my-index", "_id": "2" } ] }
```

---

## 7. Field Types

| Type | Use |
|---|---|
| `text` | Full-text search (analyzed) |
| `keyword` | Exact match, sorting, aggregations (not analyzed) |
| `integer`, `long`, `float`, `double`, `scaled_float` | Numbers |
| `boolean` | true/false |
| `date` | Dates (ISO 8601 or epoch) |
| `object` | JSON object (flattened internally) |
| `nested` | Array of objects that keep field relationships |
| `geo_point`, `geo_shape` | Geo data |
| `ip` | IPv4/IPv6 |
| `dense_vector` | Embeddings for vector search |
| `completion` | Autocomplete suggester |
| `join` | Parent/child relations |

Multi-field example (text for search, keyword for sort/aggs):

```
"name": { "type": "text", "fields": { "raw": { "type": "keyword" } } }
```

**text vs keyword:** use `text` when you want word-level search ("senior java developer" matches "java"). Use `keyword` for IDs, enums, tags, exact filters and aggregations.

---

## 8. Search Basics

```
GET my-index/_search
{
  "query": { "match": { "title": "backend engineer" } },
  "_source": ["title", "status"],
  "sort": [ { "created_at": "desc" } ],
  "from": 0,
  "size": 20
}
```

Useful options: `"track_total_hits": true`, `"timeout": "2s"`, `"explain": true`, `"profile": true`.

### Query context vs filter context

| Context | Scores? | Cached? | Use for |
|---|---|---|---|
| `must`, `should` (query) | Yes | No | Relevance ranking |
| `filter`, `must_not` | No | Yes | Yes/no conditions (status, ranges, IDs) |

Put exact-match conditions in `filter` for speed.

---

## 9. Query DSL

### Full-text

```
{ "match":        { "title": "backend engineer" } }
{ "match_phrase": { "title": "backend engineer" } }
{ "multi_match":  { "query": "python", "fields": ["title^3", "skills", "description"] } }
{ "query_string": { "query": "title:(java OR python) AND status:open" } }
```

### Term-level (exact, not analyzed)

```
{ "term":   { "status": "open" } }
{ "terms":  { "status": ["open", "paused"] } }
{ "range":  { "salary": { "gte": 1000, "lt": 5000 } } }
{ "exists": { "field": "email" } }
{ "prefix": { "title.raw": "Back" } }
{ "wildcard": { "title.raw": "*Engineer*" } }   # slow on large data
{ "ids":    { "values": ["1", "2", "3"] } }
```

Do not use `term` on an analyzed `text` field; use `match`, or a `keyword` sub-field.

### Bool (combine anything)

```
GET my-index/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "title": "engineer" } } ],
      "filter":   [ { "term": { "status": "open" } },
                    { "range": { "salary": { "gte": 1000 } } } ],
      "should":   [ { "term": { "skills": "python" } } ],
      "must_not": [ { "term": { "status": "spam" } } ],
      "minimum_should_match": 0
    }
  }
}
```

### Others

```
{ "match_all": {} }
{ "fuzzy": { "title": { "value": "enginer", "fuzziness": "AUTO" } } }
{ "nested": { "path": "experience", "query": { "term": { "experience.company": "Acme" } } } }
{ "function_score": { "query": { "match_all": {} }, "field_value_factor": { "field": "popularity" } } }
{ "geo_distance": { "distance": "10km", "location": { "lat": 19.07, "lon": 72.87 } } }
```

### Pagination

| Method | Use when |
|---|---|
| `from` + `size` | Shallow pages (default max `from + size` = 10,000) |
| `search_after` | Deep pagination, needs a stable sort (add a unique tiebreaker like `_id`) |
| Point in Time (PIT) + `search_after` | Consistent deep pagination |
| Scroll | Bulk export / reindex-style jobs |

---

## 10. Aggregations

```
GET my-index/_search
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": { "field": "status", "size": 10 },
      "aggs": { "avg_salary": { "avg": { "field": "salary" } } }
    },
    "salary_stats": { "stats": { "field": "salary" } },
    "per_month": { "date_histogram": { "field": "created_at", "calendar_interval": "month" } },
    "salary_buckets": { "range": { "field": "salary", "ranges": [ { "to": 1000 }, { "from": 1000, "to": 5000 }, { "from": 5000 } ] } },
    "unique_users": { "cardinality": { "field": "user_id" } }
  }
}
```

| Family | Examples |
|---|---|
| Bucket | `terms`, `range`, `date_histogram`, `histogram`, `filters`, `nested` |
| Metric | `avg`, `sum`, `min`, `max`, `stats`, `cardinality`, `percentiles` |
| Pipeline | `bucket_sort`, `cumulative_sum`, `moving_avg`, `derivative` |

Aggregate on `keyword`/numeric/date fields, not `text`.

---

## 11. Analysis (text processing)

An analyzer = **char filters -> tokenizer -> token filters**.

```
GET _analyze
{ "analyzer": "standard", "text": "The Quick Brown Foxes" }
```

Custom analyzer example:

```
PUT my-index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "my_synonyms"]
        }
      },
      "filter": {
        "my_synonyms": { "type": "synonym", "synonyms": ["js, javascript", "k8s, kubernetes"] }
      }
    }
  },
  "mappings": { "properties": { "title": { "type": "text", "analyzer": "my_analyzer" } } }
}
```

Common analyzers: `standard`, `simple`, `whitespace`, `keyword`, `english` (and other languages).

---

## 12. Shard Allocation and Troubleshooting

### Why is a shard unassigned?

```
GET _cluster/allocation/explain
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state
```

| Symptom | Common causes |
|---|---|
| Yellow on a single node | Replicas cannot live on the same node as the primary. Add a node or set `number_of_replicas: 0` |
| Unassigned after node loss | Node left; wait for recovery or check allocation settings |
| Shards not moving | Disk watermarks hit, allocation filtering, `allocation.enable` disabled |
| Red index | A primary is unassigned; check `allocation/explain` |

### Disk watermarks (defaults)

| Setting | Default | Effect |
|---|---|---|
| Low | 85% | No new shards allocated to the node |
| High | 90% | Tries to move shards off the node |
| Flood stage | 95% | Indices become read-only (`read_only_allow_delete`) |

Clear a flood-stage block after freeing space:

```
PUT _all/_settings
{ "index.blocks.read_only_allow_delete": null }
```

### Control allocation

```
PUT my-index/_settings
{ "index.routing.allocation.total_shards_per_node": 4 }

PUT _cluster/settings
{ "persistent": { "cluster.routing.rebalance.enable": "all" } }

POST _cluster/reroute?retry_failed=true
```

### Rolling restart routine

```
PUT _cluster/settings   { "persistent": { "cluster.routing.allocation.enable": "primaries" } }
POST _flush
# restart node, wait for it to rejoin
PUT _cluster/settings   { "persistent": { "cluster.routing.allocation.enable": null } }
GET _cluster/health?wait_for_status=green&timeout=60s
```

### Slow cluster checklist

1. `GET _cat/nodes?v` -> any node with high CPU, heap or load? (Hot node?)
2. `GET _cat/shards?v` -> are busy indices concentrated on a few nodes?
3. `GET _cat/thread_pool?v` -> rejections in `search` or `write`?
4. `GET _nodes/hot_threads` -> what is the CPU doing?
5. Slow log / `"profile": true` -> which queries are expensive?
6. Are shards too big, too small, or too many? Is heap pressure above 75%?
7. Are metrics monitored **per node**, not just cluster-wide?

---

## 13. Performance Tips

### Search

- Use `filter` context for exact conditions; results are cached.
- Return only what you need (`_source` filtering, small `size`).
- Avoid leading wildcards, big `terms` lists and deep `from` pagination.
- Prefer `keyword` fields for filters and aggregations.
- Add replicas to scale read throughput (and spread copies evenly across nodes).
- Use `preference` (e.g. a session ID) for consistent shard-copy selection and better cache use.
- Use `force_merge` only on read-only indices.

### Indexing

- Use the **Bulk API** (roughly 5-15 MB per request is a good start).
- During big loads: set `refresh_interval` to `30s` or `-1`, and `number_of_replicas` to `0`, then restore after.
- Let Elasticsearch auto-generate IDs if you do not need custom ones.
- Avoid unnecessary fields; use `"index": false` or `"enabled": false` for fields you never search.
- Turn `dynamic` mapping to `strict` or `false` to prevent mapping explosion.

### JVM and hardware

- Heap: **at most 50% of RAM**, and below about 31 GB (compressed oops). The rest is used by the OS file cache.
- Use SSDs; avoid swapping (disable swap or `bootstrap.memory_lock: true`).
- Keep heap usage under about 75%.

---

## 14. Index Lifecycle and Templates

### Index template

```
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
    "mappings": { "properties": { "@timestamp": { "type": "date" } } }
  }
}
```

### ILM policy (hot -> warm -> delete)

```
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_size": "50gb", "max_age": "7d" } } },
      "warm":   { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 }, "forcemerge": { "max_num_segments": 1 } } },
      "delete": { "min_age": "30d", "actions": { "delete": {} } }
    }
  }
}
```

---

## 15. Snapshots and Backup

```
PUT _snapshot/my_repo
{ "type": "fs", "settings": { "location": "/mnt/backups" } }

PUT _snapshot/my_repo/snap_1?wait_for_completion=true
GET _snapshot/my_repo/_all
POST _snapshot/my_repo/snap_1/_restore
{ "indices": "my-index", "rename_pattern": "(.+)", "rename_replacement": "restored_$1" }
```

On AWS/OpenSearch Service, repositories usually point to S3 and need registration with a signed request.

---

## 16. Security and Misc

```
GET _security/_authenticate
GET _license
GET /                      # version and cluster info
GET _cat/plugins?v
```

- Never expose port 9200 to the public internet without authentication and TLS.
- Use role-based access control and API keys for applications.

---

## 17. Quick Decision Guide

| Question | Answer |
|---|---|
| Small index (a few GB or less)? | 1 primary shard |
| Small, read-heavy index on N nodes? | Replicas = N - 1 (full copy on every node) |
| Large index? | Total size / target shard size (10-50 GB) = primaries |
| Need to change primary count? | Reindex, `_split` or `_shrink` |
| Need exact match / sort / aggregate? | `keyword` (or numeric/date) |
| Need word-level search? | `text` |
| Need both? | `text` with a `keyword` sub-field |
| Exact match in a query? | `filter` + `term` |
| Deep pagination? | `search_after` with PIT |
| Time-series data? | Data streams / rollover + ILM |
| Slow queries on a few nodes only? | Check per-node CPU and shard distribution |

---

## 18. Health Check One-Liner Routine

```
GET _cat/health?v
GET _cat/nodes?v&h=name,cpu,load_1m,heap.percent,disk.used_percent
GET _cat/indices?v&s=store.size:desc
GET _cat/shards?v&s=node
GET _cat/allocation?v
GET _cat/thread_pool/search,write?v&h=node_name,name,active,queue,rejected
```

Read them in order: cluster color -> which node is hot -> which index is big -> which shards live where -> are queues rejecting work.
