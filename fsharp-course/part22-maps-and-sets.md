# Part 22 - แมปและเซต (Maps and Sets)

## บทนำ (Introduction)

F# มีโครงสร้างข้อมูล immutable สำหรับ key-value pairs และ collections แบบไม่ซ้ำ:
- **Map<'K,'V>** - immutable dictionary ที่ sorted by key
- **Set<'T>** - immutable set ที่ sorted
- **dict** - mutable shorthand สำหรับ IDictionary
- **Dictionary<K,V>** - .NET mutable dictionary
- **HashSet<T>** - .NET mutable hash set

---

## 22.1 Map<'K,'V> - Immutable Dictionary

```fsharp
// Map เป็น immutable balanced binary tree (sorted by key)
// Time complexity: O(log n) สำหรับ insert, find, remove

// ============ Map.empty ============
let emptyMap : Map<string, int> = Map.empty
printfn "empty map: %A" emptyMap

// ============ สร้าง Map ============
// จาก list of tuples
let m1 = Map.ofList [("alice", 30); ("bob", 25); ("charlie", 35)]
printfn "m1: %A" m1

// จาก array of tuples
let m2 = Map.ofArray [| ("x", 1); ("y", 2); ("z", 3) |]
printfn "m2: %A" m2

// จาก seq
let m3 = Map.ofSeq (seq { yield ("a", 1); yield ("b", 2) })
printfn "m3: %A" m3

// ใช้ [ ] literal (ไม่มีใน F# แต่ใช้ ofList แทน)
let capitals = Map.ofList [
    ("Thailand", "Bangkok")
    ("Japan", "Tokyo")
    ("France", "Paris")
    ("UK", "London")
    ("USA", "Washington DC")
]
printfn "capitals: %A" capitals
```

---

## 22.2 Map.add, Map.remove

```fsharp
let m = Map.ofList [("a", 1); ("b", 2); ("c", 3)]

// ============ Map.add ============
// เพิ่ม key-value (ถ้า key มีอยู่แล้ว จะ replace)
let m2 = Map.add "d" 4 m
let m3 = Map.add "a" 100 m  // replace existing

printfn "original: %A" m    // a=1, b=2, c=3 (ไม่เปลี่ยน)
printfn "added d: %A" m2    // a=1, b=2, c=3, d=4
printfn "replaced a: %A" m3 // a=100, b=2, c=3

// Chaining adds
let populated = 
    Map.empty
    |> Map.add "name" "Alice"
    |> Map.add "city" "Bangkok"
    |> Map.add "job" "Developer"

printfn "populated: %A" populated

// ============ Map.remove ============
let m4 = Map.remove "b" m
printfn "removed b: %A" m4   // a=1, c=3

// Remove key ที่ไม่มี (ไม่ error)
let m5 = Map.remove "z" m
printfn "removed nonexistent: %A" m5  // เหมือนเดิม

// ============ การสร้าง Map จาก transformation ============
type Employee = { Id: int; Name: string; Salary: float }

let employees = [
    { Id = 1; Name = "Alice"; Salary = 50000.0 }
    { Id = 2; Name = "Bob"; Salary = 45000.0 }
    { Id = 3; Name = "Charlie"; Salary = 60000.0 }
]

// สร้าง map indexed by Id
let empById = 
    employees
    |> List.map (fun e -> (e.Id, e))
    |> Map.ofList

printfn "Employee 2: %A" empById.[2]

// สร้าง map indexed by Name
let empByName = 
    employees
    |> List.map (fun e -> (e.Name, e))
    |> Map.ofList

printfn "Employee Alice: %A" empByName.["Alice"]
```

---

## 22.3 Map.find, Map.tryFind

```fsharp
let m = Map.ofList [("a", 1); ("b", 2); ("c", 3)]

// ============ Map.find ============
// ค้นหา key (throws KeyNotFoundException ถ้าไม่มี)
let val1 = Map.find "a" m   // 1
printfn "find 'a': %d" val1

// ใช้ indexer [] (เหมือน find)
let val2 = m.["b"]   // 2
printfn "m.[\"b\"]: %d" val2

// ============ Map.tryFind ============
// ค้นหา key แบบ safe (return Option)
let opt1 = Map.tryFind "a" m   // Some 1
let opt2 = Map.tryFind "z" m   // None

printfn "tryFind 'a': %A" opt1   // Some 1
printfn "tryFind 'z': %A" opt2   // None

// ใช้ pattern matching
match Map.tryFind "b" m with
| Some value -> printfn "Found: %d" value
| None -> printfn "Not found"

// ============ defaultValue pattern ============
let getOrDefault (key: 'k) (defaultVal: 'v) (map: Map<'k,'v>) =
    map |> Map.tryFind key |> Option.defaultValue defaultVal

let score = getOrDefault "alice" 0 (Map.ofList [("alice", 95); ("bob", 87)])
let scoreUnknown = getOrDefault "charlie" 0 (Map.ofList [("alice", 95)])
printfn "alice score: %d" score         // 95
printfn "charlie score: %d" scoreUnknown // 0

// ============ Map.findKey ============
// หา key โดยใช้ predicate
let m2 = Map.ofList [(1, "one"); (2, "two"); (3, "three"); (4, "four")]
let keyWithLong = Map.findKey (fun _ v -> v.Length > 4) m2
printfn "First key with long value: %d" keyWithLong   // 3 ("three")

// ============ Map.tryFindKey ============
let tryKeyResult = Map.tryFindKey (fun k _ -> k > 5) m2
printfn "tryFindKey > 5: %A" tryKeyResult   // None

// ============ Map.pick ============
// เหมือน tryFind แต่ return 'b option จาก key-value pair
let m3 = Map.ofList [("a", 1); ("b", 2); ("c", 3)]
let picked = Map.tryPick (fun k v -> if v > 1 then Some (k, v * 10) else None) m3
printfn "tryPick: %A" picked   // Some ("b", 20)
```

