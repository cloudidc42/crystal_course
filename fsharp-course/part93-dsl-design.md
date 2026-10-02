# Part 93 - Domain Specific Languages (DSL) กับ F#

## บทนำ

DSL (Domain Specific Language) คือภาษาหรือ API ที่ออกแบบมาเพื่อแก้ปัญหาใน domain เฉพาะ F# มีเครื่องมือที่ยอดเยี่ยมสำหรับการสร้าง DSL ทั้ง internal และ external

---

## 1. Internal DSL Design Principles

```fsharp
// Internal DSL คือ DSL ที่ built on top of host language
// ข้อดี: ใช้ type system ของ host language ได้
// ข้อเสีย: ถูก constrain โดย syntax ของ host language

// หลักการออกแบบ Internal DSL:
// 1. Fluent interface
// 2. Builder pattern
// 3. Computation expressions
// 4. Operator overloading
// 5. Type-safe API

// ตัวอย่าง: Bad DSL (ไม่ type-safe)
let badQuery connection table conditions =
    sprintf "SELECT * FROM %s WHERE %s" table conditions

// ตัวอย่าง: Good DSL (type-safe)
type QueryBuilder<'T> = {
    Table: string
    Conditions: string list
    Limit: int option
    OrderBy: (string * bool) option  // column, ascending
}

let from<'T> table = { Table = table; Conditions = []; Limit = None; OrderBy = None }
let where condition qb = { qb with Conditions = condition :: qb.Conditions }
let limit n qb = { qb with Limit = Some n }
let orderBy col asc qb = { qb with OrderBy = Some (col, asc) }

// Chain
let query =
    from<{| Name: string; Age: int |}> "users"
    |> where "age > 18"
    |> orderBy "name" true
    |> limit 10
```

---

## 2. Computation Expression DSL

```fsharp
// Computation Expression เป็นเครื่องมือหลักสำหรับ DSL ใน F#

// Pipeline DSL
type PipelineStep<'a, 'b> = {
    Name: string
    Execute: 'a -> Async<Result<'b, string>>
}

type PipelineBuilder() =
    member _.Yield(x) = [x]
    member _.YieldFrom(xs) = xs
    member _.Combine(a, b) = a @ b
    member _.Delay(f) = f()
    member _.Zero() = []
    member _.For(xs, f) = xs |> List.collect f
    
    [<CustomOperation("step")>]
    member _.Step(pipeline, name, execute) =
        let step = { Name = name; Execute = execute }
        pipeline @ [step]
    
    [<CustomOperation("transform")>]
    member _.Transform(pipeline, f) =
        pipeline  // simplified

let pipeline = PipelineBuilder()

// ตัวอย่าง: Data processing pipeline
type Record = { Id: int; Data: string; ProcessedAt: System.DateTime option }

let processRecords records = pipeline {
    step "validate" (fun r ->
        async {
            if r.Id > 0 then return Ok r
            else return Error "Invalid ID"
        })
    
    step "process" (fun r ->
        async {
            return Ok { r with ProcessedAt = Some System.DateTime.Now }
        })
}

// HTTP Request DSL
type HttpRequest = {
    Method: string
    Url: string
    Headers: Map<string, string>
    Body: string option
    Timeout: int
}

type HttpRequestBuilder() =
    member _.Yield(()) = {
        Method = "GET"
        Url = ""
        Headers = Map.empty
        Body = None
        Timeout = 30000
    }
    
    [<CustomOperation("get")>]
    member _.Get(req, url) = { req with Method = "GET"; Url = url }
    
    [<CustomOperation("post")>]
    member _.Post(req, url) = { req with Method = "POST"; Url = url }
    
    [<CustomOperation("put")>]
    member _.Put(req, url) = { req with Method = "PUT"; Url = url }
    
    [<CustomOperation("delete")>]
    member _.Delete(req, url) = { req with Method = "DELETE"; Url = url }
    
    [<CustomOperation("header")>]
    member _.Header(req, key, value) = 
        { req with Headers = Map.add key value req.Headers }
    
    [<CustomOperation("body")>]
    member _.Body(req, content) = { req with Body = Some content }
    
    [<CustomOperation("timeout")>]
    member _.Timeout(req, ms) = { req with Timeout = ms }
    
    [<CustomOperation("auth")>]
    member _.Auth(req, token) = 
        { req with Headers = Map.add "Authorization" $"Bearer {token}" req.Headers }
    
    member _.Run(req) = req

let http = HttpRequestBuilder()

// ใช้งาน
let getUsers =
    http {
        get "https://api.example.com/users"
        header "Accept" "application/json"
        auth "my-jwt-token"
        timeout 5000
    }

let createUser name email =
    http {
        post "https://api.example.com/users"
        header "Content-Type" "application/json"
        auth "my-jwt-token"
        body $"""{"name":"{name}","email":"{email}"}"""
    }
```

---

## 3. Builder Pattern DSL

