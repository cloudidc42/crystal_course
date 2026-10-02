# Part 24 - ต้นไม้ (Trees)

## บทนำ (Introduction)

ต้นไม้ (Tree) เป็นโครงสร้างข้อมูลแบบ recursive ที่ประกอบด้วย node ที่มี parent-child relationships F# รองรับการนิยาม Tree ด้วย Discriminated Unions ได้อย่างสวยงาม

```fsharp
// Tree ทั่วไปใน F#
type Tree<'T> =
    | Leaf
    | Node of 'T * Tree<'T> * Tree<'T>

// ตัวอย่างสร้าง tree
let tree = 
    Node(1, 
        Node(2, 
            Node(4, Leaf, Leaf),
            Node(5, Leaf, Leaf)),
        Node(3,
            Node(6, Leaf, Leaf),
            Leaf))

printfn "Tree created: %A" tree
```

---

## 24.1 Binary Tree DU Definition

```fsharp
// ============ Binary Tree ============
type BinaryTree<'T> =
    | Empty
    | Node of value: 'T * left: BinaryTree<'T> * right: BinaryTree<'T>

// helper constructors
let leaf x = Node(x, Empty, Empty)

// ตัวอย่าง:
//       4
//      / \
//     2   6
//    / \ / \
//   1  3 5  7

let sampleTree =
    Node(4,
        Node(2,
            leaf 1,
            leaf 3),
        Node(6,
            leaf 5,
            leaf 7))

// ============ Tree Properties ============
let rec height = function
    | Empty -> 0
    | Node(_, left, right) -> 1 + max (height left) (height right)

let rec size = function
    | Empty -> 0
    | Node(_, left, right) -> 1 + size left + size right

let rec countLeaves = function
    | Empty -> 0
    | Node(_, Empty, Empty) -> 1
    | Node(_, left, right) -> countLeaves left + countLeaves right

let rec depth (node: BinaryTree<'T>) (target: 'T) =
    let rec depthAux tree d =
        match tree with
        | Empty -> -1
        | Node(v, left, right) ->
            if v = target then d
            else
                let leftDepth = depthAux left (d + 1)
                if leftDepth >= 0 then leftDepth
                else depthAux right (d + 1)
    depthAux node 0

printfn "Height: %d" (height sampleTree)     // 3
printfn "Size: %d" (size sampleTree)         // 7
printfn "Leaves: %d" (countLeaves sampleTree) // 4
printfn "Depth of 3: %d" (depth sampleTree 3) // 2
printfn "Depth of 7: %d" (depth sampleTree 7) // 2

// ============ Tree with parent tracking ============
type AnnotatedTree<'T, 'A> =
    | AEmpty
    | ANode of value: 'T * annotation: 'A * left: AnnotatedTree<'T, 'A> * right: AnnotatedTree<'T, 'A>

// Annotate with subtree size
let rec annotateSize = function
    | Empty -> (AEmpty, 0)
    | Node(v, left, right) ->
        let (al, ls) = annotateSize left
        let (ar, rs) = annotateSize right
        let total = 1 + ls + rs
        (ANode(v, total, al, ar), total)

let (annotated, _) = annotateSize sampleTree
printfn "Annotated tree created"
```

---

## 24.2 BST Operations

```fsharp
// ============ Binary Search Tree ============
// Property: left < node <= right

type BST<'T when 'T : comparison> = BinaryTree<'T>

// Insert - O(log n) average, O(n) worst
let rec bstInsert (value: 'T) (tree: BST<'T>) : BST<'T> =
    match tree with
    | Empty -> leaf value
    | Node(v, left, right) ->
        if value < v then Node(v, bstInsert value left, right)
        elif value > v then Node(v, left, bstInsert value right)
        else tree  // already exists

// Search - O(log n) average
let rec bstContains (value: 'T) (tree: BST<'T>) : bool =
    match tree with
    | Empty -> false
    | Node(v, left, right) ->
        if value = v then true
        elif value < v then bstContains value left
        else bstContains value right

// Find minimum
let rec bstMin = function
    | Empty -> failwith "Empty BST"
    | Node(v, Empty, _) -> v
    | Node(_, left, _) -> bstMin left

// Find maximum
let rec bstMax = function
    | Empty -> failwith "Empty BST"
    | Node(v, _, Empty) -> v
    | Node(_, _, right) -> bstMax right

// Delete
let rec bstDelete (value: 'T) (tree: BST<'T>) : BST<'T> =
    match tree with
    | Empty -> Empty
    | Node(v, left, right) ->
        if value < v then
            Node(v, bstDelete value left, right)
        elif value > v then
            Node(v, left, bstDelete value right)
        else
            // Found the node to delete
            match left, right with
            | Empty, _ -> right
            | _, Empty -> left
            | _ ->
                // Replace with in-order successor (min of right subtree)
                let successor = bstMin right
                Node(successor, left, bstDelete successor right)

// Build BST from list
let bstOfList lst = List.fold (fun t x -> bstInsert x t) Empty lst

// ============ ใช้งาน BST ============
let bst = bstOfList [5; 3; 7; 1; 4; 6; 8; 2]

printfn "BST built from [5;3;7;1;4;6;8;2]"
printfn "Contains 4: %b" (bstContains 4 bst)    // true
printfn "Contains 9: %b" (bstContains 9 bst)    // false
printfn "Min: %d" (bstMin bst)                   // 1
printfn "Max: %d" (bstMax bst)                   // 8

let bst2 = bstDelete 3 bst
printfn "After deleting 3, contains 3: %b" (bstContains 3 bst2)  // false
printfn "After deleting 3, contains 4: %b" (bstContains 4 bst2)  // true
```

