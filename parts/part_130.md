# Part 130: GraphQL Client - การทำ GraphQL Queries ใน Crystal

## บทนำ

GraphQL เป็น query language สำหรับ APIs ที่ช่วยให้ client ระบุได้ว่าต้องการข้อมูลอะไรบ้าง ในบทนี้เราจะเรียนรู้การทำ GraphQL queries ด้วย HTTP client ใน Crystal

## GraphQL Query พื้นฐาน

```crystal
require "http/client"
require "json"

# GraphQL Client อย่างง่าย
class GraphQLClient
  def initialize(@endpoint : String, @headers : HTTP::Headers? = nil)
  end
  
  def query(query : String, variables : Hash(String, JSON::Any::Type)? = nil) : JSON::Any
    body = build_request(query, variables: variables)
    
    headers = @headers.dup || HTTP::Headers.new
    headers["Content-Type"] = "application/json"
    headers["Accept"] = "application/json"
    
    response = HTTP::Client.post(@endpoint, headers: headers, body: body.to_json)
    
    unless response.status.ok?
      raise "GraphQL request failed: #{response.status_code} - #{response.body}"
    end
    
    data = JSON.parse(response.body)
    
    # ตรวจสอบ errors
    if errors = data["errors"]?.try(&.as_a?)
      error_messages = errors.map { |e| e["message"]?.try(&.as_s?) || "Unknown error" }
      raise "GraphQL errors: #{error_messages.join(", ")}"
    end
    
    data["data"] || JSON::Any.new(nil)
  end
  
  def mutation(mutation : String, variables : Hash(String, JSON::Any::Type)? = nil) : JSON::Any
    query(mutation, variables)
  end
  
  private def build_request(query : String, variables : Hash(String, JSON::Any::Type)? = nil) : Hash(String, JSON::Any::Type)
    request = Hash(String, JSON::Any::Type).new
    request["query"] = query
    request["variables"] = variables if variables
    request
  end
end

# ทดสอบกับ GitHub GraphQL API
client = GraphQLClient.new(
  "https://api.github.com/graphql",
  HTTP::Headers{"Authorization" => "bearer #{ENV["GITHUB_TOKEN"]? || "your-token"}"}
)

query = <<-GRAPHQL
  query {
    viewer {
      login
      name
      email
      repositories(first: 5) {
        nodes {
          name
          description
          stargazerCount
        }
      }
    }
  }
GRAPHQL

begin
  data = client.query(query)
  viewer = data["viewer"]
  puts "Login: #{viewer["login"]}"
  puts "Name: #{viewer["name"]}"
  
  repos = viewer["repositories"]["nodes"].as_a
  puts "\nRepositories:"
  repos.each { |r| puts "  - #{r["name"]}: ⭐#{r["stargazerCount"]}" }
rescue ex
  puts "Error: #{ex.message}"
end
```

## Parsing Responses

```crystal
require "json"

# Parse GraphQL response ด้วย JSON::Serializable
struct Repository
  include JSON::Serializable
  
  property name : String
  property description : String?
  property stargazer_count : Int32
  
  @[JSON::Field(key: "stargazerCount")]
  property stargazer_count : Int32 = 0
  
  @[JSON::Field(key: "primaryLanguage")]
  property primary_language : Language?
  
  struct Language
    include JSON::Serializable
    property name : String
    property color : String?
  end
end

struct User
  include JSON::Serializable
  
  property login : String
  property name : String?
  property email : String?
  property bio : String?
  
  @[JSON::Field(key: "repositories")]
  property repositories : RepositoryConnection?
  
  struct RepositoryConnection
    include JSON::Serializable
    property nodes : Array(Repository) = [] of Repository
    
    @[JSON::Field(key: "totalCount")]
    property total_count : Int32 = 0
  end
end

# Parse response
response_json = """
{
  "data": {
    "user": {
      "login": "crystal-lang",
      "name": "Crystal Language",
      "email": "crystal@example.com",
      "repositories": {
        "totalCount": 25,
        "nodes": [
          {
            "name": "crystal",
            "description": "The Crystal Programming Language",
            "stargazerCount": 18000,
            "primaryLanguage": {"name": "Crystal", "color": "#000100"}
          }
        ]
      }
    }
  }
}
"""

data = JSON.parse(response_json)["data"]
user = User.from_json(data["user"].to_json)

puts "User: #{user.name} (@#{user.login})"
if repos = user.repositories
  puts "Total repos: #{repos.total_count}"
  repos.nodes.each do |repo|
    puts "  #{repo.name}: #{repo.description}"
    puts "    ⭐ #{repo.stargazer_count} | #{repo.primary_language.try(&.name) || "Unknown"}"
  end
end
```

