# Part 25 - กราฟ (Graphs)

## บทนำ (Introduction)

กราฟ (Graph) เป็นโครงสร้างข้อมูลที่ประกอบด้วย:
- **Vertices/Nodes** - จุดยอด
- **Edges** - เส้นเชื่อม

กราฟใช้แทน networks, relationships, routes, dependencies และอื่นๆ อีกมากมาย

---

## 25.1 Graph Representations

```fsharp
// ============ Adjacency List Representation ============
// เหมาะสำหรับ sparse graphs (เส้นน้อย)
// Space: O(V + E)

// Undirected graph as Map<int, int list>
let undirectedGraph = Map.ofList [
    (0, [1; 2])
    (1, [0; 3; 4])
    (2, [0; 5; 6])
    (3, [1])
    (4, [1; 5])
    (5, [2; 4; 7])
    (6, [2; 7])
    (7, [5; 6])
]

// Directed graph (เส้นมีทิศทาง)
let directedGraph = Map.ofList [
    (0, [1; 2])
    (1, [3])
    (2, [3; 4])
    (3, [5])
    (4, [5])
    (5, [])
]

// Weighted graph as Map<int, (int * float) list>
// (neighbor, weight)
let weightedGraph = Map.ofList [
    (0, [(1, 4.0); (2, 1.0)])
    (1, [(3, 1.0)])
    (2, [(1, 2.0); (3, 5.0)])
    (3, [])
]

// ============ Adjacency Matrix Representation ============
// เหมาะสำหรับ dense graphs (เส้นมาก)
// Space: O(V^2)

let n = 5
let adjMatrix = Array2D.create n n false

// Add edges
let addEdge (matrix: bool[,]) from to_ =
    matrix.[from, to_] <- true
    matrix.[to_, from] <- true   // undirected

addEdge adjMatrix 0 1
addEdge adjMatrix 0 2
addEdge adjMatrix 1 3
addEdge adjMatrix 1 4
addEdge adjMatrix 2 3

printfn "Adjacency Matrix:"
for i in 0..n-1 do
    for j in 0..n-1 do
        printf "%s " (if adjMatrix.[i, j] then "1" else "0")
    printfn ""

// ============ Edge List Representation ============
type Edge = { From: int; To: int; Weight: float }

let edgeList = [
    { From = 0; To = 1; Weight = 4.0 }
    { From = 0; To = 2; Weight = 1.0 }
    { From = 1; To = 3; Weight = 1.0 }
    { From = 2; To = 1; Weight = 2.0 }
    { From = 2; To = 3; Weight = 5.0 }
]

// Convert edge list to adjacency list
let edgeListToAdjList (edges: Edge list) =
    edges |> List.fold (fun m e ->
        let neighbors = m |> Map.tryFind e.From |> Option.defaultValue []
        Map.add e.From ((e.To, e.Weight) :: neighbors) m
    ) Map.empty

let adjFromEdges = edgeListToAdjList edgeList
printfn "\nAdj list from edges: %A" adjFromEdges
```

---

## 25.2 Directed vs Undirected Graphs

