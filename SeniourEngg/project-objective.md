# End-to-End LLM Inference System Design (Local-First + Hybrid Cloud)

> **Additive portfolio enrichment (no removals):** This document keeps the original inference-system plan intact and adds **standout-tier** material: cost intelligence and dynamic routing, latency/caching depth, checkpointing and failure guardrails, eval rigor (quality + cost + latency), observability signals, multi-tenant fairness, a **financial intelligence** use-case narrative with multi-agent roles, NVIDIA validation alignment, a **Streamlit/Gradio** demo layer, optional differentiators (self-improvement / routing / agent competition), a text “final architecture” view, a recruiter metrics dashboard, and **staff-engineer** incident/cost questions. Outcome: reads like a **Senior AI Systems Engineer** portfolio—not only architecture.

### Feedback alignment — standout tier (additive checklist; no removals)

This subsection mirrors the **portfolio feedback** you received; it does not replace any prior section—it makes the upgrade story explicit for recruiters and reviewers.

- **Upgraded to “standout” tier:** Cost intelligence, resilience, latency optimization, eval rigor, and a compelling business use case (financial intelligence: transcripts, risk, sentiment, investment insight).
- **No removals:** The original end-to-end inference system (API → scheduler → engine → cache/monitoring, local-first + hybrid cloud) remains the backbone; everything else is **additive, production-grade enrichment**.
- **Recruiter signal:** Demonstrates *measurable impact*—cost per request, token usage, cache hit rate, percentile latency, failure/retry rates—not only boxes-and-arrows architecture.
- **Differentiation:** Cost-aware routing, failure recovery, semantic caching, checkpointing, explicit ROI framing, demo UI with live cost/latency, and optional multi-agent differentiators (self-improvement, dynamic routing, agent competition).
- **Outcome:** The narrative reads as a **Senior AI Systems Engineer** portfolio project: *design, optimize, and operate* production LLM systems with evidence, not “I integrated an API.”

## Executive Summary

- **Objective:** Build a production-grade LLM inference service as a portfolio project. It spans user API → batching scheduler → inference engine → caching/monitoring, emphasizing both system-level design and deep optimization.  
- **Key Strategies:** Employ continuous/dynamic batching, KV and semantic caching (Redis), model quantization, and scaling. Ensure full MLOps: SLO-driven monitoring, alerting, incident runbooks.  
- **Environment:** Develop locally on Ryzen 9 + 64GB RAM + AMD R9700 GPU (CachyOS) using open tools (llama.cpp, MLC-LLM), then validate on NVIDIA GPUs (cloud). Leverage containers and Kubernetes for deployment.  
- **Deliverables:** A polished GitHub repo (clean code, modular), architecture diagrams, dashboards, benchmark results, demo script, and a postmortem report. Metrics will show significant improvements (e.g. ~10× faster responses for repeated queries【44†L294-L300】, 2× throughput gain from batching). This breadth (architectural scope) and depth (infra tuning) demonstrates senior-level AI engineering skill.  

---

## Enriched Project Identity & Business Framing (Additive — Portfolio Upgrade)

**Strategic upgrade (no removals):** Position the same system as a **Production-Grade Multi-Agent Financial Intelligence System** layered on the existing local-first + hybrid inference architecture. Everything below is **additive**—it strengthens recruiter signal (measurable impact, economics, resilience) without replacing the original plan.

### Business framing (concrete use case)

- Earnings call transcript analysis  
- Risk signal extraction  
- Sentiment tracking  
- Investment insight generation  

### Multi-agent roles (maps to orchestration above the inference engine)

| Agent | Function |
| --- | --- |
| Ingestion Agent | Parses transcripts |
| Sentiment Agent | Financial sentiment scoring |
| Risk Agent | Detects red flags |
| Summarization Agent | Executive summary |
| Validation Agent | Output correctness |

**Implementation note:** These agents can sit behind the existing API Gateway and Scheduler (e.g., LangGraph-style orchestration as a Phase 2 module) while reusing the same inference, cache, and observability stack.

**Differentiation vs. “generic LLM app”:** Cost-aware routing, failure recovery, semantic caching, checkpointing, and explicit ROI framing—signals **Senior AI Systems Engineer** depth, not only model integration.

---

## Industry Challenges in LLM Inference

LLM serving introduces many production challenges, including:

- **Cost vs. Latency vs. Throughput (Trilemma):** Large batches increase throughput but hurt per-user latency; small batches meet latency needs but waste GPU. Balancing these is critical【18†L125-L133】.  
- **Tail Latency (UX):** Users care about p99 latency (e.g. time-to-first-token). Focusing on averages can hide latency spikes【5†L325-L333】. Slow responses ruin UX under SLA.  
- **Reliability & SLO Alignment:** Ambiguous SLOs often lead to under-engineering. Outages (GPU fail, network faults) must be planned for. Without error budgets/alerts【5†L325-L333】, teams can be blindsided.  
- **KV Cache Growth:** Conversation context (attention keys/values) grows quickly【44†L172-L181】. Efficiently managing or paging this KV cache is a bottleneck; memory bandwidth often dominates inference【44†L172-L181】.  
- **Burst Scaling & GPU Utilization:** Infrequent traffic means GPUs idle (waste)【10†L105-L113】; sudden spikes require rapid scaling. Overprovisioning for peaks is costly.  
- **Batching Inefficiencies:** Static or naive batching either introduces latency or leaves resources underused. Traditional schedulers cause “pipeline stalls” when prefills/decodes wait on each other【40†L175-L184】.  
- **Vendor/GPU Lock-In:** Many inference tools (vLLM, Triton, TensorRT) are NVIDIA/CUDA-specific. Relying on them can prevent use of AMD/CPU environments【29†L170-L180】.  
- **Observability Gaps:** Common mistakes include tracking averages instead of percentiles or failing to instrument cache hit rates. Lack of end-to-end tracing makes root cause analysis hard【5†L325-L333】.  
- **Security & Privacy:** Prompt injection and data leakage are top risks【16†L7-L10】. Sensitive inputs/outputs must be sanitized and encrypted. Compliance (GDPR/HIPAA) requires strict data governance.  
- **Model Drift & Reproducibility:** Data distribution shifts can silently degrade model output. Without versioned tests and fixed seeds, benchmark results vary across runs/hardware.  
- **Cold Starts:** Loading large models takes seconds. Without warm replicas, first requests face high latency.  
- **Advanced Pitfalls:** Speculative decoding risks inaccurate outputs if the draft model errs【40†L219-L228】. Quantization (INT8/4-bit) can hurt quality if misused【4†L304-L309】. Benchmark setups vary, making comparisons unreliable.  