## Building Query Strings

```crystal
# GraphQL Query Builder
class GraphQLQueryBuilder
  def initialize
    @fields = [] of String
    @args = {} of String => String
    @fragments = [] of String
  end
  
  def field(name : String, &block : GraphQLQueryBuilder ->)
    builder = GraphQLQueryBuilder.new
    block.call(builder)
    @fields << "#{name} { #{builder.build} }"
    self
  end
  
  def field(*names : String)
    names.each { |n| @fields << n }
    self
  end
  
  def argument(name : String, value : String | Int32 | Bool | Nil)
    @args[name] = case value
    when String then "\"#{value.gsub('"', '\\"')}\""
    when Nil    then "null"
    else value.to_s
    end
    self
  end
  
  def build : String
    @fields.join("\n")
  end
  
  def to_query(operation_name : String, operation_type : String = "query") : String
    args_str = @args.empty? ? "" : "(#{@args.map { |k, v| "#{k}: #{v}" }.join(", ")})"
    "#{operation_type} #{operation_name}#{args_str} { #{build} }"
  end
end

# ตัวอย่างการสร้าง query
builder = GraphQLQueryBuilder.new

query = builder
  .field("viewer") { |q|
    q.field("login", "name", "email")
    q.field("repositories") { |r|
      r.argument("first", 10)
      r.argument("orderBy", "{field: STARGAZERS, direction: DESC}")
      r.field("nodes") { |n|
        n.field("name", "description", "stargazerCount", "url")
      }
    }
  }
  .to_query("GetViewerInfo")

puts query
```

## Fragments

```crystal
# ใช้ GraphQL Fragments เพื่อ reuse fields
module GraphQLFragments
  REPO_FIELDS = <<-GQL
    fragment RepoFields on Repository {
      id
      name
      description
      url
      stargazerCount
      forkCount
      updatedAt
      primaryLanguage {
        name
        color
      }
    }
  GQL
  
  USER_FIELDS = <<-GQL
    fragment UserFields on User {
      id
      login
      name
      email
      avatarUrl
      bio
    }
  GQL
end

# Query ที่ใช้ fragments
def get_user_repos_query(username : String) : String
  """
  #{GraphQLFragments::REPO_FIELDS}
  #{GraphQLFragments::USER_FIELDS}
  
  query GetUserRepos($username: String!, $limit: Int = 10) {
    user(login: $username) {
      ...UserFields
      repositories(first: $limit, orderBy: {field: STARGAZERS, direction: DESC}) {
        totalCount
        nodes {
          ...RepoFields
        }
      }
    }
  }
  """
end

puts get_user_repos_query("crystal-lang")
```

## Variables

```crystal
require "http/client"
require "json"

# ใช้ Variables ใน GraphQL
class GraphQLClientWithVars < GraphQLClient
  def query_with_vars(query : String, vars : Hash(String, JSON::Any)) : JSON::Any
    body = {
      "query" => query,
      "variables" => vars,
    }.to_json
    
    headers = @headers.dup || HTTP::Headers.new
    headers["Content-Type"] = "application/json"
    
    response = HTTP::Client.post(@endpoint, headers: headers, body: body)
    
    data = JSON.parse(response.body)
    raise "GraphQL error" if data["errors"]?
    
    data["data"] || JSON::Any.new(nil)
  end
end

# ตัวอย่าง query พร้อม variables
SEARCH_QUERY = <<-GQL
  query SearchRepos($query: String!, $first: Int = 10, $after: String) {
    search(query: $query, type: REPOSITORY, first: $first, after: $after) {
      repositoryCount
      pageInfo {
        hasNextPage
        endCursor
      }
      edges {
        node {
          ... on Repository {
            name
            description
            stargazerCount
            url
            owner {
              login
            }
          }
        }
      }
    }
  }
GQL

client = GraphQLClientWithVars.new(
  "https://api.github.com/graphql",
  HTTP::Headers{"Authorization" => "bearer #{ENV["GITHUB_TOKEN"]? || "token"}"}
)

vars = {
  "query" => JSON::Any.new("language:crystal stars:>100"),
  "first" => JSON::Any.new(5),
}

begin
  data = client.query_with_vars(SEARCH_QUERY, vars)
  count = data["search"]["repositoryCount"]
  puts "พบ #{count} repositories"
  
  data["search"]["edges"].as_a.each do |edge|
    repo = edge["node"]
    puts "  #{repo["owner"]["login"]}/#{repo["name"]}: ⭐#{repo["stargazerCount"]}"
  end
rescue ex
  puts "Error: #{ex.message}"
end
```

