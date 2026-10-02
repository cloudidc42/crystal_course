# Part 66 - Repository Pattern กับ F#

## บทนำ (Introduction)

Repository Pattern เป็น design pattern ที่ช่วยแยกการเข้าถึงข้อมูล (data access logic) ออกจาก business logic ทำให้โค้ดอ่านง่าย, ทดสอบง่าย, และเปลี่ยน data source ได้โดยไม่ส่งผลกระทบต่อ business logic

ใน F# เราสามารถ implement Repository Pattern ได้หลายวิธี:
- ใช้ interface (แบบ OOP)
- ใช้ function records (แบบ functional)
- ใช้ computation expressions

---

## 1. Domain Models

```fsharp
// Domain.fs
module Domain

open System

// ========================================
// 1.1 Core domain types
// ========================================

type UserId = UserId of int
type ProductId = ProductId of int
type OrderId = OrderId of int

type Email = Email of string

type User = {
    Id: UserId
    Name: string
    Email: Email
    CreatedAt: DateTime
    IsActive: bool
}

type Category = {
    Id: int
    Name: string
    Description: string option
}

type Product = {
    Id: ProductId
    Name: string
    Price: decimal
    Stock: int
    CategoryId: int
    Description: string option
    IsActive: bool
    CreatedAt: DateTime
}

type OrderStatus =
    | Pending
    | Processing
    | Shipped
    | Delivered
    | Cancelled

type OrderItem = {
    ProductId: ProductId
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

type Order = {
    Id: OrderId
    UserId: UserId
    Items: OrderItem list
    TotalAmount: decimal
    Status: OrderStatus
    CreatedAt: DateTime
    UpdatedAt: DateTime option
}

// ========================================
// 1.2 DTOs
// ========================================

type CreateUserRequest = {
    Name: string
    Email: string
}

type UpdateUserRequest = {
    Id: UserId
    Name: string
    IsActive: bool
}

type CreateProductRequest = {
    Name: string
    Price: decimal
    Stock: int
    CategoryId: int
    Description: string option
}

type CreateOrderRequest = {
    UserId: UserId
    Items: (ProductId * int) list  // (productId, quantity)
}

// ========================================
// 1.3 Query parameters
// ========================================

type PagedQuery = {
    Page: int
    PageSize: int
}

type ProductFilter = {
    CategoryId: int option
    MinPrice: decimal option
    MaxPrice: decimal option
    SearchTerm: string option
    InStockOnly: bool
}

type PagedResult<'T> = {
    Items: 'T list
    TotalCount: int
    Page: int
    PageSize: int
    TotalPages: int
}

// ========================================
// 1.4 Error types
// ========================================

type RepositoryError =
    | NotFound of string
    | DuplicateKey of string
    | ValidationError of string
    | DatabaseError of string
    | ConcurrencyError of string

type Result<'T> = Result<'T, RepositoryError>
```

---

## 2. Generic Repository Interface (แบบ OOP)

```fsharp
// Interfaces.fs
module Interfaces

open Domain
open System.Threading.Tasks

// ========================================
// 2.1 Generic repository interface
// ========================================

type IRepository<'T, 'TId> =
    abstract member GetByIdAsync: 'TId -> Task<'T option>
    abstract member GetAllAsync: unit -> Task<'T list>
    abstract member AddAsync: 'T -> Task<Result<'T>>
    abstract member UpdateAsync: 'T -> Task<Result<'T>>
    abstract member DeleteAsync: 'TId -> Task<Result<unit>>
    abstract member ExistsAsync: 'TId -> Task<bool>

// ========================================
// 2.2 Specific repository interfaces
// ========================================

type IUserRepository =
    inherit IRepository<User, UserId>
    abstract member GetByEmailAsync: Email -> Task<User option>
    abstract member GetActiveUsersAsync: unit -> Task<User list>
    abstract member SearchAsync: string -> PagedQuery -> Task<PagedResult<User>>

type IProductRepository =
    inherit IRepository<Product, ProductId>
    abstract member GetByCategoryAsync: int -> Task<Product list>
    abstract member SearchAsync: ProductFilter -> PagedQuery -> Task<PagedResult<Product>>
    abstract member UpdateStockAsync: ProductId -> int -> Task<Result<unit>>
    abstract member GetLowStockAsync: int -> Task<Product list>

type IOrderRepository =
    inherit IRepository<Order, OrderId>
    abstract member GetByUserAsync: UserId -> Task<Order list>
    abstract member GetByStatusAsync: OrderStatus -> Task<Order list>
    abstract member UpdateStatusAsync: OrderId -> OrderStatus -> Task<Result<unit>>
    abstract member GetRecentOrdersAsync: int -> Task<Order list>

// ========================================
// 2.3 Unit of Work interface
// ========================================

type IUnitOfWork =
    abstract member Users: IUserRepository
    abstract member Products: IProductRepository
    abstract member Orders: IOrderRepository
    abstract member CommitAsync: unit -> Task<int>
    abstract member RollbackAsync: unit -> Task<unit>
```

