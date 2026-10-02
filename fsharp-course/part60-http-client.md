# Part 60 - HTTP Client

## บทนำ

HTTP Client ใน F# ใช้สำหรับส่ง HTTP requests ไปยัง external services, APIs, หรือ microservices อื่น บทนี้จะครอบคลุมวิธีการใช้งาน HttpClient อย่างถูกต้องและมีประสิทธิภาพ

---

## 1. HttpClient ใน F#

```fsharp
open System.Net.Http
open System.Text.Json

// ไม่ควรสร้าง HttpClient ใหม่ทุกครั้ง (socket exhaustion)
// ควรใช้ shared instance หรือ HttpClientFactory

// Bad - สร้างใหม่ทุกครั้ง (ห้ามใช้!)
let badPractice () =
    use client = new HttpClient()  // Don't do this!
    client

// Good - shared instance (สำหรับ single application)
let sharedClient = new HttpClient()

// Better - HttpClientFactory (แนะนำ)
// ดูหัวข้อ HttpClientFactory ด้านล่าง

// Basic GET request
let simpleGet () =
    task {
        use client = new HttpClient()
        let! response = client.GetAsync("https://api.example.com/data")
        let! content = response.Content.ReadAsStringAsync()
        return content
    }

// GET with base URL
let clientWithBaseUrl () =
    let client = new HttpClient()
    client.BaseAddress <- System.Uri("https://api.example.com")
    client.DefaultRequestHeaders.Accept.Add(
        System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"))
    client.Timeout <- System.TimeSpan.FromSeconds(30)
    client
```

---

## 2. Async HTTP Requests

```fsharp
open System.Net.Http
open System.Text.Json
open System.Threading.Tasks

let client = new HttpClient()
client.BaseAddress <- System.Uri("https://jsonplaceholder.typicode.com")

// Async GET
let asyncGet (url: string) =
    task {
        let! response = client.GetAsync(url)
        response.EnsureSuccessStatusCode() |> ignore
        let! content = response.Content.ReadAsStringAsync()
        return content
    }

// Async GET with deserialization
let asyncGetJson<'T> (url: string) =
    task {
        let! response = client.GetAsync(url)
        response.EnsureSuccessStatusCode() |> ignore
        let! result = response.Content.ReadFromJsonAsync<'T>()
        return result
    }

// Parallel requests
let parallelGet (urls: string list) =
    task {
        let tasks = urls |> List.map (fun url -> asyncGet url)
        let! results = Task.WhenAll(tasks)
        return results |> Array.toList
    }

// Sequential requests
let sequentialGet (urls: string list) =
    task {
        let results = ResizeArray<string>()
        for url in urls do
            let! result = asyncGet url
            results.Add(result)
        return results |> Seq.toList
    }

// Streaming response
let streamResponse (url: string) =
    task {
        let! response = client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead)
        response.EnsureSuccessStatusCode() |> ignore
        use! stream = response.Content.ReadAsStreamAsync()
        use reader = new System.IO.StreamReader(stream)
        
        let mutable line = ""
        let results = ResizeArray<string>()
        
        while not reader.EndOfStream do
            let! l = reader.ReadLineAsync()
            line <- l
            if line <> null then
                results.Add(line)
        
        return results |> Seq.toList
    }
```

---

## 3. GET, POST, PUT, DELETE

