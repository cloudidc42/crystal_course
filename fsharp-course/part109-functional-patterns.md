# Part 109 - Functional Patterns ขั้นสูง

## บทนำ (Introduction)

Functional Patterns ขั้นสูงเป็นเทคนิคและแนวคิดจาก functional programming theory ที่ช่วยให้โค้ดมีความสวยงาม ยืดหยุ่น และ composable มากขึ้น

---

## 1. Continuation Passing Style (CPS)

### แนวคิด CPS

CPS เป็น style ของการเขียนโปรแกรมที่ control flow ถูกส่งผ่านเป็น "continuation" functions แทนที่จะ return ค่ากลับโดยตรง

```fsharp
// Direct Style (ทั่วไป)
let add x y = x + y
let result = add 3 4  // = 7

// CPS Style
let addCPS x y (k: int -> 'r) = k (x + y)

addCPS 3 4 (fun result ->
    printfn "Result: %d" result)

// CPS ช่วยให้ handle async operations ได้ชัดเจน
let divCPS x y (onSuccess: int -> 'r) (onError: string -> 'r) =
    if y = 0 then onError "Division by zero"
    else onSuccess (x / y)

divCPS 10 2 (fun r -> printfn "OK: %d" r) (fun e -> printfn "Error: %s" e)
divCPS 10 0 (fun r -> printfn "OK: %d" r) (fun e -> printfn "Error: %s" e)

// ===== CPS สำหรับ Recursive Functions =====

// Factorial ธรรมดา (ไม่ tail-recursive)
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

// CPS factorial (tail-recursive)
let rec factorialCPS n (k: int -> 'r) =
    if n <= 1 then k 1
    else factorialCPS (n - 1) (fun r -> k (n * r))

let result = factorialCPS 10 id  // id = fun x -> x
printfn "10! = %d" result

// ===== CPS Pipeline =====

// เปลี่ยน pipeline ให้เป็น CPS
let parseCPS (input: string) (k: int -> 'r) (err: string -> 'r) =
    match System.Int32.TryParse(input) with
    | true, n -> k n
    | false, _ -> err (sprintf "Cannot parse '%s'" input)

let validateCPS (n: int) (k: int -> 'r) (err: string -> 'r) =
    if n >= 0 then k n
    else err "Number must be non-negative"

let squareCPS (n: int) (k: int -> 'r) =
    k (n * n)

// Chain CPS functions
let processCPS input onSuccess onError =
    parseCPS input
        (fun n ->
            validateCPS n
                (fun validated ->
                    squareCPS validated onSuccess)
                onError)
        onError

processCPS "4" (printfn "Result: %d") (printfn "Error: %s")
processCPS "-1" (printfn "Result: %d") (printfn "Error: %s")
processCPS "abc" (printfn "Result: %d") (printfn "Error: %s")
```

---

## 2. Church Encoding

### แทน Data ด้วย Functions

Church encoding เป็นวิธีการแทน data types และ operations ด้วยฟังก์ชันล้วนๆ

