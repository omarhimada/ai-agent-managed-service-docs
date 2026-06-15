# Deployment Notes

## Suggested Deployment Model

Use independently deployable services with shared platform infrastructure.

| Component | Deployment Option |
| --- | --- |
| Agent Gateway | ECS Fargate, EKS, or Lambda behind API Gateway. |
| Agent Orchestrator | ECS Fargate or EKS for long-running and streaming workloads. |
| RAG Retrieval Service | ECS Fargate or EKS with low-latency access to vector store. |
| Ingestion Workers | ECS Fargate tasks, Lambda, or Kubernetes jobs. |
| Embedding Workers | Queue-driven workers with concurrency controls. |
| Model Router | Internal service with provider adapters and fallback logic. |

## Environments

Recommended environments:

- `dev`: fast iteration, lower rate limits, synthetic data.
- `staging`: production-like, integration testing, load testing.
- `prod`: tenant-isolated, monitored, backed up, audited.

## Configuration

Keep these configurable per environment and per tenant:

- Cognito user pool ID and app client ID.
- Allowed origins for web and mobile clients.
- Allowed model providers.
- Bedrock model IDs.
- Cohere model names.
- Max request tokens and output tokens.
- RAG top-K and reranking policy.
- Data retention period.
- Rate limits and quota limits.

## Secrets

Store secrets in AWS Secrets Manager:

- Cohere API key.
- Third-party tool credentials.
- Database credentials.
- Webhook signing secrets.

Do not expose provider keys to web or mobile clients.

## Observability

Track these metrics:

- API latency and error rate.
- Auth failures.
- Model provider latency.
- Token usage by tenant, agent, provider, and endpoint.
- RAG retrieval latency.
- Empty retrieval rate.
- Citation coverage.
- Ingestion job duration and failure rate.
- Queue depth and worker retries.

## Reliability Patterns

- Use provider fallback in the Model Router.
- Use queues for ingestion and embeddings.
- Add idempotency keys for document upload and indexing jobs.
- Use dead-letter queues for failed ingestion tasks.
- Cache frequently accessed agent configs and JWKS keys.
- Apply circuit breakers around Bedrock and Cohere calls.
- Support streaming and non-streaming response modes.

## Multi-Tenant Controls

- Enforce tenant isolation in every database and vector query.
- Include tenant ID in partition keys where appropriate.
- Use permission metadata in every document chunk.
- Keep audit logs immutable or append-only.
- Support tenant-level provider allowlists and budget limits.
