# Part 106 - Parser Combinators กับ F#

## บทนำ (Introduction)

Parser Combinators เป็น technique สำหรับสร้าง parsers โดยการรวม (combine) parsers เล็กๆ เข้าด้วยกันเพื่อสร้าง parsers ที่ซับซ้อนขึ้น FParsec เป็น library ยอดนิยมสำหรับ F# ที่ implement แนวคิดนี้

---

## 1. What are Parser Combinators?

### แนวคิดหลัก

```
Parser = Function ที่รับ input string และ return result + ส่วนที่เหลือ

type Parser<'T> = string -> Result<'T * string, string>
```

### ตัวอย่างง่ายๆ

```fsharp
// Parser Combinator แบบ naive (เพื่อแสดงแนวคิด)
type ParseResult<'T> = 
    | Success of value: 'T * remaining: string
    | Failure of error: string

type Parser<'T> = string -> ParseResult<'T>

// Parser ที่ parse ตัวอักษรเดียว
let pChar (expected: char) : Parser<char> =
    fun input ->
        if input.Length = 0 then
            Failure "Unexpected end of input"
        elif input.[0] = expected then
            Success (expected, input.[1..])
        else
            Failure (sprintf "Expected '%c' but got '%c'" expected input.[0])

// Combinator: AND (sequence)
let andThen (p1: Parser<'A>) (p2: Parser<'B>) : Parser<'A * 'B> =
    fun input ->
        match p1 input with
        | Failure err -> Failure err
        | Success (v1, remaining1) ->
            match p2 remaining1 with
            | Failure err -> Failure err
            | Success (v2, remaining2) ->
                Success ((v1, v2), remaining2)

// Combinator: OR (choice)
let orElse (p1: Parser<'T>) (p2: Parser<'T>) : Parser<'T> =
    fun input ->
        match p1 input with
        | Success result -> Success result
        | Failure _ -> p2 input

// ทดสอบ
let parseHello = 
    pChar 'H' 
    |> andThen (pChar 'e')
    |> andThen (pChar 'l')
    |> andThen (pChar 'l')
    |> andThen (pChar 'o')

match parseHello "Hello, World!" with
| Success (result, remaining) -> printfn "Parsed! Remaining: %s" remaining
| Failure err -> printfn "Failed: %s" err
```

---

## 2. FParsec Library

### การติดตั้ง

```xml
<PackageReference Include="FParsec" Version="1.1.1" />
```

```fsharp
// หรือใน script
#r "nuget: FParsec, 1.1.1"

open FParsec
```

### Types ใน FParsec

```fsharp
// Parser<'T, 'U> = Parser ที่ return 'T กับ user state 'U
// Parser<'T> = Parser<'T, unit> (ไม่มี user state)

// Result types
// Success<'T, 'U>(value, state, position)
// Failure<'T, 'U>(error, state)

// รัน parser
let run (p: Parser<'T, unit>) (input: string) : ParserResult<'T, unit> =
    FParsec.CharParsers.run p input

// รัน และ print ผลลัพธ์
let test p input =
    match run p input with
    | Success (result, _, _) -> printfn "Success: %A" result
    | Failure (err, _, _) -> printfn "Failure: %s" err
```

---

## 3. Basic Parsers: pchar, pstring

