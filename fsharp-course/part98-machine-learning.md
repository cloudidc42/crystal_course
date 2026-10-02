# Part 98 - Machine Learning กับ F#

## บทนำ

F# เป็นภาษาที่เหมาะสมสำหรับ Machine Learning เนื่องจากมี immutability, type safety, และ functional programming paradigm ที่ช่วยให้ code มีความถูกต้องและทดสอบได้ง่าย บทนี้ครอบคลุม ML.NET, DiffSharp, และ TorchSharp

---

## 1. ML.NET กับ F#

```xml
<!-- NuGet packages -->
<PackageReference Include="Microsoft.ML" Version="3.0.0" />
<PackageReference Include="Microsoft.ML.FastTree" Version="3.0.0" />
<PackageReference Include="Microsoft.ML.LightGbm" Version="3.0.0" />
<PackageReference Include="Microsoft.ML.ImageAnalytics" Version="3.0.0" />
<PackageReference Include="Microsoft.ML.TimeSeries" Version="3.0.0" />
```

### ML.NET Pipeline Overview

```fsharp
open Microsoft.ML
open Microsoft.ML.Data
open System

// ML.NET ทำงานผ่าน Pipeline:
// Data -> Transform -> Trainer -> Model -> Prediction

// สร้าง MLContext
let mlContext = MLContext(seed = 42)
```

---

## 2. Data Loading

```fsharp
// Data types
[<CLIMutable>]
type HouseData = {
    Size: float32
    Bedrooms: float32
    Bathrooms: float32
    YearBuilt: float32
    Neighborhood: string
    Price: float32
}

[<CLIMutable>]
type HousePrediction = {
    [<ColumnName("Score")>]
    PredictedPrice: float32
}

// Load from CSV
let loadCsvData (mlContext: MLContext) path =
    mlContext.Data.LoadFromTextFile<HouseData>(
        path,
        hasHeader = true,
        separatorChar = ',')

// Load from IEnumerable
let loadFromMemory (mlContext: MLContext) (data: HouseData list) =
    mlContext.Data.LoadFromEnumerable(data)

// Load from SQL (using a custom reader)
let loadFromSql connectionString query =
    // Custom implementation
    let connection = new System.Data.SqlClient.SqlConnection(connectionString)
    let cmd = connection.CreateCommand()
    cmd.CommandText <- query
    connection.Open()
    let reader = cmd.ExecuteReader()
    
    let data = System.Collections.Generic.List<HouseData>()
    while reader.Read() do
        data.Add({
            Size = reader.GetFloat(0)
            Bedrooms = float32 (reader.GetInt32(1))
            Bathrooms = float32 (reader.GetFloat(2))
            YearBuilt = float32 (reader.GetInt32(3))
            Neighborhood = reader.GetString(4)
            Price = float32 (reader.GetDecimal(5))
        })
    
    data :> seq<HouseData>

// Split data into train/test
let splitData (mlContext: MLContext) (data: IDataView) testFraction =
    let split = mlContext.Data.TrainTestSplit(data, testFraction = testFraction, seed = 42)
    split.TrainSet, split.TestSet
```

---

## 3. Feature Engineering