```fsharp
open System.Net.Http
open System.Net.Http.Json
open System.Text.Json

let baseClient =
    let client = new HttpClient()
    client.BaseAddress <- System.Uri("https://api.example.com")
    client.DefaultRequestHeaders.Add("Accept", "application/json")
    client

// GET request
let getUser (id: int) =
    task {
        let! response = baseClient.GetAsync($"/api/users/{id}")
        
        if response.IsSuccessStatusCode then
            let! user = response.Content.ReadFromJsonAsync<User>()
            return Ok user
        elif response.StatusCode = System.Net.HttpStatusCode.NotFound then
            return Error "User not found"
        else
            return Error $"Request failed: {response.StatusCode}"
    }

// GET list
let getUsers () =
    task {
        let! users = baseClient.GetFromJsonAsync<User list>("/api/users")
        return users
    }

// POST - Create
type CreateUserRequest = {
    Name: string
    Email: string
    Role: string
}

let createUser (request: CreateUserRequest) =
    task {
        let! response = baseClient.PostAsJsonAsync("/api/users", request)
        
        if response.IsSuccessStatusCode then
            let! created = response.Content.ReadFromJsonAsync<User>()
            return Ok created
        else
            let! error = response.Content.ReadAsStringAsync()
            return Error $"Failed to create: {error}"
    }

// POST with string content
let postWithStringContent (url: string) (jsonBody: string) =
    task {
        use content = new System.Net.Http.StringContent(
            jsonBody,
            System.Text.Encoding.UTF8,
            "application/json")
        
        let! response = baseClient.PostAsync(url, content)
        response.EnsureSuccessStatusCode() |> ignore
        let! result = response.Content.ReadAsStringAsync()
        return result
    }

// PUT - Full replace
let replaceUser (id: int) (user: CreateUserRequest) =
    task {
        let! response = baseClient.PutAsJsonAsync($"/api/users/{id}", user)
        
        if response.IsSuccessStatusCode then
            let! updated = response.Content.ReadFromJsonAsync<User>()
            return Ok updated
        else
            return Error $"Failed: {response.StatusCode}"
    }

// PATCH - Partial update
let patchUser (id: int) (patch: {| Name: string option; Email: string option |}) =
    task {
        // PATCH usually needs JsonPatch or custom format
        use content = new System.Net.Http.StringContent(
            JsonSerializer.Serialize(patch),
            System.Text.Encoding.UTF8,
            "application/merge-patch+json")
        
        use request = new HttpRequestMessage(new HttpMethod("PATCH"), $"/api/users/{id}")
        request.Content <- content
        
        let! response = baseClient.SendAsync(request)
        
        if response.IsSuccessStatusCode then
            let! patched = response.Content.ReadFromJsonAsync<User>()
            return Ok patched
        else
            return Error $"Patch failed: {response.StatusCode}"
    }

// DELETE
let deleteUser (id: int) =
    task {
        let! response = baseClient.DeleteAsync($"/api/users/{id}")
        
        match response.StatusCode with
        | System.Net.HttpStatusCode.NoContent -> return Ok ()
        | System.Net.HttpStatusCode.NotFound -> return Error "User not found"
        | code -> return Error $"Unexpected status: {code}"
    }
```

---

## 4. Setting Headers

```fsharp
open System.Net.Http
open System.Net.Http.Headers

// Default headers สำหรับทุก request
let configuredClient =
    let client = new HttpClient()
    client.BaseAddress <- System.Uri("https://api.example.com")
    
    // Accept headers
    client.DefaultRequestHeaders.Accept.Clear()
    client.DefaultRequestHeaders.Accept.Add(MediaTypeWithQualityHeaderValue("application/json"))
    
    // Authorization
    client.DefaultRequestHeaders.Authorization <- 
        AuthenticationHeaderValue("Bearer", "my-token-here")
    
    // Custom headers
    client.DefaultRequestHeaders.Add("X-API-Key", "my-api-key")
    client.DefaultRequestHeaders.Add("X-Client-Version", "1.0.0")
    client.DefaultRequestHeaders.Add("Accept-Language", "th-TH, en-US")
    
    // User Agent
    client.DefaultRequestHeaders.UserAgent.Add(
        ProductInfoHeaderValue("MyApp", "1.0"))
    
    client

// Per-request headers
let requestWithCustomHeaders (url: string) (token: string) =
    task {
        use request = new HttpRequestMessage(HttpMethod.Get, url)
        
        // Add headers to specific request
        request.Headers.Authorization <- AuthenticationHeaderValue("Bearer", token)
        request.Headers.Add("X-Request-Id", System.Guid.NewGuid().ToString())
        request.Headers.Add("X-Timestamp", System.DateTime.UtcNow.ToString("O"))
        
        let! response = configuredClient.SendAsync(request)
        let! content = response.Content.ReadAsStringAsync()
        return content
    }

// Conditional request headers
let conditionalGet (url: string) (etag: string option) (lastModified: System.DateTime option) =
    task {
        use request = new HttpRequestMessage(HttpMethod.Get, url)
        
        match etag with
        | Some tag -> request.Headers.IfNoneMatch.Add(EntityTagHeaderValue($"\"{tag}\""))
        | None -> ()
        
        match lastModified with
        | Some date -> request.Headers.IfModifiedSince <- System.DateTimeOffset(date) |> System.Nullable
        | None -> ()
        
        let! response = configuredClient.SendAsync(request)
        
        if response.StatusCode = System.Net.HttpStatusCode.NotModified then
            return None  // Use cached version
        else
            let! content = response.Content.ReadAsStringAsync()
            return Some content
    }

// Content-Type headers
let postWithContentType (url: string) (data: obj) (contentType: string) =
    task {
        let json = System.Text.Json.JsonSerializer.Serialize(data)
        use content = new StringContent(json, System.Text.Encoding.UTF8, contentType)
        
        let! response = configuredClient.PostAsync(url, content)
        return response.StatusCode
    }
```

