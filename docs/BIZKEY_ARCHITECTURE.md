# Bizkey AI on AnythingLLM — architecture review and implementation plan

**Repository:** `Cowork-Uni/anything-llm`  
**Development branch:** `dev_CoworkUni`  
**Status:** Architecture review and design only. No application behavior changed.

## Goal

Extend AnythingLLM as a local-first Bizkey AI foundation with permission-controlled **Knowledge Cells**, structured data access, and a reviewed, evidence-based **Company Memory**. Each customer must have a separate deployment and database; a workspace is **not** a customer security boundary.

## Confirmed existing architecture (source inspection)

| Area | Existing implementation | Key source |
| --- | --- | --- |
| UI | React, Vite | `frontend/src/`, `frontend/package.json` |
| API and business logic | Node.js/Express | `server/index.js`, `server/endpoints/` |
| Document pipeline | Separate collector and workspace embedding | `collector/`, `server/models/documents.js` |
| Workspace records | Prisma models | `server/prisma/schema.prisma`, `server/models/workspace.js` |
| Workspace membership | `workspace_users` mapping | `server/models/workspaceUsers.js` |
| Permissions | `admin`, `manager`, `default` role checks | `server/utils/middleware/multiUserProtected.js` |
| Vector stores | LanceDB, pgvector, other providers | `server/utils/vectorDbProviders/` |
| Existing memory | Personal/global/workspace scoped memory (limits apply) | `server/models/memory.js` |
| Docker | App port 3001, persisted storage via volumes | `docker/docker-compose.yml` |
| Local LLM | Ollama configured by `OLLAMA_BASE_PATH` | `docker/.env.example` |

### Design implications

1. **Workspace != Knowledge Cell (yet).** Build Knowledge Cells on top of workspaces for initial UX, but introduce explicit cell metadata, cell permissions, provenance and lifecycle before claiming full security isolation.
2. **Company Memory != existing memory.** Existing memory is user-scoped and size-limited; introduce an independent, auditable store of proposed/reviewed/rejected facts and conclusions.
3. **Keep transactional SQL separate from RAG.** Live analytical queries should use the customer SQL interface through an authenticated read-only MCP integration initially. Do not embed bulk sales transaction rows into the vector store.
4. **Customer isolation is deployment-level.** One customer per deployment, app database, vector namespace/data store, file storage, credentials and log scope. Never route two customers through the same app database.
5. **Document readiness is explicit.** The UI must show: uploaded → processed → embedded → available in cell, with failure/retry and provenance.
6. **Secure permissions at the API, not just the UI.** Extend server authorization for cell membership, document references, retrieval, memory reads/writes and background jobs. Review agent tools for scope leakage.
7. **Preserve upstream compatibility.** Avoid wholesale rewrites of core chat/collector interfaces; isolate additions in `server/utils/bizkey/`, `server/endpoints/bizkey/` (after route registration), and Bizkey-namespaced UI components.

## MVP design

### Knowledge Cells

Candidate metadata:
- `id`, `workspaceId`, `name`, `description`, `status`, `createdAt`, `updatedAt`
- `ownerUserId`, `classification`, `defaultModelPolicy`
- Permissions: `cellId`, `userId`, `role` (e.g. viewer/contributor/approver/manager)
- Scoped documents and evidence pointers; no cross-cell search by default

Suggested pilot cells: Finance, Sales, Strategy. Company Memory is a curated, separately permissioned capability; it may be presented as a fourth cell in the UI.

### Reviewed Company Memory

Proposed record model:
- `id`, `cellId`, `statement`, `kind` (fact, inference, decision)
- `status` (proposed, approved, rejected, superseded)
- `createdBy`, `approvedBy`, timestamps, validity interval
- `evidence[]` (source document ID, chunk/page, SQL query/run ID and snapshot parameters)
- `confidence`, `reviewNotes`, `supersedesId`, `version`

Only approved items may be injected as authoritative knowledge. AI-generated proposals must not self-approve. Every answer should distinguish sourced evidence from inference.

### Pilot deployment

- Windows 11 + Docker Desktop running AnythingLLM in a dedicated container.
- The already running Ollama instance on the Windows host through `http://host.docker.internal:11434`; verify connectivity in the container before configuration. Model name must be discovered, never assumed.
- Initial LanceDB for ease of local testing. Reevaluate pgvector later if relational governance or operational scale makes it valuable.
- Persistent application/collector volumes; keep secrets in non-versioned environment files.
- No exposure to the public Internet before authentication, authorization, storage, backup and threat review.

## Implementation phases and acceptance criteria

### Phase 0 — Secure baseline and boot
- Document a reproducible local setup; connect to Ollama; upload a sample PDF; ask a grounded question with citations.
- Verify the document persists after restart and does not appear in another workspace to an unauthorized user.
- Do not change `master`.

### Phase 1 — Knowledge Cell UX
- Introduce cell name/description/status and a clear **“Lägg till i kunskapscell”** upload action.
- Display ingestion progress and a distinct **Ready** state only after embedding succeeds.
- Keep existing chat, workspace, and document features usable.

### Phase 2 — Cell authorization
- Add server-enforced membership, document access, retrieval scoping, tool authorization and audit events.
- Test viewer vs contributor vs approver; test attempts at cross-cell access via direct APIs and agent tools.

### Phase 3 — Company Memory
- Store versioned candidate conclusions with evidence.
- Human approval workflow and source-backed retrieval of approved memory.
- Test corrections and superseded conclusions so stale facts are not presented as current.

### Phase 4 — Structured SQL pilot
- Integrate the existing Bizkey SQL MCP interface with least-privilege credentials.
- In pilot, query `dbo.Sales_V_total` through controlled, paginated, limited tools, using the correct spelling.
- Disallow arbitrary SQL and writes by default; introduce write access only as a separate, explicitly approved design.
- Save query provenance, timestamps and analytical conclusions, not large raw result sets.

### Phase 5 — Governance and costs
- Per-user and per-cell token usage, optional budgets and approvals.
- External model routing policy, audit, backups, recovery, and deployment packaging.
- Validate the one-customer-per-installation rule.

## Immediate next actions

1. Bring up the **unmodified** fork on the local development machine and verify Ollama connectivity.
2. Validate authentication and workspace isolation with two test users before relying on workspaces for sensitive material.
3. Implement the minimum Knowledge Cell UI and API on `dev_CoworkUni`, with focused tests and no production data.

## Licensing

The upstream `LICENSE` states MIT and requires retaining the original copyright and permission notice in copies or substantial portions. Review any third-party dependencies and assets separately.

## Cautions

This is a static source inspection. No runtime tests, dependency audit, penetration testing, deployment or database migration have been executed as part of this review.
