# 🎥 YouTube RAG — AI-Powered Video Knowledge Base

An end-to-end **Retrieval-Augmented Generation (RAG)** system built with **n8n** that transforms YouTube video transcripts into a searchable AI knowledge base.

The system automatically extracts transcripts, processes timestamped chunks, generates vector embeddings, stores them in **Supabase**, and enables AI-powered question answering through both **general retrieval** and **video-specific metadata filtering**.

It also includes a controlled **data deletion pipeline** that allows indexed videos to be removed from the vector database through Google Sheets.

---

## ✨ Features

- 🎬 **Automated YouTube Transcript Extraction** — Extracts full transcripts from YouTube videos.
- 🧩 **Smart Processing & Chunking** — Processes transcript segments while preserving context.
- ⏱️ **Timestamp-Aware Chunks** — Preserves precise start, end, and duration information.
- 🧠 **Google Gemini Embeddings** — Converts transcript chunks into vector representations.
- 🗄️ **Supabase Vector Storage** — Stores searchable transcript embeddings and metadata.
- 🔎 **Semantic Similarity Search** — Retrieves relevant transcript content based on meaning.
- 🎯 **Cohere Reranking** — Improves retrieval relevance before generation.
- 💬 **Conversational YouTube RAG Agent** — Ask questions across the indexed video library.
- 🏷️ **Metadata-Filtered Retrieval** — Restricts retrieval to a specific video.
- 🔗 **Source Attribution** — Returns video titles, timestamps, and source URLs.
- 📊 **Google Sheets Management** — Tracks indexed videos and their processing state.
- 🗑️ **Controlled Data Cleanup** — Removes transcript vectors through a spreadsheet status update.
- 🔄 **Fully Automated n8n Workflow** — Connects the complete ingestion, retrieval, and cleanup lifecycle.
- 🔐 **Credential-Safe Design** — Public workflow export is sanitized of credentials and secrets.

---

## 🏗️ System Architecture

The system is organized into four core pipelines:

1. **Transcript Ingestion & Storage**
2. **General YouTube RAG Agent**
3. **Metadata-Filtered RAG Agent**
4. **Transcript Deletion & Cleanup**