---

## 5. JSON Body

```fsharp
open System.Net.Http
open System.Net.Http.Json
open System.Text.Json
open System.Text.Json.Serialization

// Custom JSON options
let jsonOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    opts.WriteIndented <- false
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.WhenWritingNull
    opts.Converters.Add(JsonFSharpConverter())
    opts

let client = new HttpClient()
client.BaseAddress <- System.Uri("https://api.example.com")

// POST JSON with default serializer
let postJson<'TRequest, 'TResponse> (url: string) (data: 'TRequest) =
    task {
        let! response = client.PostAsJsonAsync(url, data, jsonOptions)
        response.EnsureSuccessStatusCode() |> ignore
        let! result = response.Content.ReadFromJsonAsync<'TResponse>(jsonOptions)
        return result
    }

// GET JSON with deserialization
let getJson<'T> (url: string) =
    task {
        let! result = client.GetFromJsonAsync<'T>(url, jsonOptions)
        return result
    }

// Manual JSON serialization for complex scenarios
let sendComplexRequest (url: string) (data: ComplexType) =
    task {
        let json = JsonSerializer.Serialize(data, jsonOptions)
        use content = new StringContent(json, System.Text.Encoding.UTF8, "application/json")
        
        use request = new HttpRequestMessage(HttpMethod.Post, url)
        request.Content <- content
        request.Headers.Add("X-Custom", "header-value")
        
        let! response = client.SendAsync(request)
        
        if not response.IsSuccessStatusCode then
            let! errorBody = response.Content.ReadAsStringAsync()
            failwith $"Request failed {response.StatusCode}: {errorBody}"
        
        let! responseJson = response.Content.ReadAsStringAsync()
        return JsonSerializer.Deserialize<ResponseType>(responseJson, jsonOptions)
    }

// Batch requests
type BatchRequest = {
    Requests: HttpRequestItem list
}

and HttpRequestItem = {
    Method: string
    Url: string
    Body: System.Text.Json.JsonElement option
}

let sendBatchRequests (items: (HttpMethod * string * obj option) list) =
    task {
        let tasks =
            items
            |> List.map (fun (method, url, body) ->
                task {
                    use request = new HttpRequestMessage(method, url)
                    
                    match body with
                    | Some b ->
                        let json = JsonSerializer.Serialize(b, jsonOptions)
                        request.Content <- new StringContent(json, System.Text.Encoding.UTF8, "application/json")
                    | None -> ()
                    
                    let! response = client.SendAsync(request)
                    let! content = response.Content.ReadAsStringAsync()
                    return (response.StatusCode, content)
                })
        
        let! results = System.Threading.Tasks.Task.WhenAll(tasks)
        return results |> Array.toList
    }
```

---

## 6. Reading Response

```fsharp
open System.Net.Http
open System.Net.Http.Json
open System.Text.Json

let client = new HttpClient()

// Read response details
let readFullResponse (url: string) =
    task {
        let! response = client.GetAsync(url)
        
        // Status
        let status = response.StatusCode
        let isSuccess = response.IsSuccessStatusCode
        
        // Headers
        let headers = response.Headers |> Seq.map (fun h -> h.Key, h.Value |> Seq.toList) |> Map.ofSeq
        let contentType = response.Content.Headers.ContentType?.MediaType
        let contentLength = response.Content.Headers.ContentLength
        
        // Body
        let! body = response.Content.ReadAsStringAsync()
        
        return {|
            statusCode = int status
            isSuccess = isSuccess
            headers = headers
            contentType = contentType
            contentLength = contentLength
            body = body
        |}
    }

// Read JSON with error handling
let readJsonSafe<'T> (response: HttpResponseMessage) =
    task {
        let! body = response.Content.ReadAsStringAsync()
        
        if response.IsSuccessStatusCode then
            try
                let result = JsonSerializer.Deserialize<'T>(body)
                return Ok result
            with ex ->
                return Error $"JSON parse error: {ex.Message}"
        else
            return Error $"HTTP {int response.StatusCode}: {body}"
    }

// Typed response wrapper
type HttpResult<'T> =
    | Success of 'T * int  // data, status code
    | ClientError of int * string  // status, error message
    | ServerError of int * string
    | NetworkError of exn

let safeRequest<'T> (url: string) =
    task {
        try
            let! response = client.GetAsync(url)
            let statusCode = int response.StatusCode
            
            if response.IsSuccessStatusCode then
                let! data = response.Content.ReadFromJsonAsync<'T>()
                return Success(data, statusCode)
            elif statusCode >= 400 && statusCode < 500 then
                let! errorMsg = response.Content.ReadAsStringAsync()
                return ClientError(statusCode, errorMsg)
            else
                let! errorMsg = response.Content.ReadAsStringAsync()
                return ServerError(statusCode, errorMsg)
        with ex ->
            return NetworkError ex
    }

// Handle response in pattern matching style
let processResponse (url: string) =
    task {
        let! result = safeRequest<User list> url
        
        match result with
        | Success(users, 200) ->
            printfn $"Got {users.Length} users"
            return Ok users
        | Success(_, code) ->
            return Error $"Unexpected success status: {code}"
        | ClientError(404, _) ->
            return Error "Resource not found"
        | ClientError(401, _) ->
            return Error "Authentication required"
        | ClientError(code, msg) ->
            return Error $"Client error {code}: {msg}"
        | ServerError(code, msg) ->
            return Error $"Server error {code}: {msg}"
        | NetworkError ex ->
            return Error $"Network error: {ex.Message}"
    }
```