---

## 3. In-Memory Repository (สำหรับ Testing)

```fsharp
// InMemoryRepository.fs
module InMemoryRepository

open System
open System.Collections.Concurrent
open Domain
open Interfaces

// ========================================
// 3.1 Generic in-memory repository
// ========================================

type InMemoryUserRepository() =
    let store = ConcurrentDictionary<UserId, User>()
    let mutable nextId = 1

    let newId () =
        let id = nextId
        nextId <- nextId + 1
        UserId id

    interface IUserRepository with
        member _.GetByIdAsync(id) = task {
            return store.TryGetValue(id) |> function
                | true, user -> Some user
                | _ -> None
        }

        member _.GetAllAsync() = task {
            return store.Values |> Seq.toList
        }

        member _.AddAsync(user) = task {
            let id = newId()
            let newUser = { user with Id = id }
            if store.TryAdd(id, newUser) then
                return Ok newUser
            else
                return Error (DuplicateKey "User already exists")
        }

        member _.UpdateAsync(user) = task {
            if store.ContainsKey(user.Id) then
                store.[user.Id] <- user
                return Ok user
            else
                return Error (NotFound $"User {user.Id} not found")
        }

        member _.DeleteAsync(id) = task {
            if store.TryRemove(id) |> fst then
                return Ok ()
            else
                return Error (NotFound $"User {id} not found")
        }

        member _.ExistsAsync(id) = task {
            return store.ContainsKey(id)
        }

        member _.GetByEmailAsync(email) = task {
            return store.Values |> Seq.tryFind (fun u -> u.Email = email)
        }

        member _.GetActiveUsersAsync() = task {
            return store.Values |> Seq.filter (fun u -> u.IsActive) |> Seq.toList
        }

        member _.SearchAsync(searchTerm) (query) = task {
            let term = searchTerm.ToLower()
            let filtered =
                store.Values
                |> Seq.filter (fun u ->
                    let (Email e) = u.Email
                    u.Name.ToLower().Contains(term) || e.ToLower().Contains(term)
                )
                |> Seq.toList
            let total = filtered.Length
            let items =
                filtered
                |> List.skip ((query.Page - 1) * query.PageSize)
                |> List.truncate query.PageSize
            return {
                Items = items
                TotalCount = total
                Page = query.Page
                PageSize = query.PageSize
                TotalPages = (total + query.PageSize - 1) / query.PageSize
            }
        }

// ========================================
// 3.2 In-memory Product repository
// ========================================

type InMemoryProductRepository() =
    let store = ConcurrentDictionary<ProductId, Product>()
    let mutable nextId = 1

    interface IProductRepository with
        member _.GetByIdAsync(id) = task {
            return store.TryGetValue(id) |> function
                | true, p -> Some p
                | _ -> None
        }

        member _.GetAllAsync() = task {
            return store.Values |> Seq.toList
        }

        member _.AddAsync(product) = task {
            let id = ProductId nextId
            nextId <- nextId + 1
            let newProduct = { product with Id = id }
            store.[id] <- newProduct
            return Ok newProduct
        }

        member _.UpdateAsync(product) = task {
            if store.ContainsKey(product.Id) then
                store.[product.Id] <- product
                return Ok product
            else
                return Error (NotFound $"Product {product.Id} not found")
        }

        member _.DeleteAsync(id) = task {
            if store.TryRemove(id) |> fst then return Ok ()
            else return Error (NotFound $"Product {id} not found")
        }

        member _.ExistsAsync(id) = task {
            return store.ContainsKey(id)
        }

        member _.GetByCategoryAsync(categoryId) = task {
            return store.Values |> Seq.filter (fun p -> p.CategoryId = categoryId) |> Seq.toList
        }

        member _.SearchAsync(filter) (query) = task {
            let filtered =
                store.Values
                |> Seq.filter (fun p ->
                    let catMatch =
                        filter.CategoryId |> Option.map (fun c -> p.CategoryId = c) |> Option.defaultValue true
                    let minPriceMatch =
                        filter.MinPrice |> Option.map (fun m -> p.Price >= m) |> Option.defaultValue true
                    let maxPriceMatch =
                        filter.MaxPrice |> Option.map (fun m -> p.Price <= m) |> Option.defaultValue true
                    let searchMatch =
                        filter.SearchTerm
                        |> Option.map (fun s -> p.Name.ToLower().Contains(s.ToLower()))
                        |> Option.defaultValue true
                    let stockMatch = not filter.InStockOnly || p.Stock > 0
                    catMatch && minPriceMatch && maxPriceMatch && searchMatch && stockMatch && p.IsActive
                )
                |> Seq.toList
            let total = filtered.Length
            return {
                Items = filtered |> List.skip ((query.Page - 1) * query.PageSize) |> List.truncate query.PageSize
                TotalCount = total
                Page = query.Page
                PageSize = query.PageSize
                TotalPages = (total + query.PageSize - 1) / query.PageSize
            }
        }

        member _.UpdateStockAsync(id) (newStock) = task {
            match store.TryGetValue(id) with
            | true, product ->
                store.[id] <- { product with Stock = newStock }
                return Ok ()
            | _ ->
                return Error (NotFound $"Product {id} not found")
        }

        member _.GetLowStockAsync(threshold) = task {
            return store.Values |> Seq.filter (fun p -> p.Stock <= threshold && p.IsActive) |> Seq.toList
        }
```