```text
                     ┌──────────────────────┐
                     │    YouTube Video     │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │   n8n Form Trigger   │
                     │   Title + Video URL  │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │   Apify Transcript   │
                     │      Extraction      │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │ Transcript Processing│
                     │ Text + Timestamps    │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │   Gemini Embeddings  │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │ Supabase Vector Store│
                     │      documents       │
                     └──────────┬───────────┘
                                │
                      ┌─────────┴─────────┐
                      │                   │
                      ▼                   ▼
             ┌──────────────────┐ ┌────────────────────┐
             │   General RAG    │ │ Metadata Filter RAG│
             │ Qwen + Retrieval │ │  Qwen + Filtered   │
             │   + Reranking    │ │     Retrieval      │
             └────────┬─────────┘ └──────────┬─────────┘
                      │                     │
                      └──────────┬──────────┘
                                 │
                                 ▼
                     ┌──────────────────────┐
                     │     AI Response      │
                     │   + Source Context   │
                     └──────────────────────┘

                         Google Sheets
                               │
                        Status = Remove
                               │
                               ▼
                     ┌──────────────────┐
                     │ Delete & Cleanup │
                     └─────────┬────────┘
                               │
                               ▼
                    Delete matching vectors
                               │
                               ▼
                        Status = Removed


📖 Deep Dive: For the complete technical documentation, see [`Documentation/architecture.md`](Documentation/architecture.md).

🔄 1. Transcript Ingestion & Storage
The ingestion pipeline transforms raw YouTube videos into structured, vector-indexed knowledge.
Pipeline Flow

On Form Submission
        │
        ▼
Get Transcript
        │
        ├──────────────► Transcript
        │
        └──────────────► Timestamps
                              │
                              ▼
                            Merge
                              │
                              ▼
                    Default Data Loader
                              │
                              ▼
                     Gemini Embeddings
                              │
                              ▼
                    Supabase Vector Store
                              │
                              ▼
                     Google Sheets


Process
User Submission
The user provides a video title and YouTube URL through an n8n Form.
Transcript Extraction
The URL is sent to the Apify transcript scraper.
Processing
Transcript segments are normalized and grouped into timestamp-aware chunks.
Metadata Enrichment
Each stored document includes:
- videoTitle
- timestamp
- videoURL
Vectorization
Google Gemini converts transcript chunks into vector embeddings.
Storage & Logging
Embeddings are stored in Supabase, while Google Sheets maintains a lightweight index of processed videos.

🧠 2. General YouTube RAG Agent
The general RAG pipeline allows users to query the complete indexed YouTube knowledge base.
Pipeline Flow

User Query
    │
    ▼
YouTube RAG Agent
    │
    ▼
Supabase Vector Search
    │
    ▼
Cohere Reranker
    │
    ▼
Relevant Transcript Chunks
    │
    ▼
Qwen 3.7 Flash
    │
    ▼
Grounded Response

The agent retrieves relevant transcript chunks from Supabase and uses Cohere reranking to improve the relevance of the retrieved context before generating the final response with Qwen.
Responses are grounded in retrieved transcript content and preserve source metadata such as video title, timestamp, and URL.

🎥 3. Metadata-Filtered RAG Agent
The metadata-filtered pipeline is designed for questions about a specific YouTube video.

Pipeline Flow

Video Insights Form
(Video Title + Query)
        │
        ▼
     RAG Agent
        │
        ▼
Supabase Vector Store
        │
        ▼
Filter: videoTitle
        │
        ▼
Relevant Chunks
        │
        ▼
    AI Response

Why Metadata Filtering?
When multiple videos contain similar topics or terminology, unrestricted retrieval can introduce context from unrelated videos.
The metadata filter restricts retrieval to documents belonging to the selected video.

All Documents
      │
      ▼
Filter by videoTitle
      │
      ▼
Selected Video
      │
      ▼
Semantic Retrieval
      │
      ▼
Relevant Context

This provides more precise video-specific answers and reduces cross-video context mixing.

🗑️ 4. Transcript Deletion & Cleanup
The deletion pipeline provides lifecycle management for indexed transcripts.

Pipeline Flow

Google Sheets
Status = "Remove"
        │
        ▼
Google Sheets Trigger
        │
        ▼
Filter
        │
        ▼
Loop Over Items
        │
        ▼
Extract Video URL
        │
        ▼
Delete Matching Supabase Vectors
        │
        ▼
Update Google Sheets
Status = "Removed"


The video URL is used to identify the transcript documents stored in Supabase.
After successful deletion, the corresponding spreadsheet row is updated from Remove to Removed.
🧰 Technology Stack
Layer	Technology
Workflow Automation	n8n
Transcript Extraction	Apify
LLM	Qwen Cloud — qwen3.7-flash
Embeddings	Google Gemini
Vector Database	Supabase + pgvector
Reranking	Cohere
Knowledge Base Management	Google Sheets
Data Processing	JavaScript / n8n Code Nodes
Architecture	Retrieval-Augmented Generation (RAG)


📸 Screenshots
Complete Workflow

Transcript Ingestion Pipeline

General RAG Agent

Metadata-Filtered RAG Agent

Delete & Cleanup Pipeline

📁 Project Structure

youtube-rag-n8n/
│
├── Workflow/
│   └── youtube-rag-n8n.json
│
├── Screenshots/
│   ├── workflow-overview.png
│   ├── transcript-pipeline.png
│   ├── rag-agent.png
│   ├── metadata-filter-rag.png
│   └── delete-pipeline.png
│
├── Documentation/
│   └── architecture.md
│
├── README.md
├── .gitignore
└── .env.example


⚙️ Setup & Installation
Prerequisites
Before importing the workflow, make sure you have:
- An active n8n instance
- A Supabase project with pgvector enabled
- An Apify account and API token
- Google Cloud / Gemini API access
- Google Sheets OAuth credentials
- Qwen Cloud API access
- Cohere API access
Quick Start
1. Import the Workflow
Import:

Workflow/youtube-rag-n8n.json
nto your n8n instance.

2. Configure Credentials
Configure the required n8n credentials for:
- Apify
- Supabase
- Google Sheets
- Google Gemini
- Qwen Cloud
- Cohere
3. Configure Supabase
Create and configure the required documents table with vector support.
The vector store is used to store:
- Transcript content
- Embeddings
- Video metadata
- Timestamp information
4. Configure Google Sheets
Create a tracking sheet with:

Title | URL | Status | Transcript

The Status column is used by the deletion pipeline.
🔐 Security
The published workflow is sanitized for public sharing.
- No API keys or access tokens are included.
- Credential references are removed from the public workflow export.
- Sensitive values are replaced with placeholders.
- Google Sheet IDs and API tokens are represented using placeholders.
- Local environment files and credential files are excluded through .gitignore.
Note: Credentials must be configured separately inside your own n8n instance.

🧠 Design Principles
Grounded Generation
Prompts are designed to keep responses grounded in retrieved transcript context and avoid unsupported claims.
Source Traceability
Transcript chunks retain video title, timestamp, and URL metadata so generated answers can be traced back to the original source.
Metadata-Aware Retrieval
The system can restrict retrieval to a specific video when precise video-level querying is required.
Retrieval Quality
The general RAG pipeline uses reranking to improve the relevance of retrieved transcript chunks.
Data Lifecycle Management
Indexed videos can be removed from the vector database through a controlled Google Sheets workflow.
Modular Architecture
Each major responsibility is isolated into a dedicated workflow section, making the system easier to understand, maintain, and extend.

👨‍💻 Author
Ahmed Gabr Elzanbeel
AI Automation & Integration Engineer
Specializing in:
- AI Agents
- RAG Systems
- n8n Workflow Automation
- API & Workflow Integrations
- AI Automation

⭐ Support
If you find this project useful, consider giving the repository a ⭐ on GitHub.
