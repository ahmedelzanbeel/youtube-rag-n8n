# 🎥 YouTube RAG — AI-Powered Video Knowledge Base

An end-to-end **Retrieval-Augmented Generation (RAG)** system built with **n8n** that transforms YouTube video transcripts into a searchable AI knowledge base.

The system automatically extracts transcripts, processes timestamped chunks, generates vector embeddings, stores them in **Supabase**, and enables AI-powered question answering through both **general retrieval** and **video-specific metadata filtering**.

It also includes a controlled **data deletion pipeline** that allows indexed videos to be removed from the vector database through Google Sheets.

---

## ✨ Features

- 🎬 Automated YouTube transcript extraction
- 🧩 Transcript processing and chunking
- ⏱️ Timestamp-aware document chunks
- 🧠 Google Gemini embeddings
- 🗄️ Supabase vector storage
- 🔎 Semantic similarity search
- 🎯 Cohere reranking for improved retrieval
- 💬 Conversational YouTube RAG agent
- 🏷️ Metadata-filtered retrieval for specific videos
- 🔗 Source attribution using video metadata and timestamps
- 📊 Google Sheets knowledge-base management
- 🗑️ Controlled transcript/vector deletion
- 🔄 Fully automated n8n workflow
- 🔐 Credential-safe public workflow export

---

# 🏗️ System Architecture

The system is organized into four major pipelines:

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
                         │  Gemini Embeddings   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Supabase Vector Store│
                         │      documents       │
                         └──────────┬───────────┘
                                    │
                      ┌─────────────┴─────────────┐
                      │                           │
                      ▼                           ▼
             ┌──────────────────┐       ┌────────────────────┐
             │  General RAG     │       │ Metadata Filter RAG│
             │                  │       │                    │
             │ Qwen + Retrieval │       │ Qwen + Filtered    │
             │ + Reranking      │       │ Retrieval          │
             └─────────┬────────┘       └──────────┬─────────┘
                       │                           │
                       └─────────────┬─────────────┘
                                     ▼
                            ┌──────────────────┐
                            │   AI Response    │
                            │ + Source Context │
                            └──────────────────┘


                    Google Sheets
                          │
                    Status = Remove
                          │
                          ▼
                 ┌───────────────────┐
                 │ Delete & Cleanup  │
                 └─────────┬─────────┘
                           │
                           ▼
                  Delete matching vectors
                           │
                           ▼
                    Status = Removed
For the complete technical architecture:
[Read the System Architecture](Documentation/architecture.md)
🔄 1. Transcript Ingestion & Storage
The ingestion pipeline transforms a YouTube video into structured, searchable knowledge.
Workflow
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
The user submits:
- Video Title
- YouTube URL
The workflow then:
1. Sends the URL to the Apify transcript scraper.
2. Extracts timestamped transcript segments.
3. Normalizes the transcript text.
4. Groups transcript segments into chunks.
5. Calculates start and end timestamps.
6. Adds video metadata.
7. Generates vector embeddings using Google Gemini.
8. Stores the documents in Supabase.
9. Registers the indexed video in Google Sheets.
🧩 Transcript Processing
The workflow uses dedicated n8n Code nodes to process the extracted transcript.
Transcript Processing
The transcript is normalized by:
- Removing unnecessary newlines
- Trimming whitespace
- Combining transcript segments
- Normalizing repeated spaces
Timestamp Processing
Transcript segments are grouped into chunks of 20 segments.
Each chunk contains:
Text
Start Timestamp
End Timestamp
Duration
Formatted Timestamp
Chunk Number
Segment Count
This allows retrieved content to remain connected to its original location in the YouTube video.
🏷️ Metadata
Every indexed transcript chunk contains metadata such as:
videoTitle
timestamp
videoURL
This metadata is important for:
- Source attribution
- Video-specific retrieval
- Traceability
- Vector deletion
- Knowledge-base management
Example:
Video Title
    │
    ├── Timestamp
    │
    └── Video URL
🧠 2. General YouTube RAG Agent
The general RAG agent allows users to ask questions across the entire indexed YouTube knowledge base.
Workflow
Chat Message
     │
     ▼
YouTube RAG Agent
     │
     ├──────────────► Qwen Cloud
     │
     └──────────────► Supabase
                           │
                           ▼
                     Vector Search
                           │
                           ▼
                       Reranker
                           │
                           ▼
                    Relevant Chunks
                           │
                           ▼
                     AI Response
