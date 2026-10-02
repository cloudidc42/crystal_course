# Part 95 - Security กับ F#

## บทนำ

Security เป็นสิ่งที่ต้องพิจารณาตั้งแต่ต้นในการพัฒนา application ไม่ใช่ add-on ในภายหลัง บทนี้จะครอบคลุม OWASP Top 10 สำหรับ .NET และ best practices ด้าน security ใน F#

---

## 1. OWASP Top 10 สำหรับ .NET

```fsharp
// OWASP Top 10 (2021):
// A01: Broken Access Control
// A02: Cryptographic Failures
// A03: Injection (SQL, XSS, etc.)
// A04: Insecure Design
// A05: Security Misconfiguration
// A06: Vulnerable Components
// A07: Authentication Failures
// A08: Software Integrity Failures
// A09: Security Logging Failures
// A10: Server-Side Request Forgery

// ใน F#/ASP.NET Core เราสามารถป้องกันได้ด้วย:
// - Middleware pipeline
// - Attribute-based authorization
// - Input validation
// - Output encoding
// - Proper cryptography
```

---

## 2. Input Validation

```fsharp
// Input validation เป็น first line of defense

module Validation =
    
    type ValidationError =
        | Required of fieldName: string
        | TooShort of fieldName: string * minLength: int
        | TooLong of fieldName: string * maxLength: int
        | InvalidFormat of fieldName: string * message: string
        | OutOfRange of fieldName: string * min: 'obj * max: 'obj
    
    type ValidationResult<'T> = Result<'T, ValidationError list>
    
    // Validate string
    let validateRequired fieldName (value: string) =
        if System.String.IsNullOrWhiteSpace(value) then
            Error [Required fieldName]
        else
            Ok value
    
    let validateMinLength fieldName minLen (value: string) =
        if value.Length < minLen then
            Error [TooShort(fieldName, minLen)]
        else
            Ok value
    
    let validateMaxLength fieldName maxLen (value: string) =
        if value.Length > maxLen then
            Error [TooLong(fieldName, maxLen)]
        else
            Ok value
    
    // Email validation
    let validateEmail (email: string) =
        let pattern = @"^[^@\s]+@[^@\s]+\.[^@\s]+$"
        if System.Text.RegularExpressions.Regex.IsMatch(email, pattern) then
            Ok (email.Trim().ToLowerInvariant())
        else
            Error [InvalidFormat("Email", "Invalid email format")]
    
    // URL validation
    let validateUrl (url: string) =
        match System.Uri.TryCreate(url, System.UriKind.Absolute) with
        | true, uri when uri.Scheme = "https" || uri.Scheme = "http" ->
            Ok url
        | _ ->
            Error [InvalidFormat("URL", "Must be a valid HTTP/HTTPS URL")]
    
    // Phone validation (Thai format)
    let validateThaiPhone (phone: string) =
        let cleaned = System.Text.RegularExpressions.Regex.Replace(phone, @"[\s\-\(\)]", "")
        let pattern = @"^(0[689]\d{8}|\+66[689]\d{8})$"
        if System.Text.RegularExpressions.Regex.IsMatch(cleaned, pattern) then
            Ok cleaned
        else
            Error [InvalidFormat("Phone", "Invalid Thai phone number")]
    
    // Numeric range validation
    let validateRange fieldName min max (value: int) =
        if value < min || value > max then
            Error [OutOfRange(fieldName, min, max)]
        else
            Ok value
    
    // Combine validations
    let (<*>) (vf: Result<'a -> 'b, 'e list>) (vx: Result<'a, 'e list>) =
        match vf, vx with
        | Ok f, Ok x -> Ok (f x)
        | Error e1, Error e2 -> Error (e1 @ e2)
        | Error e, _ | _, Error e -> Error e
    
    let (<!>) f x = Result.map f (Ok x)
    
    // User registration validation
    type UserRegistration = {
        Username: string
        Email: string
        Password: string
        Age: int
    }
    
    let validateRegistration username email password age =
        let usernameResult = 
            validateRequired "Username" username
            |> Result.bind (validateMinLength "Username" 3)
            |> Result.bind (validateMaxLength "Username" 50)
        
        let emailResult = validateEmail email
        
        let passwordResult =
            validateRequired "Password" password
            |> Result.bind (validateMinLength "Password" 8)
            |> Result.bind (fun pwd ->
                let hasUpper = pwd |> Seq.exists System.Char.IsUpper
                let hasLower = pwd |> Seq.exists System.Char.IsLower
                let hasDigit = pwd |> Seq.exists System.Char.IsDigit
                let hasSpecial = pwd |> Seq.exists (fun c -> not (System.Char.IsLetterOrDigit(c)))
                if hasUpper && hasLower && hasDigit && hasSpecial then Ok pwd
                else Error [InvalidFormat("Password", "Must contain upper, lower, digit, and special character")])
        
        let ageResult = validateRange "Age" 18 120 age
        
        // Combine all validations
        match usernameResult, emailResult, passwordResult, ageResult with
        | Ok u, Ok e, Ok p, Ok a ->
            Ok { Username = u; Email = e; Password = p; Age = a }
        | u, e, p, a ->
            let errors = [
                match u with Error errs -> yield! errs | _ -> ()
                match e with Error errs -> yield! errs | _ -> ()
                match p with Error errs -> yield! errs | _ -> ()
                match a with Error errs -> yield! errs | _ -> ()
            ]
            Error errors
```