```fsharp
// ============ Graph Type ============
type GraphType = Directed | Undirected

type Graph<'V when 'V : comparison> = {
    Vertices: Set<'V>
    Adjacency: Map<'V, 'V list>
    GraphType: GraphType
}

module Graph =
    let empty graphType = { 
        Vertices = Set.empty
        Adjacency = Map.empty
        GraphType = graphType
    }
    
    let addVertex v g = { g with Vertices = Set.add v g.Vertices }
    
    let addEdge from to_ g =
        let adj = 
            g.Adjacency 
            |> Map.tryFind from 
            |> Option.defaultValue []
        let newAdj = Map.add from (to_ :: adj) g.Adjacency
        
        match g.GraphType with
        | Undirected ->
            let adj2 = newAdj |> Map.tryFind to_ |> Option.defaultValue []
            let newAdj2 = Map.add to_ (from :: adj2) newAdj
            { g with 
                Vertices = g.Vertices |> Set.add from |> Set.add to_
                Adjacency = newAdj2 }
        | Directed ->
            { g with 
                Vertices = g.Vertices |> Set.add from |> Set.add to_
                Adjacency = newAdj }
    
    let neighbors v g = 
        g.Adjacency |> Map.tryFind v |> Option.defaultValue []
    
    let vertexCount g = Set.count g.Vertices
    
    let edgeCount g =
        let total = g.Adjacency |> Map.fold (fun acc _ vs -> acc + List.length vs) 0
        match g.GraphType with
        | Undirected -> total / 2
        | Directed -> total
    
    let hasEdge from to_ g =
        g.Adjacency 
        |> Map.tryFind from 
        |> Option.map (List.contains to_)
        |> Option.defaultValue false
    
    let inDegree v g =
        g.Adjacency 
        |> Map.fold (fun acc _ neighbors -> 
            acc + (if List.contains v neighbors then 1 else 0)
        ) 0
    
    let outDegree v g = 
        g.Adjacency |> Map.tryFind v |> Option.map List.length |> Option.defaultValue 0
    
    let degree v g =
        match g.GraphType with
        | Undirected -> outDegree v g
        | Directed -> inDegree v g + outDegree v g

// ============ ใช้งาน ============
let socialNetwork = 
    Graph.empty Undirected
    |> Graph.addEdge "Alice" "Bob"
    |> Graph.addEdge "Alice" "Charlie"
    |> Graph.addEdge "Bob" "Diana"
    |> Graph.addEdge "Charlie" "Diana"
    |> Graph.addEdge "Diana" "Eve"

printfn "Vertices: %d" (Graph.vertexCount socialNetwork)
printfn "Edges: %d" (Graph.edgeCount socialNetwork)
printfn "Alice's friends: %A" (Graph.neighbors "Alice" socialNetwork)
printfn "Alice's degree: %d" (Graph.degree "Alice" socialNetwork)

let webGraph =
    Graph.empty Directed
    |> Graph.addEdge "homepage" "products"
    |> Graph.addEdge "homepage" "about"
    |> Graph.addEdge "products" "checkout"
    |> Graph.addEdge "about" "contact"
    |> Graph.addEdge "checkout" "confirmation"

printfn "\nWeb graph edges: %d" (Graph.edgeCount webGraph)
printfn "homepage out-degree: %d" (Graph.outDegree "homepage" webGraph)
printfn "checkout in-degree: %d" (Graph.inDegree "checkout" webGraph)
```

---

## 25.3 DFS Implementation

```fsharp
open System.Collections.Generic

// ============ DFS - Recursive ============
let dfsRecursive (graph: Map<'V, 'V list>) (start: 'V) =
    let visited = HashSet<'V>()
    let order = ResizeArray<'V>()
    
    let rec dfs v =
        if not (visited.Contains(v)) then
            visited.Add(v) |> ignore
            order.Add(v)
            let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
            for neighbor in neighbors do
                dfs neighbor
    
    dfs start
    order |> Seq.toList

// ============ DFS - Iterative ============
let dfsIterative (graph: Map<'V, 'V list>) (start: 'V) =
    let visited = HashSet<'V>()
    let stack = Stack<'V>()
    let order = ResizeArray<'V>()
    
    stack.Push(start)
    
    while stack.Count > 0 do
        let v = stack.Pop()
        if not (visited.Contains(v)) then
            visited.Add(v) |> ignore
            order.Add(v)
            let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
            for neighbor in List.rev neighbors do
                if not (visited.Contains(neighbor)) then
                    stack.Push(neighbor)
    
    order |> Seq.toList

// ============ DFS All Paths ============
let dfsAllPaths (graph: Map<'V, 'V list>) (start: 'V) (target: 'V) =
    let visited = HashSet<'V>()
    let paths = ResizeArray<'V list>()
    
    let rec dfs v path =
        if v = target then
            paths.Add(List.rev path)
        else
            visited.Add(v) |> ignore
            let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
            for neighbor in neighbors do
                if not (visited.Contains(neighbor)) then
                    dfs neighbor (neighbor :: path)
            visited.Remove(v) |> ignore
    
    dfs start [start]
    paths |> Seq.toList

// ============ ใช้งาน ============
let graph = Map.ofList [
    (0, [1; 2])
    (1, [0; 3; 4])
    (2, [0; 5])
    (3, [1])
    (4, [1; 5])
    (5, [2; 4])
]

printfn "DFS recursive from 0: %A" (dfsRecursive graph 0)
printfn "DFS iterative from 0: %A" (dfsIterative graph 0)

let dirGraph = Map.ofList [
    (0, [1; 2])
    (1, [3; 4])
    (2, [4])
    (3, [5])
    (4, [5])
    (5, [])
]

printfn "All paths 0->5: %A" (dfsAllPaths dirGraph 0 5)
// [[0;1;3;5]; [0;1;4;5]; [0;2;4;5]]
```

---

## 25.4 BFS Implementation