```fsharp
// Feature Engineering: แปลงข้อมูลให้เหมาะกับ ML models

// One-hot encoding สำหรับ categorical variables
let oneHotEncode (mlContext: MLContext) (data: IDataView) =
    let pipeline = 
        mlContext.Transforms.Categorical.OneHotEncoding("NeighborhoodEncoded", "Neighborhood")
    
    pipeline.Fit(data).Transform(data)

// Normalization
let normalizeFeatures (mlContext: MLContext) (data: IDataView) featureColumn =
    mlContext.Transforms.NormalizeMinMax(featureColumn)
    |> fun p -> p.Fit(data).Transform(data)

// Feature concatenation
let concatenateFeatures (mlContext: MLContext) (inputColumns: string[]) outputColumn =
    mlContext.Transforms.Concatenate(outputColumn, inputColumns)

// Complete feature engineering pipeline
let buildFeaturePipeline (mlContext: MLContext) =
    mlContext.Transforms.Categorical.OneHotEncoding("NeighborhoodEncoded", "Neighborhood")
    |> fun p ->
        p.Append(mlContext.Transforms.Concatenate(
            "Features",
            [| "Size"; "Bedrooms"; "Bathrooms"; "YearBuilt"; "NeighborhoodEncoded" |]))
    |> fun p ->
        p.Append(mlContext.Transforms.NormalizeMinMax("Features"))

// Custom transformation
let addFeatures (mlContext: MLContext) =
    // Feature crossing: combine features
    mlContext.Transforms.CustomMapping<{| Size: float32; Bedrooms: float32 |}, {| SizePerBedroom: float32 |}>(
        fun input -> {| SizePerBedroom = if input.Bedrooms > 0.0f then input.Size / input.Bedrooms else 0.0f |},
        contractName = "SizePerBedroom")

// Handling missing values
let handleMissingValues (mlContext: MLContext) (columns: string[]) =
    mlContext.Transforms.ReplaceMissingValues(
        columns |> Array.map (fun col -> InputOutputColumnPair(col, col)),
        replacementMode = MissingValueReplacingEstimator.ReplacementMode.Mean)
```

---

## 4. Binary Classification

```fsharp
// Binary Classification: Spam Detection

[<CLIMutable>]
type EmailData = {
    Text: string
    WordCount: float32
    HasLinks: bool
    FromKnownDomain: bool
    IsSpam: bool
}

[<CLIMutable>]
type SpamPrediction = {
    [<ColumnName("PredictedLabel")>]
    IsSpam: bool
    [<ColumnName("Probability")>]
    Probability: float32
    [<ColumnName("Score")>]
    Score: float32
}

let trainSpamDetector (mlContext: MLContext) (data: IDataView) =
    // Feature pipeline
    let featurePipeline =
        mlContext.Transforms.Text.FeaturizeText("TextFeatures", "Text")
        |> fun p ->
            p.Append(mlContext.Transforms.Concatenate(
                "Features",
                [| "TextFeatures"; "WordCount" |]))
    
    // Trainer
    let trainer = mlContext.BinaryClassification.Trainers.LightGbm(
        labelColumnName = "IsSpam",
        featureColumnName = "Features",
        numberOfLeaves = 50,
        numberOfIterations = 100,
        learningRate = 0.1)
    
    // Full pipeline
    let pipeline = featurePipeline.Append(trainer)
    
    // Train
    let model = pipeline.Fit(data)
    model

// Evaluate binary classifier
let evaluateBinaryClassifier (mlContext: MLContext) (model: ITransformer) (testData: IDataView) =
    let predictions = model.Transform(testData)
    let metrics = mlContext.BinaryClassification.Evaluate(
        predictions,
        labelColumnName = "IsSpam")
    
    {|
        Accuracy = metrics.Accuracy
        AUC = metrics.AreaUnderRocCurve
        F1Score = metrics.F1Score
        Precision = metrics.PositivePrecision
        Recall = metrics.PositiveRecall
    |}

// Predict
let predictSpam (mlContext: MLContext) (model: ITransformer) (email: EmailData) =
    let engine = mlContext.Model.CreatePredictionEngine<EmailData, SpamPrediction>(model)
    engine.Predict(email)
```

---

## 5. Multi-class Classification

