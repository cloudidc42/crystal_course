# Part 107 - Data Science กับ F#

## บทนำ (Introduction)

F# มี ecosystem ที่แข็งแกร่งสำหรับ Data Science ด้วย libraries เช่น Deedle (data frames), FSharp.Stats (statistics), และ XPlot.Plotly (visualization)

---

## 1. Deedle Data Frames

### การติดตั้ง

```fsharp
// ใน script
#r "nuget: Deedle, 3.0.0"
#r "nuget: FSharp.Data, 6.3.0"

open Deedle
open FSharp.Data
```

### สร้าง DataFrame

```fsharp
open Deedle

// ===== จาก Records =====

type SaleRecord = {
    Date: System.DateTime
    Product: string
    Region: string
    Quantity: int
    UnitPrice: float
    Revenue: float
}

let salesData = 
    Frame.ofRecords [
        { Date = System.DateTime(2024, 1, 1); Product = "Apple"; Region = "North"; Quantity = 100; UnitPrice = 25.0; Revenue = 2500.0 }
        { Date = System.DateTime(2024, 1, 2); Product = "Banana"; Region = "South"; Quantity = 150; UnitPrice = 15.0; Revenue = 2250.0 }
        { Date = System.DateTime(2024, 1, 3); Product = "Cherry"; Region = "North"; Quantity = 80; UnitPrice = 45.0; Revenue = 3600.0 }
        { Date = System.DateTime(2024, 1, 4); Product = "Apple"; Region = "East"; Quantity = 120; UnitPrice = 25.0; Revenue = 3000.0 }
        { Date = System.DateTime(2024, 1, 5); Product = "Banana"; Region = "North"; Quantity = 200; UnitPrice = 15.0; Revenue = 3000.0 }
        { Date = System.DateTime(2024, 1, 6); Product = "Cherry"; Region = "South"; Quantity = 90; UnitPrice = 45.0; Revenue = 4050.0 }
        { Date = System.DateTime(2024, 1, 7); Product = "Apple"; Region = "West"; Quantity = 110; UnitPrice = 25.0; Revenue = 2750.0 }
        { Date = System.DateTime(2024, 1, 8); Product = "Banana"; Region = "East"; Quantity = 175; UnitPrice = 15.0; Revenue = 2625.0 }
    ]

// แสดงข้อมูล
printfn "Shape: %d rows x %d columns" (salesData |> Frame.countRows) (salesData |> Frame.countCols)
salesData |> Frame.print

// ===== จาก Series =====

let dates = Series.ofValues [
    System.DateTime(2024, 1, 1)
    System.DateTime(2024, 1, 2)
    System.DateTime(2024, 1, 3)
]

let prices = Series.ofValues [100.0; 105.0; 103.0]
let volumes = Series.ofValues [1000; 1200; 900]

let stockData = 
    Frame.ofColumns [
        "Date", dates |> Series.mapValues box
        "Close", prices |> Series.mapValues box
        "Volume", volumes |> Series.mapValues box
    ]

// ===== จาก CSV =====

let loadFromCsv (path: string) =
    Frame.ReadCsv(path, hasHeaders = true, inferTypes = true)

// ===== จาก FSharp.Data CSV Provider =====

type SalesCsv = CsvProvider<"data/sales.csv", HasHeaders = true>

let csvData = SalesCsv.Load("data/sales.csv")
let frame = Frame.ofRecords csvData.Rows
```

### Data Loading และ Manipulation

