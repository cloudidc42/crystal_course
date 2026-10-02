# Part 97 - Cloud Services กับ F#

## บทนำ

F# สามารถทำงานกับ Cloud Services ต่างๆ ได้อย่างมีประสิทธิภาพ บทนี้จะครอบคลุม Azure SDK, AWS SDK, และ Cloud-native patterns

---

## 1. Azure SDK สำหรับ .NET

```xml
<!-- NuGet packages -->
<PackageReference Include="Azure.Storage.Blobs" Version="12.19.0" />
<PackageReference Include="Azure.Storage.Queues" Version="12.17.0" />
<PackageReference Include="Azure.Messaging.ServiceBus" Version="7.16.0" />
<PackageReference Include="Azure.Identity" Version="1.10.0" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.5.0" />
<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.20.0" />
```

### Azure Identity (Authentication)

```fsharp
open Azure.Identity
open Azure.Core

// DefaultAzureCredential - ลอง credentials หลายๆ แบบตามลำดับ
// 1. Environment variables
// 2. Workload identity (Kubernetes)
// 3. Managed identity
// 4. Azure CLI
// 5. Visual Studio
// 6. Azure PowerShell

let getCredential () =
    // ใช้ DefaultAzureCredential สำหรับ production
    DefaultAzureCredential() :> TokenCredential

let getDevelopmentCredential () =
    // ใช้ AzureCliCredential สำหรับ local development
    AzureCliCredential() :> TokenCredential

// Custom credential chain
let getCustomCredential () =
    ChainedTokenCredential(
        ManagedIdentityCredential(),
        AzureCliCredential()) :> TokenCredential
```

---

## 2. Azure Blob Storage

```fsharp
open Azure.Storage.Blobs
open Azure.Storage.Blobs.Models
open System.IO

// Create client
let createBlobServiceClient (connectionString: string) =
    BlobServiceClient(connectionString)

let createBlobServiceClientWithIdentity (accountName: string) =
    let uri = Uri($"https://{accountName}.blob.core.windows.net")
    BlobServiceClient(uri, DefaultAzureCredential())

// Container operations
let createContainerIfNotExists (client: BlobServiceClient) containerName = async {
    let containerClient = client.GetBlobContainerClient(containerName)
    let! _ = containerClient.CreateIfNotExistsAsync(PublicAccessType.None)
    return containerClient
}

// Upload blob
let uploadBlob (containerClient: BlobContainerClient) blobName (content: Stream) = async {
    let blobClient = containerClient.GetBlobClient(blobName)
    let uploadOptions = BlobUploadOptions(
        HttpHeaders = BlobHttpHeaders(ContentType = "application/octet-stream"),
        TransferOptions = StorageTransferOptions(
            MaximumConcurrency = System.Nullable(4)))
    
    let! response = blobClient.UploadAsync(content, uploadOptions)
    return response.Value
}

// Upload text
let uploadText (containerClient: BlobContainerClient) blobName (text: string) = async {
    let blobClient = containerClient.GetBlobClient(blobName)
    use stream = new MemoryStream(System.Text.Encoding.UTF8.GetBytes(text))
    let! _ = blobClient.UploadAsync(stream, overwrite = true)
    return blobClient.Uri.ToString()
}

// Download blob
let downloadBlob (containerClient: BlobContainerClient) blobName = async {
    let blobClient = containerClient.GetBlobClient(blobName)
    let! download = blobClient.DownloadContentAsync()
    return download.Value.Content.ToArray()
}

// List blobs
let listBlobs (containerClient: BlobContainerClient) prefix = async {
    let options = BlobTraits.None
    let pages = containerClient.GetBlobsAsync(options, prefix = prefix)
    let results = System.Collections.Generic.List<BlobItem>()
    
    let mutable hasMore = true
    let enumerator = pages.GetAsyncEnumerator()
    
    try
        while hasMore do
            let! more = enumerator.MoveNextAsync()
            if more then
                results.Add(enumerator.Current)
            else
                hasMore <- false
    finally
        do! enumerator.DisposeAsync()
    
    return results |> Seq.toList
}

// Delete blob
let deleteBlob (containerClient: BlobContainerClient) blobName = async {
    let blobClient = containerClient.GetBlobClient(blobName)
    let! _ = blobClient.DeleteIfExistsAsync()
    return ()
}

// Generate SAS URL
let generateSasUrl (containerClient: BlobContainerClient) blobName (duration: System.TimeSpan) =
    let blobClient = containerClient.GetBlobClient(blobName)
    let sasBuilder = Sas.BlobSasBuilder(
        BlobContainerName = containerClient.Name,
        BlobName = blobName,
        Resource = "b",
        ExpiresOn = System.DateTimeOffset.UtcNow + duration)
    sasBuilder.SetPermissions(Sas.BlobSasPermissions.Read)
    blobClient.GenerateSasUri(sasBuilder).ToString()

// ตัวอย่าง: File storage service
type IBlobStorageService =
    abstract Upload: containerName: string -> fileName: string -> content: byte[] -> Async<string>
    abstract Download: containerName: string -> fileName: string -> Async<byte[]>
    abstract Delete: containerName: string -> fileName: string -> Async<unit>
    abstract GetUrl: containerName: string -> fileName: string -> string

type AzureBlobStorageService(client: BlobServiceClient) =
    interface IBlobStorageService with
        member _.Upload containerName fileName content = async {
            let container = client.GetBlobContainerClient(containerName)
            let! _ = container.CreateIfNotExistsAsync()
            let blob = container.GetBlobClient(fileName)
            use stream = new MemoryStream(content)
            let! _ = blob.UploadAsync(stream, overwrite = true)
            return blob.Uri.ToString()
        }
        
        member _.Download containerName fileName = async {
            let container = client.GetBlobContainerClient(containerName)
            let blob = container.GetBlobClient(fileName)
            let! result = blob.DownloadContentAsync()
            return result.Value.Content.ToArray()
        }
        
        member _.Delete containerName fileName = async {
            let container = client.GetBlobContainerClient(containerName)
            let blob = container.GetBlobClient(fileName)
            let! _ = blob.DeleteIfExistsAsync()
            return ()
        }
        
        member _.GetUrl containerName fileName =
            let container = client.GetBlobContainerClient(containerName)
            let blob = container.GetBlobClient(fileName)
            blob.Uri.ToString()
```

