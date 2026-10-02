# Part 49 - Quotations (Code as Data)

## บทนำ

F# Quotations ช่วยให้เราแสดง F# code เป็น data structure ที่สามารถ analyze, transform, และ execute ได้ในเวลา runtime คล้ายกับ LINQ Expression Trees ใน C# แต่ทรงพลังกว่า

## 1. Quoted Expressions <@ expr @>

```fsharp
open Microsoft.FSharp.Quotations
open Microsoft.FSharp.Quotations.Patterns
open Microsoft.FSharp.Quotations.DerivedPatterns

// <@ expr @> สร้าง typed quotation
let simpleQuote: Expr<int> = <@ 1 + 2 @>
printfn "Quotation: %A" simpleQuote

// Quotation เป็น Expr<T> - มี type information
let addQuote: Expr<int -> int -> int> = <@ fun x y -> x + y @>
printfn "Function quote: %A" addQuote

// Quotation ของ expression ที่ซับซ้อน
let complexQuote = <@ 
    let x = 10
    let y = 20
    x + y 
@>
printfn "Complex: %A" complexQuote
```

```fsharp
// Quotation ของ function calls
let listOpsQuote = <@
    [1; 2; 3; 4; 5]
    |> List.filter (fun x -> x % 2 = 0)
    |> List.map (fun x -> x * x)
    |> List.sum
@>

printfn "List ops quote type: %s" (listOpsQuote.GetType().Name)
```

## 2. <@@ expr @@> Raw Quoted Expressions

```fsharp
// <@@ expr @@> สร้าง untyped quotation (Expr ไม่มี type parameter)
let rawQuote: Expr = <@@ 1 + 2 @@>
printfn "Raw quotation: %A" rawQuote

// เปรียบเทียบ typed vs untyped
let typed: Expr<int> = <@ 42 @>
let untyped: Expr = <@@ 42 @@>

// แปลงระหว่างกัน
let typedToUntyped: Expr = typed.Raw
let untypedToTyped: Expr<int> = Expr.Cast<int>(untyped)

printfn "typed: %A" typed
printfn "untyped: %A" untyped
```

```fsharp
// Raw quotation ใช้ใน generic contexts
let printQuotation (expr: Expr) =
    printfn "Expression: %A" expr
    printfn "Type: %s" expr.Type.Name

printQuotation <@@ "Hello" @@>
printQuotation <@@ 42 @@>
printQuotation <@@ [1; 2; 3] @@>
```

## 3. Expr Type

```fsharp
// Expr มี properties สำคัญ
let examineExpr (expr: Expr) =
    printfn "Type: %s" expr.Type.FullName
    printfn "CustomAttributes: %A" expr.CustomAttributes

let intExpr = <@@ 42 @@>
let strExpr = <@@ "Hello" @@>
let listExpr = <@@ [1; 2; 3] @@>

examineExpr intExpr
examineExpr strExpr
examineExpr listExpr
```

```fsharp
// Expr มี subtypes ต่างๆ
// Lambda, Application, Let, Value, Call, etc.

let analyzeExpr (expr: Expr) =
    match expr with
    | Value(v, t) -> 
        printfn "Value: %A (type: %s)" v t.Name
    | Lambda(param, body) -> 
        printfn "Lambda: %s -> ..." param.Name
    | Application(f, arg) -> 
        printfn "Application: function applied to arg"
    | Let(var, value, body) ->
        printfn "Let: %s = ..." var.Name
    | Call(None, methodInfo, args) ->
        printfn "Static call: %s.%s" methodInfo.DeclaringType.Name methodInfo.Name
    | Call(Some obj, methodInfo, args) ->
        printfn "Instance call: .%s" methodInfo.Name
    | _ ->
        printfn "Other: %s" (expr.GetType().Name)

analyzeExpr <@@ 42 @@>
analyzeExpr <@@ fun x -> x + 1 @@>
analyzeExpr <@@ let x = 5 in x * 2 @@>
analyzeExpr <@@ List.length [1; 2; 3] @@>
```

## 4. Examining Quotation Structure