---

## 4. Functional Approach (แบบ Functional)

```fsharp
// FunctionalRepository.fs
module FunctionalRepository

open Domain
open System.Threading.Tasks

// ========================================
// 4.1 Repository เป็น function records
// ========================================

type UserRepository = {
    GetById: UserId -> Task<User option>
    GetAll: unit -> Task<User list>
    GetByEmail: Email -> Task<User option>
    Create: CreateUserRequest -> Task<Result<User, RepositoryError>>
    Update: UpdateUserRequest -> Task<Result<User, RepositoryError>>
    Delete: UserId -> Task<Result<unit, RepositoryError>>
    Search: string -> PagedQuery -> Task<PagedResult<User>>
}

type ProductRepository = {
    GetById: ProductId -> Task<Product option>
    GetAll: unit -> Task<Product list>
    Create: CreateProductRequest -> Task<Result<Product, RepositoryError>>
    Update: Product -> Task<Result<Product, RepositoryError>>
    Delete: ProductId -> Task<Result<unit, RepositoryError>>
    Search: ProductFilter -> PagedQuery -> Task<PagedResult<Product>>
    UpdateStock: ProductId -> int -> Task<Result<unit, RepositoryError>>
}

// ========================================
// 4.2 In-memory implementations (functional)
// ========================================

let createInMemoryUserRepository () : UserRepository =
    let mutable users: Map<UserId, User> = Map.empty
    let mutable nextId = 1

    {
        GetById = fun id -> task {
            return Map.tryFind id users
        }

        GetAll = fun () -> task {
            return users |> Map.values |> Seq.toList
        }

        GetByEmail = fun email -> task {
            return users |> Map.values |> Seq.tryFind (fun u -> u.Email = email)
        }

        Create = fun req -> task {
            let email = Email req.Email
            match users |> Map.values |> Seq.tryFind (fun u -> u.Email = email) with
            | Some _ ->
                return Error (DuplicateKey $"Email {req.Email} already exists")
            | None ->
                let id = UserId nextId
                nextId <- nextId + 1
                let user = {
                    Id = id
                    Name = req.Name
                    Email = email
                    CreatedAt = System.DateTime.UtcNow
                    IsActive = true
                }
                users <- Map.add id user users
                return Ok user
        }

        Update = fun req -> task {
            match Map.tryFind req.Id users with
            | None ->
                return Error (NotFound $"User {req.Id} not found")
            | Some existing ->
                let updated = { existing with Name = req.Name; IsActive = req.IsActive }
                users <- Map.add req.Id updated users
                return Ok updated
        }

        Delete = fun id -> task {
            match Map.tryFind id users with
            | None ->
                return Error (NotFound $"User {id} not found")
            | Some _ ->
                users <- Map.remove id users
                return Ok ()
        }

        Search = fun term query -> task {
            let termLower = term.ToLower()
            let filtered =
                users
                |> Map.values
                |> Seq.filter (fun u ->
                    let (Email e) = u.Email
                    u.Name.ToLower().Contains(termLower) || e.ToLower().Contains(termLower)
                )
                |> Seq.toList
            let total = filtered.Length
            return {
                Items = filtered |> List.skip ((query.Page - 1) * query.PageSize) |> List.truncate query.PageSize
                TotalCount = total
                Page = query.Page
                PageSize = query.PageSize
                TotalPages = (total + query.PageSize - 1) / query.PageSize
            }
        }
    }

// ========================================
// 4.3 ตัวอย่าง Dapper implementation
// ========================================

open System.Data
open Dapper

let createDapperUserRepository (conn: IDbConnection) : UserRepository =
    {
        GetById = fun (UserId id) -> task {
            let! result = conn.QueryFirstOrDefaultAsync<User>(
                "SELECT id as Id, name as Name, email as Email, created_at as CreatedAt, is_active as IsActive FROM users WHERE id = @Id",
                {| Id = id |}
            )
            return result |> Option.ofObj
        }

        GetAll = fun () -> task {
            let! result = conn.QueryAsync<User>(
                "SELECT id as Id, name as Name, email as Email, created_at as CreatedAt, is_active as IsActive FROM users ORDER BY name"
            )
            return result |> Seq.toList
        }

        GetByEmail = fun (Email email) -> task {
            let! result = conn.QueryFirstOrDefaultAsync<User>(
                "SELECT id as Id, name as Name, email as Email, created_at as CreatedAt, is_active as IsActive FROM users WHERE email = @Email",
                {| Email = email |}
            )
            return result |> Option.ofObj
        }

        Create = fun req -> task {
            try
                let sql = """
                    INSERT INTO users (name, email, created_at, is_active)
                    VALUES (@Name, @Email, @CreatedAt, @IsActive)
                    RETURNING id
                """
                let! newId = conn.QuerySingleAsync<int>(sql, {|
                    Name = req.Name
                    Email = req.Email
                    CreatedAt = System.DateTime.UtcNow
                    IsActive = true
                |})
                let user = {
                    Id = UserId newId
                    Name = req.Name
                    Email = Email req.Email
                    CreatedAt = System.DateTime.UtcNow
                    IsActive = true
                }
                return Ok user
            with ex ->
                return Error (DatabaseError ex.Message)
        }

        Update = fun req -> task {
            let (UserId id) = req.Id
            let! rowsAffected = conn.ExecuteAsync(
                "UPDATE users SET name = @Name, is_active = @IsActive WHERE id = @Id",
                {| Name = req.Name; IsActive = req.IsActive; Id = id |}
            )
            if rowsAffected > 0 then
                let! updated = conn.QueryFirstOrDefaultAsync<User>(
                    "SELECT id as Id, name as Name, email as Email, created_at as CreatedAt, is_active as IsActive FROM users WHERE id = @Id",
                    {| Id = id |}
                )
                return Ok updated
            else
                return Error (NotFound $"User {id} not found")
        }

        Delete = fun (UserId id) -> task {
            let! rowsAffected = conn.ExecuteAsync(
                "DELETE FROM users WHERE id = @Id",
                {| Id = id |}
            )
            if rowsAffected > 0 then return Ok ()
            else return Error (NotFound $"User {id} not found")
        }

        Search = fun term query -> task {
            let offset = (query.Page - 1) * query.PageSize
            let! items = conn.QueryAsync<User>(
                """SELECT id as Id, name as Name, email as Email, created_at as CreatedAt, is_active as IsActive
                   FROM users WHERE name ILIKE @Term OR email ILIKE @Term
                   ORDER BY name LIMIT @PageSize OFFSET @Offset""",
                {| Term = $"%%{term}%%"; PageSize = query.PageSize; Offset = offset |}
            )
            let! total = conn.QuerySingleAsync<int>(
                "SELECT COUNT(*) FROM users WHERE name ILIKE @Term OR email ILIKE @Term",
                {| Term = $"%%{term}%%" |}
            )
            return {
                Items = items |> Seq.toList
                TotalCount = total
                Page = query.Page
                PageSize = query.PageSize
                TotalPages = (total + query.PageSize - 1) / query.PageSize
            }
        }
    }
```