```fsharp
// Multi-class Classification: Sentiment Analysis

[<CLIMutable>]
type ReviewData = {
    ReviewText: string
    Rating: float32  // 1-5 stars
    Sentiment: string  // "positive", "negative", "neutral"
}

[<CLIMutable>]
type SentimentPrediction = {
    [<ColumnName("PredictedLabel")>]
    PredictedSentiment: string
    [<ColumnName("Score")>]
    Scores: float32[]
}

let trainSentimentClassifier (mlContext: MLContext) (data: IDataView) =
    let pipeline =
        mlContext.Transforms.Conversion.MapValueToKey(
            "Label", "Sentiment")
        |> fun p ->
            p.Append(mlContext.Transforms.Text.FeaturizeText("TextFeatures", "ReviewText"))
        |> fun p ->
            p.Append(mlContext.Transforms.Concatenate("Features", [|"TextFeatures"; "Rating"|]))
        |> fun p ->
            p.Append(mlContext.MulticlassClassification.Trainers.LightGbm(
                labelColumnName = "Label",
                featureColumnName = "Features",
                numberOfLeaves = 31,
                numberOfIterations = 150,
                learningRate = 0.1))
        |> fun p ->
            p.Append(mlContext.Transforms.Conversion.MapKeyToValue(
                "PredictedLabel", "PredictedLabel"))
    
    pipeline.Fit(data)

let evaluateMulticlass (mlContext: MLContext) model (testData: IDataView) =
    let predictions = model.Transform(testData)
    let metrics = mlContext.MulticlassClassification.Evaluate(
        predictions,
        labelColumnName = "Label")
    
    {|
        MacroAccuracy = metrics.MacroAccuracy
        MicroAccuracy = metrics.MicroAccuracy
        TopKAccuracy = metrics.TopKAccuracy
        LogLoss = metrics.LogLoss
    |}
```

---

## 6. Regression

```fsharp
// Regression: House Price Prediction

let trainHousePriceModel (mlContext: MLContext) (data: IDataView) =
    let pipeline =
        mlContext.Transforms.Categorical.OneHotEncoding("NeighborhoodEncoded", "Neighborhood")
        |> fun p ->
            p.Append(mlContext.Transforms.Concatenate(
                "Features",
                [|"Size"; "Bedrooms"; "Bathrooms"; "YearBuilt"; "NeighborhoodEncoded"|]))
        |> fun p ->
            p.Append(mlContext.Transforms.NormalizeMinMax("Features"))
        |> fun p ->
            p.Append(mlContext.Regression.Trainers.FastTree(
                labelColumnName = "Price",
                featureColumnName = "Features",
                numberOfLeaves = 50,
                numberOfTrees = 100,
                minimumExampleCountPerLeaf = 5,
                learningRate = 0.1))
    
    pipeline.Fit(data)

let evaluateRegression (mlContext: MLContext) model (testData: IDataView) =
    let predictions = model.Transform(testData)
    let metrics = mlContext.Regression.Evaluate(
        predictions,
        labelColumnName = "Price",
        scoreColumnName = "Score")
    
    {|
        RSquared = metrics.RSquared
        RMSE = metrics.RootMeanSquaredError
        MAE = metrics.MeanAbsoluteError
        MSE = metrics.MeanSquaredError
    |}

// Neural network regression
let trainNeuralNetRegression (mlContext: MLContext) (data: IDataView) =
    let pipeline =
        mlContext.Transforms.Categorical.OneHotEncoding("NeighborhoodEncoded", "Neighborhood")
        |> fun p ->
            p.Append(mlContext.Transforms.Concatenate("Features",
                [|"Size"; "Bedrooms"; "Bathrooms"; "YearBuilt"; "NeighborhoodEncoded"|]))
        |> fun p ->
            p.Append(mlContext.Transforms.NormalizeMinMax("Features"))
        |> fun p ->
            p.Append(mlContext.Regression.Trainers.Sdca(
                labelColumnName = "Price",
                featureColumnName = "Features",
                maximumNumberOfIterations = 100))
    
    pipeline.Fit(data)
```

---

## 7. Clustering