```fsharp
// ===== Church Booleans =====

// true = fun t f -> t
// false = fun t f -> f
type ChurchBool<'a> = 'a -> 'a -> 'a

let churchTrue : ChurchBool<'a> = fun t _ -> t
let churchFalse : ChurchBool<'a> = fun _ f -> f

let toNativeBool (cb: ChurchBool<bool>) = cb true false

let churchAnd (a: ChurchBool<'a>) (b: ChurchBool<'a>) : ChurchBool<'a> =
    fun t f -> a (b t f) f

let churchOr (a: ChurchBool<'a>) (b: ChurchBool<'a>) : ChurchBool<'a> =
    fun t f -> a t (b t f)

let churchNot (a: ChurchBool<'a>) : ChurchBool<'a> =
    fun t f -> a f t

// Test
printfn "true AND false = %b" (toNativeBool (churchAnd churchTrue churchFalse))
printfn "true OR false = %b" (toNativeBool (churchOr churchTrue churchFalse))
printfn "NOT true = %b" (toNativeBool (churchNot churchTrue))

// ===== Church Numerals =====

// 0 = fun f x -> x
// 1 = fun f x -> f x
// 2 = fun f x -> f (f x)
// n = apply f n times to x
type ChurchNum = (int -> int) -> int -> int

let zero : ChurchNum = fun _ x -> x
let one : ChurchNum = fun f x -> f x
let two : ChurchNum = fun f x -> f (f x)
let three : ChurchNum = fun f x -> f (f (f x))

let toInt (cn: ChurchNum) = cn (fun n -> n + 1) 0

let succ (cn: ChurchNum) : ChurchNum =
    fun f x -> f (cn f x)

let plus (m: ChurchNum) (n: ChurchNum) : ChurchNum =
    fun f x -> m f (n f x)

let mult (m: ChurchNum) (n: ChurchNum) : ChurchNum =
    fun f -> m (n f)

// Test
printfn "zero = %d" (toInt zero)
printfn "succ(two) = %d" (toInt (succ two))
printfn "2 + 3 = %d" (toInt (plus two three))
printfn "2 * 3 = %d" (toInt (mult two three))

// ===== Church Pairs =====

// pair(x, y) = fun f -> f x y
// fst = fun p -> p (fun x _ -> x)
// snd = fun p -> p (fun _ y -> y)

let churchPair x y = fun f -> f x y
let churchFst p = p (fun x _ -> x)
let churchSnd p = p (fun _ y -> y)

let pair = churchPair "hello" 42
printfn "fst = %s" (churchFst pair)
printfn "snd = %d" (churchSnd pair)

// ===== Church List =====

// cons = fun h t f x -> f h (t f x)
// nil = fun f x -> x

type ChurchList<'a, 'r> = ('a -> 'r -> 'r) -> 'r -> 'r

let nil : ChurchList<'a, 'r> = fun _ x -> x

let cons (h: 'a) (t: ChurchList<'a, 'r>) : ChurchList<'a, 'r> =
    fun f x -> f h (t f x)

let toNativeList (cl: ChurchList<'a, 'a list>) =
    cl (fun h t -> h :: t) []

let churchList123 = cons 1 (cons 2 (cons 3 nil))
let native = toNativeList churchList123
printfn "Church list: %A" native

let churchLength (cl: ChurchList<'a, int>) = cl (fun _ acc -> acc + 1) 0
printfn "Length: %d" (churchLength churchList123)
```

---

## 3. Y Combinator

### Fixed-Point Combinator

```fsharp
// Y Combinator ช่วยให้สร้าง recursive functions ได้โดยไม่ต้องใช้ recursive definitions

// ===== Y Combinator ใน Lazy Language (conceptual) =====
// Y = λf. (λx. f (x x)) (λx. f (x x))

// ===== Z Combinator (strict version สำหรับ F#) =====
// Z = λf. (λx. f (λv. x x v)) (λx. f (λv. x x v))

let z = fun f ->
    let inner = fun x -> f (fun v -> x x v)
    inner inner

// ใช้สร้าง factorial
let factZ = z (fun self -> fun n ->
    if n <= 1 then 1
    else n * self (n - 1))

printfn "factorial(10) = %d" (factZ 10)

// ใช้สร้าง fibonacci
let fibZ = z (fun self -> fun n ->
    if n <= 1 then n
    else self (n - 1) + self (n - 2))

printfn "fibonacci(10) = %d" (fibZ 10)

// ===== Memoized Y Combinator =====

let memoZ f =
    let cache = System.Collections.Generic.Dictionary<int, int>()
    
    let rec memoRec n =
        match cache.TryGetValue(n) with
        | true, v -> v
        | false, _ ->
            let v = f memoRec n
            cache.[n] <- v
            v
    
    memoRec

let fibMemo = memoZ (fun self n ->
    if n <= 1 then n
    else self (n - 1) + self (n - 2))

// เร็วกว่ามากสำหรับ large n
printfn "fibonacci(40) = %d" (fibMemo 40)
```

---

## 4. Fixed-Point Combinators