```fsharp
open Deedle

// ===== Select Columns =====

let revenueOnly = salesData?Revenue
let selected = salesData |> Frame.sliceCols ["Product"; "Revenue"; "Quantity"]

// ===== Filter Rows =====

let northSales = 
    salesData 
    |> Frame.filterRows (fun _ row -> row.GetAs<string>("Region") = "North")

let highRevenue =
    salesData
    |> Frame.filterRows (fun _ row -> row.GetAs<float>("Revenue") > 3000.0)

// ===== Sort =====

let sortedByRevenue = salesData |> Frame.sortRowsBy (fun r -> r.GetAs<float>("Revenue"))

// ===== Add Columns =====

let profit = salesData?Revenue - (salesData?Quantity * Series.mapValues (fun q -> q * 10.0) salesData?Quantity)
let withProfit = salesData |> Frame.addColumn "Profit" profit

// ===== Rename Columns =====

let renamed = 
    salesData 
    |> Frame.mapColKeys (fun k ->
        match k with
        | "UnitPrice" -> "Price"
        | "Quantity" -> "Qty"
        | other -> other)

// ===== Group By =====

let byProduct = salesData |> Frame.groupRowsByString "Product"

// ===== Pivot Table =====

let pivotTable =
    salesData
    |> Frame.pivotTable
        (fun _ r -> r.GetAs<string>("Region"))
        (fun _ r -> r.GetAs<string>("Product"))
        (fun frame -> frame?Revenue |> Stats.sum)
```

---

## 2. Statistical Operations

```fsharp
open Deedle

let revenues = salesData?Revenue

// ===== Basic Statistics =====

let mean = revenues |> Stats.mean
let stddev = revenues |> Stats.stdDev
let variance = revenues |> Stats.variance
let median = revenues |> Stats.median
let min = revenues |> Stats.min
let max = revenues |> Stats.max
let sum = revenues |> Stats.sum
let count = revenues |> Stats.count
let kurtosis = revenues |> Stats.kurt
let skewness = revenues |> Stats.skew

printfn "Revenue Statistics:"
printfn "  Mean:     %.2f" mean
printfn "  Std Dev:  %.2f" stddev
printfn "  Median:   %.2f" median
printfn "  Min:      %.2f" min
printfn "  Max:      %.2f" max
printfn "  Sum:      %.2f" sum
printfn "  Count:    %.0f" count

// ===== Percentiles =====

let p25 = revenues |> Stats.percentile 25.0
let p50 = revenues |> Stats.percentile 50.0
let p75 = revenues |> Stats.percentile 75.0
let p90 = revenues |> Stats.percentile 90.0

printfn "\nPercentiles:"
printfn "  P25: %.2f" p25
printfn "  P50: %.2f" p50
printfn "  P75: %.2f" p75
printfn "  P90: %.2f" p90

// ===== Moving Average =====

let movingAvg3 = revenues |> Stats.movingMean 3
let movingStd3 = revenues |> Stats.movingStdDev 3

// ===== Correlation =====

let correlation = 
    Stats.correlation salesData?Quantity salesData?Revenue

printfn "\nCorrelation (Qty vs Revenue): %.4f" correlation

// ===== Rolling Statistics =====

let rollingMean5 = revenues |> Stats.movingMean 5
let expandingMean = revenues |> Stats.expandingMean
```

---

## 3. FSharp.Stats

### การติดตั้ง

```fsharp
#r "nuget: FSharp.Stats, 0.5.0"

open FSharp.Stats
open FSharp.Stats.Distributions
```

### Descriptive Statistics

```fsharp
open FSharp.Stats

// ===== Basic Stats =====

let data = [| 2.5; 3.1; 2.8; 4.2; 3.7; 2.9; 3.5; 4.0; 2.6; 3.8 |]

// Mean
let mean = Seq.mean data
let weightedMean = Seq.meanWeighted data (Array.create 10 1.0)

// Variance
let variance = Seq.var data
let populationVariance = Seq.varPopulation data

// Standard Deviation
let stdDev = Seq.stdev data
let populationStdDev = Seq.stdevPopulation data

// Coefficient of Variation
let cv = stdDev / mean * 100.0
printfn "CV: %.2f%%" cv

// ===== Moment Statistics =====

let skewness = Seq.skewness data
let kurtosis = Seq.kurtosis data

printfn "Skewness: %.4f" skewness
printfn "Kurtosis: %.4f" kurtosis

// ===== Range Statistics =====

let range = Seq.range data  // max - min
let interquartileRange = Seq.iqr data

// ===== Summary Statistics =====

let summarize (name: string) (values: float[]) =
    printfn "\n%s Statistics:" name
    printfn "  N:          %d" values.Length
    printfn "  Mean:       %.4f" (Seq.mean values)
    printfn "  Std Dev:    %.4f" (Seq.stdev values)
    printfn "  Min:        %.4f" (Array.min values)
    printfn "  Q1:         %.4f" (Quantile.computePercentile values 0.25)
    printfn "  Median:     %.4f" (Quantile.computePercentile values 0.50)
    printfn "  Q3:         %.4f" (Quantile.computePercentile values 0.75)
    printfn "  Max:        %.4f" (Array.max values)
    printfn "  Skewness:   %.4f" (Seq.skewness values)
    printfn "  Kurtosis:   %.4f" (Seq.kurtosis values)

summarize "Revenue" [| 2500.0; 2250.0; 3600.0; 3000.0; 3000.0; 4050.0; 2750.0; 2625.0 |]
```