---

## 3. SQL Injection Prevention

```fsharp
// SQL Injection: หนึ่งในช่องโหว่ที่พบบ่อยที่สุด

module SqlSecurity =
    open System.Data
    open Microsoft.Data.SqlClient
    
    // BAD: String concatenation (vulnerable to SQL injection!)
    let badGetUser (connectionString: string) username =
        use conn = new SqlConnection(connectionString)
        conn.Open()
        use cmd = conn.CreateCommand()
        // *** VULNERABLE! ***
        cmd.CommandText <- $"SELECT * FROM Users WHERE Username = '{username}'"
        use reader = cmd.ExecuteReader()
        ()
    
    // GOOD: Parameterized query
    let goodGetUser (connectionString: string) (username: string) =
        use conn = new SqlConnection(connectionString)
        conn.Open()
        use cmd = conn.CreateCommand()
        cmd.CommandText <- "SELECT Id, Username, Email FROM Users WHERE Username = @Username"
        cmd.Parameters.AddWithValue("@Username", username) |> ignore
        
        use reader = cmd.ExecuteReader()
        if reader.Read() then
            Some {|
                Id = reader.GetInt32("Id")
                Username = reader.GetString("Username")
                Email = reader.GetString("Email")
            |}
        else
            None
    
    // GOOD: Stored Procedure
    let getUserByStoredProc (connectionString: string) userId =
        use conn = new SqlConnection(connectionString)
        conn.Open()
        use cmd = conn.CreateCommand()
        cmd.CommandType <- CommandType.StoredProcedure
        cmd.CommandText <- "sp_GetUserById"
        cmd.Parameters.AddWithValue("@UserId", userId) |> ignore
        
        use reader = cmd.ExecuteReader()
        if reader.Read() then
            Some {| Id = userId; Name = reader.GetString("Name") |}
        else
            None
    
    // GOOD: Dapper (auto-parameterized)
    // open Dapper
    // let getUserDapper (conn: IDbConnection) userId =
    //     conn.QueryFirstOrDefault<User>(
    //         "SELECT * FROM Users WHERE Id = @Id",
    //         {| Id = userId |})
    
    // GOOD: Entity Framework (LINQ to SQL, auto-parameterized)
    // let getUserEF (ctx: DbContext) userId =
    //     ctx.Users.FirstOrDefault(fun u -> u.Id = userId)
    
    // Input sanitization for LIKE queries
    let sanitizeLikeInput (input: string) =
        // Escape special characters in LIKE pattern
        input.Replace("[", "[[]")
             .Replace("%", "[%]")
             .Replace("_", "[_]")
    
    let searchUsers (connectionString: string) searchTerm =
        use conn = new SqlConnection(connectionString)
        conn.Open()
        use cmd = conn.CreateCommand()
        cmd.CommandText <- "SELECT * FROM Users WHERE Username LIKE @Pattern"
        let sanitized = sanitizeLikeInput searchTerm
        cmd.Parameters.AddWithValue("@Pattern", $"%{sanitized}%") |> ignore
        
        use reader = cmd.ExecuteReader()
        [
            while reader.Read() do
                yield reader.GetString("Username")
        ]
```