```fsharp
// ===== Fixed Point =====

// fix f = x where f x = x

// สำหรับ functions: fix f = f (fix f) = f (f (fix f)) = ...
// ใช้สำหรับสร้าง recursive structures

let fix<'a, 'b> (f: ('a -> 'b) -> 'a -> 'b) : 'a -> 'b =
    let rec g x = f g x
    g

// Factorial ผ่าน fix
let factorial = fix (fun self n ->
    if n <= 1 then 1
    else n * self (n - 1))

printfn "5! = %d" (factorial 5)

// ===== Fix สำหรับ Type =====

// สร้าง recursive type ที่ infinite depth
type FixType<'f> = Unfix of (FixType<'f> -> 'f)

let fix' (f: 'a -> 'a) : 'a =
    let inner = Unfix (fun x ->
        let (Unfix g) = x
        f (g x))
    let (Unfix g) = inner
    g inner

// ===== Corecursion =====

// Infinite streams
type Stream<'a> =
    | Stream of 'a * Lazy<Stream<'a>>

let head (Stream (h, _)) = h
let tail (Stream (_, t)) = t.Value

let unfold (seed: 'S) (f: 'S -> 'A * 'S) : Stream<'A> =
    let rec go s = 
        let (a, s') = f s
        Stream (a, lazy go s')
    go seed

// Fibonacci stream
let fibStream = unfold (0, 1) (fun (a, b) -> (a, (b, a + b)))

// Take n elements
let rec take n (stream: Stream<'a>) =
    if n = 0 then []
    else head stream :: take (n - 1) (tail stream)

printfn "First 10 Fibonacci: %A" (take 10 fibStream)

// Natural numbers stream
let nats = unfold 0 (fun n -> (n, n + 1))
printfn "First 10 naturals: %A" (take 10 nats)
```

---

## 5. Defunctionalization

### แปลง Higher-Order Functions เป็น First-Order

```fsharp
// ===== Original Higher-Order Program =====

// ปัญหา: higher-order functions ยากต่อการ serialize หรือ transmit

let applyTwice f x = f (f x)
let addThree = applyTwice (fun x -> x + 3)  // ไม่ serializable!

// ===== Defunctionalized Version =====

// แทน function ด้วย data type ที่ describe operation
type Transform =
    | Add of int
    | Multiply of int
    | Negate
    | Compose of Transform * Transform

// Interpreter สำหรับ Transform
let rec applyTransform (t: Transform) (x: int) =
    match t with
    | Add n -> x + n
    | Multiply n -> x * n
    | Negate -> -x
    | Compose (t1, t2) -> applyTransform t2 (applyTransform t1 x)

// ใช้งาน
let myTransform = Compose(Add 3, Multiply 2)  // (x + 3) * 2
printfn "Transform 5: %d" (applyTransform myTransform 5)  // = 16

// Serializable!
let json = System.Text.Json.JsonSerializer.Serialize(myTransform)
printfn "Serialized: %s" json

let deserialized = System.Text.Json.JsonSerializer.Deserialize<Transform>(json)
printfn "After deserialize: %d" (applyTransform deserialized 5)

// ===== Defunctionalized Continuations =====

// แปลง CPS ให้เป็น data-driven

type Continuation =
    | Done
    | FactStep of n: int * k: Continuation
    | PrintResult of Continuation

let runContinuation (start: int) =
    let rec apply cont acc =
        match cont with
        | Done -> acc
        | FactStep (n, k) -> apply k (n * acc)
        | PrintResult k ->
            printfn "Factorial result: %d" acc
            apply k acc
    
    let rec buildCont n acc =
        if n <= 1 then acc
        else buildCont (n - 1) (FactStep (n, acc))
    
    let cont = buildCont start (PrintResult Done)
    apply cont 1

runContinuation 10
```

---

## 6. Tagless Final Encoding

### Type-safe DSL ที่ Extensible