## Pagination

```crystal
require "http/client"
require "json"

# GraphQL Pagination (Cursor-based)
class GraphQLPaginator(T)
  def initialize(
    @client : GraphQLClient,
    @query : String,
    @variables : Hash(String, JSON::Any),
    @data_path : Array(String),
    @page_info_path : Array(String)
  )
  end
  
  def each_page(&block : Array(JSON::Any) ->)
    cursor = nil
    
    loop do
      vars = @variables.dup
      vars["after"] = JSON::Any.new(cursor) if cursor
      
      response = @client.query(@query, vars.transform_values { |v| v.raw })
      
      data = get_nested(response, @data_path)
      page_info = get_nested(response, @page_info_path)
      
      nodes = data["nodes"]?.try(&.as_a?) || [] of JSON::Any
      block.call(nodes)
      
      has_next = page_info["hasNextPage"]?.try(&.as_bool?) || false
      break unless has_next
      
      cursor = page_info["endCursor"]?.try(&.as_s?)
      break unless cursor
    end
  end
  
  def all : Array(JSON::Any)
    results = [] of JSON::Any
    each_page { |page| results.concat(page) }
    results
  end
  
  private def get_nested(data : JSON::Any, path : Array(String)) : JSON::Any
    path.reduce(data) { |d, key| d[key] }
  end
end

# ใช้งาน pagination
REPOS_QUERY = <<-GQL
  query GetRepos($login: String!, $first: Int = 20, $after: String) {
    user(login: $login) {
      repositories(first: $first, after: $after) {
        pageInfo {
          hasNextPage
          endCursor
        }
        nodes {
          name
          stargazerCount
        }
      }
    }
  }
GQL

client = GraphQLClient.new(
  "https://api.github.com/graphql",
  HTTP::Headers{"Authorization" => "bearer #{ENV["GITHUB_TOKEN"]? || "token"}"}
)

paginator = GraphQLPaginator(JSON::Any).new(
  client,
  REPOS_QUERY,
  {"login" => JSON::Any.new("crystal-lang"), "first" => JSON::Any.new(10)},
  ["user", "repositories"],
  ["user", "repositories", "pageInfo"]
)

all_repos = paginator.all
puts "Total repos: #{all_repos.size}"
all_repos.sort_by { |r| -r["stargazerCount"].as_i }.first(5).each do |r|
  puts "  #{r["name"]}: ⭐#{r["stargazerCount"]}"
end
```

## Subscriptions (WebSocket)