```fsharp
// ดู structure ของ quotation แบบ recursive
let rec printStructure (indent: int) (expr: Expr) =
    let prefix = String.replicate indent "  "
    
    match expr with
    | Value(v, t) ->
        printfn "%sValue(%A : %s)" prefix v t.Name
    
    | Var(v) ->
        printfn "%sVar(%s : %s)" prefix v.Name v.Type.Name
    
    | Lambda(param, body) ->
        printfn "%sLambda(%s ->" prefix param.Name
        printStructure (indent + 1) body
        printfn "%s)" prefix
    
    | Application(f, arg) ->
        printfn "%sApplication(" prefix
        printfn "%s  Function:" prefix
        printStructure (indent + 2) f
        printfn "%s  Argument:" prefix
        printStructure (indent + 2) arg
        printfn "%s)" prefix
    
    | Let(var, value, body) ->
        printfn "%sLet %s =" prefix var.Name
        printStructure (indent + 1) value
        printfn "%sIn" prefix
        printStructure (indent + 1) body
    
    | Call(obj, methodInfo, args) ->
        let objStr = match obj with Some _ -> "instance" | None -> "static"
        printfn "%sCall[%s] %s.%s" prefix objStr methodInfo.DeclaringType.Name methodInfo.Name
        for arg in args do
            printStructure (indent + 1) arg
    
    | IfThenElse(cond, thenBranch, elseBranch) ->
        printfn "%sIfThenElse" prefix
        printfn "%s  Condition:" prefix
        printStructure (indent + 2) cond
        printfn "%s  Then:" prefix
        printStructure (indent + 2) thenBranch
        printfn "%s  Else:" prefix
        printStructure (indent + 2) elseBranch
    
    | _ ->
        printfn "%s[%s]" prefix (expr.GetType().Name)

// ทดสอบ
printfn "\nStructure ของ <@ fun x -> x * 2 @>:"
printStructure 0 <@@ fun (x: int) -> x * 2 @@>

printfn "\nStructure ของ <@ if x > 0 then x else -x @>:"
printStructure 0 <@@ fun (x: int) -> if x > 0 then x else -x @@>
```

## 5. Pattern Matching บน Expr

```fsharp
// Pattern matching ด้วย active patterns จาก Patterns module
let extractValue (expr: Expr) =
    match expr with
    | Int32(n) -> printfn "Int32: %d" n
    | String(s) -> printfn "String: %s" s
    | Bool(b) -> printfn "Bool: %b" b
    | Double(d) -> printfn "Double: %f" d
    | Value(v, t) -> printfn "Value: %A (%s)" v t.Name
    | _ -> printfn "Other expression"

extractValue <@@ 42 @@>
extractValue <@@ "hello" @@>
extractValue <@@ true @@>
extractValue <@@ 3.14 @@>
```

```fsharp
// Extract operations
let getOperation (expr: Expr) =
    match expr with
    | SpecificCall <@@ (+) @@> (None, _, [lhs; rhs]) ->
        printfn "Addition: (%A) + (%A)" lhs rhs
        Some "+"
    | SpecificCall <@@ (-) @@> (None, _, [lhs; rhs]) ->
        printfn "Subtraction: (%A) - (%A)" lhs rhs
        Some "-"
    | SpecificCall <@@ (*) @@> (None, _, [lhs; rhs]) ->
        printfn "Multiplication: (%A) * (%A)" lhs rhs
        Some "*"
    | SpecificCall <@@ (/) @@> (None, _, [lhs; rhs]) ->
        printfn "Division: (%A) / (%A)" lhs rhs
        Some "/"
    | _ ->
        printfn "Unknown operation"
        None

getOperation <@@ 10 + 5 @@> |> ignore
getOperation <@@ 10 - 5 @@> |> ignore
getOperation <@@ 10 * 5 @@> |> ignore
```

## 6. Building Expressions Programmatically