---

## 4. XSS Prevention

```fsharp
// Cross-Site Scripting (XSS) Prevention

module XssPrevention =
    open System.Web
    
    // ใช้ HTML encoding ก่อน output ไปที่ HTML
    let encodeHtml (value: string) =
        HttpUtility.HtmlEncode(value)
    
    let encodeHtmlAttribute (value: string) =
        HttpUtility.HtmlAttributeEncode(value)
    
    let encodeJavaScript (value: string) =
        HttpUtility.JavaScriptStringEncode(value)
    
    let encodeUrl (value: string) =
        HttpUtility.UrlEncode(value)
    
    // ตัวอย่าง: Razor-style template
    let renderUserProfile (username: string) (bio: string) =
        // Always encode user input before inserting into HTML
        let encodedName = encodeHtml username
        let encodedBio = encodeHtml bio
        $"""
        <div class="profile">
            <h1>{encodedName}</h1>
            <p>{encodedBio}</p>
        </div>
        """
    
    // Content Security Policy header
    let addCspHeader (response: Microsoft.AspNetCore.Http.HttpResponse) =
        let csp = 
            "default-src 'self'; " +
            "script-src 'self' 'nonce-{nonce}'; " +  // nonce-based CSP
            "style-src 'self'; " +
            "img-src 'self' data:; " +
            "font-src 'self'; " +
            "connect-src 'self' https://api.example.com; " +
            "frame-ancestors 'none'; " +
            "base-uri 'self';"
        
        response.Headers.["Content-Security-Policy"] <- csp
    
    // HTML sanitization (allow some safe tags)
    let sanitizeHtml (input: string) =
        // ใช้ library เช่น HtmlSanitizer
        // var sanitizer = new HtmlSanitizer();
        // sanitizer.Sanitize(input)
        
        // Simple approach: strip all HTML tags
        System.Text.RegularExpressions.Regex.Replace(input, "<[^>]+>", "")
    
    // Giraffe middleware สำหรับ security headers
    open Giraffe
    
    let securityHeaders : HttpHandler =
        setHttpHeader "X-Content-Type-Options" "nosniff"
        >=> setHttpHeader "X-Frame-Options" "DENY"
        >=> setHttpHeader "X-XSS-Protection" "1; mode=block"
        >=> setHttpHeader "Referrer-Policy" "strict-origin-when-cross-origin"
        >=> setHttpHeader "Permissions-Policy" "geolocation=(), microphone=(), camera=()"
```

---

## 5. CSRF Protection

```fsharp
// Cross-Site Request Forgery (CSRF) Protection

module CsrfProtection =
    open Microsoft.AspNetCore.Antiforgery
    open Microsoft.AspNetCore.Http
    
    // Startup configuration
    let configureAntiforgery (services: IServiceCollection) =
        services.AddAntiforgery(fun opts ->
            opts.HeaderName <- "X-XSRF-TOKEN"
            opts.Cookie.Name <- "XSRF-TOKEN"
            opts.Cookie.SameSite <- SameSiteMode.Strict
            opts.Cookie.SecurePolicy <- CookieSecurePolicy.Always)
        |> ignore
    
    // Middleware: เพิ่ม CSRF token ใน cookie
    let addCsrfToken (antiforgery: IAntiforgery) : HttpHandler =
        fun next ctx ->
            let tokens = antiforgery.GetAndStoreTokens(ctx)
            ctx.Response.Cookies.Append("XSRF-TOKEN", tokens.RequestToken,
                CookieOptions(
                    HttpOnly = false,  // JavaScript readable
                    Secure = true,
                    SameSite = SameSiteMode.Strict))
            next ctx
    
    // Validate CSRF token
    let validateCsrf (antiforgery: IAntiforgery) : HttpHandler =
        fun next ctx -> task {
            try
                do! antiforgery.ValidateRequestAsync(ctx)
                return! next ctx
            with _ ->
                ctx.Response.StatusCode <- 403
                return! ctx.Response.WriteAsync("CSRF token validation failed")
        }
    
    // SameSite cookies (modern CSRF protection)
    let configureSameSiteCookie (cookieOptions: CookieOptions) =
        cookieOptions.SameSite <- SameSiteMode.Strict
        cookieOptions.Secure <- true
        cookieOptions.HttpOnly <- true
        cookieOptions
```