---

## 3. Azure Queue Storage

```fsharp
open Azure.Storage.Queues
open Azure.Storage.Queues.Models

// Create queue client
let createQueueClient (connectionString: string) queueName =
    let client = QueueClient(connectionString, queueName)
    client.CreateIfNotExists() |> ignore
    client

// Send message
let sendMessage (client: QueueClient) (message: string) = async {
    let! _ = client.SendMessageAsync(message)
    return ()
}

// Send message with delay
let sendDelayedMessage (client: QueueClient) message (delay: System.TimeSpan) = async {
    let! _ = client.SendMessageAsync(message, visibilityTimeout = System.Nullable delay)
    return ()
}

// Receive messages
let receiveMessages (client: QueueClient) maxMessages = async {
    let! messages = client.ReceiveMessagesAsync(maxMessages)
    return messages.Value |> Array.toList
}

// Process and delete message
let processMessage (client: QueueClient) (message: QueueMessage) (handler: string -> Async<unit>) = async {
    try
        do! handler message.MessageText
        let! _ = client.DeleteMessageAsync(message.MessageId, message.PopReceipt)
        return Ok ()
    with ex ->
        return Error ex.Message
}

// Peek at messages (without dequeuing)
let peekMessages (client: QueueClient) maxMessages = async {
    let! messages = client.PeekMessagesAsync(maxMessages)
    return messages.Value |> Array.toList
}

// Get queue properties
let getQueueProperties (client: QueueClient) = async {
    let! props = client.GetPropertiesAsync()
    return {|
        MessageCount = props.Value.ApproximateMessagesCount
    |}
}

// Message processor with retry
let processWithRetry (client: QueueClient) maxRetries handler = async {
    let! messages = receiveMessages client 5
    
    for msg in messages do
        let dequeueCount = msg.DequeueCount
        
        if dequeueCount > maxRetries then
            // Move to dead-letter queue
            printfn "Message exceeded retry limit: %s" msg.MessageText
            let! _ = client.DeleteMessageAsync(msg.MessageId, msg.PopReceipt)
            ()
        else
            let! result = processMessage client msg handler
            match result with
            | Ok () -> ()
            | Error err ->
                printfn "Error processing message (attempt %d): %s" dequeueCount err
}
```

