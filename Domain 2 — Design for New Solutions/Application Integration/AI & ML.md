# AWS AI/ML Services: SageMaker, Comprehend, Rekognition, Textract, Transcribe, Bedrock

## 1. Core Concepts & Theory

---

### Amazon SageMaker

**What it is:** Fully managed service to build, train, and deploy machine learning models at scale. Covers the entire ML lifecycle.

#### Key Components

**SageMaker Studio:**
- Integrated IDE for ML development
- Notebooks, experiments, model registry, pipelines in one UI
- Collaborative (share notebooks, experiments)

**SageMaker Training:**
- Managed training infrastructure (provisions instances, runs training, shuts down)
- Built-in algorithms: XGBoost, Linear Learner, K-Means, BlazingText, Image Classification, Object Detection, etc.
- Bring your own: Custom Docker containers, scripts (TensorFlow, PyTorch, MXNet, Hugging Face)
- **Spot Training:** Up to 90% cost savings using Spot Instances (with checkpointing)
- **Distributed training:** Data parallelism and model parallelism for large models
- **Managed Warm Pools:** Keep instances warm between training jobs (reduce startup time)

**SageMaker Inference:**
- **Real-time endpoints:** Persistent, low-latency (auto-scaling)
- **Serverless inference:** On-demand, scales to zero (cold start latency)
- **Batch transform:** Process large datasets offline
- **Asynchronous inference:** Queue requests, process long-running inferences (up to 1 hour)
- **Multi-model endpoints:** Host multiple models on one endpoint (cost savings)
- **Multi-container endpoints:** Run inference pipeline (pre-process → model → post-process)
- **Shadow testing:** Compare new model vs. production model with live traffic

**SageMaker Pipelines:**
- CI/CD for ML workflows
- Define DAG of steps: Processing, Training, Evaluation, Registration, Deployment
- Integrates with Model Registry for versioning and approval

**SageMaker Model Registry:**
- Central catalog of trained models
- Version tracking, approval workflows (Pending → Approved → Rejected)
- Model lineage and metadata

**SageMaker Feature Store:**
- Centralized store for ML features
- **Online store:** Low-latency reads for real-time inference
- **Offline store:** S3-based for batch training
- Feature versioning, sharing across teams

**SageMaker Ground Truth:**
- Data labeling service (human + ML-assisted)
- Active learning: Model labels easy examples, humans label hard ones
- Supports: Image, text, video, 3D point cloud labeling
- **Ground Truth Plus:** Fully managed labeling workforce

**SageMaker Data Wrangler:**
- Visual data preparation (import, transform, analyze, export)
- 300+ built-in transformations
- Export to: Processing job, Pipeline, Feature Store, notebook

**SageMaker Clarify:**
- Bias detection in data and models
- Model explainability (SHAP values)
- Monitoring for bias drift in production

**SageMaker Model Monitor:**
- Detect data drift, model quality drift, bias drift
- Baseline → continuous monitoring → CloudWatch alerts
- Types: Data Quality, Model Quality, Bias Drift, Feature Attribution Drift

**SageMaker Canvas:**
- No-code ML for business analysts
- Point-and-click model building
- Auto ML under the hood

**SageMaker JumpStart:**
- Pre-trained models and solutions (Foundation Models, ML solutions)
- One-click deployment of popular models (Llama, Stable Diffusion, etc.)
- Fine-tuning support

**SageMaker Neo:**
- Compile models for edge deployment
- Optimize for specific hardware (ARM, Intel, NVIDIA, etc.)
- Deploy to edge via IoT Greengrass

**SageMaker Edge Manager:**
- Manage ML models on edge devices (fleets)
- Model packaging, deployment, monitoring on edge

#### Key Quotas & Limits
- Max training job duration: 28 days (default 5 days)
- Real-time endpoint payload: 6 MB
- Async inference payload: 1 GB (via S3)
- Batch transform: No payload limit (S3-based)
- Serverless inference: 6 MB payload, max concurrency per endpoint configurable
- Max model size for real-time: 500 GB (with S3 model artifacts)

---

### Amazon Comprehend

**What it is:** Fully managed NLP (Natural Language Processing) service that extracts insights from text without ML expertise.

#### Capabilities
- **Entity Recognition:** People, places, dates, quantities, organizations
- **Sentiment Analysis:** Positive, negative, neutral, mixed
- **Key Phrase Extraction:** Important phrases in text
- **Language Detection:** Identify language (100+ languages)
- **Syntax Analysis:** Parts of speech (noun, verb, adjective)
- **Topic Modeling:** Discover topics across a document collection (async batch)
- **PII Detection & Redaction:** Identify and mask personally identifiable information
- **Custom Entity Recognition:** Train custom models for domain-specific entities
- **Custom Classification:** Train custom text classifiers

#### Comprehend Medical
- Specialized for medical/healthcare text
- Extracts: Medical conditions, medications, dosages, procedures, anatomy
- Identifies relationships between medical entities
- HIPAA eligible

#### Key Features
- **Real-time (synchronous):** Single document analysis
- **Async (batch):** Large document collections (submitted as S3 jobs)
- **Custom models:** Train on your labeled data via Comprehend console/API
- **Flywheel:** Manage custom model lifecycle (training, evaluation, versioning)

#### Quotas
- Real-time: Up to 100 KB per document (UTF-8)
- Batch: Up to 5 MB per document
- Custom models: Up to 100,000 documents for training
- Languages: 12 for most features, 100+ for language detection

---

### Amazon Rekognition

**What it is:** Fully managed computer vision service for image and video analysis.

#### Capabilities
- **Object & Scene Detection:** Labels (thousands of categories)
- **Facial Analysis:** Age range, gender, emotions, attributes (glasses, beard)
- **Face Comparison:** Compare faces across images (similarity score)
- **Face Search (Collections):** Index faces, search against a collection (1:N matching)
- **Celebrity Recognition:** Identify famous people
- **Text in Image (OCR):** Detect and read text in images
- **Content Moderation:** Detect unsafe/inappropriate content
- **Custom Labels:** Train custom object/scene detection models with few images
- **PPE Detection:** Personal Protective Equipment (hard hats, face covers, gloves)

#### Rekognition Video
- Analyze stored videos (S3) or live streaming video (Kinesis Video Streams)
- Same capabilities as image plus: Person tracking, path tracking, activity detection
- **Streaming Video Events:** Real-time alerts on live streams (connected home use case)
- Asynchronous: Start job → SNS notification on completion → get results

#### Key Features
- **Face Collections:** Serverless face database (IndexFaces → SearchFaces)
- **Custom Labels:** Train with as few as 10 images per class
- **Confidence scores:** All detections include confidence (filter by threshold)

#### Quotas
- Image size: Max 15 MB (S3), 5 MB (inline bytes)
- Faces per collection: 20 million
- Face search: Returns up to 4,096 faces
- Video: Max 10 GB file size, 2 hours duration
- Custom Labels: Min 10 images per label for training

---

### Amazon Textract

**What it is:** Fully managed service that extracts text, handwriting, and structured data from scanned documents using ML.

#### Capabilities
- **Text Detection (DetectDocumentText):** Raw OCR — lines and words
- **Document Analysis (AnalyzeDocument):** Forms (key-value pairs), Tables, Signatures
- **Expense Analysis (AnalyzeExpense):** Receipts and invoices (vendor, line items, totals)
- **ID Analysis (AnalyzeID):** Driver's licenses, passports (structured extraction)
- **Lending Analysis:** Mortgage/loan document packages (classify + extract)
- **Layout Analysis:** Detect titles, headers, paragraphs, lists, page numbers

