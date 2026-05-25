---
name: ocr-integration
description: Integrates OCR capabilities into PayStation microservices — Security (KYC), SCM (invoices/products), File Service (document search), and Payment (cheques/receipts). Covers API integration with Ollama OCR service, data mapping, database schema, and async processing flows.
---

# OCR Integration — PayStation Microservices

Integrate OCR (Optical Character Recognition) into existing PayStation services using the Ollama OCR API running on Laptop A or in Docker.

> ⏱ **Session history is tracked below.**

## Architecture

```
                           ┌─────────────────────────────┐
                           │   Ollama OCR Service         │
                           │   http://192.168.68.113:5004 │
                           │   Model: qwen2.5-vl:7b      │
                           └──────────┬──────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐          ┌──────────────────┐          ┌──────────────────┐
│  Security API │          │    SCM API        │          │   File Service   │
│  /api/kyc     │          │  /api/products    │          │  /api/files      │
│               │          │  /api/invoices    │          │                  │
│  NID/Passport │          │  Product images   │          │  All documents   │
│  KYC docs     │          │  Purchase invoices │          │  Search index    │
└───────────────┘          └──────────────────┘          └──────────────────┘
```

## Hardware Reference (Your Laptops)

### Laptop A (Ollama / GPU Host)

| Component | Spec |
|-----------|------|
| **CPU** | AMD Ryzen 7 4800H (8 cores, 16 threads) |
| **RAM** | 23.4 GB |
| **GPU** | NVIDIA GeForce RTX 3050 Ti (4GB VRAM) — CUDA enabled |
| **Free Space** | D: 38 GB free (install models here) |
| **IP** | `192.168.68.113` |

### Laptop B (Client / Services)

| Component | Spec |
|-----------|------|
| **GPU** | Intel UHD Graphics 620 (shared, no dedicated VRAM) |
| **RAM** | 15.9 GB |
| **IP** | `192.168.68.107` |

> **Run Ollama on Laptop A** (GPU). Laptop B and Docker services connect via `http://192.168.68.113:11434`.

---

## Prerequisites

- Ollama OCR API running on Laptop A (see `ollama-ocr` skill)
- OCR API endpoint: `http://192.168.68.113:5004`
- Ollama endpoint: `http://192.168.68.113:11434`
- .NET 8 services with HTTP client configured

---

## 1. Integration Setup

### 1.1 Add HttpClient in each service's Program.cs

```csharp
// In Security.API/Program.cs, SCM.API/Program.cs, FileService/Program.cs
builder.Services.AddHttpClient("OcrApi", client =>
{
    client.BaseAddress = new Uri(builder.Configuration["OcrApi:BaseUrl"] ?? "http://192.168.68.113:5004");
    client.Timeout = TimeSpan.FromSeconds(120);
});
```

### 1.2 Add configuration to appsettings.json

```json
{
  "OcrApi": {
    "BaseUrl": "http://192.168.68.113:5004",
    "TimeoutSeconds": 120,
    "DefaultModel": "qwen2.5-vl:7b"
  }
}
```

### 1.3 Create shared OCR client library

Create `Services/Core/Ocr.Core/OcrClient.cs`:

```csharp
namespace Core.Ocr;

public class OcrClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<OcrClient> _logger;

    public OcrClient(HttpClient httpClient, ILogger<OcrClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<string> ExtractTextAsync(string imageBase64, CancellationToken ct = default)
    {
        var response = await _httpClient.PostAsJsonAsync("/api/ocr/extract-base64",
            new { imageBase64 }, ct);
        response.EnsureSuccessStatusCode();

        var result = await response.Content.ReadFromJsonAsync<OcrTextResult>(ct);
        return result?.ExtractedText ?? string.Empty;
    }

    public async Task<T> ExtractDocumentAsync<T>(string imageBase64, CancellationToken ct = default) where T : class
    {
        var response = await _httpClient.PostAsJsonAsync("/api/ocr/document",
            new { imageBase64 }, ct);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<T>(ct) ?? throw new InvalidOperationException("OCR returned null");
    }
}

public record OcrTextResult(string ExtractedText);
```

---

## 2. Security API — KYC Document OCR

### 2.1 Database Schema

```sql
-- Add to Security.DAL/Migrations
CREATE TABLE kyc_documents (
    id BIGINT PRIMARY KEY IDENTITY(1,1),
    company_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    document_type VARCHAR(50) NOT NULL, -- 'NID', 'PASSPORT', 'BIRTH_CERTIFICATE'
    document_number VARCHAR(100),
    full_name NVARCHAR(200),
    date_of_birth DATE,
    issue_date DATE,
    expiry_date DATE,
    address NVARCHAR(500),
    father_name NVARCHAR(200),
    mother_name NVARCHAR(200),
    image_path VARCHAR(500),
    extracted_json NVARCHAR(MAX), -- Full OCR output as JSON
    ocr_status VARCHAR(20) DEFAULT 'PENDING', -- PENDING, PROCESSED, FAILED
    ocr_confidence DECIMAL(5,2),
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    updated_at DATETIME2,
    CONSTRAINT fk_kyc_user FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### 2.2 KYC Service with OCR

```csharp
// Security.Manager/Services/KycService.cs
public class KycService
{
    private readonly OcrClient _ocrClient;
    private readonly IRepository<KycDocument> _kycRepo;