```crystal
require "http/web_socket"
require "json"

# GraphQL Subscriptions ผ่าน WebSocket
class GraphQLSubscriptionClient
  def initialize(@ws_endpoint : String, @http_endpoint : String, @token : String? = nil)
    @ws = nil
    @subscriptions = {} of String => Proc(JSON::Any, Nil)
  end
  
  def connect
    headers = HTTP::Headers.new
    headers["Authorization"] = "Bearer #{@token}" if @token
    
    @ws = HTTP::WebSocket.new(URI.parse(@ws_endpoint), headers)
    
    ws = @ws.not_nil!
    
    # Initialize connection (graphql-ws protocol)
    ws.send({"type" => "connection_init"}.to_json)
    
    ws.on_message do |msg|
      handle_message(msg)
    end
    
    ws.on_close do
      puts "Connection closed"
    end
  end
  
  def subscribe(query : String, variables : Hash(String, JSON::Any)? = nil, &block : JSON::Any ->)
    id = Random::Secure.hex(8)
    @subscriptions[id] = block
    
    payload = {
      "id" => id,
      "type" => "subscribe",
      "payload" => {
        "query" => query,
        "variables" => variables || {} of String => JSON::Any,
      },
    }
    
    @ws.try(&.send(payload.to_json))
    id
  end
  
  def unsubscribe(id : String)
    @subscriptions.delete(id)
    @ws.try(&.send({"id" => id, "type" => "complete"}.to_json))
  end
  
  def run
    @ws.try(&.run)
  end
  
  private def handle_message(msg : String)
    data = JSON.parse(msg)
    
    case data["type"]?.try(&.as_s?)
    when "connection_ack"
      puts "Connected to GraphQL server"
    when "next"
      id = data["id"]?.try(&.as_s?)
      payload = data["payload"]?
      
      if id && payload
        @subscriptions[id]?.try { |handler| handler.call(payload) }
      end
    when "error"
      puts "GraphQL error: #{data["payload"]}"
    end
  rescue JSON::ParseException
    # ignore
  end
end

# ตัวอย่าง subscription
# sub_client = GraphQLSubscriptionClient.new("ws://localhost:4000/graphql", "http://localhost:4000/graphql")
# sub_client.connect
# sub_client.subscribe("subscription { newMessage { content author } }") { |data|
#   puts "New message: #{data["data"]["newMessage"]}"
# }
# sub_client.run
```

## Batching Requests

```crystal
require "http/client"
require "json"

# Batch หลาย GraphQL queries ในคำขอเดียว
class BatchGraphQLClient
  def initialize(@endpoint : String, @headers : HTTP::Headers? = nil)
  end
  
  def batch(*requests : {String, Hash(String, JSON::Any::Type)?}) : Array(JSON::Any)
    body = requests.map do |query, variables|
      req = {"query" => query} of String => JSON::Any::Type
      req["variables"] = variables if variables
      req
    end
    
    headers = @headers.dup || HTTP::Headers.new
    headers["Content-Type"] = "application/json"
    
    response = HTTP::Client.post(@endpoint, headers: headers, body: body.to_json)
    
    unless response.status.ok?
      raise "Batch request failed: #{response.status_code}"
    end
    
    JSON.parse(response.body).as_a.map do |result|
      result["data"] || JSON::Any.new(nil)
    end
  end
end

# ตัวอย่าง batch request (ถ้า server support)
batch_client = BatchGraphQLClient.new("https://api.example.com/graphql")

query1 = "query { user(id: 1) { name email } }"
query2 = "query { product(id: 42) { name price } }"
query3 = "query { orders(userId: 1) { id total } }"

# ส่ง 3 queries พร้อมกัน
results = batch_client.batch(
  {query1, nil},
  {query2, nil},
  {query3, nil}
)

puts "User: #{results[0]}"
puts "Product: #{results[1]}"
puts "Orders: #{results[2]}"
```

## Error Handling

```crystal
require "http/client"
require "json"

# GraphQL Error Handling ที่สมบูรณ์
class GraphQLError < Exception
  property errors : Array(GraphQLErrorDetail)
  
  struct GraphQLErrorDetail
    include JSON::Serializable
    
    property message : String
    property path : Array(JSON::Any)?
    
    @[JSON::Field(key: "extensions")]
    property extensions : Hash(String, JSON::Any)?
    
    def code : String?
      extensions.try { |e| e["code"]?.try(&.as_s?) }
    end
  end
  
  def initialize(@errors)
    super(@errors.map(&.message).join(", "))
  end
end

class RobustGraphQLClient
  def initialize(@endpoint : String, @token : String? = nil)
    @headers = HTTP::Headers{
      "Content-Type" => "application/json",
      "Accept" => "application/json",
    }
    @headers["Authorization"] = "Bearer #{@token}" if @token
  end
  
  def query(query : String, variables : Hash(String, JSON::Any::Type)? = nil) : JSON::Any
    body = {"query" => query} of String => JSON::Any::Type
    body["variables"] = variables if variables
    
    response = HTTP::Client.post(@endpoint, headers: @headers, body: body.to_json)
    
    case response.status_code
    when 200..299
      handle_response(response.body)
    when 401
      raise "Authentication required"
    when 403
      raise "Access denied"
    when 404
      raise "GraphQL endpoint not found"
    when 429
      raise "Rate limited"
    when 500..599
      raise "Server error: #{response.status_code}"
    else
      raise "HTTP Error: #{response.status_code}"
    end
  rescue IO::TimeoutError
    raise "GraphQL request timed out"
  rescue Socket::Error => ex
    raise "Network error: #{ex.message}"
  end
  
  private def handle_response(body : String) : JSON::Any
    data = JSON.parse(body)
    
    # ตรวจสอบ GraphQL errors
    if errors_json = data["errors"]?
      errors = errors_json.as_a.map { |e| GraphQLError::GraphQLErrorDetail.from_json(e.to_json) }
      raise GraphQLError.new(errors)
    end
    
    data["data"] || JSON::Any.new(nil)
  end
end

# ทดสอบ error handling
client = RobustGraphQLClient.new("https://api.github.com/graphql", ENV["GITHUB_TOKEN"]?)

begin
  data = client.query("{ viewer { login } }")
  puts "Login: #{data["viewer"]["login"]}"
rescue GraphQLError => ex
  puts "GraphQL Error: #{ex.message}"
  ex.errors.each do |err|
    puts "  - #{err.message} (code: #{err.code})"
  end
rescue ex
  puts "Error: #{ex.message}"
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: GitHub Stats Dashboard

```crystal
require "http/client"
require "json"