```fsharp
open FParsec

// ===== pchar =====

// Parse ตัวอักษรเดียว
let parseA = pchar 'A'
test parseA "ABC"    // Success: 'A'
test parseA "123"    // Failure

// anyChar - parse ตัวอักษรใดก็ได้
test anyChar "Hello"  // Success: 'H'

// satisfy - parse char ที่ตรงกับเงื่อนไข
let parseDigit = satisfy System.Char.IsDigit
test parseDigit "123"  // Success: '1'
test parseDigit "abc"  // Failure

// letter - parse ตัวอักษร a-z, A-Z
test letter "abc"      // Success: 'a'
test letter "123"      // Failure

// digit - parse ตัวเลข 0-9
test digit "123"       // Success: '1'
test digit "abc"       // Failure

// upper/lower - parse ตัวพิมพ์ใหญ่/เล็ก
test upper "ABC"       // Success: 'A'
test lower "abc"       // Success: 'a'

// space - parse whitespace
test spaces " \t\n"    // Success: ()

// ===== pstring =====

// Parse string ที่กำหนด
let parseHello = pstring "Hello"
test parseHello "Hello, World!"  // Success: "Hello"
test parseHello "World"           // Failure

// pstringCI - case insensitive
let parseHelloCI = pstringCI "hello"
test parseHelloCI "HELLO World"   // Success: "hello"
test parseHelloCI "Hello World"   // Success: "hello"

// ===== skipString =====

let skipHello = skipString "Hello"
test skipHello "Hello, World!"   // Skip และ return ()

// ===== regex =====

let parseIdentifier = regex @"[a-zA-Z_][a-zA-Z0-9_]*"
test parseIdentifier "myVariable123"  // Success: "myVariable123"
test parseIdentifier "123abc"          // Failure
```

---

## 4. Numeric Parsers

```fsharp
open FParsec

// ===== pint32 =====

test pint32 "42"        // Success: 42
test pint32 "-42"       // Success: -42
test pint32 "3.14"      // Success: 3 (หยุดที่ .)
test pint32 "abc"       // Failure

// ===== pint64 =====

test pint64 "9999999999"  // Success: 9999999999L

// ===== pfloat =====

test pfloat "3.14"       // Success: 3.14
test pfloat "-2.5e10"    // Success: -2.5e10
test pfloat "42"         // Success: 42.0
test pfloat "abc"        // Failure

// ===== puint32 =====

test puint32 "42"        // Success: 42u
test puint32 "-42"       // Failure (unsigned)

// ===== numberLiteral =====

let numberParser =
    numberLiteral NumberLiteralOptions.DefaultFloat "number"
    |>> fun nl ->
        if nl.IsFloat then float nl.String
        else float (int64 nl.String)

test numberParser "42"     // Success: 42.0
test numberParser "3.14"   // Success: 3.14
test numberParser "1e5"    // Success: 100000.0

// ===== Hexadecimal =====

let hexParser = pstring "0x" >>. many1 hex |>> fun chars ->
    System.Convert.ToInt32(System.String(Array.ofList chars), 16)

test hexParser "0xFF"     // Success: 255
test hexParser "0x1A2B"   // Success: 6699
```

---

## 5. Whitespace Parsers

```fsharp
open FParsec

// ===== spaces/spaces1 =====

// spaces - parse 0 หรือมากกว่า whitespace
test spaces "   hello"    // Success: ()
test spaces "hello"       // Success: () (0 spaces is OK)

// spaces1 - parse 1 หรือมากกว่า whitespace
test spaces1 "   hello"   // Success: ()
test spaces1 "hello"      // Failure (ต้องมี whitespace อย่างน้อย 1)

// ===== skipWhitespace =====

let ws = spaces  // alias

// ===== Token parser pattern =====

// สร้าง token ที่ skip whitespace รอบๆ
let token p = p .>> spaces
let tok c = token (pchar c)
let keyword s = token (pstring s)

// ===== lexeme =====

// Parse และ skip whitespace หลัง
let lexeme p = p .>> spaces

let integer = lexeme pint32
let float' = lexeme pfloat

// ===== ตัวอย่างการใช้งาน =====

// Parse arithmetic expression แบบง่าย
let simpleExpr : Parser<int, unit> =
    spaces >>. pint32 .>> spaces

test simpleExpr "  42  "    // Success: 42
test simpleExpr "\t 100\n"  // Success: 100
```

---

## 6. Combining Parsers

### Sequential Combination