```fsharp
// Builder Pattern สำหรับ complex object construction

// Email Builder
type Email = {
    From: string
    To: string list
    Cc: string list
    Bcc: string list
    Subject: string
    Body: string
    IsHtml: bool
    Attachments: (string * byte[]) list
}

type EmailBuilder private (email: Email) =
    static member Create() = 
        EmailBuilder({
            From = ""
            To = []
            Cc = []
            Bcc = []
            Subject = ""
            Body = ""
            IsHtml = false
            Attachments = []
        })
    
    member _.From(address) = EmailBuilder({ email with From = address })
    member _.To(address) = EmailBuilder({ email with To = address :: email.To })
    member _.To(addresses) = EmailBuilder({ email with To = addresses @ email.To })
    member _.Cc(address) = EmailBuilder({ email with Cc = address :: email.Cc })
    member _.Bcc(address) = EmailBuilder({ email with Bcc = address :: email.Bcc })
    member _.Subject(subject) = EmailBuilder({ email with Subject = subject })
    member _.TextBody(body) = EmailBuilder({ email with Body = body; IsHtml = false })
    member _.HtmlBody(body) = EmailBuilder({ email with Body = body; IsHtml = true })
    member _.Attach(name, data) = 
        EmailBuilder({ email with Attachments = (name, data) :: email.Attachments })
    member _.Build() = 
        if email.From = "" then failwith "From address required"
        elif email.To = [] then failwith "At least one To address required"
        elif email.Subject = "" then failwith "Subject required"
        else email

// ใช้งาน Builder
let welcomeEmail username =
    EmailBuilder.Create()
        .From("noreply@example.com")
        .To($"{username}@example.com")
        .Subject("Welcome to our platform!")
        .HtmlBody($"""
            <h1>Welcome, {username}!</h1>
            <p>Thank you for registering.</p>
        """)
        .Build()

// Database Connection Builder
type DbConfig = {
    Host: string
    Port: int
    Database: string
    Username: string
    Password: string
    PoolSize: int
    Timeout: int
    SslMode: string
}

type DbConfigBuilder private (config: DbConfig) =
    static member Create() =
        DbConfigBuilder({
            Host = "localhost"
            Port = 5432
            Database = ""
            Username = ""
            Password = ""
            PoolSize = 10
            Timeout = 30
            SslMode = "prefer"
        })
    
    member _.Host(host) = DbConfigBuilder({ config with Host = host })
    member _.Port(port) = DbConfigBuilder({ config with Port = port })
    member _.Database(db) = DbConfigBuilder({ config with Database = db })
    member _.Username(user) = DbConfigBuilder({ config with Username = user })
    member _.Password(pass) = DbConfigBuilder({ config with Password = pass })
    member _.PoolSize(size) = DbConfigBuilder({ config with PoolSize = size })
    member _.Timeout(seconds) = DbConfigBuilder({ config with Timeout = seconds })
    member _.Ssl(mode) = DbConfigBuilder({ config with SslMode = mode })
    
    member _.ConnectionString() =
        $"Host={config.Host};Port={config.Port};Database={config.Database};" +
        $"Username={config.Username};Password={config.Password};" +
        $"Pooling=true;MinPoolSize=1;MaxPoolSize={config.PoolSize};" +
        $"CommandTimeout={config.Timeout};SslMode={config.SslMode}"
    
    member _.Build() = config

let dbConfig =
    DbConfigBuilder.Create()
        .Host("db.example.com")
        .Database("shopdb")
        .Username("admin")
        .Password("secret")
        .PoolSize(20)
        .Ssl("require")
        .Build()
```

---

## 4. Operator Overloading สำหรับ DSL

```fsharp
// Operator overloading ช่วยให้ DSL อ่านง่ายขึ้น

// Vector operations DSL
[<Struct>]
type Vec2 = { X: float; Y: float }

module Vec2 =
    let create x y = { X = x; Y = y }
    let zero = { X = 0.0; Y = 0.0 }
    let length v = sqrt (v.X * v.X + v.Y * v.Y)
    let normalize v = 
        let len = length v
        if len = 0.0 then zero
        else { X = v.X / len; Y = v.Y / len }
    
    // Operators
    static member (+) (a: Vec2, b: Vec2) = { X = a.X + b.X; Y = a.Y + b.Y }
    static member (-) (a: Vec2, b: Vec2) = { X = a.X - b.X; Y = a.Y - b.Y }
    static member (*) (v: Vec2, s: float) = { X = v.X * s; Y = v.Y * s }
    static member (*) (s: float, v: Vec2) = { X = v.X * s; Y = v.Y * s }
    static member (/) (v: Vec2, s: float) = { X = v.X / s; Y = v.Y / s }
    static member (~-) (v: Vec2) = { X = -v.X; Y = -v.Y }
    
    // Dot product
    static member ( *. ) (a: Vec2, b: Vec2) = a.X * b.X + a.Y * b.Y

// ใช้งาน
let velocity = Vec2.create 3.0 4.0
let gravity = Vec2.create 0.0 -9.8
let newVelocity = velocity + gravity * 0.1

// Money DSL
[<Struct>]
type Money = { Amount: decimal; Currency: string }

module Money =
    let create amount currency = { Amount = amount; Currency = currency }
    let usd amount = create amount "USD"
    let thb amount = create amount "THB"
    let eur amount = create amount "EUR"
    
    static member (+) (a: Money, b: Money) =
        if a.Currency <> b.Currency then
            failwith $"Cannot add {a.Currency} and {b.Currency}"
        { a with Amount = a.Amount + b.Amount }
    
    static member (-) (a: Money, b: Money) =
        if a.Currency <> b.Currency then
            failwith $"Cannot subtract {a.Currency} from {b.Currency}"
        { a with Amount = a.Amount - b.Amount }
    
    static member (*) (m: Money, factor: decimal) =
        { m with Amount = m.Amount * factor }
    
    static member (/) (m: Money, divisor: decimal) =
        { m with Amount = m.Amount / divisor }
    
    // Custom operators
    let (@@) (amount: decimal) (currency: string) = create amount currency
    let (|>) (m: Money) (f: Money -> Money) = f m

// ใช้งาน
let price = 100.0M @@ "USD"
let tax = price * 0.07M
let total = price + tax

// Query DSL with operators
type FilterExpr<'a> =
    | Eq of ('a -> obj) * obj
    | Gt of ('a -> obj) * obj
    | Lt of ('a -> obj) * obj
    | And of FilterExpr<'a> * FilterExpr<'a>
    | Or of FilterExpr<'a> * FilterExpr<'a>
    | Not of FilterExpr<'a>

let (===) field value = Eq(field, box value)
let (>>>) field value = Gt(field, box value)
let (<<<) field value = Lt(field, box value)
let (&&&) = And
let (|||) = Or
let (!!!) = Not

// ใช้งาน
type User = { Name: string; Age: int; Active: bool }

let filter =
    (fun (u: User) -> box u.Age) >>> 18
    &&& (fun u -> box u.Active) === true

let matchesFilter (filter: FilterExpr<'a>) (item: 'a) =
    let rec eval = function
        | Eq(f, v) -> f item = v
        | Gt(f, v) -> (f item :?> IComparable).CompareTo(v) > 0
        | Lt(f, v) -> (f item :?> IComparable).CompareTo(v) < 0
        | And(a, b) -> eval a && eval b
        | Or(a, b) -> eval a || eval b
        | Not f -> not (eval f)
    eval filter
```