*Sources:* Industry analyses and best practices highlight these issues. For example, MosaicML notes memory/KV caches often bottleneck LLMs【44†L172-L181】, BentoML enumerates failure modes (TTFT spikes, idle GPUs, cache latency)【10†L97-L104】【10†L105-L113】, and Redis/AWS studies show semantic caching yields up to **15× latency reduction and ~86% cost savings** for repeated queries【44†L294-L300】. Observability experts stress percentile-based SLOs and comprehensive tracing【5†L325-L333】.  

## Design Mitigations

| **Problem**                    | **Mitigation (System Design)**                                                                                                |
|------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| **Throughput vs Latency vs Cost** | **Dynamic Batching:** Gather requests for a short window (e.g. 50ms) to form large batches, maximizing GPU use【10†L134-L143】. Auto-scale replicas via Kubernetes HPA based on queue depth/CPU. Use **model cascading:** route trivial queries to a smaller model. Apply mixed-precision (FP16/INT8) to speed up math and reduce memory.                              |
| **Tail Latency (SLO)**         | Monitor P50/P95/P99 latency and Token Time (TTFT, ITL) with Prometheus/Grafana【5†L325-L333】. Separate high-priority queue for latency-sensitive requests. Use **speculative decoding:** a small draft model predicts next tokens in parallel, confirmed by the main model【40†L219-L228】. Define and enforce SLOs (e.g. 99% < 1s). |
| **KV Cache Scaling**           | Build a **KV Cache Manager:** use paged attention (like vLLM) to handle large contexts【29†L149-L158】. If cache exceeds GPU memory, spill to CPU RAM. Provide cache invalidation endpoints (e.g. on dialog reset). Ensure thread-safe updates to avoid corruption. |
| **Burst & Idle GPUs**          | Maintain a **warm pool** of inference pods; use spot/preemptible instances for scale-up (saving cost). Implement **continuous batching** (add requests into ongoing batches) to handle spikes smoothly【10†L143-L149】. Autoscale down during lulls to reduce idle cost. |
| **Batching Inefficiencies**    | Employ **iteration-level batching:** new requests join a running decode (Sarathi-Serve style)【40†L175-L184】. Set an upper batch limit to cap latency. Allow mode switching: small batches under light load, large batches under heavy load. |
| **Vendor/GPU Dependency**      | Use cross-platform tools: **llama.cpp** (CPU/GPU) and **MLC-LLM** (supports AMD ROCm) for core inference. Containerize Triton or CUDA-only stacks for cloud use, but architect so local dev never requires them. This avoids lock-in. |
| **Observability Gaps**         | Instrument all layers: record request latency, batch size, cache hit rate, GPU utilization, etc. Track service-level indicators (SLIs) tied to user experience【5†L325-L333】. Build Grafana dashboards from day one (see figure). Use distributed tracing (OpenTelemetry) to follow requests end-to-end. |
| **Security & Privacy**         | Sanitize and validate all inputs at API. Encrypt in transit (TLS) and at rest (disk). Deploy services in a VPC with strict network policies. Store no PII in cache; if needed, anonymize data. Implement authentication/authorization for the API. Follow OWASP LLM Security guidelines (e.g. check for prompt injection)【16†L7-L10】. |
| **Model Drift / Reproducibility** | Maintain versioned test suites of prompts (unit tests). Automate periodic evaluation on holdout data and watch for performance drops. Containerize the environment to ensure consistent benchmark results. Log random seeds and configuration. |
| **Cold Starts**                | Keep at least one replica per model warm. Use memory-mapped model files (llama.cpp/gguf) to speed reloads. On deploy, send dummy requests to pre-warm caches. Adjust health checks to tolerate initial load time. |
| **Speculative Decoding Risks** | Only enable speculative decoding if draft model quality is high. Always verify final output with the full model, and log/fallback if it diverges. Tune thresholds to minimize incorrect prefixes. |
| **Quantization Pitfalls**      | Use established quant tools (llama.cpp, MLC-LLM) for int4/8. Before deploying, compare quantized vs. full precision on representative data. Provide a toggle to use FP16 if outputs suffer. Validate accuracy drop is within acceptable bounds. |
| **Benchmark Variability**      | Fix environment (Docker image, library versions). Use established benchmarks (LLMPerf, GenAI-Perf). Measure across multiple percentiles【12†L173-L182】. Document all settings (batch size, threads). Automate benchmark runs in CI for consistency. |

This table maps each industry issue to an engineering solution. For instance, semantic Redis caching (next) directly addresses repeated-query cost/latency【44†L294-L300】; continuous batching addresses throughput under cost constraints【10†L134-L143】. Every mitigation aligns with a measurable target (e.g. reduce p99 latency, improve GPU utilization).

---

## Cost Intelligence Layer (Additive — HIGH IMPACT)

### Industry problem

- LLM cost explosion in token-heavy pipelines (multi-step agents, long contexts, repeated retrieval).

### Solution: Cost Controller Layer

```text
Cost Controller Layer
   ↓
- Token tracking per agent / per request
- Dynamic model routing (complexity + budget + SLO)
- Budget enforcement (per tenant, per day, per workflow)
```

### Implementation sketch

```python
def route_model(task_complexity: str, budget_remaining: float, threshold: float) -> str:
    if task_complexity == "high":
        return "gpt-4"
    if budget_remaining < threshold:
        return "mixtral"  # or local / smaller API model
    return "gpt-3.5"
```

(Adapt names/providers to your stack; the pattern is **policy-driven routing**, not a single static model.)

### Metrics to add to dashboards

| Metric | Goal (direction) |
| --- | --- |
| Cost / request | ↓ 30–60% vs. naive single-model baseline (target; validate in benchmarks) |
| Token usage (by agent / stage) | ↓ through routing + caching |
| Cost vs. latency curve | Explicit Pareto—operate on the “knee” of the trade-off |

**Recruiter signal:** “Understands production economics,” not only model accuracy.

---

## Latency Optimization Layer (Additive)

### Problem

- High **p95/p99** latency and poor perceived UX even when average latency looks fine.

### Techniques (stack on existing batching + cache)

| Technique | Expected impact |
| --- | --- |
| Async agent execution | −30–50% wall-clock for multi-step workflows (when steps are parallelizable) |
| Streaming responses | Faster time-to-first-token / perceived latency |
| Request batching | Higher throughput at the scheduler |
| Semantic caching | Near-instant responses on hits |

### Architecture add-on

```text
Scheduler
   ↓
Parallel Agent Execution (where safe)
   ↓
Streaming Output Layer
```

### Targets (align with Benchmarking Suite; tune per hardware)