---

## 5. Specification Pattern (รูปแบบ Specification)

```fsharp
// Specification.fs
module Specification

open Domain

// ========================================
// 5.1 Specification type
// ========================================

type Specification<'T> = {
    IsSatisfiedBy: 'T -> bool
}

// สร้าง specifications
let spec f = { IsSatisfiedBy = f }

/// Combine ด้วย AND
let (&&.) (s1: Specification<'T>) (s2: Specification<'T>) =
    spec (fun x -> s1.IsSatisfiedBy x && s2.IsSatisfiedBy x)

/// Combine ด้วย OR
let (||.) (s1: Specification<'T>) (s2: Specification<'T>) =
    spec (fun x -> s1.IsSatisfiedBy x || s2.IsSatisfiedBy x)

/// NOT
let not_ (s: Specification<'T>) =
    spec (fun x -> not (s.IsSatisfiedBy x))

// ========================================
// 5.2 Product specifications
// ========================================

let activeProduct = spec (fun (p: Product) -> p.IsActive)
let inStock = spec (fun (p: Product) -> p.Stock > 0)
let priceAbove min = spec (fun (p: Product) -> p.Price >= min)
let priceBelow max = spec (fun (p: Product) -> p.Price <= max)
let inCategory catId = spec (fun (p: Product) -> p.CategoryId = catId)
let nameContains term = spec (fun (p: Product) -> p.Name.ToLower().Contains(term.ToLower()))

// Combine specifications
let affordableActiveProducts maxPrice =
    activeProduct &&. inStock &&. (priceBelow maxPrice)

let premiumProducts =
    activeProduct &&. inStock &&. (priceAbove 10000m)

// ========================================
// 5.3 User specifications
// ========================================

let activeUser = spec (fun (u: User) -> u.IsActive)
let nameMatches pattern = spec (fun (u: User) -> u.Name.ToLower().Contains(pattern.ToLower()))

// ========================================
// 5.4 Repository with specifications
// ========================================

type ISpecificationRepository<'T> =
    abstract member FindAsync: Specification<'T> -> System.Threading.Tasks.Task<'T list>
    abstract member CountAsync: Specification<'T> -> System.Threading.Tasks.Task<int>

type InMemorySpecificationProductRepo(products: Product list) =
    interface ISpecificationRepository<Product> with
        member _.FindAsync(spec) = task {
            return products |> List.filter spec.IsSatisfiedBy
        }
        member _.CountAsync(spec) = task {
            return products |> List.filter spec.IsSatisfiedBy |> List.length
        }

// ========================================
// 5.5 Usage example
// ========================================

let findProductsExample (repo: ISpecificationRepository<Product>) =
    task {
        // ค้นหาสินค้าที่ active และมี stock และราคาระหว่าง 1000-5000
        let spec = activeProduct &&. inStock &&. (priceAbove 1000m) &&. (priceBelow 5000m)
        let! products = repo.FindAsync(spec)
        return products

        // ค้นหาสินค้า category 1 ที่ชื่อมี "laptop"
        // let laptopSpec = (inCategory 1) &&. (nameContains "laptop") &&. activeProduct
        // let! laptops = repo.FindAsync(laptopSpec)
    }
```

