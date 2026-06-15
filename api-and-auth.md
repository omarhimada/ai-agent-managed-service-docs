# API and Auth Flow

## Cognito Token Access

Clients authenticate using Amazon Cognito. After sign-in, the client receives an access token and calls the Agent API with a bearer token.

```http
POST /v1/agents/{agentId}/messages
Authorization: Bearer <cognito_access_token>
Content-Type: application/json

{
  "conversationId": "conv_123",
  "message": "Summarize the latest onboarding guide.",
  "stream": true
}
```

## Token Validation Requirements

The Auth Middleware should validate:

- JWT signature using Cognito JWKS.
- Issuer matches the configured Cognito user pool.
- Token is not expired.
- Audience or client ID is trusted.
- Required scopes are present.
- Tenant and user context can be resolved safely.

## Suggested Claims

| Claim | Purpose |
| --- | --- |
| `sub` | Stable user identifier. |
| `client_id` or `aud` | Application client validation. |
| `scope` | API authorization. |
| `cognito:groups` | Role or group-based access. |
| `tenant_id` | Tenant routing, if added through custom claims. |

Do not trust tenant IDs sent as plain request body parameters unless they are checked against the authenticated user.

## Core API Endpoints

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/v1/agents` | `GET` | List agents available to the authenticated user. |
| `/v1/agents/{agentId}` | `GET` | Read agent configuration and capabilities. |
| `/v1/agents/{agentId}/messages` | `POST` | Send a message to an agent. |
| `/v1/agents/{agentId}/stream` | `POST` or `WS` | Stream model responses and tool events. |
| `/v1/conversations` | `GET` | List user conversations. |
| `/v1/conversations/{conversationId}` | `GET` | Read conversation history. |
| `/v1/knowledge-bases` | `GET` | List accessible knowledge bases. |
| `/v1/knowledge-bases/{kbId}/documents` | `POST` | Upload documents for RAG ingestion. |
| `/v1/knowledge-bases/{kbId}/jobs/{jobId}` | `GET` | Check ingestion or indexing status. |

## Example Agent Request

```json
{
  "conversationId": "conv_123",
  "input": {
    "type": "text",
    "text": "What changed in the mobile release notes?"
  },
  "context": {
    "channel": "mobile",
    "locale": "en-US"
  },
  "rag": {
    "enabled": true,
    "knowledgeBaseIds": ["kb_product_docs"],
    "topK": 8
  },
  "modelPolicy": {
    "preferredProvider": "bedrock",
    "fallbackProvider": "cohere"
  }
}
```

## Example Agent Response

```json
{
  "conversationId": "conv_123",
  "messageId": "msg_456",
  "answer": "The latest mobile release notes mention improvements to sign-in, offline sync, and push notification reliability.",
  "citations": [
    {
      "documentId": "doc_789",
      "title": "Mobile Release Notes",
      "chunkId": "chunk_12"
    }
  ],
  "usage": {
    "provider": "bedrock",
    "model": "configured-model-id",
    "inputTokens": 1420,
    "outputTokens": 184
  }
}
```