```fsharp
open FParsec

// ===== >>. (then right) =====
// รัน p1 แล้ว p2, return ผลลัพธ์ของ p2

let parseAfterColon = pchar ':' >>. spaces >>. pint32
test parseAfterColon ": 42"    // Success: 42

// ===== .>> (then left) =====
// รัน p1 แล้ว p2, return ผลลัพธ์ของ p1

let parseBeforeComma = pint32 .>> pchar ','
test parseBeforeComma "42,rest"  // Success: 42

// ===== .>>. (tuple) =====
// รัน p1 แล้ว p2, return tuple

let parsePair = pint32 .>> spaces .>>. pint32
test parsePair "1 2"    // Success: (1, 2)

// ===== pipe2, pipe3, ... =====

let parsePoint =
    pipe2
        (pchar '(' >>. pint32)
        (pchar ',' >>. pint32 .>> pchar ')')
        (fun x y -> (x, y))

test parsePoint "(10,20)"   // Success: (10, 20)

// pipe3
let parseTriple =
    pipe3
        (pchar '(' >>. pint32 .>> pchar ',')
        (pint32 .>> pchar ',')
        (pint32 .>> pchar ')')
        (fun a b c -> (a, b, c))

test parseTriple "(1,2,3)"  // Success: (1, 2, 3)

// ===== tuple2, tuple3 =====

let parsePair2 = tuple2 pint32 (spaces >>. pint32)
test parsePair2 "10 20"    // Success: (10, 20)

// ===== |>> (map) =====

// Transform ผลลัพธ์
let parseDouble = pint32 |>> fun n -> n * 2
test parseDouble "21"   // Success: 42

// ===== >>% (return constant) =====

let parseTrue = pstring "true" >>% true
let parseFalse = pstring "false" >>% false
test parseTrue "true and more"    // Success: true
test parseFalse "false remaining" // Success: false
```

---

## 7. Choice: <|>

```fsharp
open FParsec

// ===== <|> (or) =====

let parseHelloOrWorld =
    pstring "Hello" <|> pstring "World"

test parseHelloOrWorld "Hello!"  // Success: "Hello"
test parseHelloOrWorld "World!"  // Success: "World"
test parseHelloOrWorld "Other"   // Failure

// ===== choice =====

let parseBool =
    choice [
        pstring "true" >>% true
        pstring "false" >>% false
        pstring "yes" >>% true
        pstring "no" >>% false
    ]

test parseBool "true"    // Success: true
test parseBool "yes"     // Success: true
test parseBool "false"   // Success: false

// ===== attempt =====

// Backtracking - ถ้า parser ล้มเหลวหลังจาก consume บาง input
// ใช้ attempt เพื่อ restore position

let parseKeyword =
    attempt (pstring "if" .>> notFollowedBy letter)
    <|> attempt (pstring "in" .>> notFollowedBy letter)
    <|> (many1 letter |>> System.String.Concat)

test parseKeyword "if "      // Success: "if"
test parseKeyword "in "      // Success: "in"
test parseKeyword "if_var"   // Failure -> Success: "if_var" (backtrack)

// ===== notFollowedBy =====

// ตรวจสอบว่า ไม่ตามด้วย pattern ที่กำหนด
let notKeyword s = 
    notFollowedBy (pstring s) >>% s

// ===== followedBy =====

// ตรวจสอบว่าตามด้วย pattern (lookahead, ไม่ consume)
let followedByDigit = followedBy digit

test followedByDigit "123"  // Success: ()
test followedByDigit "abc"  // Failure
```

---

## 8. Repetition: many, many1