```fsharp
// ============ BFS ============
let bfs (graph: Map<'V, 'V list>) (start: 'V) =
    let visited = HashSet<'V>()
    let queue = Queue<'V>()
    let order = ResizeArray<'V>()
    
    queue.Enqueue(start)
    visited.Add(start) |> ignore
    
    while queue.Count > 0 do
        let v = queue.Dequeue()
        order.Add(v)
        
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        for neighbor in neighbors do
            if not (visited.Contains(neighbor)) then
                visited.Add(neighbor) |> ignore
                queue.Enqueue(neighbor)
    
    order |> Seq.toList

// ============ BFS - Shortest Path ============
let bfsShortestPath (graph: Map<'V, 'V list>) (start: 'V) (target: 'V) =
    if start = target then Some [start]
    else
        let visited = HashSet<'V>()
        let queue = Queue<'V * 'V list>()
        
        queue.Enqueue((start, [start]))
        visited.Add(start) |> ignore
        
        let mutable result = None
        
        while queue.Count > 0 && result.IsNone do
            let (v, path) = queue.Dequeue()
            let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
            for neighbor in neighbors do
                if not (visited.Contains(neighbor)) then
                    let newPath = path @ [neighbor]
                    if neighbor = target then
                        result <- Some newPath
                    visited.Add(neighbor) |> ignore
                    queue.Enqueue((neighbor, newPath))
        
        result

// ============ BFS - All Distances ============
let bfsDistances (graph: Map<'V, 'V list>) (start: 'V) =
    let distances = Dictionary<'V, int>()
    let queue = Queue<'V>()
    
    distances.[start] <- 0
    queue.Enqueue(start)
    
    while queue.Count > 0 do
        let v = queue.Dequeue()
        let d = distances.[v]
        
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        for neighbor in neighbors do
            if not (distances.ContainsKey(neighbor)) then
                distances.[neighbor] <- d + 1
                queue.Enqueue(neighbor)
    
    distances |> Seq.map (fun kv -> (kv.Key, kv.Value)) |> Map.ofSeq

// ============ BFS Level-by-Level ============
let bfsLevels (graph: Map<'V, 'V list>) (start: 'V) =
    let visited = HashSet<'V>()
    let mutable current = [start]
    visited.Add(start) |> ignore
    let levels = ResizeArray<'V list>()
    
    while not (List.isEmpty current) do
        levels.Add(current)
        let next = 
            current 
            |> List.collect (fun v ->
                graph |> Map.tryFind v |> Option.defaultValue []
                |> List.filter (fun n -> 
                    if visited.Contains(n) then false
                    else visited.Add(n) |> ignore; true))
        current <- next
    
    levels |> Seq.toList

// ============ ใช้งาน ============
printfn "BFS from 0: %A" (bfs graph 0)
printfn "Shortest path 0->5: %A" (bfsShortestPath graph 0 5)
printfn "Distances from 0: %A" (bfsDistances graph 0)
printfn "BFS levels from 0: %A" (bfsLevels graph 0)
```

---

## 25.5 Topological Sort

```fsharp
// ============ Topological Sort ============
// สำหรับ Directed Acyclic Graph (DAG)

// DFS-based topological sort
let topologicalSortDFS (graph: Map<'V, 'V list>) =
    let allVertices = 
        graph 
        |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    let visited = HashSet<'V>()
    let stack = Stack<'V>()
    
    let rec dfs v =
        if not (visited.Contains(v)) then
            visited.Add(v) |> ignore
            let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
            for neighbor in neighbors do
                dfs neighbor
            stack.Push(v)
    
    for v in allVertices do
        dfs v
    
    stack |> Seq.toList

// Kahn's algorithm (BFS-based)
let topologicalSortKahn (graph: Map<'V, 'V list>) =
    let allVertices = 
        graph 
        |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    // Calculate in-degrees
    let inDegree = Dictionary<'V, int>()
    for v in allVertices do
        if not (inDegree.ContainsKey(v)) then
            inDegree.[v] <- 0
    
    for KeyValue(_, neighbors) in graph do
        for n in neighbors do
            inDegree.[n] <- inDegree.[n] + 1
    
    // Start with nodes that have in-degree 0
    let queue = Queue<'V>()
    for v in allVertices do
        if inDegree.[v] = 0 then
            queue.Enqueue(v)
    
    let result = ResizeArray<'V>()
    
    while queue.Count > 0 do
        let v = queue.Dequeue()
        result.Add(v)
        
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        for neighbor in neighbors do
            inDegree.[neighbor] <- inDegree.[neighbor] - 1
            if inDegree.[neighbor] = 0 then
                queue.Enqueue(neighbor)
    
    if result.Count = allVertices.Length then
        Some (result |> Seq.toList)
    else
        None  // Cycle detected!

// ============ ใช้งาน: Build System ============
let buildDeps = Map.ofList [
    ("app", ["service"; "ui"])
    ("service", ["database"; "cache"])
    ("ui", ["components"; "styles"])
    ("database", ["utils"])
    ("cache", ["utils"])
    ("components", [])
    ("styles", [])
    ("utils", [])
]

printfn "Build order (DFS): %A" (topologicalSortDFS buildDeps)
match topologicalSortKahn buildDeps with
| Some order -> printfn "Build order (Kahn): %A" order
| None -> printfn "Circular dependency detected!"

// ============ Course Prerequisites ============
let coursePrereqs = Map.ofList [
    ("CS101", [])
    ("CS201", ["CS101"])
    ("CS202", ["CS101"])
    ("CS301", ["CS201"; "CS202"])
    ("CS302", ["CS201"])
    ("CS401", ["CS301"; "CS302"])
]

match topologicalSortKahn coursePrereqs with
| Some order -> printfn "\nCourse order: %A" order
| None -> printfn "Circular prerequisite!"
```