#### Textract vs Rekognition Text Detection
- **Rekognition:** Text in natural images (signs, license plates, product labels)
- **Textract:** Text in documents (forms, invoices, contracts, PDFs, scans)
- **Key rule:** Document → Textract. Real-world image → Rekognition

#### Key Features
- **Synchronous API:** Single-page documents (real-time)
- **Asynchronous API:** Multi-page documents (PDF up to 3,000 pages)
- **Queries:** Ask natural language questions about a document ("What is the patient's name?")
- **Adapters:** Custom adapters to improve extraction on specific document types
- **Human Review (A2I integration):** Route low-confidence extractions to human reviewers

#### Quotas
- Sync: Single page, max 10 MB
- Async: Up to 3,000 pages, max 500 MB
- PDF: Max 3,000 pages
- TIFF: Max 3,000 pages
- Supported formats: PDF, PNG, JPEG, TIFF

---

### Amazon Transcribe

**What it is:** Fully managed speech-to-text service that converts audio to text using ASR (Automatic Speech Recognition).

#### Capabilities
- **Batch Transcription:** Process audio files in S3
- **Real-time (Streaming) Transcription:** Live audio via WebSocket or HTTP/2
- **Custom Vocabulary:** Add domain-specific words (medical terms, brand names)
- **Custom Language Models:** Train on domain text for better accuracy
- **Automatic Language Identification:** Detect spoken language
- **Multi-language Identification:** Detect multiple languages in one audio
- **Speaker Diarization:** Identify who said what (multiple speakers)
- **Channel Identification:** Separate audio channels (e.g., agent vs. customer)
- **Vocabulary Filtering:** Mask or remove profanity/unwanted words
- **Content Redaction:** Remove PII from transcripts (SSN, credit card, etc.)
- **Subtitle Generation:** Generate SRT/VTT files for video captioning

#### Transcribe Medical
- Medical-specific speech recognition
- HIPAA eligible
- Specialized vocabularies for clinical conversations

#### Transcribe Call Analytics
- Purpose-built for call center audio
- Sentiment analysis per turn
- Call summarization
- Issue detection, action items
- Integrates with Contact Lens for Amazon Connect

#### Key Features
- **Supported formats:** MP3, MP4, WAV, FLAC, OGG, AMR, WebM
- **Languages:** 100+ languages and dialects
- **Streaming:** Real-time with partial results (progressive output)
- **Toxicity Detection:** Identify toxic content in speech

#### Quotas
- Audio file size: 2 GB max
- Audio duration: 14,400 seconds (4 hours) batch
- Streaming: Unlimited duration
- Custom vocabulary: 50,000 entries
- Concurrent batch jobs: 250

---

### Amazon Bedrock

**What it is:** Fully managed service for building generative AI applications using foundation models (FMs) from multiple providers (Anthropic, Meta, Amazon, Cohere, AI21, Stability AI, Mistral).

#### Key Concepts

**Foundation Models Available:**
- **Anthropic Claude:** Text generation, analysis, coding (Claude 3 Opus/Sonnet/Haiku, Claude 4)
- **Amazon Titan:** Text, embeddings, image generation, multimodal
- **Meta Llama:** Open-source LLMs (Llama 3.1, Llama 4)
- **Cohere:** Text generation, embeddings (Command, Embed)
- **AI21 Labs:** Text generation (Jamba)
- **Stability AI:** Image generation (Stable Diffusion)
- **Mistral AI:** Text generation (Mistral, Mixtral)

**Bedrock APIs:**
- **InvokeModel:** Single synchronous request
- **InvokeModelWithResponseStream:** Streaming response
- **Converse API:** Unified multi-turn conversation API (works across models)
- **Batch Inference:** Process large batches asynchronously

#### Bedrock Features

**Knowledge Bases:**
- RAG (Retrieval Augmented Generation) — fully managed
- Ingest data sources: S3, web crawl, Confluence, Salesforce, SharePoint
- Vector stores: OpenSearch Serverless, Aurora PostgreSQL, Pinecone, Redis, MongoDB
- Automatic chunking, embedding, and indexing
- Query → retrieve relevant context → augment prompt → generate response

**Agents:**
- Autonomous multi-step task execution
- Define actions (API calls via Lambda or OpenAPI schemas)
- Agent plans, executes, and returns results
- Chain-of-thought reasoning with tool use
- **Return of Control:** Agent asks user for input during execution

**Guardrails:**
- Content filtering (hate, violence, sexual, profanity)
- Denied topic detection (block specific subjects)
- PII filtering (mask or block PII in inputs/outputs)
- Word/phrase filters
- Contextual grounding check (reduce hallucination)
- Apply to any Bedrock model call

**Model Customization:**
- **Fine-tuning:** Train model on your labeled data (Titan, Llama, Cohere)
- **Continued Pre-training:** Extend model knowledge with unlabeled domain data
- Custom models stored securely (not shared with other customers)
- Training data in S3

**Provisioned Throughput:**
- Reserve model capacity for guaranteed performance
- Consistent latency under high load
- Required for custom/fine-tuned models in production
- Commitment terms: 1-month or 6-month

**Model Evaluation:**
- Automatic evaluation (built-in metrics: accuracy, robustness, toxicity)
- Human evaluation (custom workforce)
- Compare multiple models side-by-side

**Bedrock Marketplace:**
- Access additional models beyond base offerings
- Third-party and specialized models

#### Quotas
- Input token limit: Model-dependent (Claude 4: 200K context)
- Output token limit: Model-dependent
- Request payload: 20 MB (InvokeModel)
- Guardrails: Applied per-request
- Knowledge Base: Up to 10,000 data sources per knowledge base
- Agents: Up to 12 action groups per agent

---

## 2. Design Patterns & Best Practices

### When to Use What

| Scenario | Use |
|----------|-----|
| Build/train/deploy custom ML models | SageMaker |
| Text analysis (sentiment, entities, PII) | Comprehend |
| Image/video analysis (faces, objects, moderation) | Rekognition |
| Extract data from documents (forms, tables, invoices) | Textract |
| Speech-to-text (audio/video transcription) | Transcribe |
| Generative AI (text generation, summarization, chat) | Bedrock |
| RAG (search + generate answers from your data) | Bedrock Knowledge Bases |
| Autonomous AI agents with tool use | Bedrock Agents |
| No-code ML for business users | SageMaker Canvas |
| Content moderation (images) | Rekognition |
| Content moderation (text, gen-AI output) | Bedrock Guardrails or Comprehend |
| Document processing pipeline | Textract + Comprehend + A2I |
| Call center analytics | Transcribe Call Analytics + Comprehend |
| Edge ML deployment | SageMaker Neo + Edge Manager |
| Face search / identity verification | Rekognition Face Collections |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Training a custom NER model when standard entities suffice | Unnecessary complexity and cost | Comprehend built-in entity recognition |
| Using Rekognition for document OCR | Rekognition is for natural images, not documents | Textract |
| Building custom speech-to-text on SageMaker | Reinventing the wheel, massive effort | Transcribe (+ Custom Vocabulary for domain terms) |
| Using SageMaker for simple text classification | Over-engineering for a managed service task | Comprehend Custom Classification |
| Fine-tuning a foundation model when RAG works | Fine-tuning is expensive and slow | Bedrock Knowledge Bases (RAG) |
| Using Bedrock for structured data predictions | LLMs aren't for tabular ML (regression/classification) | SageMaker built-in algorithms |
| Deploying a model 24/7 for occasional inference | Paying for idle compute | SageMaker Serverless Inference or Batch Transform |
| Processing documents page-by-page synchronously | Slow, hits rate limits | Textract async API for multi-page docs |

