# Chapter 7: Elasticsearch Basics

> **Estimated study time:** 2 days | **Priority:** 🟡 Medium

---

## Key Concepts

- Inverted index
- Index, document, field mappings
- Analyzers and tokenizers
- Query DSL (match, term, bool, range, nested)
- Aggregations
- Spring Data Elasticsearch
- When to use Elasticsearch vs a relational DB

---

## Questions & Answers

---

### Q1 — 🟢 What is an inverted index and why is it the foundation of Elasticsearch?

<details><summary>Click to reveal answer</summary>

An **inverted index** maps each unique term to the list of documents that contain it — the reverse of a forward index (document → words).

**Example:**

| Document | Content |
|----------|---------|
| doc1 | "Spring Boot performance" |
| doc2 | "Redis performance tips" |
| doc3 | "Spring Security guide" |

**Inverted index:**
```
"spring"      → [doc1, doc3]
"boot"        → [doc1]
"performance" → [doc1, doc2]
"redis"       → [doc2]
"security"    → [doc3]
```

**Query: "spring performance"**
1. Tokenize query → ["spring", "performance"]
2. Lookup: spring → [doc1, doc3], performance → [doc1, doc2]
3. Intersect (for AND) → [doc1]
4. Score by TF-IDF or BM25, rank, return

**Why it's fast:**
- Token lookup is O(1) (hash map or B-tree into the inverted index)
- No full-table scan needed
- Scales to millions of documents

**Contrast with SQL LIKE:** `WHERE content LIKE '%spring%'` requires a full sequential scan — O(n). The inverted index enables O(1) term lookup, then O(k) document retrieval where k = number of matching docs.

</details>

---

### Q2 — 🟢 What is the relationship between an Index, Document, and Field in Elasticsearch?

<details><summary>Click to reveal answer</summary>

Elasticsearch's hierarchy mirrors a database schema:

| Elasticsearch | Relational DB analogy |
|--------------|----------------------|
| **Cluster** | Database server |
| **Index** | Table |
| **Document** | Row |
| **Field** | Column |
| **Mapping** | Schema / DDL |

```json
// A document in the "products" index
{
  "_index": "products",
  "_id": "1",
  "_source": {
    "name": "Spring Boot in Action",
    "description": "A comprehensive guide to Spring Boot",
    "price": 39.99,
    "categories": ["books", "java", "spring"],
    "published_at": "2024-01-15",
    "author": {
      "name": "Craig Walls",
      "country": "US"
    }
  }
}
```

**Create an index with explicit mapping:**
```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "product_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }  // for exact match and sorting
        }
      },
      "price": { "type": "double" },
      "categories": { "type": "keyword" },
      "published_at": { "type": "date" },
      "author": {
        "type": "object",
        "properties": {
          "name": { "type": "text" },
          "country": { "type": "keyword" }
        }
      }
    }
  }
}
```

**Field types:**
| Type | Use for |
|------|---------|
| `text` | Full-text search (analyzed, tokenized) |
| `keyword` | Exact match, aggregations, sorting |
| `integer`, `long`, `double` | Numeric fields |
| `date` | Date/time (ISO 8601 by default) |
| `boolean` | True/false |
| `nested` | Array of objects preserving relationship |
| `geo_point` | Latitude/longitude |

</details>

---

### Q3 — 🟡 What are analyzers in Elasticsearch and how do they work?

<details><summary>Click to reveal answer</summary>

An **analyzer** processes text during indexing AND at query time (for `text` fields). It consists of three components:

1. **Character filter** — pre-processing (strip HTML, replace characters)
2. **Tokenizer** — splits text into tokens (words)
3. **Token filter** — transforms tokens (lowercase, remove stop words, stem)

```
Input: "Spring Boot is AMAZING!"
→ Character filter: (none in standard)
→ Tokenizer (standard): ["Spring", "Boot", "is", "AMAZING"]
→ Token filter (lowercase): ["spring", "boot", "is", "amazing"]
→ Token filter (stop): ["spring", "boot", "amazing"]  (removes "is")
→ Token filter (snowball): ["spring", "boot", "amaz"]  (stemming)
```

**Built-in analyzers:**

| Analyzer | Behavior |
|----------|---------|
| `standard` | Lowercase + standard tokenizer (Unicode) |
| `english` | Standard + English stop words + stemming |
| `whitespace` | Splits on whitespace only, preserves case |
| `keyword` | No tokenization (whole string as one token) |
| `simple` | Lowercase, splits on non-letter characters |