---

## 22.4 Map.containsKey

```fsharp
let m = Map.ofList [("a", 1); ("b", 2); ("c", 3)]

// ============ Map.containsKey ============
let hasA = Map.containsKey "a" m    // true
let hasZ = Map.containsKey "z" m    // false

printfn "containsKey 'a': %b" hasA
printfn "containsKey 'z': %b" hasZ

// ใช้กับ if expression
let checkKey key =
    if Map.containsKey key m then
        printfn "'%s' exists with value %d" key m.[key]
    else
        printfn "'%s' does not exist" key

checkKey "b"
checkKey "z"

// ============ Map.count ============
printfn "count: %d" (Map.count m)   // 3

// ============ Map.isEmpty ============
printfn "isEmpty (m): %b" (Map.isEmpty m)              // false
printfn "isEmpty (empty): %b" (Map.isEmpty Map.empty)  // true

// ============ Map.keys / Map.values (F# 7+) ============
// Map.keys - ได้ sequence ของ keys
let keys = m |> Map.keys |> Seq.toList
printfn "keys: %A" keys   // ["a"; "b"; "c"]

// Map.values - ได้ sequence ของ values
let values = m |> Map.values |> Seq.toList
printfn "values: %A" values   // [1; 2; 3]

// ทำ Map.toList ได้ key-value pairs
let pairs = Map.toList m
printfn "toList: %A" pairs   // [("a", 1); ("b", 2); ("c", 3)]

// Map.toArray
let arr = Map.toArray m
printfn "toArray: %A" arr

// Map.toSeq
let sq = Map.toSeq m
printfn "toSeq: %A" (Seq.toList sq)
```

---

## 22.5 Map.map, Map.filter, Map.fold

```fsharp
let prices = Map.ofList [("apple", 1.5); ("banana", 0.75); ("cherry", 3.0); ("date", 5.0)]

// ============ Map.map ============
// แปลง values
let doubled = Map.map (fun _ price -> price * 2.0) prices
printfn "doubled prices: %A" doubled

// แปลง key และ value
let labeled = Map.map (fun name price -> sprintf "%s: $%.2f" name price) prices
printfn "labeled: %A" labeled

// ============ Map.filter ============
// กรองตามเงื่อนไข
let affordable = Map.filter (fun _ price -> price <= 2.0) prices
printfn "affordable: %A" affordable

let startsWithB = Map.filter (fun name _ -> name.StartsWith("b")) prices
printfn "starts with b: %A" startsWithB

// ============ Map.fold ============
// สะสมค่าจาก left (in key order)
let totalCost = Map.fold (fun acc _ price -> acc + price) 0.0 prices
printfn "total cost: %.2f" totalCost   // 10.25

// สร้าง summary string
let summary = 
    Map.fold (fun acc name price -> 
        acc + sprintf "%s=$%.2f " name price
    ) "" prices
printfn "summary: %s" (summary.Trim())

// ============ Map.foldBack ============
// สะสมค่าจาก right (in reverse key order)
let revSummary = 
    Map.foldBack (fun name price acc -> 
        sprintf "%s=$%.2f %s" name price acc
    ) prices ""
printfn "revSummary: %s" (revSummary.Trim())

// ============ Map.iter ============
// iterate ทุก key-value pair
Map.iter (fun name price ->
    printfn "  %s: $%.2f" name price
) prices

// ============ Map.iteri (ไม่มีใน F#) ============
// ต้องใช้ Map.iter แทน (key ถูกส่งมาด้วยอยู่แล้ว)

// ============ Map.partition ============
// แยกเป็นสอง map
let (cheap, expensive) = Map.partition (fun _ price -> price < 2.0) prices
printfn "cheap: %A" cheap
printfn "expensive: %A" expensive

// ============ Map.exists ============
let hasCheap = Map.exists (fun _ price -> price < 1.0) prices
printfn "has cheap item: %b" hasCheap   // true

// ============ Map.forall ============
let allExpensive = Map.forall (fun _ price -> price > 2.0) prices
printfn "all expensive: %b" allExpensive   // false
```

---

## 22.6 Map.toList, Map.ofList, Map Merge Patterns