```fsharp
open FParsec

// ===== many =====
// Parse 0 หรือมากกว่า

let manyDigits = many digit |>> System.String.Concat
test manyDigits "12345abc"  // Success: "12345"
test manyDigits "abcdef"    // Success: "" (0 matches)

// ===== many1 =====
// Parse 1 หรือมากกว่า

let many1Digits = many1 digit |>> System.String.Concat
test many1Digits "12345abc"  // Success: "12345"
test many1Digits "abcdef"    // Failure

// ===== manyTill =====
// Parse จนกว่าจะเจอ end parser

let parseComment = 
    pstring "//" >>. manyTill anyChar newline |>> System.String.Concat

test parseComment "// This is a comment\nnext line"
// Success: " This is a comment"

// ===== sepBy / sepBy1 =====
// Parse separated by delimiter

let csvLine = sepBy pint32 (pchar ',')
test csvLine "1,2,3,4,5"  // Success: [1; 2; 3; 4; 5]
test csvLine ""             // Success: [] (0 items)

let csvLine1 = sepBy1 pint32 (pchar ',')
test csvLine1 "1,2,3"     // Success: [1; 2; 3]
test csvLine1 ""           // Failure (ต้องมีอย่างน้อย 1)

// ===== sepEndBy =====
// Separated and optionally ended

let items = sepEndBy pint32 (pchar ',')
test items "1,2,3,"   // Success: [1; 2; 3] (ตาม comma ท้าย ok)

// ===== count =====
// Parse จำนวนที่กำหนด

let threeDigits = count 3 digit
test threeDigits "123abc"  // Success: ['1'; '2'; '3']
test threeDigits "12abc"   // Failure (ต้องการ 3 ตัว)

// ===== parray =====

let threeInts = parray 3 (pint32 .>> spaces)
test threeInts "10 20 30 rest"  // Success: [|10; 20; 30|]

// ===== manyChars =====

let identifier = 
    (letter <|> pchar '_') .>>. manyChars (letter <|> digit <|> pchar '_')
    |>> (fun (first, rest) -> string first + rest)

test identifier "myVariable123"  // Success: "myVariable123"
test identifier "_private"       // Success: "_private"
test identifier "123abc"         // Failure
```

---

## 9. Optional: opt

```fsharp
open FParsec

// ===== opt =====
// Return Some value หรือ None

let optionalSign = opt (pchar '-' <|> pchar '+')
test optionalSign "-42"   // Success: Some '-'
test optionalSign "+42"   // Success: Some '+'
test optionalSign "42"    // Success: None

// ===== optional =====
// Parse หรือ skip (return unit)

let optionalSemicolon = optional (pchar ';')
test optionalSemicolon ";"    // Success: ()
test optionalSemicolon "next" // Success: ()

// ===== ตัวอย่าง: signed number =====

let signedInt =
    opt (pchar '-') .>>. pint32
    |>> fun (sign, n) ->
        match sign with
        | Some '-' -> -n
        | _ -> n

test signedInt "-42"    // Success: -42
test signedInt "42"     // Success: 42

// ===== defaultValue =====

let optWithDefault defaultVal p =
    opt p |>> Option.defaultValue defaultVal

let portParser = 
    pstring "http://" >>. many1 (noneOf ":/") |>> System.String.Concat
    .>>. (optWithDefault 80 (pchar ':' >>. pint32))

test portParser "http://example.com:8080"  // Success: ("example.com", 8080)
test portParser "http://example.com"       // Success: ("example.com", 80)
```

---

## 10. Recursive Parsers