### Well-Architected Framework Alignment

**Reliability:**
- SageMaker endpoints: Multi-AZ deployment, auto-scaling
- All AI services: Regional, multi-AZ by default
- Bedrock: Service-managed infrastructure, automatic failover
- Async patterns: S3 input → process → S3 output (durable pipeline)

**Security:**
- SageMaker: VPC isolation for training/inference, KMS encryption, IAM roles per component
- Bedrock: Data not used to train base models, VPC endpoints, encryption at rest/transit
- All services: IAM policies, CloudTrail logging, VPC endpoints available
- Comprehend/Textract: Process PII data securely (don't log sensitive content)

**Performance:**
- SageMaker: GPU instances for training, auto-scaling for inference, multi-model endpoints
- Bedrock: Provisioned Throughput for consistent latency, streaming for faster perceived response
- Comprehend: Batch mode for large document collections
- Rekognition: Face collections for fast 1:N search

**Cost:**
- SageMaker: Spot Training (90% savings), Serverless Inference (pay-per-use), multi-model endpoints
- Bedrock: On-demand pricing (pay per token), batch inference for bulk (50% cheaper)
- Managed services (Comprehend, Rekognition, Textract, Transcribe): Pay per API call, no infrastructure

**Operational Excellence:**
- SageMaker Model Monitor: Automated drift detection
- SageMaker Pipelines: Reproducible ML workflows
- Bedrock Model Evaluation: Compare models before production
- CloudWatch integration across all services

### Integration Patterns

```
┌────────────────────────────────────────────────────────────────────┐
│                  Common AI/ML Architecture Patterns                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Document Processing Pipeline:                                       │
│  S3 (upload) → Lambda → Textract → Comprehend → DynamoDB/OpenSearch │
│                                                                      │
│  Media Analysis:                                                     │
│  S3 (video) → Rekognition Video → SNS → Lambda → DynamoDB          │
│  Kinesis Video → Rekognition Streaming → Lambda → Alert             │
│                                                                      │
│  Call Center Analytics:                                               │
│  Audio → Transcribe → Comprehend (sentiment) → QuickSight           │
│                                                                      │
│  RAG Application:                                                     │
│  S3 docs → Bedrock Knowledge Base (embed + index) →                  │
│  User query → Retrieve → Augment → Bedrock model → Response         │
│                                                                      │
│  ML Pipeline:                                                         │
│  S3 → SageMaker Processing → Training → Model Registry →            │
│  Approval → SageMaker Endpoint (auto-scaling)                        │
│                                                                      │
│  Content Moderation:                                                  │
│  Upload → Rekognition (image) + Comprehend (text) →                  │
│  Step Functions → Approve/Reject → Notify                            │
│                                                                      │
│  Intelligent Search:                                                  │
│  Documents → Textract + Comprehend → OpenSearch/Kendra → Bedrock    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Security & Compliance

### SageMaker Security

**Network Isolation:**
- Training jobs and endpoints can run in VPC (private subnets)
- `EnableNetworkIsolation`: Prevent containers from making outbound network calls
- VPC endpoints for SageMaker API, runtime, and Studio
- Inter-container encryption for distributed training

**Data Encryption:**
- At-rest: S3 (SSE-S3, SSE-KMS), EBS volumes (KMS), notebook instances (KMS)
- In-transit: TLS 1.2 for all API calls, inter-container encryption (optional)
- Model artifacts encrypted in S3

**IAM:**
- Execution roles: Separate roles for training, inference, pipelines
- Resource-based policies for endpoints
- Condition keys: `sagemaker:VpcSubnets`, `sagemaker:InstanceTypes`

**Governance:**
- Model Cards: Document model details, intended use, limitations
- Model Dashboard: Central view of all models and their status
- Audit: CloudTrail logs all SageMaker API calls

### Bedrock Security

**Data Privacy:**
- **Customer data is NOT used to train or improve base foundation models**
- Inputs and outputs are not stored by AWS (unless you configure logging)
- Model customization data remains in your account (S3)
- Fine-tuned models are private to your account

**Encryption:**
- Data at rest: KMS encryption for model customization, knowledge bases
- Data in transit: TLS 1.2
- Customer-managed KMS keys supported

**Access Control:**
- IAM policies: Control model access per user/role
- Model access management: Explicitly enable models before use (per-account, per-region)
- VPC endpoints: Access Bedrock without internet

**Guardrails (Content Safety):**
- Input/output filtering
- PII detection and masking
- Denied topics
- Word filters
- Apply across all model invocations

**Logging & Monitoring:**
- Model invocation logging: Log inputs/outputs to S3 or CloudWatch
- CloudTrail: API-level logging
- CloudWatch metrics: Invocation count, latency, errors, token usage

### Managed AI Services Security (Comprehend, Rekognition, Textract, Transcribe)

**Common Security Features:**
- All support VPC endpoints (no internet needed)
- All support KMS encryption for output
- All log to CloudTrail
- All are HIPAA eligible (Comprehend Medical, Transcribe Medical)
- Customer data processed transiently (not stored unless you configure output)

**PII Handling:**
- Comprehend: PII detection and redaction built-in
- Transcribe: Content redaction (PII removal from transcripts)
- Textract: May extract PII from documents — handle downstream
- Bedrock Guardrails: PII filtering on LLM inputs/outputs

**Cross-Account Access:**
- SageMaker endpoints: Resource policy for cross-account invocation
- Bedrock: Cross-account model access not native — use assume-role pattern
- S3 buckets shared cross-account for training data (bucket policies)

---

## 4. Cost Optimization

### SageMaker Pricing

**Training:**
- Per-instance-hour (varies by instance type: ml.m5.large to ml.p4d.24xlarge)
- **Spot Training:** Up to 90% savings (recommended for non-critical training)
- **Managed Warm Pools:** Reduce startup time, pay for warm time

**Inference:**
- Real-time endpoints: Per-instance-hour (always running)
- Serverless: Per-second of compute + per-request
- Batch Transform: Per-instance-hour (only while processing)
- Async: Per-instance-hour (can scale to zero)

**Cost Optimization:**
- **Spot Training** for all fault-tolerant training (checkpointing enabled)
- **Serverless Inference** for infrequent/unpredictable traffic
- **Multi-model endpoints** to host many models on one instance
- **Auto-scaling** endpoints based on invocation metrics
- **Batch Transform** instead of real-time for offline processing
- **Right-size instances**: Start small, scale up based on metrics
- **SageMaker Savings Plans**: Up to 64% savings with 1-year or 3-year commitment

### Bedrock Pricing

**On-Demand:**
- Per input token + per output token (varies by model)
- Claude Sonnet: ~$3/million input tokens, ~$15/million output tokens
- Amazon Titan Text: ~$0.30/million input tokens, ~$0.40/million output tokens

**Provisioned Throughput:**
- Fixed hourly rate for reserved capacity
- Required for custom/fine-tuned models
- 1-month or 6-month commitment options

**Batch Inference:**
- 50% discount vs on-demand
- Results within 24 hours

**Knowledge Bases:**
- Storage (vector DB): Varies by provider (OpenSearch Serverless charges separately)
- Embedding generation: Per-token pricing
- Retrieval: Per-query

**Cost Optimization:**
- Use smaller models when possible (Haiku for simple tasks, Sonnet for complex)
- Batch inference for non-real-time workloads (50% savings)
- Cache common queries at application layer
- Prompt engineering to reduce token usage
- Knowledge Bases: Chunk documents appropriately (not too small = more retrievals)
- Guardrails run before model → block bad requests early (save model invocation cost)

### Managed AI Services Pricing

| Service | Pricing Model |
|---------|--------------|
| Comprehend | Per-unit (100 characters) or per-document (custom models) |
| Rekognition | Per-image ($0.001–$0.004) or per-minute (video) |
| Textract | Per-page ($0.0015 text, $0.05 forms/tables) |
| Transcribe | Per-second of audio ($0.00024/sec standard) |

**Cost Optimization Tips:**
- Comprehend: Use batch mode for large collections (cheaper per-unit)
- Rekognition: Use confidence threshold to reduce downstream processing
- Textract: Only call features you need (DetectText is cheaper than AnalyzeDocument)
- Transcribe: Use batch (cheaper than streaming) when real-time isn't required
- All: Process in bulk rather than individual API calls where possible

### Cost Traps
- SageMaker notebook instances left running 24/7 (use auto-stop)
- Real-time endpoints with no auto-scaling during off-hours
- Bedrock: Verbose prompts with unnecessary context (token waste)
- Provisioned Throughput unused (committed but not utilized)
- Using expensive models (Claude Opus) for simple tasks (use Haiku)
- SageMaker Development Endpoints left idle
- Not using Spot for training (easy 90% savings left on table)
- Running Rekognition on every video frame instead of sampling

---

## 5. High Availability, Disaster Recovery & Resilience

### SageMaker HA

**Training:**
- Training jobs are NOT HA (if instance fails mid-training, job fails)
- **Mitigation:** Enable checkpointing to S3, use Managed Spot with checkpoints
- Job retries: Configure in SageMaker Pipelines

**Inference:**
- Real-time endpoints: **Multi-AZ by default** (instances spread across AZs)
- Auto-scaling: Target tracking on `InvocationsPerInstance`
- **Endpoint update:** Blue/green deployment (new instances provisioned before old removed)
- **Inference components:** Share endpoint capacity across multiple models with guaranteed resources

**Multi-Region:**
- Deploy endpoints in multiple regions + Route 53 failover
- Model artifacts in S3 (replicate with CRR)
- No native multi-region replication for endpoints

### Bedrock HA
- **Fully managed, multi-AZ by default**
- **Regional service** — deploy in multiple regions for DR
- Foundation models always available (managed by AWS)
- Knowledge Bases: Backed by managed vector stores (OpenSearch Serverless = multi-AZ)
- **Fallback pattern:** If primary model throttled → invoke alternative model (e.g., Claude → Titan)

### Managed AI Services HA
- All managed services (Comprehend, Rekognition, Textract, Transcribe) are:
  - Multi-AZ by default
  - Serverless (no infrastructure to manage)
  - Regional (replicate workload in second region for DR)
- **Async pattern for resilience:** S3 input → Service → S3 output (durable, retryable)

### Resilience Patterns

**ML Inference Resilience:**
```
Route 53 (latency-based) →
  Region A: API Gateway → SageMaker Endpoint (auto-scaling, multi-AZ)
  Region B: API Gateway → SageMaker Endpoint (warm standby)
```

**Bedrock Resilience:**
```
Application → Try primary model (Claude Sonnet) →
  If throttled/error → Fallback model (Titan/alternative) →
  If still failing → Return cached response / graceful degradation
```

**Document Processing Resilience:**
```
S3 Upload → SQS (buffer) → Lambda → Textract (async) →
  Success → Process results
  Failure → DLQ → Retry / Alert
```

### RPO/RTO

| Service | RPO | RTO | Strategy |
|---------|-----|-----|----------|
| SageMaker Endpoint | Zero (stateless inference) | Minutes (auto-scaling, new instances) | Multi-AZ + auto-scaling |
| SageMaker Training | Last checkpoint | Minutes–hours (restart from checkpoint) | Checkpointing to S3 |
| Bedrock | Zero (managed service) | Seconds (retry/failover to another model) | Multi-model fallback |
| Managed AI Services | Zero (stateless) | Seconds (retry) | Retry + multi-region |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company needs to extract key-value pairs (like "Name: John Smith", "Date: 2024-03-15") from scanned PDF forms uploaded to S3. Which service should they use?

**A:** Amazon Textract with `AnalyzeDocument` (Forms feature). Textract detects form fields as key-value pairs from documents. For multi-page PDFs, use the asynchronous API.

---

**Q2:** A social media company needs to automatically detect and remove inappropriate images uploaded by users. Which service?

**A:** Amazon Rekognition Content Moderation (`DetectModerationLabels`). It identifies unsafe content categories (nudity, violence, drugs, etc.) with confidence scores. Set a threshold and auto-reject.

---

**Q3:** A healthcare company wants to extract medical conditions and medications from clinical notes. Which service?

**A:** Amazon Comprehend Medical. It's specialized for medical text and extracts conditions, medications, dosages, procedures, and relationships between them. It's HIPAA eligible.

---

**Q4:** A company wants to build a chatbot that answers questions using information from their internal documentation stored in S3. Which approach requires the least development effort?

**A:** Amazon Bedrock Knowledge Bases (RAG). Point it at the S3 documentation, it automatically chunks, embeds, indexes, and at query time retrieves relevant context and generates answers. No custom code for the retrieval pipeline.

---

**Q5:** A company needs to transcribe customer support calls and identify sentiment per speaker turn. Which services?

**A:** Amazon Transcribe Call Analytics. It provides transcription, speaker diarization, and per-turn sentiment analysis in a single service. Alternatively: Transcribe (with speaker diarization) + Comprehend (sentiment analysis).

---

**Q6:** A company wants to deploy an ML model for real-time inference but traffic is unpredictable — some hours have zero requests, others have thousands. Which SageMaker inference option minimizes cost?

**A:** SageMaker Serverless Inference. It scales to zero when no traffic (no cost) and auto-scales up for demand. Trade-off: cold start latency on first request after idle.

---

**Q7:** A company is building a generative AI application and wants to prevent the model from generating content about competitors or revealing PII in responses. Which Bedrock feature?

**A:** Bedrock Guardrails. Configure denied topics (competitor names/discussions), PII filters (detect and mask), and content filters. Guardrails are applied to both input and output of model invocations.

---

### Tricky / Scenario-Based Questions

**Q1:** A company fine-tunes a foundation model on Bedrock using proprietary training data. They're concerned that their data might be used to improve the base model or become accessible to other customers. Is this a valid concern?

**A:** **No.** AWS Bedrock guarantees that customer data (inputs, outputs, fine-tuning data) is **never used to train or improve base foundation models** and is never accessible to other customers. Fine-tuned model weights are private to the customer's account. Additionally, customers can enable encryption with their own KMS keys. **Key fact for exam:** "Your data is your data" — Bedrock does not use customer data for model improvement. This is a critical security/compliance differentiator.

---

**Q2:** A company uses SageMaker real-time endpoints for inference. They deploy a new model version, but during the update, they experience 5 minutes of increased latency. Users complain. How do they achieve zero-downtime deployments?

**A:** SageMaker already uses **blue/green deployment** for endpoint updates (new instances are provisioned before old ones are terminated). The latency issue is likely because the new model is loading (container startup + model download from S3). **Fixes:** (1) Use **Inference Components** with managed instance pools. (2) Use **Shadow Testing** to pre-warm the new model variant. (3) Use **Auto-scaling with minimum instances** to ensure capacity. (4) **Pre-warm** by sending test traffic to the new variant before shifting production. **Key concept:** SageMaker endpoint updates ARE blue/green by default, but model loading time can still cause perceived latency.

---

**Q3:** A company processes millions of images daily with Rekognition. They want to identify when a previously banned user's face appears in new uploads. What's the most efficient architecture?

**A:** Use Rekognition **Face Collections**. (1) Index banned users' faces into a collection (`IndexFaces`). (2) For each new upload, call `SearchFacesByImage` against the collection. This returns matches with similarity scores. Set a threshold (e.g., 99%) for positive identification. **Why not CompareFaces:** CompareFaces is 1:1 (source vs target). With millions of banned users, you'd need millions of comparisons. Face Collections does 1:N search efficiently (indexed). **Key distinction:** `CompareFaces` = 1:1. `SearchFacesByImage` / `SearchFaces` = 1:N (collection-based).

---

**Q4:** A company uses Amazon Textract to process invoices. For 5% of documents, Textract returns low-confidence results (< 80%). The company needs human review for these cases but wants to minimize custom development. What should they use?

**A:** **Amazon Augmented AI (A2I)** integrated with Textract. A2I allows you to define conditions (confidence < 80%), and automatically routes those documents to a human review workflow (private workforce, AWS Marketplace workforce, or Mechanical Turk). Results feed back into the pipeline. **Key integration:** Textract natively integrates with A2I — you configure a human review workflow and conditions in the API call. No custom orchestration needed.

---

**Q5:** A company wants to use Bedrock for their customer-facing chatbot, but they need to ensure the model can call their order management API to look up order status. The model should autonomously decide when to call the API. What Bedrock feature should they use?

**A:** **Bedrock Agents.** Define an action group that describes the order lookup API (OpenAPI schema or Lambda function). The agent uses chain-of-thought reasoning to decide when to call the API, formats the request, processes the response, and generates a natural language answer. **Wrong answer:** "Knowledge Bases" — KB is for document retrieval, not API calls. **Wrong answer:** "Custom prompt with API instructions" — the model can't actually make API calls without an execution framework. Agents provide that execution framework.

---

**Q6:** A company trains ML models on SageMaker using sensitive healthcare data. Compliance requires that training instances cannot access the internet and data cannot leave the VPC. How do they configure this?

**A:** (1) Run training job in a **VPC** (specify subnets and security groups). (2) Enable **`EnableNetworkIsolation: true`** — this prevents the container from making ANY outbound network call. (3) Use **VPC endpoints** for SageMaker API, S3, and ECR (to pull container images). (4) Encrypt data with customer-managed **KMS keys**. (5) Enable **inter-container traffic encryption** for distributed training. **Key setting:** `EnableNetworkIsolation` is the critical parameter — it completely air-gaps the training container from the network.

---

**Q7:** A company uses Comprehend for sentiment analysis on customer reviews. They now need to classify reviews into custom categories (e.g., "shipping issue", "product quality", "customer service"). Standard Comprehend doesn't have these categories. What should they do?

**A:** Use **Comprehend Custom Classification**. Train a custom multi-class classifier on labeled examples of their specific categories. Upload training data (labeled text file with categories), train the model, then create a real-time endpoint or run batch jobs. **Wrong answer:** "Use SageMaker to train a custom model" — possible but over-engineering when Comprehend Custom handles text classification natively with minimal ML expertise. **Wrong answer:** "Use Bedrock" — could work for zero-shot classification but Comprehend Custom is purpose-built, cheaper for high-volume, and deterministic.

---

### Common Exam Traps & Pitfalls

1. **Textract vs Rekognition for text:** Textract = documents (forms, PDFs, invoices). Rekognition = text in images (signs, license plates). The exam will try to confuse these.

2. **Bedrock data privacy:** Customer data is NEVER used to train base models. This is a key compliance answer on the exam.

3. **SageMaker Spot Training requires checkpointing.** Without checkpoints, if the instance is reclaimed, all training progress is lost. Always mention checkpointing with Spot.

4. **Comprehend vs Bedrock for text tasks:** Comprehend = deterministic NLP (entities, sentiment, classification). Bedrock = generative (summarization, Q&A, creative text). Different use cases.

5. **RAG (Knowledge Bases) vs Fine-tuning:** RAG = add current/proprietary data without retraining (dynamic, cheaper). Fine-tuning = change model behavior/style (expensive, static). Exam loves this distinction.

6. **SageMaker inference options confusion:**
   - Real-time = persistent, low-latency, always-on
   - Serverless = scales to zero, cold starts
   - Async = long-running (up to 1 hour), queued
   - Batch = offline, large datasets

7. **Rekognition Face Collections are NOT face recognition models.** They're indexed face databases. You don't train anything — you index faces and search.

8. **Comprehend Medical ≠ regular Comprehend for healthcare.** They're separate APIs. Comprehend Medical is specifically for clinical/medical text with specialized entity types.

9. **SageMaker Multi-Model Endpoints:** Load models dynamically per request (from S3). Good when you have thousands of similar models (one per customer). Not the same as multi-container (different models in pipeline).

10. **Bedrock Guardrails apply to both inputs AND outputs.** They filter before the model sees the prompt and after the model generates a response. Both directions matter.

11. **A2I (Augmented AI) integrates natively with Textract AND Rekognition.** For Comprehend or other services, you use custom A2I workflows.

12. **SageMaker Canvas is for business users (no-code).** If the question says "data scientists" or "ML engineers" → SageMaker Studio. If it says "business analysts, no ML experience" → Canvas.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| SageMaker real-time payload | 6 MB max |
| SageMaker async payload | 1 GB (via S3) |
| SageMaker training max duration | 28 days |
| SageMaker Spot savings | Up to 90% |
| Bedrock data privacy | Customer data NEVER used for base model training |
| Textract sync API | Single page only |
| Textract async API | Up to 3,000 pages |
| Textract vs Rekognition | Documents vs natural images |
| Comprehend document limit | 100 KB (real-time) |
| Rekognition face collection limit | 20 million faces |
| Rekognition image max | 15 MB (S3), 5 MB (inline) |
| Transcribe audio max | 4 hours (batch), unlimited (streaming) |
| Transcribe audio file max | 2 GB |
| Bedrock batch inference savings | 50% vs on-demand |
| RAG vs Fine-tuning | RAG = add data dynamically. Fine-tune = change model behavior |

### Key Differentiators

| Feature | SageMaker | Bedrock | Comprehend/Rekognition/Textract/Transcribe |
|---------|-----------|---------|-------------------------------------------|
| Customization | Full (build any model) | Fine-tune or RAG | Custom models (limited) or use as-is |
| Expertise needed | High (ML engineers) | Medium (developers) | Low (any developer) |
| Infrastructure | Managed instances | Fully serverless | Fully serverless |
| Use case | Custom ML models | Generative AI apps | Specific AI tasks (NLP, vision, docs, speech) |
| Pricing | Per instance-hour | Per token | Per API call/unit |

### Service Selection: "If the question says X, think Y"

| Question Keywords | Service |
|-------------------|---------|
| "Extract text from documents, forms, tables" | Textract |
| "Sentiment analysis", "entity recognition" | Comprehend |
| "Detect faces", "moderate images", "label objects" | Rekognition |
| "Speech to text", "transcribe audio" | Transcribe |
| "Generate text", "summarize", "chatbot" | Bedrock |
| "Answer questions from company documents" | Bedrock Knowledge Bases (RAG) |
| "Train custom ML model", "algorithm selection" | SageMaker |
| "Deploy model with auto-scaling" | SageMaker Endpoints |
| "No-code ML for business users" | SageMaker Canvas |
| "Medical text analysis" | Comprehend Medical |
| "Identify faces in large database" | Rekognition Face Collections |
| "Human review of low-confidence AI results" | Amazon A2I |
| "Prevent AI from generating harmful content" | Bedrock Guardrails |
| "AI agent that calls APIs autonomously" | Bedrock Agents |
| "Deploy ML on edge devices" | SageMaker Neo + Edge Manager |
| "Monitor model quality in production" | SageMaker Model Monitor |
| "Detect bias in ML model" | SageMaker Clarify |

### Decision Flowchart

```
Question mentions...                                    → Think...
──────────────────────────────────────────────────────────────────────
"extract from document/form/invoice/receipt"           → Textract
"text in natural image (sign, plate)"                 → Rekognition (DetectText)
"sentiment", "entities", "PII in text"                → Comprehend
"face search", "face matching", "content moderation"  → Rekognition
"transcribe audio", "speech-to-text"                  → Transcribe
"generative AI", "LLM", "foundation model"           → Bedrock
"RAG", "answer from documents"                        → Bedrock Knowledge Bases
"AI agent", "autonomous API calls"                    → Bedrock Agents
"content safety", "guardrails"                        → Bedrock Guardrails
"train custom model"                                  → SageMaker
"no-code ML"                                          → SageMaker Canvas
"deploy model at scale"                               → SageMaker Endpoints
"model monitoring", "data drift"                      → SageMaker Model Monitor
"bias detection"                                      → SageMaker Clarify
"medical NLP"                                         → Comprehend Medical
"call center transcription + analytics"               → Transcribe Call Analytics
"label training data"                                 → SageMaker Ground Truth
"edge ML deployment"                                  → SageMaker Neo
```

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company uses Bedrock to generate product descriptions. Sometimes the model hallucinates product features that don't exist. They have a product database with accurate specifications. How should they reduce hallucination?

**A:** Implement **RAG using Bedrock Knowledge Bases.** Index the product database (S3 or structured data) into a knowledge base. At query time, retrieve the specific product's specs and include them as context in the prompt. The model generates descriptions grounded in actual data. Additionally, enable **Bedrock Guardrails contextual grounding check** to verify the output is supported by the retrieved context. **Wrong answer:** "Fine-tune the model on product data" — fine-tuning teaches style/behavior but doesn't guarantee factual accuracy. RAG provides verifiable source data at inference time.

---

**Q2:** A company processes 10,000 documents daily with Textract. Each document is 20 pages. They use the synchronous API and experience throttling. How should they redesign?

**A:** Switch to the **asynchronous API** (StartDocumentAnalysis). Submit jobs referencing documents in S3, Textract processes them and sends SNS notification on completion. Architecture: S3 upload → Lambda → Start Textract async job → SNS → Lambda → Process results. **Additional optimization:** Use SQS to queue submissions and control concurrency, respecting Textract's rate limits. **Key rule:** Sync API = single page only. Multi-page documents REQUIRE async API. 10,000 × 20 pages = must use async + queue-based throttling.

---

**Q3:** A company wants to use Rekognition to verify user identity (selfie vs. ID photo) for account creation. They're concerned about bias in face comparison across different demographics. How should they address this?

**A:** (1) Use `CompareFaces` API with a high similarity threshold (e.g., 99%). (2) **Rekognition provides confidence scores per match** — set thresholds appropriately and test across demographics. (3) Implement a **human review (A2I)** for borderline cases (e.g., confidence 90-99%). (4) Monitor false acceptance/rejection rates per demographic in your own metrics. **Key awareness:** AWS has improved Rekognition's accuracy across demographics, but the exam expects you to know about bias concerns and mitigation (human review, threshold tuning, monitoring).

---

**Q4:** A company has a SageMaker endpoint serving a recommendation model. During Black Friday, inference latency spikes from 50ms to 2 seconds. Auto-scaling is enabled with target tracking on `InvocationsPerInstance`. What's wrong?

**A:** Auto-scaling is reactive — it takes **minutes to provision new instances** (download model, start container). By the time new instances are ready, the spike has caused latency issues. **Fixes:** (1) **Scheduled scaling**: Pre-scale before known events (Black Friday). (2) Set **minimum instance count** higher based on expected peak. (3) Use **Provisioned Concurrency** (for Serverless) or higher base capacity. (4) Monitor `OverheadLatency` and `ModelLatency` to identify bottleneck. (5) Use **Inference Components** with pre-warmed capacity. **Key concept:** Auto-scaling protects against gradual growth, NOT sudden spikes. Predictable peaks need scheduled scaling.

---

**Q5:** A company uses Comprehend to analyze customer feedback. They need to detect whether feedback mentions specific product features they've defined (e.g., "battery life", "screen quality", "shipping speed") — not just generic entities. Standard entity recognition doesn't detect these. What should they use?

**A:** **Comprehend Custom Entity Recognition.** Train a custom entity model with labeled examples of your domain-specific entities. Provide training data with annotated text showing where your custom entities appear. After training, the model detects your custom entities in new text. **Alternative for simpler cases:** Use Comprehend **Key Phrase Extraction** + post-processing rules. **Wrong answer:** "Use Bedrock to classify" — works but not deterministic, more expensive at scale. Custom Comprehend is purpose-built for this.

---

**Q6:** A company wants to build an AI assistant that can search their Confluence wiki, look up data in their CRM via API, and generate formatted reports. The assistant should handle multi-step tasks autonomously. What's the best architecture on AWS?

**A:** **Bedrock Agents** with multiple action groups: (1) Knowledge Base action (Confluence content indexed in Bedrock Knowledge Bases). (2) CRM action group (Lambda function that calls CRM API, defined via OpenAPI schema). (3) Report generation action (Lambda that formats and stores reports). The agent orchestrates: understands the request → retrieves wiki context → calls CRM for data → generates report. **Key capability:** Bedrock Agents handle multi-step planning, tool selection, and execution autonomously.

---

**Q7:** A company uses SageMaker for fraud detection. The model works well initially, but after 3 months, precision drops significantly. CloudWatch shows no infrastructure issues. What's likely happening and how do they detect it?

**A:** **Model drift** — the real-world data distribution has changed (new fraud patterns, seasonal changes, user behavior shifts). The model's training data no longer represents current reality. **Detection:** Use **SageMaker Model Monitor** with Data Quality and Model Quality monitoring. Set up a baseline from training data, then monitor inference data for statistical drift (KL divergence, KS test). When drift is detected → trigger alert → retrain model on recent data. **Key concept:** ML models degrade over time (data drift, concept drift). Model Monitor is the AWS answer for automated detection.

---

**Q8:** A company processes insurance claims. Each claim has a scanned form (Textract), a recorded call (Transcribe), and a written description (Comprehend). They want to combine insights from all three into a single decision-support system. What orchestration approach?

**A:** Use **Step Functions** to orchestrate the multi-modal processing pipeline: (1) Parallel state: Textract (extract form data) + Transcribe (transcribe call) + Comprehend (analyze description) — all run concurrently. (2) Wait for all to complete. (3) Lambda aggregates results. (4) Optional: Bedrock generates a summary/recommendation from combined insights. (5) If confidence is low → A2I for human review. **Architecture strength:** Step Functions handles the parallel fan-out, error handling per branch, and aggregation naturally.

---

**Q9:** A company fine-tunes a Bedrock model on their proprietary data. After fine-tuning, the model performs well on their specific task but gives poor responses to general questions it previously handled well. What happened?

**A:** **Catastrophic forgetting.** Fine-tuning on a narrow dataset can cause the model to lose its general capabilities. **Mitigations:** (1) Include diverse general examples in fine-tuning data (not just task-specific). (2) Use a lower learning rate / fewer epochs. (3) Instead of fine-tuning, consider **RAG** (Knowledge Bases) which doesn't alter the model. (4) Use **Continued Pre-training** first (broader domain knowledge) then fine-tune (specific task). **Key exam distinction:** RAG adds knowledge without changing the model. Fine-tuning changes the model itself (risk of forgetting). For most enterprise use cases, RAG is preferred.

---

**Q10:** A company uses Transcribe to transcribe medical dictations. Standard Transcribe misrecognizes medical terms like "acetaminophen" and "tachycardia." They've added these to a Custom Vocabulary but accuracy is still poor. What else can they do?

**A:** (1) Switch to **Transcribe Medical** — it's specifically trained on medical speech and vocabularies. (2) If already using Transcribe Medical, create a **Custom Language Model** trained on their specific medical domain text (dictation transcripts, medical literature). (3) Ensure the custom vocabulary includes pronunciation hints (IPA or SoundsLike). (4) Verify the audio quality and language/accent settings are correct. **Key distinction:** Custom Vocabulary = tell Transcribe about specific words. Custom Language Model = teach Transcribe your domain's language patterns. Transcribe Medical = pre-built medical ASR. Use all three together for best results.

---

## 9. Service Comparison Tables

### AI/ML Service Selection Matrix

| Dimension | SageMaker | Bedrock | Comprehend | Rekognition | Textract | Transcribe |
|-----------|-----------|---------|------------|-------------|----------|------------|
| **Primary Use** | Custom ML lifecycle | Generative AI apps | NLP (text analysis) | Computer vision | Document data extraction | Speech-to-text |
| **Expertise** | ML engineers | Developers | Any developer | Any developer | Any developer | Any developer |
| **Customization** | Unlimited (any model) | Fine-tune / RAG / Agents | Custom classifiers, entities | Custom Labels | Adapters, Queries | Custom Vocabulary, Language Models |
| **Infrastructure** | Managed instances | Serverless | Serverless | Serverless | Serverless | Serverless |
| **Pricing** | Per instance-hour | Per token | Per unit/API call | Per image/minute | Per page | Per second of audio |
| **HA Model** | Multi-AZ endpoints | Multi-AZ (managed) | Multi-AZ (managed) | Multi-AZ (managed) | Multi-AZ (managed) | Multi-AZ (managed) |
| **Exam Tip** | "train custom model", "deploy endpoint" | "generate", "summarize", "RAG", "agent" | "sentiment", "entities", "PII text" | "faces", "objects", "moderate images" | "forms", "tables", "invoices" | "audio to text", "call center" |

### RAG (Knowledge Bases) vs Fine-Tuning vs Prompt Engineering

| Dimension | RAG (Knowledge Bases) | Fine-Tuning | Prompt Engineering |
|-----------|----------------------|-------------|-------------------|
| **Use Case** | Add proprietary/current data | Change model behavior/style | Adapt output without training |
| **Data Freshness** | Real-time (index updates) | Static (training snapshot) | None (no data added) |
| **Cost** | Low (per-query retrieval) | High (training compute) | Zero (prompt only) |
| **Setup Time** | Hours (index documents) | Days (training + evaluation) | Minutes |
| **Model Change** | None (same base model) | Yes (creates new model) | None |
| **Risk** | Low (model unchanged) | Medium (catastrophic forgetting) | Low |
| **Best For** | "Answer from our docs", "current data" | "Write in our brand voice", "domain expert" | "Format output as JSON", "role-play" |
| **Exam Tip** | "proprietary data", "current information", "documents" | "change model behavior", "domain-specific responses" | "format", "instructions", "few-shot examples" |

### SageMaker Inference Options

| Dimension | Real-Time | Serverless | Async | Batch Transform |
|-----------|-----------|------------|-------|-----------------|
| **Latency** | Low (ms) | Variable (cold start) | Minutes (queued) | Offline |
| **Payload Max** | 6 MB | 6 MB | 1 GB (S3) | Unlimited (S3) |
| **Duration Max** | 60 seconds | 60 seconds | 1 hour | No limit |
| **Scales to Zero** | No | Yes | Yes (min instances = 0) | N/A (on-demand) |
| **Cost** | Highest (always on) | Pay-per-use | Per-instance (can scale down) | Per-instance (temp) |
| **Best For** | Production APIs, consistent traffic | Infrequent, unpredictable | Large payloads, long inference | Bulk offline processing |
| **Exam Tip** | "low-latency API" | "infrequent traffic", "cost-sensitive" | "large input", "video/audio" | "process dataset offline" |

### Textract vs Rekognition vs Comprehend for Text

| Dimension | Textract | Rekognition (DetectText) | Comprehend |
|-----------|----------|--------------------------|------------|
| **Input** | Documents (PDF, scanned images) | Natural images (photos) | Plain text (UTF-8) |
| **Extracts** | Text + structure (forms, tables) | Text in image (up to 100 words) | Entities, sentiment, key phrases |
| **Use Case** | Invoice processing, form extraction | License plate reader, sign detection | Customer feedback analysis |
| **Structured Output** | Key-value pairs, table cells, layout | Words + bounding boxes | Entity types + confidence scores |
| **Exam Tip** | "document", "form", "invoice", "PDF" | "text in photo", "sign", "real-world" | "sentiment", "entities", "PII in text" |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick Bedrock over SageMaker when:
- "generative AI", "text generation", "summarization"
- "chatbot", "conversational AI"
- "no ML expertise", "developer-friendly"
- "foundation model", "LLM"
- "minimal infrastructure management"
- "RAG", "answer questions from documents"
- **Exam signal:** Generative text/image + managed + no custom training

### Pick SageMaker over Bedrock when:
- "custom model", "proprietary algorithm"
- "train from scratch", "tabular data prediction"
- "specific ML framework (XGBoost, TensorFlow, PyTorch)"
- "full ML lifecycle", "experiment tracking"
- "regression", "classification", "clustering" on structured data
- "deploy model endpoint with auto-scaling"
- **Exam signal:** Custom ML + training + structured data + full lifecycle

### Pick Comprehend over Bedrock for text tasks when:
- "deterministic results" (same input → same output)
- "high volume, low cost" (Comprehend is cents per 1000 units)
- "standard NLP tasks" (sentiment, entities, key phrases)
- "PII detection at scale" (built-in, certified)
- "custom classification with labeled data"
- **Exam signal:** Standard NLP + scale + cost-sensitivity + determinism

### Pick Bedrock over Comprehend when:
- "zero-shot classification" (no training data available)
- "summarization", "generation", "creative text"
- "complex reasoning about text"
- "conversational context needed"
- **Exam signal:** Generation + reasoning + no labeled data

### Pick Textract over Rekognition when:
- "document", "PDF", "form", "invoice", "receipt"
- "key-value pairs", "table extraction"
- "structured data from unstructured documents"
- "multi-page document processing"
- **Exam signal:** Document processing (even if scanned image)

### Pick Rekognition over Textract when:
- "natural image" (not a document)
- "faces", "objects", "scenes", "celebrities"
- "content moderation"
- "video analysis"
- "text on signs, license plates, product labels"
- **Exam signal:** Photo/video analysis, face operations, moderation

### Pick RAG (Knowledge Bases) over Fine-Tuning when:
- "proprietary data that changes frequently"
- "current/up-to-date information needed"
- "minimize cost and effort"
- "don't want to alter the base model"
- "verifiable sources" (citations)
- **Exam signal:** Dynamic data + cost + source attribution

### Pick Fine-Tuning over RAG when:
- "change model's writing style or tone"
- "model should behave as domain expert" (not just retrieve facts)
- "reduce inference cost" (shorter prompts after fine-tuning)
- "specialized task format" (always output specific structure)
- **Exam signal:** Behavior change + style + static domain knowledge

### Pick SageMaker Serverless over Real-Time Endpoints when:
- "unpredictable/infrequent traffic"
- "minimize cost", "scale to zero"
- "cold start latency acceptable"
- "dev/test environments"
- **Exam signal:** Infrequent + cost + not latency-critical

### Pick SageMaker Batch Transform over Real-Time when:
- "process entire dataset", "offline scoring"
- "no real-time requirement"
- "large-scale prediction job"
- "cost-sensitive bulk processing"
- **Exam signal:** Bulk + offline + dataset processing

---

## 11. "Gotcha" Differences the Exam Tests

### Textract Gotchas
1. **Sync API = single page ONLY.** Multi-page documents MUST use async. If you see "100-page PDF + Textract sync" → wrong.
2. **Textract ≠ OCR.** It does OCR PLUS structure extraction (forms, tables, layout). Don't confuse with basic OCR tools.
3. **Textract Queries let you ask questions about documents** ("What is the total amount?") — but it's extracting answers from the document, NOT generating them (not LLM).
4. **AnalyzeDocument is more expensive than DetectDocumentText.** Only use AnalyzeDocument when you need forms/tables/signatures — don't over-call.
5. **Textract outputs Block objects with relationships.** Understanding parent-child Block structure is needed to reconstruct tables/forms programmatically.

### Rekognition Gotchas
6. **Face Collections are serverless indexed databases, not models.** You don't "train" them — you index faces and search. They scale to 20 million faces.
7. **CompareFaces = 1:1 comparison.** SearchFacesByImage = 1:N (collection). Don't confuse these for the exam.
8. **Rekognition Video is ASYNC** (StartFaceDetection → SNS notification → GetFaceDetection). Not real-time for stored video.
9. **Custom Labels requires minimum 10 images per label.** It's for specialized objects not in the standard label set.
10. **Content Moderation returns categories and confidence scores.** You set the threshold — Rekognition doesn't auto-reject anything.

### Comprehend Gotchas
11. **Comprehend ≠ Translate.** Comprehend analyzes text (NLP). Translate converts languages. They complement each other.
12. **Comprehend Medical is a SEPARATE API** (not a feature flag on regular Comprehend). Different endpoints, different pricing.
13. **Custom models in Comprehend require your own labeled data.** Unlike Bedrock zero-shot, you need training examples.
14. **Topic Modeling is batch-only.** You can't do real-time topic modeling — it requires analyzing a corpus of documents together.
15. **PII detection and redaction are separate API calls.** DetectPiiEntities (identifies) vs ContainsPiiEntities (boolean check) vs CreateRedactedCopy.

### SageMaker Gotchas
16. **Real-time endpoint payload max = 6 MB.** For larger payloads → Async Inference (up to 1 GB via S3). Batch Transform for unlimited.
17. **SageMaker Spot Training can be interrupted.** Without checkpointing to S3, all training progress is lost when Spot is reclaimed.
18. **Multi-Model Endpoints load models on-demand from S3.** First call to a model has higher latency (model loading). Frequently called models stay cached.
19. **SageMaker endpoints are NOT serverless by default.** You must choose "Serverless" explicitly. Default = real-time (always-on instances).
20. **Model Monitor baselines must be created from training data.** You can't monitor drift without establishing what "normal" looks like first.
21. **EnableNetworkIsolation = no internet AND no inter-container communication.** For distributed training in isolated mode, you need a special setup.

### Bedrock Gotchas
22. **You must explicitly request model access in each region.** Models are not enabled by default — you enable them per-account, per-region.
23. **Bedrock Knowledge Bases need a vector store (separate cost).** OpenSearch Serverless, Aurora PostgreSQL, etc. are billed separately.
24. **Fine-tuned models require Provisioned Throughput to deploy.** You can't use on-demand pricing for custom models.
25. **Bedrock Guardrails are PER-REQUEST overhead.** They add latency (content filtering takes time). For latency-critical apps, balance safety vs. speed.
26. **Bedrock Agents have token limits per turn.** Very long conversations may exceed context. Use conversation summarization.
27. **Knowledge Base chunking strategy matters.** Too small = fragmented context. Too large = irrelevant noise. Default chunking works for most cases but custom chunking available for optimization.

### Cross-Service Gotchas
28. **A2I integrates natively with Textract and Rekognition only.** For Comprehend or Transcribe output, you need custom A2I task types.
29. **Transcribe + Comprehend is NOT the same as Transcribe Call Analytics.** Call Analytics bundles both into a single service with call-specific features (summarization, categories, issues).
30. **SageMaker Canvas ≠ SageMaker Autopilot.** Canvas = no-code UI for business users. Autopilot = automated ML that generates code/notebooks for ML engineers.

---

## 12. Decision Tree: Choosing the Right AI/ML Service

```
START: What is the primary need?
│
├─── Need to PROCESS DOCUMENTS?
│    ├─── Extract text + structure (forms, tables)? → Textract
│    │    ├─── Multi-page PDF? → Textract Async API
│    │    ├─── Low confidence results? → A2I human review
│    │    └─── Ask questions about document? → Textract Queries
│    └─── Read text in natural images? → Rekognition DetectText
│
├─── Need to ANALYZE TEXT (NLP)?
│    ├─── Standard tasks (sentiment, entities, PII)? → Comprehend
│    │    ├─── Medical/healthcare text? → Comprehend Medical
│    │    └─── Custom categories? → Comprehend Custom Classification
│    ├─── Generate/summarize/explain? → Bedrock
│    └─── Translate language? → Amazon Translate
│
├─── Need to ANALYZE IMAGES/VIDEO?
│    ├─── Detect objects, scenes, faces? → Rekognition
│    │    ├─── Search faces in database? → Rekognition Face Collections
│    │    ├─── Content moderation? → Rekognition Content Moderation
│    │    ├─── Custom objects? → Rekognition Custom Labels
│    │    └─── Live video stream? → Rekognition + Kinesis Video Streams
│    └─── Generate images? → Bedrock (Stability AI / Amazon Titan)
│
├─── Need SPEECH-TO-TEXT?
│    ├─── General transcription? → Transcribe
│    │    ├─── Medical dictation? → Transcribe Medical
│    │    ├─── Call center audio? → Transcribe Call Analytics
│    │    └─── Real-time? → Transcribe Streaming
│    └─── Text-to-speech? → Amazon Polly
│
├─── Need GENERATIVE AI?
│    ├─── Text generation/chat/summarization? → Bedrock
│    │    ├─── Needs company data for answers? → Bedrock Knowledge Bases (RAG)
│    │    ├─── Needs to call APIs? → Bedrock Agents
│    │    ├─── Needs content safety? → Bedrock Guardrails
│    │    └─── Needs custom behavior? → Bedrock Fine-Tuning
│    └─── Image generation? → Bedrock (Stability AI / Titan Image)
│
├─── Need CUSTOM ML MODEL?
│    ├─── Tabular data (classification/regression)? → SageMaker
│    │    ├─── No ML expertise? → SageMaker Canvas
│    │    ├─── Need auto-ML? → SageMaker Autopilot
│    │    └─── Full control? → SageMaker Studio + Training
│    ├─── Need to deploy model at scale? → SageMaker Endpoints
│    │    ├─── Infrequent traffic? → Serverless Inference
│    │    ├─── Large payloads (>6 MB)? → Async Inference
│    │    └─── Offline bulk scoring? → Batch Transform
│    └─── Edge deployment? → SageMaker Neo + Edge Manager
│
└─── Need to ORCHESTRATE AI pipeline?
     ├─── Multi-step AI processing? → Step Functions
     ├─── ML pipeline (train → evaluate → deploy)? → SageMaker Pipelines
     └─── Human review loop? → A2I (Augmented AI)
```