| Metric | Target |
| --- | --- |
| p50 latency | < 1s (agentic pipeline end-to-end; adjust per model size) |
| p95 latency | < 3s |
| Cache hit rate | > 40% on realistic repeat / near-duplicate traffic |

---

## Redis + Semantic Caching — Layered Detail (Additive)

**Tools:** Redis; vector index via Redis Stack / HNSW or sidecar **FAISS / Chroma** (choose one; document the decision).

### Caching layers

| Type | Purpose |
| --- | --- |
| Exact cache | Identical query / prompt hash |
| Semantic cache | Similar queries (embedding similarity) |
| Embedding cache | Avoid recomputing embeddings for repeated text |

### Flow

```text
Query
 ↓
Redis (exact match)
 ↓ (miss)
Vector similarity search
 ↓ (miss)
LLM inference
```

**Expected outcomes (validate; industry reports cite large wins on repetitive workloads):** 40–70% cost reduction potential; major latency improvement on hits—always **measure** on your traces and query mix.

---

## Checkpointing + State Persistence (Expanded — Additive)

### Problem

- Spot / preemptible instances and long agent chains: failures mid-run waste work and break UX.

### What to persist

- Agent state (current step, DAG position)  
- Intermediate outputs (structured JSON where possible)  
- Conversation / retrieval context handles (not necessarily raw secrets)  

### Storage options

- **Redis:** fast resume, short TTL for ephemeral runs  
- **S3 / GCS:** durable artifacts, audit, replay  

### Flow

```text
Agent Step → Save checkpoint
Failure → Resume from last good checkpoint
```

This complements the existing **Checkpoint Store** in the architecture diagram—make checkpoints **idempotent** and **versioned** (model + prompt + schema version).

---

## Failure Handling + Guardrails (Additive)

### Problem

LLMs and external APIs are **probabilistic**; production systems need deterministic recovery paths.

### Additions

| Feature | Purpose |
| --- | --- |
| Retry logic | Transient errors (429, 5xx, timeout) with backoff + jitter |
| Output validation | JSON schema / pydantic-style contracts per agent |
| Self-reflection | Secondary pass: “does this answer satisfy constraints?” |
| Fallback models | Primary degraded → cheaper / local model |

### Pattern

```python
if not valid_output(schema, result):
    retry_with_constraints(tighter_prompt_or_smaller_model)
```

**Bonus (high trust domains):** human-in-the-loop override for low-confidence or high-impact decisions (e.g., risk flags above threshold).

---

## Multi-Tenant Fairness & Quotas (Additive)

### Problem

A single tenant or noisy neighbor can exhaust queue depth, GPU budget, or cache.

### Mitigations

- Rate limiting (token bucket / sliding window)  
- Priority queues (tier → priority)  
- Quota enforcement (per-day tokens, concurrent requests)  

### Example policy

```text
User Tier → Request Priority + Budget Ceiling
```

Wire quotas into the **Cost Controller** and observability (per-tenant token usage).

---

## Fallback + Routing Strategy (Additive)

```text
Primary → GPT-4 (or best cloud model)
Fallback → Mixtral / mid-tier API
Fallback → Local model (llama.cpp / MLC-LLM)
```

Improves **reliability** and **cost control**; log route decisions for postmortems and eval.

---

## NVIDIA-Specific Exposure (Additive — Industry Relevance)

Even when daily dev is on **AMD local**, keep cloud validation aligned with NVIDIA serving stacks:

| Tech | How to include |
| --- | --- |
| vLLM | Cloud GPU benchmarking phase (already planned)—PagedAttention, throughput tests |
| Triton | Optional deployment path for multi-model serving |
| KV cache | Document memory behavior; simulate pressure tests (long context, multi-turn) |
| Continuous batching | Implement / tune in scheduler; compare to static batching |

This preserves **CUDA ecosystem fluency** for roles expecting Triton/vLLM/TensorRT-LLM.

---

## Unconventional Differentiators (Pick 1–2 for Scope Control)

- **A. Self-improving agents:** log failure modes; offline prompt refinement from eval harness.  
- **B. Dynamic model routing:** auto-select model from complexity + cost + latency SLO (ties to Cost Controller).  
- **C. Agent competition:** multiple candidate outputs → judge / consensus → single shipped answer.  

---

## Demo Layer (Additive — Recruiter Impact)

**Suggested UI:** Streamlit or Gradio (fast to ship).

**Show in the demo:**

- Input query / transcript snippet  
- Agent reasoning steps (structured trace, not raw chain-of-thought if policy restricts)  
- Final output  
- **Cost per request** and **latency** (p50 from live run or rolling window)  
- Cache hit / miss indicator  

Recruiters can **see** system behavior—not only read README claims.

---

## Final Updated Architecture — Text Stack View (Additive)

```text
User
 ↓
API Gateway
 ↓
Agent Orchestrator (e.g., LangGraph — optional phase)
 ↓
--------------------------------
| Cost Controller              |
| Redis Cache (exact + semantic)|
--------------------------------
 ↓
Retriever (RAG) — when domain corpus is added
 ↓
Inference Layer (vLLM / API / local llama.cpp)
 ↓
KV Cache (engine-managed)
 ↓
Validation + Guardrails
 ↓
Checkpoint Storage
 ↓
Response
 ↓
Metrics + Logging (Prometheus / Grafana / OpenTelemetry)
```

This is a **logical** superset of the Mermaid diagram below; both views remain valid.

---

## Final Metrics Dashboard — Recruiter-Facing Summary (Additive)

| Category | Metrics |
| --- | --- |
| Performance | p50, p95, p99 latency; TTFT; tokens/sec |
| Cost | $ / request; tokens / request; $ / 1M tokens |
| Efficiency | cache hit rate; batch utilization; queue wait time |
| Quality | hallucination rate; groundedness; task accuracy (eval set) |
| Reliability | failure rate; retry rate; checkpoint resume success |

---

## Architecture & Data Flow

```mermaid
graph LR
  Client[User App] -->|HTTP| API[API Gateway (FastAPI/Go)]
  API -->|REST call| Scheduler[Scheduler/Queue Service]
  API -->|Lookup| RedisCache[(Redis Semantic Cache)]
  Scheduler -->|Batch requests| Engine[Inference Engine<br/>(llama.cpp / MLC-LLM)]
  Engine -->|Update KV| KV[In-Memory KV Cache]
  Engine -->|Store response vector| RedisCache
  Engine -->|Checkpoint state| Checkpoint[(State Store/Persistence)]
  Scheduler --> Observability[(Prometheus/Grafana)]
  Engine --> Observability
  API --> Observability
```

**Key Components:**