```fsharp
// Clustering: Customer Segmentation

[<CLIMutable>]
type CustomerData = {
    CustomerId: string
    TotalPurchases: float32
    AverageOrderValue: float32
    PurchaseFrequency: float32
    DaysSinceLastPurchase: float32
}

[<CLIMutable>]
type CustomerCluster = {
    [<ColumnName("PredictedLabel")>]
    ClusterId: uint32
    [<ColumnName("Score")>]
    Distances: float32[]
}

let clusterCustomers (mlContext: MLContext) (data: IDataView) numClusters =
    let pipeline =
        mlContext.Transforms.Concatenate(
            "Features",
            [|"TotalPurchases"; "AverageOrderValue"; "PurchaseFrequency"; "DaysSinceLastPurchase"|])
        |> fun p ->
            p.Append(mlContext.Transforms.NormalizeMinMax("Features"))
        |> fun p ->
            p.Append(mlContext.Clustering.Trainers.KMeans(
                featureColumnName = "Features",
                numberOfClusters = numClusters,
                optimizationTolerance = 1e-6f,
                maximumNumberOfIterations = 1000))
    
    pipeline.Fit(data)

let evaluateClustering (mlContext: MLContext) model (testData: IDataView) =
    let predictions = model.Transform(testData)
    let metrics = mlContext.Clustering.Evaluate(
        predictions,
        scoreColumnName = "Score",
        featureColumnName = "Features")
    
    {|
        AverageDistance = metrics.AverageDistance
        DaviesBouldinIndex = metrics.DaviesBouldinIndex
        NormalizedMutualInformation = metrics.NormalizedMutualInformation
    |}

// Predict customer segment
let predictCustomerSegment (mlContext: MLContext) model (customer: CustomerData) =
    let engine = mlContext.Model.CreatePredictionEngine<CustomerData, CustomerCluster>(model)
    let prediction = engine.Predict(customer)
    
    {|
        Segment = prediction.ClusterId
        Distances = prediction.Distances
    |}
```

---

## 8. Model Training และ Cross-Validation

```fsharp
// Cross-validation
let crossValidate (mlContext: MLContext) (pipeline: IEstimator<ITransformer>) (data: IDataView) numFolds =
    let cvResults = mlContext.Regression.CrossValidate(
        data,
        pipeline,
        numberOfFolds = numFolds,
        labelColumnName = "Price")
    
    let metrics = cvResults |> Array.map (fun cv -> cv.Metrics)
    
    {|
        MeanRSquared = metrics |> Array.averageBy (fun m -> m.RSquared)
        StdRSquared = 
            let mean = metrics |> Array.averageBy (fun m -> m.RSquared)
            let variance = metrics |> Array.averageBy (fun m -> (m.RSquared - mean) ** 2.0)
            sqrt variance
        MeanRMSE = metrics |> Array.averageBy (fun m -> m.RootMeanSquaredError)
    |}

// Hyperparameter tuning (manual grid search)
let gridSearchRegression (mlContext: MLContext) (data: IDataView) =
    let trainData, validData = splitData mlContext data 0.2
    
    let hyperparams = [
        (10, 50, 0.1)   // numberOfLeaves, numberOfTrees, learningRate
        (20, 100, 0.05)
        (50, 200, 0.01)
    ]
    
    let results = 
        hyperparams
        |> List.map (fun (leaves, trees, lr) ->
            let pipeline =
                mlContext.Transforms.Concatenate("Features", [|"Size"; "Bedrooms"; "Price"|])
                |> fun p ->
                    p.Append(mlContext.Regression.Trainers.FastTree(
                        labelColumnName = "Price",
                        numberOfLeaves = leaves,
                        numberOfTrees = trees,
                        learningRate = lr))
            
            let model = pipeline.Fit(trainData)
            let predictions = model.Transform(validData)
            let metrics = mlContext.Regression.Evaluate(predictions, labelColumnName = "Price")
            
            {| Leaves = leaves; Trees = trees; LR = lr; RSquared = metrics.RSquared; RMSE = metrics.RootMeanSquaredError |})
    
    // Best model
    results |> List.maxBy (fun r -> r.RSquared)
```

---

## 9. Model Evaluation