---

## 25.6 Shortest Path - Dijkstra's Algorithm

```fsharp
open System.Collections.Generic

// ============ Dijkstra's Algorithm ============
// Single-source shortest path สำหรับ non-negative weights
// Time: O((V + E) log V) with priority queue

let dijkstra (graph: Map<int, (int * float) list>) (start: int) =
    let dist = Dictionary<int, float>()
    let prev = Dictionary<int, int>()
    let visited = HashSet<int>()
    
    // Priority queue: (distance, vertex)
    // Use simple sorted approach for clarity
    let pq = SortedSet<float * int>(Comparer.Create(fun (d1, v1) (d2, v2) ->
        let c = compare d1 d2
        if c <> 0 then c else compare v1 v2))
    
    // Initialize
    let allVertices = 
        graph 
        |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: List.map fst vs)
        |> List.distinct
    
    for v in allVertices do
        dist.[v] <- System.Double.PositiveInfinity
    
    dist.[start] <- 0.0
    pq.Add((0.0, start)) |> ignore
    
    while pq.Count > 0 do
        let (d, u) = pq.Min
        pq.Remove((d, u)) |> ignore
        
        if not (visited.Contains(u)) then
            visited.Add(u) |> ignore
            
            let neighbors = graph |> Map.tryFind u |> Option.defaultValue []
            for (v, w) in neighbors do
                let newDist = dist.[u] + w
                if newDist < dist.[v] then
                    pq.Remove((dist.[v], v)) |> ignore
                    dist.[v] <- newDist
                    prev.[v] <- u
                    pq.Add((newDist, v)) |> ignore
    
    // Build result
    let distances = dist |> Seq.map (fun kv -> (kv.Key, kv.Value)) |> Map.ofSeq
    
    let getPath target =
        let path = ResizeArray<int>()
        let mutable current = target
        while current <> start do
            path.Add(current)
            match prev.TryGetValue(current) with
            | true, p -> current <- p
            | false, _ -> 
                current <- start
                path.Clear()
        path.Add(start)
        path |> Seq.rev |> Seq.toList
    
    (distances, getPath)

// ============ ใช้งาน ============
let roadNetwork = Map.ofList [
    (0, [(1, 4.0); (2, 1.0)])
    (1, [(3, 1.0); (4, 5.0)])
    (2, [(1, 2.0); (3, 8.0); (5, 2.0)])
    (3, [(4, 2.0); (6, 3.0)])
    (4, [(6, 1.0)])
    (5, [(3, 5.0); (6, 4.0)])
    (6, [])
]

let (distances, getPath) = dijkstra roadNetwork 0

printfn "Shortest distances from 0:"
distances |> Map.iter (fun v d -> printfn "  0 -> %d: %.1f" v d)

printfn "\nShortest path 0 -> 6:"
printfn "  Path: %A" (getPath 6)
printfn "  Distance: %.1f" distances.[6]

// ============ Bellman-Ford ============
// Handles negative weights, detects negative cycles
let bellmanFord (edges: (int * int * float) list) (vertices: int) (start: int) =
    let dist = Array.create vertices System.Double.PositiveInfinity
    dist.[start] <- 0.0
    
    // Relax edges V-1 times
    for _ in 1..vertices - 1 do
        for (u, v, w) in edges do
            if dist.[u] <> System.Double.PositiveInfinity && dist.[u] + w < dist.[v] then
                dist.[v] <- dist.[u] + w
    
    // Check for negative cycles
    let hasNegCycle = 
        edges |> List.exists (fun (u, v, w) ->
            dist.[u] <> System.Double.PositiveInfinity && dist.[u] + w < dist.[v])
    
    if hasNegCycle then None
    else Some (dist |> Array.indexed |> Array.map (fun (i, d) -> (i, d)) |> Map.ofArray)

let graphEdges = [
    (0, 1, 4.0); (0, 2, 1.0)
    (1, 3, 1.0)
    (2, 1, 2.0); (2, 3, 8.0)
    (3, 4, 2.0)
]

match bellmanFord graphEdges 5 0 with
| Some dists -> 
    printfn "\nBellman-Ford from 0: %A" dists
| None -> 
    printfn "Negative cycle detected!"
```

