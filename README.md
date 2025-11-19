# Booklaxy

Booklaxy is an AI-powered reading companion that converts your book into structured summaries, character cards, and relationship graphs in minutes, without revealing spoilers.

The platform runs on a serverless GCP pipeline that processes PDFs through optimized LLM workflows and delivers a smooth, spoiler-free reading experience on mobile and desktop

**Core Challenge:** Maintain entity coherence across 30-60 independent LLM API calls while processing 1000+ page books; tracking character identities across chapters where each call has lossy context and achieved 95% accuracy at $0.08-0.40 per book (5-10x cost reduction vs naive approach).

## Architecture

```
                                    ┌──────────────┐
                         ┌─────────►│  FireStore   │◄────────────┐
                         │          │    (Jobs)    │             │
                         │          └──────────────┘             │
                         │                                       │
                  ┌──────┴──────┐   ┌──────────────┐   ┌─────────┴─────────┐
                  │   FastAPI   │   │   Pub/Sub    │   │  Workers          │
                  │   Gateway   ┼──►│    Topic     ┼──►│  - summarization  │──► Gemini API (text)
                  │             │   │              │   │  - images         │    Runware API & Vertex AI backup (images)
                  └──────▲──────┘   └──────────────┘   └─────┬───────┬─────┘
                         │                                   │       │
 ┌────────────┐   ┌──────┴──────┐   ┌──────────────┐         │       │
 │  MongoDB   │   │  Next.js    │   |     GCS      │         │       │
 │   Atlas    │◄──┤  Frontend   ┼──►│ (PDFs, Imgs) │◄────────┘       │
 │            │   │  (SSR/RSC)  │   │              │                 │
 └────────────┘   └──────▲──────┘   └──────────────┘                 │
                         │                                           │
                         └───────────────────────────────────────────┘
```

**Core Flow:**
1. Client uploads PDF → Next.js server action → GCS storage
2. Next.js calls FastAPI → publishes job to PubSub (tracked in Firestore) → returns 202 Accepted
3. Worker pulls message → downloads PDF from GCS → processes with LLM
4. Worker sends results via webhook → Next.js updates MongoDB
5. Client polls Next.js and the Graph & UI update in real-time

**Key Design Decisions:**
- **Async worker pattern**: PubSub decouples API from 300-1200s processing jobs. No HTTP timeouts, independent scaling. Workers isolated per job for true parallelism without resource contention.
- **Docker image**: One build deployed as 3 Cloud Run services for the backend. 40% faster deploys, guaranteed dependency consistency. One build for the frontend.
- **Data store separation**: MongoDB (user data, Next.js only), Firestore (job tracking, workers only), GCS (objects, shared).

**Production Engineering:**
- **Security**: API key authentication, least-privilege IAM service accounts, secrets managed via GCP Secret Manager, input validation on PDF uploads
- **Fault tolerance**: Multi-provider image generation with automatic failover (Runware primary, Vertex AI backup)
- **Resilience**: Per-chapter checkpointing enables job recovery without full reprocessing on worker failures
- **Cost governance**: Automated budget monitoring and alerting to prevent cost overruns
- **Performance**: API optimized for sub-200ms responses; workers scale independently with acceptable cold starts for batch workloads

## Core Innovation: Entity Resolution Across LLM Context Boundaries

Production LLM pipeline that maintains entity coherence across 30-60 independent API calls. Processes 1000+ page books into structured narratives while tracking character identities across chapters; analogous to maintaining session state where each call has lossy context.

### The Problem

Character "Captain Morgan" dies in chapter 2. New character "Captain Williams" appears in chapter 7. By chapter 10, sliding context window only retains chapters 7-10. How does the system distinguish two different captains vs. name variants ("The Captain" / "Morgan")?

**Naive approach:** Send all characters to LLM for deduplication → +$1/book in redundant API calls, hallucination-prone entity merging.

### Solution: Hybrid State Management

**Persistent character registry** outside LLM context + deterministic entity tracking:

- **Sliding context window**: Last 3 chapter summaries for plot continuity
- **Full character history**: Registry maintains all character attributes across chapters
- **Deterministic fingerprinting**: Fuzzy matching (Levenshtein + role-attribute comparison) resolves name variants before LLM processing
- **Conditional disambiguation**: LLM called only on detected conflicts, not every character

**Key optimization:** Book text sent once per chapter (not chunked/repeated). Character deduplication runs only when fuzzy matching detects potential conflicts.

**LLM Engineering:**
- JSON schema enforcement eliminates parsing failures and reduces attribute hallucinations
- Few-shot examples adapt per genre (fantasy vs biography require different character attribute extraction)
- Token budget: ~80K tokens/chapter, ~1M tokens/book total
- Malformed responses trigger retry with simplified prompt, then rule-based fallback

### LLM Call Architecture

**Book-level initialization (2 calls):**
- Chapter boundary detection across full document
- Attribute schema extraction (prevents mid-book schema drift)

**Per-chapter processing (2-3 calls × N chapters):**
1. Analysis: Full chapter text → structured summary extraction
2. Deduplication: Conditional character disambiguation (0 to N calls, triggered on naming conflicts)
3. Review: Attribute normalization and cross-reference validation

**Image generation (1 + M calls for M characters):**
- Batch prompt generation: Single call produces all character image prompts
- Runware API: Parallel image generation for each character

### Results

- **Cost**: $0.08-0.40 per book (varies by length: shorter books ~$0.08, longer books ~$0.40)
- **Accuracy**: ~95% entity resolution accuracy across 20+ books
  Definition: accuracy measures the percentage of real narrative characters that are correctly detected, based on manual review and ChatGPT-assisted verification.
- **Efficiency**: ~1M tokens/book, deduplication triggered on 12% of characters
- **Naive baseline**: $1.50-4.00/book with LLM-only deduplication (5-10x cost increase, higher error rates)

## Technology Stack

**Backend:**
- **FastAPI + Cloud Run**: Event-driven microservices with async request handlers, serverless auto-scaling
- **Gemini 2.5 Flash Lite**: Cost-optimized LLM with structured outputs to reduce hallucinations
- **PubSub**: Message queue decouples API from long-running tasks, handles traffic spikes independently
- **Firestore**: Serverless job state tracking with multi-instance safety guarantees

**Frontend:**
- **Next.js 14 App Router**: React Server Components by default, reducing client-side JavaScript where possible.
- **NextAuth.js v5**: Session management, OAuth
- **MongoDB Atlas**: Denormalized reads, connection pooling
- **Zod schemas**: Shared validation across client/server boundaries

**Infrastructure:**
- **Single Docker image**: Multi-stage build with BuildKit cache mounts, deployed as 3 services
- **Cloud Build CI/CD**: Parallel service deployments with health checks
- **IAM service accounts**: Least-privilege access per service

## System Characteristics

**Production Characteristics:**
- **Scalability**: Serverless auto-scaling with PubSub decoupling handles traffic spikes independently from processing capacity
- **Reliability**: Async processing prevents cascade failures; PubSub retry policies with dead letter queues; idempotent job handling prevents duplicate processing
- **Observability**: Full request tracing, structured logging, and Cloud Monitoring integration for production debugging
- **Performance**: API responds in <200ms; full book processing 300-1200s depending on length; frontend optimized with SSR and code splitting


<br>

-----
This README is shared publicly for portfolio purposes only.<br>
Copyright © 2025 Olivier Delfosse.