```fsharp
open FParsec

// ===== Forward Reference =====

// สำหรับ recursive grammars ต้องใช้ createParserForwardedToRef
let expr, exprRef = createParserForwardedToRef<int, unit>()

// ===== Arithmetic Expression Parser =====

// Grammar:
// expr = term (('+' | '-') term)*
// term = factor (('*' | '/') factor)*
// factor = number | '(' expr ')'

let ws = spaces
let number = pint32 .>> ws

let factor = 
    number <|> (pchar '(' >>. ws >>. expr .>> ws .>> pchar ')')

let term = 
    factor .>>. many (ws >>. (pchar '*' <|> pchar '/') .>> ws .>>. factor)
    |>> fun (first, rest) ->
        rest |> List.fold (fun acc (op, n) ->
            match op with
            | '*' -> acc * n
            | '/' -> acc / n
            | _ -> acc) first

do exprRef.Value <-
    term .>>. many (ws >>. (pchar '+' <|> pchar '-') .>> ws .>>. term)
    |>> fun (first, rest) ->
        rest |> List.fold (fun acc (op, n) ->
            match op with
            | '+' -> acc + n
            | '-' -> acc - n
            | _ -> acc) first

// ทดสอบ
test expr "1 + 2 * 3"        // Success: 7
test expr "(1 + 2) * 3"      // Success: 9
test expr "10 - 2 * 3 + 4"   // Success: 8

// ===== JSON Value (Recursive) =====

type JsonValue =
    | JsonNull
    | JsonBool of bool
    | JsonNumber of float
    | JsonString of string
    | JsonArray of JsonValue list
    | JsonObject of (string * JsonValue) list

let jvalue, jvalueRef = createParserForwardedToRef<JsonValue, unit>()

let jnull = pstring "null" >>% JsonNull

let jbool = 
    (pstring "true" >>% JsonBool true) <|>
    (pstring "false" >>% JsonBool false)

let jnumber = pfloat |>> JsonNumber

let jstring =
    let escaped = 
        pchar '\\' >>. (
            pchar '"' >>% '"' <|>
            pchar '\\' >>% '\\' <|>
            pchar 'n' >>% '\n' <|>
            pchar 't' >>% '\t' <|>
            pchar 'r' >>% '\r'
        )
    let normalChar = noneOf "\"\\"
    pchar '"' >>. many (escaped <|> normalChar) .>> pchar '"'
    |>> (Array.ofList >> System.String >> JsonString)

let jarray =
    pchar '[' >>. ws >>. sepBy (jvalue .>> ws) (pchar ',' .>> ws) .>> pchar ']'
    |>> JsonArray

let jobject =
    let jmember = jstring .>> ws .>> pchar ':' .>> ws .>>. jvalue
    pchar '{' >>. ws >>. 
    sepBy (jmember .>> ws) (pchar ',' .>> ws) .>> 
    pchar '}'
    |>> fun members ->
        members |> List.map (fun (k, v) -> 
            (match k with JsonString s -> s | _ -> ""), v)
        |> JsonObject

do jvalueRef.Value <- 
    ws >>. choice [
        jnull
        jbool
        jnumber
        jstring
        jarray
        jobject
    ]

// ทดสอบ JSON parser
test jvalue """null"""
test jvalue """true"""
test jvalue """42"""
test jvalue """"hello world""""
test jvalue """[1, 2, 3]"""
test jvalue """{"name": "สมชาย", "age": 30}"""
test jvalue """
{
    "users": [
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"}
    ],
    "count": 2
}
"""
```

---

## 11. Building a JSON Parser (Complete)

