## See Also

[Installation & Maintenance](Installation & Maintenance.md)

***

# Solr (v5.*)

* Solr is a standalone server for indexing and searching and is based on Apache Lucene library
* Solr is "serverlization" of Lucene with features beyond those of Lucene
* Solr supports geo-spatial search
* data import request handler, DataImportHandler, is a Solr contrib that provides a configuration driven way to import data directly from SQL databases and XMLs into Solr with both full and incremental imports
* Solr supports content extraction from and indexing of documents such as MS Word, PDF, images, audio, etc.; extraction is done via Solr Cell framework (a.k.a. ExtractionRequestHandler) and is based on Apache Tika toolkit; import of HTML documents is supported via Apache Nutch
* Solr can be extended with added features by means of plugins
* Solr can store its data according to a predefined scheme or its data can be schemaless
* the process of feeding Solr with data through its REST-like API, files, or the admin site is called *indexing*; inside Solr, indexing is analyzing/transforming source data into data structures that are optimized for searching
* the process of searching withing previously indexed data is called *querying*
* indexed data is also referred to as simply *index*
* the data of a Solr instance is organized into cores, which are logically separate indexes
* SolrCloud is coordinated by Zookeeper and is organized into clusters
* with reference to SolrCloud, a logically separate index is called *collection*; a collection is composed of one or more *shards*; a copy of a shard in SolrCloud is called *replica*
* the default port for Solr's REST-like API is 8983; a query can be made with e.g. `<host>:8983/solr/<corename>/select?q=<searchquery>&wt=json&indent=true`
* the admin site is available at `<host>:8983/solr`
* if installed with `install_solr_service.sh`, Solr binaries are located in `/opt/solr/bin` directory; the binaries are `solr` and `post`
* multiple files can be added to the index with `post -c <corename> *.ext`
* in schemaless mode, the internal schema is generated automatically and stored in `managed-schema`; types of fields are inferred from incoming data and all fields are multi-valued; as new data is added to the index, the internal schema is modified and `managed-schema` is updated; the default query field is `_text_` and all fields are automatically copied into it
* in schemaless mode, the internal schema is updated via HTTP requests, e.g. PUT JSON with a new field to `<host>:8983/solr/<corename>/fields/<newfieldname>`; no core reloading is required in this case

## Data Structure

* an index is composed of individual units of information called *documents*
* it is separate documents that are the primary content of query results
* a document is comprised by *fields* of possibly different types, such as text fields for full-text search, string fields, numeric fields, date/time fields, multi-valued fields, etc.
* a field in a document is how Solr interprets of a portion of the source data that was indexed in connection with this document; no exact field-to-field correspondence is required
* Solr can transform the source data during indexing with or without storing the source data for further retrieval
* if the index is not data-driven, then what types and what fields should be used by every field in the core's index is declared in the core's schema
* if the index is data-driven i.e. schemaless, fields and their types are deferred from incoming data at index-time
* even if the index is schema-based, field types can be hinted to Solr by means of their names and new fields can be indexed if the fields are *dynamic fields*
* every document in an index should normally contain a field that is unique throughout the index; such field is often a string field named `id`
* it's possible copy one field to another at index time via *copy field* mechanism; it's used either to index the same field differently, or to add multiple fields to the same field for easier/faster searching

## Data Flow