---

## 4. Azure Service Bus

```fsharp
open Azure.Messaging.ServiceBus

// Create Service Bus client
let createServiceBusClient (connectionString: string) =
    ServiceBusClient(connectionString)

let createServiceBusClientWithIdentity (fullyQualifiedNamespace: string) =
    ServiceBusClient(fullyQualifiedNamespace, DefaultAzureCredential())

// Send message to queue/topic
let sendServiceBusMessage (client: ServiceBusClient) queueOrTopic (message: obj) = async {
    use sender = client.CreateSender(queueOrTopic)
    let json = System.Text.Json.JsonSerializer.Serialize(message)
    let sbMessage = ServiceBusMessage(json)
    sbMessage.ContentType <- "application/json"
    sbMessage.CorrelationId <- System.Guid.NewGuid().ToString()
    
    do! sender.SendMessageAsync(sbMessage) |> Async.AwaitTask
    return sbMessage.MessageId
}

// Send batch of messages
let sendBatch (client: ServiceBusClient) queueName (messages: obj list) = async {
    use sender = client.CreateSender(queueName)
    use! batch = sender.CreateMessageBatchAsync() |> Async.AwaitTask
    
    for msg in messages do
        let json = System.Text.Json.JsonSerializer.Serialize(msg)
        let sbMessage = ServiceBusMessage(json)
        if not (batch.TryAddMessage(sbMessage)) then
            // Batch full, send it and create new batch
            do! sender.SendMessagesAsync(batch) |> Async.AwaitTask
    
    do! sender.SendMessagesAsync(batch) |> Async.AwaitTask
}

// Process messages
let processMessages (client: ServiceBusClient) queueName handler = async {
    use processor = client.CreateProcessor(queueName, ServiceBusProcessorOptions(
        MaxConcurrentCalls = 4,
        AutoCompleteMessages = false,
        MaxAutoLockRenewalDuration = System.TimeSpan.FromMinutes(5.0)))
    
    processor.ProcessMessageAsync.Add(fun args -> task {
        try
            let body = args.Message.Body.ToString()
            do! handler body
            do! args.CompleteMessageAsync(args.Message)
        with ex ->
            printfn "Error processing: %s" ex.Message
            do! args.AbandonMessageAsync(args.Message)
    })
    
    processor.ProcessErrorAsync.Add(fun args -> task {
        printfn "Error: %s" args.Exception.Message
    })
    
    do! processor.StartProcessingAsync() |> Async.AwaitTask
    do! Async.Sleep(System.Threading.Timeout.Infinite)
    do! processor.StopProcessingAsync() |> Async.AwaitTask
}

// Topics and Subscriptions
let sendToTopic (client: ServiceBusClient) topicName message = async {
    use sender = client.CreateSender(topicName)
    let json = System.Text.Json.JsonSerializer.Serialize(message)
    let sbMessage = ServiceBusMessage(json)
    do! sender.SendMessageAsync(sbMessage) |> Async.AwaitTask
}

let receiveFromSubscription (client: ServiceBusClient) topicName subscriptionName = async {
    use receiver = client.CreateReceiver(topicName, subscriptionName)
    let! messages = receiver.ReceiveMessagesAsync(10) |> Async.AwaitTask
    return messages |> Seq.toList
}
```

---

## 5. Azure Functions กับ F#

