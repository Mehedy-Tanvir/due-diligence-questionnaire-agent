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