```fsharp
// json-parser.fsx
#r "nuget: FParsec, 1.1.1"

open FParsec

// ===== JSON Types =====

type Json =
    | JNull
    | JBool of bool
    | JNumber of decimal
    | JString of string
    | JArray of Json list
    | JObject of Map<string, Json>

// ===== Helpers =====

let ws = spaces
let ws1 = spaces1

let betweenSpaces p = between ws ws p

// ===== Primitive Parsers =====

let jNull = pstring "null" >>% JNull .>> ws

let jBool =
    choice [
        pstring "true" >>% JBool true
        pstring "false" >>% JBool false
    ] .>> ws

let jNumber =
    numberLiteral 
        (NumberLiteralOptions.AllowFraction ||| NumberLiteralOptions.AllowExponent) 
        "number"
    |>> fun nl -> JNumber (decimal nl.String)
    .>> ws

// ===== String Parser =====

let jString =
    let unicodeEscape =
        pstring "\\u" >>. count 4 hex 
        |>> fun chars ->
            System.Convert.ToChar(System.Convert.ToInt32(System.String(Array.ofList chars), 16))
    
    let escaped =
        pchar '\\' >>. choice [
            pchar '"'  >>% '"'
            pchar '\\' >>% '\\'
            pchar '/'  >>% '/'
            pchar 'b'  >>% '\b'
            pchar 'f'  >>% '\f'
            pchar 'n'  >>% '\n'
            pchar 'r'  >>% '\r'
            pchar 't'  >>% '\t'
            unicodeEscape
        ]
    
    let normalChar = noneOf "\"\\"
    
    between (pchar '"') (pchar '"') (manyChars (escaped <|> normalChar))
    |>> JString
    .>> ws

// ===== Forward Reference =====

let json, jsonRef = createParserForwardedToRef<Json, unit>()

// ===== Array Parser =====

let jArray =
    between 
        (pchar '[' .>> ws) 
        (pchar ']' .>> ws)
        (sepBy json (pchar ',' .>> ws))
    |>> JArray

// ===== Object Parser =====

let jObject =
    let key = 
        between (pchar '"') (pchar '"') (manyChars (noneOf "\""))
        .>> ws .>> pchar ':' .>> ws
    
    let pair = key .>>. json |>> fun (k, v) -> (k, v)
    
    between
        (pchar '{' .>> ws)
        (pchar '}' .>> ws)
        (sepBy pair (pchar ',' .>> ws))
    |>> (Map.ofList >> JObject)

// ===== Main Parser =====

do jsonRef.Value <- 
    ws >>. choice [
        jNull
        jBool
        jNumber
        jString
        jArray
        jObject
    ]

// ===== Pretty Printer =====

let rec prettyPrint (indent: int) (json: Json) : string =
    let ind = System.String(' ', indent)
    let ind2 = System.String(' ', indent + 2)
    
    match json with
    | JNull -> "null"
    | JBool b -> if b then "true" else "false"
    | JNumber n -> 
        if n = System.Math.Floor(n) then sprintf "%d" (int n)
        else sprintf "%g" (float n)
    | JString s -> sprintf "\"%s\"" s
    | JArray items ->
        if List.isEmpty items then "[]"
        else
            let inner = items |> List.map (prettyPrint (indent + 2)) |> String.concat (sprintf ",\n%s" ind2)
            sprintf "[\n%s%s\n%s]" ind2 inner ind
    | JObject map ->
        if Map.isEmpty map then "{}"
        else
            let inner =
                map
                |> Map.toList
                |> List.map (fun (k, v) -> sprintf "\"%s\": %s" k (prettyPrint (indent + 2) v))
                |> String.concat (sprintf ",\n%s" ind2)
            sprintf "{\n%s%s\n%s}" ind2 inner ind

// ===== Query Functions =====

let get (key: string) (json: Json) =
    match json with
    | JObject map -> Map.tryFind key map
    | _ -> None

let getArray (json: Json) =
    match json with
    | JArray items -> Some items
    | _ -> None

let getString (json: Json) =
    match json with
    | JString s -> Some s
    | _ -> None

let getNumber (json: Json) =
    match json with
    | JNumber n -> Some n
    | _ -> None

// ===== Test =====

let testJson = """
{
    "name": "สมชาย ใจดี",
    "age": 30,
    "active": true,
    "address": {
        "street": "123 ถนนสุขุมวิท",
        "city": "กรุงเทพมหานคร",
        "zip": "10110"
    },
    "skills": ["F#", "C#", "Python"],
    "score": null,
    "rating": 4.5
}
"""

match run json testJson with
| Success (result, _, _) ->
    printfn "Parse successful!"
    printfn "\nFormatted:"
    printfn "%s" (prettyPrint 0 result)
    
    // Query
    result |> get "name" |> Option.bind getString |> Option.iter (printfn "\nName: %s")
    result |> get "age" |> Option.bind getNumber |> Option.iter (printfn "Age: %g")
    result |> get "skills" |> Option.bind getArray |> Option.iter (fun skills ->
        printfn "Skills: %s" (skills |> List.choose getString |> String.concat ", "))
    
| Failure (err, _, _) ->
    printfn "Parse failed: %s" err
```

---

## 12. Building a CSV Parser

```fsharp
// csv-parser.fsx
#r "nuget: FParsec, 1.1.1"

open FParsec

// ===== CSV Parser =====

// CSV field: quoted string หรือ unquoted string
let quotedField =
    let escaped = pstring "\"\"" >>% '"'
    let normalChar = noneOf "\""
    between (pchar '"') (pchar '"') (manyChars (escaped <|> normalChar))

let unquotedField =
    manyChars (noneOf ",\r\n")

let csvField = quotedField <|> unquotedField

let csvRecord = sepBy csvField (pchar ',')

let csvFile = sepEndBy csvRecord newline .>> eof

// ===== Parse CSV Data =====

let parseCsv (content: string) =
    match run csvFile content with
    | Success (rows, _, _) -> Ok rows
    | Failure (err, _, _) -> Error err

// ===== Test =====

let csvData = """Name,Age,City,Email
"สมชาย ใจดี",30,"กรุงเทพ",somchai@example.com
"สมหญิง รักดี",25,"เชียงใหม่",somying@example.com
"""

match parseCsv csvData with
| Ok rows ->
    let headers = rows.[0]
    let dataRows = rows |> List.skip 1 |> List.filter (fun r -> r <> [""])
    
    printfn "Headers: %A" headers
    printfn "\nRows:"
    for row in dataRows do
        for (header, value) in List.zip headers row do
            printfn "  %s: %s" header value
        printfn ""
        
| Error err ->
    printfn "CSV parse error: %s" err
```