---

## 25.7 Minimum Spanning Tree

```fsharp
// ============ Kruskal's Algorithm ============
// MST สำหรับ undirected weighted graph
// Time: O(E log E)

// Union-Find (Disjoint Set Union)
type UnionFind(n: int) =
    let parent = Array.init n id
    let rank = Array.create n 0
    
    member _.Find(x: int) =
        let mutable root = x
        while parent.[root] <> root do
            root <- parent.[root]
        // Path compression
        let mutable curr = x
        while curr <> root do
            let next = parent.[curr]
            parent.[curr] <- root
            curr <- next
        root
    
    member this.Union(x: int, y: int) =
        let rx = this.Find(x)
        let ry = this.Find(y)
        if rx = ry then false
        else
            if rank.[rx] < rank.[ry] then parent.[rx] <- ry
            elif rank.[rx] > rank.[ry] then parent.[ry] <- rx
            else parent.[ry] <- rx; rank.[rx] <- rank.[rx] + 1
            true
    
    member this.Connected(x: int, y: int) = this.Find(x) = this.Find(y)

let kruskalMST (vertices: int) (edges: (int * int * float) list) =
    let sortedEdges = edges |> List.sortBy (fun (_, _, w) -> w)
    let uf = UnionFind(vertices)
    let mst = ResizeArray<int * int * float>()
    let mutable totalWeight = 0.0
    
    for (u, v, w) in sortedEdges do
        if uf.Union(u, v) then
            mst.Add((u, v, w))
            totalWeight <- totalWeight + w
    
    (mst |> Seq.toList, totalWeight)

// ============ Prim's Algorithm ============
// MST สำหรับ dense graphs
// Time: O(V^2) naive, O(E log V) with priority queue

let primMST (graph: Map<int, (int * float) list>) (start: int) =
    let vertices = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: List.map fst vs)
        |> List.distinct
    
    let inMST = HashSet<int>()
    let key = Dictionary<int, float>()
    let parent = Dictionary<int, int>()
    
    for v in vertices do
        key.[v] <- System.Double.PositiveInfinity
    
    key.[start] <- 0.0
    
    while inMST.Count < vertices.Length do
        // Find minimum key vertex not in MST
        let u = 
            vertices 
            |> List.filter (fun v -> not (inMST.Contains(v)))
            |> List.minBy (fun v -> key.[v])
        
        inMST.Add(u) |> ignore
        
        let neighbors = graph |> Map.tryFind u |> Option.defaultValue []
        for (v, w) in neighbors do
            if not (inMST.Contains(v)) && w < key.[v] then
                key.[v] <- w
                parent.[v] <- u
    
    let mstEdges = 
        parent 
        |> Seq.map (fun kv -> (kv.Value, kv.Key, key.[kv.Key]))
        |> Seq.toList
    
    let totalWeight = mstEdges |> List.sumBy (fun (_, _, w) -> w)
    (mstEdges, totalWeight)

// ============ ใช้งาน ============
let networkEdges = [
    (0, 1, 2.0); (0, 3, 6.0)
    (1, 2, 3.0); (1, 3, 8.0); (1, 4, 5.0)
    (2, 4, 7.0)
    (3, 4, 9.0)
]

let (mstEdges, mstWeight) = kruskalMST 5 networkEdges
printfn "Kruskal MST edges: %A" mstEdges
printfn "Kruskal MST weight: %.1f" mstWeight

let networkAdj = Map.ofList [
    (0, [(1, 2.0); (3, 6.0)])
    (1, [(0, 2.0); (2, 3.0); (3, 8.0); (4, 5.0)])
    (2, [(1, 3.0); (4, 7.0)])
    (3, [(0, 6.0); (1, 8.0); (4, 9.0)])
    (4, [(1, 5.0); (2, 7.0); (3, 9.0)])
]

let (primEdges, primWeight) = primMST networkAdj 0
printfn "Prim MST edges: %A" primEdges
printfn "Prim MST weight: %.1f" primWeight
```

---

## 25.8 Cycle Detection