---

## 6. Unit of Work Pattern

```fsharp
// UnitOfWork.fs
module UnitOfWork

open Domain
open Interfaces
open System.Data
open Dapper

// ========================================
// 6.1 Concrete Unit of Work
// ========================================

type DapperUnitOfWork(connection: IDbConnection) =
    let mutable transaction: IDbTransaction option = None

    let conn =
        if connection.State <> ConnectionState.Open then
            connection.Open()
        connection

    member private _.EnsureTransaction() =
        match transaction with
        | Some t -> t
        | None ->
            let t = conn.BeginTransaction()
            transaction <- Some t
            t

    interface IUnitOfWork with
        member this.Users =
            FunctionalRepository.createDapperUserRepository conn
            :> IUserRepository

        member this.Products =
            // สร้าง product repository...
            Unchecked.defaultof<IProductRepository>  // placeholder

        member this.Orders =
            // สร้าง order repository...
            Unchecked.defaultof<IOrderRepository>  // placeholder

        member _.CommitAsync() = task {
            match transaction with
            | Some t ->
                t.Commit()
                t.Dispose()
                transaction <- None
                return 1  // rows affected
            | None ->
                return 0
        }

        member _.RollbackAsync() = task {
            match transaction with
            | Some t ->
                t.Rollback()
                t.Dispose()
                transaction <- None
            | None ->
                ()
        }

    interface System.IDisposable with
        member _.Dispose() =
            match transaction with
            | Some t -> t.Dispose()
            | None -> ()
            conn.Dispose()

// ========================================
// 6.2 In-memory Unit of Work (for testing)
// ========================================

type InMemoryUnitOfWork() =
    let userRepo = InMemoryRepository.InMemoryUserRepository()
    let productRepo = InMemoryRepository.InMemoryProductRepository()
    let mutable committed = false

    interface IUnitOfWork with
        member _.Users = userRepo :> IUserRepository
        member _.Products = productRepo :> IProductRepository
        member _.Orders = Unchecked.defaultof<IOrderRepository>

        member _.CommitAsync() = task {
            committed <- true
            return 1
        }

        member _.RollbackAsync() = task {
            committed <- false
        }

    member _.WasCommitted = committed
```