    public async Task<KycDocument> ProcessKycDocumentAsync(long userId, IFormFile file)
    {
        using var ms = new MemoryStream();
        await file.CopyToAsync(ms);
        var base64 = Convert.ToBase64String(ms.ToArray());

        // Extract document fields via OCR
        var extracted = await _ocrClient.ExtractDocumentAsync<KycExtractedData>(base64);

        var kycDoc = new KycDocument
        {
            UserId = userId,
            DocumentType = extracted.DocumentType,
            DocumentNumber = extracted.DocumentNumber,
            FullName = extracted.FullName,
            DateOfBirth = extracted.DateOfBirth,
            IssueDate = extracted.IssueDate,
            ExpiryDate = extracted.ExpiryDate,
            Address = extracted.Address,
            ExtractedJson = JsonSerializer.Serialize(extracted),
            OcrStatus = "PROCESSED"
        };

        await _kycRepo.AddAsync(kycDoc);
        return kycDoc;
    }
}

public class KycExtractedData
{
    public string? DocumentType { get; set; }
    public string? DocumentNumber { get; set; }
    public string? FullName { get; set; }
    public DateTime? DateOfBirth { get; set; }
    public DateTime? IssueDate { get; set; }
    public DateTime? ExpiryDate { get; set; }
    public string? Address { get; set; }
    public string? FatherName { get; set; }
    public string? MotherName { get; set; }
}
```

### 2.3 KYC Controller Endpoint

```csharp
// Security.API/Controllers/KycController.cs
[HttpPost("upload-document")]
public async Task<IActionResult> UploadKycDocument(
    IFormFile file,
    [FromQuery] long userId)
{
    var result = await _kycService.ProcessKycDocumentAsync(userId, file);
    return Ok(new
    {
        documentId = result.Id,
        documentType = result.DocumentType,
        documentNumber = result.DocumentNumber,
        fullName = result.FullName,
        ocrStatus = result.OcrStatus
    });
}
```

---

## 3. SCM API — Product & Invoice OCR

### 3.1 Product Image → Auto Catalog

```csharp
// SCM.Manager/Services/ProductOcrService.cs
public class ProductOcrService
{
    private readonly OcrClient _ocrClient;

    public async Task<ProductSuggestion> SuggestProductFromImageAsync(string imageBase64)
    {
        var prompt = @"
Analyze this product image. Return JSON with:
- product_name: detected product name
- category: suggested category
- brand: brand name if visible
- description: short description
- estimated_price: price if visible
Return ONLY valid JSON.";

        var result = await _ocrClient.ExtractWithPromptAsync<ProductSuggestion>(imageBase64, prompt);
        return result;
    }
}
```

### 3.2 Invoice/Purchase Order OCR

```csharp
[HttpPost("invoices/ocr")]
public async Task<IActionResult> ProcessInvoiceImage(IFormFile file)
{
    using var ms = new MemoryStream();
    await file.CopyToAsync(ms);
    var base64 = Convert.ToBase64String(ms.ToArray());

    var prompt = @"
Extract invoice details from this image. Return JSON with:
- invoice_number
- vendor_name
- vendor_address
- date
- total_amount
- line_items: array of { description, quantity, unit_price, amount }
- tax_amount
- grand_total
Return ONLY valid JSON.";

    var invoice = await _ocrClient.ExtractWithPromptAsync<InvoiceData>(base64, prompt);

    // Auto-create purchase entry
    var purchase = await _purchaseService.CreateFromInvoiceAsync(invoice);
    return Ok(purchase);
}
```

---

## 4. File Service — Document Search

### 4.1 Auto-Extract Text on Upload

```csharp
// FileService.Manager/Services/FileProcessingService.cs
public class FileProcessingService
{
    private readonly OcrClient _ocrClient;

    public async Task ProcessUploadedFileAsync(FileRecord file)
    {
        if (!IsImageFile(file.Extension)) return;

        var base64 = await File.ReadAllBytesAsync(file.PhysicalPath);
        var text = await _ocrClient.ExtractTextAsync(Convert.ToBase64String(base64));

        // Store extracted text for search
        file.ExtractedText = text;
        file.IsSearchable = true;
        file.OcrProcessedAt = DateTime.UtcNow;

        // Index for full-text search
        await _searchIndex.IndexDocumentAsync(file.Id, text);
    }
}
```

### 4.2 Search Endpoint

```csharp
[HttpGet("search")]
public async Task<IActionResult> SearchDocuments([FromQuery] string query)
{
    var results = await _searchIndex.SearchAsync(query);
    return Ok(results);
}
```

---

## 5. Async Processing with Background Jobs

For large volumes, use a background queue instead of real-time OCR:

### 5.1 Using IHostedService + Channel

```csharp
// OcrBackgroundService.cs
public class OcrBackgroundService : BackgroundService
{
    private readonly Channel<OcrJob> _channel;
    private readonly IServiceScopeFactory _scopeFactory;