---

## 5. Type-safe Query Builder DSL

```fsharp
// Type-safe SQL Query Builder
module QueryDSL =
    
    // Type-safe column reference
    type Column<'T, 'V> = {
        TableAlias: string
        Name: string
        Selector: 'T -> 'V
    }
    
    type OrderDirection = Asc | Desc
    
    type WhereClause =
        | Equals of string * obj
        | GreaterThan of string * obj
        | LessThan of string * obj
        | Like of string * string
        | In of string * obj list
        | Between of string * obj * obj
        | IsNull of string
        | IsNotNull of string
        | And of WhereClause * WhereClause
        | Or of WhereClause * WhereClause
        | Not of WhereClause
    
    type JoinType = Inner | Left | Right | Full
    
    type SelectQuery<'T> = {
        Table: string
        Alias: string
        Columns: string list
        WhereClause: WhereClause option
        Joins: (JoinType * string * string * WhereClause) list
        OrderBy: (string * OrderDirection) list
        GroupBy: string list
        Having: WhereClause option
        Limit: int option
        Offset: int option
    }
    
    // SQL generation
    let private whereToSql = 
        let rec build = function
            | Equals(col, v) -> $"{col} = '{v}'"
            | GreaterThan(col, v) -> $"{col} > {v}"
            | LessThan(col, v) -> $"{col} < {v}"
            | Like(col, pattern) -> $"{col} LIKE '{pattern}'"
            | In(col, values) -> 
                let vals = values |> List.map string |> String.concat ", "
                $"{col} IN ({vals})"
            | Between(col, low, high) -> $"{col} BETWEEN {low} AND {high}"
            | IsNull col -> $"{col} IS NULL"
            | IsNotNull col -> $"{col} IS NOT NULL"
            | And(a, b) -> $"({build a}) AND ({build b})"
            | Or(a, b) -> $"({build a}) OR ({build b})"
            | Not w -> $"NOT ({build w})"
        build
    
    let toSql (query: SelectQuery<'T>) =
        let cols = 
            if query.Columns = [] then "*"
            else String.concat ", " query.Columns
        
        let sb = System.Text.StringBuilder()
        sb.Append($"SELECT {cols}") |> ignore
        sb.Append($" FROM {query.Table}") |> ignore
        if query.Alias <> "" then
            sb.Append($" AS {query.Alias}") |> ignore
        
        for (joinType, table, alias, condition) in query.Joins do
            let joinStr = match joinType with
                         | Inner -> "INNER JOIN"
                         | Left -> "LEFT JOIN"
                         | Right -> "RIGHT JOIN"
                         | Full -> "FULL OUTER JOIN"
            sb.Append($" {joinStr} {table} AS {alias} ON {whereToSql condition}") |> ignore
        
        match query.WhereClause with
        | Some w -> sb.Append($" WHERE {whereToSql w}") |> ignore
        | None -> ()
        
        if query.GroupBy <> [] then
            sb.Append($" GROUP BY {String.concat ", " query.GroupBy}") |> ignore
        
        match query.Having with
        | Some h -> sb.Append($" HAVING {whereToSql h}") |> ignore
        | None -> ()
        
        if query.OrderBy <> [] then
            let orderParts = 
                query.OrderBy 
                |> List.map (fun (col, dir) -> 
                    match dir with
                    | Asc -> $"{col} ASC"
                    | Desc -> $"{col} DESC")
            sb.Append($" ORDER BY {String.concat ", " orderParts}") |> ignore
        
        match query.Limit with
        | Some n -> sb.Append($" LIMIT {n}") |> ignore
        | None -> ()
        
        match query.Offset with
        | Some n -> sb.Append($" OFFSET {n}") |> ignore
        | None -> ()
        
        sb.ToString()
    
    // DSL functions
    let from<'T> table = {
        Table = table
        Alias = ""
        Columns = []
        WhereClause = None
        Joins = []
        OrderBy = []
        GroupBy = []
        Having = None
        Limit = None
        Offset = None
    }
    
    let alias a q = { q with Alias = a }
    let select cols q = { q with Columns = cols }
    let where c q = { q with WhereClause = Some c }
    let andWhere c q = 
        match q.WhereClause with
        | None -> { q with WhereClause = Some c }
        | Some existing -> { q with WhereClause = Some (And(existing, c)) }
    let join joinType table alias on q =
        { q with Joins = q.Joins @ [(joinType, table, alias, on)] }
    let orderBy col dir q = { q with OrderBy = q.OrderBy @ [(col, dir)] }
    let groupBy cols q = { q with GroupBy = cols }
    let having c q = { q with Having = Some c }
    let limit n q = { q with Limit = Some n }
    let offset n q = { q with Offset = Some n }
    
    // Condition helpers
    let col name = name
    let eq c v = Equals(c, v)
    let gt c v = GreaterThan(c, v)
    let lt c v = LessThan(c, v)
    let like c p = Like(c, p)
    let iin c vs = In(c, vs |> List.map box)
    let between c low high = Between(c, low, high)
    let isNull c = IsNull c
    let andC = And
    let orC = Or
    let notC = Not
    
    // ใช้งาน
    let query = 
        from<{| Id: int; Name: string; CategoryId: int |}> "products"
        |> alias "p"
        |> select ["p.id"; "p.name"; "p.price"; "c.name AS category"]
        |> join Inner "categories" "c" (eq "p.category_id" "c.id")
        |> where (andC 
            (gt "p.price" 100)
            (eq "c.name" "Electronics"))
        |> orderBy "p.price" Desc
        |> limit 20
        |> offset 0
        |> toSql
    
    // ผลลัพธ์:
    // SELECT p.id, p.name, p.price, c.name AS category 
    // FROM products AS p 
    // INNER JOIN categories AS c ON p.category_id = 'c.id' 
    // WHERE (p.price > 100) AND (c.name = 'Electronics') 
    // ORDER BY p.price DESC 
    // LIMIT 20 OFFSET 0
```