---

## 4. Hypothesis Testing

```fsharp
open FSharp.Stats
open FSharp.Stats.Testing

// ===== T-Test =====

// One sample t-test
let sample = [| 5.2; 4.8; 5.5; 4.9; 5.3; 5.1; 4.7; 5.4; 5.0; 5.2 |]
let hypotheticalMean = 5.0

let tTestResult = TTest.oneSample hypotheticalMean sample
printfn "\nOne-Sample T-Test:"
printfn "  T-statistic: %.4f" tTestResult.Statistic
printfn "  P-value:     %.4f" tTestResult.PValue
printfn "  Significant: %b" (tTestResult.PValue < 0.05)

// Two sample t-test (independent)
let group1 = [| 5.2; 4.8; 5.5; 4.9; 5.3; 5.1 |]
let group2 = [| 4.7; 4.5; 4.9; 4.6; 4.8; 4.4 |]

let twoSampleTest = TTest.twoSampleUnpaired group1 group2
printfn "\nTwo-Sample T-Test:"
printfn "  T-statistic: %.4f" twoSampleTest.Statistic
printfn "  P-value:     %.4f" twoSampleTest.PValue
printfn "  Groups differ significantly: %b" (twoSampleTest.PValue < 0.05)

// ===== Chi-Square Test =====

// Goodness of fit
let observed = [| 20; 15; 25; 10; 30 |]
let expected = [| 20; 20; 20; 20; 20 |]

let chiSquare = ChiSquare.goodnessOfFit observed expected
printfn "\nChi-Square Test:"
printfn "  Chi²: %.4f" chiSquare.Statistic
printfn "  P-value: %.4f" chiSquare.PValue

// ===== ANOVA =====

let anovaGroups = [
    [| 10.0; 11.0; 12.0; 9.0; 10.5 |]
    [| 13.0; 14.0; 12.5; 13.5; 14.5 |]
    [| 11.0; 10.5; 11.5; 12.0; 11.8 |]
]

let anova = Anova.oneWay anovaGroups
printfn "\nANOVA:"
printfn "  F-statistic: %.4f" anova.Statistic
printfn "  P-value:     %.4f" anova.PValue
```

---

## 5. Distributions

```fsharp
open FSharp.Stats
open FSharp.Stats.Distributions

// ===== Normal Distribution =====

let normal = Continuous.normal 0.0 1.0  // mean=0, sd=1

// PDF (Probability Density Function)
printfn "\nNormal Distribution N(0,1):"
printfn "  PDF(0) = %.4f" (normal.PDF 0.0)
printfn "  PDF(1) = %.4f" (normal.PDF 1.0)
printfn "  PDF(2) = %.4f" (normal.PDF 2.0)

// CDF (Cumulative Distribution Function)
printfn "  CDF(0) = %.4f" (normal.CDF 0.0)
printfn "  CDF(1.96) = %.4f" (normal.CDF 1.96)
printfn "  P(X <= 1.96) = %.4f" (normal.CDF 1.96)
printfn "  P(X in [-1.96, 1.96]) = %.4f" (normal.CDF 1.96 - normal.CDF -1.96)

// Inverse CDF (Quantile)
printfn "  Q(0.025) = %.4f" (normal.InverseCDF 0.025)  // -1.96
printfn "  Q(0.975) = %.4f" (normal.InverseCDF 0.975)  // 1.96

// Sampling
let samples = Array.init 1000 (fun _ -> normal.Sample())
printfn "\nSamples (n=1000):"
printfn "  Mean:    %.4f (expected 0.0)" (Seq.mean samples)
printfn "  Std Dev: %.4f (expected 1.0)" (Seq.stdev samples)

// ===== Other Distributions =====

// Poisson Distribution
let poisson = Discrete.poisson 3.5  // lambda=3.5

printfn "\nPoisson(λ=3.5):"
for k in 0..10 do
    printfn "  P(X=%d) = %.4f" k (poisson.PMF k)

// Binomial Distribution
let binomial = Discrete.binomial 0.3 10  // p=0.3, n=10

printfn "\nBinomial(p=0.3, n=10):"
for k in 0..10 do
    printfn "  P(X=%d) = %.4f" k (binomial.PMF k)

// Exponential Distribution
let exponential = Continuous.exponential 2.0  // rate=2

printfn "\nExponential(rate=2):"
printfn "  Mean: %.4f (expected %.4f)" (exponential.Mean) (1.0/2.0)
```