**Custom analyzer example:**
```json
PUT /products
{
  "settings": {
    "analysis": {
      "filter": {
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        }
      },
      "analyzer": {
        "product_search_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stop", "english_stemmer"]
        }
      }
    }
  }
}
```

**Test your analyzer:**
```json
POST /_analyze
{
  "analyzer": "english",
  "text": "Spring Boot applications are running"
}
// Returns: ["spring", "boot", "applic", "run"]
```

</details>

---

### Q4 — 🟡 Explain the key query types in the Elasticsearch Query DSL.

<details><summary>Click to reveal answer</summary>

**Full-text queries (analyzed):**
```json
// match: standard full-text search
GET /products/_search
{
  "query": {
    "match": {
      "description": "spring boot performance"
    }
  }
}

// match_phrase: words must appear in order and adjacent
{
  "query": {
    "match_phrase": {
      "description": "spring boot"
    }
  }
}

// multi_match: search across multiple fields
{
  "query": {
    "multi_match": {
      "query": "spring boot",
      "fields": ["name^3", "description"],   // name boosted 3x
      "type": "best_fields"
    }
  }
}
```

**Term-level queries (not analyzed — exact match):**
```json
// term: exact value (use on keyword fields)
{
  "query": {
    "term": { "categories": "java" }
  }
}

// terms: match any of the values (SQL IN)
{
  "query": {
    "terms": { "categories": ["java", "spring"] }
  }
}

// range: numeric or date ranges
{
  "query": {
    "range": {
      "price": { "gte": 10, "lte": 50 }
    }
  }
}
```

**Compound queries:**
```json
// bool: combine multiple queries with AND (must), OR (should), NOT (must_not)
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "spring" } }
      ],
      "filter": [
        { "term": { "categories": "java" } },
        { "range": { "price": { "lte": 50 } } }
      ],
      "should": [
        { "match": { "name": "boot" } }
      ],
      "must_not": [
        { "term": { "categories": "deprecated" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

**Difference between `must` and `filter`:**
- `must`: contributes to relevance score (_score)
- `filter`: binary yes/no, no scoring, results are cached → faster for numeric/date filters

</details>

---

### Q5 — 🟡 What are aggregations in Elasticsearch?

<details><summary>Click to reveal answer</summary>

Aggregations compute summaries over search results (like SQL `GROUP BY` + aggregate functions).

**Types:**

| Type | SQL equivalent | Example |
|------|---------------|---------|
| `terms` | GROUP BY | Group products by category |
| `sum`, `avg`, `min`, `max` | SUM/AVG/MIN/MAX | Average price |
| `date_histogram` | GROUP BY date_trunc | Orders per month |
| `range` | CASE WHEN | Price buckets |
| `nested` | JOIN | Aggregations on nested objects |

```json
GET /products/_search
{
  "size": 0,   // don't return documents, only aggregations
  "query": {
    "term": { "in_stock": true }
  },
  "aggs": {
    "by_category": {
      "terms": {
        "field": "categories",
        "size": 10
      },
      "aggs": {
        "avg_price": {
          "avg": { "field": "price" }
        },
        "price_ranges": {
          "range": {
            "field": "price",
            "ranges": [
              { "to": 20 },
              { "from": 20, "to": 50 },
              { "from": 50 }
            ]
          }
        }
      }
    },
    "orders_over_time": {
      "date_histogram": {
        "field": "published_at",
        "calendar_interval": "month"
      }
    }
  }
}
```

**Response:**
```json
{
  "aggregations": {
    "by_category": {
      "buckets": [
        {
          "key": "java",
          "doc_count": 150,
          "avg_price": { "value": 34.5 }
        },
        {
          "key": "spring",
          "doc_count": 80,
          "avg_price": { "value": 42.0 }
        }
      ]
    }
  }
}
```

</details>

---

### Q6 — 🟡 How do you integrate Elasticsearch with a Spring Boot application?

<details><summary>Click to reveal answer</summary>

**Dependencies:**
```xml
<dependency>
  <groupId>org.springframework.data</groupId>
  <artifactId>spring-data-elasticsearch</artifactId>
</dependency>
```

**Configuration:**
```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: ${ELASTIC_PASSWORD}
    connection-timeout: 5s
    socket-timeout: 10s