```fsharp
// ============ Map.toList / Map.ofList ============
let m = Map.ofList [("c", 3); ("a", 1); ("b", 2)]

// toList - ได้ list sorted by key
let lst = Map.toList m
printfn "toList (sorted): %A" lst
// Output: toList (sorted): [("a", 1); ("b", 2); ("c", 3)]

// ofList - สร้าง Map (keys ต้องไม่ซ้ำ, ถ้าซ้ำจะ override)
let fromList = Map.ofList [("x", 10); ("x", 20); ("y", 30)]  // x จะเป็น 20
printfn "ofList with dup: %A" fromList

// ============ Map Merge Patterns ============

// Pattern 1: union (เอา key ของ m2 เมื่อ conflict)
let merge (m1: Map<'k,'v>) (m2: Map<'k,'v>) =
    Map.fold (fun acc k v -> Map.add k v acc) m1 m2

let base1 = Map.ofList [("a", 1); ("b", 2); ("c", 3)]
let updates = Map.ofList [("b", 20); ("d", 4)]  // override b, add d
let merged = merge base1 updates
printfn "merged: %A" merged
// Output: merged: map [("a", 1); ("b", 20); ("c", 3); ("d", 4)]

// Pattern 2: merge with conflict resolution
let mergeWith (resolve: 'k -> 'v -> 'v -> 'v) (m1: Map<'k,'v>) (m2: Map<'k,'v>) =
    Map.fold (fun acc k v ->
        match Map.tryFind k acc with
        | Some existing -> Map.add k (resolve k existing v) acc
        | None -> Map.add k v acc
    ) m1 m2

// เอาค่าสูงสุดเมื่อ conflict
let m1 = Map.ofList [("a", 10); ("b", 5); ("c", 15)]
let m2 = Map.ofList [("b", 20); ("c", 3); ("d", 8)]
let mergedMax = mergeWith (fun _ v1 v2 -> max v1 v2) m1 m2
printfn "mergeWith max: %A" mergedMax
// a=10, b=20, c=15, d=8

// Pattern 3: intersect (เฉพาะ keys ที่มีในทั้งสอง map)
let intersect (m1: Map<'k,'v>) (m2: Map<'k,'v2>) : Map<'k,'v * 'v2> =
    Map.fold (fun acc k v ->
        match Map.tryFind k m2 with
        | Some v2 -> Map.add k (v, v2) acc
        | None -> acc
    ) Map.empty m1

let names = Map.ofList [(1, "Alice"); (2, "Bob"); (3, "Charlie")]
let scores = Map.ofList [(1, 95); (3, 87); (4, 72)]
let both = intersect names scores
printfn "intersect: %A" both
// Output: intersect: map [(1, ("Alice", 95)); (3, ("Charlie", 87))]

// Pattern 4: difference (keys ใน m1 แต่ไม่ใน m2)
let difference (m1: Map<'k,'v>) (m2: Map<'k,'v2>) : Map<'k,'v> =
    Map.filter (fun k _ -> not (Map.containsKey k m2)) m1

let diff = difference names scores
printfn "difference: %A" diff   // map [(2, "Bob")]

// Pattern 5: counting words ด้วย Map
let wordCount (text: string) =
    text.Split([|' '; ','; '.'; '!'; '?'|], System.StringSplitOptions.RemoveEmptyEntries)
    |> Array.map (fun w -> w.ToLower())
    |> Array.fold (fun (acc: Map<string, int>) word ->
        let count = acc |> Map.tryFind word |> Option.defaultValue 0
        Map.add word (count + 1) acc
    ) Map.empty

let text = "the quick brown fox jumps over the lazy dog the fox"
let wc = wordCount text
printfn "Word counts: %A" wc
```

---

## 22.7 Set<'T> - Immutable Set

```fsharp
// Set เป็น immutable balanced binary tree (sorted)
// Time complexity: O(log n) สำหรับทุก operations

// ============ Set.empty ============
let emptySet : Set<int> = Set.empty
printfn "empty set: %A" emptySet

// ============ สร้าง Set ============
// จาก list
let s1 = Set.ofList [3; 1; 4; 1; 5; 9; 2; 6; 5; 3]   // ซ้ำถูกลบออก
printfn "s1 (from list with dups): %A" s1
// Output: s1: set [1; 2; 3; 4; 5; 6; 9]

// จาก array
let s2 = Set.ofArray [| "apple"; "banana"; "apple"; "cherry" |]
printfn "s2: %A" s2

// จาก seq
let s3 = Set.ofSeq (seq { 1..5 })
printfn "s3: %A" s3

// ใช้ set literal (F# 6+)
// let s4 = set [1; 2; 3; 4; 5]   // shorthand สำหรับ Set.ofList

// ============ Set.count ============
printfn "count s1: %d" (Set.count s1)   // 7

// ============ Set.isEmpty ============
printfn "isEmpty emptySet: %b" (Set.isEmpty emptySet)   // true
printfn "isEmpty s1: %b" (Set.isEmpty s1)               // false

// ============ Minimum/Maximum ============
printfn "min s1: %d" (Set.minElement s1)   // 1
printfn "max s1: %d" (Set.maxElement s1)   // 9
```