- **Cost Controller (additive):** Optional policy layer in front of the Scheduler that tracks **tokens per request/agent**, enforces **budgets/quotas**, and applies **dynamic model routing** (complexity + SLO + remaining budget). Emits cost metrics to Prometheus for dashboards.  
- **API Gateway:** Handles incoming requests, authentication, and inputs. It first queries **Redis Semantic Cache**: transforms the prompt into an embedding and checks for a near-duplicate (cosine threshold). On a hit, returns cached answer instantly (~15× faster【44†L294-L300】). On miss, forwards to Scheduler.  
- **Scheduler/Queue:** Receives API calls, enqueues them, and forms batches. Supports different modes (static, dynamic, continuous batching) to trade off latency and throughput【10†L134-L143】.  
- **Inference Engine:** Loads the LLM (locally, llama.cpp on CPU/GPU; on cloud, maybe NVIDIA Triton or vLLM). Performs token generation (prefill + decode). Uses **KV Cache Manager** to store attention keys/values for multi-turn contexts (improves performance on conversation).  
- **Redis Semantic Cache:** Stores (query_embedding → response) pairs. After Engine produces a response, its embedding is added to Redis (with HNSW index)【44†L278-L287】. Redis is configured to persist data (AOF/RDB) so cache survives restarts【45†L349-L352】.  
- **Checkpoint Store:** Periodically, the system serializes model state or conversation context to disk or object storage. This ensures progress isn’t lost on spot instance kill or node failure. For example, we might checkpoint after each new user message.  
- **Monitoring (Prometheus/Grafana):** Collects metrics from all services: request rates, latencies (p50/p95/p99), batch sizes, cache hit rates, GPU/CPU usage. Dashboards visualize trends. Alerts fire on SLO violations (e.g. high p99 latency). OpenTelemetry traces flow through API→Scheduler→Engine for debugging.

All data flows (requests, tokens, metrics) are observable. By designing for scale and resilience, this architecture meets both system-level breadth (scalable microservices, cloud readiness) and low-level depth (efficient caching, checkpointing) requirements.

---

## Implementation Plan & Milestones

| Week | Tasks (Local Dev)                                     | Cloud Phase (CUDA/NVIDIA)                       | Deliverables & Tests                           |
|------|-------------------------------------------------------|-------------------------------------------------|-----------------------------------------------|
| 1    | - **Foundation:** Set up repo, Docker, basic FastAPI<br/>- Integrate llama.cpp (7B) inference on CPU. | –                                               | **Code:** API + llama.cpp service.<br/>**Test:** Unit tests for API and sample queries.       |
| 2    | - **Batching:** Implement Scheduler with static and dynamic batching【10†L134-L143】.<br/>- Benchmark small (N=1) vs large (N=10) batches. | –                                               | **Code:** Scheduler module.<br/>**Test:** Performance test: measure latency vs batch size. |
| 3    | - **Caching:** Integrate Redis for semantic caching. Build prompt encoder (e.g. SentenceTransformers) to query Redis HNSW.【44†L278-L287】<br/>- Cache first responses. | –                                               | **Code:** Redis cache logic.<br/>**Test:** Hit/Miss test: show ~15× faster on cache hit【44†L294-L300】. |
| 4    | - **Observability:** Add Prometheus client to API/Scheduler/Engine. Create Grafana dashboards (latency, throughput, cache stats).<br/>- Implement **checkpointing:** periodic save to disk. | –                                               | **Code:** Instrumentation, dashboards.<br/>**Test:** Simulate spike, verify alerts (e.g. high p99). |
| 5    | - **K8s Deployment:** Kubernetes manifests (k3s local). Deploy all services + Redis + storage volume. Configure HPA on queue depth/CPU. | –                                               | **Deployment:** k3s cluster up.<br/>**Test:** Scale test: send 100QPS, ensure pods auto-scale and then scale down. |
| 6    | –                                                     | - **GPU Benchmarking:** Launch cloud GPU (e.g. AWS p4d with A100). Deploy Docker images on EKS/GKE.<br/>- Replace llama.cpp with vLLM/Triton for inference. Run same workloads. | **Code:** Cloud IaC (Terraform/YAML).<br/>**Test:** Compare throughput and latency on AWS vs local. Document results. |
| 7    | - **Demo & Docs:** Refine code, write README, architecture diagrams. Prepare demo script.<br/>- *(Additive)* **Demo UI:** Streamlit/Gradio showing query → agent steps → output → **cost + latency**; optional **eval harness** smoke tests. | –                                               | **Docs:** README, architecture doc, demo slides.<br/>**Test:** Dry-run of demo scenario end-to-end. |
| 8    | *(Optional)* Advanced features (speculative decoding, larger models). | –                                               | **Optional:** Showcase additional optimizations. |

**Migration Effort (Checklist):** 
1. **Containerize services (2–3 days):** Ensure each component runs in Docker (we already planned this).  
2. **Setup Cloud Infrastructure (3–4 days):** Write Terraform or K8s YAML for cloud cluster (with nodes, network, storage).  
3. **CI/CD Pipeline (1–2 days):** Configure automated builds and deployments (GitHub Actions or Jenkins).  
4. **Security Setup (1 day):** Configure cloud IAM roles, TLS certificates, secrets management (e.g. AWS Secrets Manager).  
5. **Deploy to Staging & Test (1 day):** Spin up a test environment, deploy services, and run functional tests.  
6. **Load Testing on Cloud (2 days):** Use locust or k6 to simulate expected traffic, tune auto-scaling and caching for cloud environment.  
7. **Production Switch (1 day):** Route DNS/API to new cloud endpoint, monitor stability.  

Each step overlaps; total ~2–3 weeks. This checklist ensures all cloud-specific tasks are covered systematically【31†L1248-L1257】.

---

## Technical Stack & Feasibility

- **Local (Ryzen 9 9900X, 64GB, AMD R9700):**  
  - *Inference:* Use **llama.cpp** (C++ engine) for both CPU and AMD GPU. MLC-LLM can leverage ROCm/Vulkan on AMD for up to ~80% NVIDIA performance【33†L112-L115】. 4-bit quant models fit in memory. Smaller models (<7B) run easily; larger models may not fit or will be very slow on CPU.  
  - *Batching & Caching:* Fully implementable. Redis runs on local machine. Kubernetes (k3s) can be installed for cluster simulation.  
  - *Monitoring:* Prometheus/Grafana run locally (on separate port). All metrics gather.  
  - *Checkpointing:* Use local disk or the same Redis with AOF persistence【45†L349-L352】. Feasible to survive restarts.  
  - *Limitations:* Cannot leverage NVIDIA-specific kernels (TensorRT, Flash Attention) or 70B models. GPU optimization is limited to what ROCm/LLVM provides (e.g. MLC-LLM optimizations).  