---

## 13. Building a Calculator Parser

```fsharp
// calculator.fsx
#r "nuget: FParsec, 1.1.1"

open FParsec

// ===== AST =====

type Expr =
    | Number of float
    | Variable of string
    | BinaryOp of Expr * char * Expr
    | UnaryMinus of Expr
    | FunctionCall of string * Expr list
    | IfExpr of Expr * Expr * Expr  // if condition then true-expr else false-expr

// ===== Parser =====

let ws = spaces
let str s = pstring s .>> ws

let number = pfloat .>> ws |>> Number

let variable = 
    many1Chars2 (letter <|> pchar '_') (letter <|> digit <|> pchar '_')
    .>> ws
    |>> Variable

let expr, exprRef = createParserForwardedToRef<Expr, unit>()

// Function call: name(arg1, arg2, ...)
let functionCall =
    many1Chars (letter <|> pchar '_') .>> ws .>> pchar '(' .>> ws
    .>>. sepBy expr (str ",")
    .>> str ")"
    |>> (fun (name, args) -> FunctionCall(name, args))

let factor =
    ws >>. choice [
        attempt functionCall
        number
        attempt variable
        str "(" >>. expr .>> str ")"
        str "-" >>. factor |>> UnaryMinus
    ]

let term =
    factor .>>. many (ws >>. (pchar '*' <|> pchar '/' <|> pchar '%') .>> ws .>>. factor)
    |>> fun (first, ops) ->
        ops |> List.fold (fun acc (op, right) ->
            BinaryOp(acc, op, right)) first

do exprRef.Value <-
    term .>>. many (ws >>. (pchar '+' <|> pchar '-') .>> ws .>>. term)
    |>> fun (first, ops) ->
        ops |> List.fold (fun acc (op, right) ->
            BinaryOp(acc, op, right)) first

// ===== Evaluator =====

let rec evaluate (vars: Map<string, float>) (expr: Expr) : float =
    match expr with
    | Number n -> n
    | Variable name ->
        Map.tryFind name vars
        |> Option.defaultWith (fun () -> failwith (sprintf "Unknown variable: %s" name))
    | UnaryMinus e -> -(evaluate vars e)
    | BinaryOp (left, op, right) ->
        let l = evaluate vars left
        let r = evaluate vars right
        match op with
        | '+' -> l + r
        | '-' -> l - r
        | '*' -> l * r
        | '/' -> if r = 0.0 then failwith "Division by zero" else l / r
        | '%' -> l % r
        | _ -> failwith (sprintf "Unknown operator: %c" op)
    | FunctionCall (name, args) ->
        let argValues = args |> List.map (evaluate vars)
        match name, argValues with
        | "sqrt", [x] -> sqrt x
        | "abs", [x] -> abs x
        | "pow", [x; y] -> System.Math.Pow(x, y)
        | "max", [x; y] -> max x y
        | "min", [x; y] -> min x y
        | "sin", [x] -> sin x
        | "cos", [x] -> cos x
        | "log", [x] -> log x
        | _ -> failwith (sprintf "Unknown function: %s" name)
    | IfExpr (cond, trueExpr, falseExpr) ->
        if evaluate vars cond <> 0.0 then evaluate vars trueExpr
        else evaluate vars falseExpr

// ===== Formatter =====

let rec formatExpr = function
    | Number n -> sprintf "%g" n
    | Variable name -> name
    | UnaryMinus e -> sprintf "-(  %s)" (formatExpr e)
    | BinaryOp (l, op, r) -> sprintf "(%s %c %s)" (formatExpr l) op (formatExpr r)
    | FunctionCall (name, args) ->
        sprintf "%s(%s)" name (args |> List.map formatExpr |> String.concat ", ")
    | IfExpr (c, t, f) ->
        sprintf "if %s then %s else %s" (formatExpr c) (formatExpr t) (formatExpr f)

// ===== Calculator =====

let calculate (input: string) (vars: Map<string, float>) =
    match run (expr .>> eof) input with
    | Success (ast, _, _) ->
        try
            let result = evaluate vars ast
            Ok result
        with ex ->
            Error ex.Message
    | Failure (err, _, _) ->
        Error (sprintf "Parse error: %s" err)

// ===== Tests =====

let vars = Map.ofList [("x", 10.0); ("y", 5.0); ("pi", System.Math.PI)]

let testCalc expr expected =
    match calculate expr vars with
    | Ok result ->
        let ok = abs(result - expected) < 1e-10
        printfn "%s %s = %g (expected %g)" 
            (if ok then "✓" else "✗") 
            expr result expected
    | Error err ->
        printfn "✗ %s -> Error: %s" expr err

testCalc "1 + 2" 3.0
testCalc "10 * 2 + 5" 25.0
testCalc "(10 + 5) * 2" 30.0
testCalc "x + y" 15.0
testCalc "sqrt(x * x + y * y)" (sqrt 125.0)
testCalc "pow(2, 10)" 1024.0
testCalc "sin(pi)" 0.0
testCalc "-x + 20" 10.0
```

