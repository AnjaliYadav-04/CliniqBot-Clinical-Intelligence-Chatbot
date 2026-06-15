# CliniqBot — Clinical Intelligence Chatbot

> An AI-powered medical Q&A chatbot that ingests clinical knowledge from PDF documents, indexes it in a vector database, and answers patient or clinician queries using a Retrieval-Augmented Generation (RAG) pipeline.

[![Python](https://img.shields.io/badge/Python-3.10-blue)](https://www.python.org/)
[![Flask](https://img.shields.io/badge/Flask-3.1.1-lightgrey)](https://flask.palletsprojects.com/)
[![LangChain](https://img.shields.io/badge/LangChain-0.3.26-green)](https://langchain.com/)
[![Pinecone](https://img.shields.io/badge/Pinecone-Serverless-purple)](https://www.pinecone.io/)
[![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o-orange)](https://openai.com/)
[![Docker](https://img.shields.io/badge/Docker-Containerised-blue)](https://www.docker.com/)
[![AWS](https://img.shields.io/badge/AWS-ECR%20%2B%20EC2-yellow)](https://aws.amazon.com/)

---

## Table of Contents

- [System Architecture](#system-architecture)
- [Workflow Diagram](#workflow-diagram)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [File Structure](#file-structure)
- [Installation & Setup](#installation--setup)
- [How to Run](#how-to-run)
- [AWS CI/CD Deployment](#aws-cicd-deployment)

---

## System Architecture

CliniqBot is built on a **Retrieval-Augmented Generation (RAG)** architecture. It separates the system into two distinct phases:

**Phase 1 — Offline Indexing (`store_index.py`)**
Medical knowledge PDFs are loaded, chunked, embedded using a HuggingFace sentence-transformer model, and stored in a Pinecone vector index. This is a one-time setup step.

**Phase 2 — Online Inference (`app.py`)**
At runtime, the Flask web server receives a user query. LangChain's retrieval chain fetches the top-3 semantically similar chunks from Pinecone, then passes them as context to GPT-4o via a structured `ChatPromptTemplate`. The model returns a concise, grounded answer.

### Core Pipeline Components

| Component | Technology | Role |
|---|---|---|
| PDF Loader | `PyPDFLoader` + `DirectoryLoader` (LangChain) | Loads all `.pdf` files from `data/` |
| Text Splitter | `RecursiveCharacterTextSplitter` (LangChain) | Splits docs into 500-token chunks with 20-token overlap |
| Embedding Model | `sentence-transformers/all-MiniLM-L6-v2` (HuggingFace) | Generates 384-dimensional dense vectors |
| Vector Store | Pinecone Serverless (cosine similarity, AWS `us-east-1`) | Stores and retrieves embeddings at scale |
| LLM | `gpt-4o` via `langchain-openai` | Generates answers grounded in retrieved context |
| RAG Chain | `create_retrieval_chain` + `create_stuff_documents_chain` | Chains retriever → prompt → LLM → answer |
| Web Interface | Flask 3.1.1 + jQuery AJAX | Serves the chat UI and handles `/get` POST requests |

---

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 1 — OFFLINE INDEXING                   │
│                      (run store_index.py once)                  │
│                                                                  │
│   data/*.pdf                                                     │
│       │                                                          │
│       ▼                                                          │
│   DirectoryLoader + PyPDFLoader                                 │
│       │  Load all PDF pages as LangChain Document objects       │
│       ▼                                                          │
│   filter_to_minimal_docs()                                       │
│       │  Retain page_content + source metadata only            │
│       ▼                                                          │
│   RecursiveCharacterTextSplitter                                 │
│       │  chunk_size=500, chunk_overlap=20                       │
│       ▼                                                          │
│   HuggingFaceEmbeddings                                          │
│       │  sentence-transformers/all-MiniLM-L6-v2 (384-dim)      │
│       ▼                                                          │
│   PineconeVectorStore.from_documents()                           │
│       │  Upserts all chunk vectors → "medical-chatbot" index    │
│       ▼                                                          │
│   Pinecone Serverless Index (AWS us-east-1, cosine metric)      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 2 — ONLINE INFERENCE                   │
│                        (Flask web server)                        │
│                                                                  │
│   User types query in chat.html                                  │
│       │  jQuery AJAX POST → /get                                │
│       ▼                                                          │
│   Flask /get route receives msg                                  │
│       │                                                          │
│       ▼                                                          │
│   PineconeVectorStore.as_retriever()                             │
│       │  search_type="similarity", k=3                          │
│       │  Fetches top-3 matching document chunks                 │
│       ▼                                                          │
│   ChatPromptTemplate                                             │
│       │  System prompt: "You are a Medical assistant…"          │
│       │  Injects retrieved {context} + user {input}             │
│       ▼                                                          │
│   ChatOpenAI (gpt-4o)                                            │
│       │  Generates concise grounded answer (≤3 sentences)       │
│       ▼                                                          │
│   Response returned to chat.html → displayed as bot message     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Features

- **PDF knowledge ingestion** — Loads an entire directory of medical PDFs using LangChain's `DirectoryLoader` + `PyPDFLoader`. No manual document handling required.
- **Intelligent chunking** — `RecursiveCharacterTextSplitter` with 500-token chunks and 20-token overlap preserves sentence boundaries and contextual continuity across splits.
- **Lightweight local embeddings** — `sentence-transformers/all-MiniLM-L6-v2` runs locally via HuggingFace with no external API call, producing compact 384-dimensional vectors optimised for semantic similarity.
- **Serverless vector search** — Pinecone Serverless index with cosine similarity retrieves the top-3 most relevant chunks in milliseconds, auto-scaling with no infrastructure management.
- **GPT-4o answers** — The retrieved context is injected into a structured `ChatPromptTemplate` and passed to GPT-4o, which produces concise, evidence-grounded responses of no more than three sentences.
- **Grounded responses only** — The system prompt instructs the model to explicitly say "I don't know" if the answer is not present in the retrieved context, preventing hallucination.
- **Flask + jQuery chat UI** — A clean, dark-themed chat interface served by Flask. Messages are sent and received via AJAX with no page reload.
- **Docker-ready** — A `Dockerfile` packages the entire application for container-based deployment.
- **AWS ECR + EC2 deployment** — Containerised image is pushed to Elastic Container Registry and pulled onto an EC2 Ubuntu instance for production serving.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.10 |
| Web framework | Flask 3.1.1 |
| LLM orchestration | LangChain 0.3.26, langchain-community 0.3.26 |
| LLM | OpenAI GPT-4o (`langchain-openai 0.3.24`) |
| Embeddings | HuggingFace `sentence-transformers/all-MiniLM-L6-v2` (`sentence-transformers 4.1.0`) |
| Vector database | Pinecone Serverless (`langchain-pinecone 0.2.8`) |
| PDF parsing | PyPDF (`pypdf 5.6.1`) |
| Environment config | `python-dotenv 1.1.0` |
| Frontend | HTML5, Bootstrap 4, jQuery 3.3.1 |
| Containerisation | Docker (`python:3.10-slim-buster`) |
| Cloud deployment | AWS ECR + EC2 (Ubuntu) |
| CI/CD | GitHub Actions (self-hosted runner on EC2) |

---

## File Structure

```
CliniqBot-Clinical-Intelligence-Chatbot/
│
├── app.py                    # Flask server — RAG chain setup and /get endpoint
├── store_index.py            # One-time script: load PDFs → chunk → embed → upsert to Pinecone
├── setup.py                  # Package metadata (setuptools)
├── requirements.txt          # Python dependencies
├── Dockerfile                # Container definition (python:3.10-slim-buster)
├── template.sh               # Shell script to scaffold the project directory structure
├── .env                      # API keys (not committed — see below)
│
├── src/
│   ├── __init__.py
│   ├── helper.py             # load_pdf_file(), filter_to_minimal_docs(), text_split(), download_hugging_face_embeddings()
│   └── prompt.py             # system_prompt string for the RAG chain
│
├── templates/
│   └── chat.html             # Flask Jinja2 template — Bootstrap chat UI with jQuery AJAX
│
├── static/
│   └── style.css             # Dark gradient chat interface styling
│
└── research/
    └── trials.ipynb          # Jupyter notebook — development and experimentation log
```

---

## Installation & Setup

### Step 0 — Clone the repository

```bash
git clone https://github.com/AnjaliYadav-04/CliniqBot-Clinical-Intelligence-Chatbot
cd CliniqBot-Clinical-Intelligence-Chatbot
```

### Step 1 — Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate        # Linux / macOS
.venv\Scripts\activate           # Windows
```

### Step 2 — Install dependencies

```bash
pip install -r requirements.txt
```

### Step 3 — Configure environment variables

Create a `.env` file in the root directory:

```env
PINECONE_API_KEY=your_pinecone_api_key_here
OPENAI_API_KEY=your_openai_api_key_here
```

> **Note:** The project uses OpenAI (GPT-4o) and Pinecone only. No Groq, LlamaCloud, or other LLM API keys are required.

### Step 4 — Add your medical PDF knowledge base

Place your medical reference PDF files inside a `data/` directory at the project root:

```bash
mkdir data
# Copy your PDFs into data/
```

### Step 5 — Index your documents into Pinecone

Run this once to chunk, embed, and upsert all PDFs into your Pinecone serverless index:

```bash
python store_index.py
```

This will:
1. Load all `*.pdf` files from `data/`
2. Split them into 500-token chunks with 20-token overlap
3. Embed each chunk using `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions)
4. Create a Pinecone serverless index named `medical-chatbot` (if it doesn't already exist)
5. Upsert all chunk vectors into the index

---

## How to Run

### Option 1 — Flask web interface (recommended)

```bash
python app.py
```

Open your browser at `http://localhost:8080` to access the chat interface.

### Option 2 — Docker container

```bash
docker build -t cliniqbot .
docker run -p 8080:8080 \
  -e PINECONE_API_KEY=your_key \
  -e OPENAI_API_KEY=your_key \
  cliniqbot
```

---

## How the Chat Works

1. User types a medical question in the chat input field.
2. jQuery sends a `POST` request to the Flask `/get` endpoint.
3. Flask passes the message to the LangChain RAG chain:
   - The Pinecone retriever finds the top-3 most semantically similar chunks from the indexed PDFs.
   - The chunks are injected as `{context}` into the `ChatPromptTemplate`.
   - GPT-4o generates a concise answer of no more than three sentences.
4. The answer is returned and displayed in the chat window as a bot message.

---

## AWS CI/CD Deployment

### 1. Create an IAM user with the following policies

- `AmazonEC2ContainerRegistryFullAccess`
- `AmazonEC2FullAccess`

### 2. Create an ECR repository

Note your ECR URI after creation, e.g.:

```
315865595366.dkr.ecr.us-east-1.amazonaws.com/cliniqbot
```

### 3. Launch an EC2 instance (Ubuntu) and install Docker

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
newgrp docker
```

### 4. Register EC2 as a GitHub Actions self-hosted runner

Go to your GitHub repo → **Settings → Actions → Runners → New self-hosted runner** and follow the setup commands provided.

### 5. Add GitHub Secrets

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_DEFAULT_REGION` | e.g. `us-east-1` |
| `ECR_REPO` | Your ECR URI |
| `PINECONE_API_KEY` | Pinecone API key |
| `OPENAI_API_KEY` | OpenAI API key |

### Deployment flow (GitHub Actions)

On every push to `main`:

1. Docker image is built from source
2. Image is pushed to AWS ECR
3. EC2 runner pulls the latest image from ECR
4. Docker container is restarted on EC2 with updated environment variables

---

## What This Project Is

| What it IS | 
|---|---|
| A RAG-based medical Q&A chatbot |
| Uses Flask as the web framework | 
| Uses GPT-4o as the LLM | 
| Uses HuggingFace local embeddings | 
| Single-turn Q&A over indexed PDFs | 
| Pinecone for vector retrieval | 

---

*Built by Anjali Yadav · AI/ML Engineer · University of Mumbai*

*CliniqBot — Bringing clinical intelligence to conversational AI.*