```fsharp
// สร้าง Expr โดยใช้ Expr module functions
let buildAddExpression () =
    // สร้าง Value nodes
    let five = Expr.Value(5)
    let three = Expr.Value(3)
    
    // สร้าง Add call
    let addMethod = 
        typeof<Microsoft.FSharp.Core.Operators>.GetMethod("op_Addition", 
            [| typeof<int>; typeof<int> |])
    
    // ถ้าหา method ไม่เจอ, ใช้ วิธีอื่น
    match addMethod with
    | null ->
        printfn "ไม่พบ Add method"
    | m ->
        let addExpr = Expr.Call(m, [five; three])
        printfn "Built expression: %A" addExpr

buildAddExpression ()
```

```fsharp
// สร้าง Lambda expression
let buildLambda () =
    // สร้าง parameter
    let param = Var("x", typeof<int>)
    let paramExpr = Expr.Var(param)
    
    // สร้าง body: x * x
    let body = <@@ %%paramExpr * %%paramExpr @@>
    
    // สร้าง lambda
    let lambda = Expr.Lambda(param, body)
    printfn "Lambda: %A" lambda
    printfn "Lambda type: %s" lambda.Type.Name

buildLambda ()
```

```fsharp
// Splicing: แทรก expression เข้าใน quotation
let splicingExample () =
    // Typed splicing (%%)
    let innerExpr: Expr<int> = <@ 42 @>
    let outerExpr = <@ %innerExpr + 1 @>
    printfn "Spliced typed: %A" outerExpr
    
    // Untyped splicing (%%%)
    let rawInner: Expr = <@@ 100 @@>
    let withRaw = <@ %%rawInner + 10 @>
    // Note: ใช้ %% กับ Expr<T> หรือ %%% กับ Expr
    printfn "Expression with raw splice: %A" (Expr.Value(110))

splicingExample ()
```

## 7. Evaluating Quotations

```fsharp
// F# ไม่มี built-in quotation evaluator
// แต่ FSharp.Quotations.Evaluator package ช่วยได้

// ตัวอย่าง: สร้าง evaluator อย่างง่าย

// evaluate arithmetic expressions
let rec evalArith (expr: Expr) : int =
    match expr with
    | Int32(n) -> n
    | SpecificCall <@@ (+) @@> (_, _, [lhs; rhs]) ->
        evalArith lhs + evalArith rhs
    | SpecificCall <@@ (-) @@> (_, _, [lhs; rhs]) ->
        evalArith lhs - evalArith rhs
    | SpecificCall <@@ (*) @@> (_, _, [lhs; rhs]) ->
        evalArith lhs * evalArith rhs
    | SpecificCall <@@ (/) @@> (_, _, [lhs; rhs]) ->
        evalArith lhs / evalArith rhs
    | _ ->
        failwithf "ไม่รองรับ expression: %A" expr

// ทดสอบ
let e1 = <@@ 10 + 5 @@>
let e2 = <@@ 3 * 4 @@>
let e3 = <@@ 20 - 8 @@>

printfn "10 + 5 = %d" (evalArith e1)
printfn "3 * 4 = %d" (evalArith e2)
printfn "20 - 8 = %d" (evalArith e3)
```

```fsharp
// Evaluator ที่รองรับ variables
let evalWithEnv (env: Map<string, int>) (expr: Expr) : int =
    let rec eval e =
        match e with
        | Int32(n) -> n
        | Var(v) ->
            match env |> Map.tryFind v.Name with
            | Some n -> n
            | None -> failwithf "ตัวแปร %s ไม่พบ" v.Name
        | SpecificCall <@@ (+) @@> (_, _, [l; r]) -> eval l + eval r
        | SpecificCall <@@ (-) @@> (_, _, [l; r]) -> eval l - eval r
        | SpecificCall <@@ (*) @@> (_, _, [l; r]) -> eval l * eval r
        | Let(var, value, body) ->
            let varValue = eval value
            eval body  // Note: ในตัวอย่างนี้ไม่ได้ handle Let properly
        | _ -> failwith "ไม่รองรับ"
    eval expr

let env = Map.ofList [("x", 10); ("y", 20)]
// ตัวอย่าง: evalWithEnv env <@@ x + y @@>
printfn "Eval with env: simple demo"
```

## 8. Reflection.FSharpValue

