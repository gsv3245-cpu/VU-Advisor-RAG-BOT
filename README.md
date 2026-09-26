# VU AI Academic Advisor (DATA308)

Author: Ganga Sagar Verma

## Live demo

Open the deployed application here:

https://vu-advisor.vercel.app/

This is the hosted version of the VU AI Academic Advisor chatbot for quick access and testing.

## Project overview

This project is a retrieval-augmented academic advisor built for Vidyashilp University’s B.Tech (Data Science) programme. It helps students and advisors answer curriculum-related questions using official academic data, university policy documents, and structured course records instead of guesswork.

The system is designed to answer questions about:
- semester-wise course plans and batch progression
- credits, L-T-P structure, prerequisites, electives, and baskets
- minor requirements and eligibility
- academic calendar and policy rules from the Student Handbook and SOP
- conflicts or missing information when the source data is unclear or inconsistent

Every answer is grounded in source material and cites the exact data source, sheet cell, or handbook page. The assistant can ask follow-up questions when details are incomplete and can flag conflicting information.

This project combines structured university data with document retrieval and LLM-based reasoning to create a reliable academic guidance assistant that works in both text and voice-friendly workflows.

## Core RAG workflow reflected in the project

The project follows the same conceptual flow shown in the class diagram/image for building a basic RAG application, but extends it with academic data validation, tool use, and grounding checks.

1. Import required libraries
   - Python packages for LangChain, FastAPI, vector search, PDF/Excel ingestion, and environment management

2. Load the source documents
   - Read the Student Handbook PDF, SOP PDF, and academic Excel files
   - Parse curriculum, minor, and semester structure information

3. Combine the data sources
   - Merge structured academic facts with text/document knowledge into one retrieval pipeline

4. Split the documents into chunks
   - Break long PDF sections into smaller passages for better retrieval precision

5. Convert text to embeddings
   - Generate embeddings using Azure OpenAI or AWS Bedrock Titan embeddings

6. Store the embeddings in a vector database
   - Persist the embeddings and metadata in a retrieval-friendly knowledge base

7. Create a retriever
   - Combine BM25 lexical search and dense vector search using reciprocal rank fusion

8. Choose the LLM
   - Use Azure OpenAI chat models for reasoning and answer generation

9. Create the prompt
   - Use a system prompt that instructs the model to answer only from the provided academic sources and to ask for missing information when needed

10. Ask a question
   - Send a student query through the retrieval + reasoning flow

11. Build the user interface
   - Expose the app via FastAPI and the static web frontend

12. Run the application
   - Launch the advisor locally or through the deployed Vercel app

This is the simplified RAG lifecycle that the project implements at a larger, more rigorous scale with academic validation, evidence checks, and deterministic tools.

## Assignment alignment and design goals

This project is designed to match the requirements of the assignment described in [GenAI_Assignment.pdf](GenAI_Assignment.pdf). It is not just a document Q&A demo: it is structured to behave like an academic decision-support system that can:

- answer curriculum and policy questions using official university sources
- recognize when the available information is insufficient
- ask follow-up questions before making assumptions
- use synthetic, anonymized student profiles instead of real personal data
- detect conflicts and unsupported guidance
- compare a baseline model with a grounded RAG + tool pipeline
- evaluate reliability, correctness, and evidence quality using a predefined test set

### Mapping to the assignment phases

1. Build the advisor
   - Ingests course, batch, credit, basket, and policy data from official sources
   - Builds a retrieval layer over the curriculum and handbook content
   - Uses a tool-calling LangGraph agent to combine retrieval and structured facts

2. Challenge the advisor
   - Includes synthetic student profiles in `eval/profiles.json`
   - Evaluates edge cases such as missing information, insufficient prerequisites, conflicting policy interpretations, and unavailable courses

3. Improve through prompt engineering
   - Uses a structured prompt with role/context constraints and evidence checks
   - Includes a tool-first pipeline rather than relying on a bare LLM answer
   - Adds verification logic to strip unsupported citations and reduce hallucination

4. Evaluate and measure
   - Uses a predefined evaluation set in `eval/test_cases.json`
   - Measures correctness, hallucination rate, eligibility decisions, evidence correctness, response time, and handling of missing/conflicting information

5. Compare and analyze
   - The README and code structure explicitly describe the progression from basic prompting to RAG and then to RAG + structured student data

