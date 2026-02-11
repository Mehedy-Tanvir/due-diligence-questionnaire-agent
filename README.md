# Due Diligence Questionnaire Agent (Skeleton Design)

## Overview

The Due Diligence Questionnaire Agent is a system designed to automate the process of answering due diligence and compliance questionnaires using a company’s internal documents.

Users upload reference documents such as policies, reports, and financial materials, along with a questionnaire file containing structured or semi-structured questions. The system parses the questionnaire, retrieves relevant information from the uploaded documents, and generates draft answers for each question.
Each generated answer includes:

- An explicit indication of whether the question can be answered from the available documents
- Citations pointing to the exact document sections used
- A confidence score representing the system’s certainty

The system supports human review and manual overrides, preserving both AI-generated and human-edited answers for auditability. It also provides an evaluation mechanism to compare AI answers against human ground-truth responses.

This document describes the system design, data models, workflows, and tradeoffs at a skeleton level. It focuses on clarity and correctness rather than production implementation details.

---

## End-to-End Workflow

The system follows a clear, asynchronous workflow from document ingestion to answer evaluation:

1. **Document Upload**  
   Users upload reference documents such as PDFs, Word files, spreadsheets, or presentations. Each document is associated with an organization and stored with basic metadata (name, type, upload time).

2. **Document Ingestion & Indexing**  
   Uploaded documents are parsed to extract text and structural information. The extracted content is split into manageable chunks and indexed for retrieval. A multi-layer index is maintained to support both semantic search and precise citation at the document section or page level.

3. **Questionnaire Upload & Parsing**  
   Users upload a questionnaire file containing sections and questions. The system parses the file into a structured representation, preserving question order and section hierarchy.

4. **Project Creation**  
   A questionnaire project is created linking the uploaded questionnaire with a selected document scope (specific documents or ALL_DOCS). Project creation and preparation occur asynchronously, with status updates exposed to the user.

5. **Answer Generation (Asynchronous)**  
   For each parsed question, the system retrieves relevant document chunks, determines whether the question can be answered, and generates a draft answer. Each answer includes supporting citations and a confidence score.

6. **Human Review & Manual Overrides**  
   Generated answers are presented for human review. Reviewers can confirm, reject, or manually edit answers. Manual responses are stored alongside AI-generated outputs, preserving an audit trail.

7. **Evaluation Against Ground Truth**  
   When ground-truth answers are available, the system evaluates AI-generated answers against human responses and produces quantitative scores and qualitative feedback.

8. **Regeneration on Change**  
   If new documents are uploaded or project configuration changes, affected projects are marked as outdated and eligible for regeneration, ensuring answers remain consistent with the latest data.

---

## Core Data Models & Statuses

This section describes the primary entities in the system and their lifecycle states. The models are conceptual and intended to illustrate structure and behavior rather than implementation details.

### Core Entities

**Project**

- id
- name
- questionnaireId
- documentScope (specific document IDs or ALL_DOCS)
- status
- createdAt
- updatedAt

A Project represents a single questionnaire run against a defined set of documents.

---

**Document**

- id
- filename
- fileType (PDF, DOCX, XLSX, PPTX)
- uploadTime
- ingestionStatus
- metadata (page count, source, etc.)

Documents serve as the source of truth for answer generation and citation.

---

**Questionnaire**

- id
- sourceFile
- sectionCount
- questionCount
- createdAt

Represents the uploaded questionnaire file and its parsed structure.

---

**Question**

- id
- questionnaireId
- sectionTitle
- orderIndex
- text

Each Question belongs to a questionnaire and preserves its original ordering and section context.

---

**Answer**

- id
- questionId
- projectId
- answerText
- answerable (true / false / partial)
- confidenceScore
- status
- citations
- createdAt
- updatedAt

Stores the AI-generated answer and its review state.

---

**Citation**

- documentId
- chunkId
- pageNumber
- boundingBox (optional)
- excerpt

Citations link specific answer statements to exact locations in source documents.

---

**EvaluationResult**

- id
- answerId
- groundTruthText
- similarityScore
- explanation
- evaluatedAt

Represents the comparison between AI-generated answers and human ground-truth responses.

---

### Status & State Transitions

**Project Status**

- CREATED
- INDEXING
- READY
- OUTDATED
- COMPLETED

Allowed transitions:

- CREATED → INDEXING → READY
- READY → OUTDATED (on document or configuration change)
- READY → COMPLETED

---

**Answer Status**

- GENERATED
- CONFIRMED
- REJECTED
- MANUAL_UPDATED
- MISSING_DATA

Allowed transitions:

- GENERATED → CONFIRMED
- GENERATED → REJECTED
- GENERATED → MANUAL_UPDATED
- GENERATED → MISSING_DATA
- MANUAL_UPDATED → CONFIRMED