```fsharp
// Azure Functions isolated process model

// <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.20.0" />
// <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http" Version="3.1.0" />
// <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.ServiceBus" Version="5.13.0" />
// <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Timer" Version="4.3.0" />

open Microsoft.Azure.Functions.Worker
open Microsoft.Azure.Functions.Worker.Http
open Microsoft.Extensions.Logging
open System.Net

// HTTP Trigger
type HttpFunctions(logger: ILogger<HttpFunctions>) =
    
    [<Function("GetUsers")>]
    member _.GetUsers([<HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "users")>] req: HttpRequestData) =
        let response = req.CreateResponse(HttpStatusCode.OK)
        response.Headers.Add("Content-Type", "application/json")
        
        let users = [
            {| Id = 1; Name = "Alice" |}
            {| Id = 2; Name = "Bob" |}
        ]
        
        response.WriteAsJsonAsync(users)
        |> Async.AwaitTask
        |> Async.map (fun _ -> response)
        |> Async.StartAsTask
    
    [<Function("CreateUser")>]
    member _.CreateUser([<HttpTrigger(AuthorizationLevel.Function, "post", Route = "users")>] req: HttpRequestData) = task {
        let! body = req.ReadAsStringAsync()
        
        logger.LogInformation("Creating user: {Body}", body)
        
        let response = req.CreateResponse(HttpStatusCode.Created)
        response.Headers.Add("Content-Type", "application/json")
        do! response.WriteAsJsonAsync({| Id = System.Guid.NewGuid(); Message = "Created" |})
        return response
    }

// Timer Trigger
type TimerFunctions(logger: ILogger<TimerFunctions>) =
    
    [<Function("DailyCleanup")>]
    member _.Cleanup([<TimerTrigger("0 0 2 * * *")>] timer: TimerInfo) =
        logger.LogInformation("Running daily cleanup at {Time}", System.DateTime.UtcNow)
        // Do cleanup work

// Service Bus Trigger
type ServiceBusFunctions(logger: ILogger<ServiceBusFunctions>) =
    
    [<Function("ProcessOrder")>]
    member _.ProcessOrder([<ServiceBusTrigger("orders-queue", Connection = "ServiceBusConnection")>] message: string) = task {
        logger.LogInformation("Processing order: {Message}", message)
        // Process the order message
    }

// Blob Trigger
type BlobFunctions(logger: ILogger<BlobFunctions>) =
    
    [<Function("ProcessUpload")>]
    member _.ProcessUpload([<BlobTrigger("uploads/{name}", Connection = "AzureWebJobsStorage")>] stream: System.IO.Stream, name: string) = task {
        logger.LogInformation("Processing upload: {Name}", name)
        // Process the uploaded file
    }

// Program.fs
module AzureFunctionsProgram =
    open Microsoft.Extensions.Hosting
    open Microsoft.Extensions.DependencyInjection
    
    [<EntryPoint>]
    let main _ =
        let host =
            HostBuilder()
                .ConfigureFunctionsWorkerDefaults()
                .ConfigureServices(fun ctx services ->
                    services.AddSingleton<IMyService, MyService>() |> ignore)
                .Build()
        
        host.Run()
        0
```

---

## 6. AWS SDK กับ F#

```xml
<!-- AWS SDK packages -->
<PackageReference Include="AWSSDK.S3" Version="3.7.0" />
<PackageReference Include="AWSSDK.SQS" Version="3.7.0" />
<PackageReference Include="AWSSDK.Lambda" Version="3.7.0" />
<PackageReference Include="AWSSDK.SecretsManager" Version="3.7.0" />
```

---

## 7. AWS S3