---

## 6. Correlation Analysis

```fsharp
open FSharp.Stats

// ===== Pearson Correlation =====

let height = [| 165.0; 170.0; 175.0; 160.0; 180.0; 168.0; 172.0; 178.0 |]
let weight = [| 60.0; 65.0; 70.0; 55.0; 80.0; 63.0; 68.0; 75.0 |]

let pearson = Correlation.pearson height weight
printfn "\nPearson Correlation:"
printfn "  r = %.4f" pearson

// ===== Spearman Correlation =====

let spearman = Correlation.spearman height weight
printfn "Spearman Correlation:"
printfn "  ρ = %.4f" spearman

// ===== Correlation Matrix =====

let data = [|
    height
    weight
    [| 25.0; 28.0; 30.0; 22.0; 35.0; 26.0; 29.0; 32.0 |]  // BMI-like
|]

let names = [| "Height"; "Weight"; "BMI" |]

printfn "\nCorrelation Matrix:"
printf "%-10s" ""
for name in names do printf "%-10s" name
printfn ""

for i in 0..data.Length-1 do
    printf "%-10s" names.[i]
    for j in 0..data.Length-1 do
        let r = Correlation.pearson data.[i] data.[j]
        printf "%-10.4f" r
    printfn ""

// ===== Linear Regression =====

// Simple Linear Regression: y = a + bx
let linearRegression (x: float[]) (y: float[]) =
    let n = float x.Length
    let sumX = Array.sum x
    let sumY = Array.sum y
    let sumXY = Array.map2 (*) x y |> Array.sum
    let sumX2 = x |> Array.map (fun xi -> xi * xi) |> Array.sum
    
    let slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX)
    let intercept = (sumY - slope * sumX) / n
    
    let predict xi = intercept + slope * xi
    
    let yPred = x |> Array.map predict
    let yMean = sumY / n
    
    // R² (coefficient of determination)
    let ssTot = y |> Array.map (fun yi -> (yi - yMean) ** 2.0) |> Array.sum
    let ssRes = Array.map2 (fun yi yh -> (yi - yh) ** 2.0) y yPred |> Array.sum
    let r2 = 1.0 - ssRes / ssTot
    
    slope, intercept, r2, predict

let slope, intercept, r2, predict = linearRegression height weight

printfn "\nLinear Regression: Weight = %.4f × Height + %.4f" slope intercept
printfn "R² = %.4f" r2
printfn "Prediction for height 170: %.2f" (predict 170.0)
```

---

## 7. XPlot.Plotly for Visualization

### การติดตั้ง

```fsharp
#r "nuget: XPlot.Plotly, 4.0.6"
#r "nuget: XPlot.Plotly.Interactive, 4.0.6"  // สำหรับ Jupyter

open XPlot.Plotly
```

### Plotly Charts