6. Deploy and demonstrate
   - The project includes a live hosted app at https://vu-advisor.vercel.app/
   - The FastAPI app exposes a chat interface and voice-input support

7. Documentation and presentation
   - This README captures the architecture, data flow, evaluation methodology, and project explanation in a form suitable for submission and presentation

## How the system works step by step

The project follows a real RAG + tool-calling workflow, not just a simple chat prompt. The pipeline is organized as a series of stages that transform raw university data into a usable academic assistant.

### 1. Source ingestion

The project reads original source material from:
- Excel files for semester spreads, structure details, and minors
- PDF files for the Student Handbook and SOP documents

These inputs are processed by the ingestion layer under `advisor/ingest/` using format-aware adapters. The build process converts each source into a common internal representation called `SourceUnit`, which keeps:
- the source type (table or document section)
- source metadata such as sheet name or page number
- the extracted content itself

This is important because the system is designed to preserve provenance: every answer can be traced back to a workbook cell, PDF page, or curated text section.

### 2. Data normalization and validation

Once the raw content is ingested, the build pipeline cleans and normalizes it into a structured SQLite database.

The builder creates tables for:
- course offerings by batch and semester
- option lists for each course slot
- basket and credit requirements
- minors and minor credits
- academic terms and calendar mapping
- known policy and data inconsistencies as tracked issues

The ingestion script validates the resulting dataset against rules such as:
- every course slot is accounted for
- course totals are consistent across batches
- basket totals match the structure sheets
- minors meet their specified credit totals
- prerequisite relationships and calendar mappings remain coherent

This validation stage is essential because the assistant is meant to answer academic questions with reliable, not speculative, information.

### 3. Knowledge base generation

After validation succeeds, the project builds a search-ready knowledge base under `knowledge_base/` containing:
- a SQLite database with structured academic facts
- curated markdown versions of the Student Handbook and SOP
- chunked document text suitable for retrieval
- embeddings and embedding metadata for semantic search
- validation and provenance reports

This is a hybrid knowledge layer: structured, exact facts are stored in SQLite, while unstructured policy text is chunked and indexed for semantic retrieval.

### 4. Retrieval layer

The retrieval system is implemented in `advisor/retrieval.py` and uses hybrid search.

It combines:
- BM25 lexical matching for exact keywords, course codes, and academic terminology
- dense embedding-based semantic matching for paraphrased or concept-based queries
- reciprocal rank fusion (RRF) to combine both retrieval strategies

This matters because academic questions often mix exact codes like `MATH201` with natural-language wording like “which courses are needed before data structures?” The hybrid search handles both kinds of queries.

### 5. Agent and decision logic

The core reasoning engine lives in `advisor/graph.py` and uses LangGraph.

The agent loop works like this:
1. the user asks a question
2. the system checks for relevant course mentions or profile context
3. the agent decides whether tool calls are needed
4. deterministic tools fetch structured academic facts from SQLite
5. retrieval tools search policy documents and curated text
6. the model produces an answer only from evidence that was actually returned by the tools
7. the verification stage strips unsupported citations and checks for conflicts, missing data, or follow-up questions

This architecture prevents the LLM from inventing facts outside the verified source set.

### 6. Deterministic academic tools

The project does not rely on the model to do all reasoning by itself. Instead, it exposes a set of structured Python tools in `advisor/tools.py` that answer common academic queries, including:
- `lookup_courses`: find courses by batch, semester, bucket, credits, or other filters
- `course_details`: fetch detailed information for a specific course code or title
- `semester_plan`: show the course plan for a given batch and semester
- `credit_structure`: explain basket-wise credit requirements
- `minor_info`: retrieve minor programmes and requirements
- `prerequisite_chain`: show prerequisite dependencies
- `academic_calendar`: map batch and semester to academic time periods
- `check_eligibility`: decide if a student can take a course based on batch, prerequisites, CGPA, and completed courses
- `search_documents`: retrieve policy or curriculum passages from the curated corpus
- `list_data_issues`: surface known inconsistencies or warnings
- `run_sql`: safely query the SQLite database with read-only SQL

These tools are what make the assistant academically grounded instead of simply conversational.

### 7. Web and voice interface

The FastAPI app in `advisor/server.py` exposes:
- a REST API for chat requests
- audio transcription for voice-based input
- preview mode when Azure config is missing
- a frontend interface served from `advisor/static/`