---

## 6. Authentication Best Practices

```fsharp
// Authentication: ยืนยันตัวตนของผู้ใช้

module Authentication =
    open Microsoft.AspNetCore.Identity
    open Microsoft.Extensions.DependencyInjection
    open BCrypt.Net
    
    // Password Hashing ด้วย BCrypt
    let hashPassword (password: string) =
        BCrypt.HashPassword(password, workFactor = 12)
    
    let verifyPassword (password: string) (hash: string) =
        BCrypt.Verify(password, hash)
    
    // ตัวอย่าง: User service
    type User = {
        Id: int
        Username: string
        Email: string
        PasswordHash: string
        IsEmailVerified: bool
        FailedLoginAttempts: int
        LockedUntil: System.DateTime option
        LastLoginAt: System.DateTime option
        CreatedAt: System.DateTime
    }
    
    type AuthError =
        | InvalidCredentials
        | AccountLocked of until: System.DateTime
        | EmailNotVerified
        | TooManyAttempts
    
    let private maxFailedAttempts = 5
    let private lockoutDuration = System.TimeSpan.FromMinutes(15.0)
    
    let authenticate (findUser: string -> Async<User option>) (updateUser: User -> Async<unit>) username password =
        async {
            let! userOpt = findUser username
            
            match userOpt with
            | None ->
                // Constant-time comparison to prevent timing attacks
                BCrypt.HashPassword("dummy", workFactor = 12) |> ignore
                return Error InvalidCredentials
            
            | Some user ->
                // Check account lockout
                match user.LockedUntil with
                | Some until when until > System.DateTime.UtcNow ->
                    return Error (AccountLocked until)
                | _ ->
                
                // Check email verification
                if not user.IsEmailVerified then
                    return Error EmailNotVerified
                
                // Verify password
                if not (BCrypt.Verify(password, user.PasswordHash)) then
                    let attempts = user.FailedLoginAttempts + 1
                    let lockedUntil =
                        if attempts >= maxFailedAttempts then
                            Some (System.DateTime.UtcNow + lockoutDuration)
                        else None
                    
                    do! updateUser { user with 
                            FailedLoginAttempts = attempts
                            LockedUntil = lockedUntil }
                    
                    if attempts >= maxFailedAttempts then
                        return Error TooManyAttempts
                    else
                        return Error InvalidCredentials
                
                else
                    // Successful login - reset failed attempts
                    do! updateUser { user with 
                            FailedLoginAttempts = 0
                            LockedUntil = None
                            LastLoginAt = Some System.DateTime.UtcNow }
                    return Ok user
        }
    
    // Multi-Factor Authentication (TOTP)
    // <PackageReference Include="OtpNet" Version="1.4.0" />
    // open OtpNet
    
    // let generateTotpSecret () =
    //     let secretKey = KeyGeneration.GenerateRandomKey(20)
    //     Base32Encoding.ToString(secretKey)
    
    // let verifyTotp secret code =
    //     let secretKey = Base32Encoding.ToBytes(secret)
    //     let totp = new Totp(secretKey)
    //     totp.VerifyTotp(code, out _, VerificationWindow.RfcSpecifiedNetworkDelay)
    
    // Session management
    let configureSession (services: IServiceCollection) =
        services.AddSession(fun opts ->
            opts.IdleTimeout <- System.TimeSpan.FromMinutes(30.0)
            opts.Cookie.HttpOnly <- true
            opts.Cookie.IsEssential <- true
            opts.Cookie.SecurePolicy <- Microsoft.AspNetCore.Http.CookieSecurePolicy.Always
            opts.Cookie.SameSite <- Microsoft.AspNetCore.Http.SameSiteMode.Strict)
        |> ignore
```