---

## 7. Async Repositories with Result

```fsharp
// AsyncRepository.fs
module AsyncRepository

open Domain
open System.Threading.Tasks

// ========================================
// 7.1 Repository ที่ return Result type
// ========================================

type AsyncResult<'T> = Task<Result<'T, RepositoryError>>

// Helper functions
let succeed x : AsyncResult<'T> = task { return Ok x }
let fail err : AsyncResult<'T> = task { return Error err }

let bindAsync (f: 'T -> AsyncResult<'U>) (result: AsyncResult<'T>) : AsyncResult<'U> =
    task {
        let! r = result
        match r with
        | Ok value -> return! f value
        | Error err -> return Error err
    }

let mapAsync (f: 'T -> 'U) (result: AsyncResult<'T>) : AsyncResult<'U> =
    task {
        let! r = result
        return Result.map f r
    }

// ========================================
// 7.2 Repository operations
// ========================================

type SafeUserRepository = {
    FindById: UserId -> AsyncResult<User>
    FindAll: unit -> AsyncResult<User list>
    Create: CreateUserRequest -> AsyncResult<User>
    Update: UpdateUserRequest -> AsyncResult<User>
    Delete: UserId -> AsyncResult<unit>
}

let createSafeUserRepository (baseRepo: FunctionalRepository.UserRepository) : SafeUserRepository =
    {
        FindById = fun id -> task {
            let! result = baseRepo.GetById id
            return
                match result with
                | Some user -> Ok user
                | None -> Error (NotFound $"User {id} not found")
        }

        FindAll = fun () -> task {
            let! users = baseRepo.GetAll()
            return Ok users
        }

        Create = fun req -> task {
            // Validate
            if String.length req.Name < 2 then
                return Error (ValidationError "Name must be at least 2 characters")
            elif not (req.Email.Contains("@")) then
                return Error (ValidationError "Invalid email format")
            else
                return! baseRepo.Create req
        }

        Update = fun req -> task {
            return! baseRepo.Update req
        }

        Delete = fun id -> task {
            return! baseRepo.Delete id
        }
    }

// ========================================
// 7.3 Chaining operations
// ========================================

let getUserAndValidate (repo: SafeUserRepository) (userId: UserId) =
    task {
        let! userResult = repo.FindById userId
        return
            userResult
            |> Result.bind (fun user ->
                if user.IsActive then Ok user
                else Error (ValidationError "User is not active")
            )
    }

let createAndNotify (repo: SafeUserRepository) (req: CreateUserRequest) (notify: User -> unit) =
    task {
        let! result = repo.Create req
        match result with
        | Ok user ->
            notify user
            return Ok user
        | Error err ->
            return Error err
    }
```

---

## 8. Repository Testing (การทดสอบ)