```

**Entity mapping:**
```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/product-settings.json")
public class ProductDocument {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "english")
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Keyword)
    private List<String> categories;

    @Field(type = FieldType.Date, format = DateFormat.iso8601)
    private LocalDateTime publishedAt;
}
```

**Repository:**
```java
public interface ProductSearchRepository
    extends ElasticsearchRepository<ProductDocument, String> {

    // Derived query methods
    List<ProductDocument> findByCategories(String category);

    // Custom query
    @Query("{\"bool\": {\"must\": [{\"match\": {\"description\": \"?0\"}}], " +
           "\"filter\": [{\"range\": {\"price\": {\"lte\": ?1}}}]}}")
    Page<ProductDocument> searchByDescriptionWithMaxPrice(
        String description, double maxPrice, Pageable pageable);
}
```

**ElasticsearchOperations for complex queries:**
```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations operations;

    public SearchHits<ProductDocument> search(ProductSearchRequest req) {
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        if (StringUtils.hasText(req.getText())) {
            boolQuery.must(MultiMatchQuery.of(m -> m
                .fields("name^3", "description")
                .query(req.getText())
                .type(TextQueryType.BestFields)
            )._toQuery());
        }

        if (req.getCategory() != null) {
            boolQuery.filter(TermQuery.of(t -> t
                .field("categories")
                .value(req.getCategory())
            )._toQuery());
        }

        if (req.getMaxPrice() != null) {
            boolQuery.filter(RangeQuery.of(r -> r
                .field("price")
                .lte(JsonData.of(req.getMaxPrice()))
            )._toQuery());
        }

        NativeQuery query = NativeQuery.builder()
            .withQuery(boolQuery.build()._toQuery())
            .withPageable(PageRequest.of(req.getPage(), req.getSize()))
            .withSort(Sort.by(req.getSortField()).descending())
            .withAggregation("by_category",
                Aggregation.of(a -> a.terms(t -> t.field("categories").size(10))))
            .build();

        return operations.search(query, ProductDocument.class);
    }
}
```

**Sync DB → Elasticsearch:**
```java
@Component
@RequiredArgsConstructor
public class ProductIndexingService {

    private final ProductRepository dbRepo;
    private final ProductSearchRepository searchRepo;
    private final ProductMapper mapper;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductSaved(ProductSavedEvent event) {
        Product product = dbRepo.findById(event.productId()).orElseThrow();
        searchRepo.save(mapper.toDocument(product));
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductDeleted(ProductDeletedEvent event) {
        searchRepo.deleteById(String.valueOf(event.productId()));
    }
}
```

</details>

---

### Q7 — 🔴 When should you use Elasticsearch instead of a relational database for search?

<details><summary>Click to reveal answer</summary>

**Use Elasticsearch when:**

| Requirement | Elasticsearch advantage |
|-------------|------------------------|
| Full-text search with relevance ranking | BM25 scoring built in; `LIKE` in SQL is linear scan |
| Fuzzy search / typo tolerance | `fuzziness` parameter handles edit distance |
| Faceted search (category counts + filtering) | Aggregations with `post_filter` |
| Autocomplete / suggestions | `completion` field type, prefix queries |
| Log and event analytics | Excellent for time-series, date histograms |
| Large text corpora (millions of documents) | Distributed by design; horizontal scaling |
| Geospatial search | `geo_distance`, `geo_bounding_box` queries |

**Use PostgreSQL (full-text) when:**

| Situation | Why SQL is enough |
|-----------|------------------|
| Small dataset (< 1M rows) | `tsvector` + `GIN` index is very fast |
| Already on PostgreSQL | No additional infrastructure |
| ACID guarantees required | SQL FTS search and data are always in sync |
| Simple keyword search only | No need for BM25 scoring, aggregations |

**Architecture pattern — Dual writes:**
```
Write path: App → PostgreSQL (source of truth)
                 → Elasticsearch (via event or CDC)

Read path: Full-text search → Elasticsearch
           Transactional read → PostgreSQL
```

**Challenges with Elasticsearch:**
1. **Eventual consistency** — ES is not the source of truth; sync lag exists
2. **No transactions** — operations are not ACID
3. **Operational complexity** — cluster management, index lifecycle management
4. **Near-real-time** — indexed documents appear after a refresh interval (default 1s)
5. **Schema changes** — reindex required to change field types (expensive for large indexes)

**PostgreSQL full-text search setup** (for when ES is overkill):
```sql
-- Add tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Populate
UPDATE products
SET search_vector = to_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,''));

-- GIN index for fast full-text search
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- Trigger to keep it updated
CREATE TRIGGER products_search_update
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(search_vector, 'pg_catalog.english', name, description);

-- Query
SELECT *, ts_rank(search_vector, query) AS rank
FROM products, to_tsquery('english', 'spring & boot') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

</details>