---

## 22.8 Set.add, Set.remove, Set.contains

```fsharp
let s = Set.ofList [1; 2; 3; 4; 5]

// ============ Set.add ============
// เพิ่ม element (ถ้ามีแล้วไม่มีผล)
let s2 = Set.add 6 s
let s3 = Set.add 3 s   // ซ้ำ ไม่เพิ่ม

printfn "original: %A" s    // set [1; 2; 3; 4; 5]
printfn "added 6: %A" s2    // set [1; 2; 3; 4; 5; 6]
printfn "added 3: %A" s3    // set [1; 2; 3; 4; 5] (ไม่เปลี่ยน)

// Chaining adds
let populated = 
    Set.empty
    |> Set.add "alice"
    |> Set.add "bob"
    |> Set.add "charlie"
    |> Set.add "alice"  // ซ้ำ
printfn "populated: %A" populated   // set ["alice"; "bob"; "charlie"]

// ============ Set.remove ============
// ลบ element (ถ้าไม่มีไม่ error)
let s4 = Set.remove 3 s
let s5 = Set.remove 10 s   // ไม่มี ไม่ error

printfn "removed 3: %A" s4    // set [1; 2; 4; 5]
printfn "removed 10: %A" s5   // set [1; 2; 3; 4; 5] (ไม่เปลี่ยน)

// ============ Set.contains ============
let has3 = Set.contains 3 s    // true
let has10 = Set.contains 10 s  // false

printfn "contains 3: %b" has3
printfn "contains 10: %b" has10

// ============ ใช้ Set สำหรับ deduplication ============
let withDups = [1; 3; 2; 4; 1; 5; 2; 3; 6; 4; 7]
let unique = withDups |> Set.ofList |> Set.toList
printfn "deduplicated: %A" unique
// Output: deduplicated: [1; 2; 3; 4; 5; 6; 7]

// ============ Set เป็น sorted ============
let unsorted = Set.ofList [5; 3; 8; 1; 9; 2; 7]
let sortedList = Set.toList unsorted
printfn "sorted via set: %A" sortedList
// Output: sorted via set: [1; 2; 3; 5; 7; 8; 9]
```

---

## 22.9 Set Operations: Union, Intersect, Difference

```fsharp
let setA = Set.ofList [1; 2; 3; 4; 5]
let setB = Set.ofList [4; 5; 6; 7; 8]

// ============ Set.union ============
// รวมทุก elements จากทั้งสอง set
let union = Set.union setA setB
printfn "union: %A" union
// Output: union: set [1; 2; 3; 4; 5; 6; 7; 8]

// ============ Set.intersect ============
// เฉพาะ elements ที่มีในทั้งสอง set
let intersect = Set.intersect setA setB
printfn "intersect: %A" intersect
// Output: intersect: set [4; 5]

// ============ Set.difference ============
// elements ใน setA แต่ไม่ใน setB
let diff = Set.difference setA setB
printfn "difference A-B: %A" diff   // set [1; 2; 3]

let diff2 = Set.difference setB setA
printfn "difference B-A: %A" diff2  // set [6; 7; 8]

// ============ Set.unionMany ============
// รวมหลาย set
let sets = [ Set.ofList [1; 2; 3]; Set.ofList [4; 5; 6]; Set.ofList [7; 8; 9] ]
let allNums = Set.unionMany sets
printfn "unionMany: %A" allNums
// Output: unionMany: set [1; 2; 3; 4; 5; 6; 7; 8; 9]

// ============ Set.intersectMany ============
let sets2 = [ Set.ofList [1; 2; 3; 4]; Set.ofList [2; 3; 4; 5]; Set.ofList [3; 4; 5; 6] ]
let common = Set.intersectMany sets2
printfn "intersectMany: %A" common   // set [3; 4]

// ============ ตัวอย่างใช้งานจริง: Finding common interests ============
type Person = { Name: string; Hobbies: Set<string> }

let alice = { Name = "Alice"; Hobbies = Set.ofList ["reading"; "coding"; "hiking"; "cooking"] }
let bob = { Name = "Bob"; Hobbies = Set.ofList ["gaming"; "coding"; "hiking"; "music"] }
let charlie = { Name = "Charlie"; Hobbies = Set.ofList ["reading"; "travel"; "cooking"; "music"] }

let commonAliceBob = Set.intersect alice.Hobbies bob.Hobbies
printfn "Alice & Bob common: %A" commonAliceBob   // coding, hiking

let allHobbies = Set.unionMany [alice.Hobbies; bob.Hobbies; charlie.Hobbies]
printfn "All hobbies: %A" allHobbies

let uniqueToAlice = Set.difference alice.Hobbies (Set.union bob.Hobbies charlie.Hobbies)
printfn "Unique to Alice: %A" uniqueToAlice
```

---

## 22.10 Set.isSubset, Set.isSuperset

