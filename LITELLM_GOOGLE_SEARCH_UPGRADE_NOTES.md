# Upgrading OpenClaw: LiteLLM Google Search Grounding Proxy Fixes

When upgrading OpenClaw to `v2026.9.7` or newer, if you route Google Web Search grounding requests through a local LiteLLM proxy (e.g. `http://192.168.249.129:4000` or `http://localhost:4000`) and target Vertex AI or Gemini, you must apply the following overrides in `extensions/google/src/gemini-web-search-provider.runtime.ts` and `extensions/google/src/gemini-web-search-provider.shared.ts`.

---

## The Issues & Solutions

### 1. SSRF Private IP Block (Both Gemini & OpenAI-Compatible paths)

By default, OpenClaw blocks outgoing web search tool requests to private networks (SSRF prevention) using `withTrustedWebSearchEndpoint`.

- **Symptom:** `Blocked hostname or private/internal/special-use IP address`
- **Fix:** Replace `withTrustedWebSearchEndpoint` with `withSelfHostedWebSearchEndpoint` so private proxy IP addresses are permitted.

### 2. LiteLLM Google Search Tool Mapping (`openai-compatible` path)

Standard OpenAI payloads use `tools: [{ google_search: ... }]`. However, LiteLLM's Vertex AI translator strictly expects `googleSearch` (camelCase) to map the payload to Vertex AI Grounding.

- **Symptom:** Vertex AI/LiteLLM rejects or ignores the grounding tool.
- **Fix:** Update the request payload to use `googleSearch`:
  ```typescript
  body: JSON.stringify({
    model: params.model,
    messages: [{ role: "user", content: params.query }],
    tools: [{ googleSearch: googleSearch }],
  }),
  ```

### 3. Missing Explicit Role in Payload (`gemini` path)

Standard Gemini REST payloads may omit the explicit `role` when sending a single message, defaulting to `user`. However, LiteLLM's Gemini-to-Vertex parser strictly validates roles. Omitting it throws a role validation error.

- **Symptom:** `litellm.BadRequestError: Vertex_aiException BadRequestError - Please use a valid role: user, model.`
- **Fix:** Add `role: "user"` explicitly to the contents payload:
  ```typescript
  body: JSON.stringify({
    contents: [{ role: "user", parts: [{ text: params.query }] }],
    tools: [{ google_search: googleSearch }],
  }),
  ```

### 4. OpenAI-Compatible Proxy Routing & Automatic Detection

If `plugins.entries.google.config.webSearch.apiType` or `GEMINI_API_TYPE` is set to `openai-compatible`, or if `baseUrl` contains `/v1` and is not `googleapis.com`, requests are dispatched to `${baseUrl}/chat/completions` using Bearer authentication and OpenAI-compatible schema with citations extracted from `grounding_metadata` / `groundingMetadata`.

### 5. Environment Variable Fallbacks

Search base URL resolves from:

- `plugins.entries.google.config.webSearch.baseUrl`
- `models.providers.google.baseUrl`
- `GOOGLE_GEMINI_ENDPOINT`
- `GEMINI_BASE_URL`
- `GOOGLE_GEMINI_BASE_URL`

API key resolves from:

- `plugins.entries.google.config.webSearch.apiKey`
- `GEMINI_API_KEY`
- `GOOGLE_API_KEY`
- `models.providers.google.apiKey`