class GitHubGraphQLStats
  STATS_QUERY = <<-GQL
    query GitHubStats($login: String!) {
      user(login: $login) {
        login
        name
        bio
        followers { totalCount }
        following { totalCount }
        repositories(first: 100, ownerAffiliations: OWNER) {
          totalCount
          nodes {
            name
            stargazerCount
            forkCount
            primaryLanguage { name }
          }
        }
        contributionsCollection {
          totalCommitContributions
          totalPullRequestContributions
          totalIssueContributions
        }
      }
    }
  GQL
  
  def initialize(@token : String)
    @client = GraphQLClient.new(
      "https://api.github.com/graphql",
      HTTP::Headers{"Authorization" => "bearer #{@token}"}
    )
  end
  
  def stats(username : String)
    vars = {"login" => JSON::Any.new(username)}
    data = @client.query(STATS_QUERY, vars.transform_values { |v| v.raw })
    
    user = data["user"]
    puts "=== GitHub Stats for @#{user["login"]} ==="
    puts "Name: #{user["name"]}"
    puts "Followers: #{user["followers"]["totalCount"]}"
    puts "Following: #{user["following"]["totalCount"]}"
    
    repos = user["repositories"]
    puts "\nRepositories: #{repos["totalCount"]}"
    
    total_stars = repos["nodes"].as_a.sum { |r| r["stargazerCount"].as_i }
    puts "Total Stars: #{total_stars}"
    
    languages = repos["nodes"].as_a
      .map { |r| r["primaryLanguage"]["name"].as_s rescue "Unknown" }
      .tally
      .sort_by { |_, count| -count }
      .first(5)
    
    puts "\nTop Languages:"
    languages.each { |lang, count| puts "  #{lang}: #{count} repos" }
    
    contrib = user["contributionsCollection"]
    puts "\nContributions:"
    puts "  Commits: #{contrib["totalCommitContributions"]}"
    puts "  Pull Requests: #{contrib["totalPullRequestContributions"]}"
    puts "  Issues: #{contrib["totalIssueContributions"]}"
  rescue ex
    puts "Error: #{ex.message}"
  end
end

stats = GitHubGraphQLStats.new(ENV["GITHUB_TOKEN"]? || "your-token")
stats.stats("crystal-lang")
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GraphQL Client**: การส่ง queries และ mutations ผ่าน HTTP
2. **Parsing Responses**: ใช้ JSON::Serializable สำหรับ parse response
3. **Query Builder**: สร้าง query strings แบบ programmatic
4. **Fragments**: การ reuse fields ใน GraphQL
5. **Variables**: ส่ง dynamic values ผ่าน variables
6. **Pagination**: cursor-based pagination
7. **Subscriptions**: real-time data ผ่าน WebSocket
8. **Batching**: ส่งหลาย queries ในคำขอเดียว
9. **Error Handling**: จัดการ GraphQL errors
10. **GitHub GraphQL API**: ตัวอย่างการใช้งานจริง