```fsharp
// ===== Object Algebra / Tagless Final =====

// แทนที่จะใช้ Algebraic Data Type เราใช้ type class (interface ใน F#)

// Algebra interface สำหรับ arithmetic expressions
type IArith<'e> = {
    Lit: int -> 'e
    Add: 'e -> 'e -> 'e
    Mul: 'e -> 'e -> 'e
    Neg: 'e -> 'e
}

// Interpreter ที่ evaluate เป็น int
let evalInterp : IArith<int> = {
    Lit = fun n -> n
    Add = fun a b -> a + b
    Mul = fun a b -> a * b
    Neg = fun n -> -n
}

// Interpreter ที่ print expression
let printInterp : IArith<string> = {
    Lit = fun n -> string n
    Add = fun a b -> sprintf "(%s + %s)" a b
    Mul = fun a b -> sprintf "(%s * %s)" a b
    Neg = fun n -> sprintf "(-  %s)" n
}

// สร้าง expression ผ่าน interpreter
let expression interp =
    let { Lit = lit; Add = add; Mul = mul; Neg = neg } = interp
    // (3 + 4) * (-(5))
    mul (add (lit 3) (lit 4)) (neg (lit 5))

// ใช้งานกับ interpreters ต่างๆ
printfn "Evaluated: %d" (expression evalInterp)
printfn "Printed: %s" (expression printInterp)

// ===== Extensible =====

// เพิ่ม operations ใหม่โดยไม่ต้องแก้ existing code
type IArithWithIf<'e> = {
    Arith: IArith<'e>
    IfPos: 'e -> 'e -> 'e -> 'e  // if positive then ... else ...
}

let evalWithIf : IArithWithIf<int> = {
    Arith = evalInterp
    IfPos = fun cond t f -> if cond > 0 then t else f
}

let printWithIf : IArithWithIf<string> = {
    Arith = printInterp
    IfPos = fun cond t f -> sprintf "(if %s > 0 then %s else %s)" cond t f
}

let expressionWithIf interp =
    let { Arith = { Lit = lit; Add = add; Neg = neg }; IfPos = ifPos } = interp
    // if -3 > 0 then 1 else -1
    ifPos (neg (lit 3)) (lit 1) (neg (lit 1))

printfn "With if (eval): %d" (expressionWithIf evalWithIf)
printfn "With if (print): %s" (expressionWithIf printWithIf)
```

---

## 7. Data-Directed Programming

### Dispatch ตาม Data Type

```fsharp
// ===== Open Dispatch Table =====

// แทนที่จะใช้ pattern matching แบบตายตัว
// ใช้ dictionary สำหรับ dispatch

type Serializer = {
    Serialize: obj -> string
    Deserialize: string -> obj
}

let serializerRegistry = 
    System.Collections.Generic.Dictionary<System.Type, Serializer>()

let registerSerializer<'T> (serialize: 'T -> string) (deserialize: string -> 'T) =
    let serializer = {
        Serialize = fun obj -> serialize (obj :?> 'T)
        Deserialize = fun str -> deserialize str :> obj
    }
    serializerRegistry.[typeof<'T>] <- serializer

let serialize<'T> (value: 'T) =
    match serializerRegistry.TryGetValue(typeof<'T>) with
    | true, s -> s.Serialize value
    | false, _ -> 
        System.Text.Json.JsonSerializer.Serialize(value)

// Register serializers
registerSerializer<int> string System.Int32.Parse
registerSerializer<float> (sprintf "%g") System.Double.Parse
registerSerializer<string> (sprintf "\"%s\"") (fun s -> s.Trim('"'))

printfn "%s" (serialize 42)
printfn "%s" (serialize 3.14)
printfn "%s" (serialize "hello")

// ===== Visitor Pattern with Dispatch =====

type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float

type ShapeOperation<'T> = {
    OnCircle: float -> 'T
    OnRectangle: float -> float -> 'T
    OnTriangle: float -> float -> 'T
}

let processShape<'T> (op: ShapeOperation<'T>) (shape: Shape) : 'T =
    match shape with
    | Circle r -> op.OnCircle r
    | Rectangle (w, h) -> op.OnRectangle w h
    | Triangle (b, h) -> op.OnTriangle b h

let areaOp = {
    OnCircle = fun r -> System.Math.PI * r * r
    OnRectangle = fun w h -> w * h
    OnTriangle = fun b h -> 0.5 * b * h
}

let perimeterOp = {
    OnCircle = fun r -> 2.0 * System.Math.PI * r
    OnRectangle = fun w h -> 2.0 * (w + h)
    OnTriangle = fun b h -> 
        // สมมติว่าเป็น isoceles triangle
        b + 2.0 * sqrt((b/2.0)**2.0 + h**2.0)
}

let shapes = [Circle 5.0; Rectangle 4.0 6.0; Triangle 3.0 4.0]

printfn "\nShape Areas:"
for shape in shapes do
    printfn "  %A: %.2f" shape (processShape areaOp shape)

printfn "\nShape Perimeters:"
for shape in shapes do
    printfn "  %A: %.2f" shape (processShape perimeterOp shape)
```