    public OcrBackgroundService(Channel<OcrJob> channel, IServiceScopeFactory scopeFactory)
    {
        _channel = channel;
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = _scopeFactory.CreateScope();
            var ocrClient = scope.ServiceProvider.GetRequiredService<OcrClient>();

            try
            {
                var text = await ocrClient.ExtractTextAsync(job.ImageBase64, stoppingToken);
                job.OnCompleted(text);
            }
            catch (Exception ex)
            {
                job.OnFailed(ex);
            }
        }
    }
}

public record OcrJob(
    string ImageBase64,
    Action<string> OnCompleted,
    Action<Exception> OnFailed
);
```

### 5.2 Enqueue from Controller

```csharp
[HttpPost("upload")]
public async Task<IActionResult> UploadFile(IFormFile file)
{
    var jobId = Guid.NewGuid();
    var base64 = await FileToBase64Async(file);

    var job = new OcrJob(base64,
        text => _dbContext.Files.Find(jobId).ExtractedText = text,
        ex => _logger.LogError(ex, "OCR failed for {JobId}", jobId)
    );

    await _channel.Writer.WriteAsync(job);
    return Accepted(new { jobId, message = "File queued for OCR processing" });
}
```

---

## 6. Docker Compose — Full Stack with OCR

```yaml
version: '3.8'

services:
  # Existing services
  security-api:
    build: ./services/Services/Security/Security.API
    # ... existing config
    environment:
      - OcrApi__BaseUrl=http://ocr-api:5004

  scm-api:
    build: ./services/Services/SCM/SCM.API
    environment:
      - OcrApi__BaseUrl=http://ocr-api:5004

  file-service:
    build: ./services/Services/File/File.API
    environment:
      - OcrApi__BaseUrl=http://ocr-api:5004

  # New OCR service
  ocr-api:
    build: ./services/Services/Ocr/Ocr.API
    ports:
      - "5004:8080"
    environment:
      - Ollama__Endpoint=http://ollama:11434
    depends_on:
      - ollama
    restart: unless-stopped

  # Ollama (GPU accelerated)
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]

volumes:
  ollama-data:
```

---

## 7. Testing OCR Integration

### 7.1 Unit Test

```csharp
[Fact]
public async Task ExtractText_ValidImage_ReturnsText()
{
    // Arrange
    var mockHttp = new MockHttpMessageHandler();
    mockHttp.When("/api/ocr/extract-base64")
        .Respond("application/json", "{\"extractedText\": \"Test OCR output\"}");

    var client = new OcrClient(mockHttp.ToHttpClient(), NullLogger<OcrClient>.Instance);
    var base64 = Convert.ToBase64String(new byte[] { 1, 2, 3 });

    // Act
    var result = await client.ExtractTextAsync(base64);

    // Assert
    Assert.Equal("Test OCR output", result);
}
```

### 7.2 Integration Test (requires running OCR API)

```bash
# Test KYC endpoint
curl -X POST http://localhost:5001/api/kyc/upload-document?userId=123 \
  -F "file=@test-nid.jpg"

# Test product OCR
curl -X POST http://localhost:5002/api/products/ocr \
  -F "file=@product.jpg"

# Test file search
curl "http://localhost:5003/api/files/search?query=invoice"
```

---

## 8. Use Cases Summary

| Service | Use Case | OCR Input | Output |
|---------|----------|-----------|--------|
| **Security** | KYC verification | NID/Passport image | Structured JSON (name, DOB, ID, address) |
| **Security** | Cheque scan | Cheque image | Account number, amount, signature |
| **SCM** | Product catalog | Product photo | Auto-suggest name, category, price |
| **SCM** | Invoice processing | Invoice image | Vendor, items, total → auto purchase order |
| **SCM** | Barcode/QR | Label image | Code → product lookup |
| **File Service** | Document search | Any document | Full-text searchable index |
| **Payment** | Receipt OCR | Payment receipt | Amount, merchant, transaction ID |
| **Payment** | Bank statement | Statement image | Transactions table |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| OCR returns empty text | Check image quality, try different prompt, ensure base64 is correct |
| Timeout on large images | Resize image before sending (max 1024px) |
| Wrong document fields | Customize prompt per document type |
| GPU not utilized | Verify CUDA: `ollama run qwen2.5-vl:7b --verbose` |
| Queue growing too large | Add multiple OCR worker instances |
| Duplicate OCR processing | Check OCR status before processing |

---

## ⏱ Session History

> **Agent instructions:** Before making changes, read this section. After making changes, append a new entry.

---

*Add new sessions below this line.*