---

## 7. JWT Security

```fsharp
// JWT (JSON Web Token) Security

module JwtSecurity =
    open System
    open System.IdentityModel.Tokens.Jwt
    open Microsoft.IdentityModel.Tokens
    open System.Security.Claims
    
    type JwtConfig = {
        SecretKey: string
        Issuer: string
        Audience: string
        AccessTokenExpiry: TimeSpan
        RefreshTokenExpiry: TimeSpan
    }
    
    // สร้าง JWT token
    let generateAccessToken (config: JwtConfig) (userId: int) (roles: string list) =
        let key = SymmetricSecurityKey(
            Text.Encoding.UTF8.GetBytes(config.SecretKey))
        let credentials = SigningCredentials(key, SecurityAlgorithms.HmacSha256)
        
        let claims = [
            Claim(JwtRegisteredClaimNames.Sub, string userId)
            Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            Claim(JwtRegisteredClaimNames.Iat, 
                  DateTimeOffset.UtcNow.ToUnixTimeSeconds() |> string)
            for role in roles do
                Claim(ClaimTypes.Role, role)
        ]
        
        let token = JwtSecurityToken(
            issuer = config.Issuer,
            audience = config.Audience,
            claims = claims,
            expires = DateTime.UtcNow + config.AccessTokenExpiry,
            signingCredentials = credentials)
        
        JwtSecurityTokenHandler().WriteToken(token)
    
    // Validate JWT token
    let validateToken (config: JwtConfig) (token: string) =
        let key = SymmetricSecurityKey(
            Text.Encoding.UTF8.GetBytes(config.SecretKey))
        
        let parameters = TokenValidationParameters(
            ValidateIssuer = true,
            ValidIssuer = config.Issuer,
            ValidateAudience = true,
            ValidAudience = config.Audience,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = key,
            ClockSkew = TimeSpan.Zero)  // No clock skew tolerance
        
        try
            let handler = JwtSecurityTokenHandler()
            let principal = handler.ValidateToken(token, parameters, ref null)
            Ok principal
        with ex ->
            Error ex.Message
    
    // Refresh Token (ควรเก็บใน DB)
    type RefreshToken = {
        Token: string
        UserId: int
        ExpiresAt: DateTime
        IsRevoked: bool
        CreatedAt: DateTime
        ReplacedByToken: string option
    }
    
    let generateRefreshToken () =
        let bytes = Array.zeroCreate<byte> 64
        use rng = Security.Cryptography.RandomNumberGenerator.Create()
        rng.GetBytes(bytes)
        Convert.ToBase64String(bytes)
    
    // Secure key generation (ใช้ใน appsettings.json)
    let generateSecureKey () =
        let bytes = Array.zeroCreate<byte> 32
        use rng = Security.Cryptography.RandomNumberGenerator.Create()
        rng.GetBytes(bytes)
        Convert.ToBase64String(bytes)
    
    // Configure JWT in ASP.NET Core
    let configureJwt (config: JwtConfig) (services: IServiceCollection) =
        open Microsoft.AspNetCore.Authentication.JwtBearer
        
        services.AddAuthentication(fun opts ->
            opts.DefaultAuthenticateScheme <- JwtBearerDefaults.AuthenticationScheme
            opts.DefaultChallengeScheme <- JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(fun opts ->
                let key = SymmetricSecurityKey(
                    Text.Encoding.UTF8.GetBytes(config.SecretKey))
                
                opts.TokenValidationParameters <- TokenValidationParameters(
                    ValidateIssuer = true,
                    ValidIssuer = config.Issuer,
                    ValidateAudience = true,
                    ValidAudience = config.Audience,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = key,
                    ClockSkew = TimeSpan.Zero))
        |> ignore
```

---

## 8. Secrets Management