```fsharp
open Amazon.S3
open Amazon.S3.Model
open System.IO

// Create S3 client
let createS3Client (region: Amazon.RegionEndpoint) =
    new AmazonS3Client(region)

// Upload object
let uploadToS3 (client: IAmazonS3) bucketName key (content: byte[]) = async {
    let request = PutObjectRequest(
        BucketName = bucketName,
        Key = key,
        InputStream = new MemoryStream(content),
        ContentType = "application/octet-stream")
    
    request.ServerSideEncryptionMethod <- ServerSideEncryptionMethod.AES256
    
    let! response = client.PutObjectAsync(request) |> Async.AwaitTask
    return response.HttpStatusCode
}

// Upload stream
let uploadStreamToS3 (client: IAmazonS3) bucketName key (stream: Stream) = async {
    let request = PutObjectRequest(
        BucketName = bucketName,
        Key = key,
        InputStream = stream)
    
    let! _ = client.PutObjectAsync(request) |> Async.AwaitTask
    return $"s3://{bucketName}/{key}"
}

// Download object
let downloadFromS3 (client: IAmazonS3) bucketName key = async {
    let request = GetObjectRequest(BucketName = bucketName, Key = key)
    
    use! response = client.GetObjectAsync(request) |> Async.AwaitTask
    use reader = new StreamReader(response.ResponseStream)
    return! reader.ReadToEndAsync() |> Async.AwaitTask
}

// List objects
let listS3Objects (client: IAmazonS3) bucketName prefix = async {
    let request = ListObjectsV2Request(
        BucketName = bucketName,
        Prefix = prefix)
    
    let! response = client.ListObjectsV2Async(request) |> Async.AwaitTask
    return response.S3Objects |> Seq.map (fun o -> {| Key = o.Key; Size = o.Size; Modified = o.LastModified |}) |> Seq.toList
}

// Delete object
let deleteFromS3 (client: IAmazonS3) bucketName key = async {
    let request = DeleteObjectRequest(BucketName = bucketName, Key = key)
    let! _ = client.DeleteObjectAsync(request) |> Async.AwaitTask
    return ()
}

// Generate pre-signed URL
let generatePresignedUrl (client: AmazonS3Client) bucketName key (expiry: System.TimeSpan) =
    let request = GetPreSignedUrlRequest(
        BucketName = bucketName,
        Key = key,
        Expires = System.DateTime.UtcNow + expiry,
        Protocol = Protocol.HTTPS)
    
    client.GetPreSignedURL(request)

// Multipart upload สำหรับไฟล์ขนาดใหญ่
let uploadLargeFile (client: IAmazonS3) bucketName key (filePath: string) = async {
    let fileInfo = FileInfo(filePath)
    let partSize = 5L * 1024L * 1024L  // 5 MB parts
    
    // Initiate multipart upload
    let initRequest = InitiateMultipartUploadRequest(
        BucketName = bucketName,
        Key = key)
    
    let! initResponse = client.InitiateMultipartUploadAsync(initRequest) |> Async.AwaitTask
    let uploadId = initResponse.UploadId
    
    try
        let partETags = System.Collections.Generic.List<PartETag>()
        use fileStream = File.OpenRead(filePath)
        
        let mutable partNumber = 1
        let mutable filePosition = 0L
        
        while filePosition < fileInfo.Length do
            let size = min partSize (fileInfo.Length - filePosition)
            
            let uploadRequest = UploadPartRequest(
                BucketName = bucketName,
                Key = key,
                UploadId = uploadId,
                PartNumber = partNumber,
                PartSize = size,
                FileStream = fileStream)
            
            let! response = client.UploadPartAsync(uploadRequest) |> Async.AwaitTask
            partETags.Add(PartETag(PartNumber = partNumber, ETag = response.ETag))
            
            filePosition <- filePosition + size
            partNumber <- partNumber + 1
        
        // Complete multipart upload
        let completeRequest = CompleteMultipartUploadRequest(
            BucketName = bucketName,
            Key = key,
            UploadId = uploadId,
            PartETags = partETags)
        
        let! _ = client.CompleteMultipartUploadAsync(completeRequest) |> Async.AwaitTask
        return Ok $"s3://{bucketName}/{key}"
        
    with ex ->
        // Abort multipart upload on error
        let abortRequest = AbortMultipartUploadRequest(
            BucketName = bucketName,
            Key = key,
            UploadId = uploadId)
        let! _ = client.AbortMultipartUploadAsync(abortRequest) |> Async.AwaitTask
        return Error ex.Message
}
```

---

## 8. AWS SQS