```fsharp
// Model Evaluation ละเอียด

// Confusion Matrix
let analyzeClassification (mlContext: MLContext) model (testData: IDataView) =
    let predictions = model.Transform(testData)
    let metrics = mlContext.BinaryClassification.Evaluate(predictions, labelColumnName = "Label")
    
    printfn "=== Binary Classification Metrics ==="
    printfn "Accuracy:          %.4f" metrics.Accuracy
    printfn "AUC (ROC):         %.4f" metrics.AreaUnderRocCurve
    printfn "AUC (PR):          %.4f" metrics.AreaUnderPrecisionRecallCurve
    printfn "F1 Score:          %.4f" metrics.F1Score
    printfn "Precision:         %.4f" metrics.PositivePrecision
    printfn "Recall:            %.4f" metrics.PositiveRecall
    printfn "Log Loss:          %.4f" metrics.LogLoss
    
    printfn "\nConfusion Matrix:"
    let cm = metrics.ConfusionMatrix
    printfn "                  Predicted"
    printfn "                  Negative  Positive"
    printfn "Actual Negative:  %8d  %8d" cm.Counts.[0].[0] cm.Counts.[0].[1]
    printfn "Actual Positive:  %8d  %8d" cm.Counts.[1].[0] cm.Counts.[1].[1]

// Feature importance
let getFeatureImportance (mlContext: MLContext) (model: ITransformer) (data: IDataView) =
    let linearModel = model :?> ISingleFeaturePredictionTransformer<obj>
    // Feature importance depends on model type
    // For FastTree, use permutation feature importance
    
    let pfi = mlContext.Regression.PermutationFeatureImportance(
        linearModel,
        data,
        labelColumnName = "Price",
        numberOfExamplesToUse = 1000)
    
    pfi
    |> Seq.map (fun kv -> kv.Key, kv.Value.RSquared.Mean)
    |> Seq.sortByDescending snd
    |> Seq.toList
```

---

## 10. Model Saving และ Loading

```fsharp
// Save model
let saveModel (mlContext: MLContext) (model: ITransformer) (data: IDataView) path =
    let schema = data.Schema
    mlContext.Model.Save(model, schema, path)
    printfn "Model saved to: %s" path

// Load model
let loadModel (mlContext: MLContext) path =
    let mutable schema = Unchecked.defaultof<DataViewSchema>
    let model = mlContext.Model.Load(path, ref schema)
    model, schema

// Model versioning
type ModelVersion = {
    Version: int
    TrainedAt: DateTime
    Metrics: {| RSquared: float; RMSE: float |}
    ModelPath: string
    DataVersion: string
}

let saveVersionedModel mlContext model data version metrics dataVersion =
    let modelPath = $"models/house_price_v{version}.zip"
    saveModel mlContext model data modelPath
    
    {
        Version = version
        TrainedAt = DateTime.UtcNow
        Metrics = metrics
        ModelPath = modelPath
        DataVersion = dataVersion
    }

// Prediction Engine pool (thread-safe)
type ModelService<'TInput, 'TPrediction when 'TInput: (new: unit -> 'TInput) and 'TInput: not struct and 'TPrediction: (new: unit -> 'TPrediction) and 'TPrediction: not struct>(mlContext: MLContext, modelPath: string) =
    let model, _ = loadModel mlContext modelPath
    let predictionEnginePool = 
        mlContext.Model.CreatePredictionEnginePool<'TInput, 'TPrediction>(model)
    
    member _.Predict(input: 'TInput) =
        predictionEnginePool.GetPredictionEngine().Predict(input)
    
    interface System.IDisposable with
        member _.Dispose() = predictionEnginePool.Dispose()
```

---

## 11. DiffSharp สำหรับ Deep Learning

```xml
<!-- DiffSharp - Automatic Differentiation -->
<PackageReference Include="DiffSharp-lite" Version="1.0.7" />
```

```fsharp
open DiffSharp
open DiffSharp.Model
open DiffSharp.Optim

// สร้าง Neural Network ด้วย DiffSharp
type LinearModel(inputSize: int, outputSize: int) =
    inherit Model()
    
    let fc = Linear(inputSize, outputSize)
    
    do base.register()
    
    override _.forward(x) = fc.forward(x)

// Multi-layer Neural Network
type MLP(sizes: int list) =
    inherit Model()
    
    let layers = 
        sizes 
        |> List.pairwise
        |> List.mapi (fun i (inputSize, outputSize) ->
            Linear(inputSize, outputSize))
    
    do base.register()
    
    override _.forward(x) =
        layers |> List.fold (fun acc layer ->
            if acc = x then layer.forward(acc)
            else dsharp.relu(layer.forward(acc))) x

// Training loop
let trainModel (model: Model) (xTrain: Tensor) (yTrain: Tensor) epochs learningRate =
    let optimizer = Adam(model, lr = dsharp.tensor learningRate)
    
    for epoch in 1..epochs do
        model.mode(Mode.Train)
        
        let yPred = model.forward(xTrain)
        let loss = dsharp.mseLoss(yPred, yTrain)
        
        model.reverseDiff()
        loss.reverse()
        optimizer.step()
        model.zeroGrad()
        
        if epoch % 100 = 0 then
            printfn "Epoch %d: Loss = %.6f" epoch (loss.toFloat32())

// Autograd example
let autogradExample () =
    dsharp.config(backend = Backend.Reference, device = Device.CPU)
    
    let x = dsharp.tensor([|1.0f; 2.0f; 3.0f|], requiresGrad = true)
    let w = dsharp.tensor([|0.5f; -1.0f; 2.0f|], requiresGrad = true)
    
    let y = (x * w).sum()
    y.reverse()
    
    printfn "Gradient of x: %A" x.grad
    printfn "Gradient of w: %A" w.grad
```