```fsharp
// RepositoryTests.fs
module RepositoryTests

open Domain
open FunctionalRepository

// ========================================
// 8.1 Testing in-memory repository
// ========================================

let testCreateUser () =
    task {
        let repo = createInMemoryUserRepository()

        let req = {
            Name = "สมชาย ใจดี"
            Email = "somchai@test.com"
        }

        let! result = repo.Create req
        match result with
        | Ok user ->
            printfn "✓ Created user: %s (ID: %A)" user.Name user.Id
            assert (user.Name = req.Name)
            assert (user.IsActive = true)
        | Error err ->
            printfn "✗ Failed: %A" err
    }

let testDuplicateEmail () =
    task {
        let repo = createInMemoryUserRepository()

        let req = { Name = "Alice"; Email = "alice@test.com" }
        let! _ = repo.Create req
        let! result = repo.Create req

        match result with
        | Error (DuplicateKey _) ->
            printfn "✓ Duplicate email correctly rejected"
        | Ok _ ->
            printfn "✗ Should have failed with DuplicateKey"
        | Error err ->
            printfn "✗ Wrong error: %A" err
    }

let testSearchUsers () =
    task {
        let repo = createInMemoryUserRepository()

        // สร้างข้อมูลทดสอบ
        for i in 1..10 do
            let! _ = repo.Create { Name = $"User {i}"; Email = $"user{i}@test.com" }
            ()

        let query = { Page = 1; PageSize = 3 }
        let! result = repo.Search "user" query
        printfn "✓ Search: Found %d items, Total: %d, Pages: %d"
            result.Items.Length result.TotalCount result.TotalPages
        assert (result.Items.Length = 3)
        assert (result.TotalCount = 10)
    }

let testUpdateUser () =
    task {
        let repo = createInMemoryUserRepository()

        let! createResult = repo.Create { Name = "Bob"; Email = "bob@test.com" }
        match createResult with
        | Ok user ->
            let updateReq = { Id = user.Id; Name = "Robert"; IsActive = true }
            let! updateResult = repo.Update updateReq
            match updateResult with
            | Ok updated ->
                printfn "✓ Updated user: %s -> %s" user.Name updated.Name
                assert (updated.Name = "Robert")
            | Error err ->
                printfn "✗ Update failed: %A" err
        | Error err ->
            printfn "✗ Create failed: %A" err
    }

let testDeleteUser () =
    task {
        let repo = createInMemoryUserRepository()

        let! createResult = repo.Create { Name = "Charlie"; Email = "charlie@test.com" }
        match createResult with
        | Ok user ->
            let! deleteResult = repo.Delete user.Id
            match deleteResult with
            | Ok () ->
                let! found = repo.GetById user.Id
                printfn "✓ Deleted user, exists: %b" found.IsSome
                assert (found.IsNone)
            | Error err ->
                printfn "✗ Delete failed: %A" err
        | Error err ->
            printfn "✗ Create failed: %A" err
    }

let runAllTests () =
    task {
        printfn "Running Repository Tests..."
        do! testCreateUser()
        do! testDuplicateEmail()
        do! testSearchUsers()
        do! testUpdateUser()
        do! testDeleteUser()
        printfn "All tests complete!"
    }
```

---

## 9. Complete Example Program