```fsharp
open XPlot.Plotly

// ===== Line Chart =====

let lineChart () =
    let x = [| 1.0; 2.0; 3.0; 4.0; 5.0; 6.0; 7.0; 8.0; 9.0; 10.0 |]
    let y = x |> Array.map (fun xi -> 2.0 * xi + 1.0 + (System.Random().NextDouble() - 0.5) * 2.0)
    
    let trace = Scatter(
        x = x,
        y = y,
        mode = "lines+markers",
        name = "ข้อมูล",
        line = Line(color = "#007bff", width = 2.0)
    )
    
    let layout = Layout(
        title = "Line Chart Example",
        xaxis = Xaxis(title = "X"),
        yaxis = Yaxis(title = "Y")
    )
    
    let chart = Chart.Plot([trace], layout)
    chart.Show()
    chart.GetHtml()

// ===== Bar Chart =====

let barChart () =
    let products = [| "Apple"; "Banana"; "Cherry"; "Date"; "Elderberry" |]
    let revenues = [| 12500.0; 8750.0; 15200.0; 6300.0; 9800.0 |]
    
    let trace = Bar(
        x = products,
        y = revenues,
        marker = Marker(
            color = [| "#007bff"; "#28a745"; "#dc3545"; "#ffc107"; "#6c757d" |]
        ),
        text = revenues |> Array.map (fun r -> sprintf "฿%.0f" r),
        textposition = "auto"
    )
    
    let layout = Layout(
        title = "ยอดขายตามสินค้า",
        xaxis = Xaxis(title = "สินค้า"),
        yaxis = Yaxis(title = "ยอดขาย (บาท)")
    )
    
    Chart.Plot([trace], layout)

// ===== Scatter Plot =====

let scatterPlot () =
    let rng = System.Random(42)
    let n = 100
    
    let x = Array.init n (fun _ -> rng.NextGaussian(0.0, 1.0))
    let y = x |> Array.map (fun xi -> 2.0 * xi + rng.NextGaussian(0.0, 0.5))
    
    let trace = Scatter(
        x = x,
        y = y,
        mode = "markers",
        marker = Marker(
            color = y,
            colorscale = "Viridis",
            size = 8.0,
            showscale = true
        ),
        text = Array.init n (fun i -> sprintf "Point %d" i)
    )
    
    let layout = Layout(
        title = "Scatter Plot",
        xaxis = Xaxis(title = "X"),
        yaxis = Yaxis(title = "Y")
    )
    
    Chart.Plot([trace], layout)

// ===== Pie Chart =====

let pieChart () =
    let labels = [| "Q1"; "Q2"; "Q3"; "Q4" |]
    let values = [| 25.0; 30.0; 20.0; 25.0 |]
    
    let trace = Pie(
        labels = labels,
        values = values,
        textinfo = "label+percent",
        hoverinfo = "label+value+percent",
        marker = Marker(
            colors = [| "#007bff"; "#28a745"; "#ffc107"; "#dc3545" |]
        )
    )
    
    let layout = Layout(title = "Quarterly Revenue Distribution")
    
    Chart.Plot([trace], layout)

// ===== Histogram =====

let histogram () =
    let data = 
        Array.init 1000 (fun _ -> 
            let rng = System.Random()
            rng.NextDouble() * 100.0)
    
    let trace = Histogram(
        x = data,
        nbinsx = 20,
        marker = Marker(
            color = "#007bff",
            line = Line(color = "white", width = 1.0)
        ),
        name = "Distribution"
    )
    
    let layout = Layout(
        title = "Data Distribution",
        xaxis = Xaxis(title = "Value"),
        yaxis = Yaxis(title = "Frequency")
    )
    
    Chart.Plot([trace], layout)

// ===== Box Plot =====

let boxPlot () =
    let regions = [| "North"; "South"; "East"; "West" |]
    let data = 
        regions |> Array.map (fun _ ->
            Array.init 20 (fun _ -> 
                let rng = System.Random()
                50.0 + rng.NextDouble() * 50.0))
    
    let traces = 
        Array.zip regions data
        |> Array.map (fun (region, values) ->
            Box(y = values, name = region) :> Trace)
    
    let layout = Layout(title = "Sales Distribution by Region")
    
    Chart.Plot(traces, layout)

// ===== Heatmap =====

let heatmap () =
    let months = [| "Jan"; "Feb"; "Mar"; "Apr"; "May"; "Jun" |]
    let products = [| "Apple"; "Banana"; "Cherry" |]
    
    let data = 
        Array.init products.Length (fun _ ->
            Array.init months.Length (fun _ ->
                let rng = System.Random()
                1000.0 + rng.NextDouble() * 4000.0))
    
    let trace = Heatmap(
        z = data,
        x = months,
        y = products,
        colorscale = "Blues"
    )
    
    let layout = Layout(title = "Sales Heatmap")
    
    Chart.Plot([trace], layout)
```