```fsharp
let small = Set.ofList [2; 4; 6]
let medium = Set.ofList [1; 2; 3; 4; 5; 6]
let large = Set.ofList [1; 2; 3; 4; 5; 6; 7; 8; 9; 10]

// ============ Set.isSubset ============
// เช็คว่า set A เป็น subset ของ B หรือไม่
printfn "small ⊆ medium: %b" (Set.isSubset small medium)   // true
printfn "medium ⊆ small: %b" (Set.isSubset medium small)   // false
printfn "medium ⊆ medium: %b" (Set.isSubset medium medium) // true (proper = false)

// ============ Set.isProperSubset ============
// subset แต่ไม่เท่ากัน
printfn "small ⊂ medium: %b" (Set.isProperSubset small medium)   // true
printfn "medium ⊂ medium: %b" (Set.isProperSubset medium medium) // false (same set)

// ============ Set.isSuperset ============
printfn "large ⊇ medium: %b" (Set.isSuperset large medium)   // true
printfn "medium ⊇ large: %b" (Set.isSuperset medium large)   // false

// ============ Set.isProperSuperset ============
printfn "large ⊋ medium: %b" (Set.isProperSuperset large medium) // true

// ============ ตัวอย่างใช้งานจริง ============
// ตรวจสอบ permissions
let requiredPerms = Set.ofList ["read"; "write"]
let userPerms = Set.ofList ["read"; "write"; "execute"; "admin"]

let hasAccess = Set.isSubset requiredPerms userPerms
printfn "User has required permissions: %b" hasAccess   // true

// ตรวจสอบ dependencies
let installedPackages = Set.ofList ["numpy"; "pandas"; "matplotlib"; "scipy"; "sklearn"]
let projectDeps = Set.ofList ["numpy"; "pandas"; "sklearn"]

if Set.isSubset projectDeps installedPackages then
    printfn "All dependencies are installed!"
else
    let missing = Set.difference projectDeps installedPackages
    printfn "Missing packages: %A" missing
```

---

## 22.11 Set Operations for Deduplication

```fsharp
// ============ Set สำหรับ deduplication ============

// 1. Basic dedup
let dedup (lst: 'a list) = lst |> Set.ofList |> Set.toList

let withDups = [3; 1; 4; 1; 5; 9; 2; 6; 5; 3; 5]
printfn "deduplicated: %A" (dedup withDups)
// Output: deduplicated: [1; 2; 3; 4; 5; 6; 9]

// 2. Dedup แต่ preserve order
let dedupOrdered (lst: 'a list) =
    lst |> List.fold (fun (seen, acc) x ->
        if Set.contains x seen then (seen, acc)
        else (Set.add x seen, x :: acc)
    ) (Set.empty, [])
    |> snd
    |> List.rev

let orderedDeduplicated = dedupOrdered [3; 1; 4; 1; 5; 9; 2; 6; 5; 3]
printfn "order preserved: %A" orderedDeduplicated
// Output: order preserved: [3; 1; 4; 5; 9; 2; 6]

// 3. Find duplicates
let findDuplicates (lst: 'a list) =
    lst |> List.fold (fun (seen, dups) x ->
        if Set.contains x seen then (seen, Set.add x dups)
        else (Set.add x seen, dups)
    ) (Set.empty, Set.empty)
    |> snd

let dups = findDuplicates [1; 2; 3; 2; 4; 3; 5; 1]
printfn "duplicates: %A" dups   // set [1; 2; 3]

// 4. Count unique vs total
let data = [1; 2; 3; 2; 4; 1; 5; 3; 6; 4]
let total = List.length data
let unique = data |> Set.ofList |> Set.count
printfn "Total: %d, Unique: %d, Duplicates: %d" total unique (total - unique)

// 5. Set operations on strings
let text1 = "hello world"
let text2 = "world of programming"

let words1 = text1.Split(' ') |> Set.ofArray
let words2 = text2.Split(' ') |> Set.ofArray

printfn "Common words: %A" (Set.intersect words1 words2)   // world
printfn "Only in text1: %A" (Set.difference words1 words2) // hello
printfn "Only in text2: %A" (Set.difference words2 words1) // of, programming
printfn "All unique words: %A" (Set.union words1 words2)

// 6. Set map, filter, fold
let nums = Set.ofList [1; 2; 3; 4; 5; 6; 7; 8; 9; 10]

let evens = Set.filter (fun x -> x % 2 = 0) nums
let doubled = Set.map (fun x -> x * 2) nums
let total2 = Set.fold (fun acc x -> acc + x) 0 nums

printfn "evens: %A" evens
printfn "doubled: %A" doubled
printfn "total: %d" total2   // 55
```

---

## 22.12 dict - Mutable Dictionary Shorthand