---

## 24.3 Tree Traversal

```fsharp
// ============ Tree Traversal ============

// In-order: Left, Root, Right (ได้ค่า sorted สำหรับ BST)
let rec inorder = function
    | Empty -> []
    | Node(v, left, right) -> inorder left @ [v] @ inorder right

// Pre-order: Root, Left, Right (ใช้สำหรับ copy tree, serialize)
let rec preorder = function
    | Empty -> []
    | Node(v, left, right) -> [v] @ preorder left @ preorder right

// Post-order: Left, Right, Root (ใช้สำหรับ delete tree, calculate size)
let rec postorder = function
    | Empty -> []
    | Node(v, left, right) -> postorder left @ postorder right @ [v]

// Level-order (BFS): ระดับต่อระดับ
open System.Collections.Generic

let levelOrder (tree: BinaryTree<'T>) =
    let result = ResizeArray<'T list>()
    if tree = Empty then result
    else
        let queue = Queue<BinaryTree<'T>>()
        queue.Enqueue(tree)
        
        while queue.Count > 0 do
            let levelSize = queue.Count
            let level = ResizeArray<'T>()
            
            for _ in 1..levelSize do
                match queue.Dequeue() with
                | Empty -> ()
                | Node(v, left, right) ->
                    level.Add(v)
                    if left <> Empty then queue.Enqueue(left)
                    if right <> Empty then queue.Enqueue(right)
            
            if level.Count > 0 then
                result.Add(level |> Seq.toList)
        
        result

// ============ ใช้งาน ============
let tree =
    Node(4,
        Node(2, leaf 1, leaf 3),
        Node(6, leaf 5, leaf 7))

printfn "In-order: %A" (inorder tree)     // [1; 2; 3; 4; 5; 6; 7]
printfn "Pre-order: %A" (preorder tree)   // [4; 2; 1; 3; 6; 5; 7]
printfn "Post-order: %A" (postorder tree) // [1; 3; 2; 5; 7; 6; 4]
printfn "Level-order: %A" (levelOrder tree |> Seq.toList)
// [[4]; [2; 6]; [1; 3; 5; 7]]

// Iterative In-order (using explicit stack)
let inorderIterative (tree: BinaryTree<'T>) =
    let stack = Stack<BinaryTree<'T>>()
    let result = ResizeArray<'T>()
    let mutable current = tree
    
    while current <> Empty || stack.Count > 0 do
        // Go to leftmost node
        while current <> Empty do
            stack.Push(current)
            match current with
            | Node(_, left, _) -> current <- left
            | _ -> current <- Empty
        
        // Process current
        match stack.Pop() with
        | Node(v, _, right) ->
            result.Add(v)
            current <- right
        | _ -> ()
    
    result |> Seq.toList

printfn "Iterative In-order: %A" (inorderIterative tree)
// [1; 2; 3; 4; 5; 6; 7]

// Mirror tree
let rec mirror = function
    | Empty -> Empty
    | Node(v, left, right) -> Node(v, mirror right, mirror left)

let mirrored = mirror tree
printfn "Mirrored in-order: %A" (inorder mirrored)   // [7; 6; 5; 4; 3; 2; 1]
```

---

## 24.4 Tree Height and Depth

