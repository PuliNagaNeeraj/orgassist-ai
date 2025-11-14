# OrgAssist AI

[![Project Status](https://img.shields.io/badge/status-active-brightgreen.svg)]()
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)]()
[![Contributors](https://img.shields.io/badge/contributors-welcome-orange.svg)]()

OrgAssist AI is an assistant for teams and organizations that uses large language models (LLMs) and retrieval techniques to help summarize, automate, and organize work across repositories, issues, docs, and knowledge bases. It provides tools to ingest organizational artifacts (issues, PRs, documents), build retrievable knowledge (embeddings + vector store), and expose conversational and programmatic interfaces to surface context-aware answers, generate task templates, and propose automated actions.

This README gives an overview of the project, how it works, and how to get started locally and in production.

Table of contents
- About
- Key features
- High-level architecture
- Quick start
  - Requirements
  - Local development (Node, Python, or describe stack)
  - Docker Compose
- Configuration (environment variables)
- Usage examples
  - CLI
  - REST / HTTP API
  - Web UI (if available)
- Data ingestion and connectors
- How it works (pipeline)
- Testing
- Deployment
- Contributing
- Roadmap
- Troubleshooting & FAQ
- License

About
-----
OrgAssist AI helps teams:
- Quickly get summaries of repository and issue activity.
- Ask natural-language questions about your organization’s knowledge base or repos.
- Generate task templates and suggested actions (e.g., label PRs, open issues).
- Run automated workflows or human-in-the-loop suggestions using LLMs + retrieval.

Key features
------------
- Ingest content: GitHub issues/PRs, Markdown docs, meeting notes, knowledge bases.
- Embeddings and vector store for retrieval (Pinecone / Weaviate / Milvus / local FAISS).
- Conversational interface and API for question-answering over internal content.
- Automated suggestions and templated responses (issue templates, PR summaries).
- Extensible connector model to add new sources.
- Role-based access and audit logging (recommended best practices).

High-level architecture
-----------------------
A typical OrgAssist AI deployment consists of:
1. Ingestors / Connectors
   - Pulls data from GitHub, Google Drive, Notion, Confluence, or local file systems.
2. Preprocessing
   - Cleans, chunks, and transforms documents for embedding.
3. Embedding & Vector Store
   - Generates embeddings (OpenAI, HuggingFace, or other) and stores them in a vector DB.
4. Retriever + Reranker
   - Retrieves top-k relevant passages and optionally reranks by relevance.
5. LLM Orchestration
   - Prompts the LLM with context, user query, and system instructions to generate responses.
6. API / UI / Workers
   - Exposes REST endpoints and optional UI; background workers handle long tasks and batching.

Quick start
-----------

Requirements
- Node 16+ or 18+ (or Python 3.8+ if parts of the repo are Python-based)
- Docker & Docker Compose (recommended for quick reproducible runs)
- An account / API key for the embedding + LLM provider you intend to use (e.g., OpenAI, Azure OpenAI, or an open model)
- Postgres (or another relational DB) and Redis (optional, for background workers / cache)

Clone the repo
```bash
git clone https://github.com/TirupathiJayadeep/orgassist-ai.git
cd orgassist-ai
```

Local development (example Node/Express + React)
1. Copy the example env file:
```bash
cp .env.example .env
# Edit .env to add keys and connection strings
```

2. Install dependencies (server & client)
```bash
# If server is in /server and client in /web
cd server
npm install

cd ../web
npm install
```

3. Run local services (example)
```bash
# Start the database, Redis and vector DB (if using Docker Compose)
docker-compose up -d

# Launch backend
cd server
npm run dev

# Launch frontend
cd ../web
npm run start
```

Docker Compose
--------------
A provided docker-compose.yml can bootstrap:
- Postgres
- Redis
- Vector store (e.g., Pinecone emulator or Weaviate)
- Web + API

Start:
```bash
docker-compose up --build
```

Configuration (environment variables)
-------------------------------------
The project reads configuration from environment variables. Example variables:

General:
- PORT=3000
- NODE_ENV=development

Auth and security:
- JWT_SECRET=change-me
- SESSION_SECRET=change-me

Database:
- DATABASE_URL=postgres://user:pass@localhost:5432/orgassist

Redis:
- REDIS_URL=redis://localhost:6379

Embeddings & LLM:
- EMBEDDING_PROVIDER=openai | huggingface | local
- OPENAI_API_KEY=sk-...
- HUGGINGFACE_API_KEY=hf_...

Vector store:
- VECTOR_STORE=pinecone | weaviate | milvus | faiss
- PINECONE_API_KEY=...
- PINECONE_ENV=...

GitHub connector:
- GITHUB_APP_ID=
- GITHUB_PRIVATE_KEY= (PEM)
- GITHUB_WEBHOOK_SECRET=
- GITHUB_ACCESS_TOKEN=

Monitoring & Sentry:
- SENTRY_DSN=

Workers:
- WORKER_CONCURRENCY=4

Make sure to update .env with values relevant to your deployment.

Usage examples
--------------

1) Question answering via HTTP API
Assuming an endpoint POST /api/v1/query:

Request:
```bash
curl -X POST "http://localhost:3000/api/v1/query" \
  -H "Authorization: Bearer <YOUR_JWT_OR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the onboarding process for new hires?",
    "context_size": 5
  }'
```

Response (example):
```json
{
  "answer": "The onboarding process consists of the following steps: 1) Complete HR paperwork; 2) Set up dev environment; 3) Team intro; 4) Assigned onboarding mentor. See: docs/onboarding.md",
  "sources": [
    { "id": "docs/onboarding.md", "score": 0.93, "excerpt": "..." }
  ]
}
```

2) CLI example
```bash
# Query via CLI
npm run cli -- query "How do I deploy to staging?"
```