---

## 8. Data Analysis Workflow

### Complete Analysis Pipeline

```fsharp
// analysis.fsx
#r "nuget: Deedle, 3.0.0"
#r "nuget: FSharp.Stats, 0.5.0"
#r "nuget: XPlot.Plotly, 4.0.6"
#r "nuget: FSharp.Data, 6.3.0"

open System
open Deedle
open FSharp.Stats
open XPlot.Plotly

// ===== Step 1: Load Data =====

printfn "=== E-commerce Data Analysis ==="
printfn ""

type OrderData = {
    OrderId: string
    Date: DateTime
    CustomerId: string
    Product: string
    Category: string
    Quantity: int
    UnitPrice: float
    Discount: float
    Region: string
}

// สร้างข้อมูลจำลอง
let random = Random(42)

let categories = [| "Electronics"; "Clothing"; "Food"; "Books"; "Sports" |]
let regions = [| "North"; "South"; "East"; "West"; "Central" |]

let generateOrders n =
    Array.init n (fun i ->
        let category = categories.[random.Next(categories.Length)]
        {
            OrderId = sprintf "ORD-%05d" (i + 1)
            Date = DateTime(2024, random.Next(1, 13), random.Next(1, 29))
            CustomerId = sprintf "CUST-%04d" (random.Next(1, 201))
            Product = sprintf "%s-%03d" category (random.Next(1, 51))
            Category = category
            Quantity = random.Next(1, 11)
            UnitPrice = float (random.Next(50, 5001)) / 10.0
            Discount = float (random.Next(0, 21)) / 100.0
            Region = regions.[random.Next(regions.Length)]
        })

let orders = generateOrders 1000

// แปลงเป็น DataFrame
let df = Frame.ofRecords orders
let revenue = df?UnitPrice * df?Quantity * (1.0 - df?Discount)
let dfWithRevenue = df |> Frame.addColumn "Revenue" revenue

printfn "Dataset loaded: %d orders" (df |> Frame.countRows)

// ===== Step 2: Exploratory Data Analysis =====

printfn "\n--- Exploratory Data Analysis ---"

// Revenue statistics
let revSeries = dfWithRevenue?Revenue
printfn "\nRevenue Statistics:"
printfn "  Total:   ฿%.2f" (Stats.sum revSeries)
printfn "  Mean:    ฿%.2f" (Stats.mean revSeries)
printfn "  Median:  ฿%.2f" (Stats.median revSeries)
printfn "  Std Dev: ฿%.2f" (Stats.stdDev revSeries)
printfn "  Min:     ฿%.2f" (Stats.min revSeries)
printfn "  Max:     ฿%.2f" (Stats.max revSeries)

// Category analysis
printfn "\nRevenue by Category:"
let categoryRevenue =
    dfWithRevenue
    |> Frame.groupRowsByString "Category"
    |> Frame.applyLevel fst (fun df -> df?Revenue |> Stats.sum)

// ===== Step 3: Time Series Analysis =====

printfn "\n--- Time Series Analysis ---"

// Monthly revenue
let monthlyRevenue =
    dfWithRevenue
    |> Frame.mapRowKeys (fun _ -> ())  // reset index
    |> Frame.rows
    |> Series.values
    |> Seq.groupBy (fun row -> row.GetAs<DateTime>("Date").ToString("yyyy-MM"))
    |> Seq.map (fun (month, rows) ->
        let totalRev = rows |> Seq.sumBy (fun row -> row.GetAs<float>("Revenue"))
        month, totalRev)
    |> Seq.sortBy fst
    |> Seq.toList

printfn "\nMonthly Revenue:"
for month, rev in monthlyRevenue do
    printfn "  %s: ฿%.2f" month rev

// ===== Step 4: Customer Analysis =====

printfn "\n--- Customer Analysis ---"

let customerRevenue =
    dfWithRevenue
    |> Frame.rows
    |> Series.values
    |> Seq.groupBy (fun row -> row.GetAs<string>("CustomerId"))
    |> Seq.map (fun (cust, rows) ->
        let totalRev = rows |> Seq.sumBy (fun row -> row.GetAs<float>("Revenue"))
        let orderCount = rows |> Seq.length
        cust, totalRev, orderCount)
    |> Seq.toList

let totalCustomers = customerRevenue.Length
let avgRevPerCustomer = customerRevenue |> List.averageBy (fun (_, rev, _) -> rev)
let avgOrdersPerCustomer = customerRevenue |> List.averageBy (fun (_, _, cnt) -> float cnt)

printfn "Total Customers: %d" totalCustomers
printfn "Avg Revenue/Customer: ฿%.2f" avgRevPerCustomer
printfn "Avg Orders/Customer: %.2f" avgOrdersPerCustomer

// Top 10 customers
printfn "\nTop 10 Customers by Revenue:"
customerRevenue
|> List.sortByDescending (fun (_, rev, _) -> rev)
|> List.take 10
|> List.iteri (fun i (cust, rev, cnt) ->
    printfn "  %2d. %s: ฿%.2f (%d orders)" (i+1) cust rev cnt)

// ===== Step 5: Visualization =====

// Revenue by category bar chart
let categoryData = 
    orders
    |> Array.groupBy (fun o -> o.Category)
    |> Array.map (fun (cat, ords) ->
        cat, ords |> Array.sumBy (fun o -> o.UnitPrice * float o.Quantity * (1.0 - o.Discount)))
    |> Array.sortByDescending snd

let barTrace = Bar(
    x = (categoryData |> Array.map fst),
    y = (categoryData |> Array.map snd),
    marker = Marker(color = "steelblue"),
    text = categoryData |> Array.map (fun (_, rev) -> sprintf "฿%.0f" rev),
    textposition = "outside"
)

let barLayout = Layout(
    title = "Revenue by Category",
    xaxis = Xaxis(title = "Category"),
    yaxis = Yaxis(title = "Revenue (฿)")
)

let barChart = Chart.Plot([barTrace], barLayout)
// barChart.Show()

// Monthly trend line chart
let monthlyData = monthlyRevenue |> List.toArray

let lineTrace = Scatter(
    x = (monthlyData |> Array.map fst),
    y = (monthlyData |> Array.map snd),
    mode = "lines+markers",
    line = Line(color = "#007bff", width = 3.0),
    marker = Marker(size = 8.0)
)

let lineLayout = Layout(
    title = "Monthly Revenue Trend",
    xaxis = Xaxis(title = "Month"),
    yaxis = Yaxis(title = "Revenue (฿)")
)

let lineChart = Chart.Plot([lineTrace], lineLayout)
// lineChart.Show()

// ===== Step 6: Insights =====

printfn "\n--- Key Insights ---"

let topCategory = categoryData |> Array.head
printfn "1. Top Revenue Category: %s (฿%.2f)" (fst topCategory) (snd topCategory)

let (maxMonth, maxRev) = monthlyRevenue |> List.maxBy snd
printfn "2. Best Month: %s (฿%.2f)" maxMonth maxRev

let (minMonth, minRev) = monthlyRevenue |> List.minBy snd
printfn "3. Worst Month: %s (฿%.2f)" minMonth minRev

// Growth rate
let firstMonthRev = monthlyRevenue |> List.head |> snd
let lastMonthRev = monthlyRevenue |> List.last |> snd
let growthRate = (lastMonthRev - firstMonthRev) / firstMonthRev * 100.0
printfn "4. Overall Growth: %.1f%%" growthRate

printfn "\nAnalysis complete!"
```

