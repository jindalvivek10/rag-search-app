# 🔍 Google-Quality RAG Search Portal with Vertex AI

A modern, production-grade Retrieval-Augmented Generation (RAG) system. This full-stack application enables users to search through complex, unstructured PDF catalogs and obtain natural, grounded answers from Gemini, complete with interactive inline citations and source metadata linking.

---

## 🎯 Codelab Phase Mapping

This project is divided into a three-tier system: the ingestion layer, the semantic engine, and the serverless presentation app.

```
┌─────────────────────────────────┐      ┌─────────────────────────────┐      ┌───────────────────────────────┐
│     Cloud Storage Ingestion     │ ───> │     Vertex AI Datastore     │ ───> │       FastAPI Backend         │
│   (PDFs & JSONL Metadata Map)   │      │ (Digital Parser & Indexing) │      │  (Discovery Engine Gateway)   │
└─────────────────────────────────┘      └─────────────────────────────┘      └───────────────────────────────┘
                                                                                              │
                                                                                              ▼
┌─────────────────────────────────┐      ┌─────────────────────────────┐      ┌───────────────────────────────┐
│       Cloud Run Host            │ <─── │   Optimized Docker Image    │ <─── │        React Frontend         │
│  (Serverless Scalable App)      │      │     (Multi-stage + `uv`)    │      │  (Grounded Answers UI Portal) │
└─────────────────────────────────┘      └─────────────────────────────┘      └───────────────────────────────┘
```

---

## 🛠️ Step-by-Step Modernized Deployment Guide

### 1. Initialize Google Cloud Environment

Launch Google Cloud Shell, verify your target credentials, and activate the required AI and compilation engine APIs.

```bash
# Verify active authorized identity
gcloud auth list

# Confirm project context is set to your workspace
gcloud config list project

# Ensure you are targeting your project id
gcloud config set project vjindal-project-ai-basic

# Enable platform APIs for Vertex AI, Cloud Run, and Cloud Build
gcloud services enable \
    aiplatform.googleapis.com \
    run.googleapis.com \
    cloudbuild.googleapis.com
```

---

### 2. Configure Storage & Format Metadata

Vertex AI Search uses JSON Lines (`.jsonl`) to map unstructured files to custom metadata fields. We convert our standard JSON configuration to a flat JSONL dataset and upload it.

```bash
# Retrieve Alphabet financial report metadata structure
wget https://raw.githubusercontent.com/kkrishnan90/vertex-ai-search-agent-builder-demo/main/alphabet-metadata.json

# Convert JSON Array format to JSON Lines format (JSONL)
jq -c '.[]' alphabet-metadata.json > alphabet-metadata.jsonl

# Upload the formatted catalog index mapping to your GCS bucket
gcloud storage cp alphabet-metadata.jsonl gs://vjindal-vertex-rag-data/
```

> 💡 **Notice**: Remove the original `alphabet-metadata.json` file from your bucket `gs://vjindal-vertex-rag-data` if it was previously uploaded, to avoid document import warnings.

---

### 3. Setup Vertex AI Datastore & Application

The Google Cloud Console layout has been modernized. Create your search application through the **AI Applications** panel to connect your backend.

#### Set Up the Datastore
1. Search for **AI Applications** (or *Search & Conversation*) in the Google Cloud Console.
2. In the left navigation pane, click **Data Stores** -> **+ Create Data Store**.
3. Select **Cloud Storage** as your source.
4. Select **Documents with Metadata (RAG)** under unstructured ingestion.
5. Set Synchronization frequency to **One time**.
6. Set the folder path to: `gs://vjindal-vertex-rag-data`.
7. Choose **global (Global)** as the location, and name your store `vjindal_datastore_rag_codelab`.
8. Ensure **Digital Parser** is selected, then click **Create**.

#### Create the RAG Search App
1. Under **AI Applications**, click **Apps** -> **Create App**.
2. Select the **Custom search (general)** engine card.
3. Configure the Identity:
   * **Enterprise Edition**: Enabled (Checked)
   * **Generative Responses**: Enabled (Checked)
   * **App Name**: `rag-search-app`
   * **Company Name**: `Alphabet`
   * **Location**: `global (Global)`
4. Link the application to `vjindal_datastore_rag_codelab` and click **Create**.

---

### 4. Build the Container Image (Using `uv`)

To minimize deployment sizes and speed up container builds, this project uses a multi-stage Docker configuration featuring Astral's `uv` fast-installer.

```bash
# Clone the repository
git clone https://github.com/kkrishnan90/vertex-ai-search-agent-builder-demo
cd vertex-ai-search-agent-builder-demo

# Compile the multi-stage image targeting linux/amd64 platforms
docker build --platform linux/amd64 -t rag-search-app .
```