```fsharp
// ============ Height vs Depth ============
// Height: ความสูงจาก node ไปถึง leaf ที่ไกลสุด
// Depth: ระยะห่างจาก root ไปถึง node

// Height (already defined above, redefined for clarity)
let rec treeHeight = function
    | Empty -> -1   // -1 หมายถึง empty (บางครั้งนับ leaf = 0)
    | Node(_, left, right) -> 1 + max (treeHeight left) (treeHeight right)

// Depth of a specific node
let nodeDepth (target: 'T) (tree: BinaryTree<'T>) =
    let rec aux tree depth =
        match tree with
        | Empty -> None
        | Node(v, left, right) ->
            if v = target then Some depth
            else
                match aux left (depth + 1) with
                | Some d -> Some d
                | None -> aux right (depth + 1)
    aux tree 0

// Diameter (longest path between any two nodes)
let rec diameter tree =
    match tree with
    | Empty -> 0
    | Node(_, left, right) ->
        let leftHeight = treeHeight left + 1
        let rightHeight = treeHeight right + 1
        let throughRoot = leftHeight + rightHeight
        let leftDiameter = diameter left
        let rightDiameter = diameter right
        max throughRoot (max leftDiameter rightDiameter)

// All paths from root to leaf
let allPaths tree =
    let rec aux tree path =
        match tree with
        | Empty -> []
        | Node(v, Empty, Empty) -> [List.rev (v :: path)]
        | Node(v, left, right) ->
            aux left (v :: path) @ aux right (v :: path)
    aux tree []

let tree2 = 
    Node(1,
        Node(2, leaf 4, leaf 5),
        Node(3, leaf 6, Empty))

printfn "Height of tree2: %d" (treeHeight tree2)   // 2
printfn "Depth of 4: %A" (nodeDepth 4 tree2)       // Some 2
printfn "Depth of 3: %A" (nodeDepth 3 tree2)       // Some 1
printfn "Diameter: %d" (diameter tree2)            // 4 (4->2->1->3->6)
printfn "All paths: %A" (allPaths tree2)
// [[1;2;4]; [1;2;5]; [1;3;6]]

// ============ Balanced tree check ============
let rec isBalanced = function
    | Empty -> true
    | Node(_, left, right) ->
        let lh = treeHeight left
        let rh = treeHeight right
        abs (lh - rh) <= 1 && isBalanced left && isBalanced right

printfn "tree2 isBalanced: %b" (isBalanced tree2)

let unbalancedTree = 
    Node(1, 
        Node(2, 
            Node(3, 
                leaf 4, 
                Empty), 
            Empty), 
        Empty)
printfn "unbalanced isBalanced: %b" (isBalanced unbalancedTree)   // false
```

---

## 24.5 Tree Balancing Concept (AVL)

```fsharp
// ============ AVL Tree ============
// AVL Tree เป็น self-balancing BST
// Balance factor = height(left) - height(right) ∈ {-1, 0, 1}

type AVLTree<'T when 'T : comparison> =
    | AVLEmpty
    | AVLNode of value: 'T * height: int * left: AVLTree<'T> * right: AVLTree<'T>

module AVL =
    let height = function
        | AVLEmpty -> 0
        | AVLNode(_, h, _, _) -> h
    
    let balanceFactor = function
        | AVLEmpty -> 0
        | AVLNode(_, _, left, right) -> height left - height right
    
    let node v left right =
        let h = 1 + max (height left) (height right)
        AVLNode(v, h, left, right)
    
    // Right rotation
    let rotateRight = function
        | AVLNode(y, _, AVLNode(x, _, xleft, xright), yright) ->
            node x xleft (node y xright yright)
        | tree -> tree
    
    // Left rotation
    let rotateLeft = function
        | AVLNode(x, _, xleft, AVLNode(y, _, yleft, yright)) ->
            node y (node x xleft yleft) yright
        | tree -> tree
    
    // Balance the tree
    let balance tree =
        match tree with
        | AVLEmpty -> AVLEmpty
        | AVLNode(v, _, left, right) as t ->
            let bf = balanceFactor t
            if bf > 1 then
                // Left-heavy
                if balanceFactor left < 0 then
                    // Left-right case
                    let newLeft = rotateLeft left
                    rotateRight (node v newLeft right)
                else
                    // Left-left case
                    rotateRight t
            elif bf < -1 then
                // Right-heavy
                if balanceFactor right > 0 then
                    // Right-left case
                    let newRight = rotateRight right
                    rotateLeft (node v left newRight)
                else
                    // Right-right case
                    rotateLeft t
            else
                t
    
    let rec insert value = function
        | AVLEmpty -> node value AVLEmpty AVLEmpty
        | AVLNode(v, _, left, right) as t ->
            if value < v then
                balance (node v (insert value left) right)
            elif value > v then
                balance (node v left (insert value right))
            else t
    
    let rec contains value = function
        | AVLEmpty -> false
        | AVLNode(v, _, left, right) ->
            if value = v then true
            elif value < v then contains value left
            else contains value right
    
    let rec toList = function
        | AVLEmpty -> []
        | AVLNode(v, _, left, right) -> toList left @ [v] @ toList right

// ============ ใช้งาน AVL Tree ============
let avl = 
    [1..10]
    |> List.fold (fun t x -> AVL.insert x t) AVLEmpty

printfn "AVL sorted: %A" (AVL.toList avl)      // [1..10]
printfn "AVL height: %d" (AVL.height avl)       // ~4 (balanced)
printfn "AVL contains 5: %b" (AVL.contains 5 avl) // true

// เปรียบเทียบ: unbalanced BST vs AVL
let unbalancedBST = 
    [1..10]
    |> List.fold (fun t x -> bstInsert x t) Empty

printfn "Unbalanced BST height: %d" (treeHeight unbalancedBST)  // 9 (linear!)
printfn "AVL height: %d" (AVL.height avl)                       // ~4
```