```fsharp
open System.Collections.Generic

// ============ dict ============
// dict สร้าง IDictionary<'K,'V> จาก sequence of key-value pairs
let d = dict [("a", 1); ("b", 2); ("c", 3)]

// เข้าถึง
printfn "d.[\"a\"]: %d" d.["a"]

// ตรวจสอบ key
printfn "ContainsKey \"b\": %b" (d.ContainsKey("b"))
printfn "ContainsKey \"z\": %b" (d.ContainsKey("z"))

// TryGetValue
let mutable value = 0
if d.TryGetValue("c", &value) then
    printfn "Found c = %d" value
else
    printfn "Not found"

// iterate
for KeyValue(k, v) in d do
    printfn "%s -> %d" k v

// Note: dict ที่สร้างด้วย dict function เป็น ReadOnlyDictionary จริงๆ
// ถ้า add/remove จะ throw NotSupportedException
// ต้องใช้ Dictionary<K,V> ถ้าต้องการ mutate

// ============ readOnlyDict ============
let rod = readOnlyDict [("x", 10); ("y", 20)]
printfn "readOnlyDict x: %d" rod.["x"]

// ============ การแปลงจาก Map เป็น dict ============
let map = Map.ofList [("a", 1); ("b", 2); ("c", 3)]
let mutableDict = Dictionary<string, int>(map)  // แปลงเป็น Dictionary
mutableDict.["d"] <- 4   // สามารถ mutate ได้
printfn "mutableDict count: %d" mutableDict.Count
```

---

## 22.13 Dictionary<K,V> - .NET Mutable Dictionary

```fsharp
open System.Collections.Generic

// ============ สร้าง Dictionary ============
let dict1 = Dictionary<string, int>()

// เพิ่ม entries
dict1.Add("alice", 95)
dict1.Add("bob", 87)
dict1.Add("charlie", 92)

// indexer
dict1.["diana"] <- 88   // Add หรือ Update
dict1.["alice"] <- 98   // Update existing

printfn "Alice: %d" dict1.["alice"]   // 98
printfn "Count: %d" dict1.Count       // 4

// ============ ContainsKey ============
printfn "Contains alice: %b" (dict1.ContainsKey("alice"))
printfn "Contains zach: %b" (dict1.ContainsKey("zach"))

// ============ TryGetValue ============
let mutable score = 0
if dict1.TryGetValue("bob", &score) then
    printfn "Bob's score: %d" score

// ============ Remove ============
let removed = dict1.Remove("charlie")
printfn "Removed charlie: %b" removed
printfn "Count after remove: %d" dict1.Count

// ============ Keys, Values ============
printfn "Keys: %A" (dict1.Keys |> Seq.toList)
printfn "Values: %A" (dict1.Values |> Seq.toList)

// ============ iterate ============
for kvp in dict1 do
    printfn "%s -> %d" kvp.Key kvp.Value

// หรือใช้ pattern matching
for KeyValue(name, score) in dict1 do
    printfn "%s: %d" name score

// ============ สร้างจาก list ============
let fromList = 
    [("a", 1); ("b", 2); ("c", 3)]
    |> List.map (fun (k, v) -> KeyValuePair(k, v))
    |> (fun pairs -> Dictionary<string, int>(pairs))

printfn "fromList[\"a\"]: %d" fromList.["a"]

// ============ Dictionary กับ default values ============
let getOrAdd (dict: Dictionary<'k,'v>) key defaultValue =
    match dict.TryGetValue(key) with
    | true, v -> v
    | false, _ -> 
        dict.[key] <- defaultValue
        defaultValue

let cache = Dictionary<int, int>()
let fib n = 
    let rec compute n =
        if n <= 1 then n
        else
            match cache.TryGetValue(n) with
            | true, v -> v
            | false, _ ->
                let result = compute (n-1) + compute (n-2)
                cache.[n] <- result
                result
    compute n

printfn "fib(30): %d" (fib 30)   // 832040

// ============ ConcurrentDictionary ============
open System.Collections.Concurrent

let concDict = ConcurrentDictionary<string, int>()
concDict.TryAdd("key1", 1) |> ignore
concDict.AddOrUpdate("key1", 10, fun _ old -> old + 10) |> ignore
printfn "ConcurrentDict key1: %d" concDict.["key1"]   // 11
```

---

## 22.14 HashSet<T> - .NET Mutable Hash Set

