# Ollama Swift Client

A Swift client library for interacting with the
[Ollama API](https://github.com/ollama/ollama/blob/main/docs/api.md).

## Requirements

- Swift 5.7+
- macOS 13+
- [Ollama](https://ollama.com)

## Installation

### Swift Package Manager

Add the following to your `Package.swift` file:

```swift
.package(url: "https://github.com/bisonbet/ollama-swift.git", from: "1.8.0")
```

## Usage

> [!NOTE]
> The tests and example code for this library use the
> [llama3.2](https://ollama.com/library/llama3.2) model.
> Run the following command to download the model to run them yourself:
>
> ```
> ollama pull llama3.2
> ```

### Initializing the client

```swift
import Ollama

// Use the default client (http://localhost:11434)
let client = Client.default

// Or create a custom client
let customClient = Client(host: URL(string: "http://your-ollama-host:11434")!, userAgent: "MyApp/1.0")
```

### Generating text

Generate text using a specified model:

```swift
do {
    let response = try await client.generate(
        model: "llama3.2",
        prompt: "Tell me a joke about Swift programming.",
        options: [
            "temperature": 0.7,
            "max_tokens": 100
        ],
        keepAlive: .minutes(10)  // Keep model loaded for 10 minutes
    )
    print(response.response)
} catch {
    print("Error: \(error)")
}
```

#### Streaming text generation

Generate text in a streaming fashion to receive responses in real-time:

```swift
do {
    let stream = try await client.generateStream(
        model: "llama3.2",
        prompt: "Tell me a joke about Swift programming.",
        options: [
            "temperature": 0.7,
            "max_tokens": 100
        ]
    )

    var fullResponse = ""
    for try await chunk in stream {
        // Process each chunk of the response as it arrives
        print(chunk.response, terminator: "")
        fullResponse += chunk.response
    }
    print("\nFull response: \(fullResponse)")
} catch {
    print("Error: \(error)")
}
```

### Chatting with a model

Generate a chat completion:

```swift
do {
    let response = try await client.chat(
        model: "llama3.2",
        messages: [
            .system("You are a helpful assistant."),
            .user("In which city is Apple Inc. located?")
        ],
        keepAlive: .minutes(10)  // Keep model loaded for 10 minutes
    )
    print(response.message.content)
} catch {
    print("Error: \(error)")
}
```

#### Streaming chat responses

Stream chat responses to get real-time partial completions:

```swift
do {
    let stream = try await client.chatStream(
        model: "llama3.2",
        messages: [
            .system("You are a helpful assistant."),
            .user("Write a short poem about Swift programming.")
        ]
    )

    var fullContent = ""
    for try await chunk in stream {
        // Process each chunk of the message as it arrives
        if let content = chunk.message.content {
            print(content, terminator: "")
            fullContent += content
        }
    }
    print("\nComplete poem: \(fullContent)")
} catch {
    print("Error: \(error)")
}
```

You can also stream chat responses when using tools:

```swift
do {
    let stream = try await client.chatStream(
        model: "llama3.2",
        messages: [
            .system("You are a helpful assistant that can check the weather."),
            .user("What's the weather like in Portland?")
        ],
        tools: [weatherTool]
    )

    for try await chunk in stream {
        // Check if the model is making tool calls
        if let toolCalls = chunk.message.toolCalls, !toolCalls.isEmpty {
            print("Model is requesting tool: \(toolCalls[0].function.name)")
        }

        // Print content from the message as it streams
        if let content = chunk.message.content {
            print(content, terminator: "")
        }

        // Check if this is the final chunk
        if chunk.done {
            print("\nResponse complete")
        }
    }
} catch {
    print("Error: \(error)")
}
```

### Using Structured Outputs

You can request structured outputs from models by specifying a format.
Pass `"json"` to get back a JSON string,
or specify a full [JSON Schema](https://json-schema.org):

```swift
// Simple JSON format
let response = try await client.chat(
    model: "llama3.2",
    messages: [.user("List 3 colors.")],
    format: "json"
)

// Using JSON schema for more control
let schema: Value = [
    "type": "object",
    "properties": [
        "colors": [
            "type": "array",
            "items": [
                "type": "object",
                "properties": [
                    "name": ["type": "string"],
                    "hex": ["type": "string"]
                ],
                "required": ["name", "hex"]
            ]
        ]
    ],
    "required": ["colors"]
]

let response = try await client.chat(
    model: "llama3.2",
    messages: [.user("List 3 colors with their hex codes.")],
    format: schema
)

// The response will be a JSON object matching the schema:
// {
//   "colors": [
//     {"name": "papayawhip", "hex": "#FFEFD5"},
//     {"name": "indigo", "hex": "#4B0082"},
//     {"name": "navy", "hex": "#000080"}
//   ]
// }
```

The format parameter works with both `chat` and `generate` methods.

### Using Thinking Models

Some models support a "thinking" mode
where they show their reasoning process before providing the final answer.
This is particularly useful for complex reasoning tasks.

```swift
// Generate with thinking enabled
let response = try await client.generate(
    model: "deepseek-r1:8b",
    prompt: "What is 17 * 23? Show your work.",
    think: true
)

print("Thinking: \(response.thinking ?? "None")")
print("Answer: \(response.response)")
```

You can also use thinking in chat conversations:

```swift
let response = try await client.chat(
    model: "deepseek-r1:8b",
    messages: [
        .system("You are a helpful mathematician."),
        .user("Calculate 9.9 + 9.11 and explain your reasoning.")
    ],
    think: true
)

print("Thinking: \(response.message.thinking ?? "None")")
print("Response: \(response.message.content)")
```

> [!TIP]
> You can check which models support thinking by examining their capabilities:
> ```swift
> let modelInfo = try await client.showModel("deepseek-r1:8b")
> if modelInfo.capabilities.contains(.thinking) {
>     print("🧠 This model supports thinking!")
> }
> ```

### Managing Model Memory with Keep-Alive

You can control how long a model stays loaded in memory using the `keepAlive` parameter. This is useful for managing memory usage and response times.

```swift
// Use server default (typically 5 minutes)
let response = try await client.generate(
    model: "llama3.2",
    prompt: "Hello!"
    // keepAlive defaults to .default
)

// Keep model loaded for 10 minutes
let response = try await client.generate(
    model: "llama3.2",
    prompt: "Hello!",
    keepAlive: .minutes(10)
)

// Keep model loaded for 2 hours
let response = try await client.chat(
    model: "llama3.2",
    messages: [.user("Hello!")],
    keepAlive: .hours(2)
)

// Keep model loaded for 30 seconds
let response = try await client.generate(
    model: "llama3.2",
    prompt: "Hello!",
    keepAlive: .seconds(30)
)

// Keep model loaded indefinitely
let response = try await client.chat(
    model: "llama3.2",
    messages: [.user("Hello!")],
    keepAlive: .forever
)

// Unload model immediately after response
let response = try await client.generate(
    model: "llama3.2",
    prompt: "Hello!",
    keepAlive: .none
)
```

- **`.default`** - Use the server's default keep-alive behavior (default if not specified)
- **`.none`** - Unload immediately after the request
- **`.seconds(Int)`** - Keep loaded for the specified number of seconds
- **`.minutes(Int)`** - Keep loaded for the specified number of minutes
- **`.hours(Int)`** - Keep loaded for the specified number of hours
- **`.forever`** - Keep loaded indefinitely

> [!NOTE]
> Zero durations (e.g., `.seconds(0)`) are treated as `.none` (unload immediately).
> Negative durations are treated as `.forever` (keep loaded indefinitely).

### Using Tools

Ollama supports tool calling with models,
allowing models to perform complex tasks or interact with external services.

> [!NOTE]
> Tool support requires a [compatible model](https://ollama.com/search?c=tools),
> such as llama3.2.

#### Creating a Tool

Define a tool by specifying its name, description, parameters, and implementation:

```swift
struct WeatherInput: Codable {
    let city: String
}

struct WeatherOutput: Codable {
    let temperature: Double
    let conditions: String
}

let weatherTool = Tool<WeatherInput, WeatherOutput>(
    name: "get_current_weather",
    description: """
    Get the current weather for a city,
    with conditions ("sunny", "cloudy", etc.)
    and temperature in °C.
    """,
    parameters: [
        "city": [
            "type": "string",
            "description": "The city to get weather for"
        ]
    ],
    required: ["city"]
) { input async throws -> WeatherOutput in
    // Implement weather lookup logic here
    return WeatherOutput(temperature: 18.5, conditions: "cloudy")
}
```

> [!IMPORTANT]
> In version 1.3.0 and later,
> the `parameters` argument should contain only the properties object,
> not the full JSON schema of the tool.
>
> For backward compatibility,
> passing a full schema in the `parameters` argument
> (with `"type"`, `"properties"`, and `"required"` fields)
> is still supported but deprecated and will emit a warning in debug builds.
>
> <details>
> <summary>Click to see code examples of old vs. new format</summary>
>
> ```swift
> // ✅ New format
> let weatherTool = Tool<WeatherInput, WeatherOutput>(
>     name: "get_current_weather",
>     description: "Get the current weather for a city",
>     parameters: [
>         "city": [
>             "type": "string",
>             "description": "The city to get weather for"
>         ]
>     ],
>     required: ["city"]
> ) { /* implementation */ }
>
> // ❌ Deprecated format (still works but not recommended)
> let weatherTool = Tool<WeatherInput, WeatherOutput>(
>     name: "get_current_weather",
>     description: "Get the current weather for a city",
>     parameters: [
>         "type": "object",
>         "properties": [
>             "city": [
>                 "type": "string",
>                 "description": "The city to get weather for"
>             ]
>         ],
>         "required": ["city"]
>     ]
> ) { /* implementation */ }
> ```
> </details>

#### Using Tools in Chat

Provide tools to the model during chat:

```swift
let messages: [Chat.Message] = [
    .system("You are a helpful assistant that can check the weather."),
    .user("What's the weather like in Portland?")
]

let response = try await client.chat(
    model: "llama3.1",
    messages: messages,
    tools: [weatherTool]
)

// Handle tool calls in the response
if let toolCalls = response.message.toolCalls {
    for toolCall in toolCalls {
        print("Tool called: \(toolCall.function.name)")
        print("Arguments: \(toolCall.function.arguments)")
    }
}
```

#### Multi-turn Tool Conversations

Tools can be used in multi-turn conversations, where the model can use tool results to provide more detailed responses:

```swift
var messages: [Chat.Message] = [
    .system("You are a helpful assistant that can convert colors."),
    .user("What's the hex code for yellow?")
]

// First turn - model calls the tool
let response1 = try await client.chat(
    model: "llama3.1",
    messages: messages,
    tools: [rgbToHexTool]
)

enum ToolError {
    case invalidParameters
}

// Add tool response to conversation
if let toolCall = response1.message.toolCalls?.first {
    // Parse the tool arguments
    guard let args = toolCall.function.arguments,
          let redValue = args["red"],
          let greenValue = args["green"],
          let blueValue = args["blue"],
          let red = Double(redValue, strict: false),
          let green = Double(greenValue, strict: false),
          let blue = Double(blueValue, strict: false)
    else {
        throw ToolError.invalidParameters
    }

    let input = HexColorInput(
        red: red,
        green: green,
        blue: blue
    )

    // Execute the tool with the input
    let hexColor = try await rgbToHexTool(input)

    // Add the tool result to the conversation
    messages.append(.tool(hexColor))
}

// Continue conversation with tool result
messages.append(.user("What other colors are similar?"))
let response2 = try await client.chat(
    model: "llama3.1",
    messages: messages,
    tools: [rgbToHexTool]
)
```

### Fill-in-the-Middle Code Completion

Use the `suffix` parameter for code completion where you provide both the beginning and end of the code:

```swift
do {
    let response = try await client.generate(
        model: "codellama:7b-code",
        prompt: "def convert_string_to_uppercase(string: str) -> str:",
        suffix: "    return result",
        options: [
            "temperature": 0,
            "stop": ["<EOT>"]
        ]
    )
    print(response.response)
} catch {
    print("Error: \(error)")
}
```

This is particularly useful for:
- Code completion in IDEs
- Infilling code between existing lines
- Smart autocomplete features

### Generating embeddings

Generate embeddings for a given text:

```swift
do {
    let response = try await client.embed(
        model: "llama3.2",
        input: "Here is an article about llamas..."
    )
    print("Embeddings: \(response.embeddings)")
} catch {
    print("Error: \(error)")
}
```

Generate embeddings for multiple texts in a single batch:

```swift
do {
    let texts = [
        "First article about llamas...",
        "Second article about alpacas...",
        "Third article about vicuñas..."
    ]

    let response = try await client.embed(
        model: "llama3.2",
        inputs: texts
    )

    // Access embeddings for each input
    for (index, embedding) in response.embeddings.rawValue.enumerated() {
        print("Embedding \(index): \(embedding.count) dimensions")
    }
} catch {
    print("Error: \(error)")
}
```

#### Custom Embedding Dimensions

Some models support custom embedding dimensions for greater control:

```swift
let response = try await client.embed(
    model: "nomic-embed-text",
    input: "Your text here",
    dimensions: 512  // Customize the output dimensions
)
```

### Managing models

#### Listing models

List available models:

```swift
do {
    let models = try await client.listModels()
    for model in models {
        print("Model: \(model.name), Modified: \(model.modifiedAt)")
    }
} catch {
    print("Error: \(error)")
}
```

#### Retrieving model information

Get detailed information about a specific model:

```swift
do {
    let modelInfo = try await client.showModel("llama3.2")
    print("Modelfile: \(modelInfo.modelfile)")
    print("Parameters: \(modelInfo.parameters)")
    print("Template: \(modelInfo.template)")
} catch {
    print("Error: \(error)")
}
```

Get verbose model information with full data:

```swift
do {
    let modelInfo = try await client.showModel("llama3.2", verbose: true)
    // Returns more detailed information including full token data
    print("Detailed info: \(modelInfo.info)")
} catch {
    print("Error: \(error)")
}
```

#### Pulling a model

Download a model from the Ollama library:

```swift
do {
    let success = try await client.pullModel("llama3.2")
    if success {
        print("Model successfully pulled")
    } else {
        print("Failed to pull model")
    }
} catch {
    print("Error: \(error)")
}
```

Monitor pull progress with streaming:

```swift
do {
    let stream = client.pullModelStream("llama3.2")

    for try await progress in stream {
        print("Status: \(progress.status)")
        if let completed = progress.completed, let total = progress.total {
            let percentage = (Double(completed) / Double(total)) * 100
            print("Progress: \(String(format: "%.1f", percentage))%")
        }
    }
    print("Model pull complete!")
} catch {
    print("Error: \(error)")
}
```

#### Pushing a model

```swift
do {
    let success = try await client.pushModel("mynamespace/mymodel:latest")
    if success {
        print("Model successfully pushed")
    } else {
        print("Failed to push model")
    }
} catch {
    print("Error: \(error)")
}
```

Monitor push progress with streaming:

```swift
do {
    let stream = client.pushModelStream("mynamespace/mymodel:latest")

    for try await progress in stream {
        print("Status: \(progress.status)")
        if let completed = progress.completed, let total = progress.total {
            let percentage = (Double(completed) / Double(total)) * 100
            print("Upload Progress: \(String(format: "%.1f", percentage))%")
        }
    }
    print("Model push complete!")
} catch {
    print("Error: \(error)")
}
```

### Creating Custom Models

#### Basic Model Creation

Create a custom model from a Modelfile:

```swift
do {
    let modelfile = """
    FROM llama3.2
    SYSTEM You are a helpful coding assistant specialized in Swift.
    """

    let success = try await client.createModel(
        name: "swift-assistant",
        modelfile: modelfile
    )

    if success {
        print("Model created successfully!")
    }
} catch {
    print("Error: \(error)")
}
```

#### Model Quantization

Create a quantized version of a model for reduced memory usage:

```swift
do {
    let success = try await client.createModel(
        name: "llama3.2-q4",
        modelfile: "FROM llama3.2",
        quantization: "q4_0"  // Options: q4_0, q4_1, q5_0, q5_1, q8_0
    )

    if success {
        print("Quantized model created!")
    }
} catch {
    print("Error: \(error)")
}
```

Supported quantization levels:
- `q4_0` - 4-bit quantization (smallest, fastest)
- `q4_1` - 4-bit quantization (better quality)
- `q5_0` - 5-bit quantization
- `q5_1` - 5-bit quantization (better quality)
- `q8_0` - 8-bit quantization (larger, best quality)

#### Monitor Model Creation Progress

```swift
do {
    let stream = client.createModelStream(
        name: "my-custom-model",
        modelfile: "FROM llama3.2\nSYSTEM You are helpful."
    )

    for try await progress in stream {
        print("Status: \(progress.status)")
    }
    print("Model creation complete!")
} catch {
    print("Error: \(error)")
}
```

### Blob Operations

Blobs are used for uploading custom model files (GGUF, Safetensors) when creating models.

#### Check if a Blob Exists

```swift
do {
    let digest = "sha256:abc123..."
    let exists = try await client.checkBlob(digest: digest)

    if exists {
        print("Blob already exists on server")
    } else {
        print("Blob needs to be uploaded")
    }
} catch {
    print("Error: \(error)")
}
```

#### Upload a Blob

```swift
import CryptoKit

do {
    // Load your model file
    let modelData = try Data(contentsOf: URL(fileURLWithPath: "/path/to/model.gguf"))

    // Calculate SHA256 digest
    let hash = SHA256.hash(data: modelData)
    let digest = "sha256:" + hash.compactMap { String(format: "%02x", $0) }.joined()

    // Check if blob already exists
    if try await client.checkBlob(digest: digest) {
        print("Blob already exists, skipping upload")
    } else {
        // Upload the blob
        let success = try await client.createBlob(digest: digest, data: modelData)
        if success {
            print("Blob uploaded successfully!")
        }
    }

    // Now reference the blob in a Modelfile
    let modelfile = """
    FROM \(digest)
    SYSTEM You are a helpful assistant.
    """

    try await client.createModel(name: "my-custom-model", modelfile: modelfile)
} catch {
    print("Error: \(error)")
}
```

## OpenAI API Compatibility

Ollama-swift includes full support for the OpenAI-compatible API endpoints, allowing you to use this library as a drop-in replacement for OpenAI client code when working with local models.

### Using the OpenAI-Compatible Client

```swift
import Ollama

// Create an OpenAI-compatible client
let openAIClient = OpenAICompatibleClient.default

// Or with custom configuration
let customClient = OpenAICompatibleClient(
    host: URL(string: "http://localhost:11434/v1")!,
    userAgent: "MyApp/1.0"
)
```

### Chat Completions

```swift
do {
    let request = OpenAICompatibleClient.ChatCompletionRequest(
        model: "llama3.2",
        messages: [
            .system("You are a helpful assistant."),
            .user("Write a haiku about Swift programming.")
        ]
    )

    let response = try await openAIClient.createChatCompletion(request)

    if let firstChoice = response.choices.first {
        print(firstChoice.message.content)
    }

    // Access usage statistics
    if let usage = response.usage {
        print("Tokens used: \(usage.totalTokens)")
    }
} catch {
    print("Error: \(error)")
}
```

### Text Completions

```swift
do {
    let request = OpenAICompatibleClient.CompletionRequest(
        model: "codellama:7b-code",
        prompt: "def fibonacci(n):",
        suffix: "    return result",
        temperature: 0.7
    )

    let response = try await openAIClient.createCompletion(request)

    if let firstChoice = response.choices.first {
        print(firstChoice.text)
    }
} catch {
    print("Error: \(error)")
}
```

### Migration from OpenAI

If you have existing code using OpenAI's API, you can easily migrate to Ollama by:

1. Replace your OpenAI client initialization with `OpenAICompatibleClient`
2. Change the base URL to point to your Ollama instance
3. Use local model names instead of OpenAI model names

```swift
// Before (OpenAI)
// let client = OpenAI(apiKey: "...")

// After (Ollama)
let client = OpenAICompatibleClient.default

// The rest of your code can remain largely the same!
```

## Integration Guide

### iOS/macOS App Integration

1. **Add the package dependency** to your Xcode project
2. **Import the framework** in your Swift files:
   ```swift
   import Ollama
   ```

3. **Initialize the client** (preferably as a singleton):
   ```swift
   class OllamaService {
       static let shared = OllamaService()
       let client = Client.default

       private init() {}
   }
   ```

4. **Make async calls** from your UI code:
   ```swift
   Task {
       do {
           let response = try await OllamaService.shared.client.chat(
               model: "llama3.2",
               messages: [.user("Hello!")]
           )
           await MainActor.run {
               // Update UI with response
           }
       } catch {
           print("Error: \(error)")
       }
   }
   ```

### SwiftUI Integration Example

```swift
import SwiftUI
import Ollama

struct ContentView: View {
    @State private var messages: [Chat.Message] = []
    @State private var inputText = ""
    @State private var isLoading = false

    let client = Client.default

    var body: some View {
        VStack {
            ScrollView {
                ForEach(messages.indices, id: \.self) { index in
                    MessageRow(message: messages[index])
                }
            }

            HStack {
                TextField("Type a message...", text: $inputText)
                    .textFieldStyle(.roundedBorder)

                Button("Send") {
                    sendMessage()
                }
                .disabled(isLoading || inputText.isEmpty)
            }
            .padding()
        }
    }

    func sendMessage() {
        let userMessage = Chat.Message.user(inputText)
        messages.append(userMessage)
        inputText = ""
        isLoading = true

        Task {
            do {
                var allMessages = messages

                let response = try await client.chat(
                    model: "llama3.2",
                    messages: allMessages
                )

                await MainActor.run {
                    messages.append(response.message)
                    isLoading = false
                }
            } catch {
                await MainActor.run {
                    isLoading = false
                }
                print("Error: \(error)")
            }
        }
    }
}

struct MessageRow: View {
    let message: Chat.Message

    var body: some View {
        HStack {
            if message.role == .user {
                Spacer()
            }

            Text(message.content)
                .padding()
                .background(message.role == .user ? Color.blue : Color.gray)
                .foregroundColor(.white)
                .cornerRadius(10)

            if message.role == .assistant {
                Spacer()
            }
        }
        .padding(.horizontal)
    }
}
```

### Server-Side Swift Integration

```swift
import Vapor
import Ollama

func routes(_ app: Application) throws {
    let ollamaClient = Client.default

    app.post("chat") { req async throws -> Response in
        struct ChatRequest: Content {
            let message: String
        }

        let chatRequest = try req.content.decode(ChatRequest.self)

        let response = try await ollamaClient.chat(
            model: "llama3.2",
            messages: [.user(chatRequest.message)]
        )

        return Response(
            status: .ok,
            body: .init(string: response.message.content)
        )
    }
}
```

### Best Practices

1. **Reuse the client instance** - Create one client and reuse it throughout your app
2. **Handle errors gracefully** - Network requests can fail, always use try-catch
3. **Use streaming for long responses** - Provide better UX with real-time updates
4. **Set appropriate timeouts** - Some operations can take time
5. **Use keep-alive strategically** - Balance memory usage and response time
6. **Monitor progress** - Use streaming methods for pull/push/create operations

### Performance Tips

1. **Use appropriate quantization levels** for your use case
2. **Batch embeddings requests** when processing multiple texts
3. **Keep models loaded** with `keepAlive` for frequently-used models
4. **Use local models** to avoid network latency
5. **Stream responses** for better perceived performance

## License

This project is available under the MIT license.
See the LICENSE file for more info.