---

## 6. Configuration DSL

```fsharp
// Configuration DSL
module ConfigDSL =
    
    type ConfigValue =
        | StringValue of string
        | IntValue of int
        | FloatValue of float
        | BoolValue of bool
        | ListValue of ConfigValue list
        | ObjectValue of Map<string, ConfigValue>
    
    type ConfigBuilder() =
        let mutable config : Map<string, ConfigValue> = Map.empty
        
        member _.Yield(()) = Map.empty
        member _.Zero() = Map.empty
        member _.Delay(f) = f()
        member _.Combine(a: Map<string, ConfigValue>, b) = 
            Map.fold (fun acc k v -> Map.add k v acc) a b
        
        [<CustomOperation("set")>]
        member _.Set(cfg: Map<string, ConfigValue>, key: string, value: string) =
            Map.add key (StringValue value) cfg
        
        [<CustomOperation("setInt")>]
        member _.SetInt(cfg: Map<string, ConfigValue>, key: string, value: int) =
            Map.add key (IntValue value) cfg
        
        [<CustomOperation("setBool")>]
        member _.SetBool(cfg: Map<string, ConfigValue>, key: string, value: bool) =
            Map.add key (BoolValue value) cfg
        
        [<CustomOperation("section")>]
        member _.Section(cfg: Map<string, ConfigValue>, name: string, subConfig: Map<string, ConfigValue>) =
            Map.add name (ObjectValue subConfig) cfg
    
    let config = ConfigBuilder()
    
    // ใช้งาน
    let appConfig =
        config {
            set "app.name" "MyApp"
            set "app.version" "1.0.0"
            setInt "app.maxConnections" 100
            setBool "app.debug" false
            section "database" (config {
                set "host" "localhost"
                setInt "port" 5432
                set "name" "mydb"
            })
            section "cache" (config {
                set "host" "redis://localhost"
                setInt "ttl" 3600
            })
        }

// Environment-based config DSL
module EnvConfig =
    
    type EnvVar<'T> = {
        Name: string
        Default: 'T option
        Parser: string -> 'T option
        Description: string
    }
    
    let envString name desc defaultVal = {
        Name = name
        Default = defaultVal
        Parser = Some
        Description = desc
    }
    
    let envInt name desc defaultVal = {
        Name = name
        Default = defaultVal
        Parser = fun s -> match System.Int32.TryParse(s) with true, i -> Some i | _ -> None
        Description = desc
    }
    
    let envBool name desc defaultVal = {
        Name = name
        Default = defaultVal
        Parser = fun s ->
            match s.ToLower() with
            | "true" | "1" | "yes" -> Some true
            | "false" | "0" | "no" -> Some false
            | _ -> None
        Description = desc
    }
    
    let required (ev: EnvVar<'T>) =
        match System.Environment.GetEnvironmentVariable(ev.Name) with
        | null -> 
            match ev.Default with
            | Some d -> d
            | None -> failwith $"Required environment variable {ev.Name} is not set"
        | value ->
            match ev.Parser value with
            | Some v -> v
            | None -> failwith $"Invalid value for {ev.Name}: {value}"
    
    let optional (ev: EnvVar<'T>) =
        match System.Environment.GetEnvironmentVariable(ev.Name) with
        | null -> ev.Default
        | value -> ev.Parser value
    
    // ใช้งาน
    type AppConfig = {
        DatabaseUrl: string
        Port: int
        Debug: bool
        ApiKey: string option
    }
    
    let loadConfig () = {
        DatabaseUrl = required (envString "DATABASE_URL" "PostgreSQL connection string" None)
        Port = required (envInt "PORT" "HTTP server port" (Some 8080))
        Debug = required (envBool "DEBUG" "Enable debug mode" (Some false))
        ApiKey = optional (envString "API_KEY" "External API key" None)
    }
```

