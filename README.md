# AutoVaultLK — Project Documentation

> **Second-Hand Vehicle Marketplace with Intelligent Search and Automated Inventory Processing**
> Department of Computer Science and Engineering, University of Moratuwa — Group 25 (PID 11)

This repository holds the full documentation set for **AutoVaultLK**: the proposal and feasibility
study, the requirements and architecture specifications, the sprint plan, the test plan, the build
and test instructions, the final report, and the end-user manuals.

It is a **documentation repository** — the source code lives in the project's service repositories.

---

## About the project

Second-hand vehicles in Sri Lanka are advertised on platforms that store data inconsistently and
search it with fixed filters only. Dealers re-type inventories listing by listing, the same make is
spelt many ways, and a buyer who asks for a *"family friendly vehicle"* gets poor results.

AutoVaultLK addresses both problems:

- **Hybrid search** — a rules-first natural-language parser resolves a free-text query into exact SQL
  filters, uses trigram matching for misspelt words, calls a language model only when confidence
  falls below 0.6 (and whitelists everything it returns), and ranks the remaining descriptive text
  with 384-dimensional sentence embeddings. Hard constraints stay exact; only ranking is semantic.
- **Automated bulk ingestion** — a dealer uploads a CSV/JSON inventory file plus an optional ZIP of
  images. An ETL pipeline validates and normalises each row against a curated vehicle dictionary,
  generates search text and embeddings, processes and matches the images by registration number,
  and records progress and rejected rows with reasons. Imported listings are held in
  *pending review*, so automation never publishes unchecked data.

### Roles

| Role | Capabilities |
|------|--------------|
| **Guest** | Browse the catalogue, filter and natural-language search, view vehicle details |
| **Buyer** | Everything a guest can do, plus favourites, saved searches and recommendations |
| **Dealer** | Manual and bulk listing management, upload tracking, rejected-row reports, dashboard |
| **Administrator** | Dealer verification, user management, upload monitoring, reports, audit logs, dictionary curation |

---

## Architecture at a glance

A React web client talks to a gateway that exposes one route prefix per service. Five NestJS
services share a single PostgreSQL instance but are isolated by schema, each with a least-privilege
database role. Asynchronous work is decoupled by a queue; each ETL stage passes only a reference to
its data in object storage.

| Component | Responsibility |
|-----------|----------------|
| `web-frontend` | React + Vite client with role-specific interfaces |
| `api-gateway` | nginx locally, Amazon API Gateway on AWS |
| `auth-user-service` | Accounts, dealer verification state, JWTs and refresh tokens |
| `marketplace-service` | Listings, favourites, recommendations and the hybrid search engine |
| `ingestion-service` | Upload intake and the ETL workflow |
| `admin-service` | Approvals, reports and auditing |
| `notification-service` | Recording and sending e-mail notifications |

**Database** — PostgreSQL 17 with `pgvector` and `pg_trgm`: 18 tables across five service-owned
schemas, with HNSW, GIN trigram and JSONB indexes.

**Bulk-upload pipeline** — the browser uploads the CSV and ZIP directly to S3 through presigned
URLs, the service creates an `UploadJob` and queues it on SQS, and a Step Functions state machine
splits the file into 250-row chunks, processing up to ten in parallel through
parse-normalise → optional language-model repair → validate → enrich → embed → load → process-images
→ aggregate → notify.

**Tech stack** — TypeScript throughout; NestJS + TypeORM; React + Vite; PostgreSQL with pgvector and
pg_trgm; Transformers.js running `all-MiniLM-L6-v2`; Sharp for images; Groq for the optional
language-model step; Docker Compose for local development; Terraform, Lambda, S3, SQS, Step
Functions, RDS and SES on AWS; GitHub Actions for CI/CD.

---

## Contents

### Planning and requirements

| Document | What it covers |
|----------|----------------|
| [Proposal.pdf](Proposal.pdf) | Project overview, objectives, need, scope, deliverables and chosen technologies |
| [Feasibility study.pdf](Feasibility%20study.pdf) | Financial, technical, resource, time, risk and social/legal feasibility |
| [SRS.V1.pdf](SRS.V1.pdf) · [SRS.V2.pdf](SRS.V2.pdf) | Software Requirements Specification — functional and non-functional requirements per service, interfaces, data dictionary, CSV/JSON upload schema |
| [SAD.V1.pdf](SAD.V1.pdf) · [SAD-V2.pdf](SAD-V2.pdf) | Software Architecture Document — ADRs, use-case/logical/process/deployment views, API catalogue, security, CI/CD and testing architecture |
| [Sprints.pdf](Sprints.pdf) | Agile Scrum plan: roles, Definition of Done, eight epics and seven two-week sprint backlogs |

> **Versions.** V2 of the SRS and SAD is current. V1 is kept for traceability; V2 corrects sections
> that contradicted the decision to keep the ETL → Marketplace write boundary synchronous, and
> updates the team size and remaining timeline.

### Verification and delivery