---

## 24.6 Rose Tree (N-ary Tree)

```fsharp
// ============ Rose Tree ============
// แต่ละ node มีลูกได้หลายคน

type RoseTree<'T> = 
    | RoseNode of value: 'T * children: RoseTree<'T> list

// Constructors
let roseLeaf v = RoseNode(v, [])
let roseNode v children = RoseNode(v, children)

// Example: file system tree
let fileSystem =
    roseNode "root" [
        roseNode "home" [
            roseNode "alice" [
                roseLeaf "readme.txt"
                roseLeaf "photo.jpg"
            ]
            roseNode "bob" [
                roseLeaf "notes.md"
            ]
        ]
        roseNode "etc" [
            roseLeaf "config.json"
            roseLeaf "hosts"
        ]
        roseLeaf "boot"
    ]

// ============ Rose Tree Operations ============
let rec roseMap (f: 'T -> 'U) = function
    | RoseNode(v, children) -> RoseNode(f v, List.map (roseMap f) children)

let rec roseFold (f: 'acc -> 'T -> 'acc) (acc: 'acc) = function
    | RoseNode(v, children) ->
        let acc' = f acc v
        List.fold (fun a c -> roseFold f a c) acc' children

let rec roseToList = function
    | RoseNode(v, children) -> v :: List.collect roseToList children

let rec roseSize = function
    | RoseNode(_, children) -> 1 + List.sumBy roseSize children

let rec roseHeight = function
    | RoseNode(_, []) -> 0
    | RoseNode(_, children) -> 1 + List.map roseHeight children |> List.max

// ============ ใช้งาน ============
printfn "File system size: %d" (roseSize fileSystem)
printfn "File system height: %d" (roseHeight fileSystem)
printfn "All files: %A" (roseToList fileSystem)

// ============ Print tree ============
let rec printTree indent = function
    | RoseNode(v, children) ->
        printfn "%s%A" indent v
        for child in children do
            printTree (indent + "  ") child

printfn "\nFile System:"
printTree "" fileSystem

// ============ Find in Rose Tree ============
let rec roseFind (pred: 'T -> bool) = function
    | RoseNode(v, children) ->
        if pred v then Some v
        else
            children |> List.tryPick (roseFind pred)

let found = roseFind (fun s -> s = "notes.md") fileSystem
printfn "Found notes.md: %A" found   // Some "notes.md"

// ============ Rose Tree to paths ============
let rec rosePaths = function
    | RoseNode(v, []) -> [[v]]
    | RoseNode(v, children) ->
        children
        |> List.collect rosePaths
        |> List.map (fun path -> v :: path)

printfn "All paths:"
rosePaths fileSystem |> List.iter (fun p -> printfn "  %A" p)
```

---

## 24.7 Trie Implementation

