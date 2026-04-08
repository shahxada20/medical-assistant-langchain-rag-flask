# MediAssist: AI-Powered Medical RAG Chatbot

[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-active-success.svg)]()

A production-grade Retrieval-Augmented Generation (RAG) system for clinical information retrieval, featuring a web-based chat interface backed by a Flask REST API.

---

## Table of Contents

- [Overview](#overview)
- [How RAG Works](#how-rag-works)
- [System Architecture](#system-architecture)
- [Tech Stack](#tech-stack)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [ETL Pipeline](#etl-pipeline)
- [API Reference](#api-reference)
- [Frontend](#frontend)
- [Deployment](#deployment)
- [Future Roadmap](#future-roadmap)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

MediAssist is an end-to-end AI infrastructure project designed to demonstrate competencies in **Data Engineering**, **MLOps**, and **AI Infrastructure**. The system processes medical literature through a robust ETL pipeline, embeds domain knowledge into a vector database, and serves clinical queries via a web-based chat interface with responses grounded in retrieved evidence.

### Key Capabilities

- **Evidence-Based Responses**: All answers are grounded in retrieved medical context; the system explicitly declines to speculate beyond available information
- **Multi-Modal Response Modes**: Dynamic routing between DIRECT, PARTIAL, FALLBACK, and REDIRECT response strategies based on retrieval confidence
- **Clinical-Grade Prompt Engineering**: Systematic response protocols with explicit handling for contraindications, dosages, and diagnostic uncertainty
- **Web Chat Interface**: Bootstrap-based responsive UI for interactive clinical queries
- **Production-Ready Architecture**: Modular design with separation of concerns between ingestion, retrieval, and generation components

---

## How RAG Works

Retrieval-Augmented Generation (RAG) is an AI architecture that combines information retrieval with generative language models to produce accurate, context-grounded responses.

### Architecture Flow

```
┌───────────────────────────────────────────────────────────────────────┐
│                              RAG PIPELINE                             │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────┐     ┌───────────┐     ┌─────────┐     ┌──────────┐      │
│  │ Document │ ──► │ Retrieval │ ──► │ Context │ ──► │ Generate │      │
│  │  Input   │     │  Engine   │     │         │     │ Response │      │
│  └──────────┘     └───────────┘     └─────────┘     └──────────┘      │
│                      │                   │                 │          │
│                      ▼                   ▼                 ▼          │
│                 ┌───────────┐      ┌────────────┐    ┌─────────┐      │
│                 │ Vector DB │ ────►│  Medical   │    │ Groq    │      │
│                 │           │      │  Context   │    │ LLM     │      │
│                 └───────────┘      └────────────┘    └─────────┘      │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Response Generation Protocol

1. **Parse**: Extract clinical entities (symptoms, conditions, medications) from user query
2. **Retrieve**: Query Pinecone vector store for semantically similar document chunks (top-k=3)
3. **Assess**: Evaluate retrieval relevance (DIRECT, PARTIAL, or FALLBACK confidence)
4. **Generate**: Condition LLM response on retrieved context with strict anti-fabrication constraints

---

## System Architecture

### High-Level Component Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                               │
│       Chat UI (templates/chat.html + static/style.css)               │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                                │
│                 Flask Backend (app.py)                               │
│       Routes: "/" (render UI) | "/get" (POST - query handler)        │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        RAG PIPELINE LAYER                            │
│                           src/helper.py                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │
│  │  Document Load  │  │  Text Splitter  │  │  Embedding Model│       │
│  │   (PyPDF)       │  │ (RecursiveChar) │  │ (MiniLM-L6-v2)  │       │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘       │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐                            │
│  │  RAG Chain      │  │  System Prompt  │                            │
│  │  (LCEL)         │  │  (Clinical)     │                            │
│  └─────────────────┘  └─────────────────┘                            │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        STORAGE LAYER                                 │
│                    Pinecone Vector Database                          │
│              Index: medical-assistant | Dim: 384 | Metric: cosine    │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        GENERATION LAYER                              │
│                    Groq Llama 4 (via API)                            │
│              Temperature: 0 | System Prompt: Clinical Protocol       │
└──────────────────────────────────────────────────────────────────────┘
```

### ETL Pipeline Stages

| Stage | Function | Component |
|-------|----------|-----------|
| **Extract** | `load_pdf()` | Ingest medical PDFs from `Data/` directory using PyPDF |
| **Transform** | `filter_documents()` | Strip metadata, retain page content only |
| **Chunk** | `split_doc_into_chunks()` | Recursive character splitting (500/50 overlap) |
| **Embed** | `load_embeddings_model()` | HuggingFace sentence-transformers (384-dim) |
| **Load** | `store_embeddings()` | Upsert vectors to Pinecone index |

---

## Tech Stack

### Core Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Language** | Python 3.12+ | Primary development language |
| **Environment** | uv | Fast Python package management |
| **Containerization** | Docker | Planned for production deployment |

### AI/ML Stack

| Component | Technology | Configuration |
|-----------|------------|---------------|
| **LLM Provider** | Groq API | Llama 4 Scout (17B) |
| **Embeddings** | HuggingFace | sentence-transformers/all-MiniLM-L6-v2 |
| **Vector Database** | Pinecone | Serverless, 384 dimensions, cosine similarity |
| **Orchestration** | LangChain | LCEL (LangChain Expression Language) |

### Backend & API

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Web Framework** | Flask 3.1+ | RESTful API server + template rendering |
| **Environment** | python-dotenv | Configuration management |
| **Document Processing** | PyPDF, langchain-community | PDF ingestion |

### Frontend

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Styling** | Custom CSS | Dark theme, chat bubble styling |

---

## Installation

### Prerequisites

- Python 3.12 or higher
- [uv](https://github.com/astral-sh/uv) package manager
- Pinecone API key ([signup](https://www.pinecone.io/))
- Groq API key ([signup](https://console.groq.com/))

### Step 1: Clone Repository

```bash
git clone https://github.com/<username>/medical-assistant-chatbot.git
cd medical-assistant-chatbot
```

### Step 2: Initialize Environment

```bash
# Create virtual environment
uv venv

# Activate environment (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Activate environment (Unix/bash)
source .venv/bin/activate
```

### Step 3: Install Dependencies

```bash
# Sync dependencies from lock file
uv sync

# Or install from requirements.txt
pip install -r requirements.txt
```

### Step 4: Configure Environment Variables

Create a `.env` file in the project root:

```bash
# .env
PINECONE_API_KEY=pcsk_<your_api_key>
GROQ_API_KEY=gsk_<your_api_key>
HF_TOKEN=hf_<your_huggingface_token>  # Optional, for rate-limited models
```

---

## Configuration

### Default Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DEFAULT_EMBEDDING_MODEL` | sentence-transformers/all-MiniLM-L6-v2 | Embedding model for document vectors |
| `DEFAULT_LLM_MODEL` | meta-llama/llama-4-scout-17b-16e-instruct | Groq LLM for response generation |
| `DEFAULT_INDEX_NAME` | medical-assistant | Pinecone index identifier |
| `DEFAULT_INDEX_DIMENSION` | 384 | Vector dimensionality |
| `CHUNK_SIZE` | 500 | Text splitting chunk size |
| `CHUNK_OVERLAP` | 50 | Text splitting overlap |
| `RETRIEVAL_K` | 3 | Top-k documents for retrieval |

### Pinecone Index Specification

```python
ServerlessSpec(
    cloud="aws",
    region="us-east-1",
    metric="cosine",
    dimension=384
)
```

---

## Usage

### Step 1: Build the Vector Index

Before running the application, populate the Pinecone index with medical documents:

```bash
# Run the ETL pipeline to ingest PDFs from Data/
python store_index.py
```

This will:
- Load PDFs from the `Data/` directory
- Split documents into chunks
- Generate embeddings
- Store vectors in Pinecone

### Step 2: Run the Flask Application

```bash
# Start the web server (default: http://localhost:5000)
python app.py
```

### Step 3: Access the Chat Interface

Open your browser and navigate to:

```
http://localhost:5000
```

### Programmatic Usage

```python
from src.helper import (
    get_existing_vector_store,
    initialize_groq_llm,
    system_prompt,
    build_rag_chain,
    load_embeddings_model
)

# Load components
embedding = load_embeddings_model()
vector_store = get_existing_vector_store(embedding, "medical-assistant")
llm = initialize_groq_llm()
prompt = system_prompt()

# Build and execute chain
chain = build_rag_chain(vector_store, llm, prompt)
response = chain.invoke("What are the side effects of low iron?")

print(response)
```

---

## API Reference

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Render chat interface (templates/chat.html) |
| `POST` | `/get` | Submit clinical query and receive response |

### Request/Response Schema

#### POST /get

**Request (Form Data):**
```
msg: What are the symptoms of vitamin D deficiency?
```

**Response (JSON - raw string):**
```json
"Vitamin D deficiency may cause the following symptoms:\n- Bone pain and tenderness\n- Muscle weakness\n- Increased fracture risk..."
```

### Error Responses

| Status Code | Response | Description |
|-------------|----------|-------------|
| 400 | `{"error": "Empty message"}` | No message provided in request |
| 503 | `{"error": "Service unavailable"}` | RAG chain not initialized |
| 500 | `{"error": "Failed to generate response"}` | Internal error during generation |

---

## Frontend

### Chat Interface

The frontend is a single-page chat application built with:

- **Bootstrap 4.1**: Responsive layout and components
- **jQuery 3.3**: DOM manipulation and AJAX requests
- **Custom CSS**: Dark gradient theme with chat bubble styling

### File Structure

```
templates/
  └── chat.html        # Main chat interface
static/
  └── style.css        # Custom styling
```

### Features

- Real-time message display with timestamps
- Distinct styling for user vs. bot messages
- Responsive design for mobile/desktop
- Auto-scroll to latest message

---

## Deployment

> **Note**: Production deployment configuration is under development. This section will be updated with AWS infrastructure details.

### Planned Infrastructure

- **Compute**: AWS ECS Fargate / Lambda
- **Load Balancing**: Application Load Balancer (ALB)
- **Secrets Management**: AWS Secrets Manager
- **Monitoring**: CloudWatch Logs + X-Ray tracing

### Docker (Planned)

```dockerfile
# Multi-stage build for minimal production image
# TODO: Implement Dockerfile
```

### Environment Variables (Production)

| Variable | Source |
|----------|--------|
| `PINECONE_API_KEY` | AWS Secrets Manager |
| `GROQ_API_KEY` | AWS Secrets Manager |
| `FLASK_ENV` | Production |
| `LOG_LEVEL` | INFO/WARNING |

---

## Future Roadmap

### Q2 2026

- [ ] **AWS Deployment**: Containerized deployment on ECS with ALB routing
- [ ] **CI/CD Pipeline**: GitHub Actions for automated testing and deployment
- [ ] **Health Check Endpoint**: Add `/api/health` for monitoring
- [ ] **Dockerfile**: Multi-stage build for production image

### Q3 2026

- [ ] **Expanded Medical Datasets**:
  - PubMed Central Open Access subset
  - MIMIC-III clinical notes (de-identified)
  - DrugBank pharmacological data
- [ ] **Multi-Model Support**: Ollama integration for local LLM fallback
- [ ] **Caching Layer**: Redis for query result caching
- [ ] **Ingestion Endpoint**: Add `/api/ingest` for dynamic PDF upload

### Q4 2026

- [ ] **Evaluation Framework**: RAGAS metrics for retrieval/generation quality
- [ ] **Observability**: LangSmith tracing for pipeline debugging
- [ ] **Authorization**: JWT-based API authentication

---

## Project Structure

```
medical-assistant-chatbot/
├── .env                          # Environment variables (API keys)
├── .gitignore                    # Git ignore patterns
├── .python-version               # Python version specification (3.12+)
├── app.py                        # Flask application entry point
├── main.py                       # CLI placeholder
├── pyproject.toml                # Project dependencies (uv)
├── requirements.txt              # Python dependencies (pip format)
├── uv.lock                       # Dependency lock file
├── README.md                     # Project documentation
├── store_index.py                # ETL pipeline CLI - builds Pinecone index
│
├── Data/                         # Source medical PDFs
│   └── Medical_book.pdf
│
├── templates/                    # Flask templates
│   └── chat.html                 # Chat interface
│
├── static/                       # Static assets
│   └── style.css                 # Chat styling
│
├── src/                          # Core application logic
│   ├── __init__.py
│   ├── helper.py                 # ETL pipeline + RAG utilities
│   └── prompt.py                 # Clinical system prompts
│
└── notebooks/                    # Jupyter notebooks for experimentation
    └── trials.ipynb
```

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Code Standards

- **Style**: PEP 8 compliant
- **Naming**: `snake_case` (functions/variables), `PascalCase` (classes)
- **Error Handling**: Try-except blocks for all external API calls
- **Security**: No hardcoded credentials; use `os.getenv()` exclusively

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Acknowledgments

- [LangChain](https://python.langchain.com/) for orchestration framework
- [Groq](https://groq.com/) for low-latency LLM inference
- [Pinecone](https://www.pinecone.io/) for vector database infrastructure
- [HuggingFace](https://huggingface.co/) for embedding models

---

**MediAssist** - Clinical AI Infrastructure for Evidence-Based Medical Information Retrieval
