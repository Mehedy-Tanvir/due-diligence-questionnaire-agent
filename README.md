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