---

## 9. Time Series Analysis

```fsharp
// timeseries.fsx
#r "nuget: Deedle, 3.0.0"
#r "nuget: FSharp.Stats, 0.5.0"

open System
open Deedle
open FSharp.Stats

// ===== Simulate Time Series Data =====

let generateTimeSeries (start: DateTime) (days: int) =
    let rng = Random(42)
    
    Array.init days (fun i ->
        let date = start.AddDays(float i)
        let trend = float i * 0.5  // upward trend
        let seasonal = 10.0 * sin(float i * 2.0 * Math.PI / 7.0)  // weekly seasonality
        let noise = rng.NextGaussian(0.0, 2.0)
        date, 100.0 + trend + seasonal + noise)

let tsData = generateTimeSeries (DateTime(2024, 1, 1)) 365

// Create Series
let dateSeries = tsData |> Array.map fst
let valueSeries = Series.ofValues (tsData |> Array.map snd)

// ===== Moving Averages =====

// Simple Moving Average
let sma7 = valueSeries |> Stats.movingMean 7
let sma30 = valueSeries |> Stats.movingMean 30

// Exponential Moving Average (manual)
let ema (alpha: float) (series: Series<int, float>) =
    let values = series |> Series.values |> Array.ofSeq
    let result = Array.copy values
    for i in 1..values.Length-1 do
        result.[i] <- alpha * values.[i] + (1.0 - alpha) * result.[i-1]
    Series.ofValues result

let ema12 = ema 0.154 valueSeries  // 12-period EMA
let ema26 = ema 0.074 valueSeries  // 26-period EMA

// ===== Trend Analysis =====

let detectTrend (series: Series<int, float>) =
    let values = series |> Series.values |> Array.ofSeq
    let n = values.Length
    let x = Array.init n float
    
    // Linear regression
    let sumX = Array.sum x
    let sumY = Array.sum values
    let sumXY = Array.map2 (*) x values |> Array.sum
    let sumX2 = x |> Array.map (fun xi -> xi * xi) |> Array.sum
    let n' = float n
    
    let slope = (n' * sumXY - sumX * sumY) / (n' * sumX2 - sumX * sumX)
    
    if slope > 0.1 then "Uptrend"
    elif slope < -0.1 then "Downtrend"
    else "Sideways"

printfn "\nTime Series Analysis:"
printfn "Trend: %s" (detectTrend valueSeries)
printfn "Mean: %.2f" (Stats.mean valueSeries)
printfn "Std Dev: %.2f" (Stats.stdDev valueSeries)

// ===== Seasonality Detection =====

let detectSeasonality (series: Series<int, float>) (period: int) =
    let values = series |> Series.values |> Array.ofSeq
    let n = values.Length
    
    // แบ่งเป็น windows ขนาด period
    let windows = 
        [| for i in 0..period-1 do
            yield values |> Array.skip i |> Array.chunkBySize period |> Array.map Array.head |]
    
    // คำนวณ amplitude ของแต่ละ phase
    windows |> Array.map (Array.mean >> abs)

let weeklyAmplitude = detectSeasonality valueSeries 7
printfn "\nWeekly Seasonality Amplitude:"
for i, amp in weeklyAmplitude |> Array.mapi (fun i a -> (i, a)) do
    let day = 
        match i with
        | 0 -> "Mon" | 1 -> "Tue" | 2 -> "Wed" | 3 -> "Thu"
        | 4 -> "Fri" | 5 -> "Sat" | 6 -> "Sun" | _ -> "?"
    printfn "  %s: %.2f" day amp
```

---

## สรุป (Summary)

F# Data Science ecosystem ประกอบด้วย:

1. **Deedle**: Data frames สำหรับ tabular data manipulation
2. **FSharp.Stats**: Statistical computations, hypothesis testing
3. **XPlot.Plotly**: Interactive data visualizations
4. **FSharp.Data**: Data loading จาก CSV, JSON, HTML
5. **Analysis Workflow**: Load → Clean → Analyze → Visualize → Insights

F# เหมาะสำหรับ Data Science เพราะ:
- Type safety ป้องกัน data errors
- Functional programming ทำให้ pipeline data transformation สวยงาม
- Interop กับ .NET libraries ที่มีอยู่แล้วมากมาย

---

*ไปต่อที่ Part 108: Testing Patterns ขั้นสูง*