The app also supports synthetic student profiles for evaluation and testing, which lets the assistant answer personalized eligibility questions without exposing real student data.

### 8. Why this architecture is effective

This project combines exact facts and generative reasoning in a controlled way:
- exact facts stay in SQLite and are validated before use
- policy documents and guidance are stored in a retrieval layer
- a tool-calling LLM answers using those sources
- source citations are enforced in the verification stage
- the system can say “not enough information” instead of guessing

That makes it more trustworthy for academic advising, especially in domains where errors can have real consequences.

## Technology stack

### Core backend
- Python 3.x
- FastAPI for the web API
- Uvicorn as the ASGI server
- LangGraph for orchestration of the reasoning workflow
- LangChain Core / LangChain OpenAI for model integration
- Azure OpenAI for chat, embeddings, and speech transcription
- AWS Bedrock Titan embeddings as an alternative embedding backend

### Data and knowledge layer
- SQLite for the structured academic database
- OpenPyXL for Excel parsing
- PyMuPDF-based PDF extraction and text cleanup
- Pandas and NumPy for data handling and vector operations
- BM25 (rank-bm25) for lexical retrieval
- dense vector embeddings for semantic retrieval
- JSONL chunk storage for knowledge retrieval documents

### Frontend and UX
- static HTML/CSS/JS UI located in `advisor/static/`
- assets for branding and supporting visuals
- optional voice transcription flow through Azure speech services

### Validation and testing
- pytest for automated checks
- custom validation logic during the ingestion/build process
- evaluation scripts under `eval/` that benchmark factual, policy, eligibility, conflict, and missing-information questions

### Environment and configuration
- python-dotenv to load environment variables
- `.env.example` contains the required Azure and AWS configuration for chat, embeddings, and transcription

## Repository structure

```text
VU-advisor/
├── advisor/                     # core application logic
│   ├── ingest/                  # format-aware ingestion and build pipeline
│   ├── static/                  # frontend UI assets
│   ├── config.py                # environment and source paths
│   ├── embeddings.py            # embedding provider abstraction
│   ├── graph.py                 # LangGraph orchestration and verification
│   ├── retrieval.py             # hybrid retrieval and ranking logic
│   ├── server.py                # FastAPI API and chat routes
│   ├── tools.py                 # deterministic academic tools
│   └── prompts.py               # system prompt and assistant instructions
├── eval/                        # evaluation test cases and synthetic profiles
├── knowledge_base/              # generated knowledge data and retrieval artifacts
├── assets/                      # branding and visual assets
├── .env.example                 # environment template
├── requirements.txt             # Python dependencies
├── README.md                    # project documentation
├── GenAI_Assignment.pdf          # assignment brief
├── Semester_Spread_Structures_Sept_2026.xlsx
├── MinorCoursesforBTech_Students.xlsx
├── 4. Student Handbook Aug 2026.pdf
├── SOP STUDENT 17082026 - Final.pdf
└── ...
```

## Important limitations and assumptions

- The project uses synthetic student profiles, not real student records, in line with the assignment requirement.
- The system is grounded in the provided academic documents and spreadsheet data only; it does not invent answers beyond those sources.
- Azure OpenAI credentials are required for full chat and transcription functionality. Without them, the application runs in preview mode and displays the retrieval evidence instead of generating a final answer.
- The knowledge base must be rebuilt after source data changes so that SQLite tables and retrieval indexes stay in sync.
- The project is designed for reliability and academic trustworthiness, which means it prefers asking clarifying questions or reporting insufficient information over guessing.

## Quick start

```bash
pip install -r requirements.txt
python -m advisor.ingest.build          # rebuild knowledge_base/ from the 4 source files (fails if validation fails)
python -m pytest tests                  # 10 tests, no credentials needed
cp .env.example .env                    # fill in Azure OpenAI endpoint, key, deployment names
python -m advisor.ingest.build          # re-run once .env is filled: adds dense embeddings (Bedrock Titan v2 or Azure; cached)
uvicorn advisor.server:app              # open http://127.0.0.1:8000
python eval/run_eval.py --judge         # 4-stage evaluation -> eval/results/
```

Without `.env` the web app runs in **preview mode**: it shows the passages retrieval would ground an answer in.

## Data: what is converted into what

