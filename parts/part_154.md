# Part 154: Crystal กับ Elasticsearch

## บทนำ

Elasticsearch เป็น search engine ที่มีประสิทธิภาพสูง เหมาะสำหรับ Full-Text Search, Log Analytics และ Real-time Data Analysis Crystal ไม่มี official shard สำหรับ Elasticsearch แต่เราสามารถใช้ HTTP client ของ Crystal เชื่อมต่อได้โดยตรง ซึ่งทำให้มีความยืดหยุ่นสูง

## การตั้งค่า HTTP Client

### shard.yml

```yaml
# shard.yml
name: elasticsearch_app
version: 1.0.0

dependencies:
  http:
    github: crystal-lang/crystal
    version: "*"
```

### Elasticsearch Client

```crystal
# src/elasticsearch/client.cr
require "http/client"
require "json"
require "uri"

class ElasticsearchClient
  class Error < Exception; end
  class NotFoundError < Error; end
  class IndexError < Error; end
  
  property host : String
  property port : Int32
  property scheme : String
  
  def initialize(
    @host : String = "localhost",
    @port : Int32 = 9200,
    @scheme : String = "http",
    @username : String? = nil,
    @password : String? = nil
  )
  end
  
  # GET request
  def get(path : String, params : Hash(String, String) = {} of String => String) : JSON::Any
    response = http_client.get(build_url(path, params), headers: default_headers)
    handle_response(response)
  end
  
  # POST request
  def post(path : String, body : JSON::Any | Hash | NamedTuple | Nil = nil) : JSON::Any
    response = http_client.post(
      build_url(path),
      headers: default_headers,
      body: body.try(&.to_json)
    )
    handle_response(response)
  end
  
  # PUT request
  def put(path : String, body : JSON::Any | Hash | NamedTuple | Nil = nil) : JSON::Any
    response = http_client.put(
      build_url(path),
      headers: default_headers,
      body: body.try(&.to_json)
    )
    handle_response(response)
  end
  
  # DELETE request
  def delete(path : String) : JSON::Any
    response = http_client.delete(build_url(path), headers: default_headers)
    handle_response(response)
  end
  
  # ตรวจสอบ health
  def health : NamedTuple(status: String, cluster_name: String, nodes: Int32)
    result = get("/_cluster/health")
    {
      status: result["status"].as_s,
      cluster_name: result["cluster_name"].as_s,
      nodes: result["number_of_nodes"].as_i
    }
  end
  
  # ตรวจสอบ connection
  def ping : Bool
    response = http_client.head(build_url("/"))
    response.success?
  rescue
    false
  end
  
  private def http_client : HTTP::Client
    client = HTTP::Client.new(@host, @port, @scheme == "https")
    
    if username = @username
      password = @password || ""
      client.basic_auth(username, password)
    end
    
    client.connect_timeout = 5.seconds
    client.read_timeout = 30.seconds
    client
  end
  
  private def default_headers : HTTP::Headers
    HTTP::Headers{
      "Content-Type" => "application/json",
      "Accept" => "application/json"
    }
  end
  
  private def build_url(path : String, params : Hash(String, String) = {} of String => String) : String
    url = "#{@scheme}://#{@host}:#{@port}#{path}"
    unless params.empty?
      query = URI::Params.build { |p| params.each { |k, v| p.add(k, v) } }
      url += "?#{query}"
    end
    url
  end
  
  private def handle_response(response : HTTP::Client::Response) : JSON::Any
    body = response.body.presence
    result = body ? JSON.parse(body) : JSON::Any.new(nil)
    
    case response.status_code
    when 200, 201
      result
    when 404
      raise NotFoundError.new("ไม่พบ resource: #{response.body}")
    when 400..499
      error = result["error"]?.try(&.["reason"]?.try(&.as_s)) || response.body
      raise Error.new("Client Error #{response.status_code}: #{error}")
    when 500..599
      raise Error.new("Server Error #{response.status_code}: #{response.body}")
    else
      result
    end
  end
end
```

## Index Management