```fsharp
open Amazon.SQS
open Amazon.SQS.Model

// Create SQS client
let createSqsClient (region: Amazon.RegionEndpoint) =
    new AmazonSQSClient(region)

// Send message
let sendSqsMessage (client: IAmazonSQS) queueUrl (message: string) = async {
    let request = SendMessageRequest(
        QueueUrl = queueUrl,
        MessageBody = message,
        DelaySeconds = 0)
    
    let! response = client.SendMessageAsync(request) |> Async.AwaitTask
    return response.MessageId
}

// Send message with attributes
let sendSqsMessageWithAttributes (client: IAmazonSQS) queueUrl message attributes = async {
    let request = SendMessageRequest(
        QueueUrl = queueUrl,
        MessageBody = message)
    
    for (key, value) in attributes do
        request.MessageAttributes.[key] <- MessageAttributeValue(
            DataType = "String",
            StringValue = value)
    
    let! response = client.SendMessageAsync(request) |> Async.AwaitTask
    return response.MessageId
}

// Receive messages
let receiveSqsMessages (client: IAmazonSQS) queueUrl maxMessages = async {
    let request = ReceiveMessageRequest(
        QueueUrl = queueUrl,
        MaxNumberOfMessages = maxMessages,
        WaitTimeSeconds = 20,  // Long polling
        VisibilityTimeout = 30,
        MessageAttributeNames = ["All"])
    
    let! response = client.ReceiveMessageAsync(request) |> Async.AwaitTask
    return response.Messages |> Seq.toList
}

// Delete message after processing
let deleteSqsMessage (client: IAmazonSQS) queueUrl receiptHandle = async {
    let request = DeleteMessageRequest(
        QueueUrl = queueUrl,
        ReceiptHandle = receiptHandle)
    
    let! _ = client.DeleteMessageAsync(request) |> Async.AwaitTask
    return ()
}

// Worker loop
let sqsWorkerLoop (client: IAmazonSQS) queueUrl handler cancellationToken = async {
    while not (System.Threading.CancellationToken.IsCancellationRequested) do
        let! messages = receiveSqsMessages client queueUrl 10
        
        for msg in messages do
            try
                do! handler msg.Body
                do! deleteSqsMessage client queueUrl msg.ReceiptHandle
            with ex ->
                printfn "Error processing message: %s" ex.Message
                // Message will become visible again after VisibilityTimeout
}

// FIFO Queue
let sendFifoMessage (client: IAmazonSQS) queueUrl messageGroupId message = async {
    let request = SendMessageRequest(
        QueueUrl = queueUrl,
        MessageBody = message,
        MessageGroupId = messageGroupId,
        MessageDeduplicationId = System.Guid.NewGuid().ToString())
    
    let! response = client.SendMessageAsync(request) |> Async.AwaitTask
    return response.MessageId
}
```

---

## 9. AWS Secrets Manager

```fsharp
open Amazon.SecretsManager
open Amazon.SecretsManager.Model

// Create client
let createSecretsClient (region: Amazon.RegionEndpoint) =
    new AmazonSecretsManagerClient(region)

// Get secret value
let getSecret (client: IAmazonSecretsManager) secretName = async {
    let request = GetSecretValueRequest(SecretId = secretName)
    
    try
        let! response = client.GetSecretValueAsync(request) |> Async.AwaitTask
        return Ok response.SecretString
    with
    | :? ResourceNotFoundException ->
        return Error $"Secret not found: {secretName}"
    | ex ->
        return Error ex.Message
}

// Get JSON secret
let getJsonSecret<'T> (client: IAmazonSecretsManager) secretName = async {
    let! result = getSecret client secretName
    return result |> Result.map System.Text.Json.JsonSerializer.Deserialize<'T>
}

// Create secret
let createSecret (client: IAmazonSecretsManager) name value = async {
    let request = CreateSecretRequest(
        Name = name,
        SecretString = value,
        Description = $"Created at {System.DateTime.UtcNow}")
    
    let! response = client.CreateSecretAsync(request) |> Async.AwaitTask
    return response.ARN
}

// Rotate secret
let rotateSecret (client: IAmazonSecretsManager) secretName lambdaArn = async {
    let request = RotateSecretRequest(
        SecretId = secretName,
        RotationLambdaARN = lambdaArn,
        RotationRules = RotationRulesType(AutomaticallyAfterDays = System.Nullable 30L))
    
    let! _ = client.RotateSecretAsync(request) |> Async.AwaitTask
    return ()
}

// Load database config from Secrets Manager
let loadDbConfigFromSecrets (client: IAmazonSecretsManager) secretName = async {
    let! result = getJsonSecret<{| username: string; password: string; host: string; dbname: string; port: int |}> client secretName
    
    return result |> Result.map (fun secret ->
        $"Host={secret.host};Port={secret.port};Database={secret.dbname};Username={secret.username};Password={secret.password}")
}
```