---

## 8. Generic Programming

### Structural Polymorphism

```fsharp
// ===== Generic Functor =====

// Functor = Type constructor ที่มี map
type IFunctor<'F> =
    abstract member Map<'A, 'B> : ('A -> 'B) -> 'F -> 'F

// ===== Free Monad =====

type Free<'F, 'A> =
    | Pure of 'A
    | Impure of 'F * (obj -> Free<'F, 'A>)

// ===== Existentials =====

type SomeList = SomeList : 'a list * ('a -> string) -> SomeList

let lists = [
    SomeList ([1; 2; 3], string)
    SomeList (["a"; "b"; "c"], id)
    SomeList ([true; false; true], string)
]

for SomeList (list, show) in lists do
    printfn "[%s]" (list |> List.map show |> String.concat ", ")

// ===== Generic Fold =====

// Catamorphism: generic fold ที่ structure-preserving
type Expr =
    | Num of int
    | Var of string
    | Add of Expr * Expr
    | Mul of Expr * Expr
    | Neg of Expr

// Fold function
let rec foldExpr
    (onNum: int -> 'r)
    (onVar: string -> 'r)
    (onAdd: 'r -> 'r -> 'r)
    (onMul: 'r -> 'r -> 'r)
    (onNeg: 'r -> 'r)
    (expr: Expr) : 'r =
    
    match expr with
    | Num n -> onNum n
    | Var v -> onVar v
    | Add (l, r) -> onAdd (foldExpr onNum onVar onAdd onMul onNeg l) (foldExpr onNum onVar onAdd onMul onNeg r)
    | Mul (l, r) -> onMul (foldExpr onNum onVar onAdd onMul onNeg l) (foldExpr onNum onVar onAdd onMul onNeg r)
    | Neg e -> onNeg (foldExpr onNum onVar onAdd onMul onNeg e)

// ใช้ fold สำหรับ evaluation
let evaluate (env: Map<string, int>) =
    foldExpr
        id                    // Num n -> n
        (fun v -> Map.find v env)  // Var v -> lookup
        (+)                   // Add
        (*)                   // Mul
        (~-)                  // Neg

// ใช้ fold สำหรับ pretty printing
let prettyPrint =
    foldExpr
        string
        id
        (fun l r -> sprintf "(%s + %s)" l r)
        (fun l r -> sprintf "(%s * %s)" l r)
        (fun e -> sprintf "(-  %s)" e)

// ใช้ fold สำหรับ collecting variables
let collectVars =
    foldExpr
        (fun _ -> Set.empty)
        Set.singleton
        Set.union
        Set.union
        id

// Test
let expr = Add (Mul (Num 2, Var "x"), Neg (Var "y"))
let env = Map.ofList [("x", 3); ("y", 5)]

printfn "\nExpression: %s" (prettyPrint expr)
printfn "Variables: %A" (collectVars expr)
printfn "Value (x=3, y=5): %d" (evaluate env expr)
```

---

## 9. Type Class Simulation ใน F#