### การสร้างและจัดการ Index

```crystal
# src/elasticsearch/index_manager.cr
class IndexManager
  def initialize(@client : ElasticsearchClient)
  end
  
  # ตรวจสอบว่า index มีอยู่หรือไม่
  def index_exists?(index_name : String) : Bool
    @client.get("/#{index_name}")
    true
  rescue ElasticsearchClient::NotFoundError
    false
  end
  
  # สร้าง index สำหรับบทความ
  def create_articles_index
    mapping = {
      "settings" => {
        "number_of_shards" => 1,
        "number_of_replicas" => 0,
        "analysis" => {
          "analyzer" => {
            "thai_analyzer" => {
              "type" => "custom",
              "tokenizer" => "standard",
              "filter" => ["lowercase", "stop"]
            },
            "autocomplete_analyzer" => {
              "type" => "custom",
              "tokenizer" => "standard",
              "filter" => ["lowercase", "edge_ngram_filter"]
            },
            "autocomplete_search_analyzer" => {
              "type" => "custom",
              "tokenizer" => "standard",
              "filter" => ["lowercase"]
            }
          },
          "filter" => {
            "edge_ngram_filter" => {
              "type" => "edge_ngram",
              "min_gram" => 1,
              "max_gram" => 20
            }
          }
        }
      },
      "mappings" => {
        "properties" => {
          "id" => {"type" => "keyword"},
          "title" => {
            "type" => "text",
            "analyzer" => "standard",
            "fields" => {
              "keyword" => {"type" => "keyword"},
              "autocomplete" => {
                "type" => "text",
                "analyzer" => "autocomplete_analyzer",
                "search_analyzer" => "autocomplete_search_analyzer"
              }
            }
          },
          "content" => {
            "type" => "text",
            "analyzer" => "standard"
          },
          "author" => {
            "type" => "text",
            "fields" => {
              "keyword" => {"type" => "keyword"}
            }
          },
          "tags" => {"type" => "keyword"},
          "status" => {"type" => "keyword"},
          "category" => {"type" => "keyword"},
          "views" => {"type" => "integer"},
          "rating" => {"type" => "float"},
          "published_at" => {"type" => "date"},
          "created_at" => {"type" => "date"}
        }
      }
    }
    
    if index_exists?("articles")
      puts "Index 'articles' มีอยู่แล้ว"
      return
    end
    
    result = @client.put("/articles", mapping)
    puts "สร้าง index 'articles' สำเร็จ: #{result["acknowledged"]}"
  end
  
  # ลบ index
  def delete_index(index_name : String) : Bool
    @client.delete("/#{index_name}")
    puts "ลบ index '#{index_name}' สำเร็จ"
    true
  rescue ElasticsearchClient::NotFoundError
    puts "ไม่พบ index '#{index_name}'"
    false
  end
  
  # Reindex - สร้าง index ใหม่จาก index เดิม
  def reindex(source : String, destination : String) : Int64
    result = @client.post("/_reindex", {
      "source" => {"index" => source},
      "dest"   => {"index" => destination}
    })
    result["total"].as_i64
  end
  
  # ดู mapping ของ index
  def get_mapping(index_name : String) : JSON::Any
    @client.get("/#{index_name}/_mapping")
  end
  
  # อัปเดต settings ของ index
  def update_settings(index_name : String, settings : Hash)
    @client.put("/#{index_name}/_settings", settings)
  end
end
```

## Indexing Documents

### การ Index เอกสาร

