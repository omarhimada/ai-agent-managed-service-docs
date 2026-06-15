# Architecture

## System Overview

The AI Agent managed service exposes a secure API to web and mobile applications. Clients authenticate through Amazon Cognito and call the Agent Gateway using a bearer token. The backend validates the token, resolves tenant and user context, routes requests to the Agent Orchestrator, and selects either AWS Bedrock, Cohere API, or both depending on workload and policy.

The service supports two primary runtime paths:

1. **Agent inference path**: Handles user prompts, tool calls, memory lookup, RAG retrieval, guardrails, and response generation.
2. **RAG training path**: Handles document ingestion, chunking, embedding generation, indexing, metadata tagging, and knowledge-base refresh.

## Microservices

| Service | Responsibility |
| --- | --- |
| Web App / Mobile App | User-facing clients that authenticate with Cognito and call the Agent API. |
| Amazon Cognito | User pool, identity federation, JWT token issuance, app client management. |
| API Gateway / Load Balancer | Public HTTPS entry point for REST, GraphQL, or WebSocket traffic. |
| Auth Middleware | Validates Cognito JWTs, extracts claims, resolves tenant/user permissions. |
| Agent Gateway Service | Normalizes client requests, applies rate limits, handles request routing. |
| Agent Orchestrator Service | Plans agent steps, invokes tools, performs RAG retrieval, manages model selection. |
| Model Router Service | Routes model calls to AWS Bedrock or Cohere based on provider policy, latency, cost, task type, and availability. |
| RAG Retrieval Service | Searches vector indexes and document metadata for context. |
| Ingestion Service | Accepts documents, URLs, structured records, and batch uploads for RAG processing. |
| Embedding Service | Generates embeddings using Bedrock embedding models, Cohere embeddings, or configured provider. |
| Vector Store | Stores embeddings and document chunks for semantic retrieval. |
| Metadata Store | Stores tenants, users, agent configs, documents, permissions, and audit records. |
| Object Storage | Stores original documents, processed files, chunk manifests, and training artifacts. |
| Queue / Event Bus | Decouples ingestion, chunking, embedding, indexing, and async agent tasks. |
| Observability Service | Logs, traces, metrics, token usage, model latency, and evaluation results. |

## Recommended AWS Stack

| Layer | AWS / Service Option |
| --- | --- |
| Identity | Amazon Cognito |
| API entry | Amazon API Gateway or Application Load Balancer |
| Compute | ECS Fargate, EKS, or AWS Lambda depending on workload |
| Async processing | Amazon SQS, EventBridge, or SNS |
| Object storage | Amazon S3 |
| Metadata database | Amazon DynamoDB or Amazon Aurora PostgreSQL |
| Vector database | Amazon OpenSearch Serverless vector engine, Aurora PostgreSQL with pgvector, Pinecone, Weaviate, or Qdrant |
| LLM provider | Amazon Bedrock and Cohere API |
| Secrets | AWS Secrets Manager |
| Observability | Amazon CloudWatch, AWS X-Ray, OpenTelemetry, and a log analytics stack |

## Model Strategy

The platform should treat Bedrock and Cohere as interchangeable provider integrations behind a Model Router.

Example routing policies:

- Use Bedrock for AWS-native security, private networking, and organization-approved foundation models.
- Use Cohere for embeddings, reranking, multilingual retrieval, or generation tasks where Cohere models are preferred.
- Allow per-tenant configuration for allowed providers, model IDs, max tokens, temperature, and fallback behavior.
- Track provider latency, error rate, token usage, and cost per request.

## Security Boundaries

- All client calls require a valid Cognito JWT.
- Backend services use IAM roles and service-to-service authentication.
- Tenant ID must be derived from trusted claims or server-side lookup, not from unchecked client input.
- RAG retrieval must filter by tenant, workspace, user permissions, and document visibility.
- Provider API keys and Bedrock access must remain server-side only.
- Store audit logs for prompts, document access, tool calls, model usage, and admin actions.

## Runtime Request Flow

1. User signs in with Cognito from web or mobile.
2. Client receives Cognito access token.
3. Client calls Agent API with `Authorization: Bearer <token>`.
4. Auth Middleware verifies token signature, issuer, audience, expiry, and scopes.
5. Agent Gateway applies rate limits and creates a request context.
6. Agent Orchestrator builds the agent plan.
7. RAG Retrieval Service fetches relevant chunks from the vector store.
8. Model Router calls Bedrock or Cohere API.
9. Response is post-processed, logged, and returned to the client.