---

## 10. Cloud-Native Patterns

```fsharp
// Circuit Breaker Pattern
module CircuitBreaker =
    
    type CircuitState =
        | Closed      // Normal operation
        | Open        // Failing, reject requests
        | HalfOpen    // Testing recovery
    
    type CircuitBreaker = {
        mutable State: CircuitState
        mutable FailureCount: int
        mutable LastFailureTime: System.DateTime
        FailureThreshold: int
        RecoveryTimeout: System.TimeSpan
    }
    
    let create failureThreshold recoveryTimeout = {
        State = Closed
        FailureCount = 0
        LastFailureTime = System.DateTime.MinValue
        FailureThreshold = failureThreshold
        RecoveryTimeout = recoveryTimeout
    }
    
    let execute (cb: CircuitBreaker) (operation: unit -> Async<'a>) = async {
        match cb.State with
        | Open ->
            let elapsed = System.DateTime.UtcNow - cb.LastFailureTime
            if elapsed >= cb.RecoveryTimeout then
                cb.State <- HalfOpen
                try
                    let! result = operation()
                    cb.State <- Closed
                    cb.FailureCount <- 0
                    return Ok result
                with ex ->
                    cb.State <- Open
                    cb.LastFailureTime <- System.DateTime.UtcNow
                    return Error $"Circuit still open: {ex.Message}"
            else
                return Error "Circuit breaker is open"
        
        | Closed | HalfOpen ->
            try
                let! result = operation()
                if cb.State = HalfOpen then
                    cb.State <- Closed
                    cb.FailureCount <- 0
                return Ok result
            with ex ->
                cb.FailureCount <- cb.FailureCount + 1
                cb.LastFailureTime <- System.DateTime.UtcNow
                
                if cb.FailureCount >= cb.FailureThreshold then
                    cb.State <- Open
                    printfn "Circuit breaker opened after %d failures" cb.FailureCount
                
                return Error ex.Message
    }

// Retry Pattern with Exponential Backoff
module RetryPolicy =
    
    type RetryConfig = {
        MaxRetries: int
        InitialDelay: System.TimeSpan
        MaxDelay: System.TimeSpan
        BackoffMultiplier: float
        JitterFactor: float
    }
    
    let defaultConfig = {
        MaxRetries = 3
        InitialDelay = System.TimeSpan.FromMilliseconds(100.0)
        MaxDelay = System.TimeSpan.FromSeconds(30.0)
        BackoffMultiplier = 2.0
        JitterFactor = 0.1
    }
    
    let withRetry (config: RetryConfig) (operation: unit -> Async<'a>) = async {
        let mutable attempt = 0
        let mutable lastError = None
        let mutable result = None
        
        while attempt <= config.MaxRetries && result = None do
            try
                let! r = operation()
                result <- Some r
            with ex ->
                lastError <- Some ex
                attempt <- attempt + 1
                
                if attempt <= config.MaxRetries then
                    let delay = 
                        min
                            (config.InitialDelay.TotalMilliseconds * (config.BackoffMultiplier ** float (attempt - 1)))
                            config.MaxDelay.TotalMilliseconds
                    
                    let jitter = delay * config.JitterFactor * (System.Random().NextDouble() - 0.5)
                    let totalDelay = int (delay + jitter)
                    
                    printfn "Attempt %d failed. Retrying in %d ms..." attempt totalDelay
                    do! Async.Sleep totalDelay
        
        return 
            match result with
            | Some r -> Ok r
            | None -> Error (lastError.Value.Message)
    }

// Bulkhead Pattern (limit concurrent operations)
module Bulkhead =
    
    type BulkheadPolicy(maxConcurrent: int, maxQueue: int) =
        let semaphore = new System.Threading.SemaphoreSlim(maxConcurrent, maxConcurrent)
        let queueSemaphore = new System.Threading.SemaphoreSlim(maxQueue, maxQueue)
        
        member _.Execute(operation: unit -> Async<'a>) = async {
            let mutable acquired = false
            try
                acquired <- queueSemaphore.Wait(0)  // Non-blocking check
                if not acquired then
                    return Error "Queue full - request rejected"
                
                do! semaphore.WaitAsync() |> Async.AwaitTask
                
                try
                    let! result = operation()
                    return Ok result
                finally
                    semaphore.Release() |> ignore
            finally
                if acquired then queueSemaphore.Release() |> ignore
        }
        
        interface System.IDisposable with
            member _.Dispose() =
                semaphore.Dispose()
                queueSemaphore.Dispose()

// Health check aggregator
module CloudHealthChecks =
    
    type HealthStatus = Healthy | Degraded | Unhealthy
    
    type HealthCheckResult = {
        Name: string
        Status: HealthStatus
        Message: string
        Duration: System.TimeSpan
    }
    
    let runCheck name (check: unit -> Async<unit>) = async {
        let sw = System.Diagnostics.Stopwatch.StartNew()
        try
            do! check()
            sw.Stop()
            return { Name = name; Status = Healthy; Message = "OK"; Duration = sw.Elapsed }
        with ex ->
            sw.Stop()
            return { Name = name; Status = Unhealthy; Message = ex.Message; Duration = sw.Elapsed }
    }
    
    let aggregateHealth checks = async {
        let! results = checks |> List.map (fun (name, check) -> runCheck name check) |> Async.Parallel
        
        let overallStatus = 
            if results |> Array.exists (fun r -> r.Status = Unhealthy) then Unhealthy
            elif results |> Array.exists (fun r -> r.Status = Degraded) then Degraded
            else Healthy
        
        return {|
            Status = overallStatus
            Checks = results
            Timestamp = System.DateTime.UtcNow
        |}
    }
```