---

## 14. Error Messages

```fsharp
open FParsec

// ===== Custom Error Messages =====

let withError (msg: string) p =
    p <?> msg  // <?> ตั้งชื่อ parser

let identifier = 
    many1Chars2 (letter <|> pchar '_') (letter <|> digit <|> pchar '_')
    <?> "identifier"

let integer = pint32 <?> "integer number"

// ===== Position Information =====

let withPosition (p: Parser<'T, 'U>) : Parser<'T * Position, 'U> =
    fun stream ->
        let pos = stream.Position
        let reply = p stream
        match reply.Status with
        | Ok -> Reply((reply.Result, pos))
        | _ -> Reply(reply.Status, reply.Error)

let positionedInt = withPosition pint32

match run positionedInt "42" with
| Success ((n, pos), _, _) ->
    printfn "Number %d at line %d, column %d" n pos.Line pos.Column
| Failure _ -> ()

// ===== Error Recovery =====

// Parse list แต่ skip items ที่ parse ไม่ได้
let parseIntOrSkip =
    many (attempt (pint32 .>> spaces) <|> (skipMany1 (noneOf " \n") .>> spaces >>% 0))

test parseIntOrSkip "1 hello 3 world 5"
// Success: [1; 0; 3; 0; 5]

// ===== Diagnostic Parser =====

let diagnosticParser (name: string) (p: Parser<'T, 'U>) =
    fun stream ->
        let pos = stream.Position
        printfn "[DEBUG] Trying %s at line %d, col %d" name pos.Line pos.Column
        let reply = p stream
        match reply.Status with
        | Ok -> printfn "[DEBUG] %s succeeded" name
        | _ -> printfn "[DEBUG] %s failed" name
        reply
```

---

## สรุป (Summary)

Parser Combinators และ FParsec ช่วยให้เราสร้าง parsers ที่ซับซ้อนได้อย่างสวยงาม:

1. **Basic Parsers**: pchar, pstring, pint32, pfloat
2. **Combinators**: .>>., >>., .<< สำหรับ sequencing
3. **Choice**: <|>, choice สำหรับ alternatives
4. **Repetition**: many, many1, sepBy สำหรับ repetition
5. **Optional**: opt สำหรับ optional parsers
6. **Recursive**: createParserForwardedToRef สำหรับ recursive grammars
7. **Error Messages**: <?> สำหรับ custom error messages

---

*ไปต่อที่ Part 107: Data Science กับ F#*
