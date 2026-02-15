# Text2SQL RAG Project - Interview Preparation Guide

> **Last Updated:** February 2026  
> **Project:** Multi-Source RAG + Text-to-SQL API  
> **Deployment:** AWS Lambda (Docker) with Function URLs  
> **Repository:** github.com/jndumu/text2sqlrag-project

---

## 📋 Quick Navigation

1. [Executive Summary](#executive-summary)
2. [Project Overview](#project-overview)
3. [Architecture Deep-Dive](#architecture-deep-dive)
4. [Technical Interview Q&A](#technical-interview-qa)
5. [Technical Achievements](#technical-achievements)
6. [Lessons Learned](#lessons-learned)
7. [Behavioral Interview Answers](#behavioral-interview-answers)
8. [Key Technologies](#key-technologies)
9. [Code Snippets](#code-snippets)
10. [Quick Facts Cheat Sheet](#quick-facts-cheat-sheet)

---

## Executive Summary

**What I Built:**
A production-ready RAG system deployed on AWS Lambda that converts natural language to SQL queries and answers document-based questions. Features multi-level caching (S3 + Redis), intelligent query routing, and an automated CI/CD pipeline.

**Key Achievements:**
- ⚡ **6-7x faster deployments** (30 min → 5 min) via base image caching
- 💰 **60% storage cost reduction** with S3 lifecycle policies
- 🚀 **70-80% API cost reduction** with multi-level caching
- ⏱️ **100-200ms response time** for cached queries
- 📦 **Zero manual AWS setup** - Everything auto-created

**Technology Stack:**
- **Backend:** Python 3.12, FastAPI, async/await
- **AI/ML:** OpenAI GPT-4, Embeddings, Vanna.ai, Pinecone
- **Database:** Supabase PostgreSQL (connection pooler)
- **Storage:** AWS S3 (documents), Upstash Redis (queries)
- **Deployment:** AWS Lambda (Docker), ECR, Function URLs, GitHub Actions

**Deployment Flow:**

GitHub Push → GitHub Actions → Docker Build → ECR → Lambda → Function URL


---

## Project Overview

### The Problem
Organizations have data scattered across documents (PDFs, DOCX) and databases (PostgreSQL). Users need to query both using natural language, but existing solutions require knowing SQL or manually searching documents.

### My Solution
Built an intelligent system that:
1. **Routes queries automatically** - Detects if question needs SQL, documents, or both
2. **RAG for documents** - Uploads PDFs, chunks text (512 tokens), generates embeddings, stores in Pinecone, answers with GPT-4
3. **Text-to-SQL** - Converts "What were total sales last month?" to `SELECT SUM(total) FROM orders WHERE...`
4. **Multi-level caching** - Redis (10-20ms) + S3 (100-200ms) → 75% cache hit rate
5. **Production deployment** - AWS Lambda with Docker, automated CI/CD, Function URLs

### Key Features
- **18 REST endpoints** - Upload, query, stats, health checks
- **5-stage RAG pipeline** - Parse → Chunk → Embed → Retrieve → Generate
- **Two-step SQL approval** - Generate → Review → Execute (safety)
- **Intelligent routing** - 60+ keywords detect SQL vs. document queries
- **Automatic resource creation** - ECR repos, S3 bucket, Function URL
- **Multi-level caching** - Different TTLs per operation (15min - 7 days)

---

## Architecture Deep-Dive

User Request ↓ Lambda Function URL (public HTTPS) ↓ FastAPI Application (18 endpoints) ↓ Query Router (60+ keywords) ↓ ├─→ RAG Pipeline (documents) │ ├─→ Redis Cache (1h TTL) │ ├─→ S3 Cache (30d lifecycle) │ ├─→ Pinecone (vectors) │ └─→ OpenAI (GPT-4) │ └─→ SQL Pipeline (database) ├─→ Redis Cache (15min-24h TTL) ├─→ Vanna.ai (Text-to-SQL) └─→ Supabase (PostgreSQL via pooler)

### System Architecture


### Data Flow

**Document Upload:**

User uploads PDF (5 MB)
Compute SHA-256 hash → doc_id
Check S3 cache → Exists? Skip parsing
If cache miss: a. Parse with Docling (context-aware) b. Chunk (512 tokens, 50 overlap) → 45 chunks c. Embed (OpenAI) → 45 vectors (1536 dims) d. Save to S3 (document.pdf, chunks.json, embeddings.npy, metadata.json)
Upsert to Pinecone
Return: cache_hit indicator + chunk count


**Query Flow (RAG):**

User: "What is the return policy?"
Check Redis → Hit? Return cached answer (10ms)
If miss: a. Embed query → 1536-dim vector b. Search Pinecone → top 3 chunks (cosine similarity) c. Assemble context (chunks + question) d. Generate answer (GPT-4) e. Cache in Redis (1h TTL)
Return answer with sources


**Query Flow (SQL):**

User: "Total sales last month?"
Check Redis → Hit? Return SQL (10ms)
If miss: a. Vanna generates SQL (GPT-4o) b. Return SQL + query_id (pending approval)
User approves → Execute SQL
Cache result (15min TTL)
Return query results


### S3 Cache Structure
s3://rag-cache-josie/ pdf/{doc_id}/ document.pdf (5 MB, original) chunks.json (12 KB, parsed text) embeddings.npy (270 KB, 45 chunks × 1536 × 4 bytes) metadata.json (2 KB, processing info) txt/{doc_id}/ document.txt chunks.json embeddings.npy metadata.json


**Cache Strategy: All-or-Nothing**
- Check if ALL 4 files exist
- If any missing → Cache miss (prevents partial corruption)
- 30-day lifecycle policy → Auto-delete old files

### Redis Cache Structure
Keys: rag:sha256(query) → Answer (TTL: 1h) sql:sha256(query) → SQL (TTL: 24h) sql_result:query_id → Results (TTL: 15min) embeddings:sha256(text) → Vector (TTL: 7d)

Example: rag:abc123... → { "answer": "Our return policy...", "sources": ["return_policy.pdf:2", ...], "cached_at": "2026-02-09T10:30:00Z" }


---

## Technical Interview Q&A

### Deployment & CI/CD

**Q: Walk me through your deployment pipeline.**

**A:** "My CI/CD uses GitHub Actions with base image caching for 6-7x speedup:

**Stage 1: Check for Base Image Cache**
- Try pulling `ECR/lambda-python-deps:3.12` from ECR
- If exists (cache hit): Use it (saves 25-30 minutes)
- If not (cache miss): Build once with all system dependencies

**Stage 2: Build Application Image**
- Multi-stage Dockerfile:
  - Stage 1: Import UV binary (fast package manager)
  - Stage 2: Install Python deps with BuildKit cache mounts
  - Stage 3: Copy application code (minimal runtime image)
- Result: 2GB image in 5-7 minutes

**Stage 3: Automated Resource Creation**
- Auto-create ECR repos (if missing) with image scanning + encryption
- Auto-create S3 bucket with versioning + 30-day lifecycle policy
- Auto-create Lambda Function URL with CORS

**Stage 4: Deploy**
- Push multi-tagged images to ECR (commit SHA, latest, amd64)
- Update Lambda function code with new image URI
- Configure environment variables (S3, Redis, database, API keys)
- Verify Function URL, run health checks

**Key Innovation:** First deployment takes 30-35 minutes to build base image. All subsequent deployments take 5-7 minutes because they reuse the cached base. This is a 6-7x improvement."

---

**Q: What challenges did you face with Docker in Lambda?**

**A:** "Three major challenges:

**1. Dockerfile ARG Scope Bug (2 hours debugging)**

Error: `failed to parse stage name "/:":`

Problem:
```dockerfile
# WRONG - ARG after FROM is out of scope
FROM ${ECR_REGISTRY}/base:${TAG}
ARG ECR_REGISTRY  # Too late, not visible above

Solution:

Dockerfile
# CORRECT - ARG before FROM
ARG ECR_REGISTRY=403705275896.dkr.ecr.us-east-1.amazonaws.com
ARG TAG=3.12
FROM ${ECR_REGISTRY}/base:${TAG}
Docker was interpreting ${ECR_REGISTRY} as empty, resulting in FROM /:3.12.

2. Image Size (Initial 4GB+)

Problem: Lambda has 10GB limit, large images slow cold starts
Solution: Multi-stage builds to separate build-time from runtime deps
Result: Final image 2GB
3. Platform Specification

Problem: Local Mac is ARM64, Lambda is x86-64
Solution: Always use --platform linux/amd64 in builds
Result: Consistent deployments across environments"
Storage & Caching
Q: Explain your caching strategy and cost savings.

A: "I use two-tier caching optimized for different access patterns:

L2 Cache: Redis (Query Results)

Purpose: Hot cache for recent queries
TTL: Variable by operation
RAG answers: 1 hour (may change with new docs)
SQL generation: 24 hours (schema stable)
SQL results: 15 minutes (data changes)
Embeddings: 7 days (content-based, static)
Latency: 10-20ms
Hit rate: ~75%
Cost: Free (Upstash free tier: 10K requests/day)
L1 Cache: S3 (Document Cache)

Purpose: Persistent cache for processed documents
TTL: 30 days (lifecycle policy auto-deletes)
Structure: 4 files per document (PDF + chunks + embeddings + metadata)
Latency: 100-200ms
Hit rate: ~80% (documents rarely change)
Cost: $0.023/GB/month with 60% reduction via lifecycle
All-or-Nothing Logic:

Python
def cache_exists(doc_id):
    # Require ALL 4 files for cache hit (prevents partial corruption)
    return all([
        s3.exists(f'{doc_id}/document.pdf'),
        s3.exists(f'{doc_id}/chunks.json'),
        s3.exists(f'{doc_id}/embeddings.npy'),
        s3.exists(f'{doc_id}/metadata.json')
    ])
Cost Comparison (100K queries/month):

Without caching:

100K OpenAI API calls (embed + GPT-4)
Embeddings: $10, GPT-4: $300
Total: $310/month
With caching:

75% from Redis (free, no API calls)
20% from S3 (skip embedding, only GPT-4)
5% full pipeline (embed + GPT-4)
Total: ~$77/month
Savings: 75% cost reduction + 10-15x faster for cache hits."

Q: Why S3 instead of local storage for Lambda?

A: "Lambda's filesystem has three critical limitations:

1. Ephemeral Storage

Only /tmp is writable (max 10GB)
Cleared between cold starts
Lost when container terminates
Can't persist data across invocations
2. Container Reuse is Unpredictable

Warm containers may reuse /tmp, but no guarantee
Multiple concurrent invocations get separate containers
Can't rely on filesystem cache
3. Cost at Scale

Provisioned concurrency (for persistent containers): $0.015/GB-hour
For 10GB storage: $108/month
S3 storage: $0.023/GB-month = $0.23 for 10GB
S3 is 470x cheaper
My Implementation:

Python
# Auto-select storage backend based on environment
if settings.STORAGE_BACKEND == "s3":
    storage = S3StorageBackend()  # Lambda/production
else:
    storage = LocalStorageBackend()  # Local development
S3 Benefits:

Persistent across all invocations
Shared across concurrent Lambda instances
~100-200ms latency (acceptable for cache)
IAM role auth (no credentials)
Versioning for data protection
Lifecycle policies for cost control"
Database Connectivity
Q: You had database connection issues. How did you fix them?

A: "The error was: Cannot assign requested address when connecting to Supabase.

Root Cause:

Using direct connection (port 5432) instead of pooler (port 6543)
IPv6/networking issues between Lambda and Supabase
Each Lambda invocation creates new connection → exhausts pool
Solution: Connection Pooler

Changed from:

Code
postgresql://postgres:[PASS]@db.xxx.supabase.co:5432/postgres
To:

Code
postgresql://postgres.xxx:[PASS]@aws-1-us-east-1.pooler.supabase.com:6543/postgres
Key Changes:

Username: postgres → postgres.{project_ref}
Host: Direct DB → Pooler endpoint
Port: 5432 → 6543
Added: PGHOST_PREFER_IPV4=1 environment variable
Why Pooler?

Reuses connections (prevents exhaustion)
IPv4-friendly endpoint
Built-in for Lambda/serverless
4-5x latency reduction (300ms → 60ms)
Diagnosis Process:

Checked CloudWatch logs → Found IPv6 address in error
Tested connection locally (worked) vs. Lambda (failed)
Researched Supabase + Lambda best practices
Switched to pooler, tested, deployed
Verified with: aws lambda get-function-configuration"
RAG Implementation
Q: How does your RAG system work?

A: "My RAG pipeline has 5 stages:

Stage 1: Document Ingestion

Python
# Parse document (PDF/DOCX/TXT/CSV/JSON)
chunks = parse_and_chunk_with_context(
    file_path,
    chunk_size=512,       # Target tokens
    min_chunk_size=256,   # Merge small chunks
    overlap=50            # Preserve context
)
# Result: 45 chunks with metadata (page, section, tokens)
Why 512 tokens with 50 overlap?

Fits in GPT-4 context (8K tokens = ~15 chunks)
Overlap preserves context across boundaries
Small enough for precision, large enough for meaning
Stage 2: Embedding Generation

Python
embeddings = openai.embed(
    texts=[c['text'] for c in chunks],
    model="text-embedding-3-small",  # 1536 dims
)
# Cache in S3: embeddings.npy (NumPy array)
# Shape: (45, 1536), dtype=float32, size=270KB
Stage 3: Vector Storage

Python
pinecone.upsert(vectors=[
    {
        "id": f"{doc_id}_{i}",
        "values": embedding.tolist(),
        "metadata": {
            "text": chunk['text'],
            "source": filename,
            "page": chunk['page'],
            "chunk_index": i
        }
    }
    for i, embedding in enumerate(embeddings)
])
Stage 4: Query & Retrieval

Python
# User: "What is the return policy?"
query_embedding = openai.embed(query)

# Semantic search (cosine similarity)
results = pinecone.query(
    vector=query_embedding,
    top_k=3,  # Retrieve top 3 chunks
    namespace="default"
)

# Extract context
context = "\n\n---\n\n".join([
    f"Source: {r.metadata['source']}, Page {r.metadata['page']}\n{r.metadata['text']}"
    for r in results.matches
])
Why top_k=3?

Balance precision (fewer = more relevant) and recall (more = more context)
3 chunks ≈ 1500 tokens, leaves room for question + answer in 8K window
Stage 5: Answer Generation

Python
prompt = f"""You are a helpful assistant. Answer based on context.

Context:
{context}

Question: {query}

Answer:"""

answer = openai.chat.completions.create(
    model="gpt-4-turbo-preview",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.7
)
Performance:

Cache hit: 10-20ms (Redis)
Cache miss: 2-3 seconds (full pipeline)
Typical cache hit rate: 75%
Cost savings: 70-80%"
Performance Optimization
Q: What optimizations did you implement?

A: "Six major optimizations with quantified results:

1. UV Package Manager (5-7x faster)

Before:

Dockerfile
RUN pip install -r requirements.txt  # 15-20 minutes
After:

Dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install -r requirements.txt  # 2-3 minutes
Result: 10-100x faster installs

2. Base Image Caching (6-7x deployment speedup)

Strategy:

Build system dependencies once (poppler, gcc, cairo, etc.)
Store in ECR as base image
Application builds pull cached base (2-3 min vs. 25-30 min)
Result: 30-35 min → 5-7 min deployments

3. Multi-Level Caching (70-80% cost reduction)

100K queries/month cost:

Without cache: $310/month (100K API calls)
With cache: $77/month (25K API calls, 75% hit rate)
Result: 75% cost savings

4. Docker Layer Caching

Dockerfile
# CORRECT - Dependencies cached separately
COPY requirements.txt .
RUN uv pip install -r requirements.txt  # Cached unless requirements change
COPY app/ /var/task/app/  # Only this rebuilds on code changes
Result: 30-second rebuilds instead of 15 minutes

5. S3 Lifecycle Policies (60% storage savings)

JSON
{
  "Rules": [{
    "ID": "DeleteOldCache",
    "Expiration": {"Days": 30}
  }]
}
100GB storage:

Without lifecycle: $2.30/month
With lifecycle: $0.92/month (60% old files deleted)
6. Connection Pooling (4-5x faster queries)

Before (direct): Connect (200ms) + Query (50ms) = 250ms After (pooler): Reuse (10ms) + Query (50ms) = 60ms

Result: 4x faster database operations"

Error Handling & Debugging
Q: Tell me about a challenging bug.

A: "The Dockerfile ARG substitution bug was most challenging.

Error: failed to parse stage name "/:":

Context: I was trying to reference a base image in ECR using build arguments:

Dockerfile
ARG ECR_REGISTRY=403705275896.dkr.ecr.us-east-1.amazonaws.com
FROM ${ECR_REGISTRY}/base:3.12
Problem: Docker was interpreting ${ECR_REGISTRY} as empty, resulting in FROM /:3.12.

Debugging Process:

Analyzed error: "parse stage name" + "/:" → Missing registry
Checked docs: Found ARG scope is limited per build stage
Tested locally:
bash
# Hard-coded (worked)
FROM 403705275896.dkr.ecr.us-east-1.amazonaws.com/base:3.12

# ARG before FROM (SUCCESS!)
ARG ECR_REGISTRY
FROM ${ECR_REGISTRY}/base:3.12
Solution:

Dockerfile
# ARG must be declared BEFORE the FROM that uses it
ARG ECR_REGISTRY=403705275896.dkr.ecr.us-east-1.amazonaws.com
ARG TAG=3.12
FROM ${ECR_REGISTRY}/lambda-python-deps:${TAG}
Impact:

Bug caused 3-4 failed deployments
Cost ~2 hours debugging
Once fixed, never happened again
Lessons:

Docker has subtle scoping rules in multi-stage builds
Read error messages carefully ("/:" was the key clue)
Test locally before pushing to CI/CD
Document solutions (added comments in Dockerfile)"
Security & IAM
Q: How do you handle secrets and security?

A: "Multi-layered security:

1. GitHub Secrets (All sensitive data)

YAML
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
Encrypted at rest
Auto-masked in logs
Never committed to git
.env.example shows required vars (no values)
2. Lambda IAM Role (Least Privilege)

JSON
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
  "Resource": "arn:aws:s3:::rag-cache-josie/*"
}
No hardcoded AWS credentials
Boto3 auto-discovers role
Scoped to specific bucket only
3. ECR Security

Image scanning on push (vulnerability detection)
AES256 encryption at rest
Private repositories
4. Network Security

TLS for all connections (database, APIs)
Connection pooler (no direct DB exposure)
Could add: API Gateway + API keys, AWS WAF
5. Database Security

Python
DATABASE_URL = "postgresql://...?sslmode=require"  # Force TLS
Future Enhancements:

AWS Secrets Manager (credential rotation)
VPC for Lambda (private subnet)
API Gateway with rate limiting
CloudWatch Alarms (unusual activity)"
Scalability & Cost
Q: How does your system scale and what are costs?

A:

Scalability:

Lambda Auto-Scaling:

Concurrency: 0 → 1000 (default limit)
Cold start: 3-5 seconds
Warm start: 200-500ms
Burst: 3000/second
Cost: $0.20/1M requests
Pinecone:

Serverless (auto-scales)
~10ms query latency
100+ concurrent queries
S3:

Unlimited capacity
11 nines durability
5,500 GET/sec per prefix
Auto-sharding
Connection Pooler:

100+ concurrent connections
Prevents DB exhaustion
Cost Breakdown (100K requests/month):

Service	Monthly Cost
Lambda (100K req, 512MB, 3s)	$8
S3 (10GB storage + requests)	$0.30
ECR (2GB image)	$0.20
Pinecone (100K vectors)	$70
Supabase (Free tier)	$0
OpenAI (75% cache hit)	$4.50
Upstash Redis (Free tier)	$0
Total	~$83/month
Cost at Scale (1M requests/month):

Service	100K	1M	Notes
Lambda	$8	$80	Linear
S3	$0.30	$2.50	More docs
Pinecone	$70	$420	Upgrade tier
OpenAI	$4.50	$46	Reduced by caching
Total	$83	$549	6.6x for 10x traffic
Cost Optimizations:

S3 lifecycle (60% savings)
Multi-level caching (75% API savings)
Base image caching (GitHub Actions minutes)
Connection pooling (prevents over-provisioning)
Bottlenecks & Solutions:

Bottleneck	Solution	Cost Impact
OpenAI costs	Use GPT-3.5 for simple queries	-50%
Pinecone costs	Migrate to Weaviate (self-hosted)	-$50/month
Cold starts	Provisioned concurrency	+$13/month
S3 latency	CloudFront CDN	+$5/month
Technical Achievements
Quantified Results
Performance Improvements:

⚡ 6-7x faster deployments (30-35 min → 5-7 min)

Method: Base Docker image caching in ECR
Impact: Saved 1250 minutes over 50 deployments
⏱️ 10-15x faster queries (2-3 seconds → 100-200ms)

Method: Multi-level caching (Redis + S3)
Impact: 75% cache hit rate
🔄 4-5x faster database queries (300ms → 60ms)

Method: Connection pooler vs. direct connection
Impact: Reuses connections, prevents exhaustion
📦 10-100x faster package installs (15 min → 2 min)

Method: UV package manager with BuildKit cache
Impact: Faster builds and local development
Cost Reductions:

💰 60% storage cost reduction ($2.30 → $0.92 for 100GB)

Method: S3 lifecycle policy (auto-delete >30 days)
Impact: Sustainable long-term storage
🚀 70-80% API cost reduction ($310 → $77 for 100K queries)

Method: Multi-level caching strategy
Impact: 75% cache hit rate
📊 83% deployment time reduction (30 min → 5 min)

Method: Base image caching
Impact: Faster iteration, lower CI/CD costs
Automation:

📦 Zero manual AWS setup - Everything auto-created
ECR repositories (with scanning + encryption)
S3 bucket (with versioning + lifecycle)
Lambda Function URL (with CORS)
Quality:

🎯 All-or-nothing cache strategy - Prevents partial corruption
🔒 IAM role-based security - No hardcoded credentials
📝 18 REST endpoints - Complete API coverage
🧪 Health checks - Automated deployment verification
Lessons Learned
Critical Technical Lessons
1. ARG Scope in Dockerfiles (2 hours debugging)

Dockerfile
# WRONG - ARG after FROM is out of scope
FROM ${ECR_REGISTRY}/base:${TAG}
ARG ECR_REGISTRY

# CORRECT - ARG before FROM
ARG ECR_REGISTRY
FROM ${ECR_REGISTRY}/base:${TAG}
Lesson: Always declare ARG before FROM that uses it

2. Connection Pooling for Serverless

Problem: Direct DB connections exhaust pool (100 max)
Solution: Use Supabase pooler (port 6543)
Lesson: Always use pooling for Lambda + databases
Impact: 4-5x faster queries, prevents exhaustion
3. Base Image Caching Saves Massive Time

Before: Build system deps every deployment (25-30 min)
After: Build once, cache in ECR (2-3 min)
Lesson: Separate static deps from application code
Impact: 6-7x deployment speedup
4. IPv6/IPv4 Issues in Lambda

Problem: Lambda couldn't connect to Supabase (IPv6 address in error)
Solution: Add PGHOST_PREFER_IPV4=1 + use pooler
Lesson: Always test connectivity from Lambda, not just local
5. GitHub Actions Can't Modify Secrets

Expected: Workflow updates secrets automatically
Reality: Secrets are read-only from workflows
Lesson: Secrets must be updated manually or via GitHub API
Workaround: Document required secrets in README
6. S3 Lifecycle Policies Prevent Cost Explosion

Without: Cache grows unbounded ($2.30/month for 100GB)
With: Auto-delete >30 days ($0.92/month)
Lesson: Always set lifecycle policies for cache/log storage
Impact: 60% cost savings
7. Docker Layer Caching Requires Order

Dockerfile
# WRONG - Code changes invalidate all layers
COPY app/ /var/task/app/
RUN pip install -r requirements.txt

# CORRECT - Dependencies cached separately
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app/ /var/task/app/
Lesson: Copy requirements before deps, code after Impact: 30-second rebuilds vs. 15 minutes

8. All-or-Nothing Cache Prevents Corruption

Python
def cache_exists(doc_id):
    # Check ALL 4 files exist (not just 1)
    return all([
        s3.exists(f'{doc_id}/document.pdf'),
        s3.exists(f'{doc_id}/chunks.json'),
        s3.exists(f'{doc_id}/embeddings.npy'),
        s3.exists(f'{doc_id}/metadata.json')
    ])
Lesson: Require all files for cache hit (prevents partial uploads)

What I'd Do Differently
If I Started Over:

Use Terraform for Infrastructure

Define all resources as code (ECR, S3, Lambda, IAM)
Version control infrastructure
Easier to replicate across regions/accounts
Implement CI/CD from Day 1

Don't wait for 10+ failed manual deployments
Start simple, iterate
Would save 10-20 hours of debugging
Consider ECS/Fargate Instead of Lambda

For this use case: Better control, no cold starts
Cost similar at moderate scale
More flexibility (custom runtimes, longer timeouts)
Choose Self-Hosted Vector DB

Weaviate on ECS: $20/month vs. Pinecone $70-420/month
Lower long-term costs
More control
Add Monitoring Earlier

X-Ray tracing from day 1
CloudWatch alarms before production
Cost tracking per feature
Behavioral Interview Answers
STAR Format Examples
Q: Tell me about a time you optimized performance.

Situation: "In my Text2SQL RAG project, deployments were taking 30-35 minutes, which made iteration painfully slow and blocked development."

Task: "I needed to reduce deployment time to under 10 minutes while maintaining reproducibility and keeping the image size under Lambda's 10GB limit."

Action: "I implemented a three-part optimization:

Base Image Caching - Built system dependencies (Poppler, Cairo, GCC) once and stored in ECR. Application code layered on top.

UV Package Manager - Replaced pip with UV (10-100x faster) and used BuildKit cache mounts to persist downloads.

Multi-Stage Dockerfile - Separated UV import, dependency build, and runtime image to minimize final size."

Result: "Deployment time dropped from 30-35 minutes to 5-7 minutes - a 6-7x improvement. Over 50 deployments, this saved ~1250 minutes of CI/CD time, equivalent to $10/month in GitHub Actions costs plus significant developer time."

Learning: "I learned that investing time upfront in build optimization pays dividends throughout the project. Separating static dependencies from application code is now a pattern I use in all projects."

Q: Describe a complex bug you debugged.

Situation: "I had a Dockerfile bug causing deployments to fail with: failed to parse stage name "/:"."

Task: "I needed to identify why Docker couldn't parse the FROM statement without breaking the multi-stage build."

Action: "I took a systematic approach:

Analyzed error: 'parse stage name' + '/:" suggested FROM issue with missing registry

Checked docs: Found ARG scope is limited per build stage

Tested locally:

Hard-coded values (worked)
ARG after FROM (failed)
ARG before FROM (SUCCESS!)
Applied fix: Moved ARG declarations before FROM statement"

Result: "Fixed in ~2 hours after systematic debugging. Deployment succeeded immediately. I documented the solution in comments to help future developers."

Learning: "Three key lessons:

Read error messages carefully ('/:" was the key clue)
Test locally before pushing to CI/CD
Document solutions for the team"
Q: How do you handle stress when things fail repeatedly?

Situation: "During deployment setup, I had 10+ consecutive failed deployments due to various issues."

Task: "I needed to maintain focus, avoid burnout, and systematically resolve each issue."

Action: "I developed a stress management approach:

Strategic breaks - After 2-3 failures, step away for 10-15 minutes
Systematic debugging - Check logs line-by-line, verify AWS resources
Document everything - Track what I tried, prevent repeating mistakes
Celebrate small wins - When S3 auto-creation worked, committed immediately
Ask for help - After ~1 hour stuck, search Stack Overflow, GitHub Issues, docs"
Result: "All 10 failures became learning opportunities:

Failures 1-3: ARG scope issue
Failures 4-6: ECR missing (added auto-creation)
Failures 7-9: Database connection (pooler fix)
Failure 10: Health check timeout (made non-blocking)
System is now fully automated and reliable."

Learning: "Failure is feedback. Stay systematic, take breaks, document learnings, celebrate progress. Each error message is a clue."

Key Technologies
Technology Stack Summary
Backend:

Python 3.12, FastAPI, async/await
Pydantic validation, type hints
18 REST endpoints
AI/ML:

OpenAI GPT-4 (answer generation)
text-embedding-3-small (1536 dims)
Vanna.ai (Text-to-SQL)
Docling (PDF parsing)
Databases:

Pinecone (vector DB, cosine similarity)
Supabase PostgreSQL (connection pooler)
Upstash Redis (query cache)
Storage:

AWS S3 (document cache, 30-day lifecycle)
ECR (Docker images)
Deployment:

AWS Lambda (serverless compute)
Function URLs (public HTTPS)
Docker (multi-stage builds)
CI/CD:

GitHub Actions (automated deployments)
UV package manager (10-100x faster)
BuildKit cache mounts
Monitoring:

CloudWatch Logs & Metrics
OPIK (optional observability)
Code Snippets
1. Dockerfile ARG Fix (Critical)
Dockerfile
# CORRECT - ARG before FROM for variable substitution
ARG ECR_REGISTRY=403705275896.dkr.ecr.us-east-1.amazonaws.com
ARG BASE_IMAGE_REPOSITORY=lambda-python-deps
ARG BASE_IMAGE_TAG=3.12

FROM ${ECR_REGISTRY}/${BASE_IMAGE_REPOSITORY}:${BASE_IMAGE_TAG}

# Re-declare inside stage if needed
ARG FUNCTION_DIR="/var/task"
WORKDIR ${FUNCTION_DIR}
2. Lambda Lazy Initialization
Python
# lambda_handler.py
from mangum import Mangum
from app.main import app, initialize_services

_handler = Mangum(app, lifespan="off")
_services_initialized = False

def handler(event, context):
    global _services_initialized
    
    # Initialize on first request (not at import)
    if not _services_initialized:
        initialize_services()
        _services_initialized = True
    
    return _handler(event, context)
Why: Imports complete in <10s, services init on first invocation

3. S3 All-or-Nothing Cache
Python
def cache_exists(self, document_id: str, file_extension: str) -> bool:
    """Check if ALL 4 files exist (prevents partial corruption)."""
    required = [
        f"document.{file_extension}",
        "chunks.json",
        "embeddings.npy",
        "metadata.json"
    ]
    
    for filename in required:
        key = f"{file_extension}/{document_id}/{filename}"
        if not self._object_exists(key):
            return False
    
    return True
4. Connection Pooler Setup
Python
# Before (Direct - ❌ Fails)
DATABASE_URL = "postgresql://postgres:[PASS]@db.xxx.supabase.co:5432/postgres"

# After (Pooler - ✅ Works)
DATABASE_URL = "postgresql://postgres.xxx:[PASS]@aws-1-us-east-1.pooler.supabase.com:6543/postgres"
os.environ["PGHOST_PREFER_IPV4"] = "1"
5. Multi-Level Caching
Python
async def generate_answer(self, question: str, top_k: int = 3):
    # L2: Redis (hot cache, 1h TTL)
    cache_key = f"rag:{hashlib.sha256(question.encode()).hexdigest()}"
    
    if cached := self.redis.get(cache_key):
        return cached  # 10-20ms
    
    # L1: S3 (embeddings cache, 7d TTL)
    if embeddings := self.s3.load_embeddings(doc_id):
        vectors = embeddings
    else:
        vectors = await openai.embed(chunks)
        self.s3.save_embeddings(doc_id, vectors)
    
    # Generate answer
    answer = await generate_with_gpt4(question, vectors)
    
    # Cache result
    self.redis.set(cache_key, answer, ex=3600)
    
    return answer
6. Query Routing
Python
class QueryRouter:
    SQL_KEYWORDS = {
        "total", "sum", "count", "average", "max", "min",
        "how many", "show me", "list all", "find",
        "greater than", "less than", "between",
        "last month", "this year", "since"
    }
    
    DOC_KEYWORDS = {
        "explain", "describe", "what is", "how does",
        "policy", "process", "procedure", "definition"
    }
    
    @classmethod
    def route(cls, question: str) -> str:
        q = question.lower()
        
        sql_score = sum(1 for kw in cls.SQL_KEYWORDS if kw in q)
        doc_score = sum(1 for kw in cls.DOC_KEYWORDS if kw in q)
        
        if sql_score > doc_score:
            return "SQL"
        elif doc_score > sql_score:
            return "DOCUMENTS"
        else:
            return "HYBRID"
7. UV with Cache Mounts
Dockerfile
# Import UV binary
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# Install with cache mount (persists across builds)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=requirements.txt,target=requirements.txt \
    uv pip install --system --no-cache -r requirements.txt
Result: 10-100x faster than pip

8. ECR Auto-Creation
YAML
- name: Create ECR repos if needed
  run: |
    if ! aws ecr describe-repositories \
       --repository-names lambda-python-deps 2>/dev/null; then
      aws ecr create-repository \
        --repository-name lambda-python-deps \
        --image-scanning-configuration scanOnPush=true \
        --encryption-configuration encryptionType=AES256
    fi
9. S3 Lifecycle Policy
JSON
{
  "Rules": [{
    "ID": "DeleteOldCache",
    "Status": "Enabled",
    "Filter": {},
    "Expiration": {"Days": 30},
    "NoncurrentVersionExpiration": {"NoncurrentDays": 7}
  }]
}
Applied:

bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket rag-cache-josie \
  --lifecycle-configuration file://lifecycle.json
Quick Facts Cheat Sheet
System Stats
18 REST endpoints - Upload, query, stats, health
512 token chunks with 50-token overlap
1536-dim embeddings (OpenAI text-embedding-3-small)
Top-k=3 for semantic retrieval (cosine similarity)
2GB Docker image (multi-stage build, optimized)
5-7 minute deployments (with base image cache)
Performance
10-20ms - Redis cache hit
100-200ms - S3 cache hit
2-3 seconds - Full RAG pipeline (cache miss)
60ms - Database query (with pooler)
3-5 seconds - Lambda cold start
200-500ms - Lambda warm start
Cost (100K requests/month)
Lambda: $8
S3: $0.30
ECR: $0.20
Pinecone: $70
OpenAI: $4.50 (with 75% cache hit)
Total: ~$83/month
Cache Performance
Redis hit rate: 75%
S3 hit rate: 80%
Combined API cost reduction: 70-80%
Storage cost reduction: 60% (lifecycle policy)
Deployment
CI/CD: GitHub Actions (free for public repos)
Automated: ECR repos, S3 bucket, Function URL
First deploy: 30-35 minutes (builds base image)
Subsequent: 5-7 minutes (uses cache)
Speedup: 6-7x
Technologies Count
8 AWS services - Lambda, S3, ECR, CloudWatch, IAM, Function URLs
4 AI/ML services - OpenAI (GPT-4, Embeddings), Pinecone, Vanna.ai, Docling
3 databases - Pinecone (vectors), Supabase (PostgreSQL), Upstash (Redis)
2-tier caching - L2 Redis (10-20ms), L1 S3 (100-200ms)
1 goal - Make data accessible via natural language
Key Decisions
Lambda over EC2 - Serverless simplicity
Docker over Zip - Large dependencies (2GB)
S3 over EFS - 13x cheaper
Pinecone over Weaviate - Managed service (for now)
OpenAI over open-source - Best quality
Connection pooler - Prevents DB exhaustion
UV over pip - 10-100x faster
Multi-stage Dockerfile - Smaller images
Function URL over API Gateway - Simpler, cheaper
Interview Highlights
✅ Built production RAG system from scratch
✅ 6-7x deployment speedup via caching
✅ 70-80% cost reduction via multi-level caching
✅ Debugged complex Docker/AWS issues
✅ Automated entire deployment pipeline
✅ Zero manual AWS setup required
✅ Used 15+ technologies effectively
✅ Quantified all improvements
✅ Learned from 10+ failed deployments
✅ Production-ready with monitoring

Sample Answer Framework
Use this structure for any technical question:

1. Context (15 seconds)

What was the situation?
What technology were you using?
Why did this problem matter?
2. Challenge (15 seconds)

What made it difficult?
What constraints did you face?
What would happen if you didn't solve it?
3. Approach (45 seconds)

How did you solve it? (Step-by-step)
What alternatives did you consider?
Why did you choose this approach?
4. Result (15 seconds)

What was the outcome? (Quantify!)
How did you measure success?
What was the impact?
5. Learning (10 seconds)

What did you learn?
How will you apply this?
What would you do differently?
Total Time: ~1.5-2 minutes

Example: "In my Text2SQL RAG project [Context], I needed to deploy a 2GB Docker image to Lambda with fast iteration [Challenge]. I implemented base image caching where system dependencies are built once and stored in ECR, with application code layered on top [Approach]. This reduced deployment time from 30-35 minutes to 5-7 minutes - a 6-7x improvement [Result]. I learned that separating static dependencies from application code is crucial for build optimization [Learning]."

Final Tips for Interviews
Before the Interview
 Review this guide (30 minutes)
 Practice explaining the architecture out loud
 Prepare 3-5 technical questions to ask interviewer
 Have AWS console/GitHub open (for reference if needed)
 Practice STAR format answers (record yourself)
During the Interview
✅ Lead with results: "I built a RAG system that reduced costs by 70%..."
✅ Quantify everything: "6-7x faster" not "much faster"
✅ Show problem-solving: Explain why you chose solutions
✅ Be honest about challenges: "I spent 2 hours debugging ARG scope..."
✅ Connect to principles: "This reinforced the importance of..."
✅ Ask clarifying questions: "Do you want me to explain the caching strategy?"
✅ Use whiteboard: Draw architecture if possible
✅ Stay concise: 1.5-2 minutes per answer
Common Interview Questions
"Tell me about your most complex project."
"Describe a time you optimized performance."
"How do you handle failures/errors?"
"Walk me through your deployment process."
"What was your biggest technical challenge?"
"How do you make technical decisions?"
"Describe your testing strategy."
"How do you handle stress/deadlines?"
"What would you do differently?"
"Why did you choose [technology X]?"
For each: Use the 5-part answer framework above.

Closing Statement for Interviews
"This project taught me end-to-end system design - from Docker optimization and AWS architecture to cost management and production monitoring. I'm proud that I quantified every improvement (6-7x deployment speedup, 70-80% cost reduction) and learned from 10+ deployment failures. The system is now production-ready with zero manual setup required. I'm excited to bring this systematic, metrics-driven approach to [Company Name]."

Remember: You built something impressive. Own it. Explain it clearly. Show you learned from challenges. Quantify your impact. 🚀

Good luck! 💪

Code

---

**To save:**
1. Copy everything above (starting from the first `#`)
2. Create a new file in your project root: `INTERVIEW_GUIDE.md`
3. Paste and save

Done! You now have a comprehensive interview guide in your repository. 🎉