---

## 7. Test DSL

```fsharp
// Test DSL ที่อ่านง่าย
module TestDSL =
    
    // Specification DSL
    type Spec = {
        Description: string
        Tests: (string * (unit -> bool)) list
    }
    
    type SpecBuilder() =
        member _.Yield(()) = { Description = ""; Tests = [] }
        member _.Zero() = { Description = ""; Tests = [] }
        member _.Delay(f) = f()
        member _.Combine(a: Spec, b: Spec) =
            { a with Tests = a.Tests @ b.Tests }
        
        [<CustomOperation("describe")>]
        member _.Describe(spec: Spec, desc: string) =
            { spec with Description = desc }
        
        [<CustomOperation("it")>]
        member _.It(spec: Spec, desc: string, test: unit -> bool) =
            { spec with Tests = spec.Tests @ [(desc, test)] }
        
        [<CustomOperation("should")>]
        member _.Should(spec: Spec, desc: string, actual: 'a, expected: 'a) =
            { spec with Tests = spec.Tests @ [(desc, fun () -> actual = expected)] }
    
    let spec = SpecBuilder()
    
    // Assertion helpers
    let equals expected actual =
        if actual = expected then ()
        else failwith $"Expected: {expected}\nActual: {actual}"
    
    let shouldBe = equals
    let notEqual expected actual =
        if actual <> expected then ()
        else failwith $"Values should not be equal: {actual}"
    
    let shouldContain item collection =
        if Seq.contains item collection then ()
        else failwith $"Collection does not contain: {item}"
    
    let shouldBeTrue b =
        if b then ()
        else failwith "Expected true but got false"
    
    let shouldBeFalse b =
        if not b then ()
        else failwith "Expected false but got true"
    
    let throws<'exn when 'exn :> exn> (action: unit -> unit) =
        try
            action()
            failwith $"Expected exception of type {typeof<'exn>.Name}"
        with
        | :? 'exn -> ()
        | ex -> failwith $"Expected {typeof<'exn>.Name} but got {ex.GetType().Name}"
    
    // BDD-style DSL
    type ScenarioStep =
        | Given of string * (unit -> unit)
        | When of string * (unit -> unit)
        | Then of string * (unit -> unit)
        | And of string * (unit -> unit)
    
    type Scenario = {
        Title: string
        Steps: ScenarioStep list
    }
    
    type BddBuilder() =
        member _.Yield(()) = { Title = ""; Steps = [] }
        member _.Zero() = { Title = ""; Steps = [] }
        member _.Delay(f) = f()
        member _.Combine(a: Scenario, b: Scenario) =
            { a with Steps = a.Steps @ b.Steps }
        
        [<CustomOperation("scenario")>]
        member _.Scenario(s: Scenario, title) = { s with Title = title }
        
        [<CustomOperation("given")>]
        member _.Given(s: Scenario, desc, action) = 
            { s with Steps = s.Steps @ [Given(desc, action)] }
        
        [<CustomOperation("when'")>]
        member _.When(s: Scenario, desc, action) = 
            { s with Steps = s.Steps @ [When(desc, action)] }
        
        [<CustomOperation("then'")>]
        member _.Then(s: Scenario, desc, assertion) = 
            { s with Steps = s.Steps @ [Then(desc, assertion)] }
        
        [<CustomOperation("andAlso")>]
        member _.AndAlso(s: Scenario, desc, action) = 
            { s with Steps = s.Steps @ [And(desc, action)] }
    
    let bdd = BddBuilder()
    
    // ตัวอย่าง
    type ShoppingCart = {
        mutable Items: (string * int * decimal) list
        mutable Total: decimal
    }
    
    let runScenario (scenario: Scenario) =
        printfn "\nScenario: %s" scenario.Title
        for step in scenario.Steps do
            match step with
            | Given(desc, action) ->
                printf "  Given %s" desc
                action()
                printfn " ✓"
            | When(desc, action) ->
                printf "  When %s" desc
                action()
                printfn " ✓"
            | Then(desc, assertion) ->
                printf "  Then %s" desc
                assertion()
                printfn " ✓"
            | And(desc, action) ->
                printf "  And %s" desc
                action()
                printfn " ✓"
    
    let cartScenario =
        let cart = { Items = []; Total = 0.0M }
        
        bdd {
            scenario "Adding items to shopping cart"
            given "an empty cart" (fun () ->
                shouldBe 0 cart.Items.Length)
            when' "I add 2 apples at $1.50 each" (fun () ->
                cart.Items <- ("Apple", 2, 1.5M) :: cart.Items
                cart.Total <- cart.Items |> List.sumBy (fun (_, qty, price) -> decimal qty * price))
            then' "the cart should have 1 item type" (fun () ->
                shouldBe 1 cart.Items.Length)
            andAlso "the total should be $3.00" (fun () ->
                shouldBe 3.0M cart.Total)
        }
```

---

## 8. HTML DSL