The agent is responsible for:
- Understanding the user's question
- Searching the vector database
- Retrieving relevant transcript chunks
- Using transcript metadata
- Maintaining chronological context when required
- Producing grounded answers
The configured chat model is:
qwen3.7-flash
🎯 Retrieval & Reranking
The general RAG pipeline uses semantic vector retrieval to identify relevant transcript chunks.
The retrieval process is:
User Query
    │
    ▼
Gemini Embedding
    │
    ▼
Vector Search
    │
    ▼
Candidate Chunks
    │
    ▼
Cohere Reranker
    │
    ▼
Relevant Context
    │
    ▼
Qwen
    │
    ▼
Final Answer
The Cohere reranker re-evaluates retrieved candidates and improves the relevance of the context supplied to the language model.
🎥 3. Metadata-Filtered RAG Agent
The second RAG pipeline is designed for questions about a specific YouTube video.
The user provides:
Video Title
Query
The selected title is used as metadata filtering criteria during vector retrieval.
Workflow
Video Insights Form
        │
        ├── Video Title
        │
        └── Query
              │
              ▼
         RAG Agent 2
              │
              ├──────────► Qwen Cloud
              │
              └──────────► Supabase
                                  │
                                  ▼
                         Metadata Filter
                         videoTitle = selected
                                  │
                                  ▼
                         Relevant Chunks
                                  │
                                  ▼
                            AI Response
Why Metadata Filtering?
Without filtering, similar content from different videos could be retrieved.
Metadata filtering restricts the search to documents belonging to the selected video.
All Documents
      │
      ▼
Filter by Video Title
      │
      ▼
Selected Video
      │
      ▼
Semantic Retrieval
      │
      ▼
Relevant Context
This is particularly useful when multiple videos discuss similar subjects.
🔗 Source Attribution
The RAG agents are instructed to preserve the metadata associated with retrieved transcript chunks.
Responses can reference:
- Video title
- Exact timestamp
- Original video URL
The goal is to maintain a clear relationship between:
User Question
      ↓
Retrieved Transcript Chunk
      ↓
Source Metadata
      ↓
Original YouTube Video
      ↓
Exact Timestamp
This improves transparency and makes the generated answers easier to verify.
🗑️ 4. Transcript Deletion & Cleanup
The system includes a complete data lifecycle mechanism for removing indexed videos.
A video can be deleted by changing its status in Google Sheets.
Workflow
Google Sheets Trigger
        │
        ▼
      Filter
        │
        │ Status = Remove
        ▼
 Loop Over Items
        │
        ▼
   Edit Fields
        │
        ▼
 Delete matching vectors
        │
        ▼
 Update Google Sheets
        │
        ▼
    Status = Removed
Deletion Process
1. Google Sheets detects a row update.
2. The workflow checks the Status field.
3. Only rows with Remove continue.
4. The video URL is extracted.
5. Matching transcript documents are identified using metadata.
6. The associated records are deleted from Supabase.
7. The spreadsheet status is changed to Removed.
This provides controlled lifecycle management for indexed content.
📊 Data Lifecycle
The complete lifecycle of an indexed YouTube video is:
YouTube URL
     │
     ▼
Transcript Extraction
     │
     ▼
Text Processing
     │
     ▼
Chunking + Timestamps
     │
     ▼
Metadata Enrichment
     │
     ▼
Gemini Embeddings
     │
     ▼
Supabase Vector Store
     │
     ├──────────────► General RAG
     │
     └──────────────► Metadata-Filtered RAG


Google Sheets
     │
     │ Status = Remove
     ▼
Vector Deletion
     │
     ▼
Status = Removed
🧰 Technology Stack
Layer	Technology
Workflow Automation	n8n
Transcript Extraction	Apify
Language Model	Qwen Cloud
Chat Model	qwen3.7-flash
Embeddings	Google Gemini
Vector Database	Supabase
Reranking	Cohere
Data Management	Google Sheets
Retrieval Architecture	RAG
Data Processing	n8n Code Nodes
Metadata Filtering	Supabase Vector Store


📸 Screenshots
Complete Workflow

Transcript Ingestion Pipeline

General RAG Agent

Metadata-Filtered RAG

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
⚙️ Setup
Prerequisites
You will need:
- n8n
- Supabase project
- Apify account
- Google account
- Google Sheets
- Google Gemini API access
- Qwen Cloud API access
- Cohere API access
1. Import the Workflow
Import the following file into your n8n instance:
Workflow/youtube-rag-n8n.json
2. Configure Credentials
Create the required credentials inside n8n for:
- Apify
- Supabase
- Google Sheets
- Google Gemini
- Qwen Cloud
- Cohere
Then assign the credentials to their corresponding nodes.
Credentials are not included in this repository.