```fsharp
// Secrets Management - ห้ามเก็บ secrets ใน source code!

module SecretsManagement =
    open Microsoft.Extensions.Configuration
    open Microsoft.Extensions.DependencyInjection
    
    // 1. User Secrets (development)
    // dotnet user-secrets set "Database:Password" "mypassword"
    
    let configureUserSecrets (builder: IHostApplicationBuilder) =
        // Automatically added in Development environment
        builder.Configuration.AddUserSecrets(Assembly.GetExecutingAssembly()) |> ignore
    
    // 2. Environment Variables
    let getFromEnv (key: string) =
        match System.Environment.GetEnvironmentVariable(key) with
        | null -> failwith $"Required environment variable '{key}' is not set"
        | value -> value
    
    // 3. Azure Key Vault
    // <PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" />
    
    let configureAzureKeyVault (builder: IHostApplicationBuilder) (vaultUri: string) =
        // Requires Azure.Identity package
        // builder.Configuration.AddAzureKeyVault(
        //     Uri vaultUri,
        //     DefaultAzureCredential()) |> ignore
        ()
    
    // 4. AWS Secrets Manager
    // let getAwsSecret secretName = async {
    //     let client = AmazonSecretsManagerClient()
    //     let request = GetSecretValueRequest(SecretId = secretName)
    //     let! response = client.GetSecretValueAsync(request) |> Async.AwaitTask
    //     return response.SecretString
    // }
    
    // 5. HashiCorp Vault
    // ใช้ VaultSharp NuGet package
    
    // Configuration binding with validation
    type DatabaseConfig = {
        Host: string
        Port: int
        Database: string
        Username: string
        Password: string
    }
    
    let loadDatabaseConfig (config: IConfiguration) =
        let section = config.GetSection("Database")
        
        let cfg = {
            Host = section["Host"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Host required")
            Port = section["Port"] |> Option.ofObj |> Option.bind (fun s -> match System.Int32.TryParse(s) with true, n -> Some n | _ -> None) |> Option.defaultValue 5432
            Database = section["Database"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Database required")
            Username = section["Username"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Username required")
            Password = section["Password"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Password required")
        }
        cfg
    
    // Rotate secrets
    let rotateSecret (keyVaultClient: obj) secretName = async {
        // 1. Generate new secret
        let newSecret = generateSecureKey()
        
        // 2. Update in key vault
        // await keyVaultClient.SetSecretAsync(secretName, newSecret)
        
        // 3. Update in application (if needed)
        // Some applications can reload configuration dynamically
        
        return newSecret
    }
    and generateSecureKey () =
        let bytes = Array.zeroCreate<byte> 32
        use rng = System.Security.Cryptography.RandomNumberGenerator.Create()
        rng.GetBytes(bytes)
        System.Convert.ToBase64String(bytes)
```

---

## 9. Cryptography ใน .NET