#### Multi-Stage Dockerfile (`Dockerfile`)
```dockerfile
# --- Stage 1: Build React Assets ---
FROM node:22 AS frontend-builder
WORKDIR /app/frontend
COPY frontend/package*.json ./
RUN npm install
COPY frontend/ .
RUN npm run build

# --- Stage 2: Fast python compile with `uv` ---
FROM python:3.12-slim AS backend-builder
WORKDIR /app
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

COPY backend/requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

# Integrate compiled UI files and FastAPI code
COPY --from=frontend-builder /app/frontend/dist ./frontend/dist
COPY backend/ ./backend

EXPOSE 8080
CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

---

### 5. Push to Artifact Registry & Deploy

Configure secure Docker authentication, push the built image to your repository, and deploy it to Google Cloud Run.

```bash
# Authenticate local Docker daemon with Google Cloud's Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Create the repository if not already present
gcloud artifacts repositories create search-repo \
    --repository-format=docker \
    --location=us-central1 \
    --description="Docker repository for Vertex AI Search app"

# Tag the image for regional uploading
docker tag rag-search-app us-central1-docker.pkg.dev/vjindal-project-ai-basic/search-repo/rag-search-app:latest

# Push image to Artifact Registry
docker push us-central1-docker.pkg.dev/vjindal-project-ai-basic/search-repo/rag-search-app:latest

# Deploy container as a serverless microservice to Cloud Run
gcloud run deploy rag-search-app \
    --image=us-central1-docker.pkg.dev/vjindal-project-ai-basic/search-repo/rag-search-app:latest \
    --platform=managed \
    --region=us-central1 \
    --allow-unauthenticated \
    --set-env-vars="PROJECT_ID=vjindal-project-ai-basic,LOCATION=global,ENGINE_ID=rag-search-app"
```

---

## 🏗️ Deep-Dive Code Architecture

This repository uses a decoupled frontend-backend pattern designed to run securely under a single Cloud Run URL, preventing Cross-Origin Resource Sharing (CORS) conflicts.

```
 vertex-ai-search-agent-builder-demo/
 ├── backend/                 # Python FastAPI Microservice
 │   ├── main.py              # Application Gateway & Google Client SDK wrapper
 │   └── requirements.txt     # Python Dependencies
 ├── frontend/                # React UI Single Page App
 │   ├── src/                 # SPA Components (Search Bar, Citations, PDF Viewers)
 │   └── package.json         # Node Dependencies
 ├── Dockerfile               # High-speed multi-stage builder
 └── cloudbuild.yaml          # Google CI/CD pipeline
```

### 🐍 Backend Gateway (`backend/main.py`)
The backend is written in **FastAPI** to handle two essential, secure requirements:
1. **Cloud Security (IAM Proxy)**: Web browsers cannot safely store Google Service Account keys. By running a FastAPI middleware layer inside Cloud Run, the application uses **Workload Identity/Metadata credentials** to authenticate search queries securely on the server-side, hiding private details from visitors.
2. **Result Parsing & Normalization**: The Vertex AI `SearchServiceClient` returns structured, deep metadata configurations (containing page bounding coordinates, structural tokens, and source indexes). FastAPI intercepts this response, parses out the essential segments, and flattens the JSON into an optimized schema.

#### Search Call Pipeline:
```python
# Conceptual flow inside backend/main.py
from fastapi import FastAPI
from google.cloud import discoveryengine_v1 as discoveryengine

app = FastAPI()
client = discoveryengine.SearchServiceClient()

@app.post("/api/search")
async def search_index(query: str):
    # Construct paths dynamically from environment variables
    serving_config = client.project_location_collection_engine_serving_config_path(
        project="vjindal-project-ai-basic",
        location="global",
        collection="default_collection",
        engine="rag-search-app",
        serving_config="default_serving_config"
    )

    request = discoveryengine.SearchRequest(
        serving_config=serving_config,
        query=query,
        page_size=3,
        content_search_spec={
            "extractive_content_spec": {"max_extractive_segment_count": 2},
            "summary_spec": {
                "summary_result_count": 5, 
                "include_citations": True
            }
        }
    )
    
    response = await client.search(request)
    return parse_and_flatten_response(response) # Converts nested models into readable React schemas
```

---

### ⚛️ Frontend Single-Page App (`frontend/`)
The frontend is a custom search dashboard built using **React**. It is compiled into pure HTML/CSS/JavaScript and served statically from our Python web server (`main.py` serving `/frontend/dist`).

#### Core UX Mechanisms:
1. **Dual-State Display**:
   * **Grounded Summary State**: Displays the synthetic answer directly compiled by the LLM. [...](asc_slot://start-slot-23)It shows inline markdown links mapped to reference coordinates (e.g., ``[...](asc_slot://start-slot-24), ``).
   * **Extractive Chunks List**: Lists individual paragraphs matching semantic criteria, ranking documents by factual relevance.
2. **Interactive Cite Handler**: Clicking an inline citation card dynamically parses coordinates inside the viewport, opening a PDF modal to show you the original file, and highlighting the exact page where Gemini found that fact.
3. **Ingestion Upload Utility**: The UI has an upload interface which streams local PDFs to the backend's `/api/upload` endpoint, automatically inserting the files into the `gs://vjindal-vertex-rag-data` bucket.
