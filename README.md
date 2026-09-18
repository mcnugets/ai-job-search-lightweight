# AI Job Search Lightweight

A **deterministic, extensible, token-efficient job-search framework**. It combines the strongest ideas from [MadsLorentzen/ai-job-search](https://github.com/MadsLorentzen/ai-job-search) and [santifer/career-ops](https://github.com/santifer/career-ops), while targeting their main weaknesses: repeated LLM work, expensive multi-agent orchestration, portal lock-in, and limited resumability.

> **Core idea:** evaluate both projects, identify their weak sides, adapt their strongest working ideas, and build a reusable foundation of pure code plus narrowly scoped AI.

This repository is a design and implementation foundation for:

- Fast semantic job matching with vector embeddings
- Deterministic gates, scoring, deduplication, and validation
- Pluggable job portals and data sources
- Persistent checkpoints so interrupted work resumes rather than repeats
- Pydantic models and structured agent outputs
- Optional LLM calls only where language or judgment is genuinely needed
- Human-controlled final review and submission

## Why this project exists

The current projects demonstrate two valuable but different patterns:

### AI Job Search

`ai-job-search` is a Claude Code-native, deeply structured workflow. It provides strong candidate profiling, explicit fit criteria, careful CV/cover-letter tailoring, reviewer separation, PDF inspection, ATS checks, and cautious web-research rules.

Its trade-off is that the workflow is primarily instruction-driven. A large language model performs work that can often be handled more cheaply and consistently by code: filtering, deduplication, arithmetic scoring, keyword coverage, state transitions, and artifact validation.

### Career-Ops

`career-ops` is a broader command center focused on scale. It includes portal scanning, batch workers, structured A-H reports, PDF generation, dashboards, interview support, negotiation support, and multiple AI coding CLI integrations.

Its trade-off is a larger batteries-included runtime and a substantial JavaScript/Playwright/Go surface. More capability also means more dependencies, more orchestration, and more opportunities for repeated model context.

### Our direction

This project takes a middle path:

```text
Pure code for repeatable operations
        +
Vector similarity for semantic retrieval and ranking
        +
Optional LLM calls for language, research, and ambiguous judgment
        +
Persistent typed state for resumable workflows
        +
Human approval before anything is submitted
```

## Design principles

1. **Code before agents.** If a task is deterministic, testable, and repeatable, implement it as code.
2. **Embeddings before expensive reasoning.** Use semantic retrieval to compare profiles and postings at scale; do not use an LLM merely to calculate similarity.
3. **LLMs are specialists, not the whole pipeline.** Reserve them for cover-letter prose, company research, interview coaching, and genuinely ambiguous cases.
4. **State is durable.** Every meaningful stage writes a checkpoint. A retry resumes from the last successful stage.
5. **Everything is extensible.** Portals, parsers, embedding providers, renderers, and LLM providers are adapters rather than hard-coded workflow branches.
6. **Structured boundaries.** Pydantic models validate data exchanged between fetchers, scorers, agents, and workflow stages.
7. **Human-in-the-loop at the right boundary.** The system can prepare and verify an application, but the user reviews and manually submits it.
8. **Respect external systems.** Do not bypass access controls, robots exclusions, rate limits, or terms of service.

## Proposed architecture

```text
                    ┌──────────────────────────┐
                    │ CLI / /run command layer │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │ Resumable workflow engine│
                    │ checkpoints + state DB   │
                    └──────┬─────────┬─────────┘
                           │           │
              ┌────────────▼───┐   ┌──▼────────────────┐
              │ Pure-code path │   │ Optional AI path  │
              │                 │   │                  │
              │ fetch/parse    │   │ cover letters    │
              │ normalize      │   │ company research │
              │ embed/search    │   │ interview coach  │
              │ score/gate      │   │ borderline cases│
              │ render/verify   │   └──────────────────┘
              └────────────┬────┘
                           │
                    ┌──────▼───────┐
                    │ Typed models │
                    │ Pydantic     │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Human review │
                    │ manual send  │
                    └──────────────┘
```

### Main components

```text
src/
├── models/                    # Pydantic domain and workflow models
│   ├── candidate.py
│   ├── job.py
│   ├── evaluation.py
│   └── workflow.py
├── sources/                   # Extensible job data-source adapters
│   ├── base.py
│   ├── greenhouse.py
│   ├── ashby.py
│   ├── lever.py
│   ├── json_feed.py
│   └── html_source.py
├── matching/                 # Embeddings, retrieval, and deterministic scoring
│   ├── embeddings.py
│   ├── similarity.py
│   ├── gates.py
│   └── scorer.py
├── workflows/                # Resumable state machines
│   ├── scan.py
│   ├── rank.py
│   ├── apply.py
│   └── interview.py
├── agents/                   # Optional, narrow AI integrations
│   ├── cover_letter.py
│   ├── company_research.py
│   └── interview_prep.py
├── renderers/                # CV/cover-letter output and PDF validation
├── storage/                  # SQLite, migrations, checkpoints, event log
└── cli.py                    # Commands such as scan, rank, apply, and resume
```

## Semantic similarity and vector embeddings

A job and a candidate profile contain equivalent concepts expressed in different language. For example:

```text
Candidate: "Built production ML pipelines in Python and AWS"
Posting:  "Seeking a machine-learning engineer experienced with cloud deployment"
```

Keyword matching may miss this relationship. An embedding model can encode both texts into vectors and calculate cosine similarity locally or through a configurable embedding provider.

```python
candidate_vector = embed(candidate_profile.matching_text())
job_vector = embed(job_posting.matching_text())
similarity = cosine_similarity(candidate_vector, job_vector)
```

Embeddings are **not a complete hiring judgment**. A single similarity score can miss:

- hard eligibility or language requirements;
- location, salary, clearance, and seniority constraints;
- whether a skill is required or merely mentioned;
- behavioral and motivation fit;
- misleading or duplicated job descriptions.

Therefore the proposed score combines multiple signals:

```text
final_score =
    0.35 * semantic_similarity
  + 0.25 * required_skill_coverage
  + 0.20 * experience_alignment
  + 0.10 * career_alignment
  + 0.10 * preference_alignment
```

Hard gates run before scoring. A failed work-rights, language, or location gate can exclude a posting regardless of its similarity score. Borderline or high-value results may optionally be sent to an LLM for explanation, not because the LLM is needed for arithmetic.

### Embedding strategy

The implementation should support:

- local models such as Sentence Transformers for offline, low-cost matching;
- hosted embedding APIs through an adapter;
- cached candidate and job embeddings keyed by content hash and model version;
- batch encoding for large queues;
- model/version metadata so scores remain reproducible;
- evaluation against a labelled set before changing thresholds.

Similarity scores must be calibrated on real examples. A raw cosine value must not automatically be interpreted as a universal percentage or a hiring probability.

## Persistent state and checkpointing

The system should not repeat every action after a failure or a new command invocation. Each job is represented by a durable workflow record:

```text
received
  → fetched
  → normalized
  → embedded
  → scored
  → shortlisted
  → drafted
  → reviewed
  → verified
  → ready_for_human
  → submitted_by_human
  → outcome_recorded
```

A SQLite store is the initial target because it is local, transactional, portable, and sufficient for a single-user workflow. JSON/TSV exports can remain available for inspection and interoperability.

Example checkpoint record:

```json
{
  "job_key": "acme:ml-engineer:123",
  "workflow": "apply",
  "stage": "scored",
  "status": "complete",
  "input_hash": "...",
  "model_version": "all-MiniLM-L6-v2",
  "output_ref": "evaluations/acme_ml_engineer.json",
  "attempts": 1,
  "updated_at": "2026-09-18T12:00:00Z"
}
```

A retry should:

1. load the latest valid checkpoint;
2. verify inputs and model versions have not changed;
3. skip completed idempotent stages;
4. retry only failed or stale stages;
5. atomically write the next checkpoint;
6. record errors and timing in an event log.

This is the foundation for a lightweight `/run` command:

```text
/run scan       # scan only sources not completed for this query
/run rank       # score new or stale jobs
/run apply KEY  # resume the application workflow for one job
/run resume     # continue all interrupted work
/run status     # show checkpoints and failures
```

The command is a dispatcher over a persistent workflow engine, not a promise that an LLM context remains in memory forever. Durable state is more reliable than relying on conversational context.

## Extensible data-source and portal layer

Job boards differ in APIs, authentication, pagination, rate limits, schemas, and terms of service. They should implement one normalized contract:

```python
class JobSource(Protocol):
    name: str

    def search(self, query: SearchQuery) -> Iterable[RawJob]: ...
    def fetch_detail(self, reference: JobReference) -> RawJob: ...
    def health_check(self) -> SourceHealth: ...
```

Every adapter normalizes into the same `JobPosting` model. New sources can be added in three ways:

1. **Configured feed:** map a JSON, RSS, XML, or public ATS response using YAML.
2. **Small adapter:** implement the source protocol for a portal-specific API.
3. **External plugin:** install a separately versioned package discovered through entry points.

A source must declare its access method, required credentials, rate limits, supported operations, and terms-of-service notes. The registry must not silently enable a new source merely because a package is installed.

Example configuration:

```yaml
sources:
  acme_greenhouse:
    adapter: greenhouse
    company: acme
    board_token: ${ACME_GREENHOUSE_TOKEN}
    enabled: true
    rate_limit_per_minute: 30

  public_feed:
    adapter: json_feed
    url: https://jobs.example.org/api/jobs
    mapping:
      items: data.jobs
      id: id
      title: title
      company: employer
      description: description
      url: apply_url
    enabled: true
```

The source layer is deliberately separate from matching. A portal integration should only retrieve and normalize jobs; it should not contain candidate-specific scoring or application-writing logic.

## Where AI belongs

### Pure code or local models

- source polling and pagination;
- normalization and deduplication;
- eligibility, location, language, and deadline gates;
- embedding generation and nearest-neighbour search;
- weighted score calculation;
- checkpoint transitions and retries;
- PDF compilation, text extraction, and ATS checks;
- tracker merges and integrity validation.

### Optional LLM calls

- writing a natural cover letter from verified facts;
- researching and summarizing a company;
- explaining a borderline match;
- generating interview coaching or negotiation language;
- extracting meaning from unusually unstructured documents.

The LLM output must be validated against Pydantic schemas and the candidate profile. It must never be treated as a source of truth merely because it sounds plausible.

## Human-in-the-loop boundary

This project prepares applications; it does not automatically submit them.

```text
scan → match → draft → render → verify → human reviews → human submits
```

The user remains responsible for logging into a portal, answering application questions, attaching documents, accepting terms, and clicking **Submit**. This avoids unauthorized account automation and gives the candidate a final factual and ethical review.

## Bot blocking and responsible retrieval

The framework should be **bot-aware, not bot-evasive**:

- prefer public ATS APIs, RSS/XML feeds, and employer career pages;
- respect robots.txt, published access rules, rate limits, and terms of service;
- use a normal fetch first;
- if a fetch fails, try an alternative canonical employer source;
- never use rotating proxies, credential theft, CAPTCHA circumvention, or access-control bypasses;
- record a source as unavailable when access is not permitted;
- keep portal-specific warnings visible to the user.

Browser-like headers are only appropriate when the site’s published policy permits the requested path and the failure is a client/WAF mismatch. They are not a mechanism for overriding a denial.

## Comparison

| Dimension | AI Job Search | Career-Ops | AI Job Search Lightweight |
|---|---|---|---|
| Primary strength | Deep, careful application workflow | Broad, scalable command center | Reusable foundation combining both |
| Core runtime | Claude Code instructions | Node.js, Playwright, Go, agent skills | Python core with adapters |
| Matching | LLM rubric and profile reasoning | Agent evaluation and A-H report | Embeddings + deterministic signals; optional AI refinement |
| Batch strategy | Agent-assisted ranking | Parallel headless workers | Local scoring and resumable workers |
| State | Files, CSV, JSON, archives | Pipeline/tracker files and batch state | Typed SQLite checkpoints plus exports |
| Portal model | Per-portal CLI skills | Many provider modules | Source protocol, registry, config, plugins |
| PDF path | LaTeX with strict inspection | HTML + Playwright | Pluggable renderer with verification |
| Human submission | Yes | Yes | Yes |
| Main trade-off | More tokens and repeated model work | More runtime complexity | Requires careful calibration and engineering |

## Suggested command surface

```bash
jobsearch init
jobsearch source list
jobsearch source health
jobsearch scan --query "machine learning engineer"
jobsearch rank --new --top 20
jobsearch apply --job-key acme:ml-engineer:123
jobsearch resume
jobsearch status
jobsearch export --format markdown
```

The same workflows can later be exposed as Claude Code, Codex, or other agent skills. The agent runtime becomes a user interface and optional reasoning layer, not the place where all business logic lives.

## Roadmap

### Foundation

- [ ] Pydantic domain models and schema versioning
- [ ] SQLite store, migrations, event log, and atomic checkpoints
- [ ] Idempotent workflow state machine
- [ ] Source protocol and registry
- [ ] JSON/RSS/XML/public ATS adapters
- [ ] Deterministic gates, deduplication, and weighted scoring

### Matching

- [ ] Local embedding provider with content-hash cache
- [ ] Hosted embedding-provider adapter
- [ ] Hybrid semantic, skill, experience, and preference score
- [ ] Threshold calibration and labelled evaluation fixtures
- [ ] Optional LLM escalation only for borderline matches

### Application output

- [ ] Profile-driven CV templates
- [ ] Pluggable HTML/LaTeX/Typst renderers
- [ ] PDF page-count and text-layer verification
- [ ] Optional cover-letter and research agents with strict schemas
- [ ] Human approval report before submission

### Operations

- [ ] `/run`, `resume`, and `status` command adapters
- [ ] Batch processing with bounded concurrency and retries
- [ ] Source health dashboard
- [ ] Interview, outcomes, and feedback calibration
- [ ] Optional TUI or web dashboard

## Non-goals

- Automatically submitting applications or bypassing portal protections
- Using an LLM for every ranking or state transition
- Claiming that embedding similarity is equivalent to hiring potential
- Hiding source-specific terms-of-service or data-retention constraints
- Building a hosted service that collects personal career data by default

## Status

This repository currently documents the target architecture and design direction. The README is intentionally explicit about proposed components and roadmap items; examples are not claims that every module has already been implemented.

## Credits

The design is informed by:

- [MadsLorentzen/ai-job-search](https://github.com/MadsLorentzen/ai-job-search): profile-driven workflows, careful research, PDF/ATS verification, and human-controlled applications.
- [santifer/career-ops](https://github.com/santifer/career-ops): batch processing, broad portal coverage, structured reports, multi-CLI skills, and resumable workers.

## License

MIT.