```fsharp
open System.Collections.Generic

// ============ Trie (Prefix Tree) ============
// ใช้สำหรับ string operations: autocomplete, spell check

type TrieNode = {
    mutable IsEnd: bool
    Children: Dictionary<char, TrieNode>
}

let createNode () = { IsEnd = false; Children = Dictionary<char, TrieNode>() }

type Trie() =
    let root = createNode()
    
    member _.Insert(word: string) =
        let mutable current = root
        for c in word do
            if not (current.Children.ContainsKey(c)) then
                current.Children.[c] <- createNode()
            current <- current.Children.[c]
        current.IsEnd <- true
    
    member _.Search(word: string) =
        let mutable current = root
        let mutable found = true
        let mutable i = 0
        while found && i < word.Length do
            if current.Children.ContainsKey(word.[i]) then
                current <- current.Children.[word.[i]]
                i <- i + 1
            else
                found <- false
        found && current.IsEnd
    
    member _.StartsWith(prefix: string) =
        let mutable current = root
        let mutable found = true
        let mutable i = 0
        while found && i < prefix.Length do
            if current.Children.ContainsKey(prefix.[i]) then
                current <- current.Children.[prefix.[i]]
                i <- i + 1
            else
                found <- false
        found
    
    member _.Autocomplete(prefix: string) =
        let results = ResizeArray<string>()
        
        let mutable current = root
        let mutable found = true
        for c in prefix do
            if found then
                if current.Children.ContainsKey(c) then
                    current <- current.Children.[c]
                else
                    found <- false
        
        if found then
            let rec collect node word =
                if node.IsEnd then results.Add(word)
                for KeyValue(c, child) in node.Children do
                    collect child (word + string c)
            collect current prefix
        
        results |> Seq.toList
    
    member _.Delete(word: string) =
        let rec deleteHelper node word index =
            if index = String.length word then
                if node.IsEnd then
                    node.IsEnd <- false
                    node.Children.Count = 0   // return true if node can be deleted
                else
                    false
            else
                let c = word.[index]
                match node.Children.TryGetValue(c) with
                | true, child ->
                    let shouldDelete = deleteHelper child word (index + 1)
                    if shouldDelete then
                        node.Children.Remove(c) |> ignore
                        not node.IsEnd && node.Children.Count = 0
                    else
                        false
                | false, _ -> false
        
        deleteHelper root word 0 |> ignore

// ============ ใช้งาน Trie ============
let trie = Trie()

let words = ["apple"; "app"; "application"; "apply"; "banana"; "band"; "bandwidth"; "can"]
for word in words do
    trie.Insert(word)

printfn "Search 'apple': %b" (trie.Search("apple"))     // true
printfn "Search 'app': %b" (trie.Search("app"))         // true
printfn "Search 'appl': %b" (trie.Search("appl"))       // false
printfn "StartsWith 'app': %b" (trie.StartsWith("app")) // true
printfn "StartsWith 'xyz': %b" (trie.StartsWith("xyz")) // false

printfn "\nAutocomplete 'app':"
trie.Autocomplete("app") |> List.iter (fun w -> printfn "  %s" w)

printfn "\nAutocomplete 'ban':"
trie.Autocomplete("ban") |> List.iter (fun w -> printfn "  %s" w)

trie.Delete("app")
printfn "After delete 'app', search: %b" (trie.Search("app"))     // false
printfn "After delete 'app', autocomplete 'app':"
trie.Autocomplete("app") |> List.iter (fun w -> printfn "  %s" w)
```

---

## 24.8 Expression Tree

