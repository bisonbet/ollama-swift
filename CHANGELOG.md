# Changelog

All notable changes to the ollama-swift library are documented in this file.

## [Unreleased]

### Added

#### Core API Enhancements
- **Suffix Parameter for Code Completion**: Added `suffix` parameter to `generate()` and `generateStream()` for fill-in-the-middle code completion
  - Enables advanced code completion scenarios
  - Particularly useful with code-focused models like CodeLlama
  - Example: Complete code between a prompt and a predefined ending

- **Custom Embedding Dimensions**: Added `dimensions` parameter to `embed()` methods
  - Allows customization of embedding vector dimensions
  - Useful for optimizing embedding size vs. quality tradeoffs
  - Supported by models like nomic-embed-text

- **Verbose Model Information**: Added `verbose` parameter to `showModel()`
  - Returns detailed model information including full token data
  - Helpful for debugging and understanding model internals

- **Model Quantization Support**: Added `quantization` parameter to `createModel()`
  - Supports q4_0, q4_1, q5_0, q5_1, q8_0 quantization levels
  - Enables creation of smaller, faster model variants
  - Useful for resource-constrained environments

#### Advanced Features

- **Blob Operations**: New blob management APIs for custom model creation
  - `checkBlob(digest:)`: Check if a blob exists on the server
  - `createBlob(digest:data:)`: Upload binary model files (GGUF, Safetensors)
  - Essential for creating models from local files

- **Streaming Progress for Long Operations**: Added streaming variants for model operations
  - `pullModelStream()`: Monitor model download progress with real-time updates
  - `pushModelStream()`: Track model upload progress
  - `createModelStream()`: Follow model creation status
  - Each includes progress types: `PullProgress`, `PushProgress`, `CreateProgress`

- **OpenAI API Compatibility Layer**: Full OpenAI-compatible client implementation
  - New `OpenAICompatibleClient` class
  - Compatible with `/v1/chat/completions` endpoint
  - Compatible with `/v1/completions` endpoint
  - Drop-in replacement for OpenAI client code
  - Enables easy migration from OpenAI to local models

### Enhanced

- **Improved Documentation**: Comprehensive README updates with:
  - Code completion examples
  - Custom embedding dimensions usage
  - Model quantization guide
  - Blob operations walkthrough
  - OpenAI migration guide
  - SwiftUI integration example
  - Server-side Swift (Vapor) example
  - Best practices and performance tips

- **Better Error Handling**: Improved error messages and handling throughout

### Backward Compatibility

All changes are backward compatible. Existing code will continue to work without modifications:
- New parameters are optional with sensible defaults
- Existing methods maintain their signatures
- No breaking changes to public APIs

### Migration Guide

#### Using Suffix Parameter (New Feature)

```swift
// Code completion with fill-in-the-middle
let response = try await client.generate(
    model: "codellama:7b-code",
    prompt: "def convert_string_to_uppercase(string: str) -> str:",
    suffix: "    return result"
)
```

#### Using Custom Embedding Dimensions (New Feature)

```swift
// Before (uses model default)
let response = try await client.embed(
    model: "nomic-embed-text",
    input: "Your text here"
)

// After (with custom dimensions)
let response = try await client.embed(
    model: "nomic-embed-text",
    input: "Your text here",
    dimensions: 512
)
```

#### Using Verbose Model Info (New Feature)

```swift
// Before
let modelInfo = try await client.showModel("llama3.2")

// After (with verbose details)
let modelInfo = try await client.showModel("llama3.2", verbose: true)
```

#### Using Model Quantization (New Feature)

```swift
// Create a quantized model variant
let success = try await client.createModel(
    name: "llama3.2-q4",
    modelfile: "FROM llama3.2",
    quantization: "q4_0"
)
```

#### Monitoring Model Operations (New Feature)

```swift
// Before (no progress updates)
let success = try await client.pullModel("llama3.2")

// After (with progress updates)
let stream = client.pullModelStream("llama3.2")
for try await progress in stream {
    if let completed = progress.completed, let total = progress.total {
        let percentage = (Double(completed) / Double(total)) * 100
        print("Progress: \(String(format: "%.1f", percentage))%")
    }
}
```

#### Using Blob Operations (New Feature)

```swift
import CryptoKit

// Load and upload a custom model file
let modelData = try Data(contentsOf: URL(fileURLWithPath: "/path/to/model.gguf"))
let hash = SHA256.hash(data: modelData)
let digest = "sha256:" + hash.compactMap { String(format: "%02x", $0) }.joined()

// Check if blob exists before uploading
if try await client.checkBlob(digest: digest) {
    print("Blob already exists")
} else {
    let success = try await client.createBlob(digest: digest, data: modelData)
}

// Reference the blob in a Modelfile
let modelfile = "FROM \(digest)\nSYSTEM You are helpful."
try await client.createModel(name: "my-model", modelfile: modelfile)
```

#### Using OpenAI Compatibility (New Feature)

```swift
// Create OpenAI-compatible client
let openAIClient = OpenAICompatibleClient.default

// Use OpenAI-style chat completions
let request = OpenAICompatibleClient.ChatCompletionRequest(
    model: "llama3.2",
    messages: [
        .system("You are a helpful assistant."),
        .user("Hello!")
    ]
)

let response = try await openAIClient.createChatCompletion(request)
print(response.choices.first?.message.content ?? "")
```

### Technical Details

#### New Types

- `OpenAICompatibleClient`: OpenAI-compatible API client
- `OpenAICompatibleClient.ChatCompletionRequest`: OpenAI chat completion request
- `OpenAICompatibleClient.ChatCompletionResponse`: OpenAI chat completion response
- `OpenAICompatibleClient.CompletionRequest`: OpenAI text completion request
- `OpenAICompatibleClient.CompletionResponse`: OpenAI text completion response
- `Client.PullProgress`: Progress updates for model pulls
- `Client.PushProgress`: Progress updates for model pushes
- `Client.CreateProgress`: Progress updates for model creation

#### New Methods

**Client Extensions:**
- `checkBlob(digest:) async throws -> Bool`
- `createBlob(digest:data:) async throws -> Bool`
- `pullModelStream(_:insecure:) -> AsyncThrowingStream<PullProgress, Error>`
- `pushModelStream(_:insecure:) -> AsyncThrowingStream<PushProgress, Error>`
- `createModelStream(name:modelfile:path:quantization:) -> AsyncThrowingStream<CreateProgress, Error>`

**Modified Method Signatures:**
- `generate()`: Added `suffix: String? = nil` parameter
- `generateStream()`: Added `suffix: String? = nil` parameter
- `embed(model:input:...)`: Added `dimensions: Int? = nil` parameter
- `embed(model:inputs:...)`: Added `dimensions: Int? = nil` parameter
- `showModel(_:)`: Added `verbose: Bool = false` parameter
- `createModel(name:modelfile:path:)`: Added `quantization: String? = nil` parameter

### Contributors

This update brings ollama-swift fully up to date with the latest Ollama API capabilities, ensuring feature parity with the official Ollama project.

### Future Considerations

- Streaming support for OpenAI-compatible endpoints
- Additional model management features as they become available in Ollama
- Enhanced type safety for model options and parameters
- More comprehensive error types for different failure scenarios
