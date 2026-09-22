# Mosaic: Typed Numeric Values — Three Implementation Approaches

Companion doc to RFC #70 (Typed Numeric Values & Versioned Object Contracts). All three approaches satisfy the RFC's two goals — typed numeric envelopes with precision-safe string transport, and stable versioned object discriminators for SDK routing — but they differ in *where* the type information lives.

In the annotated examples below: `// NEW` marks fields this RFC introduces; `// unchanged` marks fields that already existed in the current untyped payloads.

---

## Proposal A: Per-value inline envelope

Every numeric value is replaced with a small object carrying its own type metadata, inline, at the point of use.

```jsonc
{
  "refId": "annualizedPerformance",   // unchanged — already identifies this metric
  "metric": "returns-agg",            // unchanged
  "data": {
    "items": [
      {
        "date": "2026-05-31",         // unchanged — already a plain date string
        "aggregates": [
          {
            "refId": "3-Month",       // unchanged — already identified each aggregate
            "value": {                // NEW — value is now an object, not a raw number
              "object": "percentMetric",     // NEW — type discriminator for SDK routing
              "dataType": "numeric",          // NEW — base JSON type
              "format": "percent",            // NEW — semantic format (was previously implicit)
              "value": "0.0070878897"         // NEW — the actual number, moved here and stringified for precision
            }
          }
        ]
      }
    ]
  }
}
```

### Pros
- Self-describing at the point of use — inspect any single value in isolation and know exactly what it is.
- Simplest mental model: no separate lookup step.

### Cons
- **JSON bloat**: `object`, `dataType`, `format` repeat on every value. A 250-row time series repeats the same three keys 250 times for a field whose type never changes. This is what Rich Weston flagged on the RFC thread.

---

## Proposal B: Schema-once, inline in the payload

Type information is declared a single time per bundle (in a `schema` block that still ships as part of the response); `items` keep their existing key names but carry lean values.

```jsonc
{
  "refId": "annualizedPerformance",   // unchanged
  "metric": "returns-agg",            // unchanged
  "schema": {                          // NEW — type contract declared once for the whole bundle, shipped inline
    "schemaId": "compound-return.v1", // NEW — versioned discriminator (replaces per-value "object")
    "fields": {
      "value": {                       // NEW — declares what "value" means everywhere below, once
        "object": "percentMetric",
        "dataType": "numeric",
        "format": "percent"
      }
    },
    "context": { "isAnnualized": true } // NEW — context constant for the whole response, hoisted out of every row
  },
  "data": {
    "items": [
      {
        "date": "2026-05-31",          // unchanged
        "aggregates": [
          {
            "refId": "3-Month",        // unchanged
            "periodSlug": "3M",        // unchanged — required context, varies per row so stays here
            "startDate": "2026-02-28", // unchanged — varies per row, stays here
            "endDate": "2026-05-31",   // unchanged — varies per row, stays here
            "value": "0.0070878897"    // NEW shape, unchanged content — was a raw number, now a precision-safe string; type resolved via schema.fields, not repeated
          }
        ]
      }
    ]
  }
}
```

### Pros
- Type metadata paid for once per bundle, not once per row — directly resolves the bloat concern.
- `schemaId` doubles as the RFC's versioned discriminator without re-tagging every value.
- No external dependency — the response is fully self-contained; a client can parse it with nothing but the payload itself.

### Cons
- The `schema` block still ships on every response, every time — for a bundle with many small items, that's fixed overhead paid repeatedly across requests (just not repeated *within* a single response anymore).
- No shared, centrally-versioned source of truth across bundles — if ten different bundles all use `percentMetric`, its definition (`dataType`, `format`, pattern) is duplicated across ten `schema.fields` blocks instead of living in one place.
- Harder to validate in CI or generate client models from, since the "schema" is just payload data, not a real JSON Schema document with `$id`/`$ref` semantics.

---

## Proposal C: Decoupled — external `.schema.json`

The `schema.fields` block is pulled out of the payload entirely into a real, standalone JSON Schema file. The response carries only `schemaId` as a pointer; the definition lives outside the response and is fetched/cached separately (or shipped once at codegen/build time).