```fsharp
open Microsoft.FSharp.Reflection

// FSharpValue สำหรับ reflection บน F# types
let reflectionExample () =
    // Union case info
    type Shape =
        | Circle of radius: float
        | Rectangle of width: float * height: float
    
    let circle = Circle 5.0
    let rect = Rectangle(3.0, 4.0)
    
    // ตรวจสอบ union case
    let caseInfo, values = FSharpValue.GetUnionFields(circle, typeof<Shape>)
    printfn "Case: %s" caseInfo.Name
    printfn "Values: %A" values
    
    // สร้าง union case
    let circleType = FSharpType.GetUnionCases(typeof<Shape>) |> Array.head
    let newCircle = FSharpValue.MakeUnion(circleType, [| 10.0 |])
    printfn "New circle: %A" newCircle
    
    // Record reflection
    type Point = { X: float; Y: float }
    let point = { X = 1.0; Y = 2.0 }
    
    let fields = FSharpType.GetRecordFields(typeof<Point>)
    for field in fields do
        let value = FSharpValue.GetRecordField(point, field)
        printfn "%s = %A" field.Name value

reflectionExample ()
```

```fsharp
// Generic reflection
let genericReflection () =
    // ตรวจสอบ type parameters
    let listType = typeof<int list>
    printfn "Is list: %b" (FSharpType.IsList listType)
    
    let optionType = typeof<string option>
    printfn "Is option: %b" (listType.IsGenericType)
    
    // สร้าง value ด้วย reflection
    let makeList (items: obj[]) =
        let listConsType = typeof<int list>.Assembly.GetType("Microsoft.FSharp.Collections.FSharpList`1")
        // simplified example
        printfn "Making list with %d items" items.Length

genericReflection ()
```

## 9. Use Cases: Query Translation

```fsharp
// แปลง F# quotation เป็น SQL (simplified)
// นี่คือพื้นฐานของ query providers เช่น FSharp.Data.SqlClient

type SqlQuery =
    | Select of table: string * columns: string list
    | Where of SqlQuery * condition: string
    | OrderBy of SqlQuery * column: string * desc: bool
    | Limit of SqlQuery * count: int

// แปลง simple filter expression เป็น SQL condition
let rec exprToSql (expr: Expr) : string =
    match expr with
    | Value(v, _) ->
        match v with
        | :? int as n -> string n
        | :? string as s -> sprintf "'%s'" s
        | :? bool as b -> if b then "1" else "0"
        | v -> sprintf "'%A'" v
    
    | SpecificCall <@@ (>) @@> (_, _, [lhs; rhs]) ->
        sprintf "%s > %s" (exprToSql lhs) (exprToSql rhs)
    
    | SpecificCall <@@ (<) @@> (_, _, [lhs; rhs]) ->
        sprintf "%s < %s" (exprToSql lhs) (exprToSql rhs)
    
    | SpecificCall <@@ (=) @@> (_, _, [lhs; rhs]) ->
        sprintf "%s = %s" (exprToSql lhs) (exprToSql rhs)
    
    | SpecificCall <@@ (&&) @@> (_, _, [lhs; rhs]) ->
        sprintf "(%s) AND (%s)" (exprToSql lhs) (exprToSql rhs)
    
    | SpecificCall <@@ (||) @@> (_, _, [lhs; rhs]) ->
        sprintf "(%s) OR (%s)" (exprToSql lhs) (exprToSql rhs)
    
    | Call(Some obj, mi, args) ->
        match mi.Name with
        | "get_Name" -> "name"
        | "get_Age" -> "age"
        | _ -> sprintf "/* unknown: %s */" mi.Name
    
    | PropertyGet(Some obj, pi, []) ->
        pi.Name.ToLower()
    
    | _ ->
        sprintf "/* expression */"

// ทดสอบ
let condition1 = <@@ 30 > 25 @@>
let condition2 = <@@ true && false @@>