```crystal
# src/elasticsearch/document_indexer.cr
class DocumentIndexer
  def initialize(@client : ElasticsearchClient)
  end
  
  # Index เอกสารเดียว
  def index_article(article : Article) : Bool
    doc = {
      "id" => article.id,
      "title" => article.title,
      "content" => article.content,
      "author" => article.author,
      "tags" => article.tags,
      "status" => article.status,
      "category" => article.category,
      "views" => article.views,
      "rating" => article.rating,
      "published_at" => article.published_at.try(&.to_rfc3339),
      "created_at" => article.created_at.to_rfc3339
    }
    
    result = @client.put("/articles/_doc/#{article.id}", doc)
    result["result"].as_s.in?("created", "updated")
  end
  
  # Bulk Index - เพิ่มหลายเอกสารพร้อมกัน
  def bulk_index(articles : Array(Article)) : NamedTuple(indexed: Int32, errors: Int32)
    return {indexed: 0, errors: 0} if articles.empty?
    
    # สร้าง bulk request body
    bulk_body = String.build do |sb|
      articles.each do |article|
        # Action line
        sb << {"index" => {"_index" => "articles", "_id" => article.id}}.to_json
        sb << "\n"
        
        # Document line
        sb << {
          "id" => article.id,
          "title" => article.title,
          "content" => article.content,
          "author" => article.author,
          "tags" => article.tags,
          "status" => article.status,
          "views" => article.views,
          "created_at" => article.created_at.to_rfc3339
        }.to_json
        sb << "\n"
      end
    end
    
    # ส่ง bulk request
    response = HTTP::Client.new("localhost", 9200).post(
      "/_bulk",
      headers: HTTP::Headers{"Content-Type" => "application/x-ndjson"},
      body: bulk_body
    )
    
    result = JSON.parse(response.body)
    
    indexed = 0
    errors = 0
    
    result["items"].as_a.each do |item|
      action = item["index"]
      if action["error"]?
        errors += 1
        puts "Error indexing #{action["_id"]}: #{action["error"]["reason"]}"
      else
        indexed += 1
      end
    end
    
    {indexed: indexed, errors: errors}
  end
  
  # ลบเอกสาร
  def delete_document(index : String, id : String) : Bool
    @client.delete("/#{index}/_doc/#{id}")
    true
  rescue ElasticsearchClient::NotFoundError
    false
  end
  
  # อัปเดตเอกสาร (Partial Update)
  def update_document(index : String, id : String, fields : Hash) : Bool
    result = @client.post("/#{index}/_update/#{id}", {
      "doc" => fields,
      "doc_as_upsert" => true
    })
    result["result"].as_s.in?("updated", "created", "noop")
  end
end
```

## Full-Text Search

### Search Engine สำหรับบทความ