```fsharp
// ============ Cycle Detection in Undirected Graph ============
let hasCycleUndirected (graph: Map<int, int list>) =
    let visited = HashSet<int>()
    let vertices = graph |> Map.keys |> Seq.toList
    
    let rec dfs v parent =
        visited.Add(v) |> ignore
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        neighbors |> List.exists (fun u ->
            if not (visited.Contains(u)) then
                dfs u v
            elif u <> parent then
                true  // Found cycle
            else
                false)
    
    vertices |> List.exists (fun v ->
        if not (visited.Contains(v)) then
            dfs v -1
        else false)

// ============ Cycle Detection in Directed Graph ============
let hasCycleDirected (graph: Map<int, int list>) =
    let White = 0   // Not visited
    let Gray = 1    // In recursion stack
    let Black = 2   // Done
    
    let vertices = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    let color = Dictionary<int, int>()
    for v in vertices do
        color.[v] <- White
    
    let rec dfs v =
        color.[v] <- Gray
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        let foundCycle = neighbors |> List.exists (fun u ->
            match color.TryGetValue(u) with
            | true, Gray -> true        // Back edge = cycle
            | true, White -> dfs u      // Tree edge
            | _ -> false)
        color.[v] <- Black
        foundCycle
    
    vertices |> List.exists (fun v ->
        match color.TryGetValue(v) with
        | true, 0 -> dfs v
        | _ -> false)

// ============ Find all cycles ============
let findCycles (graph: Map<int, int list>) =
    let cycles = ResizeArray<int list>()
    let visited = HashSet<int>()
    let stack = ResizeArray<int>()
    let inStack = HashSet<int>()
    
    let vertices = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    let rec dfs v =
        visited.Add(v) |> ignore
        stack.Add(v)
        inStack.Add(v) |> ignore
        
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        for u in neighbors do
            if not (visited.Contains(u)) then
                dfs u
            elif inStack.Contains(u) then
                // Found a cycle - extract it
                let startIdx = stack.IndexOf(u)
                let cycle = stack.[startIdx..] |> Seq.toList
                cycles.Add(cycle)
        
        stack.RemoveAt(stack.Count - 1)
        inStack.Remove(v) |> ignore
    
    for v in vertices do
        if not (visited.Contains(v)) then
            dfs v
    
    cycles |> Seq.toList

// ============ ใช้งาน ============
let noCycleGraph = Map.ofList [(0, [1; 2]); (1, [3]); (2, [3]); (3, [])]
let cycleGraph = Map.ofList [(0, [1]); (1, [2]); (2, [0]); (3, [4]); (4, [3])]

printfn "Undirected cycle (no cycle): %b" (hasCycleUndirected (Map.ofList [(0, [1; 2]); (1, [0; 3]); (2, [0]); (3, [1])]))
printfn "Undirected cycle (has cycle): %b" (hasCycleUndirected (Map.ofList [(0, [1; 2]); (1, [0; 2]); (2, [0; 1])]))
printfn "Directed cycle (no cycle): %b" (hasCycleDirected noCycleGraph)
printfn "Directed cycle (has cycle): %b" (hasCycleDirected cycleGraph)
```

---

## 25.9 Connected Components

```fsharp
// ============ Connected Components ============
// หา groups ของ vertices ที่เชื่อมกัน

let connectedComponents (graph: Map<int, int list>) =
    let visited = HashSet<int>()
    let components = ResizeArray<int list>()
    
    let allVertices = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    let rec dfs v component =
        visited.Add(v) |> ignore
        component.Add(v)
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        for u in neighbors do
            if not (visited.Contains(u)) then
                dfs u component
    
    for v in allVertices do
        if not (visited.Contains(v)) then
            let component = ResizeArray<int>()
            dfs v component
            components.Add(component |> Seq.toList |> List.sort)
    
    components |> Seq.toList

// ============ Strongly Connected Components (Kosaraju's) ============
let stronglyConnectedComponents (graph: Map<int, int list>) =
    let vertices = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    // Step 1: DFS on original graph, record finish order
    let visited = HashSet<int>()
    let finishOrder = Stack<int>()
    
    let rec dfs1 v =
        visited.Add(v) |> ignore
        let neighbors = graph |> Map.tryFind v |> Option.defaultValue []
        for u in neighbors do
            if not (visited.Contains(u)) then dfs1 u
        finishOrder.Push(v)
    
    for v in vertices do
        if not (visited.Contains(v)) then dfs1 v
    
    // Step 2: Build reverse graph
    let reverseGraph = 
        graph |> Map.fold (fun acc u vs ->
            vs |> List.fold (fun acc2 v ->
                let neighbors = acc2 |> Map.tryFind v |> Option.defaultValue []
                Map.add v (u :: neighbors) acc2
            ) acc
        ) Map.empty
    
    // Step 3: DFS on reverse graph in finish order
    let visited2 = HashSet<int>()
    let sccs = ResizeArray<int list>()
    
    let rec dfs2 v component =
        visited2.Add(v) |> ignore
        component.Add(v)
        let neighbors = reverseGraph |> Map.tryFind v |> Option.defaultValue []
        for u in neighbors do
            if not (visited2.Contains(u)) then dfs2 u component
    
    while finishOrder.Count > 0 do
        let v = finishOrder.Pop()
        if not (visited2.Contains(v)) then
            let component = ResizeArray<int>()
            dfs2 v component
            sccs.Add(component |> Seq.toList |> List.sort)
    
    sccs |> Seq.toList

// ============ ใช้งาน ============
let disconnectedGraph = Map.ofList [
    (0, [1; 2])
    (1, [0])
    (2, [0])
    (3, [4])
    (4, [3])
    (5, [])
]

let components = connectedComponents disconnectedGraph
printfn "Connected components: %A" components
// [[0; 1; 2]; [3; 4]; [5]]

let directedForSCC = Map.ofList [
    (0, [1])
    (1, [2; 4])
    (2, [3])
    (3, [0])
    (4, [5])
    (5, [6])
    (6, [4])
    (7, [6])
]

let sccs = stronglyConnectedComponents directedForSCC
printfn "Strongly connected components: %A" sccs
```