3) Ingest a repository (GitHub)
```bash
# Trigger ingestion (could be a webhook)
curl -X POST "http://localhost:3000/api/v1/ingest" \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "source": "github", "owner": "org", "repo": "repo-name", "since": "2025-01-01" }'
```

Data ingestion and connectors
-----------------------------
Connectors ingest raw data from external sources and normalize it into documents. Typical connector features:
- Incremental sync (only changed content)
- Rate limiting and backoff
- Chunking strategy (token-based or sentence-based)
- Metadata extraction (repo, path, author, timestamp)

Supported connectors (examples; add as implemented):
- GitHub repos, issues, PR descriptions
- Google Docs
- Notion
- Confluence
- Local / S3 Markdown documents

How it works (pipeline)
-----------------------
1. Fetch documents using connectors.
2. Clean and split documents into chunks optimized for the embedding model.
3. Generate embeddings for chunks and upsert into the configured vector store.
4. When a user query arrives:
   - Run a vector similarity search to retrieve the top-K relevant chunks.
   - Construct a prompt combining the user question, retrieved contexts, and system instructions.
   - Query the LLM for a final answer, optionally applying a reranker or safety filters.
5. Return answer and provenance (sources + excerpts).

Testing
-------
Run unit and integration tests:

```bash
# From repo root
npm test
# or
npm run test:watch
```

For integration tests that require external services (database, vector store), use Docker Compose to bring services up before running tests:
```bash
docker-compose up -d
npm run test:integration
```

CI / GitHub Actions
-------------------
A recommended CI workflow:
- Run linting and type checks
- Run unit tests
- Build Docker images in CI for the backend and frontend
- Run integration tests against ephemeral services (Postgres, Redis)
- If all checks pass, push images to container registry and/or create a release

Security & secrets
------------------
- Never store provider API keys or private keys in the repository.
- Use secrets management (GitHub Secrets, Vault, AWS Secrets Manager).
- Ensure minimal required OAuth scopes for GitHub tokens.
- Validate and verify all incoming webhooks using shared secrets.

Deployment
----------
- Use Docker images to deploy to Kubernetes, ECS, or other orchestrators.
- Configure autoscaling for worker pods that perform ingestion and embedding tasks.
- Use managed vector DB (Pinecone, Weaviate Cloud) for production scale when possible.
- Back up Postgres and vector store metadata regularly.

Contributing
------------
Contributions are welcome! Please follow these guidelines:
1. Fork the repo and create a feature branch.
2. Open a PR with a clear description and link to any related issue.
3. Run tests and ensure CI passes.
4. Be respectful and follow the Code of Conduct.

## Suggested workflow:
- Open an issue describing the feature or bug.
- Discuss design in the issue if it’s a larger change.
- Submit a PR that references the issue.

Roadmap
-------
Planned improvements:
- Additional connectors (Slack, Jira).
- Role-based access control (RBAC) and enterprise SSO.
- Advanced analytics and usage dashboards.
- Offline / on-prem embeddings and LLM inference.
- Templates marketplace for common org workflows.

Troubleshooting & FAQ
---------------------
- Q: Why are answers hallucinating?
  A: Ensure your retriever returns high-quality context. Increase context_size or add stricter reranking. Also verify the prompt engineering and temperature settings.

- Q: My embeddings/queries are slow.
  A: Check rate limits of the embedding provider. Use batching for embeddings and a local or managed vector store optimized for your scale.

- Q: How do I add a new connector?
  A: Implement the connector interface (fetch -> normalize -> chunk -> metadata). Add configuration and unit tests.

Privacy and compliance
----------------------
- Treat ingested content as sensitive. Respect data retention policies.
- If processing PII, ensure compliance with relevant regulations (GDPR, CCPA).
- Consider using private LLM instances or on-prem inference for sensitive corpora.