---

## 7. Status Code Handling

```fsharp
open System.Net.Http
open System.Net

// Comprehensive status code handling
let handleStatusCode (response: HttpResponseMessage) =
    task {
        let! body = response.Content.ReadAsStringAsync()
        
        match response.StatusCode with
        // 2xx Success
        | HttpStatusCode.OK ->
            return Ok body
        | HttpStatusCode.Created ->
            let location = response.Headers.Location?.ToString()
            return Ok $"Created at: {location}"
        | HttpStatusCode.NoContent ->
            return Ok ""
        | HttpStatusCode.Accepted ->
            return Ok "Request accepted for processing"
        
        // 3xx Redirect (normally handled automatically)
        | HttpStatusCode.NotModified ->
            return Error "304 Not Modified - use cached version"
        
        // 4xx Client Errors
        | HttpStatusCode.BadRequest ->
            return Error $"400 Bad Request: {body}"
        | HttpStatusCode.Unauthorized ->
            return Error "401 Unauthorized - check credentials"
        | HttpStatusCode.Forbidden ->
            return Error "403 Forbidden - insufficient permissions"
        | HttpStatusCode.NotFound ->
            return Error "404 Not Found - resource doesn't exist"
        | HttpStatusCode.MethodNotAllowed ->
            let allowed = response.Headers.Allow |> Seq.toList
            return Error $"405 Method Not Allowed. Allowed: {allowed}"
        | HttpStatusCode.Conflict ->
            return Error $"409 Conflict: {body}"
        | HttpStatusCode.Gone ->
            return Error "410 Gone - resource permanently deleted"
        | HttpStatusCode.UnprocessableEntity ->
            return Error $"422 Validation Error: {body}"
        | HttpStatusCode.TooManyRequests ->
            let retryAfter = response.Headers.RetryAfter?.Delta?.TotalSeconds
            return Error $"429 Rate Limited. Retry after {retryAfter} seconds"
        
        // 5xx Server Errors
        | HttpStatusCode.InternalServerError ->
            return Error $"500 Server Error: {body}"
        | HttpStatusCode.BadGateway ->
            return Error "502 Bad Gateway - upstream service unavailable"
        | HttpStatusCode.ServiceUnavailable ->
            let retryAfter = response.Headers.RetryAfter?.Delta?.TotalSeconds
            return Error $"503 Service Unavailable. Retry after {retryAfter} seconds"
        | HttpStatusCode.GatewayTimeout ->
            return Error "504 Gateway Timeout - upstream service timed out"
        
        | code ->
            return Error $"Unexpected status: {int code}"
    }
```

---

## 8. Error Handling