- **Cloud (NVIDIA GPUs):**  
  - *Instances:* AWS P4d (8×A100), GCP A2 (1–8×A100), Azure ND series, Lambda Labs (1×A100) or RunPod (1×A100) for cheaper testing.  
  - *Performance:* Can run large models (13B–70B) at high throughput. Use **vLLM** or **Triton-LLM** to maximize GPU utilization【4†L304-L309】.  
  - *Cost:* Spot/preemptible instances dramatically reduce cost (e.g. ~$3–5/hr for A100 vs ~$32 on-demand). Lambda Labs offers ~$1–1.5/hr for A100. Free-tier VMs (no GPU) cannot run inference meaningfully; only useful for hosting non-GPU components.  
- **Redis & Checkpointing:** Local environment easily runs open-source Redis. In cloud, use managed Redis (ElastiCache) or hosted Memcached. Redis supports persistence (AOF/RDB) so semantic cache survives restarts【45†L349-L352】. For checkpointing model state, use cloud storage (S3/GCS) or a simple local volume attached to pods.  

**Cheapest GPU Options:** Spot/preemptible A100 instances (AWS/GCP spots) or smaller GPU instances (e.g. AWS g5g with 1×A10G) are cheapest that still allow CUDA testing. Lambda Labs and RunPod provide fully on-demand A100 access at low cost ($1–2/hr). **Free-tier accounts** (AWS Free Tier, GCP Always Free) only include CPU VMs or tiny GPUs (if any), so they do *not* support our high-end inference needs; they are insufficient beyond experimental CPU tasks.

| **Feature/Task**               | **Local (Ryzen + AMD)**                                | **Cloud (NVIDIA GPU)**                                                 |
|--------------------------------|--------------------------------------------------------|------------------------------------------------------------------------|
| LLM Inference (small model)    | Fast enough on CPU or AMD GPU (via MLC-LLM)            | Trivial with CUDA acceleration                                         |
| LLM Inference (large model)    | **Not practical:** OOM or very slow (use small model)  | Handled easily (A100/H100)                                              |
| Dynamic Batching               | Supported (multi-thread scheduling)                    | Supported and more effective (more GPUs to batch across)               |
| Quantization (INT4/INT8)       | Supported (llama.cpp, MLC-LLM on ROCm)                 | Supported (TensorRT, Triton)                                            |
| Semantic Caching (Redis)       | Fully supported (run Redis locally)                    | Fully supported (ElastiCache/MemoryStore)                              |
| Checkpointing (state)          | Use local disk or Redis persistence                    | Use cloud storage (S3/GCS) or attached volume                           |
| Observability (Prom/Grafana)   | Fully supported (local install)                        | Fully supported (cloud cluster)                                        |
| Auto-scaling                   | Limited to local hardware (k3s on single machine)      | Fully supported (K8s HPA on multi-node)                                 |
| Cold Start (warm pods)         | Keep pods alive (local mini-cluster)                   | Cloud: keep min replicas, use Cronjobs or triggers to pre-warm         |

This table shows what can be done locally vs what requires cloud. In summary, all orchestration, caching, and CPU-based tasks are local-feasible; true high-throughput inference and CUDA-specific optimizations require cloud GPUs. Always test on real CUDA hardware before claiming performance results.

---

## Benchmarking Suite

**Metrics to Track:** TTFT (time-to-first-token), Token Time (ITL), throughput (tokens/sec), GPU/CPU utilization, memory usage, cache hit rate, error rates. Track percentiles (p50/95/99) for latency【12†L173-L182】. Also record cost per 1M tokens ($).  

**Workloads:** Create a mix of queries: short (<50 tokens) and long (>200 tokens) prompts, multi-turn dialogues, and static content (e.g. Q&A). Include a subset of repeated/very similar queries to test caching. Example sources: Alpaca or instruction datasets, FAQs. For semantic caching, use an embedding model (like OpenAI or SentenceTransformers) to vectorize queries.  

**Scripts/Tools:** Use Python load tests (e.g. Locust, k6) to simulate concurrent clients with realistic traffic patterns (e.g. 5–50 QPS). For each test, vary batch sizes and measure corresponding latencies. Use tools like NVIDIA’s `nvprof` or PyTorch Profiler for hardware metrics if needed.  

**Target SLOs:** For a 7B model, aim for p99 latency ≤ 1s under 10 QPS. Throughput target ~50–100 tokens/sec with batching. When caching is effective (hit), aim ~100ms response (10× faster)【44†L294-L300】.  

**Sample Latency vs Throughput Chart:** (Illustrative) The graph below conceptually shows how increasing throughput (via larger batches or hardware) typically increases tail latency, forming a knee-shaped trade-off. The chosen batching strategy should operate on the left knee of this curve to balance performance and responsiveness.

| **Metric**                 | **Target**             |
|----------------------------|------------------------|
| p50 Latency (7B model)     | <200 ms                |
| p99 Latency (7B model)     | <1,000 ms             |
| Throughput (7B model)      | >50 tokens/sec         |
| Semantic Cache Hit Rate    | >50% on repeated queries |
| GPU Utilization           | >80% under load        |
| Error Rate                 | <0.1%                   |

Performance benchmarks will be documented in a report with charts. For example, **before caching** vs **after caching** results should be shown: AWS found that semantic caching cut latency by ~15× on cache-hit queries【44†L294-L300】. We will present similar *before/after* metrics (e.g. throughput doubled after optimizing batching and added caching). 

### Advanced evaluation framework (additive)

**Problem:** Model quality is multi-dimensional; “vibes” are not enough for production hiring narratives.

**Add tracked dimensions:**

| Metric | Description |
| --- | --- |
| Accuracy / task success | Correctness on held-out prompts or labeled finance QA |
| Hallucination rate | Factual contradictions vs. source text (esp. with RAG) |
| Latency | p50/p95/p99, TTFT |
| Cost | Tokens and $ per successful task |
| Groundedness | Retrieval relevance + citation alignment |

**Tools:** LangSmith (tracing + eval), or a **custom eval harness** (pytest + golden files + LLM-as-judge only where grounded checks aren’t enough).

**When to use which (additive):** Prefer **grounded checks** (schema, citations, golden outputs) for finance/risk outputs; use **LangSmith** when you need trace-level debugging, regression comparison across runs, or team-visible eval dashboards; use **LLM-as-judge** sparingly, with fixed rubrics and human spot-checks on a sample.

