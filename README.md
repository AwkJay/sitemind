# SiteMind — AI Intelligence Platform for Data-Centre EPC Delivery

SiteMind is an automated **senior-structural-reviewer** for data-centre construction projects. It
reads a Design Basis Report or submittal, extracts each engineering parameter **with its exact
source sentence**, checks it against the **real, digitised Indian code clause** that governs it,
and returns non-conformances, a cited project copilot, schedule-risk forecasting, supply-chain
visibility, and commissioning QA — in seconds, with a measured zero-hallucination citation rate.

**Live app:** [AWNI.IN](https://sitemind.awni.in/) · **Docs index:** [`docs/features.md`](docs/features.md) ·
**Architecture:** [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)

## Why it's different

Most AI compliance tools either hallucinate citations or hide their reasoning behind an LLM
"trust me." SiteMind doesn't:
- **Every citation resolves to real text** — digitised Indian Standards (IS 3043, IS 8623, etc.) and CEA regulations, never paraphrased from a model's memory.
- **Two layers — the model perceives and explains; it never decides.** An LLM (Claude Sonnet 4.6,
  via the Claude Agent SDK) reads each submittal and pulls every parameter *with its verbatim source
  span*, behind a deterministic span-verification gate; the pass/fail **decision** is deterministic
  Python evaluated against a cited clause — auditable, reproducible, impossible to fake.
- **Every number is computed, not asserted.** ROI, cost-at-risk, schedule impact, and retrieval
  confidence all come from real formulas over real inputs.
- **21 eval scripts** (18 in `backend/eval/`, 3 more in the standalone Codebook service), each
  reported separately — see [`docs/features.md`](docs/features.md#12-automated-eval-suite-21-scripts)
  for the full per-script breakdown, including the flagship result: **100% rule-decision accuracy
  vs. a 59% naive baseline (n=41)**.

## What's real vs. representative

- **Real:** every IS/CEA clause citation, the compliance decision logic, all 21 evals, document
  ingestion (an uploaded PDF/DOCX is actually parsed, with mandatory abstention on anything not
  confidently extracted), and the CPM schedule-impact recomputation.
- **Representative:** the pre-loaded project documents and schedule are synthetic, modelled on
  real public Indian data-centre tenders. The standards and the logic that checks them are real;
  so is anything you upload yourself.

## Running the servers

**Prerequisites:** Python **3.12** (the pinned numpy/pandas wheels don't build on 3.13/3.14) and
Node 18+. Every key below is **optional** — the app degrades gracefully, and the pass/fail
*decision* is deterministic whether or not any key is set.

### 1 · Minimal — fully offline, no keys
Compliance and Commissioning QA work end-to-end with **no API keys** (deterministic extraction +
deterministic checks against the cited clauses).

```bash
# Terminal 1 — backend  (creates .venv, installs deps, serves :8000)
cd backend && ./run.sh
curl localhost:8000/api/health          # expect {"status":"ok",...} before starting the frontend

# Terminal 2 — frontend
cd frontend && npm install && npm run dev   # -> http://localhost:3000
```
Open `http://localhost:3000` and check the top-bar status pill: **green** = frontend reached the
real backend; **red** = wrong API URL or backend down (you'd be seeing mock data, not a real run).

### 2 · Recommended — semantic search (free Hugging Face token)
**Copilot, Knowledge Base, and Codebook** use MiniLM sentence embeddings served through the
**Hugging Face Inference API**, which needs a free token (the local torch path was dropped to fit the
512 MB free-tier). Without it, those three surfaces return a clear *"semantic retrieval needs
HF_TOKEN"* message instead of degrading silently — Compliance and Commissioning are unaffected.

1. Create a free token (read scope) at **https://huggingface.co/settings/tokens**
2. Add it to `backend/.env`:
   ```env
   HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxx
   ```
3. Start all three services, with the retrieval + Codebook flags on:
   ```bash
   # Terminal 1 — Codebook standards service (:8010)
   cd standards-service && ./run.sh

   # Terminal 2 — backend with Knowledge Base + Codebook mounted (:8000)
   cd backend && CODEBOOK_ENABLED=1 RETRIEVAL_ENABLED=1 ./run.sh

   # Terminal 3 — frontend (:3000)
   cd frontend && npm run dev
   ```
   `RETRIEVAL_ENABLED=1` mounts the Knowledge Base (`/api/retrieval/*`); `CODEBOOK_ENABLED=1` mounts
   the Codebook API (`/api/codebook/*`) and requires the standards service on :8010.

### 3 · Optional — LLM-written prose & answers (Anthropic)
The intelligence layer (Perceive / Explain) uses Claude when a key is set; with no key it falls back
to deterministic templates. Add to `backend/.env`:
```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
ANTHROPIC_MODEL_SMART=claude-sonnet-4-6
```
(OpenAI is also supported via `OPENAI_API_KEY`.) Keys only affect prose and semantic search — **never
a verdict**: the pass/fail decision stays deterministic Python against a cited clause.

Windows instructions and a troubleshooting table live in **[`docs/SETUP.md`](docs/SETUP.md)**.

## Features

- **Compliance Agent** (`/compliance`, HERO) — upload a Design Basis Report or submittal, get NCRs
  with a cited clause, the exact source span, and a confidence-scored Action Brief.
- **Project Copilot** (`/copilot`) — cross-document Q&A with citations, hybrid BM25+dense
  retrieval, "seen-before RFI" detection, abstains below a similarity floor.
- **Schedule Risk** (`/schedule`) — CPM scheduling + leading-indicator risk detection (procurement,
  weather, workforce) with recomputed finish impact and 3-agent mitigation.
- **Supply Chain Visibility** (`/supply-chain`) — multi-tier shipment tracking, delay propagation,
  root-cause attribution, equipment-spec compliance, severity-tiered alerts.
- **Commissioning QA** (`/commissioning`) — cooling test-log CSV → deterministic pass/allowable/fail
  vs. a thermal envelope → NCRs → exportable as-commissioned quality package.
- **Project Timeline** (`/timeline`) — every NCR/RFI/risk/alert/finding from the other five pillars
  as one cross-linked, sorted event list. Pure aggregation, zero new judgment.
- **Knowledge Graph** (`/graph`) — equipment → spec → standard → RFI graph over the same structured
  data the other pillars use.
- **Codebook** (`/codebook`, `standards-service/`) — a standalone, MCP-consumable standards service
  any agent can query, plus a Console (`/codebook/console`) for browsing/uploading corpora.
- **Overview** (`/`) — platform-wide ROI per pillar with its computed basis, and a cost-at-risk
  panel showing every term of the formula, not just a headline number.

Full detail + honest caveats per feature: [`docs/features.md`](docs/features.md).

## Architecture

```
Next.js Command Center  ──REST──>  FastAPI
  (Blueprint design system)          ├─ timeline     (pure aggregation, zero new judgment)
                                      ├─ compliance   (deterministic checks + cited clauses)
                                      ├─ copilot      (hybrid retrieval, cited, abstains)
                                      ├─ schedule     (CPM + leading-indicator rules)
                                      ├─ supply_chain (delay propagation + root cause)
                                      ├─ commissioning (thermal-envelope threshold checks)
                                      ├─ impact / cost_risk (ROI + deterministic cost-at-risk)
                                      └─ standards    (real digitised IS + electrical clauses)
```
**Stack:** Python · FastAPI · scikit-learn · MiniLM embeddings (HF Inference API) · NetworkX ·
Next.js 14 · TypeScript · Tailwind · Recharts. An **intelligence layer** (Claude Sonnet 4.6 via the
Claude Agent SDK) reads, extracts, and explains; a **guarantee layer** (deterministic Python) owns
the verdict and citation. No model training anywhere, and deliberately no agent-orchestration
framework (why: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)).

Full diagram-as-code with every route and data dependency: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Known caveats (worth disclosing, not hiding)

- Some clause `verify_url`s point at a dev-machine host, not a public URL — citations resolve to
  real clause *text* in-app regardless. Don't assume every link is independently clickable yet.
- Semantic search (Copilot, Knowledge Base, Codebook) needs a free `HF_TOKEN`; Compliance and
  Commissioning run with no keys at all. See the run instructions above.
- ROI figures (≈20 engineer-hrs & ≈₹15L per issue) are labeled assumptions, not measurements.
- All project data is synthetic/representative, modelled on public tenders — disclosed, not hidden.

## Roadmap

The compliance engine is clause-driven: every check maps to a real digitised clause, so breadth
scales by adding clauses, not by retraining anything. Planned next: IS 875 Parts 1/2/4/5, IS 13920
(ductile seismic detailing), IS 800 (steel), IS 1893 Part 4, and NBC 2016 (fire/egress/electrical —
the most-cited code in real data-centre permitting).