```fsharp
open System.Net.Http
open System.Threading
open System.Threading.Tasks

let client = new HttpClient()
client.Timeout <- System.TimeSpan.FromSeconds(30)

// Error types
type HttpError =
    | Timeout
    | NetworkError of string
    | HttpError of int * string  // status code, body
    | ParseError of string
    | Cancelled

// Safe request with comprehensive error handling
let safeGet<'T> (url: string) (cancellationToken: CancellationToken) =
    task {
        try
            let! response = client.GetAsync(url, cancellationToken)
            
            if not response.IsSuccessStatusCode then
                let! body = response.Content.ReadAsStringAsync()
                return Error (HttpError(int response.StatusCode, body))
            else
                try
                    let! result = response.Content.ReadFromJsonAsync<'T>(cancellationToken = cancellationToken)
                    return Ok result
                with ex ->
                    return Error (ParseError ex.Message)
        with
        | :? TaskCanceledException when not cancellationToken.IsCancellationRequested ->
            return Error Timeout
        | :? TaskCanceledException ->
            return Error Cancelled
        | :? HttpRequestException as ex ->
            return Error (NetworkError ex.Message)
        | ex ->
            return Error (NetworkError ex.Message)
    }

// Request with retry
let requestWithRetry<'T> (url: string) (maxRetries: int) =
    task {
        let mutable attempt = 0
        let mutable result: Result<'T, HttpError> = Error (NetworkError "Not started")
        let mutable shouldRetry = true
        
        while shouldRetry && attempt < maxRetries do
            attempt <- attempt + 1
            
            let! r = safeGet<'T> url CancellationToken.None
            result <- r
            
            match r with
            | Ok _ ->
                shouldRetry <- false
            | Error (HttpError(429, _)) ->
                // Rate limited - wait longer
                let delay = attempt * 2000  // Exponential backoff
                printfn $"Rate limited, waiting {delay}ms before retry {attempt}/{maxRetries}"
                do! Task.Delay(delay)
            | Error (HttpError(code, _)) when code >= 500 ->
                // Server error - retry
                let delay = attempt * 1000
                printfn $"Server error, retry {attempt}/{maxRetries} after {delay}ms"
                do! Task.Delay(delay)
            | Error _ ->
                // Client error - don't retry
                shouldRetry <- false
        
        return result
    }
```

---

## 9. HttpClientFactory

HttpClientFactory เป็นวิธีที่แนะนำสำหรับการจัดการ HttpClient ใน ASP.NET Core

```fsharp
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open System.Net.Http

// Register HttpClientFactory
let configureServices (services: IServiceCollection) =
    // Basic HttpClient
    services.AddHttpClient() |> ignore
    
    // Named HttpClient
    services.AddHttpClient("github", fun client ->
        client.BaseAddress <- System.Uri("https://api.github.com")
        client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json")
        client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0"))
    |> ignore
    
    // Typed HttpClient
    services.AddHttpClient<GithubService>(fun client ->
        client.BaseAddress <- System.Uri("https://api.github.com")
        client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json")
        client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0"))
    |> ignore

// Use IHttpClientFactory
type ApiClient(httpClientFactory: IHttpClientFactory) =
    let client = httpClientFactory.CreateClient("github")
    
    member _.GetRepos (username: string) =
        task {
            let! response = client.GetAsync($"/users/{username}/repos")
            response.EnsureSuccessStatusCode() |> ignore
            return! response.Content.ReadAsStringAsync()
        }

// Typed HttpClient service
type GithubService(client: HttpClient) =
    member _.GetUser (username: string) =
        task {
            let! response = client.GetAsync($"/users/{username}")
            response.EnsureSuccessStatusCode() |> ignore
            return! response.Content.ReadFromJsonAsync<GithubUser>()
        }
    
    member _.GetRepos (username: string) =
        task {
            let! repos = client.GetFromJsonAsync<GithubRepo list>($"/users/{username}/repos")
            return repos
        }

// Register in DI
let configureFullServices (services: IServiceCollection) =
    services.AddHttpClient<GithubService>(fun client ->
        client.BaseAddress <- System.Uri("https://api.github.com")
        client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json")
        client.DefaultRequestHeaders.Add("User-Agent", "MyApp"))
    |> ignore
    
    services.AddTransient<ApiClient>() |> ignore
```

---

## 10. Retry Policies with Polly

```bash
dotnet add package Polly
dotnet add package Microsoft.Extensions.Http.Polly
```

