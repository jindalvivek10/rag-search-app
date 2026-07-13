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

## 🏗️ Deep-Dive Code Architecture (No Jargon)

This application uses a classic "Customer-Waiter-Chef" pattern to keep your security keys completely hidden from hackers while parsing messy raw data into a fast, beautiful user interface.

```
[ Your Screen ]  <───(Clean Data)───>  [ FastAPI Backend ]  <───(Secure SDK)───>  [ Google Vertex AI ]
 (The Customer)                          (The Waiter)                              (The Chef)
```

### 🐍 1. The Backend Gateway (`backend/main.py`)

The backend is built with **FastAPI** to execute two essential, secure jobs:

#### A. Keeping Cloud Access Keys Safe 🔒
To query Vertex AI, the code must make authenticated API calls. 
* **The Security Risk**: If our React frontend queried Google directly, our private cloud access credentials would be packed into the browser's source code where anyone could inspect and steal them.
* **The Solution**: The frontend talks only to our FastAPI backend. The backend runs securely inside Google Cloud Run and utilizes **Metadata Service Identities** to automatically obtain temporary execution credentials. Your cloud access configurations never touch the user’s computer.

#### B. Clearing Out the Clutter (Data Normalization) 🧹
Google's API returns a massive, confusing structure filled with pixel coordinate boxes, metadata confidence scores, and layout arrays. The backend loops through this data, strips out the noise, and maps it to a neat, simplified format before sending it to your screen:

```python
# Exact request constructed inside main.py
request = discoveryengine.SearchRequest(
    serving_config=serving_config,
    query=request_data.[...](asc_slot://start-slot-14)query,
    page_size=5,
    content_search_spec={
        "extractive_content_spec": {
            "max_extractive_segment_count": 1 # Pull exact matching sentences
        },
        "summary_spec": {
            "include_citations": True # Instruct Gemini to insert[...](asc_slot://start-slot-15), citation markers
        }
    }
)
```

---

### ⚛️ 2. The Frontend Client (`frontend/src/`)

The frontend is a single-page application built with **React** inside the `/frontend` directory. It coordinates what you see on the page.

#### A. State Hooks (`useState`)
React uses standard hooks to keep track of your query and loading state as they change in real-time:
* `query`: What you are searching for.
* `summary`: The markdown-based answer returned by Gemini.
* `results`: The matching PDF sources listed in the sidebar.

#### B. [...](asc_slot://start-slot-17)The Interactive Citation Parser 🏷️
When Gemini returns a summary featuring bracketed citations like `"Google Cloud grew by 20%"`, React uses a **Regular Expression** (`/\[(\d+)\]/g`) to parse the string:
1. It splits the sentence into raw text strings and numeric citations.
2. It wraps the citations into interactive `<button>` elements.
3. When clicked, React matches the citation index to the corresponding source card on your screen, highlights its border in bright blue, and scrolls it directly into view so you can verify the information instantly.
view.



## 🏗️ How the Code Works (No Jargon)

This application uses a classic **"Customer-Waiter-Chef"** pattern to keep your private cloud credentials hidden from hackers while parsing messy raw data into a fast, beautiful user interface.

```
┌─────────────────────────────────┐      ┌─────────────────────────────┐      ┌───────────────────────────────┐
│       CUSTOMER (Frontend)       │ ───> │      WAITER (Backend)       │ ───> │       CHEF (Vertex AI)        │
│    "What is our Cloud revenue?" │      │    Passes query securely    │      │  Processes vector RAG search  │
└─────────────────────────────────┘      └─────────────────────────────┘      └───────────────────────────────┘
                 ▲                                      │                                     │
                 │                                      ▼                                     ▼
┌─────────────────────────────────┐      ┌─────────────────────────────┐      ┌───────────────────────────────┐
│       CUSTOMER (Frontend)       │ <─── │      WAITER (Backend)       │ <─── │       CHEF (Vertex AI)        │
│   Renders clean, highlighted    │      │    Strips away raw clutter   │      │ Sends raw JSON metadata tree  │
│      answers and citations      │      │     and plates data nicely  │      │   (coordinates & paragraphs)  │
└─────────────────────────────────┘      └─────────────────────────────┘      └───────────────────────────────┘
```

---

### ⚛️ 1. How the React Frontend (FE) is Built

The frontend is a modern web application built using **React** and **JSX** inside the `/frontend` directory.

#### A. Development vs. Production Compilation
* **In Development**: React runs on a local development server (typically port `5173`) and compiles your changes instantly using Hot Module Replacement (HMR).
* **For Production Build**: Running `npm run build` triggers a compiler (like Vite) that takes your React code, compresses it, and flattens it into a **single folder of static assets** (`frontend/dist`). This folder contains only one `index.html` file, a highly optimized JavaScript bundle, and a minified CSS stylesheet.

#### B. State Management (The Brain of the Screen)
React uses **state hooks** (`useState`) to dynamically track changes on your screen:

| State Variable | What It Tracks | Visual Result on Your Screen |
| :--- | :--- | :--- |
| **`query`** | The raw text typed into the search bar. | Reflects typing in the search bar. |
| **`loading`** | A Boolean flag (`true` \| `false`). | Displays a loading spinner while the API is processing. |
| **`summary`** | The natural language summary from Gemini. | Rendered in the primary reading card at the top. |
| **`results`** | A list of matching source documents. | Rendered as a sidebar of clickable cards. |

---

### 🐍 2. How the FastAPI Backend API is Written

The backend is a high-performance Python web server written in **FastAPI** (`backend/main.py`). It has two primary responsibilities:

#### A. Serving the Static Frontend (Avoiding CORS Blockades)
Normally, if your browser attempts to load the UI from port `3000` but sends requests to port `8080`, security mechanisms (CORS) block the request. FastAPI solves this in production by **mounting** your compiled React production bundle directly onto the root URL:
```python
# FastAPI serves your compiled React site directly on the "/" path
app.mount("/", StaticFiles(directory="frontend/dist", html=True), name="static")
```
When you load your deployment URL, FastAPI hands your browser the static `index.html` file. The frontend and backend run seamlessly under **one single URL**.

#### B. Guarding API Routes & Credentials 🔒
Web browsers cannot safely store Google Service Account keys—hackers could easily extract them from your code. FastAPI acts as a security proxy:
* React sends requests to `POST /api/search`.
* The backend runs securely inside Google Cloud Run and utilizes **Metadata Service Identities** to automatically obtain temporary access tokens to query Vertex AI. No physical passwords or private keys are ever sent to your browser.
* The API utilizes the clean, top-level Google Cloud SDK:
```python
from google.cloud import discoveryengine  # Dynamically points to the latest stable release (v1)
```

---

### 🔄 3. The Lifecycle of a Query (Step-by-Step Control Flow)

#### Step 1: The Trigger
You type *"What was Google's Cloud revenue in 2025?"* and press Enter.
1. React sets `setLoading(true)` (rendering a loading spinner).
2. React packs the query and sends an HTTP request to the backend:
   ```javascript
   fetch("/api/search", {
     method: "POST",
     headers: { "Content-Type": "application/json" },
     body: JSON.stringify({ query: "What was Google's Cloud revenue in 2025?" })
   })
   ```

#### Step 2: The Security Handshake
1. FastAPI intercepts the request at `/api/search` and instantiates the `SearchServiceClient`.
2. It maps the incoming query into a formal Google `SearchRequest` targeting your specific cloud project (`vjindal-project-ai-basic`) and engine (`rag-search-app`).
3. The backend forwards the request securely to Google's internal APIs.

#### Step 3: Vertex AI Double-Pass Execution
Google's server runs a **two-step RAG pipeline**:
1. **Semantic Search (Retrieval)**: It searches the vector index of your GCS-hosted PDFs to locate the exact paragraphs that mention *"Google Cloud"* and *"2025 revenue"*.
2. **Grounded Generation (Gemini LLM)**: It feeds those parsed paragraphs directly into the Gemini LLM and says: *"Answer the user's question using ONLY these facts. Insert bracketed citations like [1] to show which document supported which fact."*

#### Step 4: Data Clean-Up (Raw vs. Flat Payload)
* **What Vertex AI returns to your Backend (Raw Metadata Tree)**:
  ```json
  {
    "summary": {
      "summary_text": "In 2025, Google Cloud revenue grew significantly, reaching $45B [1].",
      "summary_with_metadata": {
        "citation_metadata": {
          "citations": [{"start_index": 59, "end_index": 63, "sources": [{"reference_index": "0"}]}]
        }
      }
    },
    "results": [
      {
        "id": "doc_abc123",
        "document": {
          "name": "projects/.../documents/doc_abc123",
          "derived_struct_data": {
            "title": "Alphabet_2025_Earnings.pdf",
            "link": "gs://vjindal-vertex-rag-data/Alphabet_2025_Earnings.pdf",
            "snippets": [{"snippet": "Google Cloud segment revenue was $45,000,000,000 for the year..."}]
          }
        }
      }
    ]
  }
  ```

* **What your Backend returns to your Screen (Cleaned & Flattened)**:
  The backend filters out pixel locations, mapping ranges, and confidence metrics, returning only what matters:
  ```json
  {
    "summary": "In 2025, Google Cloud revenue grew significantly, reaching $45B [1].",
    "results": [
      {
        "title": "Alphabet_2025_Earnings.pdf",
        "link": "gs://vjindal-vertex-rag-data/Alphabet_2025_Earnings.pdf",
        "snippet": "Google Cloud segment revenue was $45,000,000,000 for the year..."
      }
    ]
  }
  ```

#### Step 5: Screen Rendering & Interactive Citations ✨
1. React receives the clean JSON packet and updates its states: `setSummary(data.summary)` and `setResults(data.results)`.
2. React sets `setLoading(false)` (the loading spinner vanishes, and the results are painted on screen).
3. **The Citation Parser**: React scans the summary string, finds the characters `[1]` using a regular expression (`/\[(\d+)\]/g`), and replaces them with a clickable HTML `<button>` badge.
4. **Interactive Scroll**: When you click the `[1]` button, React matches the index, highlights the first source document card's border in a bright blue accent, and scrolls it directly into view so you can verify the information instantly!
```