```fsharp
open System.Collections.Generic

// ============ สร้าง HashSet ============
let hs = HashSet<int>()
hs.Add(1) |> ignore
hs.Add(2) |> ignore
hs.Add(3) |> ignore
hs.Add(2) |> ignore   // ซ้ำ ไม่เพิ่ม

printfn "HashSet count: %d" hs.Count   // 3

// สร้างจาก collection
let hs2 = HashSet<string>(["apple"; "banana"; "apple"; "cherry"])
printfn "HashSet2 count: %d" hs2.Count   // 3

// ============ Contains ============
printfn "Contains 2: %b" (hs.Contains(2))   // true
printfn "Contains 5: %b" (hs.Contains(5))   // false

// ============ Remove ============
hs.Remove(2) |> ignore
printfn "After remove 2, count: %d" hs.Count   // 2

// ============ Set operations ============
let setA = HashSet<int>([1; 2; 3; 4; 5])
let setB = HashSet<int>([3; 4; 5; 6; 7])

// IntersectWith (mutates setA)
let hsForIntersect = HashSet<int>(setA)
hsForIntersect.IntersectWith(setB)
printfn "Intersect: %A" (hsForIntersect |> Seq.toList |> List.sort)

// UnionWith (mutates)
let hsForUnion = HashSet<int>(setA)
hsForUnion.UnionWith(setB)
printfn "Union: %A" (hsForUnion |> Seq.toList |> List.sort)

// ExceptWith (difference)
let hsForDiff = HashSet<int>(setA)
hsForDiff.ExceptWith(setB)
printfn "Difference A-B: %A" (hsForDiff |> Seq.toList |> List.sort)

// SymmetricExceptWith (symmetric difference: elements in either but not both)
let hsForSym = HashSet<int>(setA)
hsForSym.SymmetricExceptWith(setB)
printfn "Symmetric diff: %A" (hsForSym |> Seq.toList |> List.sort)

// IsSubsetOf / IsSupersetOf
printfn "setA IsSubset of [1..10]: %b" (setA.IsSubsetOf([1..10]))
printfn "setA IsSuperset of [1;2;3]: %b" (setA.IsSupersetOf([1;2;3]))
printfn "setA Overlaps setB: %b" (setA.Overlaps(setB))

// ============ iterate ============
for x in hs2 do
    printf "%s " x
printfn ""

// ============ HashSet เร็วกว่า Set<'T> สำหรับ lookup ============
// HashSet: O(1) average
// Set<'T>: O(log n)
```

---

## 22.15 Comparison: Map vs Dictionary vs dict

```fsharp
(*
    ============================================================
    Map<'K,'V> vs Dictionary<K,V> vs dict
    ============================================================
    
    Map<'K,'V>:
    - Immutable (functional style)
    - Sorted by key (balanced binary tree)
    - O(log n) for all operations
    - Thread-safe by default (immutable)
    - ใช้ใน functional programming
    - keys ต้อง implement IComparable
    
    Dictionary<K,V>:
    - Mutable (.NET standard)
    - Hash table (ไม่ sorted)
    - O(1) average for lookup, insert, delete
    - NOT thread-safe (ใช้ ConcurrentDictionary ถ้า concurrent)
    - ใช้เมื่อต้องการ performance สูงและ mutation
    
    dict (IDictionary<K,V>):
    - สร้าง read-only dictionary
    - ใช้สำหรับ interop หรือ passing data
    - Backed by ReadOnlyDictionary
    
    readOnlyDict:
    - Explicitly read-only
    - ชัดเจนกว่า dict
    
    ============================================================
*)

// Performance comparison example
open System.Diagnostics
open System.Collections.Generic

let n = 1_000_000
let keys = Array.init n string

// Build Map
let sw = Stopwatch()
sw.Start()
let bigMap = Array.fold (fun m k -> Map.add k (k.Length) m) Map.empty keys
sw.Stop()
printfn "Map build (1M): %d ms" sw.ElapsedMilliseconds

// Build Dictionary
sw.Restart()
let bigDict = Dictionary<string, int>()
for k in keys do
    bigDict.[k] <- k.Length
sw.Stop()
printfn "Dictionary build (1M): %d ms" sw.ElapsedMilliseconds

// Lookup Map
sw.Restart()
let mutable sum1 = 0
for k in keys.[0..999] do
    sum1 <- sum1 + Map.find k bigMap
sw.Stop()
printfn "Map lookup x1000: %d ms" sw.ElapsedMilliseconds

// Lookup Dictionary
sw.Restart()
let mutable sum2 = 0
for k in keys.[0..999] do
    sum2 <- sum2 + bigDict.[k]
sw.Stop()
printfn "Dictionary lookup x1000: %d ms" sw.ElapsedMilliseconds

// ============================================================
// เมื่อไหรใช้อะไร
// ============================================================
(*
    ใช้ Map<'K,'V> เมื่อ:
    - ต้องการ immutability
    - Functional programming style
    - ข้อมูลไม่เปลี่ยนบ่อย
    - ต้องการ thread-safety โดยไม่ต้อง lock
    - ต้องการ versioning (เก็บ snapshot หลาย version)
    
    ใช้ Dictionary<K,V> เมื่อ:
    - ต้องการ performance สูงสุด
    - มีการ add/remove/update บ่อย
    - Working with .NET APIs
    - ข้อมูลขนาดใหญ่ที่ต้องการ O(1) access
    
    ใช้ Set<'T> เมื่อ:
    - Immutable set operations
    - Functional style
    - ข้อมูลไม่เปลี่ยนบ่อย
    
    ใช้ HashSet<T> เมื่อ:
    - Mutable set operations
    - Performance สูงสุด O(1)
    - ข้อมูลใหญ่ที่ต้อง add/remove บ่อย
*)
```

---

## 22.16 Practical Examples (ตัวอย่างการใช้งานจริง)

