# Infinispan Query Skill - Verification Checklist

Use this checklist to verify that query and search advice is correct before presenting it to the user.

## Entity Mapping Checks

- [ ] Entity class is annotated with `@Indexed`
- [ ] Field annotations match the query type:
  - `@Basic` for numbers, dates, short exact-match strings
  - `@Text` for full-text searchable text (analyzer required, default `standard`)
  - `@Keyword` for non-tokenized strings (supports normalizer, sortable)
  - `@Decimal` for BigDecimal/BigInteger fields
  - `@Vector` for vector embeddings (dimension is required)
  - `@GeoField` or `@GeoPoint` for spatial fields
  - `@Embedded` for nested entities
- [ ] `sortable = true` is set on fields used in ORDER BY
- [ ] `projectable = true` is set on fields used in SELECT projections
- [ ] `aggregable = true` is set on fields used in GROUP BY aggregations
- [ ] Vector fields have correct `dimension` matching the embedding model output size
- [ ] Spatial fields use the correct `LatLng` import:
  - Embedded: `org.infinispan.api.annotations.indexing.model.LatLng`
  - Remote: `org.infinispan.commons.api.query.geo.LatLng`

## Indexing Configuration Checks

- [ ] Cache encoding is `application/x-protostream` (required for remote; recommended for embedded indexed caches)
- [ ] `<indexed-entities>` lists all entity types to be indexed
- [ ] Index storage is appropriate: `local-heap` (dev/small) or `filesystem` (production/large)
- [ ] For remote caches, Protobuf schema is registered on the server

## Ickle Query Checks

- [ ] Entity name is correct:
  - Remote: Protobuf package + message name (e.g., `book_sample.Book`)
  - Embedded: fully qualified Java class name (e.g., `org.example.Book`)
- [ ] Full-text operator `:` is only used on `@Text` or `@Keyword` fields
- [ ] Comparison operators (`=`, `>`, `<`, etc.) are used on `@Basic` or `@Decimal` fields
- [ ] String literals are enclosed in single quotes
- [ ] Named parameters use `:paramName` syntax and are set via `query.setParameter()`

## Vector Search Checks

- [ ] Only one vector predicate (`<->`) per query
- [ ] Query vector dimension matches `@Vector(dimension = N)`
- [ ] Similarity metric is appropriate for the use case
- [ ] `filtering` clause (if used) does not contain another kNN predicate
- [ ] Vector values are `float[]` or `byte[]`

## Spatial Search Checks

- [ ] Cache is indexed (spatial queries require indexing)
- [ ] `@GeoPoint` fields have matching `fieldName` on `@Latitude` and `@Longitude`
- [ ] Distance unit is valid: `m`, `km`, `mi`, `yd`, `nm`
- [ ] `within circle` has 3 args + optional unit: `(lat, lon, distance [, unit])`
- [ ] `within box` has 4 args: `(topLeftLat, topLeftLon, bottomRightLat, bottomRightLon)`
- [ ] `within polygon` has at least 3 vertex pairs

## Continuous Query Checks

- [ ] Query does not use GROUP BY, aggregation, or ORDER BY (not supported)
- [ ] Correct `Search` class import:
  - Embedded: `org.infinispan.query.Search`
  - Remote: `org.infinispan.client.hotrod.Search`
- [ ] Listener processes events quickly without blocking
- [ ] Projections are used when full entity values are not needed (reduces memory)

## Performance Checks

- [ ] Queries target indexed fields (non-indexed fields cause full scans)
- [ ] `commit-interval` is not set too low (default 1000ms is reasonable)
- [ ] `refresh-interval` is tuned for the workload (0 = fresh, > 0 = throughput)
- [ ] `queue-size` is large enough to avoid `CacheBackpressureFullException`
- [ ] Pagination is used for large result sets (`maxResults` + `startOffset`)