| Source | Converted to | Used by |
|---|---|---|
| `Semester_Spread_Structures_Sept_2026.xlsx`: 5 semester-spread grids | `offerings` (262 plan slots) and `options` (282 courses; paired cells such as `MATH401/COMP401` split, with per-course L-T-P taken from the title text) in SQLite | deterministic tools, course cards |
| same workbook: 5 `Struct_*` sheets | `baskets` (credit requirements), `structure_courses` (title-linked to codes) | credit structure, electives |
| `MinorCoursesforBTech_Students.xlsx`: 7 minors | `minors` (incl. TBA-credit placeholders), `minor_batches` (24-credit totals) | minor tool, cards |
| Student Handbook PDF (92 pp) | `knowledge_base/curated/handbook.md`: PyMuPDF plus reviewed patches (grade table image transcribed, formulas, mangled tables); 128 clause/page-aware chunks | hybrid retrieval |
| SOP PDF (6 pp, two-column) | `knowledge_base/curated/sop.md`, verbatim hand transcription with page anchors; 16 chunks | hybrid retrieval |
| derived | `terms` (batch × semester → months; batch year = admission year, Handbook §1.2), `issues` (135 detected conflicts/inconsistencies + 3 curated policy conflicts) | calendar tool, conflict flags |

Every record keeps provenance (`sheet` + cell range, or PDF/printed page). `knowledge_base/VALIDATION_REPORT.md` lists
the 42 build checks, including:

- every grid cell consumed
- each batch totals 180 credits
- basket sums vs Struct sheet
- each minor totals 24 credits
- prerequisite resolution
- ~20 hand-verified golden facts

`knowledge_base/LEGACY_PACKAGE_AUDIT.md` documents why the earlier `Knowledge_package/` was replaced. Its 2023 batch
had semesters 4–6 wrong or missing, and its records carry no batch field.

## Architecture

```
Excel ─┐ adapters → SourceUnits (IR + provenance) → validate → SQLite (exact facts) ─┐
PDF ───┘                                  └→ curated Markdown → chunks ┐          │
                                  SQLite → per-batch cards ────────────┴→ BM25 + dense + RRF
question (text | voice → Azure transcription) → LangGraph: agent (LLM + 11 tools) ⇄ tools → verify → answer + [S#] sources
```

- **Tools** (`advisor/tools.py`): pure Python over read-only SQLite:
  - `lookup_courses`, `course_details`, `semester_plan`, `credit_structure`, `minor_info`
  - `prerequisite_chain`, `academic_calendar`
  - `check_eligibility`: offered to the batch? prerequisites passed or failed? progression CGPA?
  - `list_data_issues`, `search_documents` (hybrid retrieval)
  - `run_sql`: SELECT-only, row-limited
- **Grounding**: the tools node numbers every source S1, S2, …. The `verify` node strips any citation no tool
  produced and computes flags for follow-up, conflict and insufficient information.
- **Adding a format** (DOCX/PPTX/HTML/CSV/OCR): write `extract(path) -> list[SourceUnit]` and register it in
  `advisor/ingest/__init__.py`. Re-ingestion is hash-tracked (`sources.json`), and embeddings are cached per chunk text.
- **Embeddings**: AWS Bedrock `amazon.titan-embed-text-v2:0` (1024-d) when `BEDROCK_EMBED_MODEL` + AWS keys are set, else Azure OpenAI. The model id is stored with the vectors so queries always use the same model; the weak-evidence cut-off (cosine < 0.30) was calibrated on in-scope vs off-topic probes.
- **Deliberately not used**: Neo4j/GraphRAG/RAPTOR, a reranker, multi-agent orchestration. The corpus is about
  700 chunks, so an in-memory index and a prerequisite dict answer every query in the eval set.

## Evaluation

`eval/test_cases.json` has 30 cases:

- 8 factual
- 6 policy
- 6 eligibility with synthetic profiles
- 4 missing-information
- 4 conflict
- 1 ambiguous
- 1 out-of-scope

`eval/run_eval.py` runs each case through 4 stages: Basic LLM → structured prompt → RAG → RAG + tools + student data.

- **Rule-based labels**: correct / partial / hallucinated / incorrect, from required, forbidden and behaviour patterns per case. `--judge` adds an LLM grade against the expected outcome.
- **Metrics per stage**: accuracy, hallucination rate, eligibility correctness, incorrect recommendations, source correctness, missing-info and conflict handling, average and maximum latency.

Synthetic students live in `eval/profiles.json`. They contain no real names or IDs.