```crystal
# src/elasticsearch/searcher.cr
class ArticleSearcher
  record SearchResult,
    id : String,
    title : String,
    author : String,
    tags : Array(String),
    score : Float64,
    highlight : Hash(String, Array(String))
  
  record SearchResponse,
    hits : Array(SearchResult),
    total : Int64,
    took_ms : Int32,
    max_score : Float64?
  
  def initialize(@client : ElasticsearchClient)
  end
  
  # ค้นหาพื้นฐาน
  def search(
    query : String,
    page : Int32 = 1,
    per_page : Int32 = 10,
    filters : Hash(String, String) = {} of String => String
  ) : SearchResponse
    from = (page - 1) * per_page
    
    # สร้าง query
    must_clauses = [] of Hash(String, JSON::Any)
    
    # Multi-match query
    must_clauses << {
      "multi_match" => JSON.parse({
        "query" => query,
        "fields" => ["title^3", "content^1", "author^2", "tags^2"],
        "type" => "best_fields",
        "fuzziness" => "AUTO",
        "minimum_should_match" => "75%"
      }.to_json)
    }
    
    # Filter clauses
    filter_clauses = [] of Hash(String, JSON::Any)
    filters.each do |field, value|
      filter_clauses << {"term" => JSON.parse({field => value}.to_json)}
    end
    
    # สร้าง request body
    request_body = {
      "from" => from,
      "size" => per_page,
      "query" => {
        "bool" => {
          "must" => must_clauses,
          "filter" => filter_clauses
        }
      },
      "highlight" => {
        "pre_tags" => ["<mark>"],
        "post_tags" => ["</mark>"],
        "fields" => {
          "title" => {"number_of_fragments" => 1},
          "content" => {"number_of_fragments" => 3, "fragment_size" => 150}
        }
      },
      "sort" => [
        {"_score" => "desc"},
        {"created_at" => "desc"}
      ]
    }
    
    result = @client.post("/articles/_search", request_body)
    parse_search_response(result)
  end
  
  # Advanced Search ด้วย Filter ซับซ้อน
  def advanced_search(
    query : String? = nil,
    tags : Array(String) = [] of String,
    author : String? = nil,
    date_from : Time? = nil,
    date_to : Time? = nil,
    min_rating : Float64? = nil,
    sort : String = "relevance",
    page : Int32 = 1,
    per_page : Int32 = 10
  ) : SearchResponse
    from = (page - 1) * per_page
    
    bool_query = {} of String => Array(Hash)
    
    # Must clauses
    must = [] of Hash(String, JSON::Any)
    if query && !query.empty?
      must << {
        "multi_match" => JSON.parse({
          "query" => query,
          "fields" => ["title^3", "content", "author^2"],
          "type" => "most_fields"
        }.to_json)
      }
    else
      must << {"match_all" => JSON.parse("{}".to_json)}
    end
    
    # Filter clauses
    filter = [] of Hash(String, JSON::Any)
    
    # Filter by tags
    unless tags.empty?
      filter << {
        "terms" => JSON.parse({"tags" => tags}.to_json)
      }
    end
    
    # Filter by author
    if author
      filter << {
        "term" => JSON.parse({"author.keyword" => author}.to_json)
      }
    end
    
    # Filter by date range
    if date_from || date_to
      range_filter = {} of String => String
      range_filter["gte"] = date_from.try(&.to_rfc3339) || ""
      range_filter["lte"] = date_to.try(&.to_rfc3339) || ""
      range_filter.reject! { |_, v| v.empty? }
      
      filter << {
        "range" => JSON.parse({"published_at" => range_filter}.to_json)
      }
    end
    
    # Filter by rating
    if min_rating
      filter << {
        "range" => JSON.parse({"rating" => {"gte" => min_rating}}.to_json)
      }
    end
    
    # Sort
    sort_clause = case sort
                  when "relevance" then [{"_score" => "desc"}]
                  when "newest"    then [{"created_at" => "desc"}]
                  when "oldest"    then [{"created_at" => "asc"}]
                  when "popular"   then [{"views" => "desc"}]
                  when "rating"    then [{"rating" => "desc"}]
                  else              [{"_score" => "desc"}]
                  end
    
    request_body = {
      "from" => from,
      "size" => per_page,
      "query" => {
        "bool" => {
          "must"   => must,
          "filter" => filter
        }
      },
      "highlight" => {
        "fields" => {
          "title"   => {"number_of_fragments" => 0},
          "content" => {"fragment_size" => 150, "number_of_fragments" => 3}
        }
      },
      "sort" => sort_clause,
      "aggs" => {
        "tags" => {
          "terms" => {"field" => "tags", "size" => 20}
        },
        "authors" => {
          "terms" => {"field" => "author.keyword", "size" => 10}
        }
      }
    }
    
    result = @client.post("/articles/_search", request_body)
    parse_search_response(result)
  end
  
  private def parse_search_response(result : JSON::Any) : SearchResponse
    hits_data = result["hits"]
    total = hits_data["total"]["value"].as_i64
    took_ms = result["took"].as_i
    max_score = hits_data["max_score"].as_f? 
    
    hits = hits_data["hits"].as_a.map do |hit|
      source = hit["_source"]
      
      highlights = {} of String => Array(String)
      if highlight = hit["highlight"]?
        highlight.as_h.each do |field, fragments|
          highlights[field] = fragments.as_a.map(&.as_s)
        end
      end
      
      SearchResult.new(
        id: source["id"].as_s,
        title: source["title"].as_s,
        author: source["author"].as_s,
        tags: source["tags"].as_a.map(&.as_s),
        score: hit["_score"].as_f,
        highlight: highlights
      )
    end
    
    SearchResponse.new(
      hits: hits,
      total: total,
      took_ms: took_ms,
      max_score: max_score
    )
  end
end
```

## Aggregations