* source data is sent to Solr as a part of a request, specifying the RequestHandler (e.g. `/update`), query parameters, and the fields/values to be added to the index
* the request is handled by the RequestHandler according to its settings in `sorlconfig.xml`
* a source field's value goes through analysis/transformation according to how its type is defined (`<analyzer type="index">` subtag of `<fieldType>` tag in `schema.xml`)
* the outcome of field value analysis/transformation is a set of *tokens*, which is the information by which source data is actually indexed and searched
* when Sorl receives a search query, it is processed by another RequestHandler (e.g. `/select`)
* the values in the query's fields go through analysis/transformation according to how their types are defined (`<analyzer type="query">`)
* the search query is passed to a query parser (Lucene's standard query parser, DisMax, or eDisMax)
* the query is parsed into *terms*
* the index is searched based on the terms and the results are returned in a response

## Configuration

* Solr is configured on per-core basis
* a core's configuration (also known as *config set*) stored in `/var/solr/data/<corename>/conf`; the configuration of a SolrCloud collection is stored by Zookeeper
* the initial configuration for a core is typically copied from one of the "starter" configurations when a core is created, e.g. `sudo su - solr -c '/opt/solr/bin/solr create -c <corename> -d basic_configs'`
* the main configuration files are `schema.xml`, which describes the schema to be used by the core's index, and `sorlconfig.xml`, which specifies options for the core itself; for a data-driven index, `schema.xml` is not present

### `schema.xml`

* contains details about document fields and how to treat them when documents are added to or queried from the index
* declares field types (`<fieldType>` tag) based on Solr internal types, fields and their attributes (`<field>` tag), the field with unique values (`<uniqueKey>` tag), default search field (`<defaultSearchField>`, optional, can also be defined in `sorlconfig.xml` for the query request handler), the default operator for query parsing (e.g. `<solrQueryParser defaultOperator="AND" />`, optional), copy fields (optional)
* may additionally specify which fields are required and how to index and search each field
* some of the filed types predefined by examples are:

```python

boolean
int
float
long
double
date
binary
string
text_ws
text_general
text_en
text_en_splitting
text_en_splitting_tight
lowercase
point
location

```

* the Solr internal type behind the `text_...` types is `solr.TextField` for full-text search, which allows the specification of custom text analyzers/transformers specified as a tokenizer and a list of token filters; different analyzers/transformers/filters may be specified for index-time and query-time
* unlike with other textual types, values of `string` type are not analyzed/transformed and are indexed/stored verbatim
* new field types can be added for e.g. phonetic search
* fields are declared with `<fields>` and `<field>` tags, e.g. `<fields> <field name="somefield" type="string" indexed="true" stored="true" /> ...`
* some of the available field attributes are:

```python

name            # the field's name; required
type            # the field's locally defined type; required
indexed         # whether the field should be searchable or sortable
stored          # whether the field is retrievable
multiValued     # whether the field consists of separate values
required        # whether to throw an error if the field's value does not exist
default         # a value that should be used if no value is specified at index-time

# advanced:

docValues
omitNorms
termVectors
termPositions
termOffsets

```

* a summary of common use cases and the attributes the fields or field types should have to support the case are in https://cwiki.apache.org/confluence/display/solr/Field+Properties+by+Use+Case

### `sorlconfig.xml`

* primarily, configures schema mode (schemaless or not), request handlers with *search components*, and caching; others are index configuration, event listeners, request dispatcher, highlighting configuration, the admin site, etc.
* a request handler defines the logic executed for every request
* a core can have multiple request handlers defined with `<requestHandler>` tag, e.g. `<requestHandler name="/<handler>" class="solr.SearchHandler">`; additional handlers can be defined to customize search scope, sorting, etc.
* a request handler can specify default query parameters with `<lst name="defaults">` tag, e.g.:

```xml

...
<lst name="defaults">
    <str name="echoParams">explicit</str>
    <str name="wt">json</str>
    <str name="indent">true</str>
    ...
</lst>

```

* a request handler has a configurable list of search components which perform the actual searching; default components are `QueryComponent`, `FacetComponent`, `MoreLikeThisComponent`, `HighlightComponent`, `StatsComponent`, `DebugComponent`
* instead of redefining the list of default search components, custom components can be appended to the default ones as follows:

```xml

...
<arr name="last-components">
    <str>spellcheck</str>
</arr>

```

## Querying

* the default format for returned query results is XML; another format can be specified with `wt` parameter ("writer type"), e.g. `wt=json&indent=true`
* by default, query results are ordered by relevance
* the relevance of each result is in its `score` field, which is not included by default
* the default query parser is Lucene a.k.a. standard; the query parser that is most suitable for general-purpose search is however DisMax
* the default operator for query parsing is logical OR
* some of the query parameters, as in `<host>:8983/solr/<corename>/select?q=<searchquery>&<paramname>=<paramvalue>&...`, are:
    * **`q`**: query (a.k.a. **query event**), e.g. "apple", "title:apple"; required
    * **`fq`**: **filter query**; one or more filters to constrain search results, e.g. `count:1`, `count:[1 TO 5]`, `count:(1 OR 5)`; multiple filters are ANDed
    * **`sort`**: **sorts** results on a field in the ascending or descending order, e.g. `title asc`, `created_at desc`; cannot be used on multi-valued fields
    * **`start`** and **`rows`**: **pagination**; `start` is zero-based, e.g. `&start=0` in the query URL refers to the topmost search result
    * **`fl`**: **fields** to which search results are to be narrowed so that the rest of the fields are not included, e.g. `*,score`; default is `*` (all fields); can be used for optimization
    * **`df`**: **default field** within which the search is to be performed; only takes effect if the search field is not specified explicitly as in e.g. "title:apple"
    * **`wt`**: **writer type** to be used for search results i.e. search results format, e.g. `xml`, `python`
    * **`indent`**: whether to use **indentation** for search results, e.g. `true`
    * **`debugQuery`**: whether to append **debug** info to search results, e.g. `true`
    * **`defType`**: **query parser** to be used, e.g. `edismax`; this is usually set in the default parameters for a request handler's configuration
    * **`hl`**: whether to **highlight** found terms, e.g. `&hl=true&hl.fl=<fieldname>&hl.simple.pre=%3Cstrong%3E&hl.simple.post=%3C%2Fstrong%3E` in the query URL
    * **`facet`**: whether to perform the arrangement of search results into categories based on indexed terms and including numerical counts
* query terms can be mapped to equivalent terms in `synonyms.txt`
* common words, such as "a", "an", "and", or any other words can be automatically removed from every query at index-time and query-time if they are specified as *stop words* (located in `lang/stopwords_en.txt` and `stopwords.txt`)

### Query Syntax

```python

title:apple                 # only search in title
title:apple name:pear       # only search in two fields
title:"apple pear"          # "apple" OR "pear"
apple AND pear              # "apple" AND "pear"
(apple AND pear) OR orange  # compound AND with OR
apple -pear                 # "apple" without "pear"
-pear                       # all but "pear"
app*                        # any matches that start with "app"
app*ar                      # any matches that start with "app" and end with "ar"
[1 TO 10]                   # in the range from 1 to 10, inclusive
[5 TO *]                    # with an open-ended upper limit
(apple or pear)^10.0        # boosted by factor 10.0
"apple pear"~5              # "apple" and "pear" with proximity of 5 from one another

```