```fsharp
// ============ Expression Tree ============
// แทนนิพจน์คณิตศาสตร์เป็น tree

type Expr =
    | Num of float
    | Var of string
    | Add of Expr * Expr
    | Sub of Expr * Expr
    | Mul of Expr * Expr
    | Div of Expr * Expr
    | Neg of Expr
    | Pow of Expr * Expr

// ============ Evaluate ============
let rec evaluate (env: Map<string, float>) = function
    | Num n -> n
    | Var name -> 
        env |> Map.tryFind name |> Option.defaultWith (fun () -> failwith $"Unknown variable: {name}")
    | Add(e1, e2) -> evaluate env e1 + evaluate env e2
    | Sub(e1, e2) -> evaluate env e1 - evaluate env e2
    | Mul(e1, e2) -> evaluate env e1 * evaluate env e2
    | Div(e1, e2) -> evaluate env e1 / evaluate env e2
    | Neg e -> -(evaluate env e)
    | Pow(base_, exp) -> System.Math.Pow(evaluate env base_, evaluate env exp)

// ============ Pretty Print ============
let rec prettyPrint = function
    | Num n -> string n
    | Var name -> name
    | Add(e1, e2) -> sprintf "(%s + %s)" (prettyPrint e1) (prettyPrint e2)
    | Sub(e1, e2) -> sprintf "(%s - %s)" (prettyPrint e1) (prettyPrint e2)
    | Mul(e1, e2) -> sprintf "(%s * %s)" (prettyPrint e1) (prettyPrint e2)
    | Div(e1, e2) -> sprintf "(%s / %s)" (prettyPrint e1) (prettyPrint e2)
    | Neg e -> sprintf "(-%s)" (prettyPrint e)
    | Pow(b, e) -> sprintf "(%s ^ %s)" (prettyPrint b) (prettyPrint e)

// ============ Symbolic Differentiation ============
let rec differentiate (var: string) = function
    | Num _ -> Num 0.0
    | Var name -> if name = var then Num 1.0 else Num 0.0
    | Add(e1, e2) -> Add(differentiate var e1, differentiate var e2)
    | Sub(e1, e2) -> Sub(differentiate var e1, differentiate var e2)
    | Mul(e1, e2) -> 
        Add(Mul(differentiate var e1, e2), Mul(e1, differentiate var e2))
    | Div(e1, e2) ->
        Div(Sub(Mul(differentiate var e1, e2), Mul(e1, differentiate var e2)),
            Mul(e2, e2))
    | Neg e -> Neg(differentiate var e)
    | Pow(base_, Num n) ->  // only handle constant exponent
        Mul(Mul(Num n, Pow(base_, Num (n - 1.0))), differentiate var base_)
    | Pow _ -> failwith "Not supported"

// ============ Simplify ============
let rec simplify = function
    | Add(Num 0.0, e) | Add(e, Num 0.0) -> simplify e
    | Sub(e, Num 0.0) -> simplify e
    | Sub(Num 0.0, e) -> Neg(simplify e)
    | Mul(Num 0.0, _) | Mul(_, Num 0.0) -> Num 0.0
    | Mul(Num 1.0, e) | Mul(e, Num 1.0) -> simplify e
    | Div(e, Num 1.0) -> simplify e
    | Neg(Neg e) -> simplify e
    | Add(e1, e2) -> Add(simplify e1, simplify e2)
    | Sub(e1, e2) -> Sub(simplify e1, simplify e2)
    | Mul(e1, e2) -> Mul(simplify e1, simplify e2)
    | Div(e1, e2) -> Div(simplify e1, simplify e2)
    | Neg e -> Neg(simplify e)
    | Pow(e, n) -> Pow(simplify e, simplify n)
    | e -> e

// ============ ใช้งาน ============
// f(x) = x^2 + 2x + 1
let expr = 
    Add(
        Add(
            Pow(Var "x", Num 2.0),
            Mul(Num 2.0, Var "x")),
        Num 1.0)

printfn "Expression: %s" (prettyPrint expr)

let env = Map.ofList [("x", 3.0)]
printfn "f(3) = %f" (evaluate env expr)   // 9 + 6 + 1 = 16

let deriv = differentiate "x" expr
printfn "f'(x) = %s" (prettyPrint deriv)
let simplifiedDeriv = simplify deriv
printfn "f'(x) simplified = %s" (prettyPrint simplifiedDeriv)
printfn "f'(3) = %f" (evaluate env simplifiedDeriv)   // 2*3 + 2 = 8

// ============ Collecting variables ============
let rec collectVars = function
    | Var name -> Set.singleton name
    | Num _ -> Set.empty
    | Add(e1, e2) | Sub(e1, e2) | Mul(e1, e2) | Div(e1, e2) | Pow(e1, e2) ->
        Set.union (collectVars e1) (collectVars e2)
    | Neg e -> collectVars e

let multiVarExpr = Add(Mul(Var "x", Var "y"), Mul(Var "z", Num 2.0))
printfn "Variables in expression: %A" (collectVars multiVarExpr)
```

---

## 24.9 Decision Tree Concept