```fsharp
// Type-safe HTML DSL
module HtmlDSL =
    
    type Attribute = Attr of string * string
    
    type HtmlElement =
        | Element of string * Attribute list * HtmlElement list
        | TextNode of string
        | RawHtml of string
    
    // Smart constructors
    let text s = TextNode s
    let raw s = RawHtml s
    let el tag attrs children = Element(tag, attrs, children)
    
    // Attribute helpers
    let attr name value = Attr(name, value)
    let id' v = attr "id" v
    let class' v = attr "class" v
    let href v = attr "href" v
    let src v = attr "src" v
    let alt v = attr "alt" v
    let type' v = attr "type" v
    let name' v = attr "name" v
    let value' v = attr "value" v
    let placeholder v = attr "placeholder" v
    let required' = attr "required" "required"
    let style v = attr "style" v
    let onclick v = attr "onclick" v
    let data key v = attr $"data-{key}" v
    
    // HTML elements
    let html attrs children = el "html" attrs children
    let head attrs children = el "head" attrs children
    let body attrs children = el "body" attrs children
    let div attrs children = el "div" attrs children
    let span attrs children = el "span" attrs children
    let p attrs children = el "p" attrs children
    let h1 attrs children = el "h1" attrs children
    let h2 attrs children = el "h2" attrs children
    let h3 attrs children = el "h3" attrs children
    let a attrs children = el "a" attrs children
    let img attrs = el "img" attrs []
    let form attrs children = el "form" attrs children
    let input attrs = el "input" attrs []
    let button attrs children = el "button" attrs children
    let ul attrs children = el "ul" attrs children
    let ol attrs children = el "ol" attrs children
    let li attrs children = el "li" attrs children
    let table attrs children = el "table" attrs children
    let thead attrs children = el "thead" attrs children
    let tbody attrs children = el "tbody" attrs children
    let tr attrs children = el "tr" attrs children
    let th attrs children = el "th" attrs children
    let td attrs children = el "td" attrs children
    let nav attrs children = el "nav" attrs children
    let header attrs children = el "header" attrs children
    let footer attrs children = el "footer" attrs children
    let main' attrs children = el "main" attrs children
    let section attrs children = el "section" attrs children
    let article attrs children = el "article" attrs children
    let label attrs children = el "label" attrs children
    let select attrs children = el "select" attrs children
    let option' attrs children = el "option" attrs children
    let textarea attrs children = el "textarea" attrs children
    let script attrs children = el "script" attrs children
    let link attrs = el "link" attrs []
    let meta attrs = el "meta" attrs []
    let title' attrs children = el "title" attrs children
    
    // Rendering
    let rec render = function
        | TextNode s -> System.Web.HttpUtility.HtmlEncode(s)
        | RawHtml s -> s
        | Element(tag, attrs, children) ->
            let attrsStr = 
                attrs 
                |> List.map (fun (Attr(k, v)) -> $" {k}=\"{v}\"")
                |> String.concat ""
            
            let voidElements = 
                Set.ofList ["area";"base";"br";"col";"embed";"hr";"img";
                           "input";"link";"meta";"param";"source";"track";"wbr"]
            
            if Set.contains tag voidElements then
                $"<{tag}{attrsStr}>"
            else
                let childrenStr = children |> List.map render |> String.concat ""
                $"<{tag}{attrsStr}>{childrenStr}</{tag}>"
    
    // ตัวอย่าง: สร้าง HTML page
    let productCard (product: {| Id: int; Name: string; Price: decimal; Image: string |}) =
        div [class' "card"; data "id" (string product.Id)] [
            img [src product.Image; alt product.Name; class' "card-image"]
            div [class' "card-body"] [
                h3 [class' "card-title"] [text product.Name]
                p [class' "card-price"] [text $"${product.Price:F2}"]
                button [class' "btn btn-primary"; onclick $"addToCart({product.Id})"] [
                    text "Add to Cart"
                ]
            ]
        ]
    
    let productListPage products =
        html [] [
            head [] [
                title' [] [text "Products"]
                link [rel "stylesheet"; href "/style.css"]
            ]
            body [] [
                header [class' "site-header"] [
                    nav [class' "navbar"] [
                        a [href "/"] [text "Home"]
                        a [href "/products"] [text "Products"]
                        a [href "/cart"] [text "Cart"]
                    ]
                ]
                main' [class' "container"] [
                    h1 [] [text "Our Products"]
                    div [class' "product-grid"] (List.map productCard products)
                ]
                footer [] [text "© 2024 Shop"]
            ]
        ]
        |> render
```

---

## 9. SQL DSL Example