```fsharp
// ===== Simulating Type Classes =====

// Eq type class
type EqInstance<'T> = {
    Equals: 'T -> 'T -> bool
    NotEquals: 'T -> 'T -> bool
}

let makeEq (eq: 'T -> 'T -> bool) : EqInstance<'T> = {
    Equals = eq
    NotEquals = fun a b -> not (eq a b)
}

// Instances
let intEq = makeEq (=)
let stringEqCI = makeEq (fun a b -> a.ToLower() = b.ToLower())

let testEq (eq: EqInstance<'T>) a b =
    printfn "%b" (eq.Equals a b)

testEq intEq 42 42
testEq intEq 42 43
testEq stringEqCI "Hello" "hello"

// ===== Ord type class =====

type OrdInstance<'T> = {
    Eq: EqInstance<'T>
    Compare: 'T -> 'T -> int
    LessThan: 'T -> 'T -> bool
    GreaterThan: 'T -> 'T -> bool
}

let makeOrd (compare: 'T -> 'T -> int) : OrdInstance<'T> = {
    Eq = makeEq (fun a b -> compare a b = 0)
    Compare = compare
    LessThan = fun a b -> compare a b < 0
    GreaterThan = fun a b -> compare a b > 0
}

let intOrd = makeOrd compare
let stringOrdCI = makeOrd (fun a b -> System.String.Compare(a, b, ignoreCase = true))

// Sort using Ord instance
let sortWith (ord: OrdInstance<'T>) (list: 'T list) =
    list |> List.sortWith ord.Compare

printfn "\nSorted: %A" (sortWith intOrd [5; 3; 1; 4; 2])
printfn "Sorted (CI): %A" (sortWith stringOrdCI ["Banana"; "apple"; "Cherry"])

// ===== Functor type class =====

type FunctorInstance<'F, 'A, 'B> = {
    Map: ('A -> 'B) -> 'F -> 'F
}

// ===== Monoid type class =====

type MonoidInstance<'T> = {
    Empty: 'T
    Combine: 'T -> 'T -> 'T
}

let intSum = { Empty = 0; Combine = (+) }
let intProduct = { Empty = 1; Combine = (*) }
let listMonoid<'A> : MonoidInstance<'A list> = { Empty = []; Combine = (@) }
let stringMonoid = { Empty = ""; Combine = (+) }

// fold using Monoid
let mconcat (m: MonoidInstance<'T>) (xs: 'T list) =
    xs |> List.fold m.Combine m.Empty

printfn "\nSum: %d" (mconcat intSum [1; 2; 3; 4; 5])
printfn "Product: %d" (mconcat intProduct [1; 2; 3; 4; 5])
printfn "Concat: %s" (mconcat stringMonoid ["Hello"; " "; "World"])
printfn "List: %A" (mconcat listMonoid [[1;2]; [3;4]; [5;6]])

// ===== Foldable type class =====

type FoldableInstance<'F, 'A> = {
    FoldrWith: ('A -> 'B -> 'B) -> 'B -> 'F -> 'B
}

let listFoldable<'A> : FoldableInstance<'A list, 'A> = {
    FoldrWith = fun f z xs -> List.foldBack f xs z
}

let optionFoldable<'A> : FoldableInstance<'A option, 'A> = {
    FoldrWith = fun f z opt ->
        match opt with
        | None -> z
        | Some x -> f x z
}

// Generic operations using Foldable
let toList (fold: FoldableInstance<'F, 'A>) (fa: 'F) : 'A list =
    fold.FoldrWith (fun x acc -> x :: acc) [] fa

let length (fold: FoldableInstance<'F, 'A>) (fa: 'F) : int =
    fold.FoldrWith (fun _ acc -> acc + 1) 0 fa

let sum (fold: FoldableInstance<'F, int>) (fa: 'F) : int =
    fold.FoldrWith (+) 0 fa

printfn "\nList as foldable:"
printfn "  toList: %A" (toList listFoldable [1;2;3])
printfn "  length: %d" (length listFoldable [1;2;3;4;5])
printfn "  sum: %d" (sum listFoldable [1;2;3;4;5])

printfn "\nOption as foldable:"
printfn "  toList Some: %A" (toList optionFoldable (Some 42))
printfn "  toList None: %A" (toList optionFoldable None)
```

---

## 10. Lenses

### Composable Data Access