### การวิเคราะห์ข้อมูลด้วย Aggregations

```crystal
# src/elasticsearch/aggregation_analyzer.cr
class AggregationAnalyzer
  def initialize(@client : ElasticsearchClient)
  end
  
  # สถิติตาม category
  def category_stats : Array(NamedTuple(category: String, count: Int64, avg_views: Float64))
    request = {
      "size" => 0,
      "aggs" => {
        "by_category" => {
          "terms" => {
            "field" => "category",
            "size" => 50
          },
          "aggs" => {
            "avg_views" => {
              "avg" => {"field" => "views"}
            }
          }
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    buckets = result["aggregations"]["by_category"]["buckets"].as_a
    
    buckets.map do |bucket|
      {
        category: bucket["key"].as_s,
        count: bucket["doc_count"].as_i64,
        avg_views: bucket["avg_views"]["value"].as_f
      }
    end
  end
  
  # Timeline สำหรับ date histogram
  def publication_timeline(interval : String = "month") : Array(NamedTuple(date: String, count: Int64))
    request = {
      "size" => 0,
      "aggs" => {
        "publications_over_time" => {
          "date_histogram" => {
            "field" => "published_at",
            "calendar_interval" => interval,
            "format" => "yyyy-MM"
          }
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    buckets = result["aggregations"]["publications_over_time"]["buckets"].as_a
    
    buckets.map do |bucket|
      {
        date: bucket["key_as_string"].as_s,
        count: bucket["doc_count"].as_i64
      }
    end
  end
  
  # Percentile ของ views
  def views_percentiles : Hash(String, Float64)
    request = {
      "size" => 0,
      "aggs" => {
        "views_percentiles" => {
          "percentiles" => {
            "field" => "views",
            "percents" => [25, 50, 75, 90, 95, 99]
          }
        },
        "views_stats" => {
          "stats" => {"field" => "views"}
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    percentiles = result["aggregations"]["views_percentiles"]["values"].as_h
    
    percentiles.transform_values(&.as_f)
  end
  
  # Geospatial aggregation (ถ้ามี geo field)
  def geographic_distribution : Array(NamedTuple(country: String, count: Int64))
    request = {
      "size" => 0,
      "aggs" => {
        "by_country" => {
          "terms" => {
            "field" => "country",
            "size" => 100,
            "order" => {"_count" => "desc"}
          }
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    buckets = result["aggregations"]["by_country"]["buckets"].as_a
    
    buckets.map do |bucket|
      {
        country: bucket["key"].as_s,
        count: bucket["doc_count"].as_i64
      }
    end
  end
end
```

## Autocomplete Implementation

### Autocomplete Service

```crystal
# src/elasticsearch/autocomplete.cr
class AutocompleteService
  def initialize(@client : ElasticsearchClient)
  end
  
  # Autocomplete ด้วย Edge NGram
  def suggest(prefix : String, limit : Int32 = 10) : Array(String)
    request = {
      "size" => limit,
      "_source" => ["title"],
      "query" => {
        "multi_match" => {
          "query" => prefix,
          "type" => "bool_prefix",
          "fields" => ["title", "title._2gram", "title._3gram"]
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    result["hits"]["hits"].as_a.map { |hit| hit["_source"]["title"].as_s }
  end
  
  # Suggest ด้วย Completion Suggester
  def complete(prefix : String, size : Int32 = 5) : Array(NamedTuple(text: String, score: Float64))
    request = {
      "suggest" => {
        "title_suggest" => {
          "prefix" => prefix,
          "completion" => {
            "field" => "title.suggest",
            "size" => size,
            "skip_duplicates" => true,
            "fuzzy" => {
              "fuzziness" => 1
            }
          }
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    
    options = result["suggest"]["title_suggest"][0]["options"].as_a
    options.map do |opt|
      {
        text: opt["text"].as_s,
        score: opt["_score"].as_f
      }
    end
  end
  
  # Search As You Type
  def search_as_you_type(text : String, limit : Int32 = 8) : Array(String)
    request = {
      "size" => limit,
      "_source" => ["title"],
      "query" => {
        "multi_match" => {
          "query" => text,
          "type" => "bool_prefix",
          "fields" => [
            "title",
            "title._2gram",
            "title._3gram",
            "title._index_prefix"
          ]
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    result["hits"]["hits"].as_a.map { |hit| hit["_source"]["title"].as_s }
  end
  
  # Did You Mean - การแนะนำคำที่ใกล้เคียง
  def did_you_mean(query : String) : Array(String)
    request = {
      "suggest" => {
        "text" => query,
        "did_you_mean" => {
          "phrase" => {
            "field" => "content",
            "size" => 3,
            "gram_size" => 3,
            "direct_generator" => [{
              "field" => "content",
              "suggest_mode" => "missing"
            }],
            "highlight" => {
              "pre_tag" => "<em>",
              "post_tag" => "</em>"
            }
          }
        }
      }
    }
    
    result = @client.post("/articles/_search", request)
    
    options = result["suggest"]["did_you_mean"][0]["options"].as_a
    options.map { |opt| opt["text"].as_s }
  end
end
```