```fsharp
// Type-safe SQL DSL
module SqlDSL =
    
    // SQL expression types
    type SqlExpr =
        | Literal of string
        | Param of string * obj
        | Column of string
        | TableColumn of string * string
        | BinOp of string * SqlExpr * SqlExpr
        | UnaryOp of string * SqlExpr
        | FuncCall of string * SqlExpr list
        | CaseExpr of (SqlExpr * SqlExpr) list * SqlExpr option
    
    type SqlStatement =
        | Select of {|
            Columns: SqlExpr list
            From: string
            Joins: (string * string * SqlExpr) list
            Where: SqlExpr option
            GroupBy: SqlExpr list
            Having: SqlExpr option
            OrderBy: (SqlExpr * string) list
            Limit: int option
            Offset: int option
          |}
        | Insert of {|
            Table: string
            Columns: string list
            Values: SqlExpr list list
          |}
        | Update of {|
            Table: string
            Set: (string * SqlExpr) list
            Where: SqlExpr option
          |}
        | Delete of {|
            Table: string
            Where: SqlExpr option
          |}
    
    // Smart constructors
    let col name = Column name
    let tcol table name = TableColumn(table, name)
    let param name value = Param(name, box value)
    let lit s = Literal s
    
    // Operators
    let (.=) a b = BinOp("=", a, b)
    let (.<>) a b = BinOp("<>", a, b)
    let (.>) a b = BinOp(">", a, b)
    let (.<) a b = BinOp("<", a, b)
    let (.>=) a b = BinOp(">=", a, b)
    let (.<=) a b = BinOp("<=", a, b)
    let (.&&) a b = BinOp("AND", a, b)
    let (.||) a b = BinOp("OR", a, b)
    let like a b = BinOp("LIKE", a, b)
    
    // Functions
    let count e = FuncCall("COUNT", [e])
    let sum e = FuncCall("SUM", [e])
    let avg e = FuncCall("AVG", [e])
    let max e = FuncCall("MAX", [e])
    let min e = FuncCall("MIN", [e])
    let coalesce exprs = FuncCall("COALESCE", exprs)
    let upper e = FuncCall("UPPER", [e])
    let lower e = FuncCall("LOWER", [e])
    
    // Parameterized query builder
    type ParameterizedQuery = {
        Sql: string
        Parameters: (string * obj) list
    }
    
    let rec exprToSql = function
        | Literal s -> s, []
        | Param(name, value) -> $"@{name}", [(name, value)]
        | Column name -> name, []
        | TableColumn(table, name) -> $"{table}.{name}", []
        | BinOp(op, left, right) ->
            let ls, lp = exprToSql left
            let rs, rp = exprToSql right
            $"({ls} {op} {rs})", lp @ rp
        | UnaryOp(op, expr) ->
            let es, ep = exprToSql expr
            $"{op} ({es})", ep
        | FuncCall(name, args) ->
            let argResults = args |> List.map exprToSql
            let argsStr = argResults |> List.map fst |> String.concat ", "
            let argsParams = argResults |> List.collect snd
            $"{name}({argsStr})", argsParams
        | CaseExpr(whens, else') ->
            let whenClauses = 
                whens |> List.map (fun (cond, result) ->
                    let cs, cp = exprToSql cond
                    let rs, rp = exprToSql result
                    $"WHEN {cs} THEN {rs}", cp @ rp)
            let elseClause =
                match else' with
                | None -> "", []
                | Some e ->
                    let es, ep = exprToSql e
                    $" ELSE {es}", ep
            let whenStr = whenClauses |> List.map fst |> String.concat " "
            let whenParams = whenClauses |> List.collect snd
            let elseStr, elseParams = elseClause
            $"CASE {whenStr}{elseStr} END", whenParams @ elseParams
    
    // ใช้งาน
    let findActiveUsersQuery minAge =
        let whereClause = 
            (col "u.age" .>= param "minAge" minAge)
            .&& (col "u.active" .= lit "true")
        
        let sql = 
            $"SELECT u.id, u.name, u.email " +
            $"FROM users u " +
            $"WHERE {fst (exprToSql whereClause)} " +
            $"ORDER BY u.name ASC"
        
        let _, parameters = exprToSql whereClause
        { Sql = sql; Parameters = parameters }
```

---

## 10. Complete DSL Example: Workflow Engine