```fsharp
// Program.fs
module Program

open System
open Domain
open FunctionalRepository
open Specification

[<EntryPoint>]
let main _ =
    task {
        printfn "=== Repository Pattern F# Demo ==="
        printfn "==================================="

        // ========================================
        // Create repositories
        // ========================================
        let userRepo = createInMemoryUserRepository()
        let productRepo = InMemoryRepository.InMemoryProductRepository()

        // ========================================
        // User CRUD
        // ========================================
        printfn "\n--- User Repository ---"

        // Create users
        let! u1 = userRepo.Create { Name = "Alice Johnson"; Email = "alice@example.com" }
        let! u2 = userRepo.Create { Name = "Bob Smith"; Email = "bob@example.com" }
        let! u3 = userRepo.Create { Name = "Charlie Brown"; Email = "charlie@example.com" }

        let users = [u1; u2; u3]
        printfn "Created %d users" (users |> List.choose (function Ok _ -> Some () | _ -> None) |> List.length)

        // Get all
        let! allUsers = userRepo.GetAll()
        printfn "All users:"
        for u in allUsers do
            let (Email email) = u.Email
            printfn "  [%A] %s <%s>" u.Id u.Name email

        // Search
        let! searchResult = userRepo.Search "alice" { Page = 1; PageSize = 10 }
        printfn "\nSearch 'alice': %d results" searchResult.TotalCount

        // Update
        match u1 with
        | Ok user ->
            let! updateResult = userRepo.Update { Id = user.Id; Name = "Alice Williams"; IsActive = true }
            match updateResult with
            | Ok updated -> printfn "Updated: %s -> %s" user.Name updated.Name
            | Error err -> printfn "Update error: %A" err
        | _ -> ()

        // ========================================
        // Product Repository with Specification
        // ========================================
        printfn "\n--- Product Repository with Specification ---"

        // Create products (using IProductRepository interface)
        let iProductRepo = productRepo :> Interfaces.IProductRepository
        let products = [
            { Name = "Laptop Pro"; Price = 45999m; Stock = 50; CategoryId = 1; Description = Some "Gaming laptop"; IsActive = true; CreatedAt = DateTime.UtcNow; Id = ProductId 0 }
            { Name = "Wireless Mouse"; Price = 799m; Stock = 200; CategoryId = 2; Description = None; IsActive = true; CreatedAt = DateTime.UtcNow; Id = ProductId 0 }
            { Name = "Keyboard Mech"; Price = 2499m; Stock = 0; CategoryId = 2; Description = Some "RGB Mech"; IsActive = true; CreatedAt = DateTime.UtcNow; Id = ProductId 0 }
            { Name = "Monitor 4K"; Price = 12999m; Stock = 30; CategoryId = 1; Description = None; IsActive = true; CreatedAt = DateTime.UtcNow; Id = ProductId 0 }
            { Name = "Budget Phone"; Price = 4999m; Stock = 100; CategoryId = 3; Description = None; IsActive = false; CreatedAt = DateTime.UtcNow; Id = ProductId 0 }
        ]

        for p in products do
            let! _ = iProductRepo.AddAsync(p)
            ()

        // Use specifications
        let! allProds = iProductRepo.GetAllAsync()
        printfn "Total products: %d" allProds.Length

        let activeAndInStock =
            allProds
            |> List.filter (fun p -> activeProduct.IsSatisfiedBy p && inStock.IsSatisfiedBy p)
        printfn "Active products in stock: %d" activeAndInStock.Length

        let expensive =
            allProds
            |> List.filter (premiumProducts.IsSatisfiedBy)
        printfn "Premium products (>10000): %d" expensive.Length
        for p in expensive do
            printfn "  %s - ฿%.2f" p.Name p.Price

        let affordable =
            allProds
            |> List.filter ((affordableActiveProducts 5000m).IsSatisfiedBy)
        printfn "Affordable active products (<5000): %d" affordable.Length
        for p in affordable do
            printfn "  %s - ฿%.2f (stock: %d)" p.Name p.Price p.Stock

        // ========================================
        // Run tests
        // ========================================
        printfn "\n--- Running Tests ---"
        do! RepositoryTests.runAllTests()

        // ========================================
        // Pattern summary
        // ========================================
        printfn "\n--- Repository Pattern Benefits ---"
        let benefits = [
            "Separation of concerns - business logic ไม่รู้ว่าใช้ DB อะไร"
            "Testability - ใช้ in-memory repository สำหรับ unit tests"
            "Flexibility - เปลี่ยน data source ได้โดยไม่แก้ business logic"
            "Consistency - API เดียวกันสำหรับทุก data sources"
            "Type safety - F# types ช่วย prevent errors"
        ]
        for b in benefits do
            printfn "  ✓ %s" b

        return 0
    } |> Async.AwaitTask |> Async.RunSynchronously
```

---

## สรุป (Summary)

Repository Pattern ใน F# มี 2 แนวทางหลัก:

**1. OOP approach (Interface-based)**:
- เหมาะกับโปรเจคที่ใหญ่และต้องการ extensibility
- ง่ายต่อการ inject ผ่าน DI container
- code มากกว่า แต่ explicit กว่า

**2. Functional approach (Function records)**:
- เหมาะกับ F# idioms มากกว่า
- ยืดหยุ่นกว่า สร้างและ compose ได้ง่าย
- Testability ดีเยี่ยม

```fsharp
// Key patterns:
// Generic interface: IRepository<T, TId>
// Specific interface: IUserRepository
// Unit of Work: IUnitOfWork (commit/rollback)
// Specification: spec (fun x -> condition x)
// Function record: { GetById = ...; Create = ...; ... }
// Result type: Result<T, RepositoryError>
// In-memory: สำหรับ unit testing
```
