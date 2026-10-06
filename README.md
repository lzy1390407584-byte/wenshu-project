# Wenshu: A Governed Multi-Department Policy Assistant

Wenshu is an open-source reference implementation of a multi-department policy and document assistant for universities and other organizations. It combines document ingestion, grounded hybrid retrieval, controlled agent orchestration, evidence verification, layered memory, and a feedback-driven strategy loop.

The project is designed for policy-heavy environments where an answer must be traceable to the right department, document version, and active clause—not merely plausible.

> This repository is a research and engineering reference project. Review authentication, secrets, data retention, and deployment configuration before using it with real users or sensitive documents.

## What it provides

- Document ingestion with parsing, cleaning, metadata extraction, version tracking, chunking, deduplication, and conflict detection.
- Hybrid retrieval using BM25 and vector search, rank fusion, optional reranking, and active-version validation.
- A fixed `Intent → Rewrite → Retrieval → Answer → Verify` workflow with explicit timeouts, retries, and safe fallbacks.
- Department-aware routing and isolation for centralized orchestration or separately deployed department agents.
- Five memory planes: working, episodic, user semantic, organizational knowledge, and procedural strategy memory.
- Evidence-grounded answers with source references and verifier-enforced refusal when evidence is insufficient.
- An `Execute → Observe → Reflect → Adapt → Deploy` loop that turns feedback into testable Skills, Hooks, and Rules.
- Docker Compose for local full-stack development and Kubernetes/Helm manifests for service deployment.

## Architecture

```text
┌───────────────┐       REST        ┌─────────────────────────┐
│ Next.js web UI│ ─────────────────>│ FastAPI orchestrator    │
└───────────────┘                  │ auth · routing · DAG    │
                                   │ facts · memory · policy │
                                   └────────────┬────────────┘
                                                │
                           ┌────────────────────┼───────────────────┐
                           ▼                    ▼                   ▼
                    Department Agent     Department Agent     pi Agent Runtime
                           └────────────────────┬───────────────────┘
                                                ▼
                              MongoDB · Redis · vector backend
```

The Python control plane owns permissions, department boundaries, active-document checks, memory policy, rollout decisions, and rollback. The pi service executes model-backed steps and constrained tool calls; it does not define data access or policy publication.

## Repository layout

```text
.
├── backend/                 FastAPI application, retrieval, memory, loop, and tests
├── services/pi-agent/       TypeScript model-runtime service
├── web/                     Next.js and React frontend
├── deploy/                  Kubernetes manifests and Helm chart
├── docs/                    Architecture, API, deployment, and loop documentation
├── design_files/            Design notes and diagrams
├── department_files/        Local, user-provided source documents (not tracked)
├── loadtest/                Department-agent load tests and result analysis
├── docker-compose.yml       Local multi-service environment
└── .env.example             Safe configuration template
```

## Quick start with Docker

Prerequisites: Docker Desktop with Compose, and valid model-provider credentials.

```bash
git clone <your-repository-url>
cd wenshu-project
cp .env.example .env
# Edit .env and add your own credentials and strong local secrets.
docker compose up --build -d
docker compose ps
```

Available endpoints after startup:

| Service | URL |
| --- | --- |
| Web application | <http://localhost:8080> |
| Backend API and OpenAPI | <http://localhost:8000/docs> |
| pi agent health check | <http://localhost:8100/health> |
| MongoDB | `localhost:27017` |

Seed the sample departments and ingest approved local documents:

```bash
docker compose exec backend python -m scripts.seed_data
docker compose exec backend python -m scripts.ingest_department_files --base /app/department_files
```

The seed process initializes departments, terminology, calendar data, and baseline executable Skills. The admin Loop UI tracks asynchronous work through `queued → running → completed` or `failed` states.

Stop the stack with:

```bash
docker compose down
```

Use `docker compose down -v` only when you intentionally want to remove local database volumes.

## Local development

Recommended versions: Python 3.11 and Node.js 22 or later.

### Backend

```bash
cd backend
python -m venv .venv
# Windows PowerShell: .\.venv\Scripts\Activate.ps1
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
$env:STORAGE_MODE = "memory"  # PowerShell; use export on macOS/Linux
uvicorn app.main:app --reload --port 8000
```

### pi agent

```bash
cd services/pi-agent
npm install
npm run dev
```

### Web frontend

```bash
cd web
npm install
# Set BACKEND_URL=http://localhost:8000 in your shell or local environment.
npm run dev
```

## Configuration and security

1. Copy `.env.example` to `.env` and provide credentials locally.
2. Never commit `.env`, API keys, database passwords, signing secrets, or Kubernetes secret files.
3. Set `SEED_DEMO_USERS=false` outside local development.
4. Replace all development secrets, including `AUTH_SECRET`, `INTERNAL_API_TOKEN`, MongoDB credentials, and the Redis password.
5. Restrict `CORS_ORIGINS` to the actual frontend origin.
6. Treat uploaded documents and evaluation data as potentially sensitive.

The repository intentionally tracks only `.env.example`; local `.env` files are ignored by Git.

Important configuration groups include:

| Group | Examples | Purpose |
| --- | --- | --- |
| Models | `DEEPSEEK_*`, `RELAY_*`, `EMBEDDING_*` | Chat, embedding, and reranking providers |
| Storage | `MONGODB_*`, `REDIS_*`, `STORAGE_MODE` | Persistent or in-memory runtime state |
| Retrieval | `VECTOR_BACKEND`, `BM25_TOP`, `VECTOR_TOP`, `RERANKER_*` | Hybrid retrieval behavior |
| Runtime | `PI_AGENT_ENABLED`, `PI_AGENT_URL` | Model-backed agent execution |
| Governance | `AUTH_*`, `DEPT_ID`, `DEPT_AGENTS_ENABLED` | Authentication and department isolation |
| Loop | `LOOP_*`, `SKILL_*`, `HOOK_*` | Feedback mining, evaluation, and rollout |

## Evaluation

Run the backend tests:

```bash
cd backend
pytest -q
```

Run the frontend and pi-agent build checks from the repository root:

```bash
cd web && npm run build
cd ../services/pi-agent && npm run build
```

The evaluation entry point is:

```bash
cd backend
python -m scripts.evaluate_rag
```

The checked-in evaluation fixture is `backend/evaluation/real_document_qa.json`. Load-test instructions are in [`loadtest/README.md`](loadtest/README.md).

## Design principles

- **Facts are separate from memory.** MongoDB-backed active document clauses are the authoritative evidence plane; memories and model outputs cannot silently become official policy.
- **Recall and precision are complementary.** BM25 handles identifiers, dates, and exact phrases; vector search handles semantic variation; fusion and reranking combine their strengths.
- **Deterministic boundaries surround probabilistic reasoning.** The harness owns permissions, tool access, state transitions, timeouts, verification, and rollback.
- **Degrade in capability, not in trust.** If generation, reranking, vector search, or a department service fails, the system returns a safer lower-capability result or an explicit limitation.
- **Feedback must change execution.** A strategy is not considered effective until replay, rollout records, and outcome metrics show that it changed the next execution path.

## Further documentation

- [System architecture](docs/architecture.md)
- [API guide](docs/api.md)
- [Deployment guide](docs/deployment.md)
- [Loop Engineering](docs/loop-engineering.md)
- [Backend guide](backend/README.md)
- [pi agent guide](services/pi-agent/README.md)
- [Frontend guide](web/README.md)
- [Kubernetes and Helm deployment](deploy/README.md)

## License

No license file is currently included. Add an explicit license before accepting external contributions or redistributing the project.