---

## 12. TorchSharp สำหรับ Deep Learning

```xml
<PackageReference Include="TorchSharp" Version="0.102.0" />
<PackageReference Include="libtorch-cpu" Version="2.1.0" />
```

```fsharp
open TorchSharp
open TorchSharp.Modules

// Simple Neural Network with TorchSharp
type SimpleNet() =
    inherit nn.Module<Tensor, Tensor>("SimpleNet")
    
    let fc1 = nn.Linear(784L, 256L)
    let fc2 = nn.Linear(256L, 128L)
    let fc3 = nn.Linear(128L, 10L)
    let relu = nn.ReLU()
    let dropout = nn.Dropout(0.5)
    
    do base.RegisterComponents()
    
    override _.forward(x) =
        x
        |> fc1.forward
        |> relu.forward
        |> dropout.forward
        |> fc2.forward
        |> relu.forward
        |> fc3.forward

// Training with TorchSharp
let trainTorchNet (net: SimpleNet) (xTrain: Tensor) (yTrain: Tensor) epochs =
    let optimizer = optim.Adam(net.parameters(), lr = 0.001)
    let lossFn = nn.CrossEntropyLoss()
    
    net.train() |> ignore
    
    for epoch in 1..epochs do
        optimizer.zero_grad()
        
        let outputs = net.forward(xTrain)
        let loss = lossFn.forward(outputs, yTrain)
        
        loss.backward()
        optimizer.step() |> ignore
        
        if epoch % 10 = 0 then
            printfn "Epoch %d: Loss = %.4f" epoch (loss.item<float32>())

// Convolutional Neural Network
type CNN() =
    inherit nn.Module<Tensor, Tensor>("CNN")
    
    let conv1 = nn.Conv2d(1L, 32L, kernelSize = 3L, padding = 1L)
    let conv2 = nn.Conv2d(32L, 64L, kernelSize = 3L, padding = 1L)
    let pool = nn.MaxPool2d(kernelSize = 2L)
    let fc1 = nn.Linear(64L * 7L * 7L, 256L)
    let fc2 = nn.Linear(256L, 10L)
    let relu = nn.ReLU()
    let dropout = nn.Dropout(0.25)
    let flatten = nn.Flatten()
    
    do base.RegisterComponents()
    
    override _.forward(x) =
        x
        |> conv1.forward
        |> relu.forward
        |> pool.forward
        |> conv2.forward
        |> relu.forward
        |> pool.forward
        |> flatten.forward
        |> fc1.forward
        |> relu.forward
        |> dropout.forward
        |> fc2.forward

// Save and load TorchSharp model
let saveModel (net: SimpleNet) path =
    net.save(path) |> ignore

let loadModel path =
    let net = new SimpleNet()
    net.load(path) |> ignore
    net
```

---

## 13. Complete ML Example: Product Recommendation