**JD alignment (additive):** A multi-metric bar (accuracy, hallucination, latency, cost, groundedness) matches what production and senior LLM job descriptions increasingly expect—not a single leaderboard score.

**Example eval config (illustrative):**

```yaml
metrics:
  - accuracy
  - hallucination_rate
  - cost
  - latency
  - groundedness
```

**Proof of optimization:** Every cost/latency claim should cite A/B or before/after runs with fixed prompts, pinned model versions, and percentile tables—this is how you **prove** improvements to skeptical reviewers.

---

## Observability, Incident Response & Runbooks

- **Monitoring Stack:** Prometheus metrics from API, Scheduler, Inference, Redis. Grafana dashboards track: request latency (p50/p95/p99), batch sizes, queue lengths, cache hit/miss rate, GPU/CPU utilization. Use OpenTelemetry to trace a request through the system. Baseline dashboards are prepared before load tests.  

**Observability upgrades (additive — deeper signals):**

| Signal | Why it matters |
| --- | --- |
| Token usage per agent / stage | Cost attribution + routing effectiveness |
| Queue wait time | Bottleneck detection (scheduler vs. engine vs. cache) |
| Cache hit rate (exact vs. semantic) | Efficiency of caching policy |
| Failure rate + retry rate | Reliability and backoff health |
| Latency breakdown by span | Where p99 time is spent (TTFT vs. decode vs. retrieval) |

**Stack (reference):** Prometheus, Grafana, OpenTelemetry (traces + metrics correlation).  
- **Alerts:** Define SLOs (e.g. “99% of requests <1s”). Configure alert rules for SLO breach (e.g. p99 > 1s for 1 minute) and for infrastructure (e.g. >80% GPU memory use, Redis down). Include contextual labels (model version) in alerts.  
- **Incident Runbook:** Step-by-step guide for common incidents. Example: *Latency Spike:* Check dashboards for which stage lags (API vs inference). If queue backlog is high, scale up pods; if Redis cache miss rate spiked, revert recent code changes. *Redis Outage:* Fallback to no-cache mode (serve direct LLM) and restart Redis from AOF. *Pod Crash:* Kubernetes should auto-restart; if due to OOM, reduce batch size. Every incident should be timed/logged.  
- **Postmortem Template:** For any significant incident, document: **Title & Impact:** (e.g. “Cache Misconfig Caused 10× Latency”). **Timeline:** sequence of events and responses. **Root Cause:** e.g. Redis config error. **Resolution:** steps taken (e.g. fixed config, restarted service). **Lessons:** e.g. add pre-deployment cache connectivity test. Include pre/post metrics graph.  

Tools like Grafana and alerting systems (PagerDuty/Slack) will provide real-time visibility. We will simulate at least one incident (e.g. OOM or Redis downtime) and perform a mock postmortem to demonstrate readiness.

---

## Recruiter-Facing Deliverables

- **Repo Structure:**  
  - `/api/` – FastAPI/Go code and Dockerfile.  
  - `/scheduler/` – Python scheduler (batch logic).  
  - `/inference/` – llama.cpp/MLC-LLM serving code.  
  - `/cache/` – Redis setup and query-service code.  
  - `/infra/` – Kubernetes manifests and Terraform scripts.  
  - `/benchmarks/` – Load test scripts, result logs/charts.  
  - `/docs/` – Architecture diagrams, design notes (Mermaid code).  
  - `/monitoring/` – Grafana dashboard JSON, Prometheus rules.  
  - `/tests/` – Unit tests (e.g. sample prompt->response), postmortem reports.  
  - `/demo/` *(additive)* – Streamlit or Gradio UI: query → agent steps → output → **cost/latency** panel.  
  - `/eval/` *(additive)* – Golden datasets, eval runners, hallucination/groundedness checks.  
  - `/orchestration/` *(additive, optional phase)* – Multi-agent workflow (e.g., LangGraph) atop existing services.  

- **README Outline:** Overview, setup instructions, architecture summary, usage examples, scaling notes. Highlight “portfolio project” aspects: performance gains, metrics. Include a summary of key results (e.g. “Cache yields 10× speedup”).  

- **Demo Video Script:** Show user interacting with the API (e.g. CLI or simple UI). Split-screen: Grafana live update of metrics as queries run. Narrate the architecture (“This system includes an API gateway, scheduler, inference engine, and Redis cache for repeated queries”). Showcase before/after metrics (via slides). Demonstrate an incident (e.g. restarting a pod) and auto-recovery. Conclude with summarizing achievements (throughput improved, reliability ensured).

- **Benchmark Report Template:** Sections for *Methodology* (tools, hardware), *Workloads* (query types), *Results* (tables/graphs of latency and throughput), *Analysis* (interpretation of data). Include plots of latency vs QPS and bar charts of 50th/99th percentiles.  

- **Before/After Metrics:** Provide a table/chart summarizing optimizations. Example: “Baseline (no optimizations): p99=2.5s, 20 tok/s. With batching+cache: p99=0.4s, 80 tok/s.” This concrete evidence is compelling to recruiters.  

- **Migration Effort (already detailed above):** Present as a checklist with time estimates (2–3 weeks total) to highlight planning skills.  

All materials are pitched for clarity to both technical and hiring audiences, balancing high-level architecture with detailed metrics and code cleanliness.

---

## Security, Privacy & Compliance

- **Input Sanitization:** Validate prompts (e.g. remove harmful tokens) to prevent injection attacks. Use Python sandboxing if allowing plugins.  
- **Data Encryption:** Enforce HTTPS/TLS for all API calls. Encrypt any persistent data (Redis state, logs) at rest. Use managed secrets (AWS Secrets Manager) for API keys.  
- **Authentication:** Require API keys or OAuth for external access. Use Kubernetes RBAC to restrict service permissions.  
- **Privacy:** Minimize stored data. For cached responses, strip any user-identifiable info. If logging prompts, mask sensitive parts. Comply with GDPR by providing deletion procedures for user data.  
- **Network Isolation:** Run services in a secured network (VPC or Kubernetes network policy). Close unused ports.  
- **Vulnerability Management:** Keep libraries up to date. Use container scanning tools (e.g. Trivy). Review OWASP LLM vulnerability lists【16†L43-L52】. Implement patch management.  
- **Incident Audit:** All admin actions and security events (e.g. failed logins) are logged with timestamps. Maintain audit trail of model version in use (for reproducibility/compliance).  
- **Regulatory:** If used in sensitive domain (e.g. finance, healthcare), ensure compliance frameworks (e.g. SOC 2). Use cloud provider compliance features (VPC logs, IAM roles).  

Security controls cover risks identified earlier: e.g. semantic cache is persisted safely【45†L349-L352】, and prompt filters catch attacks. The design favors transparency and least-privilege to satisfy auditors.