```fsharp
// Cryptography best practices

module Cryptography =
    open System.Security.Cryptography
    
    // Hashing
    let sha256Hash (data: string) =
        let bytes = System.Text.Encoding.UTF8.GetBytes(data)
        let hash = SHA256.HashData(bytes)
        System.Convert.ToHexString(hash).ToLower()
    
    let sha512Hash (data: string) =
        let bytes = System.Text.Encoding.UTF8.GetBytes(data)
        let hash = SHA512.HashData(bytes)
        System.Convert.ToHexString(hash).ToLower()
    
    // HMAC for message authentication
    let hmacSha256 (key: byte[]) (data: string) =
        use hmac = new HMACSHA256(key)
        let bytes = System.Text.Encoding.UTF8.GetBytes(data)
        let hash = hmac.ComputeHash(bytes)
        System.Convert.ToBase64String(hash)
    
    // Symmetric encryption (AES-GCM)
    let encrypt (plaintext: string) (key: byte[]) =
        let plaintextBytes = System.Text.Encoding.UTF8.GetBytes(plaintext)
        let nonce = Array.zeroCreate<byte> AesGcm.NonceByteSizes.MaxSize
        let tag = Array.zeroCreate<byte> AesGcm.TagByteSizes.MaxSize
        let ciphertext = Array.zeroCreate<byte> plaintextBytes.Length
        
        RandomNumberGenerator.Fill(nonce)
        
        use aes = new AesGcm(key, AesGcm.TagByteSizes.MaxSize)
        aes.Encrypt(nonce, plaintextBytes, ciphertext, tag)
        
        // Return nonce + tag + ciphertext
        Array.concat [nonce; tag; ciphertext]
    
    let decrypt (encryptedData: byte[]) (key: byte[]) =
        let nonceSize = AesGcm.NonceByteSizes.MaxSize
        let tagSize = AesGcm.TagByteSizes.MaxSize
        
        let nonce = encryptedData.[0..nonceSize-1]
        let tag = encryptedData.[nonceSize..nonceSize+tagSize-1]
        let ciphertext = encryptedData.[nonceSize+tagSize..]
        let plaintext = Array.zeroCreate<byte> ciphertext.Length
        
        use aes = new AesGcm(key, AesGcm.TagByteSizes.MaxSize)
        aes.Decrypt(nonce, ciphertext, tag, plaintext)
        
        System.Text.Encoding.UTF8.GetString(plaintext)
    
    // Key derivation (PBKDF2)
    let deriveKey (password: string) (salt: byte[]) iterations keyLength =
        use pbkdf2 = new Rfc2898DeriveBytes(
            password, salt, iterations, HashAlgorithmName.SHA256)
        pbkdf2.GetBytes(keyLength)
    
    // Generate cryptographically secure random bytes
    let secureRandom length =
        let bytes = Array.zeroCreate<byte> length
        RandomNumberGenerator.Fill(bytes)
        bytes
    
    // Secure comparison (constant time)
    let secureEquals (a: string) (b: string) =
        CryptographicOperations.FixedTimeEquals(
            System.Text.Encoding.UTF8.GetBytes(a),
            System.Text.Encoding.UTF8.GetBytes(b))
    
    // RSA
    let generateRsaKeyPair () =
        use rsa = RSA.Create(2048)
        let privateKey = rsa.ExportRSAPrivateKey()
        let publicKey = rsa.ExportRSAPublicKey()
        privateKey, publicKey
    
    let rsaEncrypt (publicKey: byte[]) (data: string) =
        use rsa = RSA.Create()
        rsa.ImportRSAPublicKey(publicKey, ref 0)
        rsa.Encrypt(
            System.Text.Encoding.UTF8.GetBytes(data), 
            RSAEncryptionPadding.OaepSHA256)
    
    let rsaDecrypt (privateKey: byte[]) (ciphertext: byte[]) =
        use rsa = RSA.Create()
        rsa.ImportRSAPrivateKey(privateKey, ref 0)
        let plaintext = rsa.Decrypt(ciphertext, RSAEncryptionPadding.OaepSHA256)
        System.Text.Encoding.UTF8.GetString(plaintext)
    
    // Digital signatures
    let sign (privateKey: byte[]) (data: string) =
        use rsa = RSA.Create()
        rsa.ImportRSAPrivateKey(privateKey, ref 0)
        rsa.SignData(
            System.Text.Encoding.UTF8.GetBytes(data),
            HashAlgorithmName.SHA256,
            RSASignaturePadding.Pkcs1)
    
    let verify (publicKey: byte[]) (data: string) (signature: byte[]) =
        use rsa = RSA.Create()
        rsa.ImportRSAPublicKey(publicKey, ref 0)
        rsa.VerifyData(
            System.Text.Encoding.UTF8.GetBytes(data),
            signature,
            HashAlgorithmName.SHA256,
            RSASignaturePadding.Pkcs1)
```

---

## 10. HTTPS Enforcement และ Security Headers