printfn "SQL condition 1: %s" (exprToSql condition1)
printfn "SQL condition 2: %s" (exprToSql condition2)
```

```fsharp
// Query provider concept
type QueryBuilder() =
    let mutable conditions: string list = []
    let mutable orderBy: (string * bool) option = None
    let mutable limitCount: int option = None
    
    member _.Filter (condition: string) =
        conditions <- condition :: conditions
        ()
    
    member _.ToSql (tableName: string) (columns: string list) =
        let select = sprintf "SELECT %s FROM %s" (String.concat ", " columns) tableName
        let where = 
            if conditions.IsEmpty then ""
            else sprintf " WHERE %s" (conditions |> List.rev |> String.concat " AND ")
        let order = 
            match orderBy with
            | Some (col, desc) -> sprintf " ORDER BY %s %s" col (if desc then "DESC" else "ASC")
            | None -> ""
        let limit =
            match limitCount with
            | Some n -> sprintf " LIMIT %d" n
            | None -> ""
        select + where + order + limit

let qb = QueryBuilder()
qb.Filter "age > 18"
qb.Filter "name IS NOT NULL"
printfn "SQL: %s" (qb.ToSql "users" ["id"; "name"; "age"])
```

## 10. LINQ Expression Trees Comparison

```fsharp
(*
เปรียบเทียบ F# Quotations กับ C# Expression Trees:

| คุณสมบัติ           | F# Quotations          | C# Expression Trees     |
|--------------------|-----------------------|------------------------|
| Syntax             | <@ expr @>            | Expression<Func<...>>  |
| Typed              | ใช่                   | ใช่                     |
| Untyped            | <@@ expr @@>          | Expression (non-generic)|
| First-class        | ใช่                   | ต้องผ่าน lambda         |
| Splicing           | %%                    | ไม่รองรับ              |
| Pattern matching   | built-in              | ไม่รองรับ natively      |
| Evaluation         | ต้องมี library        | Compile() method       |
| Use in LINQ        | FSharp.Linq           | native                 |
*)

// ตัวอย่าง: แปลง quotation เป็น LINQ Expression
open System.Linq.Expressions