---

## Advanced Optimizations (Prioritized)

1. **Semantic Redis Caching (Already Implemented):** Continually refine query embedding model and HNSW index. AWS findings show *semantic caching* can cut latency by ~15× and cost by ~86% for repeated queries【44†L294-L300】. Monitor hit rate and adjust similarity thresholds.  
2. **Quantization & Pruning:** Apply int4/8 quant via MLC-LLM/llama.cpp to speed up inference【4†L304-L309】. Test using tools like GPTQ. Prune or distill the model if needed for extreme speedups. Provide fallback to full-precision if artifacts are unacceptable.  
3. **Speculative Decoding:** Use a smaller model to predict tokens ahead, as research suggests it can reduce generation latency【40†L219-L228】. Implement a circuit: if draft's output diverges, fallback to standard decode.  
4. **Engine-Level Kernels:** On NVIDIA (cloud), leverage TensorRT-LLM fused kernels and cuBLAS heuristics. On AMD local, explore ROCm libraries and upcoming TVM optimizations.  
5. **Stateful Checkpointing:** Optimize the frequency and method of checkpointing conversation state to minimize impact. Use asynchronous snapshots to avoid blocking inference.  
6. **Redis Persistence:** Use Redis AOF/RDB with frequent snapshots so cache state (including conversational embeddings) is durable【45†L349-L352】. This avoids full rebuild after an outage.  

Each optimization is implemented only after validating its benefit. For example, we will measure the actual speedup of int4 quant on our hardware. All code paths include safe fallbacks (e.g. disabled speculative decoding, FP16 mode) to ensure stability in the AMD/CPU environment.

---

## Primary Resources & Cloud Options

