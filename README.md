------------------------------------------------
# OpenRouter API Client Library

OpenRouter API Client Library is a Rust client for interfacing with the OpenRouter API. The library is designed to be modular, type‑safe, and intuitive. It uses a type‑state builder pattern for configuring and validating the client at compile time, ensuring that all required configuration (such as setting the base URL and API key) happens before attempting a request.

> **Note:** This project is still in development. Many features are planned but not yet implemented.

## Features

- **Modular Organization:** The code is organized into clear modules for models, API endpoints, common types, and utilities.
- **Type‑State Builder:** Guarantees compile‑time validation of client configuration for a robust development experience.
- **HTTP Integration:** Uses [reqwest](https://crates.io/crates/reqwest) with rustls‑tls for secure asynchronous HTTP requests.
- **Robust Error Handling:** Centralized error management using the `thiserror` crate ensures consistent error types across the library.
- **Structured Outputs:** Optionally request structured responses and enable JSON Schema validation using user‑provided schemas. This helps enforce consistent, type‑safe response formats.
- **Tool Calling Capability:** Define function‑type tools that the model can invoke. The client supports multiple concurrent tool calls per response and validates that each tool call conforms to the expected format.
- **Provider Preferences & Routing:** Configure model fallbacks, routing preferences, and provider filtering via a new strongly‑typed interface.
- **Web Search Endpoint:** A new endpoint to perform web search queries. It accepts a minimal request structure (query and an optional number of results) and returns a structured set of search results.
- **Future Roadmap:**
  - Streaming support for real‑time completions.
  - Text completion endpoint.
  - Endpoints for credits, generation metadata, and available models.
  - Extended tests and documentation improvements.

## Getting Started

### Installation

Add the following to your project's `Cargo.toml`:

```toml
[dependencies]
openrouter_api = { git = "https://github.com/yourusername/openrouter_api.git", branch = "main" }
```

Ensure that you have Rust installed (tested with Rust v1.83.0) and that you're using Cargo for building and testing.

### Example Usage

Minimal Chat Example

------------------------------------------------
use openrouter_api::{OpenRouterClient, Ready, Result};
use openrouter_api::types::chat::{ChatCompletionRequest, Message};

#[tokio::main]
async fn main() -> Result<()> {
    // Ensure your API key is set in the environment.
    let api_key = std::env::var("OPENROUTER_API_KEY")
        .expect("OPENROUTER_API_KEY must be set");

    // Build the client (Unconfigured -> NoAuth -> Ready).
    let client = OpenRouterClient::new()
        .with_base_url("https://openrouter.ai/api/v1/")?
        .with_api_key(api_key);

    // Create a minimal chat completion request.
    let request = ChatCompletionRequest {
        model: "openai/gpt-4o".to_string(),
        messages: vec![Message {
            role: "user".to_string(),
            content: "Hello, world!".to_string(),
            name: None,
            tool_calls: None,
        }],
        stream: None,
        response_format: None,
        tools: None,
        provider: None,
        models: None,
        transforms: None,
    };

    // Invoke the chat completion endpoint.
    let response = client.chat_completion(request).await?;

    // Output the model's response.
    if let Some(choice) = response.choices.first() {
        println!("Chat Response: {}", choice.message.content);
    }
    Ok(())
}

------------------------------------------------

Below is a minimal example that demonstrates how to use the new Web Search endpoint.

```rust
use openrouter_api::{OpenRouterClient, Ready, Result};
use openrouter_api::types::web_search::{WebSearchRequest, WebSearchResponse};

#[tokio::main]
async fn main() -> Result<()> {
    // Ensure you have set your API key in the environment.
    let api_key = std::env::var("OPENROUTER_API_KEY")
        .expect("OPENROUTER_API_KEY must be set");

    // Build the client (Unconfigured -> NoAuth -> Ready).
    let client = OpenRouterClient::new()
        .with_base_url("https://openrouter.ai/api/v1/")?
        .with_api_key(api_key);

    // Create a minimal web search request.
    let request = WebSearchRequest {
        query: "rust programming".into(),
        num_results: Some(5),
    };

    // Invoke the web search endpoint.
    let response: WebSearchResponse = client.web_search().search(request).await?;

    // Print out the search results.
    println!("Search query: {}", response.query);
    for result in response.results {
        println!("Title: {}\nURL: {}\n", result.title, result.url);
    }

    Ok(())
}
```

Below is a more complete example that creates a client, configures it with the API key, sends a chat completion request with structured output enabled, and demonstrates tool calling capabilities:

```rust
use openrouter_api::{
    OpenRouterClient, Ready,
    types::chat::{ChatCompletionRequest, Message},
    Result,
};
use openrouter_api::models::structured::{JsonSchemaConfig, JsonSchemaDefinition};
use openrouter_api::models::tool::{Tool, FunctionDescription};
use serde_json::json;

#[tokio::main]
async fn main() -> Result<()> {
    // Make sure to set your API key in an environment variable.
    let api_key = std::env::var("OPENROUTER_API_KEY")
        .expect("OPENROUTER_API_KEY must be set");

    // Create an OpenRouter client.
    let client = OpenRouterClient::new()
        // The base URL must have a trailing slash.
        .with_base_url("https://openrouter.ai/api/v1/")?
        .with_http_referer("https://yourwebsite.com")
        .with_site_title("Your Application")
        .with_api_key(api_key);

    // Create a chat completion request using an available model.
    let request = ChatCompletionRequest {
        model: "openai/gpt-4o".to_string(),
        messages: vec![
            Message {
                role: "user".to_string(),
                content: "What is the meaning of life?".to_string(),
                name: None,
                tool_calls: None,
            }
        ],
        stream: None,
        response_format: None,
        // Optionally, provide a list of tools the model may call.
        tools: Some(vec![
            Tool::Function {
                function: FunctionDescription {
                    name: "get_current_weather".into(),
                    description: Some("Retrieve the current weather for a specified location".into()),
                    parameters: json!({
                        "type": "object",
                        "properties": {
                            "location": { "type": "string", "description": "City or location name" }
                        },
                        "required": ["location"]
                    }),
                }
            }
        ]),
        provider: None,
        models: None,
        transforms: None,
    };

    // Alternatively, if using the unified request builder, you can enable structured outputs:
    let schema_def = JsonSchemaDefinition {
        schema_type: "object".into(),
        properties: {
            let mut map = serde_json::Map::new();
            map.insert(
                "result".into(),
                json!({ "type": "string", "description": "The answer to the query" }),
            );
            map
        },
        required: Some(vec!["result".into()]),
        additional_properties: Some(false),
    };

    let json_schema_config = JsonSchemaConfig {
        name: "answer".into(),
        strict: true,
        schema: schema_def,
    };

    // Build request payload using the unified builder with structured output enabled.
    let request_payload = client
        .completion_request(vec![
            Message {
                role: "user".to_string(),
                content: "What is the meaning of life?".to_string(),
                name: None,
                tool_calls: None,
            }
        ])
        .with_structured_output(json_schema_config, true, false)
        .build();

    println!("Structured Request payload:\n{}", serde_json::to_string_pretty(&request_payload)?);

    // Invoke the chat completion endpoint.
    let response = client.chat_completion(request).await?;

    println!("Model used: {}", response.model);
    if let Some(choice) = response.choices.first() {
        println!("Response: {}", choice.message.content);
    }
    println!("Usage: {:?}", response.usage);

    Ok(())
}
```



### Running Tests

Before running tests, set the environment variable `OPENROUTER_API_KEY` with your API key:

```bash
export OPENROUTER_API_KEY=sk-...
cargo test
```

This will run the integration tests in `tests/integration_tests.rs`, which now include scenarios for structured outputs, tool calling, provider preferences, and the new web search endpoint.

## Implementation Plan

The project is under active development. The following roadmap outlines upcoming features and milestones:

### Phase 1: Core Functionality (Completed or In Progress)
- [x] **Client Foundation:**
  - Implement the type‑state builder for configuration.
  - Validate required parameters at compile time.
  - Build a basic HTTP client with custom headers.
- [x] **Chat Completion Endpoint:**
  - Implement the `chat_completion` method using the correct URL joining logic.
  - Provide basic error handling (e.g., decoding response JSON).
- [x] **Core Data Models:**
  - Implement types for chat messages, requests, responses, and usage.

### Phase 2: Additional Endpoints and Features
- [ ] **Streaming Support:**
  - Implement streaming versions of the chat completions using Server-Sent Events.
- [ ] **Text Completion Endpoint:**
  - Create an API module and associated types for text completions.
- [ ] **Models & Credits Endpoints:**
  - Implement endpoints to list available models and retrieve credit information.
- [x] **Tool Calling & Structured Outputs:**
  - Add support for tool calls, allowing the specification of callable functions.
  - Enable structured responses with JSON Schema validation and fallback behavior.
- [x] **Provider Preferences & Model Routing:**
  - Enable configuration for provider preferences, fallbacks, and routing options.
- [x] **Web Search Endpoint:**
  - Implement a new endpoint for web search queries.
  - Define strongly‑typed request and response models for search operations.

### Phase 3: Robust Testing & Documentation
- [ ] **Unit & Integration Tests:**
  - Expand test coverage with mocks and integration tests for each endpoint.
- [ ] **Documentation Improvements:**
  - Enhance inline documentation and API docs using rustdoc.
  - Provide additional usage examples in the `examples/` directory.
- [ ] **Continuous Integration (CI):**
  - Set up a CI pipeline for continuous builds and testing.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request with your ideas or improvements. When contributing, follow the code style guidelines and ensure that all tests pass.

## License

Distributed under either the MIT license or the Apache License, Version 2.0. See [LICENSE](LICENSE) for more details.

–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––––––
## Debugging and Verbose Test Output

For detailed logging during tests, you can run:

```bash
cargo test -- --nocapture
```

------------------------------------------------