```fsharp
open Polly
open Polly.Extensions.Http
open System.Net.Http
open Microsoft.Extensions.DependencyInjection

// Retry policy
let retryPolicy =
    HttpPolicyExtensions
        .HandleTransientHttpError()  // Handles 5xx and network errors
        .OrResult(fun msg -> msg.StatusCode = System.Net.HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(
            retryCount = 3,
            sleepDurationProvider = (fun retryAttempt -> 
                System.TimeSpan.FromSeconds(float (System.Math.Pow(2, float retryAttempt)))),
            onRetry = (fun outcome retryDelay retryCount context ->
                printfn $"Retry {retryCount} after {retryDelay.TotalSeconds}s. Reason: {outcome.Exception?.Message ?? string outcome.Result?.StatusCode}"))

// Timeout policy
let timeoutPolicy =
    Policy.TimeoutAsync<HttpResponseMessage>(System.TimeSpan.FromSeconds(10))

// Circuit breaker policy
let circuitBreakerPolicy =
    HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking = 5,
            durationOfBreak = System.TimeSpan.FromSeconds(30),
            onBreak = (fun outcome breakDelay ->
                printfn $"Circuit OPEN for {breakDelay.TotalSeconds}s. Reason: {outcome.Exception?.Message ?? string outcome.Result?.StatusCode}"),
            onReset = (fun () ->
                printfn "Circuit CLOSED - resuming normal operation"),
            onHalfOpen = (fun () ->
                printfn "Circuit HALF-OPEN - testing connection"))

// Combine policies (wrap - inner executes first)
let combinedPolicy = 
    Policy.WrapAsync(timeoutPolicy, retryPolicy, circuitBreakerPolicy)

// Register with HttpClientFactory
let configurePolly (services: IServiceCollection) =
    services
        .AddHttpClient<ResilientService>(fun client ->
            client.BaseAddress <- System.Uri("https://api.example.com"))
        .AddTransientHttpErrorPolicy(fun policy ->
            policy.WaitAndRetryAsync(3, fun retryAttempt ->
                System.TimeSpan.FromSeconds(float retryAttempt)))
        .AddTransientHttpErrorPolicy(fun policy ->
            policy.CircuitBreakerAsync(5, System.TimeSpan.FromSeconds(30)))
    |> ignore

// Use Polly with typed client
type ResilientService(client: HttpClient) =
    member _.GetData () =
        task {
            let! response = client.GetAsync("/api/data")
            response.EnsureSuccessStatusCode() |> ignore
            return! response.Content.ReadAsStringAsync()
        }
```

---

## 11. Circuit Breaker with Polly

```fsharp
open Polly
open Polly.CircuitBreaker
open System.Net.Http

// Circuit breaker states:
// CLOSED: ทำงานปกติ
// OPEN: หยุดส่ง request (เมื่อ failures เกิน threshold)
// HALF-OPEN: ทดสอบ 1 request เพื่อดูว่า service กลับมาแล้วหรือยัง

type CircuitBreakerService() =
    let mutable breakerState = "CLOSED"
    
    let circuitBreaker =
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking = 3,
                durationOfBreak = System.TimeSpan.FromSeconds(60),
                onBreak = (fun ex duration ->
                    breakerState <- "OPEN"
                    printfn $"Circuit breaker opened for {duration.TotalSeconds}s"),
                onReset = fun () ->
                    breakerState <- "CLOSED"
                    printfn "Circuit breaker reset",
                onHalfOpen = fun () ->
                    breakerState <- "HALF-OPEN"
                    printfn "Circuit breaker half-open")
    
    member _.State = breakerState
    
    member _.ExecuteAsync (url: string) =
        task {
            use client = new HttpClient()
            
            try
                let! result = circuitBreaker.ExecuteAsync(fun () ->
                    client.GetAsync(url))
                return Ok (result.StatusCode)
            with
            | :? BrokenCircuitException ->
                return Error "Circuit breaker is open - service unavailable"
            | ex ->
                return Error ex.Message
        }

// Advanced circuit breaker with fallback
let withFallback (primary: unit -> Task<'T>) (fallback: unit -> Task<'T>) =
    let breaker = 
        Policy
            .Handle<HttpRequestException>()
            .CircuitBreakerAsync(3, System.TimeSpan.FromSeconds(30))
    
    let fallbackPolicy =
        Policy<'T>
            .Handle<BrokenCircuitException>()
            .OrHandle<HttpRequestException>()
            .FallbackAsync(fun ct -> fallback())
    
    let combined = Policy.WrapAsync(fallbackPolicy, breaker)
    
    combined.ExecuteAsync(fun () -> primary())
```

---

## 12. TypedHttpClient