---

## Document Ingestion & Indexing

Uploaded documents are processed to extract content and organize it for efficient retrieval. The system supports multiple file formats and maintains a structured index to enable both semantic search and precise citations.

### Multi-Format Parsing

- **Supported formats:** PDF, DOCX, XLSX, PPTX
- **Parsing approach:**
  - Extract textual content and structural metadata (headings, tables, bullet points)
  - Retain page numbers, table coordinates, and other location references
  - Normalize text for consistent indexing (remove formatting noise, handle line breaks, etc.)

### Chunking

- Documents are split into manageable chunks to improve retrieval efficiency:
  - Typical chunk size: 300–500 words (configurable)
  - Each chunk includes:
    - Source document ID
    - Page number
    - Chunk sequence index
    - Optional bounding boxes or table coordinates for precise citation

### Two-Layer Index

1. **Answer Retrieval Layer**
   - Supports semantic search across sections or chunks
   - Optimized for quickly finding candidate chunks relevant to a question

2. **Citation Layer**
   - Stores chunk-level references with exact positions (page, table, paragraph)
   - Enables precise citation in generated answers

### ALL_DOCS Invalidation

- Projects configured to use `ALL_DOCS` automatically become `OUTDATED` when new documents are ingested
- Ensures subsequent answer generation considers the latest documents

### Status Flow

- **Document ingestion status:** PENDING → PARSING → INDEXED → ERROR
- **Project impact:** ALL_DOCS projects marked OUTDATED → eligible for regeneration

---

## Questionnaire Parsing & Project Lifecycle

Questionnaires define the set of questions to be answered against a selected document scope. The system supports structured and semi-structured questionnaire formats and preserves their original organization and ordering.

### Questionnaire Parsing

Questionnaires may be uploaded in various formats (e.g. DOCX, XLSX, PDF). The parsing process extracts a structured representation while retaining contextual information.

Parsing responsibilities include:

- Detecting sections and subsections
- Extracting individual questions
- Preserving original question order and section hierarchy
- Normalizing question text for downstream processing

Each parsed questionnaire is stored as a `Questionnaire` entity, with individual `Question` entities created for each extracted question.

### Handling Complex Question Structures

The parser supports common due diligence patterns, including:

- Multi-part questions (e.g. 1.a, 1.b, 1.c)
- Binary (Yes/No) questions with follow-up explanations
- Free-text and table-based question layouts

Multi-part questions are either:

- Represented as separate `Question` entities linked by shared section context, or
- Treated as a single composite question when answers are expected to be provided together

The chosen representation is configurable at project creation time.

### Project Creation

A `Project` represents a single questionnaire execution against a defined document scope.

During project creation, the user specifies:

- A questionnaire
- A document scope (specific document IDs or `ALL_DOCS`)

Project creation is asynchronous and progresses through defined lifecycle states.

### Project Preparation & Lifecycle

Once created, a project undergoes the following steps:

1. **CREATED**  
   Project metadata is stored and validation checks are performed.

2. **INDEXING**  
   The system verifies that all required documents are fully ingested and indexed.  
   If indexing is incomplete, the project remains in this state until dependencies are resolved.

3. **READY**  
   The project is eligible for answer generation.

4. **OUTDATED**  
   The project is marked outdated when:
   - New documents are ingested and the scope is `ALL_DOCS`
   - The document scope or questionnaire configuration changes

5. **COMPLETED**  
   All questions have generated answers and the project has passed initial processing.

### Automatic Regeneration Triggers

Projects in the `OUTDATED` state are eligible for regeneration. Regeneration may be triggered:

- Automatically upon document ingestion (for `ALL_DOCS` projects)
- Manually by a user
- Programmatically via an API

Regeneration reuses the existing questionnaire structure while re-running answer generation using the latest document index.

### Status Visibility & Error Handling

Project status updates are exposed to users to support transparency in asynchronous processing.

If parsing or preparation fails:

- The project is moved to an error state
- Error details are recorded for debugging and user feedback

---

## Answer Generation with Citations & Confidence

Answer generation is performed asynchronously for each question in a project once the project reaches the `READY` state. The process combines document retrieval, answerability assessment, and answer synthesis to produce auditable draft responses.

### Retrieval Strategy

For each question, the system retrieves relevant document chunks from the indexed corpus using the answer retrieval layer.

Retrieval behavior includes:

- Semantic search across chunk embeddings
- Optional keyword or section-based filtering
- Limiting results to the project’s document scope

The top-K most relevant chunks are selected as candidate evidence for answer generation.

### Answerability Assessment

Before generating an answer, the system determines whether the question can be answered using the retrieved content.

Each question is classified as:

- **Answerable (TRUE):** sufficient evidence exists to support a complete answer
- **Partially Answerable (PARTIAL):** some relevant information exists, but gaps remain
- **Not Answerable (FALSE):** no relevant content is found in the document corpus

