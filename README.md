# Streaming Backends for Enterprise AI Assistants: What Is True, What Is Hype, and How to Build It

A detailed reference on Debezium, Kafka, Flink and retrieval-augmented generation behind assistants like J.P. Morgan's Connect Coach and its treasury analytics tools, grounded in public reporting and current research.

---


October 7, 2026.

---

## How to read this

The write-up you were given makes a case that an AI assistant "like Connect Coach" needs a streaming backend, and it makes that case with a mix of true engineering principles and invented specifics. This document keeps the principles, replaces the invented specifics with what is publicly known, adds the production realities the write-up skipped, and connects the whole thing to the current research. Where I state a fact about J.P. Morgan, it is linked to a public source. Where I describe an architecture, it is a reference pattern, not a claim about what any firm runs. Where only you can supply a detail about your own work, it is in `[BRACKETS]`.

The structure:

1. What is actually public about Connect Coach, LLM Suite, J.P. Morgan Payments' treasury assistant and agent API, and Fusion
2. The argument for freshness, made properly
3. The reference architecture, stage by stage, including the production failure modes
4. Correctness in finance: what has to be true for "what is my cash position right now" to be answered safely
5. Cost: streaming versus warehouse, honestly
6. Governance: classification, entitlements, supervision, and agents that act
7. The research frontier: streaming RAG, temporal retrieval, the KV cache, CEP and agents
8. A phased rollout plan
9. Metrics
10. How to talk about this in an interview without overclaiming
Appendices: the $5 million wire traced end to end; SQL Server and Oracle capture side by side; a data contract; a failure-mode catalog; an architect's FAQ; a glossary
11. Sources

---

## 1. What is actually public

### 1.1 Connect Coach

