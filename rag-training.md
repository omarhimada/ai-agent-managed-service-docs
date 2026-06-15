# RAG and Training Pipeline

## Purpose

The platform should support retrieval-augmented generation by ingesting customer, product, support, policy, or developer documentation into a searchable knowledge base. In this context, "training" means preparing knowledge for retrieval, not necessarily fine-tuning a foundation model.

## RAG Pipeline

1. **Upload / connect sources**
   - Files, PDFs, docs, web pages, support tickets, CRM records, code docs, or structured datasets.

2. **Store originals**
   - Save source files in object storage with tenant, source, checksum, and version metadata.

3. **Parse and normalize**
   - Extract text, tables, metadata, document structure, headings, and links.

4. **Chunk documents**
   - Split content into retrievable chunks using heading-aware and token-aware chunking.

5. **Generate embeddings**
   - Use AWS Bedrock embedding models, Cohere embeddings, or a tenant-specific configured provider.

6. **Index chunks**
   - Store vectors, chunk text, source references, permissions, and timestamps in the vector store.

7. **Evaluate retrieval quality**
   - Run test prompts, check citation quality, detect missing context, and tune chunking or reranking.

8. **Serve RAG at inference time**
   - Retrieve top chunks, optionally rerank with Cohere, assemble context, and pass it to Bedrock or Cohere generation.

## RAG Job States

| State | Meaning |
| --- | --- |
| `uploaded` | Source received and stored. |
| `parsing` | Content extraction is running. |
| `chunking` | Text is being split into retrievable chunks. |
| `embedding` | Embeddings are being generated. |
| `indexing` | Chunks and vectors are being committed. |
| `ready` | Knowledge base is available for retrieval. |
| `failed` | Pipeline failed and requires retry or admin review. |

## Retrieval Design

Each chunk should include:

```json
{
  "tenantId": "tenant_123",
  "knowledgeBaseId": "kb_product_docs",
  "documentId": "doc_456",
  "chunkId": "chunk_789",
  "text": "...",
  "embedding": [0.0123, -0.0456],
  "metadata": {
    "title": "Mobile Onboarding Guide",
    "sourceUri": "s3://bucket/path/file.pdf",
    "version": "2026-06-15",
    "visibility": "internal",
    "allowedGroups": ["support", "product"],
    "createdAt": "2026-06-15T12:00:00Z"
  }
}
```

## Query-Time Flow

1. Authenticated user sends prompt.
2. Agent Orchestrator resolves tenant, user, groups, and accessible knowledge bases.
3. Retrieval Service embeds the query.
4. Vector Store returns candidate chunks filtered by tenant and permissions.
5. Optional reranking step improves context relevance.
6. Orchestrator builds the model prompt with retrieved context and citations.
7. Model Router invokes Bedrock or Cohere.
8. Agent returns answer with citations and usage metadata.

## Reranking

Cohere can be used as a reranking provider after vector retrieval. This is useful when the vector search returns semantically similar but not necessarily answer-relevant chunks.

Example retrieval stack:

- Embedding provider: Bedrock or Cohere.
- Initial retrieval: vector top 50.
- Reranking provider: Cohere.
- Final context: top 5 to 10 chunks.
- Generation provider: Bedrock or Cohere.

## Fine-Tuning vs RAG

Use RAG when the agent needs fresh, tenant-specific, source-citable knowledge. Use fine-tuning only when the objective is behavior, style, classification patterns, or task-specific response format that cannot be solved through retrieval and prompting alone.