```fsharp
// HTTPS และ Security Headers

module HttpsSecurity =
    open Microsoft.AspNetCore.Builder
    open Microsoft.AspNetCore.HttpsPolicy
    open Microsoft.Extensions.DependencyInjection
    
    let configureHttps (app: WebApplication) =
        // Redirect HTTP to HTTPS
        app.UseHttpsRedirection() |> ignore
        
        // HSTS (HTTP Strict Transport Security)
        app.UseHsts() |> ignore
    
    let configureHsts (services: IServiceCollection) =
        services.AddHsts(fun opts ->
            opts.Preload <- true
            opts.IncludeSubDomains <- true
            opts.MaxAge <- System.TimeSpan.FromDays(365.0)) |> ignore
    
    // Security headers middleware
    let addSecurityHeaders (next: RequestDelegate) =
        RequestDelegate(fun ctx -> task {
            // Prevent clickjacking
            ctx.Response.Headers.["X-Frame-Options"] <- "DENY"
            
            // Prevent MIME type sniffing
            ctx.Response.Headers.["X-Content-Type-Options"] <- "nosniff"
            
            // XSS protection
            ctx.Response.Headers.["X-XSS-Protection"] <- "1; mode=block"
            
            // Referrer policy
            ctx.Response.Headers.["Referrer-Policy"] <- "strict-origin-when-cross-origin"
            
            // Permissions policy
            ctx.Response.Headers.["Permissions-Policy"] <- 
                "geolocation=(), microphone=(), camera=(), payment=()"
            
            // Content Security Policy
            ctx.Response.Headers.["Content-Security-Policy"] <- 
                "default-src 'self'; " +
                "script-src 'self'; " +
                "style-src 'self' 'unsafe-inline'; " +
                "img-src 'self' data: https:; " +
                "font-src 'self'; " +
                "connect-src 'self'; " +
                "frame-ancestors 'none';"
            
            return! next.Invoke(ctx)
        })

// Rate Limiting
module RateLimiting =
    open Microsoft.AspNetCore.RateLimiting
    open System.Threading.RateLimiting
    
    let configureRateLimiting (services: IServiceCollection) =
        services.AddRateLimiter(fun opts ->
            // Fixed window rate limiter
            opts.AddFixedWindowLimiter("fixed", fun limiterOpts ->
                limiterOpts.PermitLimit <- 100  // 100 requests
                limiterOpts.Window <- System.TimeSpan.FromMinutes(1.0)
                limiterOpts.QueueProcessingOrder <- QueueProcessingOrder.OldestFirst
                limiterOpts.QueueLimit <- 10) |> ignore
            
            // Sliding window rate limiter
            opts.AddSlidingWindowLimiter("sliding", fun limiterOpts ->
                limiterOpts.PermitLimit <- 100
                limiterOpts.Window <- System.TimeSpan.FromMinutes(1.0)
                limiterOpts.SegmentsPerWindow <- 6
                limiterOpts.QueueProcessingOrder <- QueueProcessingOrder.OldestFirst
                limiterOpts.QueueLimit <- 10) |> ignore
            
            // Token bucket (for API rate limiting per user)
            opts.AddTokenBucketLimiter("token", fun limiterOpts ->
                limiterOpts.TokenLimit <- 100
                limiterOpts.QueueProcessingOrder <- QueueProcessingOrder.OldestFirst
                limiterOpts.QueueLimit <- 10
                limiterOpts.ReplenishmentPeriod <- System.TimeSpan.FromSeconds(10.0)
                limiterOpts.TokensPerPeriod <- 20
                limiterOpts.AutoReplenishment <- true) |> ignore
            
            opts.OnRejected <- fun ctx _ -> task {
                ctx.HttpContext.Response.StatusCode <- 429
                do! ctx.HttpContext.Response.WriteAsync(
                    "Too many requests. Please try again later.")
            })
        |> ignore
```

---

## สรุป

Security ใน F#/.NET ต้องพิจารณา:

1. **Input Validation** - Validate ทุก input จาก user
2. **SQL Injection** - ใช้ parameterized queries เสมอ
3. **XSS** - Encode output ก่อน render ใน HTML
4. **CSRF** - ใช้ antiforgery tokens และ SameSite cookies
5. **Authentication** - BCrypt สำหรับ passwords, JWT สำหรับ tokens
6. **Secrets** - ใช้ environment variables หรือ vault
7. **Cryptography** - ใช้ AES-GCM, RSA จาก .NET libraries
8. **HTTPS** - Enforce HTTPS, เพิ่ม security headers
9. **Rate Limiting** - ป้องกัน brute force attacks

**Security is not optional - it's a requirement!**

---

*ต่อไป: Part 96 - การ Deploy (Deployment and DevOps)*