## Real-World Search Feature

### Search API สำหรับ Web Application

```crystal
# src/api/search_controller.cr
require "http/server"
require "json"
require "../elasticsearch/client"
require "../elasticsearch/searcher"
require "../elasticsearch/autocomplete"

class SearchController
  def initialize
    @es_client = ElasticsearchClient.new(
      host: ENV.fetch("ES_HOST", "localhost"),
      port: ENV.fetch("ES_PORT", "9200").to_i
    )
    @searcher = ArticleSearcher.new(@es_client)
    @autocomplete = AutocompleteService.new(@es_client)
    @indexer = DocumentIndexer.new(@es_client)
    @analytics = AggregationAnalyzer.new(@es_client)
  end
  
  # GET /api/search?q=query&page=1&per_page=10
  def search(context : HTTP::Server::Context)
    params = context.request.query_params
    
    query    = params["q"]? || ""
    page     = params["page"]?.try(&.to_i) || 1
    per_page = params["per_page"]?.try(&.to_i) || 10
    status   = params["status"]?
    author   = params["author"]?
    
    # Validate parameters
    page = [page, 1].max
    per_page = per_page.clamp(1, 100)
    
    filters = {} of String => String
    filters["status"] = status if status
    filters["author.keyword"] = author if author
    
    if query.empty?
      response = {
        "hits" => [] of Nil,
        "total" => 0,
        "page" => page,
        "per_page" => per_page,
        "message" => "กรุณาระบุคำค้นหา"
      }
    else
      result = @searcher.search(query, page, per_page, filters)
      
      response = {
        "hits" => result.hits.map { |hit|
          {
            "id" => hit.id,
            "title" => hit.title,
            "author" => hit.author,
            "tags" => hit.tags,
            "score" => hit.score,
            "highlight" => hit.highlight
          }
        },
        "total" => result.total,
        "page" => page,
        "per_page" => per_page,
        "took_ms" => result.took_ms
      }
    end
    
    context.response.content_type = "application/json"
    context.response.print(response.to_json)
  end
  
  # GET /api/autocomplete?q=prefix
  def autocomplete(context : HTTP::Server::Context)
    prefix = context.request.query_params["q"]? || ""
    limit  = context.request.query_params["limit"]?.try(&.to_i) || 8
    
    suggestions = if prefix.size >= 2
                    @autocomplete.suggest(prefix, limit)
                  else
                    [] of String
                  end
    
    context.response.content_type = "application/json"
    context.response.print({"suggestions" => suggestions}.to_json)
  end
  
  # GET /api/analytics/stats
  def analytics_stats(context : HTTP::Server::Context)
    stats = {
      "category_stats" => @analytics.category_stats,
      "views_percentiles" => @analytics.views_percentiles,
      "timeline" => @analytics.publication_timeline("month")
    }
    
    context.response.content_type = "application/json"
    context.response.print(stats.to_json)
  end
end

# เริ่ม HTTP Server
controller = SearchController.new

server = HTTP::Server.new do |context|
  path = context.request.path
  method = context.request.method
  
  case {method, path}
  when {"GET", /^\/api\/search/}
    controller.search(context)
  when {"GET", /^\/api\/autocomplete/}
    controller.autocomplete(context)
  when {"GET", /^\/api\/analytics\/stats/}
    controller.analytics_stats(context)
  else
    context.response.status = HTTP::Status::NOT_FOUND
    context.response.print("Not Found")
  end
end

puts "Search API กำลังทำงานที่ http://localhost:8080"
server.listen("0.0.0.0", 8080)
```