---

## 25.10 Practical Example: Dependency Resolution

```fsharp
// ============ Package Dependency Resolution ============
open System.Collections.Generic

type Package = {
    Name: string
    Version: string
    Dependencies: string list
}

// Resolve installation order
let resolveDependencies (packages: Map<string, Package>) (root: string) =
    let visited = HashSet<string>()
    let installing = HashSet<string>()  // detect circular deps
    let order = ResizeArray<string>()
    
    let rec install name =
        if installing.Contains(name) then
            failwith $"Circular dependency detected involving {name}"
        
        if not (visited.Contains(name)) then
            installing.Add(name) |> ignore
            
            match packages |> Map.tryFind name with
            | Some pkg ->
                for dep in pkg.Dependencies do
                    install dep
                order.Add(name)
                visited.Add(name) |> ignore
            | None ->
                failwith $"Package not found: {name}"
            
            installing.Remove(name) |> ignore
    
    install root
    order |> Seq.toList

// ============ Version conflict detection ============
type VersionedDep = { Name: string; MinVersion: int; MaxVersion: int }

let checkVersionConflicts (packages: Map<string, Package>) =
    let conflicts = ResizeArray<string>()
    
    // For each package, check if its dependencies exist
    for KeyValue(pkgName, pkg) in packages do
        for dep in pkg.Dependencies do
            if not (packages.ContainsKey(dep)) then
                conflicts.Add($"{pkgName} requires missing package {dep}")
    
    conflicts |> Seq.toList

// ============ ใช้งาน ============
let packages = Map.ofList [
    ("app", { Name = "app"; Version = "1.0"; Dependencies = ["http-client"; "database"] })
    ("http-client", { Name = "http-client"; Version = "2.0"; Dependencies = ["json-parser"; "utils"] })
    ("database", { Name = "database"; Version = "3.0"; Dependencies = ["utils"] })
    ("json-parser", { Name = "json-parser"; Version = "1.5"; Dependencies = [] })
    ("utils", { Name = "utils"; Version = "1.0"; Dependencies = [] })
]

let installOrder = resolveDependencies packages "app"
printfn "Installation order: %A" installOrder
// [json-parser; utils; http-client; database; app]

let conflicts = checkVersionConflicts packages
if conflicts.IsEmpty then
    printfn "No conflicts found"
else
    conflicts |> List.iter (printfn "Conflict: %s")

// ============ Network Routing ============
type Router = { Id: string; Links: (string * float) list }

let networkRouters = Map.ofList [
    ("A", { Id = "A"; Links = [("B", 10.0); ("C", 15.0)] })
    ("B", { Id = "B"; Links = [("A", 10.0); ("D", 12.0); ("E", 15.0)] })
    ("C", { Id = "C"; Links = [("A", 15.0); ("D", 10.0)] })
    ("D", { Id = "D"; Links = [("B", 12.0); ("C", 10.0); ("F", 1.0)] })
    ("E", { Id = "E"; Links = [("B", 15.0); ("F", 5.0)] })
    ("F", { Id = "F"; Links = [("D", 1.0); ("E", 5.0)] })
]

// Convert to adjacency list with weights
let routerGraph = 
    networkRouters |> Map.map (fun _ router -> router.Links)

// Use Dijkstra to find routing table
printfn "\nNetwork routing table from A:"
let dist = Dictionary<string, float>()
let prev = Dictionary<string, string>()
let visited = HashSet<string>()

for v in routerGraph.Keys do
    dist.[v] <- System.Double.PositiveInfinity
dist.["A"] <- 0.0

let pq = SortedSet<float * string>(Comparer.Create(fun (d1, v1) (d2, v2) ->
    let c = compare d1 d2
    if c <> 0 then c else compare v1 v2))
pq.Add((0.0, "A")) |> ignore

while pq.Count > 0 do
    let (d, u) = pq.Min
    pq.Remove((d, u)) |> ignore
    
    if not (visited.Contains(u)) then
        visited.Add(u) |> ignore
        let neighbors = routerGraph |> Map.tryFind u |> Option.defaultValue []
        for (v, w) in neighbors do
            let newDist = dist.[u] + w
            if newDist < dist.[v] then
                pq.Remove((dist.[v], v)) |> ignore
                dist.[v] <- newDist
                prev.[v] <- u
                pq.Add((newDist, v)) |> ignore

for v in ["B"; "C"; "D"; "E"; "F"] do
    let rec buildPath node =
        match prev.TryGetValue(node) with
        | true, p -> buildPath p @ [node]
        | false, _ -> [node]
    printfn "  A -> %s: distance=%.1f path=%A" v dist.[v] (buildPath v)
```

