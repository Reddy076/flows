# AI Service — Complete Feature List
> This document lists all features, endpoints, and integrations handled by `ai-service`.
> Reference this to understand the full scope of what the AI microservice does.

---

## 1. Core AI Capabilities (`/api/ai`)

Exposes artificial intelligence features for the password manager, leveraging external Large Language Models (LLMs) to analyze, categorize, and assist users.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| **Service Health Check** | `GET` | `/api/ai/health` | `AiController` / `LlmClientService` |
| **Analyze Password** | `POST` | `/api/ai/analyze-password` | `PasswordAnalysisService` |
| **Categorize Vault Entry** | `POST` | `/api/ai/categorize-entry` | `SmartVaultCategorizationService` |
| **Security Chatbot** | `POST` | `/api/ai/chat` | `SecurityChatbotService` |

---

## Feature Details & Key Design Notes

### Health Check (`/api/ai/health`)
- Verifies if the AI service is operational and if it can successfully connect to the configured LLM provider.
- Returns the provider name and an `UP` or `DOWN` status mapping to a 200 or 503 HTTP status code respectively.

### Password Analysis (`/api/ai/analyze-password`)
- Evaluates a password's strength conceptually rather than just using regex rules.
- Asks the LLM to return exactly:
  1. A strength rating (`VERY_WEAK`, `WEAK`, `MODERATE`, `STRONG`, `VERY_STRONG`)
  2. Specific vulnerabilities (e.g. "Contains common dictionary words")
  3. Actionable improvement suggestions.
- **Graceful Fallback:** If no API key is configured or the LLM call fails, the service drops back to a `analyzePasswordLocally()` method. This local fallback uses regex (length, upper, lower, special, digits, and a common password list) to provide deterministic analysis without crashing.

### Vault Categorization (`/api/ai/categorize-entry`)
- Given a `URL`, `username`, and `title`, asks the LLM to infer the nature of the credential.
- Returns a standard category enum (e.g., `WORK`, `FINANCE`, `ENTERTAINMENT`) and an array of relevant search tags, along with a confidence score.
- Returns a default `"OTHER"` classification if the LLM JSON parsing fails.

### Security Chatbot (`/api/ai/chat`)
- A conversational assistant scoped specifically to password security best practices.
- The system prompt forbids the chatbot from encouraging poor security practices.
- **Intent Detection:** Uses simple parsing of the user's message to detect intents like `"GENERATE_PASSWORD"` or `"SECURITY_ANALYSIS"`.
- If the user explicitly asks for a new password, the `SecurityChatbotService` generates a strong 16-character string securely using `java.security.SecureRandom` locally (not by the LLM) and attaches it to the response.

---

## 2. Cross-Cutting / AI Infrastructure

Features and configurations that support the core endpoints.

| Feature | Class | Role |
| :--- | :--- | :--- |
| **LLM Configuration** | `LlmConfig` | Binds configuration properties (API key, model name, max tokens, temperature) from `application.yml`. |
| **HTTP Client Config** | `OkHttpClientConfig` | Defines standard timeouts (e.g., 30s read timeout) for LLM API calls using OkHttp3. |
| **LLM Agnostic Pattern** | `LlmProvider` (Interface) | Defines a common interface layer so the app isn't hardcoded to a specific vendor. |
| **OpenAI Compatible Client** | `OpenAiProvider` | Implements `LlmProvider` to talk to any API that implements the OpenAI spec (e.g. Groq, local Ollama, or OpenAI themselves). |
| **Service Facade** | `LlmClientService` | The main bean all other services talk to; abstracts away the Provider implementation. |
| **CORS Policy** | `CorsConfig` | Allows the Angular frontend (`http://localhost:4200`) to call the AI service. |

**Key Design Notes:**
- **No Database:** `ai-service` is entirely stateless. It does not connect to MySQL; it only receives HTTP requests, pipes them to an LLM, parses the JSON responses, and returns HTTP responses.
- **Prompt Engineering:** The services use heavy "System Prompts" that explicitly instruct the LLM to skip conversational filler and *only* return strictly formatted JSON matching the DTO structures.
- **Serialization:** Uses Jackson `ObjectMapper` to parse the LLM text output back into Java Objects like `PasswordAnalysisResponse`.