```fsharp
// Complete ML Example: Product Recommendation System

open Microsoft.ML
open Microsoft.ML.Data
open Microsoft.ML.Trainers

// Data types
[<CLIMutable>]
type UserProductRating = {
    UserId: uint32
    ProductId: uint32
    Rating: float32
}

[<CLIMutable>]
type RatingPrediction = {
    [<ColumnName("Score")>]
    Rating: float32
}

// Matrix Factorization (Collaborative Filtering)
let trainRecommender (mlContext: MLContext) (data: IDataView) =
    let options = MatrixFactorizationTrainer.Options(
        MatrixColumnIndexColumnName = "UserId",
        MatrixRowIndexColumnName = "ProductId",
        LabelColumnName = "Rating",
        NumberOfIterations = 20,
        ApproximationRank = 100,
        LearningRate = 0.01,
        Lambda = 0.01)
    
    let pipeline =
        mlContext.Transforms.Conversion.MapValueToKey("UserId", "UserId")
        |> fun p ->
            p.Append(mlContext.Transforms.Conversion.MapValueToKey("ProductId", "ProductId"))
        |> fun p ->
            p.Append(mlContext.Recommendation().Trainers.MatrixFactorization(options))
    
    pipeline.Fit(data)

// Predict rating for a user-product pair
let predictRating (mlContext: MLContext) (model: ITransformer) userId productId =
    let engine = mlContext.Model.CreatePredictionEngine<UserProductRating, RatingPrediction>(model)
    let input = { UserId = uint32 userId; ProductId = uint32 productId; Rating = 0.0f }
    let prediction = engine.Predict(input)
    prediction.Rating

// Get top-N recommendations for a user
let getRecommendations (mlContext: MLContext) (model: ITransformer) userId (productIds: int list) topN =
    let engine = mlContext.Model.CreatePredictionEngine<UserProductRating, RatingPrediction>(model)
    
    productIds
    |> List.map (fun productId ->
        let input = { UserId = uint32 userId; ProductId = uint32 productId; Rating = 0.0f }
        let prediction = engine.Predict(input)
        (productId, prediction.Rating))
    |> List.sortByDescending snd
    |> List.take (min topN productIds.Length)

// Full recommendation pipeline
let buildRecommendationSystem () =
    let mlContext = MLContext(seed = 42)
    
    // Load data
    let ratings = [
        { UserId = 1u; ProductId = 1u; Rating = 5.0f }
        { UserId = 1u; ProductId = 2u; Rating = 3.0f }
        { UserId = 1u; ProductId = 4u; Rating = 1.0f }
        { UserId = 2u; ProductId = 1u; Rating = 4.0f }
        { UserId = 2u; ProductId = 3u; Rating = 5.0f }
        { UserId = 3u; ProductId = 2u; Rating = 4.0f }
        { UserId = 3u; ProductId = 3u; Rating = 2.0f }
    ]
    
    let data = mlContext.Data.LoadFromEnumerable(ratings)
    let trainData, testData = splitData mlContext data 0.2
    
    // Train
    let model = trainRecommender mlContext trainData
    
    // Evaluate
    let predictions = model.Transform(testData)
    let metrics = mlContext.Regression.Evaluate(predictions, labelColumnName = "Rating")
    
    printfn "=== Recommendation System ==="
    printfn "RMSE: %.4f" metrics.RootMeanSquaredError
    printfn "R²:   %.4f" metrics.RSquared
    
    // Predict
    let allProducts = [1..10]
    let recommendations = getRecommendations mlContext model 1 allProducts 5
    
    printfn "\nTop 5 recommendations for User 1:"
    for (productId, score) in recommendations do
        printfn "  Product %d: %.2f predicted rating" productId score
    
    // Save model
    saveModel mlContext model data "models/recommender.zip"
    
    model
```

---

## สรุป

Machine Learning ใน F#:

1. **ML.NET**: Framework สำหรับ traditional ML
   - Binary/Multi-class Classification
   - Regression
   - Clustering
   - Recommendation
   
2. **DiffSharp**: Automatic differentiation และ deep learning
   - Neural networks
   - Gradient computation
   
3. **TorchSharp**: .NET bindings สำหรับ PyTorch
   - CNN, RNN, Transformers
   - GPU acceleration

F# ทำให้ ML code:
- **Type-safe**: ไม่มี shape mismatches
- **Composable**: Pipeline style
- **Testable**: Pure functions สำหรับ transformations
- **Maintainable**: Clear data flow

---

*ต่อไป: Part 99 - โปรเจคจริง (Real-World Complete Project)*