### Step 1 — shared type vocabulary, defined once, referenced by every bundle

`types.schema.json` — maps directly to the `proposedObject` column in the Excel matrix:

```jsonc
{
  "$schema": "https://json-schema.org/draft/2020-12/schema", // NEW — standard JSON Schema envelope
  "$id": "https://mosaic.dfa.com/schemas/types.schema.json",  // NEW — stable, citable identity for this vocabulary
  "$defs": {
    "preciseCurrencyMetric": {                // NEW — corresponds to Excel's proposedObject = preciseCurrencyMetric
      "type": "string",
      "description": "Decimal currency value transported as string to preserve precision",
      "x-dataType": "numeric",
      "x-format": "decimal",
      "pattern": "^-?[0-9]+\\.[0-9]+$"
    },
    "percentMetric": {                        // NEW — corresponds to Excel's proposedObject = percentMetric
      "type": "string",
      "x-dataType": "numeric",
      "x-format": "percent",
      "pattern": "^-?[0-9]+\\.[0-9]+$"
    }
  }
}
```

### Step 2 — one contract per bundle, referencing the shared vocabulary

`asset-metric.v1.schema.json`:

```jsonc
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://mosaic.dfa.com/schemas/asset-metric.v1.schema.json", // NEW — this IS the versioned discriminator
  "title": "Asset Metric v1",
  "type": "object",
  "properties": {
    "date": { "type": "string", "format": "date" },                    // unchanged concept, now formally typed
    "aum": { "$ref": "types.schema.json#/$defs/preciseCurrencyMetric" } // NEW — points at the shared primitive instead of redefining it
  },
  "required": ["date", "aum"],
  "x-requiredContext": ["isoCurrencyCode"]    // NEW — documents what B would have put in schema.context
}
```

### Step 3 — the runtime payload carries only the pointer

```jsonc
{
  "refId": "aumHistory",                // unchanged
  "metric": "asset-agg",                // unchanged
  "schemaId": "asset-metric.v1",        // NEW — pointer only; no inline "fields" block at all
  "context": { "isoCurrencyCode": "USD" }, // NEW — still needed at runtime since it's an actual value, not a type definition
  "data": {
    "items": [
      { "date": "2026-07-31", "aum": "48213904221.55" } // unchanged keys, new precision-safe string value
    ]
  }
}
```

### Pros
- **Single source of truth**: `preciseCurrencyMetric` is defined exactly once across the entire API, referenced by every bundle that uses it — no duplication across dozens of `schema.fields` blocks.
- **Real tooling**: CI can validate every response against its schema; SDKs can be generated directly from the `.schema.json` files (e.g. via `json-schema-to-pydantic`), instead of hand-maintaining parsers.
- **Smallest runtime payload** of the three — no type metadata ships in the response body at all, only the pointer plus whatever context values are genuinely data (like `isoCurrencyCode: "USD"`).
- **True versioning**: a breaking change ships as `asset-metric.v2.schema.json` with a new `$id`; old clients pinned to v1 keep working unaffected.

### Cons
- Requires hosting/serving the schema files somewhere stable (`mosaic.dfa.com/schemas/...` or repo-relative) — an infrastructure decision the other two proposals don't need.
- A client that only has the payload (no access to the schema registry) can't interpret it standalone — there's a hard dependency on fetching or bundling the schema separately.
- `schemaId` becomes a de facto public contract the moment external clients start caching against it, so it needs the same rigor as any other public API surface (deprecation policy, hosting uptime, etc.).

---

## Recommendation

Proposal C is the strongest long-term direction: it fully resolves the bloat concern, eliminates duplicated type definitions across bundles, and turns "the schema" into an actual asset the team can validate against and generate code from — which is what Kamron's original comment (linking `json-schema-to-pydantic`) was pointing at. Proposal B is a reasonable interim step if standing up schema hosting isn't feasible yet, since it solves the row-level bloat immediately with zero new infrastructure. Proposal A should be avoided for any bundle with meaningful row counts (which is most of what DD Lens serves).