```fsharp
open System.Net.Http
open System.Net.Http.Json
open System.Text.Json
open Microsoft.Extensions.DependencyInjection

// Define domain types
type WeatherData = {
    City: string
    Temperature: float
    Description: string
    Humidity: int
    WindSpeed: float
    UpdatedAt: System.DateTime
}

type WeatherForecast = {
    City: string
    Forecasts: DailyForecast list
}

and DailyForecast = {
    Date: System.DateOnly
    HighTemp: float
    LowTemp: float
    Description: string
}

// Typed HTTP Client
type WeatherApiClient(httpClient: HttpClient) =
    let opts = 
        let o = JsonSerializerOptions()
        o.PropertyNameCaseInsensitive <- true
        o
    
    member _.GetCurrentWeather (city: string) =
        task {
            let! response = httpClient.GetAsync($"/weather/current?city={city}")
            
            if response.IsSuccessStatusCode then
                let! data = response.Content.ReadFromJsonAsync<WeatherData>(opts)
                return Ok data
            else
                let! error = response.Content.ReadAsStringAsync()
                return Error $"Failed to get weather: {error}"
        }
    
    member _.GetForecast (city: string) (days: int) =
        task {
            let! result = httpClient.GetFromJsonAsync<WeatherForecast>(
                $"/weather/forecast?city={city}&days={days}", opts)
            return result
        }
    
    member _.GetMultipleCities (cities: string list) =
        task {
            let tasks = cities |> List.map (fun city ->
                task {
                    let! result = httpClient.GetFromJsonAsync<WeatherData>($"/weather/current?city={city}", opts)
                    return (city, result)
                })
            
            let! results = System.Threading.Tasks.Task.WhenAll(tasks)
            return results |> Array.toList |> Map.ofList
        }

// Register typed client
let configureWeatherClient (services: IServiceCollection) (apiKey: string) =
    services.AddHttpClient<WeatherApiClient>(fun client ->
        client.BaseAddress <- System.Uri("https://api.weatherservice.com")
        client.DefaultRequestHeaders.Add("X-API-Key", apiKey)
        client.DefaultRequestHeaders.Add("Accept", "application/json")
        client.Timeout <- System.TimeSpan.FromSeconds(10))
    .AddTransientHttpErrorPolicy(fun policy ->
        policy.WaitAndRetryAsync(3, fun attempt ->
            System.TimeSpan.FromSeconds(float attempt * 2.0)))
    |> ignore
```

---

## 13. FSharp.Data.Http

```bash
dotnet add package FSharp.Data
```

```fsharp
open FSharp.Data

// FSharp.Data.Http provides simple HTTP utilities

// Simple GET
let simpleGet () =
    let response = Http.RequestString("https://api.example.com/data")
    printfn "Response: %s" response

// GET with options
let getWithOptions () =
    let response = Http.RequestString(
        "https://api.example.com/users",
        httpMethod = "GET",
        headers = [
            "Accept", "application/json"
            "Authorization", "Bearer token-here"
        ],
        query = [
            "page", "1"
            "limit", "20"
        ]
    )
    response

// POST with JSON body
let postJson () =
    let body = HttpRequestBody.TextRequest """{"name": "Test", "email": "test@example.com"}"""
    
    let response = Http.Request(
        "https://api.example.com/users",
        httpMethod = "POST",
        headers = ["Content-Type", "application/json"],
        body = body
    )
    
    printfn "Status: %d" response.StatusCode
    match response.Body with
    | Text t -> printfn "Body: %s" t
    | Binary b -> printfn "Binary: %d bytes" b.Length

// Async version
let asyncGet () =
    async {
        let! response = Http.AsyncRequest(
            "https://api.example.com/data",
            headers = ["Accept", "application/json"]
        )
        
        return match response.Body with
               | Text t -> t
               | Binary b -> System.Text.Encoding.UTF8.GetString(b)
    }

// Type Provider with JSON
type GitHubUser = JsonProvider<"https://api.github.com/users/github">

let getGitHubUser (username: string) =
    async {
        let! user = GitHubUser.AsyncLoad($"https://api.github.com/users/{username}")
        return {|
            login = user.Login
            name = user.Name
            company = user.Company
            followers = user.Followers
            repos = user.PublicRepos
        |}
    }
```

---

## 14. Real API Integration Examples