```fsharp
// ============ Decision Tree ============
// ใช้สำหรับ classification, rule-based systems

type Condition<'T> = 'T -> bool

type DecisionTree<'T, 'R> =
    | Leaf of result: 'R
    | Branch of 
        question: string * 
        condition: Condition<'T> * 
        trueTree: DecisionTree<'T, 'R> * 
        falseTree: DecisionTree<'T, 'R>

// Evaluate decision tree
let rec evaluate (tree: DecisionTree<'T, 'R>) (input: 'T) : 'R =
    match tree with
    | Leaf result -> result
    | Branch(_, condition, trueTree, falseTree) ->
        if condition input then evaluate trueTree input
        else evaluate falseTree input

// Explain the path taken
let rec explain (tree: DecisionTree<'T, 'R>) (input: 'T) : string list * 'R =
    match tree with
    | Leaf result -> ([], result)
    | Branch(question, condition, trueTree, falseTree) ->
        let answer = if condition input then "Yes" else "No"
        let (path, result) = 
            if condition input then explain trueTree input
            else explain falseTree input
        ((sprintf "%s -> %s" question answer) :: path, result)

// ============ ตัวอย่าง: Weather recommendation ============
type WeatherData = {
    Temperature: float  // Celsius
    IsRaining: bool
    Humidity: float    // 0-100%
    IsWeekend: bool
}

let weatherDecision =
    Branch("Is it raining?", 
        (fun d -> d.IsRaining),
        Branch("Is it weekend?",
            (fun d -> d.IsWeekend),
            Leaf "Stay home and watch movies",
            Leaf "Bring umbrella to work"),
        Branch("Is temperature above 30C?",
            (fun d -> d.Temperature > 30.0),
            Branch("Is humidity below 60%?",
                (fun d -> d.Humidity < 60.0),
                Leaf "Go to the beach!",
                Leaf "Stay in air-conditioned place"),
            Branch("Is it weekend?",
                (fun d -> d.IsWeekend),
                Leaf "Perfect for outdoor activities!",
                Leaf "Nice day for work")))

let today = { Temperature = 32.0; IsRaining = false; Humidity = 50.0; IsWeekend = true }
let (path, recommendation) = explain weatherDecision today

printfn "Weather decision:"
path |> List.iter (fun step -> printfn "  %s" step)
printfn "Recommendation: %s" recommendation

// ============ ตัวอย่าง: Loan approval ============
type LoanApplication = {
    CreditScore: int
    AnnualIncome: float
    LoanAmount: float
    ExistingDebt: float
}

let debtToIncomeRatio (app: LoanApplication) =
    (app.ExistingDebt + app.LoanAmount) / app.AnnualIncome

let loanDecision =
    Branch("Credit score >= 700?",
        (fun app -> app.CreditScore >= 700),
        Branch("Debt-to-income <= 0.4?",
            (fun app -> debtToIncomeRatio app <= 0.4),
            Leaf "Approved - Standard rate",
            Leaf "Approved - Higher rate"),
        Branch("Credit score >= 600?",
            (fun app -> app.CreditScore >= 600),
            Branch("Debt-to-income <= 0.3?",
                (fun app -> debtToIncomeRatio app <= 0.3),
                Leaf "Approved - High rate",
                Leaf "Declined - Too much debt"),
            Leaf "Declined - Poor credit"))

let applicant = {
    CreditScore = 720
    AnnualIncome = 60000.0
    LoanAmount = 20000.0
    ExistingDebt = 5000.0
}

let (loanPath, loanResult) = explain loanDecision applicant
printfn "\nLoan decision:"
loanPath |> List.iter (fun step -> printfn "  %s" step)
printfn "Result: %s" loanResult
```

---

## 24.10 Zipper for Tree Navigation

```fsharp
// ============ Zipper Pattern ============
// Zipper ให้เราสามารถ navigate ใน tree ได้อย่าง efficient
// โดยไม่ต้อง rebuild ทั้ง tree ทุกครั้ง

// Context = ที่เราอยู่ใน tree (breadcrumbs)
type Direction = GoLeft | GoRight

type TreeContext<'T> =
    | Top
    | GoedLeft of parent: 'T * right: BinaryTree<'T> * context: TreeContext<'T>
    | GoedRight of left: BinaryTree<'T> * parent: 'T * context: TreeContext<'T>

type Zipper<'T> = {
    Focus: BinaryTree<'T>
    Context: TreeContext<'T>
}

module Zipper =
    let fromTree tree = { Focus = tree; Context = Top }
    
    let goLeft z =
        match z.Focus with
        | Empty -> None
        | Node(v, left, right) ->
            Some { Focus = left; Context = GoedLeft(v, right, z.Context) }
    
    let goRight z =
        match z.Focus with
        | Empty -> None
        | Node(v, left, right) ->
            Some { Focus = right; Context = GoedRight(left, v, z.Context) }
    
    let goUp z =
        match z.Context with
        | Top -> None
        | GoedLeft(v, right, context) ->
            Some { Focus = Node(v, z.Focus, right); Context = context }
        | GoedRight(left, v, context) ->
            Some { Focus = Node(v, left, z.Focus); Context = context }
    
    let toTop z =
        let rec aux z =
            match z.Context with
            | Top -> z
            | _ -> 
                match goUp z with
                | Some z' -> aux z'
                | None -> z
        aux z
    
    let modify (f: BinaryTree<'T> -> BinaryTree<'T>) z =
        { z with Focus = f z.Focus }
    
    let replace newTree z = { z with Focus = newTree }
    
    let current z = z.Focus
    
    let isAtTop z = z.Context = Top

// ============ ใช้งาน Zipper ============
let tree = 
    Node(1,
        Node(2, leaf 4, leaf 5),
        Node(3, leaf 6, leaf 7))

let z0 = Zipper.fromTree tree
printfn "At root: %A" (Zipper.current z0)

let z1 = z0 |> Zipper.goLeft |> Option.get
printfn "After goLeft: %A" (Zipper.current z1)   // Node(2, ...)

let z2 = z1 |> Zipper.goRight |> Option.get
printfn "After goRight: %A" (Zipper.current z2)  // leaf 5

// Modify a subtree
let z3 = z2 |> Zipper.replace (leaf 99)
printfn "Replaced 5 with 99"

// Go back to root
let z4 = Zipper.toTop z3
printfn "Back at root: %A" (Zipper.current z4)
// The 5 should now be 99

printfn "In-order after modification: %A" (inorder (Zipper.current z4))
// [4; 2; 99; 1; 6; 3; 7]
```