This classification is based on:

- Retrieval confidence scores
- Coverage of required facts or entities
- Consistency across retrieved chunks

### Answer Synthesis

If a question is classified as answerable or partially answerable, the system generates a draft answer grounded strictly in the retrieved content.

Generation rules include:

- Use only retrieved chunks as source material
- Avoid introducing information not present in the documents
- Explicitly state limitations or missing data for partial answers

If a question is not answerable, the system generates a standardized response indicating that the required information is not available in the uploaded documents.

### Citation Generation

Each generated answer includes one or more citations linking specific statements to their source document locations.

Citations reference:

- Document ID
- Chunk ID
- Page number or section identifier
- Optional bounding box or table coordinates

Citations are attached at the answer level and may also be associated with specific answer segments for finer-grained traceability.

### Confidence Scoring

A confidence score is computed for each generated answer to represent the system’s certainty.

The confidence score is derived from:

- Retrieval relevance scores
- Answerability classification
- Consistency of supporting evidence

Confidence scores are normalized to a fixed range and are intended to guide human reviewers rather than serve as absolute correctness guarantees.

### Failure & Fallback Behavior

If retrieval returns no relevant chunks:

- The question is marked as not answerable
- The answer status is set to `MISSING_DATA`
- Confidence score is set to a minimal value

Errors during generation are captured and surfaced through answer status and logs, without blocking processing of other questions.

---

## Review Workflow & Manual Overrides

The system includes a structured human-in-the-loop review workflow to ensure correctness, compliance, and auditability.

AI-generated answers are never treated as final outputs without optional human validation.

### Review States

Each `Answer` progresses through defined review states:

- GENERATED
- CONFIRMED
- REJECTED
- MANUAL_UPDATED
- MISSING_DATA

### Review Actions

Reviewers can perform the following actions:

1. **Confirm**
   - Marks the answer as `CONFIRMED`
   - Indicates the AI answer is acceptable without changes

2. **Reject**
   - Marks the answer as `REJECTED`
   - Used when the AI output is incorrect or irrelevant

3. **Manual Update**
   - Reviewer edits or replaces the answer
   - Original AI answer is preserved
   - Status becomes `MANUAL_UPDATED`

4. **Mark as Missing Data**
   - Used when required information truly does not exist in documents
   - Status becomes `MISSING_DATA`

### Audit & Versioning Model

To ensure auditability:

- The original AI-generated answer is never overwritten
- Manual edits are stored as a separate version linked to the same `Answer`
- Timestamps and reviewer identity are recorded
- Status transitions are logged

This allows:

- Comparison of AI vs. human answers
- Tracking review decisions
- Regulatory audit support

### Review Workflow

1. Project reaches `READY`
2. Answers are generated asynchronously
3. Reviewer opens Project Detail view
4. Each question is reviewed individually
5. Final project may move to `COMPLETED` once review is finished

---

## Evaluation Framework

The system supports evaluation of AI-generated answers against human ground-truth responses.

Evaluation is optional but required for benchmarking and quality measurement.

### Evaluation Inputs

- AI-generated answer
- Human ground-truth answer
- Associated citations

### Evaluation Methodology

Evaluation combines multiple comparison techniques:

1. **Semantic Similarity**
   - Embedding-based similarity between AI and human answers
   - Captures meaning equivalence even if wording differs

2. **Keyword / Entity Overlap**
   - Checks presence of critical entities, dates, numbers, and obligations
   - Ensures factual completeness

3. **Structural Completeness**
   - Verifies whether required sub-parts of the question are addressed

### Evaluation Outputs

Each `EvaluationResult` includes:

- similarityScore (normalized numeric score)
- explanation (qualitative reasoning)
- coverage indicators (optional structured metrics)
- evaluatedAt timestamp

### Reporting

Evaluation results can be aggregated:

- Per Question
- Per Section
- Per Project

Example aggregate metrics:

- Average similarity score
- Percentage of CONFIRMED answers
- Percentage of MISSING_DATA answers
- Partial vs fully answerable distribution

Evaluation reports support:

- Model improvement
- QA benchmarking
- Transparency in AI performance

---

## Optional Chat Extension

The system may optionally provide a chat interface that uses the same indexed document corpus.

### Shared Retrieval Layer

Chat uses:

- The same document ingestion pipeline
- The same chunk-level citation index
- The same semantic retrieval system

This ensures consistency between:

- Questionnaire answers
- Chat-based queries

### Chat Constraints

To avoid conflicting with questionnaire workflows:

- Chat does not modify Project data
- Chat responses are not stored as official questionnaire answers
- Chat responses include citations
- Chat responses include confidence indicators

Chat is read-only with respect to questionnaire state.

---
