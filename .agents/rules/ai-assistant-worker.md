# Cloudflare Worker & Google Gemini Assistant Guidelines

## 1. Cloudflare Worker Endpoint & Routing
- **Live Worker URL**: `https://rapid-frog-ec25.lactuong64.workers.dev/`
- **Frontend Target**: Always verify that `assets/vendor.min.js` decodes to `https://rapid-frog-ec25.lactuong64.workers.dev/`.
- **Typo Protection**: Never use `5c25` or `lactxongx64` — the correct domain is `rapid-frog-ec25.lactuong64.workers.dev`.

## 2. Google Gemini API Authentication (Crucial Gotcha)
- **Key Prefix**: Google AI Studio 2026 keys start with `AQ.Ab8RN...`.
- **Required Header**: Must pass API Key via Header `x-goog-api-key: <KEY>`.
- **Do NOT**:
  - Do NOT pass as `Authorization: Bearer <KEY>` (Google will return `401: Invalid authentication credentials`).
  - Do NOT rely solely on `?key=...` for new `AQ.` keys.
- **Always Send Headers**:
  ```javascript
  headers: {
    "Content-Type": "application/json",
    "x-goog-api-key": apiKey
  }
  ```

## 3. Gemini Model Endpoints
- **Primary Model**: `gemini-flash-lite-latest` (fastest ~1s, lowest latency, zero 503 errors).
- **Fallback Model**: `gemini-flash-latest`.
- **Endpoint URL**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-flash-lite-latest:generateContent`
- **Avoid Deprecated Names**: Do not call `gemini-2.0-flash` or `gemini-2.5-flash` directly as aliases rotate in v1beta.

## 4. Response Parsing & Emotion Extraction
- **Content Parts Extraction**: Always iterate over all `content.parts` and aggregate `part.text` (do not just read `parts[0]` because metadata/thoughts may precede the text).
- **Emotion Tags**: Extract `[EMOTION: <tag>]` on both Worker and Client (`vendor.min.js`), then strip the tag from user-visible text with regex `/\[EMOTION:\s*[a-zA-Z_-]+\]/gi`.