```fsharp
open System.Collections.Generic

// ============ ตัวอย่าง 1: Frequency Counter ============
let frequencyCount (items: 'a seq) =
    items |> Seq.fold (fun (map: Map<'a, int>) item ->
        let count = map |> Map.tryFind item |> Option.defaultValue 0
        Map.add item (count + 1) map
    ) Map.empty

let nums = [1; 2; 3; 2; 1; 3; 4; 5; 1; 2]
let freq = frequencyCount nums
printfn "Frequency: %A" freq
// map [(1, 3); (2, 3); (3, 2); (4, 1); (5, 1)]

// หา element ที่เกิดบ่อยที่สุด
let mostCommon = freq |> Map.maxByValue id
// ไม่มี maxByValue ใน F# ต้องใช้ toList |> List.maxBy
let (elem, count) = freq |> Map.toList |> List.maxBy snd
printfn "Most common: %d (occurs %d times)" elem count

// ============ ตัวอย่าง 2: Graph with adjacency Map ============
type Graph = Map<int, Set<int>>

let addEdge (from: int) (to_: int) (g: Graph) : Graph =
    let neighbors = g |> Map.tryFind from |> Option.defaultValue Set.empty
    g |> Map.add from (Set.add to_ neighbors)

let addUndirectedEdge from to_ g =
    g |> addEdge from to_ |> addEdge to_ from

let emptyGraph : Graph = Map.empty

let graph = 
    emptyGraph
    |> addUndirectedEdge 0 1
    |> addUndirectedEdge 0 2
    |> addUndirectedEdge 1 3
    |> addUndirectedEdge 2 3
    |> addUndirectedEdge 3 4

printfn "Graph: %A" graph

// BFS with Map-based graph
let bfs (graph: Graph) (start: int) =
    let rec loop (queue: int list) (visited: Set<int>) (order: int list) =
        match queue with
        | [] -> List.rev order
        | node :: rest ->
            if Set.contains node visited then
                loop rest visited order
            else
                let neighbors = 
                    graph 
                    |> Map.tryFind node 
                    |> Option.defaultValue Set.empty
                    |> Set.toList
                loop (rest @ neighbors) (Set.add node visited) (node :: order)
    loop [start] Set.empty []

printfn "BFS from 0: %A" (bfs graph 0)

// ============ ตัวอย่าง 3: Memoization with Map ============
let memoize (f: 'a -> 'b) =
    let cache = Dictionary<'a, 'b>()
    fun x ->
        match cache.TryGetValue(x) with
        | true, v -> v
        | false, _ ->
            let result = f x
            cache.[x] <- result
            result

let rec expensiveFib n =
    if n <= 1 then n
    else expensiveFib (n-1) + expensiveFib (n-2)

let memoFib = 
    let cache = Dictionary<int, int64>()
    let rec fib n =
        match cache.TryGetValue(n) with
        | true, v -> v
        | false, _ ->
            let result = 
                if n <= 1 then int64 n
                else fib (n-1) + fib (n-2)
            cache.[n] <- result
            result
    fib

printfn "fib(50): %d" (memoFib 50)

// ============ ตัวอย่าง 4: Multimap ============
// Map<'k, 'v list> - key มีหลาย values

let addToMultimap (key: 'k) (value: 'v) (m: Map<'k, 'v list>) =
    let existing = m |> Map.tryFind key |> Option.defaultValue []
    Map.add key (value :: existing) m

let studentCourses =
    Map.empty
    |> addToMultimap "Alice" "Math"
    |> addToMultimap "Alice" "Science"
    |> addToMultimap "Bob" "Art"
    |> addToMultimap "Bob" "Math"
    |> addToMultimap "Alice" "History"

printfn "Alice's courses: %A" (studentCourses |> Map.tryFind "Alice")
printfn "Bob's courses: %A" (studentCourses |> Map.tryFind "Bob")

// ============ ตัวอย่าง 5: Two-pointer using Set ============
// หา pair ที่บวกกันได้ target
let findPairs (nums: int list) (target: int) =
    let seen = HashSet<int>()
    let pairs = ResizeArray<int * int>()
    for n in nums do
        let complement = target - n
        if seen.Contains(complement) then
            pairs.Add((complement, n))
        seen.Add(n) |> ignore
    pairs |> Seq.toList

let nums2 = [1; 5; 3; 7; 2; 9; 4; 6]
let pairs = findPairs nums2 10
printfn "Pairs summing to 10: %A" pairs
// Output: [(1,9); (3,7); (4,6)]
```

---

## สรุป (Summary)

```
Map<'K,'V>:
- Immutable, sorted, O(log n)
- ใช้ใน functional programming
- Thread-safe, versioning

Set<'T>:
- Immutable, sorted, O(log n)
- Union, Intersect, Difference operations
- Deduplication

Dictionary<K,V>:
- Mutable, hash-based, O(1) average
- ใช้เมื่อต้องการ performance
- Not thread-safe

HashSet<T>:
- Mutable, hash-based, O(1) average
- Set operations (IntersectWith, etc.)

dict / readOnlyDict:
- Read-only view
- ใช้สำหรับ interop
```

---

*จบ Part 22 - แมปและเซต (Maps and Sets)*