---

## 25.11 Graph Coloring

```fsharp
// ============ Graph Coloring ============
// จัดสีให้กับ vertices โดยไม่ให้ vertices ที่เชื่อมกันมีสีเดียวกัน
// ใช้สำหรับ scheduling, register allocation

let greedyColoring (graph: Map<int, int list>) =
    let vertices = 
        graph |> Map.keys 
        |> Seq.collect (fun v -> v :: (graph |> Map.tryFind v |> Option.defaultValue []))
        |> Seq.distinct
        |> Seq.toList
        |> List.sort
    
    let colors = Dictionary<int, int>()
    
    for v in vertices do
        let usedColors = 
            graph |> Map.tryFind v |> Option.defaultValue []
            |> List.choose (fun u -> 
                match colors.TryGetValue(u) with
                | true, c -> Some c
                | _ -> None)
            |> Set.ofList
        
        // Find smallest color not used by neighbors
        let color = 
            Seq.initInfinite id
            |> Seq.find (fun c -> not (Set.contains c usedColors))
        
        colors.[v] <- color
    
    colors |> Seq.map (fun kv -> (kv.Key, kv.Value)) |> Map.ofSeq

// ============ ใช้งาน ============
let conflictGraph = Map.ofList [
    (0, [1; 3])
    (1, [0; 2; 3])
    (2, [1; 4])
    (3, [0; 1; 4])
    (4, [2; 3])
]

let coloring = greedyColoring conflictGraph
printfn "Graph coloring:"
coloring |> Map.iter (fun v c -> printfn "  Vertex %d -> Color %d" v c)

let numColors = coloring |> Map.values |> Seq.distinct |> Seq.length
printfn "Colors used: %d" numColors

// ============ Exam Scheduling ============
type Exam = { Id: int; Subject: string; StudentsCount: int }
type Conflict = int * int   // Two exams that share students

let scheduleExams (exams: Exam list) (conflicts: Conflict list) =
    let conflictGraph = 
        conflicts |> List.fold (fun m (e1, e2) ->
            let n1 = m |> Map.tryFind e1 |> Option.defaultValue []
            let n2 = m |> Map.tryFind e2 |> Option.defaultValue []
            m |> Map.add e1 (e2 :: n1) |> Map.add e2 (e1 :: n2)
        ) Map.empty
    
    let slots = greedyColoring conflictGraph
    
    exams |> List.groupBy (fun e -> 
        slots |> Map.tryFind e.Id |> Option.defaultValue 0)

let exams = [
    { Id = 0; Subject = "Math"; StudentsCount = 50 }
    { Id = 1; Subject = "Physics"; StudentsCount = 40 }
    { Id = 2; Subject = "Chemistry"; StudentsCount = 35 }
    { Id = 3; Subject = "Biology"; StudentsCount = 45 }
    { Id = 4; Subject = "English"; StudentsCount = 60 }
]

let conflicts = [(0, 1); (0, 3); (1, 2); (1, 3); (2, 4); (3, 4)]
let schedule = scheduleExams exams conflicts

printfn "\nExam Schedule:"
for (slot, slotExams) in schedule do
    printfn "  Slot %d: %A" slot (slotExams |> List.map (fun e -> e.Subject))
```

---

## สรุป (Summary)

```
Graph Representations:
- Adjacency List: O(V+E) space, good for sparse
- Adjacency Matrix: O(V^2) space, good for dense

Graph Algorithms:
- DFS: O(V+E), depth-first exploration
- BFS: O(V+E), shortest path (unweighted)
- Dijkstra: O((V+E)logV), shortest path (non-negative weights)
- Bellman-Ford: O(VE), handles negative weights
- Kruskal/Prim: O(E log E), minimum spanning tree
- Topological Sort: O(V+E), ordering for DAG
- Kosaraju: O(V+E), strongly connected components

Applications:
- Social networks, routing, scheduling
- Dependency resolution, build systems
- Graph coloring, map coloring
```

---

*จบ Part 25 - กราฟ (Graphs)*