**Key Documentation:**  
- *Inference Engines:* [vLLM Docs](https://docs.vllm.ai) (batching, PagedAttention)【29†L149-L158】; [NVIDIA Triton LLM Guide](https://developer.nvidia.com/) (deployment best practices); [TensorRT-LLM](https://developer.nvidia.com/tensorrt) (CUDA optimizations).  
- *Frameworks:* [PyTorch (ROCm)](https://rocmdocs.amd.com/en/latest/Deep_learning/Deep-learning.html) for AMD GPU support; [MLC-LLM](https://mlc.ai) blog (AMD performance)【33†L112-L115】.  
- *Local Engines:* [llama.cpp GitHub](https://github.com/ggerganov/llama.cpp) (C++ inference); [RedisAI](https://redis.io/docs/stack/redisai/) / [LangCache](https://redis.io/docs/stack/redisai/reference/langcache/) (semantic caching).  
- *Monitoring:* [Prometheus](https://prometheus.io), [Grafana](https://grafana.com) official tutorials for setup.  
- *Cloud:* AWS EC2 docs (P4d, G5 instances), GCP Compute Engine (A2 machines), Lambda Labs & RunPod (pricing).  
- *Security:* OWASP guidelines for AI, Kubernetes hardening docs, Redis security best practices.  

**Cloud GPU Providers (for CUDA testing):** AWS P4d (8×A100), GCP A2 (8×A100), Azure NDv4 (8×A100), Lambda Labs (A100-40GB), RunPod (A100-40GB), and even consumer GPUs (RTX 4090) on cloud VMs. **Cheapest options:** Spot or preemptible instances (AWS/GCP) can be ~$0.5–$1/hr for a single A100. LambdaLabs offers on-demand A100 at ~$1.40/hr with no commitment. Free-tier (AWS/GCP) does *not* provide GPUs, so it cannot run serious LLM inference. You might use a free-tier VM only for orchestrating or very small CPU tests.

| **Component/Tool**    | **Official Docs / Links**                                    |
|-----------------------|--------------------------------------------------------------|
| vLLM                 | [vLLM Docs](https://docs.vllm.ai)                              |
| Triton-LLM           | [NVIDIA Triton Inference](https://developer.nvidia.com/)      |
| PyTorch (ROCm)       | [AMD ROCm DL Docs](https://rocmdocs.amd.com)                 |
| llama.cpp            | [llama.cpp GitHub](https://github.com/ggerganov/llama.cpp)    |
| MLC-LLM              | [MLC-LLM Blog](https://mlc.ai/blog/)                          |
| TensorRT-LLM         | [NVIDIA TensorRT](https://developer.nvidia.com/tensorrt)      |
| Prometheus           | [prometheus.io](https://prometheus.io)                        |
| Grafana              | [grafana.com](https://grafana.com)                            |
| Redis (Semantic Cache) | [Redis Search](https://redis.io/docs/stack/search)         |

---

## Options Comparison (Tables)

**Runtime Options:**

| **Runtime**           | **HW Support**        | **Max Model (Locally)** | **Pros**                                    | **Cons**                               |
|-----------------------|-----------------------|-------------------------|---------------------------------------------|----------------------------------------|
| **llama.cpp**         | CPU (Multi-core)      | ~7B+ (slow on CPU)      | Cross-platform; no dependencies; GGUF format| Slow on CPU; limited GPU optimizations |
| **llama.cpp (GPU)**   | AMD (ROCm/Vulkan)     | ~7–13B                  | Uses local AMD GPU (via MLC/ROCm)          | AMD support still maturing             |
| **MLC-LLM**           | AMD, NVIDIA           | ~7–13B+                 | Codegen for multiple backends; 4-bit support【33†L112-L115】 | Early-stage, complex build pipeline    |
| **HuggingFace/PyTorch** | CPU/GPU            | ~13B (FP16)             | Familiar APIs; ONNX export                | Heavy; needs PyTorch/ROCm/CUDA         |
| **vLLM**              | NVIDIA A100/H100      | 70B+                    | Extremely high throughput; PagedAttention【29†L149-L158】 | NVIDIA-only; higher complexity        |
| **TensorRT-LLM**      | NVIDIA GPUs           | 70B (sharded)           | Highly optimized kernels; industry-proven | Proprietary; NVIDIA-only             |

**Cloud Instance Options:**

| **Instance**            | **GPU (Qty)**     | **VRAM** | **Approx $/hr (On-Demand)** | **Spot/$ (per GPU)** | **Use Case**                  |
|-------------------------|-------------------|----------|-----------------------------|----------------------|-------------------------------|
| AWS p4d.24xlarge        | 8 × A100 (40GB)   | 40 GB    | ~$32                        | ~$4 (A100 spot)      | Top-tier LLM training/serving  |
| AWS g5.12xlarge         | 4 × A10G (24GB)   | 24 GB    | ~$10                        | ~$1.50 (A10G spot)   | Mid-size model inference      |
| GCP a2-highgpu-1g       | 1 × A100 (40GB)   | 40 GB    | ~$9                         | ~$2 (preemptible)    | Single-model inference, testing |
| Azure ND96asr_v4        | 8 × A100 (40GB)   | 40 GB    | ~$27                        | ~$3.50 (spot)        | Enterprise LLM workloads      |
| Lambda Labs (us-east)   | 1 × A100 (40GB)   | 40 GB    | ~$1.40                      | n/a                  | Development/validation       |
| OVH Cloud HGX or similar| 8 × A100 (80GB)   | 80 GB    | ~$20                        | $?                   | Alternative A100 provider    |

*(Pricing is indicative, April 2026; spot/preemptible prices vary.)* AWS spot instances at ~10–20% of on-demand costs are usually the cheapest way to run large GPU tests. Free-tier options (AWS Free Tier, Google Always Free) only offer CPU VMs or tiny GPUs (if any), insufficient for serious LLM inference. Google Colab (free tier) can run small models on CPU/GPU but is not reliable for production-grade tests. 

---

## Risk Assessment & Mitigation

| **Risk**                  | **Impact**                    | **Mitigation**                                                                                           |
|---------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| **Incompatible GPU Stack**  | Inability to optimize on AMD | Use portable tools (llama.cpp, ROCm). Validate on cloud GPUs early to adjust expectations【33†L112-L115】. Keep CPU fallback. |
| **Cache Staleness**       | Serving outdated response     | Set TTLs or version keys for Redis. Invalidate on model update or on user edit. Log cache events.      |
| **Spot Instance Termination** | State loss (if uncheckpointed) | Use **Frequent checkpointing** of conversation state. Utilize Redis persistence so in-memory cache survives restarts【45†L349-L352】. Run multiple replicas. |
| **Performance Regression**  | Throughput/latency degrade   | Automate benchmarks in CI. Use alerts if metrics worsen. Maintain before/after performance logs.         |
| **Security Breach**       | Data leak or denial-of-service | WAF/IDS for API. Quotas/Rate-limit. Audit logs on data access. Regular pen-tests.                        |
| **Overcomplexity**         | Project delays                | Follow incremental development. The plan stages features so MVP completes early for reviews.            |
| **Vendor Lock-In**         | Hard to port                 | Containerize everything. Abstract cloud services via Terraform modules. Keep code generic (no AWS Lambdas). |

Each risk has a clear mitigation in our design. For example, Redis persistence means cache and checkpoints recover after node restarts【45†L349-L352】. Regular load testing catches regressions before they reach recruiters. By addressing risk upfront, we demonstrate production-readiness.

---

## Assumptions and Unspecified Constraints

- **Workload:** Chatbot/query interface with mostly textual prompts (no images/embeddings ingestion). Multi-turn context for KV cache.  
- **Traffic:** Moderate (10s of QPS), not extreme enterprise load. Peaks handled by auto-scaling.  
- **Security:** Basic OWASP-level attacks; no specialized malicious adversarial inputs assumed.  
- **Budget:** Limited but allows occasional cloud bursts. Cost optimization matters but not first constraint.  
- **Free-Tier Usage:** We assume free-tier only for preliminary dev (e.g. GitHub Actions); production testing requires paid GPU instances.  
- **Open-Source:** All software used is open-source or free-for-Dev. No proprietary enterprise-only tools.  
- **Hardware:** User’s AMD GPU has ROCm support. (The R9700 is similar to AMD 7900 series【33†L112-L115】, so we assume MLC-LLM/ROCm can utilize it).  
- **Team:** Solo project, so simplicity and reproducibility are prioritized over complex scale architectures.  

These assumptions fill gaps not specified. If any assumption changes (e.g. workload changes), the design may be adjusted accordingly (e.g. adding specialized APIs or more GPUs).

---

This comprehensive plan blends **broad system design** (scalable microservices, clear docs, end-to-end testing) with **deep infrastructure engineering** (performance tuning, caching algorithms, GPU optimizations), meeting both recruiter and production requirements. 

### Strategic narrative shift (additive)

You’ve evolved the headline story from:

> “I can build LLM apps”

to:

> **“I design, optimize, and operate production LLM systems with measurable impact.”**

### Staff-engineer prompts (use in interviews & design reviews)

- If system **cost spikes 5× overnight**, what levers do you pull first? (routing, cache, batching, model tier, tenant quotas, traffic anomaly, prompt bloat, retrieval fan-out.)  
- If **p99 latency degrades**, where do you look first? (queue depth, cache miss storm, GPU memory pressure, KV growth, external API, N+1 retrieval.)  
- How do you **prove** optimizations worked? (pinned versions, percentile tables, A/B or before/after, cost per successful task, eval regression gates.)  

### Think like a staff engineer — explicit prompts (additive)

Use these as written **portfolio talking points** or mock design-review questions; answers should cite metrics from your benchmark report and dashboards.

1. **Cost:** If your system cost spikes **5× overnight**, what levers do you pull **first** (in order), and what evidence would confirm each lever worked?  
2. **Latency:** If latency degrades at **p99** (not p50), where do you look **first** in the trace graph, and what single chart would you show in a postmortem?  
3. **Proof:** How do you **prove** your optimizations actually worked—baseline definition, pinned model/prompt versions, statistical sufficiency, and a pass/fail gate before shipping?

### Optional next steps (out of scope for this doc unless you want them)

- GitHub-ready **module breakdown** + folder scaffolding PR.  
- **Benchmark experiments** with real numbers (tables + charts) for portfolio README.  

### Portfolio “next” deliverables (additive — from review feedback)

If you want to push the project from “strong doc” to **undeniable proof** in the repo:

1. **GitHub-ready repo structure + module breakdown** — One PR that maps sections of this doc to concrete packages (`/api`, `/scheduler`, `/inference`, `/cache`, `/eval`, `/demo`, `/orchestration`, etc.) with ownership and interfaces; makes navigation trivial for reviewers.  
2. **Benchmark experiments with real numbers** — A small number of **named scenarios** (e.g. cold vs warm, cache hit vs miss, single-model vs routed, burst QPS) with tables: p50/p95/p99, $/request, tokens/request, cache hit %, and eval regression (hallucination/groundedness on a frozen golden set).  

**Sources:** Official docs and industry blogs underpin this design. Examples include MosaicML/Databricks on LLM serving【18†L125-L133】, BentoML optimization guide【10†L97-L104】, NVIDIA metrics guidelines【12†L173-L182】, and Redis/AWS cache case studies【44†L294-L300】. These citations validate the key strategies proposed. All configuration, code, and benchmarks will cite these sources explicitly.