```fsharp
// ===== Basic Lens =====

type Lens<'S, 'A> = {
    Get: 'S -> 'A
    Set: 'A -> 'S -> 'S
}

module Lens =
    let get (lens: Lens<'S, 'A>) (s: 'S) = lens.Get s
    let set (lens: Lens<'S, 'A>) (a: 'A) (s: 'S) = lens.Set a s
    let over (lens: Lens<'S, 'A>) (f: 'A -> 'A) (s: 'S) = s |> lens.Set (lens.Get s |> f)
    
    // Compose lenses
    let compose (outer: Lens<'S, 'A>) (inner: Lens<'A, 'B>) : Lens<'S, 'B> = {
        Get = fun s -> s |> outer.Get |> inner.Get
        Set = fun b s -> 
            let a = outer.Get s
            let newA = inner.Set b a
            outer.Set newA s
    }

// ===== Types =====

type Address = {
    Street: string
    City: string
    PostalCode: string
}

type Person = {
    Name: string
    Age: int
    Address: Address
}

// ===== Lenses สำหรับ Person =====

let nameLens : Lens<Person, string> = {
    Get = fun p -> p.Name
    Set = fun name p -> { p with Name = name }
}

let ageLens : Lens<Person, int> = {
    Get = fun p -> p.Age
    Set = fun age p -> { p with Age = age }
}

let addressLens : Lens<Person, Address> = {
    Get = fun p -> p.Address
    Set = fun addr p -> { p with Address = addr }
}

let cityLens : Lens<Address, string> = {
    Get = fun a -> a.City
    Set = fun city a -> { a with City = city }
}

// Compose: person.address.city
let personCityLens = Lens.compose addressLens cityLens

// ===== Usage =====

let person = {
    Name = "สมชาย"
    Age = 30
    Address = { Street = "123 Main St"; City = "กรุงเทพ"; PostalCode = "10110" }
}

printfn "\nOriginal: %A" person
printfn "Name: %s" (Lens.get nameLens person)
printfn "City: %s" (Lens.get personCityLens person)

let older = Lens.over ageLens (fun age -> age + 1) person
printfn "\nAfter birthday: %d" (Lens.get ageLens older)

let movedCity = Lens.set personCityLens "เชียงใหม่" person
printfn "After moving: %s" (Lens.get personCityLens movedCity)

// ===== Lens Laws =====

// Law 1: get (set a s) = a
let law1 lens s a = Lens.get lens (Lens.set lens a s) = a

// Law 2: set (get s) s = s
let law2 lens s = Lens.set lens (Lens.get lens s) s = s

// Law 3: set a (set b s) = set a s
let law3 lens s a b = Lens.set lens a (Lens.set lens b s) = Lens.set lens a s

// Verify laws
printfn "\nLens Laws for nameLens:"
printfn "Law 1: %b" (law1 nameLens person "Bob")
printfn "Law 2: %b" (law2 nameLens person)
printfn "Law 3: %b" (law3 nameLens person "Alice" "Bob")
```

---

## 11. Trampolining

### Stack-safe Recursion

```fsharp
// ===== Trampoline =====

// ป้องกัน StackOverflowException สำหรับ deep recursion

type Trampoline<'A> =
    | Done of 'A
    | Bounce of (unit -> Trampoline<'A>)

let rec runTrampoline = function
    | Done a -> a
    | Bounce f -> runTrampoline (f())

// Factorial ด้วย trampoline
let factTrampoline n =
    let rec go n acc =
        if n <= 1 then Done acc
        else Bounce (fun () -> go (n - 1) (n * acc))
    
    runTrampoline (go n 1)

printfn "\n100! first digit: %s" ((factTrampoline 100 |> string).[0..0])

// ===== Mutual Recursion กับ Trampoline =====

// isEven/isOdd ที่ recursion ลึกมาก
let rec isEvenT n =
    if n = 0 then Done true
    else Bounce (fun () -> isOddT (n - 1))

and isOddT n =
    if n = 0 then Done false
    else Bounce (fun () -> isEvenT (n - 1))

printfn "isEven(1000000): %b" (runTrampoline (isEvenT 1000000))

// ===== Continuation Trampoline =====

type ContTrampoline<'A, 'R> =
    | Return of 'R
    | Step of 'A * ('A -> ContTrampoline<'A, 'R>)

let rec runContTrampoline = function
    | Return r -> r
    | Step (a, k) -> runContTrampoline (k a)
```

---

## สรุป (Summary)

Functional Patterns ขั้นสูงที่เรียนในบทนี้:

1. **Continuation Passing Style**: Control flow เป็น explicit continuations
2. **Church Encoding**: แทน data ด้วย pure functions
3. **Y/Z Combinator**: Recursion โดยไม่ต้อง named recursion
4. **Fixed-Point**: Infinite structures และ recursive types
5. **Defunctionalization**: แปลง HOFs เป็น serializable data
6. **Tagless Final**: Extensible DSLs โดยไม่ต้อง GADTs
7. **Data-Directed Programming**: Dispatch ตาม data
8. **Generic Programming**: Structural polymorphism
9. **Type Class Simulation**: Interface-based type classes
10. **Lenses**: Composable data access
11. **Trampolining**: Stack-safe deep recursion

---

*ไปต่อที่ Part 110: Production Checklist สำหรับ F# Applications*