```fsharp
open System.Net.Http
open System.Net.Http.Json
open System.Text.Json

// GitHub API integration
type GitHubApiClient(httpClient: HttpClient) =
    do
        httpClient.BaseAddress <- System.Uri("https://api.github.com")
        httpClient.DefaultRequestHeaders.Add("User-Agent", "MyApp")
        httpClient.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json")
    
    member _.GetUser (username: string) =
        task {
            let! response = httpClient.GetAsync($"/users/{username}")
            
            if response.StatusCode = System.Net.HttpStatusCode.NotFound then
                return None
            else
                response.EnsureSuccessStatusCode() |> ignore
                let! user = response.Content.ReadFromJsonAsync<GitHubUser>()
                return Some user
        }
    
    member _.GetRepos (username: string) =
        task {
            let! repos = httpClient.GetFromJsonAsync<GitHubRepo list>($"/users/{username}/repos?sort=updated&per_page=10")
            return repos |> Option.ofObj |> Option.defaultValue []
        }
    
    member _.SearchRepositories (query: string) =
        task {
            let encodedQuery = System.Uri.EscapeDataString(query)
            let! result = httpClient.GetFromJsonAsync<SearchResult>($"/search/repositories?q={encodedQuery}&sort=stars&order=desc&per_page=10")
            return result
        }

// Stripe Payment API integration
type StripeClient(httpClient: HttpClient, secretKey: string) =
    do
        httpClient.BaseAddress <- System.Uri("https://api.stripe.com")
        let credentials = System.Convert.ToBase64String(System.Text.Encoding.ASCII.GetBytes($"{secretKey}:"))
        httpClient.DefaultRequestHeaders.Authorization <- 
            System.Net.Http.Headers.AuthenticationHeaderValue("Basic", credentials)
    
    member _.CreatePaymentIntent (amount: int) (currency: string) =
        task {
            let data = System.Collections.Generic.Dictionary<string, string>()
            data.["amount"] <- string amount
            data.["currency"] <- currency
            
            use content = new System.Net.Http.FormUrlEncodedContent(data)
            let! response = httpClient.PostAsync("/v1/payment_intents", content)
            response.EnsureSuccessStatusCode() |> ignore
            let! pi = response.Content.ReadFromJsonAsync<PaymentIntent>()
            return pi
        }
    
    member _.GetPaymentIntent (id: string) =
        task {
            let! pi = httpClient.GetFromJsonAsync<PaymentIntent>($"/v1/payment_intents/{id}")
            return pi
        }

// Slack Webhook integration
type SlackWebhookClient(httpClient: HttpClient, webhookUrl: string) =
    member _.SendMessage (text: string) =
        task {
            let payload = {| text = text |}
            let! response = httpClient.PostAsJsonAsync(webhookUrl, payload)
            return response.IsSuccessStatusCode
        }
    
    member _.SendRichMessage (blocks: obj list) =
        task {
            let payload = {| blocks = blocks |}
            let! response = httpClient.PostAsJsonAsync(webhookUrl, payload)
            return response.IsSuccessStatusCode
        }

// OpenWeather API integration
type WeatherClient(httpClient: HttpClient, apiKey: string) =
    do
        httpClient.BaseAddress <- System.Uri("https://api.openweathermap.org")
    
    member _.GetCurrentWeather (city: string) =
        task {
            let! response = httpClient.GetAsync($"/data/2.5/weather?q={city}&appid={apiKey}&units=metric&lang=th")
            response.EnsureSuccessStatusCode() |> ignore
            let! data = response.Content.ReadFromJsonAsync<OpenWeatherResponse>()
            
            return {|
                city = data.Name
                temperature = data.Main.Temp
                feelsLike = data.Main.FeelsLike
                description = data.Weather |> Array.tryHead |> Option.map (fun w -> w.Description) |> Option.defaultValue ""
                humidity = data.Main.Humidity
                windSpeed = data.Wind.Speed
            |}
        }

// Usage example
let integrationExample () =
    task {
        use httpClient = new HttpClient()
        
        // GitHub
        let githubClient = GitHubApiClient(httpClient)
        let! user = githubClient.GetUser("github")
        printfn "GitHub user: %A" user
        
        let! repos = githubClient.GetRepos("github")
        printfn "Repos: %A" (repos |> List.map (fun r -> r.Name))
        
        // Weather
        let weatherClient = WeatherClient(httpClient, "your-api-key")
        let! weather = weatherClient.GetCurrentWeather("Bangkok")
        printfn "Bangkok weather: %A" weather
    }
```

---

## สรุป

HttpClient ใน F#:

1. **HttpClient lifecycle**: ใช้ shared instance หรือ HttpClientFactory (ห้ามสร้างใหม่ทุกครั้ง)
2. **Async**: ใช้ `task { }` computation expression สำหรับ async operations
3. **Error handling**: handle network errors, timeout, status codes
4. **HttpClientFactory**: แนะนำสำหรับ ASP.NET Core apps
5. **Polly**: Retry, circuit breaker สำหรับ resilience
6. **TypedHttpClient**: inject client ผ่าน DI พร้อม type safety

Best practices:
- ใช้ `IHttpClientFactory` เสมอใน ASP.NET Core
- กำหนด timeout ทุกครั้ง
- Handle errors อย่างครบถ้วน
- ใช้ Polly สำหรับ retry/circuit breaker
- Log requests/responses ในรูปแบบ structured logging
- ใช้ `GetFromJsonAsync`/`PostAsJsonAsync` สำหรับ JSON (ง่ายกว่า manual serialization)