let toLinqExpression<'T, 'R> (quote: Expr<'T -> 'R>) : Expression<System.Func<'T, 'R>> =
    // ต้องใช้ FSharp.Linq.RuntimeHelpers
    // LeafExpressionConverter.QuotationToExpression quote
    // สำหรับตัวอย่าง สร้าง manually
    let param = Expression.Parameter(typeof<'T>, "x")
    let body = Expression.Constant(Unchecked.defaultof<'R>)
    Expression.Lambda<System.Func<'T, 'R>>(body, param)

printfn "LINQ integration concept demonstrated"
```

## 11. Limitations of Quotations

```fsharp
(*
ข้อจำกัดของ F# Quotations:

1. ไม่รองรับ mutable variables ใน quotation
2. ไม่รองรับ computation expressions บางอย่าง
3. ต้องการ [<ReflectedDefinition>] attribute สำหรับ external methods
4. Performance overhead ใน runtime
5. ไม่รองรับ .NET dynamic types
6. ขนาดของ quotation จำกัด
7. ไม่รองรับ byref parameters
*)

// [<ReflectedDefinition>] ทำให้ function quotable
[<ReflectedDefinition>]
let addNumbers (x: int) (y: int) = x + y

// ดึง quotation ของ function ที่ mark ด้วย [<ReflectedDefinition>]
let functionQuote = <@ addNumbers @>
printfn "Function quote: %A" functionQuote
```

```fsharp
// ข้อจำกัด: ไม่สามารถ quote บาง syntax
let limitations () =
    // ✅ สามารถ quote ได้
    let q1 = <@ 1 + 2 @>
    let q2 = <@ fun x -> x * 2 @>
    let q3 = <@ [1; 2; 3] @>
    
    // ❌ ไม่สามารถ quote ได้โดยตรง:
    // - mutable bindings ใน let
    // - try/with ใน quotation
    // - some F# specific features
    
    printfn "q1: %A" q1
    printfn "q2: %A" q2
    printfn "q3: %A" q3

limitations ()
```

## 12. ตัวอย่างครบ: Expression Simplifier

```fsharp
// Simplifier สำหรับ arithmetic expressions
let rec simplify (expr: Expr) : Expr =
    match expr with
    // Simplify 0 + x = x
    | SpecificCall <@@ (+) @@> (_, _, [Int32(0); rhs]) ->
        simplify rhs
    
    // Simplify x + 0 = x
    | SpecificCall <@@ (+) @@> (_, _, [lhs; Int32(0)]) ->
        simplify lhs
    
    // Simplify 1 * x = x
    | SpecificCall <@@ (*) @@> (_, _, [Int32(1); rhs]) ->
        simplify rhs
    
    // Simplify x * 1 = x
    | SpecificCall <@@ (*) @@> (_, _, [lhs; Int32(1)]) ->
        simplify lhs
    
    // Simplify 0 * x = 0
    | SpecificCall <@@ (*) @@> (_, _, [Int32(0); _]) ->
        Expr.Value(0)
    
    // Simplify x * 0 = 0
    | SpecificCall <@@ (*) @@> (_, _, [_; Int32(0)]) ->
        Expr.Value(0)
    
    // Constant folding: evaluate constant arithmetic
    | SpecificCall <@@ (+) @@> (_, _, [Int32(a); Int32(b)]) ->
        Expr.Value(a + b)
    
    | SpecificCall <@@ (*) @@> (_, _, [Int32(a); Int32(b)]) ->
        Expr.Value(a * b)
    
    | SpecificCall <@@ (-) @@> (_, _, [Int32(a); Int32(b)]) ->
        Expr.Value(a - b)
    
    // ไม่เปลี่ยนแปลง
    | _ -> expr

// ทดสอบ
let e1 = <@@ 0 + 5 @@>
let e2 = <@@ 5 + 0 @@>
let e3 = <@@ 1 * 7 @@>
let e4 = <@@ 0 * 100 @@>
let e5 = <@@ 3 + 4 @@>
let e6 = <@@ 2 * 6 @@>

printfn "0 + 5 -> %A" (simplify e1)
printfn "5 + 0 -> %A" (simplify e2)
printfn "1 * 7 -> %A" (simplify e3)
printfn "0 * 100 -> %A" (simplify e4)
printfn "3 + 4 -> %A" (simplify e5)
printfn "2 * 6 -> %A" (simplify e6)
```

## 13. Quotation สำหรับ Code Generation

```fsharp
// ใช้ quotation สำหรับ generating code
let generateCode (expr: Expr) : string =
    let sb = System.Text.StringBuilder()
    
    let rec gen indent (e: Expr) =
        let prefix = String.replicate indent "  "
        match e with
        | Value(v, _) -> sb.Append(sprintf "%A" v) |> ignore
        | Var(v) -> sb.Append(v.Name) |> ignore
        | Lambda(param, body) ->
            sb.Append(sprintf "fun %s -> " param.Name) |> ignore
            gen indent body
        | SpecificCall <@@ (+) @@> (_, _, [l; r]) ->
            gen indent l
            sb.Append(" + ") |> ignore
            gen indent r
        | SpecificCall <@@ (*) @@> (_, _, [l; r]) ->
            sb.Append("(") |> ignore
            gen indent l
            sb.Append(") * (") |> ignore
            gen indent r
            sb.Append(")") |> ignore
        | Let(var, value, body) ->
            sb.AppendLine() |> ignore
            sb.Append(sprintf "%slet %s = " prefix var.Name) |> ignore
            gen indent value
            sb.AppendLine() |> ignore
            sb.Append(sprintf "%sin " prefix) |> ignore
            gen indent body
        | IfThenElse(cond, thenBr, elseBr) ->
            sb.Append("if ") |> ignore
            gen indent cond
            sb.Append(" then ") |> ignore
            gen indent thenBr
            sb.Append(" else ") |> ignore
            gen indent elseBr
        | _ ->
            sb.Append(sprintf "/* %s */" (e.GetType().Name)) |> ignore
    
    gen 0 expr
    sb.ToString()

// ทดสอบ
let code1 = <@@ fun (x: int) -> x * x + 1 @@>
let code2 = <@@ fun (x: int) -> if x > 0 then x else -x @@>

printfn "Code 1: %s" (generateCode code1)
printfn "Code 2: %s" (generateCode code2)
```

## 14. Quotation Splicing (Metaprogramming)

```fsharp
// Metaprogramming ด้วย quotation splicing
let makeAdder (n: int) : Expr<int -> int> =
    <@ fun x -> x + n @>

let add5 = makeAdder 5
let add10 = makeAdder 10

printfn "add5: %A" add5
printfn "add10: %A" add10

// Compose expressions
let compose (f: Expr<'a -> 'b>) (g: Expr<'b -> 'c>) : Expr<'a -> 'c> =
    <@ fun x -> (%g) ((%f) x) @>

let double: Expr<int -> int> = <@ fun x -> x * 2 @>
let addOne: Expr<int -> int> = <@ fun x -> x + 1 @>

let doubleAndAdd = compose double addOne
printfn "doubleAndAdd: %A" doubleAndAdd
```

## 15. ตัวอย่างการใช้งาน: Type-safe ORM

```fsharp
// Type-safe query builder ที่ใช้ quotations
type QueryExpr<'T> = {
    Table: string
    Conditions: string list
    OrderBy: string option
    Limit: int option
}

let emptyQuery<'T> (table: string) = {
    Table = table
    Conditions = []
    OrderBy = None
    Limit = None
}