Connect Coach is a J.P. Morgan tool for Private Bank advisors. Public reporting describes it as assisting with client meeting preparation, idea generation and research synthesis, and reducing preparation time for client meetings ([reruption.com industry case](https://reruption.com/en/knowledge/industry-cases/jpmorgans-llm-suite-turbocharging-wealth-advisor-productivity)). Coverage of LLM Suite's launch noted that it was designed to complement the firm's other applications that handle sensitive financial information, naming Connect Coach and SpectrumGPT ([Family Wealth Report](https://www.familywealthreport.com/article.php/JP-Morgan,-Morgan-Stanley-Raise-AI-Game-%E2%80%93-Media?id=201906)).

What is not public: Connect Coach's data pipeline, its hosting, whether it is batch-fed or event-fed, or which models it uses. The write-up's claim that Connect Coach is "powered by a traditional batch system (like J.P. Morgan Fusion's standard pipeline)" is not supported by anything public, and it conflates two unrelated products (see 1.4). Do not repeat it.

### 1.2 LLM Suite

LLM Suite launched firm-wide in 2024, reached about 60,000 users by mid-2024 and 140,000 by late 2024, and by 2026 is reported at roughly 250,000 employees with access and about half using it daily, running OpenAI and Anthropic models with an eight-week refresh cycle ([CeFPro](https://connect.cefpro.com/article/view/inside-jpmorgan-llm-suite-as-ai-agents-spread-across-the-bank), [reruption.com](https://reruption.com/en/knowledge/industry-cases/jpmorgans-llm-suite-turbocharging-wealth-advisor-productivity)). The firm reports 450-plus AI use cases in production and about $2 billion in annual savings attributed to AI ([Forbes, July 2026](https://www.forbes.com/sites/bernardmarr/2026/07/01/how-jpmorgan-chase-is-building-the-ai-powered-bank-of-the-future/), [AI News](https://www.artificialintelligence-news.com/news/jpmorgan-chase-ai-strategy-2025/)). Connect Coach is described as a specialized tool within this ecosystem for Private Bank advisors.

### 1.3 J.P. Morgan Payments: the treasury assistant and the agent API

The write-up's examples (cash position across ledgers, a wire that cleared ten minutes ago, a treasurer alert, a sweep transfer) are treasury and payments scenarios, not Private Bank advisor scenarios. The relevant public products are different from Connect Coach:

- **GenAI virtual analytics assistant for treasury.** Published December 2023 by J.P. Morgan Payments: a conversational prototype that lets corporate treasurers query payments data in plain language, retrieve information, create reports and visualizations, and run analytics on account balances, payment frequencies, FX flows and supplier patterns. The article is explicit that it was a prototype and that a broadly available product would take more time. It also floats a future "self-driving treasury" in which GenAI could recommend and eventually execute transactions within treasurer-defined parameters ([J.P. Morgan Payments insights](https://www.jpmorgan.com/insights/payments/data-intelligence/genai-virtual-analytics-assistant-treasury)).
- **The API for AI agents.** On April 29, 2026, J.P. Morgan Payments announced an API giving treasury platforms and their AI agents near real-time access to balances, transactions and payment data in J.P. Morgan accounts; Atlar, an AI treasury platform, was the first to use it, with customers connecting accounts "in seconds, with no IT development required." Once connected to an ERP, agents can review payments, reconcile transactions and generate forecasts ([Treasury Management International](https://treasury-management.com/news/atlar-first-to-use-j-p-morgan-payments-new-api-to-connect-ai-agents-to-banking-data-in-seconds)).
- **The API philosophy.** J.P. Morgan Payments' developer blog frames APIs as "the data foundation" for real-time treasury intelligence: an Account Balances API for current rather than end-of-day balances, a Global Payments API for ML analysis of payment patterns, and Validation Services for anomaly detection ([developer.payments.jpmorgan.com](https://developer.payments.jpmorgan.com/blog/ai-ml/rethinking-treasury-ai-apis)).
- **A job posting** for a "Product Manager, Agentic Treasury-Payments Data & Analytics, Vice President" in Hoboken confirms the firm is staffing agentic treasury analytics as a product line ([eFinancialCareers](https://www.efinancialcareers.com/jobs-United_States-Hoboken-Product_Manager_Agentic_Treasury-Payments_Data__Analytics-Vice_President.id24559293)).

What this means: the freshness argument in the write-up is right for treasury, and J.P. Morgan is publicly moving in that direction with near-real-time balance APIs for agents. But the product that argument applies to is the Payments analytics assistant and the agent API, not Connect Coach.

### 1.4 Fusion by J.P. Morgan

Fusion is a data management platform from J.P. Morgan Securities Services for institutional investors. It consolidates portfolio data across public and private holdings, ingests data from Securities Services and portfolio administrators, integrates vendor reference data (Aumni, Canoe, MSCI Private Capital Solutions, PitchBook), offers a Data Explorer, and connects to Excel, Tableau and Alteryx through Fusion Drive. Its private-markets data service launched October 24, 2024, and it uses proprietary AI/ML to find and correct data discrepancies ([Opalesque](https://www.opalesque.com/industry-updates/7525/morgan-launches-private-market-data-service-for.html)). In December 2025, Crunchbase's predictive intelligence was added to Fusion for private-company data ([Crunchbase press release](https://about.crunchbase.com/press/press-releases/crunchbase-predictive-intelligence-now-available-in-fusion-by-j-p-morgan)).

Fusion is a client-facing data platform for investors. It is not the backend of an advisor assistant, and nothing public describes its internal pipeline as "standard batch." The write-up used it as a foil; drop that.

### 1.5 The corrections, in one table

| Claim in the write-up | Status | What to say instead |
| --- | --- | --- |
| Connect Coach is powered by Fusion's batch pipeline | Unsupported; conflates two products | "Connect Coach's pipeline isn't public. The freshness argument applies most clearly to treasury analytics, where J.P. Morgan has moved to near-real-time balance APIs for agents." |
| "Zero hallucination lag" | Wrong on two counts: freshness doesn't eliminate hallucination, and stream lag is never zero | "Freshness in seconds instead of hours, and a smaller window in which the assistant answers from stale data. Hallucination is a separate problem handled by grounding, citations and evaluation." |
| "Sub-second operational partner" | Possible for the pipeline; the LLM step alone is usually one to several seconds | "Seconds end to end, dominated by retrieval and generation, with the data itself fresh to within seconds." |
| Snowflake "charges heavily" so stream instead | Over-simplified; warehouses have cheap serving tiers and streaming has 24/7 cost | "Choose by access pattern: a serving store or cache for per-request reads, the warehouse for analytics, S3 for history. Stream processing cost is continuous and must be justified by the freshness requirement." |
| "SageMaker RAG" | Unspecified; SageMaker is a model platform, not a RAG product | "A retrieval service over a vector index, with the model served through Bedrock, SageMaker or an internal gateway. Which one is an infrastructure choice." |
| Flink "triggers a Kafka event that alerts SageMaker" and the assistant "sends a push notification" | The mechanics are plausible; the governance is missing | "CEP detects the pattern; a notification service with entitlements and rate limits delivers it; any drafted action goes through an approval gate." |
| "Would you like me to draft a sweep transfer?" | A real design direction (J.P. Morgan floated "self-driving treasury" in 2023) but a high-risk action | "Drafting is fine with human approval; executing needs limits, dual control and audit, and should be the last capability shipped, not the first." |

---

## 2. The argument for freshness, made properly

The write-up's core claim is sound: an assistant that answers from last night's data will, in treasury and payments, give wrong answers about money. Here is the argument with the right qualifications.

### 2.1 Where freshness changes the answer

- **Cash position and liquidity.** Balances change with every cleared payment. A treasurer asking "what can I sweep today" needs intraday balances, which is why J.P. Morgan's own Account Balances API emphasizes current rather than end-of-day figures ([developer blog](https://developer.payments.jpmorgan.com/blog/ai-ml/rethinking-treasury-ai-apis)).
- **Payment status and exceptions.** "Did the wire go out," "why was this payment rejected," and "what's stuck" are questions whose answers are minutes old at most.
- **Fraud and anomaly signals.** Velocity, impossible-travel and account-takeover patterns are only useful while the window to act is open.
- **Client context for advisors.** Less time-critical than treasury, but a Private Bank advisor preparing for a meeting still wants the latest holdings, transactions and interactions, not a snapshot from the previous evening.

### 2.2 Where freshness does not change the answer

- Research, product documentation, policy and approved language change on human timescales. A daily or hourly refresh is fine, and a bounded corpus like approved disclosures can be preloaded rather than retrieved at all.
- Historical analysis, period-end reporting and reconciliation need completeness and reproducibility more than speed. Those stay batch, by design.
- Model training sets are built from history; the stream feeds them, but the training job is a batch.

The design rule: **stream what the decision depends on in seconds or minutes; batch what needs to be complete and reproducible; and reconcile the two on a schedule.** Most real platforms are both. The streaming RAG literature supports this: incremental index updates beat periodic rebuilds on latency and recall for changing content ([arXiv 2508.05662](https://arxiv.org/abs/2508.05662)), and under real knowledge drift, time-aware retrieval beats fine-tuning ([ACL Findings 2026](https://preview.aclanthology.org/ingest-acl/2026.findings-acl.546/)). Neither says to stream everything.

### 2.3 Freshness is not accuracy

Three distinct properties, often confused:

- **Freshness:** how old the data the assistant reads is. Streaming improves this.
- **Correctness:** whether the data is right. A stream can deliver wrong numbers quickly; exactly-once semantics, idempotent sinks and reconciliation protect this.
- **Groundedness:** whether the assistant's answer is supported by the data it read. Retrieval design, citations and evaluation protect this. Hallucination is a groundedness problem, not a freshness problem.

A streaming backend buys the first. The other two are separate engineering and governance work, and a system that delivers fresh, wrong, confidently-stated numbers is worse than a slow one.

---
## 3. The reference architecture, stage by stage

```
SYSTEMS OF RECORD          CAPTURE               BACKBONE                PROCESSING
SQL Server, Oracle  --->   Debezium on    --->   Kafka / Amazon MSK --->  Flink (Managed Service for
(ledgers, payments,        Kafka Connect /       topics per table,        Apache Flink, or on EKS)
 positions, CRM)           MSK Connect           keyed by entity,         - parse before/after images
                                                 schema registry          - clean, classify, enrich
                                                 (backward compat)        - running balances (keyed state)
                                                                          - CEP patterns (liquidity, velocity)
                                                                          - async embeddings for text
                                                                          - dedupe, route
                                                            |                       |
                                                            v                       v
STORES                                       SERVING AND GOVERNANCE          ASSISTANT AND AGENTS
S3 / Iceberg lakehouse (history, replay)     Retrieval service with           Conversational analytics
Vector index (OpenSearch, pgvector)          entitlement filters             Proactive alerts (push)
Serving store / cache (current balances)     LLM gateway (Bedrock,           Drafted actions with approval
Feature store (for ML scoring)               SageMaker, or internal)         Executed actions under limits
Warehouse (Snowflake) for analytics          Logging, retention, eval
```

### 3.1 Systems of record: SQL Server and Oracle

Treasury and payments data typically live in relational systems of record: ledgers, payment hubs, account masters, and in Private Bank, positions and CRM. Before any capture:

- **Scope by table and column.** CDC captures every change. The product decision is which tables are in, which columns are excluded (card numbers, full account numbers, personal identifiers) and what classification each carries. Exclusion at the source is cheaper and safer than filtering downstream.
- **Confirm the database is CDC-ready.** SQL Server needs CDC enabled per database and per table, with the capture job running and the SQL Server Agent available. Oracle needs supplemental logging (minimal at the database level, and all-columns or primary-key logging on captured tables) and archive-log mode with retention long enough for the connector to catch up after an outage.
- **Measure the change rate.** Bulk jobs, end-of-day postings and month-end runs create bursts that size the rest of the pipeline. A ledger that posts 50 million rows at close is a different design from one that trickles all day.

### 3.2 Capture: Debezium, with the production realities

Debezium reads the database's transaction log and emits one event per row change carrying the before image, the after image, the operation type and the source timestamp. It runs as a Kafka Connect connector, or on Amazon MSK Connect. This is log-based capture: no polling load on the source, every change in order, within seconds ([AWS: end-to-end CDC with MSK Connect](https://aws.amazon.com/blogs/big-data/build-an-end-to-end-change-data-capture-with-amazon-msk-connect-and-aws-glue-schema-registry/)).

Debezium is actively developed: 3.5 and 3.6 releases through 2026 and a 3.7 line in progress ([Debezium blog, March 2026](https://debezium.io/blog/2026/03/16/debezium-3-5-beta2-released/), [May 2026](https://debezium.io/blog/2026/05/29/debezium-3-6-beta1-released/), [August 2026](https://debezium.io/blog/2026/08/13/debezium-3-7-alpha2-released/)).

What the write-up skipped, and what a PM should know before promising "captures the wire transfer instantly":

- **Oracle is the hard one.** The Oracle connector supports LogMiner and XStream (XStream requires a GoldenGate license). A Confluent Current 2026 talk by MarketAxess engineers titled "The Plug & Play Lie: Why Your Oracle CDC Pipeline Will Fail" covers type-system traps (unbounded numerics, complex timestamps that break consumers), the need for generated connector configuration as a contract, continuous reconciliation to prove accuracy, position recovery, heartbeats for quiet tables, and regional failover ([Confluent Current 2026](https://current.confluent.io/post-conference-videos-26/the-plug-play-lie-why-your-oracle-cdc-pipeline-will-fail-ldn26)). Treat Oracle CDC as a program, not a connector.
- **The initial snapshot.** A connector starts with a consistent snapshot of existing rows, then switches to the log. Snapshotting large tables takes hours and loads the source; incremental snapshots (chunked, interleaved with streaming) are the usual answer. Plan it.
- **Large transactions.** A single transaction updating millions of rows arrives as millions of events at once. Size Kafka partitions and Flink parallelism for the burst, not the average.
- **Schema changes.** DDL on the source flows into events. Without a schema registry and compatibility rules, a column rename takes down every consumer.
- **Deletes and tombstones.** Decide how deletes are represented downstream (tombstone events, soft-delete flags) and how the vector index handles a deleted document.
- **Quiet tables and heartbeats.** A table with no changes for hours can make the connector's position look stale; heartbeat events keep offsets moving.
- **Numeric precision.** Monetary amounts must survive serialization exactly. Decimal handling mode is a configuration decision with audit consequences.

### 3.3 Backbone: Kafka or Amazon MSK, with a schema registry

- **Topics and keys.** One topic per source table is the common default; key by the entity whose changes must stay ordered (account ID, portfolio ID). Kafka orders within a partition only.
- **Schema registry and data contracts.** Avro or Protobuf schemas in a registry (AWS Glue Schema Registry or Confluent) with backward compatibility: a consumer on the new schema can read old data. The producer's serializer validates against the registry; the broker does not. Incompatible messages go to a dead-letter topic rather than crashing consumers ([AWS CDC pattern](https://aws.amazon.com/blogs/big-data/build-an-end-to-end-change-data-capture-with-amazon-msk-connect-and-aws-glue-schema-registry/)). The data contract adds ownership, semantics, quality rules and a change process with a deprecation window.
- **Retention.** Long enough to replay after an incident or a logic change. For a financial ledger stream, days at minimum; tiered storage to S3 for longer.
- **Security.** Private brokers, SASL/SCRAM or IAM authentication, encryption in transit, PrivateLink for cross-account consumers, and per-topic access control so the treasury topics are not readable by teams that shouldn't see them.
- **Idempotent producers and transactions.** Enable producer idempotence so retries don't duplicate; use transactions when a producer writes to several partitions atomically. Consumers with read-committed isolation see only committed data.

### 3.4 Processing: Flink

Amazon Managed Service for Apache Flink runs the cluster, scaling, checkpointing and failover; capacity is in KPUs (1 vCPU, 4 GB memory, 50 GB storage each). Flink on EKS gives more control at more operational cost. Jobs are written in Java or Scala (DataStream API) for stateful logic and async I/O, Python (PyFlink), or Flink SQL for declarative transforms.

What the jobs do for an assistant backend:

1. **Parse and classify.** Take the after image, keep the source timestamp as event time, attach the data classification, drop excluded fields.
2. **Running balances and positions.** Keyed state per account: apply each posting to a running balance. This is where exactly-once matters (Section 4).
3. **Enrichment and joins.** Attach counterparty, client, product and entitlement metadata from reference data or other streams. Decide how long join state is held.
4. **Complex event processing.** Pattern rules over the stream: liquidity below threshold after a sequence of outbound wires; velocity anomalies; impossible-travel sequences. FlinkCEP is the library; the 2026 argument for pairing it with agents is that CEP cuts a stream of hundreds of thousands of raw events to a few hundred confirmed, timestamped patterns per hour, so the expensive, non-deterministic model only reasons about high-signal inputs. Pattern detection is deterministic and auditable; the agent's reasoning is not ([Kai Waehner, April 2026](https://www.kai-waehner.de/blog/2026/04/28/flink-cep-and-agentic-ai-real-time-pattern-detection-as-the-foundation-for-autonomous-decisions/)).
5. **Embeddings for text.** For documents, notes and messages that feed retrieval, call an embedding model asynchronously (Flink Async I/O) so the stream doesn't block on HTTP, chunk fixed-size for streaming workloads, and upsert vectors by document ID. AWS ships this as a managed blueprint: MSK to Flink (with deduplication) to Bedrock embeddings to an OpenSearch vector index ([AWS blueprint](https://aws.amazon.com/blogs/big-data/build-up-to-date-generative-ai-applications-with-real-time-vector-embedding-blueprints-for-amazon-msk)). Structured ledger rows generally should not be embedded; they belong in a serving store and are retrieved by key, not by similarity.
6. **Deduplicate and route.** At-least-once delivery repeats on retry; dedupe on event ID. Route to the right sinks.

**Reliability.** Checkpoints snapshot state and Kafka offsets together; on failure the job restores and replays. Savepoints before releases. Watch consumer lag, backpressure, checkpoint duration and failures, and end-to-end latency. Late events: event time, watermarks, bounded lateness, side outputs reconciled by batch.

### 3.5 Stores

| Store | Role | Notes |
| --- | --- | --- |
| **Serving store or cache** (DynamoDB, Redis, a keyed table) | Current balances, positions, payment status, read by key per request | This is what answers "what is my cash position." Upsert by key; carry an as-of timestamp and the source event ID. |
| **Vector index** (OpenSearch, pgvector on Aurora) | Documents, notes, research, approved language | Upsert by document ID; metadata for entitlement and recency filtering; cite the source. |
| **S3 / Iceberg lakehouse** | History, replay, reprocessing, training sets | Partition by date; rebuild any index from here if the embedding model changes. AWS's 2026 agentic-streaming guidance uses S3 Tables (Iceberg) for this layer ([AWS](https://aws.amazon.com/blogs/big-data/powering-agentic-ai-with-real-time-streaming-data-on-aws/)). |
| **Feature store** | Features for ML scoring (fraud, forecasting) | Streaming features computed once, served to inference and training alike. |
| **Warehouse** (Snowflake or similar) | Analytics, reporting, reconciliation | Fed from the lakehouse or the stream; not in the per-request path. |

### 3.6 Serving and the LLM layer

- **Retrieval service.** Takes the user's question, resolves the user's entitlements, fetches structured facts by key from the serving store (balances, status) and unstructured context by similarity from the index (notes, research), both filtered by entitlement and recency, and returns ranked, cited context with as-of timestamps.
- **LLM gateway.** A single, logged path to approved models. Bedrock offers managed foundation models and Knowledge Bases, including a custom connector that ingests streaming documents without staging ([AWS](https://aws.amazon.com/blogs/machine-learning/stream-ingest-data-from-kafka-to-amazon-bedrock-knowledge-bases-using-custom-connectors)); SageMaker hosts custom or fine-tuned models; many banks run an internal gateway in front of either so prompts, responses, cost and model versions are logged centrally. "SageMaker RAG" is not a product; the retrieval service plus a model endpoint is the system.
- **Prompt assembly.** Stable policy text and instructions first (so prefix caching applies), then retrieved facts with their timestamps and sources, then the question. Structured facts go in as structured facts, not prose, so the model cannot quietly change a number.

### 3.7 The assistant and the agent

- **Conversational analytics.** "What is my cash position" answered from the serving store with an as-of time and the source; "show me supplier payment patterns" answered from the warehouse or lakehouse through a tool call, not from the vector index.
- **Proactive alerts.** CEP emits a pattern match; an alerting service applies entitlements (who may see this), rate limits (no alert storms), and delivery rules; the model drafts the human-readable message from a template with the facts filled in; the message carries the as-of time.
- **Drafted actions.** "Draft a sweep transfer to cover it." The agent prepares the instruction; a human with the right entitlement reviews and approves; the action executes through the existing payments API with its own validation. This is the pattern J.P. Morgan floated as "self-driving treasury within treasurer-defined parameters" in 2023 and the direction the April 2026 agent API points toward.
- **Executed actions.** The last capability to ship. Limits per action and per day, dual control above thresholds, a kill switch, full tracing, and a reconciliation that proves every executed action matches an approved instruction.

---

## 4. Correctness in finance: what has to be true for "what is my cash position right now"

The write-up's example is a $5 million wire that cleared ten minutes ago and a batch assistant that misses it. A streaming assistant can also get it wrong, in ways that are worse because they are confident and fast. Here is what has to be true.

### 4.1 The balance must be computed exactly once

A running balance is keyed state updated by each posting. If a posting is applied twice after a restart, the balance is wrong by the amount of that posting. Flink gives exactly-once inside its own state through checkpoints; end to end, the sink must commit transactionally with the checkpoint or write idempotently by key. For a serving store, the practical form is an upsert keyed by account with the last applied event ID, so a replayed event is a no-op. Say "effectively once," and say how.

### 4.2 Event time, not arrival time

A payment posted at 09:58 that arrives in the stream at 10:03 because of a connector hiccup must be applied as of 09:58. Process by event time with watermarks; the balance the assistant reports should carry an as-of time ("as of 10:03:12, including postings through 10:03:05"), so a treasurer knows what it includes.

### 4.3 Ordering within an account

All postings for one account must be applied in order. Key the Kafka topic by account so they land in one partition; never spread one account across partitions for throughput.

### 4.4 The source is the truth; the stream is a projection

The ledger is the system of record. The streaming balance is a projection of it that can drift through bugs, late data or a bad deployment. A scheduled reconciliation (hourly for treasury, nightly at minimum) compares the projection to the ledger and raises a discrepancy before a treasurer does. The write-up's "old batch style" is not eliminated; it becomes the control.

### 4.5 The assistant must not arithmetic

Models do arithmetic unreliably. The assistant should present the balance the serving store computed, with its as-of time and source, and should not sum, net or convert in the prompt. If a question needs computation (net position across currencies), a tool does it deterministically and the model explains the result.

### 4.6 Lineage and audit

Every number the assistant states should be traceable: which events, from which source tables, through which job version, as of what time. This is the same lineage discipline you built at JPMorgan with Qlik Catalog, now for a stream. Standards like OpenLineage let jobs emit it automatically. BCBS 239's principles on accuracy, completeness, timeliness and traceability of risk data are the regulatory backdrop in banking.

### 4.7 What "sub-second" really means

The pipeline from posting to serving store can be seconds. The assistant's response adds retrieval (tens to hundreds of milliseconds), prompt assembly, and model generation (one to several seconds for a useful answer). "Sub-second" describes the data path, not the conversation. Say "fresh to within seconds; answers in a few seconds."

---
## 5. Cost: streaming versus warehouse, honestly

The write-up argues that querying Snowflake on every assistant request is "incredibly expensive" and that in-memory Flink plus S3 is cheaper. Part of that is right and part is a false comparison.

### 5.1 What is right

- Per-request analytical queries against a warehouse are the wrong access pattern for an assistant answering "what is my balance." A keyed lookup in a serving store is milliseconds and fractions of a cent; a warehouse query is seconds and keeps compute warm.
- S3 is far cheaper than warehouse storage for history and replay, and Iceberg tables on S3 make that history queryable by several engines.
- A stream computes a running aggregate once and serves it many times; a warehouse recomputes it per query unless you materialize.

### 5.2 What is a false comparison

- A stream runs 24/7. Managed Flink bills per KPU-hour whether or not events are flowing; Kafka brokers and storage bill continuously; connectors bill continuously. A nightly batch bills for its run. For a workload where freshness isn't needed, the stream costs more.
- Warehouses have cheap serving options (result caching, small warehouses, materialized views) and most firms keep one anyway for analytics and reconciliation. The assistant doesn't remove it; it stops hitting it per request.
- Operational cost: a stream needs on-call, lag monitoring, state management, and replay procedures. People cost dominates for small teams.

### 5.3 The honest cost model

| Component | Cost shape | Levers |
| --- | --- | --- |
| Debezium / MSK Connect | Continuous per worker | Right-size workers; one connector per source cluster |
| Kafka / MSK | Continuous per broker plus storage | Retention tiers; tiered storage to S3; partition count |
| Managed Flink | Continuous per KPU | Autoscaling; parallelism tuned to partitions; move non-urgent jobs to micro-batch |
| Serving store | Per request plus storage | Cache hot keys; TTL on cold ones |
| Vector index | Per node plus storage | Index only what needs similarity search; structured data goes to the serving store |
| Embeddings | Per token | Embed on change, not on schedule; dedupe before embedding |
| LLM generation | Per token, dominated by prefill | Prefix caching; shorter context; smaller model for classification, larger only for drafting |
| S3 / Iceberg | Per GB, cheap | Partition and compact; lifecycle to colder tiers |
| Warehouse | Per compute-second when active | Keep it out of the per-request path; use it for analytics and reconciliation |

The decision rule from Section 2 applies: stream what the decision depends on in seconds; batch the rest. For a treasury assistant, balances, payment status and alerts justify the stream. Research, policy and documentation do not.

---

## 6. Governance: classification, entitlements, supervision, and agents that act

A streaming backend moves data faster. It also moves data more widely, to more consumers, with less time for a human to notice a mistake. The governance has to be designed in, which is the discipline your resume already carries.

### 6.1 Data classification and minimization

- Classify at the source table and column level before capture. Exclude what the assistant doesn't need; tokenize what it needs but must not display.
- Mask or tokenize at ingestion so downstream compute, indexes and prompts never hold raw sensitive fields. Masking at the end of the pipeline is a compliance finding waiting to happen.
- Keep a register of which classifications flow to which topics, stores and models. This is the control auditors ask for first.

### 6.2 Entitlements

- A treasurer sees their company's accounts; an advisor sees their book. Enforce at the retrieval and serving layer, by key and by metadata filter, not by hoping the model doesn't mention the wrong client.
- Agents inherit the entitlements of the user they act for, scoped per task, never a service account with broad access.
- Test it: an entitlement breach test belongs in the evaluation set, with zero tolerance.

### 6.3 Supervision and retention

- Prompts, retrieved context, responses, reviewer and decision are logged and retained per the applicable rules. For advisor-to-client communications, FINRA Rule 2210 on communications with the public and FINRA Rule 3110 on supervision apply; SEC Rule 17a-4 governs retention. For treasury analytics, the firm's own records and model-risk policies apply.
- Model risk: SR 11-7, the Federal Reserve and OCC guidance on model risk management, is how US banks govern models, and firms apply it to generative AI. Every model, prompt and retrieval change is a change to a model's behavior and goes through the same evaluation gate.

### 6.4 Evaluation as a release gate

- A versioned scenario set of real questions and alert cases, including adversarial ones (prompt injection through a payment memo field, a request for another client's balance).
- Automated checks: numbers in the answer match the serving store; as-of time present; citations present; no prohibited statements; no entitlement breach.
- Human grading on tone and usefulness, with an LLM-as-judge only as a calibrated input. A March 2026 benchmark on SEC filings found LLM judges over-trust structured signals even when they contradict the text ([FinReflectKG-HalluBench](https://arxiv.org/html/2603.20252v1)); for a system that hands the model structured balances, that finding matters.
- Thresholds that block release, tighter for anything that drafts or executes an action.
- Post-release monitoring: edit rate on drafts, alert precision, false alarms, escalations, drift.

### 6.5 Agents that act

The write-up ends with "would you like me to draft a sweep transfer?" J.P. Morgan itself has described the destination as recommending and eventually executing transactions "within treasurer-defined parameters" ([J.P. Morgan Payments, 2023](https://www.jpmorgan.com/insights/payments/data-intelligence/genai-virtual-analytics-assistant-treasury)). The controls for that:

- **Least privilege per task.** The agent's permission is to draft, not to execute, until the execute capability is separately approved.
- **Approval gates.** Human approval for any action that moves money or changes a record, with the approver's identity logged. Dual control above thresholds.
- **Limits.** Per-action, per-day and per-counterparty limits set by the treasurer, enforced by the payments system, not by the prompt.
- **Untrusted input.** Everything the agent reads (a payment memo, an email, a document) is data, never an instruction. Prompt injection through a free-text field in a payment is a realistic attack.
- **Tracing.** Every step: trigger, context, reasoning summary, tool calls, outcome.
- **Kill switch and rollback.** A way to stop the agent fleet in one action, and a reconciliation that proves executed actions match approved instructions.
- **Sequencing.** Ship conversational analytics first, alerts second, drafted actions third, executed actions last, each behind its own evaluation gate and its own pilot.

The 2026 agentic RAG survey's own lessons apply: retrieval precision is still the bottleneck, agents need explicit constraints, and evaluation must cover the process, not only the output ([arXiv 2501.09136](https://arxiv.org/html/2501.09136v4)).

---

## 7. The research frontier, connected to the design

### 7.1 Keeping retrieval fresh

| Paper | Finding | Design implication |
| --- | --- | --- |
| [From Static to Dynamic: Streaming RAG (arXiv 2508.05662)](https://arxiv.org/abs/2508.05662) | Incremental index updates with a compact prototype set beat periodic rebuilds: recall up about 3 points, latency under 15 ms, 900-plus docs/s in 150 MB | Per-record upserts from CDC (Section 3.4 step 5) are the right default for the document side |
| [RAG Meets Temporal Graphs, TG-RAG (arXiv 2510.13590)](https://arxiv.org/abs/2510.13590v1) | Embeddings can't distinguish the same fact at different times; timestamped knowledge with incremental updates fixes it | Every chunk and every served fact carries event time; retrieval filters by validity window |
| [RAG or Learning? (ACL Findings 2026)](https://preview.aclanthology.org/ingest-acl/2026.findings-acl.546/) | Under continuous knowledge drift, time-aware retrieval beats fine-tuning; adaptation methods suffer forgetting and temporal inconsistency | Don't fine-tune on changing data; retrieve with time structure |
| [Event-Causal RAG (arXiv 2605.06185)](https://arxiv.org/abs/2605.06185) | In long video, event-aligned memory beats fixed windows (2.5 points) and dedup adds 3 points | Chunk on events (a payment, a thread, a CDC change), not fixed sizes; dedupe before prompting |
| [AWS: Powering agentic AI with real-time streaming data (2026)](https://aws.amazon.com/blogs/big-data/powering-agentic-ai-with-real-time-streaming-data-on-aws/) | Three patterns: streaming features to inference to action; event-driven agent invocation with pre-assembled context; CDC keeping agent memory in sync | The alert path (CEP to pre-assembled context to agent) is pattern two; the serving store is pattern three |

### 7.2 Latency and cost in the model layer

| Paper | Finding | Design implication |
| --- | --- | --- |
| [CacheBlend (arXiv 2405.16444)](https://arxiv.org/abs/2405.16444) | Reuse precomputed KV caches for retrieved chunks with selective recompute: time-to-first-token 2.2 to 3.3x faster, throughput 2.8 to 5x | Stable, versioned chunks with stable IDs so caches are reusable |
| [CacheClip (arXiv 2510.10129)](https://arxiv.org/abs/2510.10129) | A small auxiliary model picks tokens to recompute; 85 to 91% of full-attention quality at 20% recompute | Quality trade exists; put reuse methods through the evaluation gate |
| [FusionRAG (SIGMOD 2026)](https://arxiv.org/pdf/2601.12904v1) | TTFT 2.66 to 9.39x faster; at least 80% quality at 15% recompute | Same |
| [Chunk-level KV reuse study (arXiv 2603.20218)](https://arxiv.org/html/2603.20218v1) | Existing methods still 7 to 18% below full prefill | In a system that states balances, accept no reuse method that moves numbers |
| [KVShareArena (arXiv 2609.10266)](https://arxiv.org/html/2609.10266v1) | Trained adapters can fail silently, fluent answers with wrong entities | A wrong account or client name is a compliance event; prefer exact prefix caching for the stable prompt and full prefill for the facts |
| [Cache-Augmented Generation (arXiv 2412.15605)](https://arxiv.org/abs/2412.15605) | Preload a small bounded corpus; zero retrieval latency | Approved language and disclosures can be preloaded; client facts cannot |

### 7.3 Detection before reasoning

The CEP-before-agents argument ([Kai Waehner, April 2026](https://www.kai-waehner.de/blog/2026/04/28/flink-cep-and-agentic-ai-real-time-pattern-detection-as-the-foundation-for-autonomous-decisions/)) is the architectural answer to volume, cost and reliability at once: deterministic pattern detection reduces half a million raw events an hour to a few hundred confirmed matches, and the model reasons only about those. It also separates the auditable part (which pattern fired, on which events, at what time) from the non-deterministic part (what the agent recommended). For a bank, that separation is what makes the alert path supervisable.

### 7.4 Where the GPU layer fits

Below the KV cache sits the kernel. FlashInfer, the attention engine under vLLM and SGLang, is the production bridge: block-sparse KV layouts, JIT-compiled attention variants, a load-balanced scheduler ([MLSys 2025](https://proceedings.mlsys.org/paper_files/paper/2025/file/dbf02b21d77409a2db30e56866a8ab3a-Paper-Conference.pdf)). An April 2026 hybrid JIT-CUDA Graph design cut time-to-first-token up to 66% on short prompts ([arXiv 2604.23467](https://arxiv.org/abs/2604.23467)). LLM-written kernels are real but, per a September 2026 workload study, move a transformer about 1% end to end because most runtime already sits in tuned libraries ([arXiv 2609.21058](https://arxiv.org/html/2609.21058)). For a bank's assistant, inference cost is moved by prompt design and caching, not by kernel work; that layer is bought from the serving stack.

---
## 8. A phased rollout plan

Treat the backend and the assistant as two programs with their own gates. Milestones are decisions, not dates.

### Phase 0: Scope and contracts (weeks, not months)
- Inventory the source tables, classify every column, agree exclusions and tokenization.
- Write the data contracts: schema, owner, semantics, change process, deprecation window.
- Define the Definition of Done for a streamed fact: exactly-once applied, event-timed, entitled, reconciled, lineaged.
- Pick the first use case by value and risk: usually payment status or balances for a single client segment, conversational only, no actions.
- Build the evaluation scenario set and the golden set for retrieval before any pipeline work.

### Phase 1: Parallel run
- Debezium on the first source (SQL Server first; Oracle is the harder program and goes second).
- Kafka topics with the registry and compatibility checks in CI; dead-letter topic with a named owner.
- Flink job computing the first projections (balances, status) into a separate serving store; existing batch keeps feeding the live system.
- Reconciliation job comparing the projection to the ledger hourly; discrepancy alerts to the pipeline owner.
- Gate: a week of reconciliation within tolerance, lag and checkpoint health within targets, no entitlement test failures.

### Phase 2: Shadow verification
- Mirror a share of live assistant queries to the new serving store and index; compare answers against the batch-fed system and the golden set on correctness, freshness and latency.
- Run the evaluation scenario set against the new path on every change.
- Gate: correctness parity or better, freshness improved as measured, latency within budget, zero entitlement breaches.

### Phase 3: Canary cutover for conversational analytics
- Route a small share of users to the streaming-fed assistant, then widen, on error rate, latency, correctness sampling and user feedback; keep the batch path as fallback for two cycles.
- Gate: thresholds held at each step; rollback exercised once on purpose.

### Phase 4: Proactive alerts
- CEP patterns agreed with the business and risk; precision and false-alarm targets set.
- Alerting service with entitlements, rate limits, delivery rules; messages drafted from templates with facts filled in and as-of times.
- Pilot with a small cohort; measure precision, action rate, and complaints.
- Gate: precision above target, false alarms below target, no entitlement breaches, supervision logging verified.

### Phase 5: Drafted actions
- Agent drafts instructions; human approval with logged identity; execution through the existing payments path with its own validation.
- Limits defined by the user; dual control above thresholds; kill switch tested.
- Gate: every executed instruction reconciles to an approved draft; evaluation set includes injection attempts through free-text fields.

### Phase 6: Executed actions within parameters
- Only after Phase 5 has run clean for an agreed period. Per-action and per-day limits enforced by the payments system; continuous reconciliation; tracing; a documented path to revoke.

### The second source: Oracle
- Run as its own Phase 1 to 3 with the MarketAxess lessons in hand: generated connector configuration as a contract, type-handling tests for numerics and timestamps, continuous reconciliation, heartbeat for quiet tables, position recovery and failover rehearsed ([Confluent Current 2026](https://current.confluent.io/post-conference-videos-26/the-plug-play-lie-why-your-oracle-cdc-pipeline-will-fail-ldn26)).

---

## 9. Metrics

| Layer | Metrics | Target shape |
| --- | --- | --- |
| Capture | Connector lag, snapshot progress, events per second by table, DLQ volume | Lag under the freshness target; DLQ triaged within the agreed time |
| Backbone | Consumer lag per group, partition skew, schema-compatibility failures in CI | Zero compatibility failures reaching production |
| Processing | Checkpoint duration and failures, backpressure, end-to-end latency posting-to-serving | Latency within the freshness target; checkpoints healthy |
| Correctness | Reconciliation discrepancies (count and amount), duplicate-apply incidents, late-event volume | Discrepancies at zero or explained; duplicates zero |
| Retrieval | Golden-set hit rate, p95 latency, entitlement violations | Violations zero |
| Assistant | Answer correctness on sampled numbers, as-of time present, citation present, user edit rate on drafts, satisfaction | Correctness at threshold; edit rate trending down |
| Alerts | Precision, recall on known events, false-alarm rate, time from event to delivery, action rate | Precision above target; storms zero |
| Actions | Approved-to-executed reconciliation, limit breaches, dual-control compliance | Reconciliation 100%; breaches zero |
| Cost | KPU-hours, broker and storage spend, tokens per answer, cost per answer, cost per alert | Cost per answer flat or falling as volume grows |
| Governance | Scenario pass rate per release, releases blocked, audit findings, evidence completeness | Findings zero; evidence complete |

---

## 10. How to talk about this in an interview without overclaiming

You owned a pipeline of this shape under an advisor co-pilot at Morgan Stanley: CDC from a SQL source into Kafka, managed Flink in Java and SQL, feeding retrieval, with S3 and API sinks, a shadow-then-canary migration, and governance in the delivery path. That is real and on your resume. The treasury assistant and agent scenarios in this document are a reference design informed by public J.P. Morgan direction, not your experience.

**The one-minute version, for a platform or program interview:**

"Freshness only matters where the decision depends on it. For balances, payment status and alerts, a batch-fed assistant answers from yesterday, so you capture changes from the ledger with Debezium, stream them through Kafka, compute running balances and detect patterns in Flink, and serve them by key with an as-of time. The hard parts aren't the tools. They're exactly-once on the balance, ordering per account, reconciliation back to the ledger so the projection can't drift, entitlements enforced at retrieval, and keeping arithmetic out of the model. I ran a migration of this shape under an advisor co-pilot: parallel run, shadow queries, canary, with batch kept as fallback. The research backs the design: incremental index updates beat rebuilds, time-aware retrieval beats fine-tuning under drift, and pattern detection before the agent is what makes the alert path auditable. And actions that move money ship last, behind approval gates and limits, after conversational analytics and alerts have run clean."

**What you can claim and what you can't:**

| Claim | Status |
| --- | --- |
| Owned requirements and migration of a CDC-to-Kafka-to-managed-Flink pipeline feeding retrieval under a regulated AI product | Yours; on the resume |
| Built evaluation gates, risk-tiered review and monitoring into the release path | Yours |
| Know the production failure modes of Oracle CDC, exactly-once on balances, CEP-before-agents, KV-cache trade-offs | Reading, with sources; say "the research and practitioner reports show" |
| Designed or built a treasury assistant, a sweep-transfer agent, or anything for J.P. Morgan Payments | Not yours; it's a reference design |
| Know how Connect Coach or Fusion is built internally | Not public; say so |

**The follow-ups to be ready for:**
- *Why not just call the balance API per request?* "If the system of record exposes a fresh balance API with the right latency and entitlements, use it; J.P. Morgan's own direction is near-real-time balance APIs for agents. The stream earns its cost when you need running aggregates, pattern detection across events, or fan-out to several consumers, or when the source can't take per-request load."
- *How do you prove the streamed balance is right?* "Exactly-once applied with an idempotent upsert by account, event-time processing, and a scheduled reconciliation to the ledger that alerts on discrepancies. The batch doesn't go away; it becomes the control."
- *What stops the model from inventing a number?* "It never computes. The serving store does; the model presents the stored figure with its as-of time and source, and the evaluation set checks every number in a sampled answer against the store."
- *What's the first thing you'd ship?* "Conversational analytics on payment status for one client segment, read-only, with the reconciliation running. Alerts second. Drafted actions third. Executed actions last."

---

## Appendix A. A worked example: the $5 million wire, traced through the system

The write-up's scenario: a treasurer asks "what is my exact cash position across my SQL and Oracle ledgers right now," and a $5 million wire cleared ten minutes ago. Here is what has to happen, with realistic timings, and where each failure would show.

| Time | Event | Component | What can go wrong here |
| --- | --- | --- | --- |
| 10:00:00.000 | Payment hub commits the inbound wire: a row insert in `payments` and a balance update in `ledger_balances` on SQL Server | System of record | Nothing yet; the ledger is the truth |
| 10:00:00.4 | SQL Server's CDC capture job writes the change to the change table; Debezium's SQL Server connector polls the change table and emits two events with the commit LSN and timestamp | Debezium | Capture job stopped (Agent down) means no events; connector polling interval adds up to seconds; a large batch posting ahead in the log delays these rows |
| 10:00:01.2 | Events land on `ledger_balances` and `payments` topics, keyed by account ID, serialized against the registered Avro schema | Kafka | A schema change on the source without a compatible registered version sends these to the dead-letter topic; a hot partition for a very active account adds lag |
| 10:00:01.5 | Flink's Kafka source reads the events; the balance job applies the posting to keyed state for the account using the event's commit timestamp as event time; the idempotency key (event ID) is recorded | Flink | Restart between read and checkpoint means replay; without idempotent sink the balance applies twice; processing-time instead of event-time misorders a late event |
| 10:00:02.0 | Flink upserts the serving store: account, balance, as-of time 10:00:00.000, last-applied event ID | Serving store | Sink write fails and retries; idempotent upsert makes the retry safe |
| 10:00:02.0 | The same event feeds a CEP job watching for liquidity patterns; no pattern fires for an inbound wire | Flink CEP | A mis-specified pattern fires a false alarm |
| 10:00:02.5 | The event is appended to the S3/Iceberg history table for replay and reconciliation | Lakehouse | Compaction lag; no effect on the answer |
| 10:10:00 | Treasurer asks the assistant for the cash position | Assistant | Entitlement check: this user may see these accounts |
| 10:10:00.1 | Retrieval service reads the serving store by account keys; gets balances with as-of times; for the Oracle-fed accounts, the same, from the Oracle pipeline | Retrieval | An Oracle connector lagging after a log switch means those balances carry an older as-of time; the answer must show it |
| 10:10:00.2 | Prompt assembled: policy prefix (cached), then structured facts with timestamps and sources, then the question; the model is told to present, not compute | Gateway | Putting the balances in as prose invites the model to restate them wrongly |
| 10:10:02 | The model answers: "As of 10:10:00, your position across SQL-fed accounts is X (postings through 10:09:58) and across Oracle-fed accounts is Y (postings through 10:07:40). The 10:00 inbound wire of $5,000,000 is included." | Assistant | If the model summed X and Y itself, that is a correctness risk; the sum should come from a tool |
| 11:00:00 | Hourly reconciliation compares the serving-store projection to a direct ledger query for a sample of accounts; any discrepancy pages the pipeline owner | Reconciliation | This is the control that catches drift the stream can't see |

Three things this trace makes visible that the write-up didn't:

1. **The Oracle side will usually be the stale one.** Oracle CDC has more moving parts (log switches, archive retention, LogMiner session resets). The assistant must show per-source as-of times rather than one "right now."
2. **"Includes the wire" is only safe because the event ID and as-of time travel with the balance.** Without them the assistant can't say what the figure includes, and the treasurer can't trust it.
3. **The reconciliation is not optional.** A projection that drifts by a replayed event or a dropped one is wrong silently; the hourly check is the only thing that makes it loud.

---

## Appendix B. SQL Server and Oracle capture, side by side

| Concern | SQL Server (Debezium SQL Server connector) | Oracle (Debezium Oracle connector) |
| --- | --- | --- |
| Mechanism | Reads SQL Server's own CDC change tables, which the capture job fills from the transaction log | LogMiner (reads redo and archive logs) or XStream (requires GoldenGate licensing) |
| Prerequisites | CDC enabled on the database and each table; SQL Server Agent running the capture job; a login with the right role | Archive-log mode; supplemental logging at database and table level; a user with LogMiner privileges; sufficient archive retention |
| Latency | Polling interval plus capture-job latency, typically seconds | Seconds to tens of seconds; log switches and large transactions add spikes |
| Snapshot | Initial consistent snapshot per table, then streaming; incremental snapshots available | Same, with care for large tables and the SCN boundary |
| Schema changes | Requires enabling CDC on a new capture instance when columns change; otherwise changes are missed | DDL events captured; type handling for Oracle numerics and timestamps needs explicit decisions |
| Known traps | Capture job stopped silently; retention of change tables; identity and computed columns | Unbounded NUMBER precision; TIMESTAMP WITH TIME ZONE handling; quiet tables and heartbeats; LogMiner session memory; failover between RAC nodes or regions |
| Reconciliation need | High | Higher |
| Reference | [Debezium SQL Server docs](https://debezium.io/documentation/reference/connectors/sqlserver.html) | [Debezium Oracle docs](https://debezium.io/documentation/reference/connectors/oracle.html); [MarketAxess, Current 2026](https://current.confluent.io/post-conference-videos-26/the-plug-play-lie-why-your-oracle-cdc-pipeline-will-fail-ldn26) |

The program implication: sequence SQL Server first, prove the pattern end to end including reconciliation, then bring Oracle on as its own phased migration with the type-handling and failover tests written before the connector is turned on.

---

## Appendix C. A data contract for a streamed table

A contract is what turns "we have Kafka" into "consumers can depend on this." One page per topic, owned by the producing team, versioned with the schema.

```
Topic: payments.ledger_balances.v1
Owner: Treasury Ledger Platform (team), on-call rotation: ledger-platform
Source: SQL Server PAYHUB.dbo.ledger_balances via Debezium 3.6, SQL Server connector
Key: account_id (string). Ordering guaranteed per key.
Schema: Avro, registered in Glue Schema Registry as ledger_balances-value
Compatibility: BACKWARD. Adding optional fields allowed. Removing or retyping a field
  requires a new major version (v2) and a 90-day deprecation window.
Semantics:
  - balance_minor: int64, minor units (cents). Never a float.
  - currency: ISO 4217.
  - as_of_ts: source commit timestamp, UTC, microseconds. This is event time.
  - event_id: unique per change; consumers must dedupe on it.
  - op: c | u | d. Deletes carry a tombstone after the delete event.
Classification: Confidential. No PII fields. account_id is tokenized.
Freshness SLO: 95% of events on topic within 5 seconds of source commit;
  alert at 30 seconds; page at 120 seconds.
Retention: 7 days hot; 400 days tiered to S3.
Dead letter: payments.ledger_balances.dlq, triaged by owner within 1 business hour.
Replay: supported from any offset within retention; consumers must be idempotent.
Consumers (registered): balance-projection (Flink), reconciliation (batch),
  lakehouse-sink (Iceberg), fraud-features (Flink).
Change process: proposal in the contracts repo; CI compatibility check; consumer
  review for breaking changes; announcement 90 days before retirement of a version.
```

What a PM owns in this: the SLO, the classification, the change process and the consumer registry. What engineering owns: the schema, the connector and the sink.

---

## Appendix D. Failure-mode catalog: what breaks, how you'd know, what you'd do

| Failure | Symptom | Detection | Response | Prevention |
| --- | --- | --- | --- | --- |
| Capture job stopped (SQL Server Agent) | Connector offsets stop advancing; serving store goes stale | Freshness SLO alert on as-of age | Restart agent; connector catches up from change tables | Monitor the agent; heartbeat events |
| Oracle archive logs purged before read | Connector cannot resume; gap in events | Connector error; reconciliation discrepancy | Re-snapshot affected tables | Retention policy sized to worst-case outage; alert on log age |
| Schema change without compatible version | Events to DLQ; consumers see nothing new | DLQ volume alert | Register compatible schema; replay from DLQ | Compatibility check in CI; contract change process |
| Hot partition | One account's lag grows; others fine | Per-partition lag metric | Repartition or salt the key for that entity with ordering preserved per sub-key | Key design review for known high-volume entities |
| Duplicate apply after restart | Balance off by one posting | Reconciliation discrepancy | Correct from ledger; fix sink idempotency | Idempotent upsert keyed on event ID; exactly-once sink |
| Late event applied out of order | Balance momentarily wrong, then corrected | Event-time vs processing-time gap metric | None if event-time processing; otherwise reprocess | Event-time semantics with watermarks |
| Checkpoint failures | Recovery would lose state | Checkpoint failure alert | Fix cause (state size, slow storage); savepoint | Size state; monitor checkpoint duration |
| Backpressure from slow sink | Lag grows across the job | Backpressure metric | Scale sink; buffer; degrade non-critical paths | Sink capacity planning; async I/O |
| Embedding endpoint slow or down | Document index stale; balances unaffected | Async I/O timeout metrics | Retry with backoff; queue | Separate the document path from the balance path |
| Entitlement filter bug | A user sees another's data | Entitlement test in evaluation; audit sampling | Kill switch for the assistant; incident process | Entitlement tests with zero tolerance in every release |
| Model restates a number wrongly | Answer disagrees with serving store | Sampled-answer check against the store | Block release; adjust prompt to structured facts | Model presents, never computes; numbers checked in evaluation |
| Alert storm | Users flooded; trust lost | Alert rate per user metric | Rate limit; pause pattern | Rate limits and dedup in the alerting service |
| Prompt injection via free text | Agent follows an instruction in a payment memo | Red-team cases in evaluation | Treat retrieved text as data; strip instructions | Input guardrails; least privilege; approval gates |
| Agent executes outside limits | Money moved incorrectly | Reconciliation of executed vs approved | Kill switch; reversal process | Limits enforced by the payments system, not the prompt; dual control |

---

## Appendix E. Questions an architect will ask, with the short answers

- **Why not query the ledger directly per request?** If the ledger can take the load and expose a fresh, entitled API, do that for point lookups. The stream earns its cost for running aggregates, pattern detection across events, fan-out to several consumers, and protecting the source from read load. J.P. Morgan's own direction, near-real-time balance APIs for agents, is the "query directly" option done properly at the provider.
- **Kafka or Kinesis?** Kafka (MSK) when you need the ecosystem (Debezium, Flink connectors, schema registry, long retention, tiered storage) and multi-consumer replay. Kinesis for simpler AWS-native pipelines with fewer moving parts. Either works; the contract and the consumer discipline matter more.
- **Flink or Kafka Streams or Spark?** Flink for event-time windows, large keyed state (running balances for millions of accounts) and CEP. Kafka Streams for simpler per-service processing. Spark Structured Streaming when the team is already Spark-centric and micro-batch latency is acceptable.
- **Managed Flink or EKS?** Managed when you want engineers on the jobs, not the cluster, and the tuning exposed is enough. EKS when you need the Flink Kubernetes Operator's control, custom images, or you already run a platform team for it.
- **Where does the vector index belong?** For documents, notes and research only. Balances and statuses are keyed facts in a serving store. Embedding structured ledger rows is a common mistake; similarity search is the wrong access pattern for "what is my balance."
- **Bedrock or SageMaker?** Bedrock for managed foundation models, Knowledge Bases and guardrails; SageMaker for custom or fine-tuned model hosting. Most banks put an internal gateway in front of both for logging, cost and model governance. The retrieval service is yours either way.
- **How do you handle exactly-once to the serving store?** Idempotent upsert keyed by account with the last-applied event ID; the Flink checkpoint records the source offsets; a replayed event is a no-op.
- **What about multi-region?** Kafka replication across zones with minimum in-sync replicas; checkpoints and savepoints in durable storage so a job can restart in another region; replayable topics so consumers catch up; per-tier RTO and RPO, with the balance tier strictest.
- **What is the first thing you'd measure?** Freshness: age of the as-of time at the moment of each answer, p95. It is the number the whole design exists to improve, and it exposes every upstream failure.

---

## Appendix F. Glossary for non-engineers

- **Change data capture (CDC):** reading a database's log of changes and emitting each one as an event, instead of querying the database repeatedly.
- **Debezium:** the open-source connector that does CDC for SQL Server, Oracle, PostgreSQL, MySQL and others.
- **Kafka / Amazon MSK:** a durable, replayable log of events that many consumers can read independently; MSK is Amazon's managed version.
- **Schema registry:** the catalog of agreed event shapes; the producer checks against it so a change doesn't break consumers.
- **Flink:** a stream processor that keeps state (such as a running balance) and computes continuously as events arrive; Amazon offers it as a managed service.
- **Checkpoint:** an automatic snapshot of a Flink job's state and position, used to recover from failure without losing or duplicating events.
- **Savepoint:** a deliberate snapshot taken before a release so the new version resumes where the old one stopped.
- **Exactly-once:** each event takes effect once, even after failures and retries; achieved by replayable sources, checkpoints, and sinks that are transactional or idempotent.
- **Idempotent:** doing the same thing twice has the same effect as once.
- **Event time:** when something happened at the source, as opposed to when the system saw it.
- **Watermark:** the system's estimate that no older events are still coming, so a time window can close.
- **CEP (complex event processing):** detecting patterns across sequences of events (three outbound wires then a balance below threshold) in real time.
- **Serving store:** a fast key-value store holding the current state (balances, statuses) that the assistant reads per request.
- **Vector index:** a store that finds documents by meaning, used for notes, research and policy, not for numbers.
- **RAG (retrieval-augmented generation):** the assistant fetches relevant facts and documents first, then the model writes the answer from them.
- **KV cache:** the model's working memory for the prompt; reusing it across requests cuts latency but can trade accuracy.
- **Prefix caching:** reusing the cache for the identical opening part of a prompt (policies, instructions); exact and free.
- **Entitlement:** the rule for who may see which data; enforced where data is retrieved, not by the model's manners.
- **Reconciliation:** a scheduled comparison of the streamed projection against the system of record to catch drift.
- **Lineage:** the record of where a number came from: source table, events, job version, time.
- **Approval gate:** a human decision required before an action (a payment, a record change) executes.
- **Dual control:** two authorized people required for an action above a threshold.

---
## 11. Sources

Company and product
- [reruption.com: JPMorgan's LLM Suite and Connect Coach](https://reruption.com/en/knowledge/industry-cases/jpmorgans-llm-suite-turbocharging-wealth-advisor-productivity)
- [Family Wealth Report: JP Morgan, Morgan Stanley raise AI game](https://www.familywealthreport.com/article.php/JP-Morgan,-Morgan-Stanley-Raise-AI-Game-%E2%80%93-Media?id=201906)
- [CeFPro: Inside JPMorgan's LLM Suite as AI agents spread across the bank](https://connect.cefpro.com/article/view/inside-jpmorgan-llm-suite-as-ai-agents-spread-across-the-bank)
- [Forbes, July 2026: How JPMorgan Chase is building the AI-powered bank of the future](https://www.forbes.com/sites/bernardmarr/2026/07/01/how-jpmorgan-chase-is-building-the-ai-powered-bank-of-the-future/)
- [AI News: JPMorgan Chase AI strategy](https://www.artificialintelligence-news.com/news/jpmorgan-chase-ai-strategy-2025/)
- [J.P. Morgan Payments: GenAI virtual analytics assistant for treasury (Dec 2023)](https://www.jpmorgan.com/insights/payments/data-intelligence/genai-virtual-analytics-assistant-treasury)
- [Treasury Management International: Atlar first to use J.P. Morgan Payments' new API for AI agents (April 2026)](https://treasury-management.com/news/atlar-first-to-use-j-p-morgan-payments-new-api-to-connect-ai-agents-to-banking-data-in-seconds)
- [J.P. Morgan Payments developer blog: Rethinking treasury AI APIs](https://developer.payments.jpmorgan.com/blog/ai-ml/rethinking-treasury-ai-apis)
- [eFinancialCareers: Product Manager, Agentic Treasury-Payments Data & Analytics, VP](https://www.efinancialcareers.com/jobs-United_States-Hoboken-Product_Manager_Agentic_Treasury-Payments_Data__Analytics-Vice_President.id24559293)
- [Opalesque: J.P. Morgan launches private market data service (Fusion)](https://www.opalesque.com/industry-updates/7525/morgan-launches-private-market-data-service-for.html)
- [Crunchbase: Predictive intelligence now available in Fusion by J.P. Morgan](https://about.crunchbase.com/press/press-releases/crunchbase-predictive-intelligence-now-available-in-fusion-by-j-p-morgan)

Engineering and AWS
- [AWS: End-to-end CDC with Amazon MSK Connect and AWS Glue Schema Registry](https://aws.amazon.com/blogs/big-data/build-an-end-to-end-change-data-capture-with-amazon-msk-connect-and-aws-glue-schema-registry/)
- [AWS: Real-time vector embedding blueprints for Amazon MSK](https://aws.amazon.com/blogs/big-data/build-up-to-date-generative-ai-applications-with-real-time-vector-embedding-blueprints-for-amazon-msk)
- [AWS: Stream ingest data from Kafka to Bedrock Knowledge Bases](https://aws.amazon.com/blogs/machine-learning/stream-ingest-data-from-kafka-to-amazon-bedrock-knowledge-bases-using-custom-connectors)
- [AWS: Powering agentic AI with real-time streaming data](https://aws.amazon.com/blogs/big-data/powering-agentic-ai-with-real-time-streaming-data-on-aws/)
- [AWS: Exploring real-time streaming for generative AI applications](https://aws.amazon.com/blogs/big-data/exploring-real-time-streaming-for-generative-ai-applications/)
- [Debezium 3.5 beta2 (March 2026)](https://debezium.io/blog/2026/03/16/debezium-3-5-beta2-released/), [3.6 beta1 (May 2026)](https://debezium.io/blog/2026/05/29/debezium-3-6-beta1-released/), [3.7 alpha2 (August 2026)](https://debezium.io/blog/2026/08/13/debezium-3-7-alpha2-released/)
- [Debezium Oracle connector documentation](https://debezium.io/documentation/reference/connectors/oracle.html)
- [Confluent Current 2026: The Plug & Play Lie, why your Oracle CDC pipeline will fail (MarketAxess)](https://current.confluent.io/post-conference-videos-26/the-plug-play-lie-why-your-oracle-cdc-pipeline-will-fail-ldn26)
- [Kai Waehner, April 2026: Flink CEP and agentic AI](https://www.kai-waehner.de/blog/2026/04/28/flink-cep-and-agentic-ai-real-time-pattern-detection-as-the-foundation-for-autonomous-decisions/)
- [FlashInfer, MLSys 2025](https://proceedings.mlsys.org/paper_files/paper/2025/file/dbf02b21d77409a2db30e56866a8ab3a-Paper-Conference.pdf)

Research
- [arXiv 2508.05662: From Static to Dynamic, a Streaming RAG approach](https://arxiv.org/abs/2508.05662)
- [arXiv 2510.13590: RAG Meets Temporal Graphs](https://arxiv.org/abs/2510.13590v1)
- [ACL Findings 2026: RAG or Learning? Knowledge drift](https://preview.aclanthology.org/ingest-acl/2026.findings-acl.546/)
- [arXiv 2605.06185: Event-Causal RAG](https://arxiv.org/abs/2605.06185)
- [arXiv 2501.09136: Agentic RAG survey (v4, April 2026)](https://arxiv.org/html/2501.09136v4)
- [arXiv 2603.20252: FinReflectKG-HalluBench](https://arxiv.org/html/2603.20252v1)
- [arXiv 2405.16444: CacheBlend](https://arxiv.org/abs/2405.16444)
- [arXiv 2510.10129: CacheClip](https://arxiv.org/abs/2510.10129)
- [arXiv 2601.12904: FusionRAG](https://arxiv.org/pdf/2601.12904v1)
- [arXiv 2603.20218: Chunk-level KV cache reuse study](https://arxiv.org/html/2603.20218v1)
- [arXiv 2609.10266: KVShareArena](https://arxiv.org/html/2609.10266v1)
- [arXiv 2412.15605: Cache-Augmented Generation](https://arxiv.org/abs/2412.15605)
- [arXiv 2604.23467: Hybrid JIT-CUDA Graph optimization](https://arxiv.org/abs/2604.23467)
- [arXiv 2609.21058: How much of a real workload can LLM-generated kernels reach?](https://arxiv.org/html/2609.21058)