| Document | What it covers |
|----------|----------------|
| [Test Plan.pdf](Test%20Plan.pdf) | Test levels and techniques, coverage matrix, per-service test inventory, entry/exit criteria, defect scales, environment spec and test-data policy |
| [Build and Test Instructions.pdf](Build%20and%20Test%20Instructions.pdf) | How to build the system and run every test suite from a clean checkout |
| [Final Report.pdf](Final%20Report.pdf) | The complete report — literature review, system models, database design, implementation, the hybrid-search algorithm, interfaces, test results and conclusions |

### User manuals

| Document | Audience |
|----------|----------|
| [User Manual (short).pdf](User%20Manuals/User%20Manual%20%28short%29.pdf) | All roles — a condensed guide covering buyer, dealer and administrator functions |
| [Buyer Manual.pdf](User%20Manuals/Buyer%20Manual.pdf) | Buyers and guests — searching, vehicle details, saved vehicles |
| [Dealer Manual.pdf](User%20Manuals/Dealer%20Manual.pdf) | Dealers — verification, dashboard, manual listings, bulk upload |
| [Admin Manual.pdf](User%20Manuals/Admin%20Manual.pdf) | Administrators — user and dealer management, upload monitoring, reports, audit logs, dictionary |

---

## Where to start

- **New to the project?** [Proposal.pdf](Proposal.pdf), then the abstract and introduction of
  [Final Report.pdf](Final%20Report.pdf).
- **Evaluating the work?** [Final Report.pdf](Final%20Report.pdf) — it summarises the models,
  implementation and results in one place.
- **Implementing or extending a service?** [SRS.V2.pdf](SRS.V2.pdf) for the requirements, then
  [SAD-V2.pdf](SAD-V2.pdf) for the architecture and ADRs.
- **Running the system?** [Build and Test Instructions.pdf](Build%20and%20Test%20Instructions.pdf).
- **Using the system?** The [user manuals](User%20Manuals/).

---

## Building and testing

Full steps are in [Build and Test Instructions.pdf](Build%20and%20Test%20Instructions.pdf). In short:

**Prerequisites** — Node.js 22 and npm, Docker Desktop (Postgres container, needed for the
integration and e2e suites), and k6 for the performance tests. Copy each service's `.env.example`
to `.env`; Groq and SMTP/SES keys are optional — without them search uses the rules-based parser and
e-mails are logged.

```bash
# in each service and in web-frontend
npm ci
npm run build

# unit tests (no database needed)
npm run test:ci

# integration, contract and API e2e tests
docker compose up -d postgres
cd database && npm ci && npm run migration:run && npm run grants \
  && npm run seed:dictionaries && npm run seed:vehicles
# then, in the relevant service
npm run test:integration   # or test:contract / test:e2e / test:security

# browser end-to-end tests
cd e2e && npm ci && npx playwright install && npm test

# performance tests
k6 run performance/scripts/listing-detail-baseline.js
```

GitHub Actions workflows in `.github/workflows/` run each service's build and tests on push, plus
Terraform validation and the production deploy.

---

## Results

- **Automated tests** — 193 test files and 2,114 passing unit tests across the six packages, with
  integration, contract and end-to-end suites run separately against a migrated database
  (Jest, Vitest + React Testing Library, Supertest, and Playwright across six journey specifications).
- **Performance** — structured search p95 53–95 ms and listing detail p95 20–60 ms against a 500 ms
  target; natural-language search p95 154 ms against a 2 s target. At the target load of 50 buyers
  with 3 dealers uploading, every endpoint stayed at p95 ≤ 126 ms with 0% errors, and a buyer ramp to
  400 users held 99.99% success.
- **Search relevance** — all 32 labelled buyer queries (misspelt makes and models, typos in fuel and
  body type, price/year/mileage phrases, descriptive wording) produced exactly the expected filters.
  The set now acts as a regression suite.
- **Usability** — average System Usability Scale score of 82.1 across 19 participants, above the
  commonly cited average of 68.
- **Security** — bcrypt (cost factor 12), 15-minute JWTs with 7-day hashed refresh tokens and re-use
  detection, RBAC, least-privilege database roles and an append-only audit trail.

The scope of these measurements, including what was deliberately left untested on AWS to stay within
the free-tier allowance, is stated in section 5 of [Final Report.pdf](Final%20Report.pdf).

---

## Team

**Group 25** — Department of Computer Science and Engineering, University of Moratuwa
**Mentor:** Mr. Bhanuka Siriwardhana

| Registration No. | Name |
|------------------|------|
| 230667F | Virusan T. |
| 230670H | Vishula J. |
| 230674A | R.P.M. Vithanage |

Development ran as seven two-week Scrum sprints from 26 June to 3 October, with the Scrum Master
role rotating between members and every change merged through a reviewed pull request.

---

## Note

These documents are coursework prepared for an academic project. The figures, cost estimates and
performance measurements describe the system as built and evaluated during the project period, and
are not maintained after submission.