### ตัวอย่างการใช้งาน

```crystal
# src/examples/search_demo.cr
require "../elasticsearch/client"
require "../elasticsearch/index_manager"
require "../elasticsearch/document_indexer"
require "../elasticsearch/searcher"
require "../elasticsearch/autocomplete"

# เชื่อมต่อ
es = ElasticsearchClient.new
puts "Elasticsearch health: #{es.health}"

# ตั้งค่า index
manager = IndexManager.new(es)
manager.create_articles_index

# Index ตัวอย่างข้อมูล
indexer = DocumentIndexer.new(es)

sample_articles = [
  {id: "1", title: "Crystal Programming Language", content: "Crystal is a fast, compiled language...", author: "Admin", tags: ["crystal", "programming"], status: "published", views: 150},
  {id: "2", title: "การเขียน Crystal เบื้องต้น", content: "Crystal เป็นภาษาที่มีไวยากรณ์คล้าย Ruby...", author: "สมชาย", tags: ["crystal", "tutorial", "thai"], status: "published", views: 320},
  {id: "3", title: "Crystal กับ PostgreSQL", content: "การใช้งาน Crystal ร่วมกับฐานข้อมูล...", author: "Admin", tags: ["crystal", "database", "postgresql"], status: "published", views: 89}
]

puts "\n=== Indexing Articles ==="
sample_articles.each do |article|
  # ต้องสร้าง Article struct จริงๆ ในโปรเจกต์จริง
  puts "Indexed: #{article[:title]}"
end

# ค้นหา
searcher = ArticleSearcher.new(es)
puts "\n=== Full-Text Search ==="
result = searcher.search("Crystal programming")
puts "พบ #{result.total} บทความใน #{result.took_ms}ms"
result.hits.each do |hit|
  puts "  [#{hit.score.round(2)}] #{hit.title} by #{hit.author}"
  hit.highlight.each do |field, fragments|
    puts "    #{field}: #{fragments.first}"
  end
end

# Autocomplete
auto = AutocompleteService.new(es)
puts "\n=== Autocomplete ==="
suggestions = auto.suggest("Crys")
puts "Suggestions for 'Crys': #{suggestions.join(", ")}"

# Analytics
analyzer = AggregationAnalyzer.new(es)
puts "\n=== Analytics ==="
category_stats = analyzer.category_stats
puts "Stats by category:"
category_stats.each do |stat|
  puts "  #{stat[:category]}: #{stat[:count]} บทความ, avg views: #{stat[:avg_views].round(1)}"
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:
- การสร้าง Elasticsearch client ด้วย HTTP ใน Crystal
- การสร้างและจัดการ Index พร้อม custom analyzers
- การ Index เอกสารทั้งแบบ single และ bulk
- Full-Text Search ด้วย Multi-Match query
- Advanced Search พร้อม filters, sorting และ pagination
- Aggregations สำหรับวิเคราะห์ข้อมูล
- Autocomplete ด้วย Edge NGram และ Completion Suggester
- การสร้าง Search API ด้วย HTTP Server

## ขั้นตอนต่อไป

ใน **Part 155** เราจะเรียนรู้เรื่อง **Background Jobs และ Queue** ซึ่งเป็นสิ่งสำคัญสำหรับงานที่ต้องทำในพื้นหลัง เช่น การส่งอีเมล, การประมวลผลข้อมูล และการทำงานที่ใช้เวลานาน