```fsharp
// Complete DSL: Workflow/State Machine DSL
module WorkflowDSL =
    
    type StateId = string
    type EventId = string
    
    type Transition<'State, 'Event, 'Context> = {
        From: StateId
        Event: EventId
        To: StateId
        Guard: 'Context -> 'Event -> bool
        Action: 'Context -> 'Event -> 'Context
    }
    
    type WorkflowDef<'State, 'Event, 'Context> = {
        InitialState: StateId
        States: Map<StateId, 'State>
        Transitions: Transition<'State, 'Event, 'Context> list
        OnEnter: Map<StateId, 'Context -> unit>
        OnExit: Map<StateId, 'Context -> unit>
    }
    
    type WorkflowBuilder<'State, 'Event, 'Context>() =
        member _.Yield(()) : WorkflowDef<'State, 'Event, 'Context> =
            { InitialState = ""
              States = Map.empty
              Transitions = []
              OnEnter = Map.empty
              OnExit = Map.empty }
        
        member _.Zero() : WorkflowDef<'State, 'Event, 'Context> =
            { InitialState = ""
              States = Map.empty
              Transitions = []
              OnEnter = Map.empty
              OnExit = Map.empty }
        
        member _.Delay(f) = f()
        member _.Combine(a: WorkflowDef<'State, 'Event, 'Context>, b) =
            { InitialState = if a.InitialState = "" then b.InitialState else a.InitialState
              States = Map.fold (fun acc k v -> Map.add k v acc) a.States b.States
              Transitions = a.Transitions @ b.Transitions
              OnEnter = Map.fold (fun acc k v -> Map.add k v acc) a.OnEnter b.OnEnter
              OnExit = Map.fold (fun acc k v -> Map.add k v acc) a.OnExit b.OnExit }
        
        [<CustomOperation("initialState")>]
        member _.InitialState(wf: WorkflowDef<'State, 'Event, 'Context>, state) =
            { wf with InitialState = state }
        
        [<CustomOperation("state")>]
        member _.State(wf: WorkflowDef<'State, 'Event, 'Context>, id, stateObj) =
            { wf with States = Map.add id stateObj wf.States }
        
        [<CustomOperation("transition")>]
        member _.Transition(wf: WorkflowDef<'State, 'Event, 'Context>, from', event, to', ?guard, ?action) =
            let t = {
                From = from'
                Event = event
                To = to'
                Guard = defaultArg guard (fun _ _ -> true)
                Action = defaultArg action (fun ctx _ -> ctx)
            }
            { wf with Transitions = t :: wf.Transitions }
        
        [<CustomOperation("onEnter")>]
        member _.OnEnter(wf: WorkflowDef<'State, 'Event, 'Context>, state, action) =
            { wf with OnEnter = Map.add state action wf.OnEnter }
        
        [<CustomOperation("onExit")>]
        member _.OnExit(wf: WorkflowDef<'State, 'Event, 'Context>, state, action) =
            { wf with OnExit = Map.add state action wf.OnExit }
    
    let workflow<'S, 'E, 'C> = WorkflowBuilder<'S, 'E, 'C>()
    
    // Workflow executor
    type WorkflowInstance<'Context> = {
        CurrentState: StateId
        Context: 'Context
    }
    
    let start (wf: WorkflowDef<_, _, 'Context>) initialContext = {
        CurrentState = wf.InitialState
        Context = initialContext
    }
    
    let trigger (wf: WorkflowDef<_, 'Event, 'Context>) event instance =
        let eventId = event.ToString()
        let transition = 
            wf.Transitions
            |> List.tryFind (fun t -> 
                t.From = instance.CurrentState && 
                t.Event = eventId &&
                t.Guard instance.Context event)
        
        match transition with
        | None -> Error $"No valid transition from {instance.CurrentState} on {eventId}"
        | Some t ->
            // Execute exit action
            match Map.tryFind instance.CurrentState wf.OnExit with
            | Some action -> action instance.Context
            | None -> ()
            
            // Execute transition action
            let newContext = t.Action instance.Context event
            
            // Execute enter action  
            match Map.tryFind t.To wf.OnEnter with
            | Some action -> action newContext
            | None -> ()
            
            Ok { CurrentState = t.To; Context = newContext }
    
    // ตัวอย่าง: Order workflow
    type OrderStatus = Pending | Confirmed | Shipped | Delivered | Cancelled
    
    type OrderEvent =
        | Confirm
        | Ship of trackingNumber: string
        | Deliver
        | Cancel of reason: string
    
    type OrderContext = {
        OrderId: int
        Status: OrderStatus
        TrackingNumber: string option
        CancelReason: string option
    }
    
    let orderWorkflow = 
        workflow<OrderStatus, OrderEvent, OrderContext> {
            initialState "pending"
            state "pending" Pending
            state "confirmed" Confirmed
            state "shipped" Shipped
            state "delivered" Delivered
            state "cancelled" Cancelled
            
            transition "pending" "Confirm" "confirmed"
                (fun ctx _ -> true)
                (fun ctx _ -> { ctx with Status = Confirmed })
            
            transition "confirmed" "Ship" "shipped"
                (fun ctx _ -> true)
                (fun ctx event ->
                    match event with
                    | Ship trackNum -> { ctx with TrackingNumber = Some trackNum }
                    | _ -> ctx)
            
            transition "shipped" "Deliver" "delivered"
                (fun ctx _ -> true)
                (fun ctx _ -> { ctx with Status = Delivered })
            
            transition "pending" "Cancel" "cancelled"
                (fun ctx _ -> true)
                (fun ctx event ->
                    match event with
                    | Cancel reason -> { ctx with CancelReason = Some reason }
                    | _ -> ctx)
            
            transition "confirmed" "Cancel" "cancelled"
                (fun ctx _ -> true)
                (fun ctx event ->
                    match event with
                    | Cancel reason -> { ctx with CancelReason = Some reason }
                    | _ -> ctx)
            
            onEnter "shipped" (fun ctx ->
                printfn "Order %d shipped! Tracking: %A" ctx.OrderId ctx.TrackingNumber)
            
            onEnter "delivered" (fun ctx ->
                printfn "Order %d delivered!" ctx.OrderId)
            
            onEnter "cancelled" (fun ctx ->
                printfn "Order %d cancelled. Reason: %A" ctx.OrderId ctx.CancelReason)
        }
    
    // ใช้งาน
    let processOrder () =
        let ctx = { OrderId = 1; Status = Pending; TrackingNumber = None; CancelReason = None }
        let instance = start orderWorkflow ctx
        
        result {
            let! confirmed = trigger orderWorkflow Confirm instance
            let! shipped = trigger orderWorkflow (Ship "TH123456789") confirmed
            let! delivered = trigger orderWorkflow Deliver shipped
            return delivered
        }
```

---

## สรุป

DSL Design ใน F# ใช้เทคนิคหลายอย่าง:

1. **Computation Expressions**: สร้าง monadic DSL ที่อ่านเหมือน imperative code
2. **Custom Operations**: เพิ่ม keyword ใน computation expression
3. **Builder Pattern**: สร้าง immutable objects อย่าง type-safe
4. **Operator Overloading**: ทำให้ DSL อ่านง่ายขึ้น
5. **Type-safe wrappers**: ป้องกัน invalid states

การออกแบบ DSL ที่ดีควร:
- อ่านเหมือน domain language
- ป้องกัน invalid states ด้วย type system
- มี good error messages
- Composable และ reusable

---

*ต่อไป: Part 94 - การเพิ่มประสิทธิภาพ (Performance Optimization)*