---

## 24.11 JSON as Tree Structure

```fsharp
// ============ JSON AST ============
// JSON สามารถแทนด้วย recursive type

type JsonValue =
    | JsonNull
    | JsonBool of bool
    | JsonNumber of float
    | JsonString of string
    | JsonArray of JsonValue list
    | JsonObject of (string * JsonValue) list

// ============ Serialize to JSON string ============
let rec jsonToString indent = function
    | JsonNull -> "null"
    | JsonBool b -> if b then "true" else "false"
    | JsonNumber n -> 
        if n = System.Math.Floor(n) then sprintf "%d" (int n)
        else sprintf "%g" n
    | JsonString s -> sprintf "\"%s\"" s
    | JsonArray items ->
        let inner = items |> List.map (jsonToString (indent + "  ")) |> String.concat (sprintf ",\n%s" (indent + "  "))
        sprintf "[\n%s  %s\n%s]" indent inner indent
    | JsonObject pairs ->
        let inner = 
            pairs 
            |> List.map (fun (k, v) -> sprintf "%s  \"%s\": %s" indent k (jsonToString (indent + "  ") v))
            |> String.concat ",\n"
        sprintf "{\n%s\n%s}" inner indent

// ============ Query JSON ============
let rec jsonGet (path: string list) (json: JsonValue) : JsonValue option =
    match path, json with
    | [], v -> Some v
    | key :: rest, JsonObject pairs ->
        pairs |> List.tryFind (fst >> (=) key) |> Option.map snd |> Option.bind (jsonGet rest)
    | index :: rest, JsonArray items ->
        match System.Int32.TryParse(index) with
        | true, i when i >= 0 && i < List.length items ->
            jsonGet rest items.[i]
        | _ -> None
    | _ -> None

// ============ ตัวอย่าง ============
let userData = 
    JsonObject [
        ("name", JsonString "Alice")
        ("age", JsonNumber 30.0)
        ("active", JsonBool true)
        ("scores", JsonArray [JsonNumber 95.0; JsonNumber 87.0; JsonNumber 92.0])
        ("address", JsonObject [
            ("city", JsonString "Bangkok")
            ("country", JsonString "Thailand")
            ("zip", JsonString "10100")
        ])
        ("tags", JsonArray [JsonString "admin"; JsonString "user"])
    ]

printfn "JSON:\n%s" (jsonToString "" userData)
printfn "\nname: %A" (jsonGet ["name"] userData)
printfn "city: %A" (jsonGet ["address"; "city"] userData)
printfn "scores[1]: %A" (jsonGet ["scores"; "1"] userData)

// ============ Transform JSON ============
let rec jsonMap (f: JsonValue -> JsonValue) = function
    | JsonArray items -> JsonArray (List.map (jsonMap f >> f) items)
    | JsonObject pairs -> JsonObject (List.map (fun (k, v) -> (k, jsonMap f v |> f)) pairs)
    | v -> f v

// Double all numbers
let doubledNums = jsonMap (function JsonNumber n -> JsonNumber (n * 2.0) | v -> v) userData
printfn "\nDoubled numbers age: %A" (jsonGet ["age"] doubledNums)

// ============ Count nodes ============
let rec jsonSize = function
    | JsonNull | JsonBool _ | JsonNumber _ | JsonString _ -> 1
    | JsonArray items -> 1 + List.sumBy jsonSize items
    | JsonObject pairs -> 1 + List.sumBy (snd >> jsonSize) pairs

printfn "JSON tree size: %d" (jsonSize userData)
```

---

## สรุป (Summary)

```
Binary Tree:
- DU: Empty | Node of 'T * Tree * Tree
- BST: insert O(log n), search O(log n), delete O(log n)
- Traversals: inorder, preorder, postorder, level-order
- AVL: self-balancing BST, O(log n) guaranteed

Rose Tree (N-ary):
- แต่ละ node มีลูกได้หลายคน
- ใช้สำหรับ file systems, DOM, organizational charts

Trie:
- Prefix tree สำหรับ strings
- Search: O(m) where m = word length
- Autocomplete, spell check

Expression Tree:
- แทน mathematical/logical expressions
- Evaluate, differentiate, simplify

Zipper:
- Navigate tree โดยมี context (breadcrumbs)
- Functional cursor สำหรับ tree editing

JSON as Tree:
- JSON structure เป็น recursive data
- Parse, serialize, query, transform
```

---

*จบ Part 24 - ต้นไม้ (Trees)*