// ใช้ quotation เพื่อสร้าง type-safe conditions
// (ในระบบจริงจะใช้ FSharp.Quotations.Evaluator)
let exprToCondition<'T> (selector: Expr<'T -> bool>) =
    // แปลง quotation เป็น SQL string
    let rec toSql e =
        match e with
        | SpecificCall <@@ (>) @@> (_, _, [prop; value]) ->
            sprintf "%s > %s" (toSql prop) (toSql value)
        | SpecificCall <@@ (<) @@> (_, _, [prop; value]) ->
            sprintf "%s < %s" (toSql prop) (toSql value)
        | PropertyGet(_, pi, _) -> pi.Name.ToLower()
        | Value(v, _) -> sprintf "%A" v
        | _ -> "..."
    
    toSql selector.Raw

// สร้าง query builder API
type TypedQuery<'T>(table: string) =
    let mutable query = emptyQuery<'T> table
    
    member _.Where(condition: string) =
        query <- { query with Conditions = condition :: query.Conditions }
        query
    
    member _.ToSql() =
        let select = sprintf "SELECT * FROM %s" query.Table
        let where =
            if query.Conditions.IsEmpty then ""
            else " WHERE " + (query.Conditions |> List.rev |> String.concat " AND ")
        let order = query.OrderBy |> Option.map (sprintf " ORDER BY %s") |> Option.defaultValue ""
        let limit = query.Limit |> Option.map (sprintf " LIMIT %d") |> Option.defaultValue ""
        select + where + order + limit

// ใช้งาน
type User = { Id: int; Name: string; Age: int }

let userQuery = TypedQuery<User>("users")
userQuery.Where("age > 18") |> ignore
userQuery.Where("name IS NOT NULL") |> ignore

printfn "SQL: %s" (userQuery.ToSql())
```

## สรุป

```fsharp
(*
F# Quotations - สรุป:

Syntax:
- <@ expr @>     - typed quotation (Expr<T>)
- <@@ expr @@>   - untyped quotation (Expr)
- %expr          - typed splice
- %%expr         - untyped splice

Expr Types (จาก Patterns module):
- Value          - constant value
- Var            - variable reference
- Lambda         - anonymous function
- Application    - function application
- Let            - let binding
- Call           - method/function call
- PropertyGet    - property access
- IfThenElse     - if expression
- Sequential     - sequence of expressions
- NewObject      - object creation
- TupleGet       - tuple access

Use Cases:
- Query translation (ORM, LINQ)
- Code generation
- Compile-time computation
- DSL creation
- Metaprogramming
- Type providers

ข้อจำกัด:
- Performance overhead
- ไม่รองรับทุก syntax
- ต้องการ [<ReflectedDefinition>] สำหรับ external methods
- Evaluation ต้องมี library

Libraries:
- FSharp.Quotations (built-in)
- FSharp.Quotations.Evaluator (evaluation)
- FSharp.Linq.RuntimeHelpers (LINQ integration)
*)

printfn "Quotations - สรุปเสร็จ!"
```