3. Configure Apify
The Get Transcript HTTP Request node uses an Apify YouTube transcript scraper.
Add your own Apify API token through your n8n credentials/environment configuration.
The public workflow contains a placeholder instead of a real token.
4. Configure Supabase
Create/configure the Supabase vector database used by the workflow.
The workflow expects a table named:
documents
The table stores:
- Document content
- Embeddings
- Metadata
The metadata includes:
videoTitle
timestamp
videoURL
5. Configure Google Sheets
Create a Google Sheet for indexed videos.
The workflow uses columns such as:
Title
URL
Status
Transcript
The spreadsheet serves as a lightweight management layer for the knowledge base.
To remove a video:
Status = Remove
After successful cleanup:
Status = Removed
6. Configure AI Models
Embeddings
Google Gemini Embeddings
Used to convert transcript chunks into vector representations.
Chat Model
Qwen Cloud
qwen3.7-flash
Used by the RAG agents to generate grounded responses.
Reranker
Cohere Reranker
Used by the general RAG pipeline to improve retrieval relevance.
▶️ Usage
Add a YouTube Video
Open the YouTube Database Submission form.
Enter:
Video Title
YouTube URL
Submit the form.
The workflow automatically:
Extract Transcript
       ↓
Process Transcript
       ↓
Create Timestamped Chunks
       ↓
Generate Embeddings
       ↓
Store in Supabase
       ↓
Register Video in Google Sheets
Ask Questions Across the Knowledge Base
Use the chat interface connected to the:
YouTube RAG Agent
You can ask questions about the indexed videos.
The agent retrieves relevant transcript chunks and generates an answer based on the retrieved context.
Ask Questions About One Specific Video
Open the:
Video Insights
form.
Enter:
Video Title
Query
The metadata filter restricts retrieval to the selected video.
Remove an Indexed Video
Open the Google Sheets database and change the video's status to:
Remove
The deletion pipeline will:
Find matching vectors
       ↓
Delete vectors
       ↓
Update spreadsheet
       ↓
Status = Removed
🔐 Security
This repository is intended for public sharing.
Sensitive credentials must never be committed to GitHub.
Do not commit:
API Keys
Access Tokens
OAuth Tokens
Passwords
Database Secrets
Service Account Private Keys
The exported workflow uses placeholders such as:
YOUR_APIFY_API_TOKEN_HERE
YOUR_GOOGLE_SHEET_ID
Configure your own credentials locally inside n8n.
📚 Documentation
Detailed technical documentation is available here:
[System Architecture →](Documentation/architecture.md)
The documentation covers:
- System architecture
- Transcript processing
- Chunking strategy
- Metadata design
- Embeddings
- Vector retrieval
- Reranking
- Metadata filtering
- Deletion pipeline
- Data lifecycle
- Technology stack
- Design principles
🎬 Demo
A complete project demonstration will be added here.
The demo will cover the complete lifecycle:
YouTube Video
      ↓
Transcript Extraction
      ↓
Vector Indexing
      ↓
General RAG Query
      ↓
Metadata-Filtered Query
      ↓
Timestamp-Based Source
      ↓
Video Deletion
🧠 Design Principles
Grounded Generation
The RAG agents are instructed to rely on retrieved transcript content instead of unsupported information.
Source Traceability
Transcript chunks retain video title, timestamp, and URL metadata.
Metadata-Aware Retrieval
The system can restrict retrieval to a specific YouTube video.
Retrieval Quality
The general RAG pipeline uses reranking to improve the relevance of retrieved chunks.
Data Lifecycle Management
Indexed videos can be removed from the vector database through a controlled workflow.
Modular Architecture
Each responsibility is separated into a dedicated section of the n8n workflow, making the system easier to understand, maintain, and extend.
👨‍💻 Author
Ahmed Gabr Elzanbeel
AI Automation & Integration Engineer
Focused on building:
- AI Agents
- RAG Systems
- n8n Workflow Automation
- API Integrations
- AI-powered Automation
- Intelligent Business Workflows
⭐ Project
If you find this project useful, consider giving it a ⭐ on GitHub.
Built with n8n + RAG + Supabase + Gemini + Qwen + Cohere.