---

## 11. Azure Key Vault

```fsharp
open Azure.Security.KeyVault.Secrets

// Create Key Vault client
let createKeyVaultClient (vaultUri: string) =
    SecretClient(Uri(vaultUri), DefaultAzureCredential())

// Get secret
let getKeyVaultSecret (client: SecretClient) secretName = async {
    try
        let! response = client.GetSecretAsync(secretName) |> Async.AwaitTask
        return Ok response.Value.Value
    with
    | :? Azure.RequestFailedException as ex ->
        return Error $"Failed to get secret: {ex.Message}"
}

// Set secret
let setKeyVaultSecret (client: SecretClient) name value = async {
    let! response = client.SetSecretAsync(name, value) |> Async.AwaitTask
    return response.Value.Id.ToString()
}

// Integration with IConfiguration
let configureKeyVault (builder: Microsoft.Extensions.Hosting.IHostApplicationBuilder) vaultUri =
    builder.Configuration.AddAzureKeyVault(
        Uri(vaultUri),
        DefaultAzureCredential())
    |> ignore
```

---

## สรุป

Cloud Services ใน F#:

1. **Azure Blob Storage**: เก็บ files, ใช้ SAS URLs
2. **Azure Queue Storage**: Simple message queuing
3. **Azure Service Bus**: Enterprise messaging, pub/sub
4. **Azure Functions**: Serverless ด้วย F#
5. **AWS S3**: Object storage, multipart uploads
6. **AWS SQS**: Simple Queue Service
7. **AWS Secrets Manager**: Secrets rotation
8. **Cloud-Native Patterns**: Circuit breaker, retry, bulkhead

F# ทำงานได้ดีกับ cloud services ด้วย:
- Async/await สำหรับ non-blocking I/O
- Result types สำหรับ error handling
- Type safety สำหรับ API responses

---

*ต่อไป: Part 98 - Machine Learning กับ F#*
