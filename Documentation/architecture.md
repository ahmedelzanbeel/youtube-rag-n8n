# System Architecture

## Overview

YouTube RAG is an AI-powered Retrieval-Augmented Generation (RAG) system built with n8n.

The system extracts YouTube video transcripts, processes them into timestamp-aware chunks, generates vector embeddings, stores them in Supabase, and enables users to query the indexed knowledge through AI-powered retrieval.

The architecture also includes metadata-filtered retrieval, reranking, source attribution, and controlled transcript deletion.

The system is organized into four main pipelines:

1. Transcript Ingestion & Storage
2. General YouTube RAG Agent
3. Metadata-Filtered RAG Agent
4. Transcript Deletion & Cleanup

---

# 1. Transcript Ingestion & Storage

The ingestion pipeline converts a YouTube video into structured and searchable knowledge.

## Flow

```text
YouTube Video
     │
     ▼
n8n Form Trigger
(Video Title + URL)
     │
     ▼
Apify Transcript Scraper
     │
     ├──────────────► Transcript Processing
     │
     └──────────────► Timestamp Chunking
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


1.1 Video Submission
The pipeline starts with an n8n Form Trigger.
The user provides:
- Video Title
- YouTube URL
These values are used throughout the workflow for transcript processing, metadata generation, storage, retrieval, and deletion.
1.2 Transcript Extraction
The Get Transcript node sends the YouTube URL to an Apify transcript scraper.
The scraper returns timestamped transcript segments containing information such as:
- Transcript text
- Start time
- Duration
The extracted transcript becomes the source knowledge for the RAG system.
1.3 Transcript Processing
The workflow uses dedicated Code nodes to process the transcript.
Transcript Processing
The Transcript node combines transcript segments into a normalized text string.
Processing includes:
- Removing unnecessary newlines
- Trimming whitespace
- Combining transcript segments
- Normalizing repeated spaces
The resulting transcript can also be stored in Google Sheets.
Timestamp Processing
The Timestamps node groups transcript segments into chunks of 20 items.
Each chunk contains:
- Combined transcript text
- Start timestamp
- End timestamp
- Duration
- Formatted timestamps
- Chunk number
- Number of transcript segments
This preserves the relationship between retrieved content and its original location in the YouTube video.
1.4 Merge
The Merge node combines the processed transcript information with timestamp data.
The resulting documents are prepared for vector storage.
1.5 Metadata Enrichment
Each stored document contains metadata such as:
videoTitle
timestamp
videoURL
This metadata enables:
- Source attribution
- Exact timestamp references
- Video-specific retrieval
- Controlled deletion
- Traceability between generated answers and original content
1.6 Embeddings
Google Gemini Embeddings converts transcript chunks into vector representations.
Conceptually:
Transcript Chunk
      │
      ▼
Gemini Embedding Model
      │
      ▼
Vector Representation
These vector representations enable semantic similarity search rather than relying only on keyword matching.
1.7 Supabase Vector Store
The generated vectors and document metadata are stored in the Supabase documents table.
Supabase acts as the persistent knowledge layer of the system.
The vector store supports:
- Semantic retrieval
- Metadata filtering
- RAG tool access
- Vector deletion
1.8 Google Sheets
After processing a video, the workflow records the video information in Google Sheets.
Stored information includes:
- Video Title
- URL
- Transcript
Google Sheets also acts as a lightweight management layer for controlling the lifecycle of indexed videos.
2. General YouTube RAG Agent
The General RAG pipeline allows users to query the complete indexed YouTube knowledge base.
Flow
Chat Message
     │
     ▼
YouTube RAG Agent
     │
     ├──────────────► Qwen Cloud
     │
     └──────────────► Supabase Vector Store
                           │
                           ▼
                       Retrieval
                           │
                           ▼
                       Reranker
                           │
                           ▼
                    Relevant Chunks
                           │
                           ▼
                     AI Response


2.1 Chat Trigger
The workflow starts when a user sends a message through the n8n chat interface.
The user can ask questions about any indexed YouTube transcript.
2.2 YouTube RAG Agent
The YouTube RAG Agent acts as the main reasoning layer.
Its responsibilities include:
- Understanding the user's request
- Querying the vector database
- Retrieving relevant transcript chunks
- Using transcript metadata
- Maintaining chronological context when necessary
- Producing grounded answers
The agent is instructed to avoid unsupported information and rely on retrieved transcript content.
2.3 Qwen Cloud
Qwen Cloud provides the language model used by the RAG agent.
Configured model:
qwen3.7-flash
The model interprets the retrieved context and generates the final response.
2.4 Supabase Retrieval
The Supabase Vector Store is exposed to the agent as a retrieval tool.
The system performs semantic similarity search against indexed transcript chunks.
The retrieval configuration uses a Top-K strategy to retrieve multiple potentially relevant chunks.
2.5 Reranking
Retrieved candidates are passed through a Cohere reranker.
The reranker evaluates candidate chunks and improves the relevance of the final context provided to the language model.
Conceptually:

User Query
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

This improves retrieval precision, especially when multiple transcript chunks contain semantically similar information.
3. Metadata-Filtered RAG Agent
The Metadata-Filtered RAG pipeline is designed for questions about a specific YouTube video.
Instead of searching the entire knowledge base, retrieval is restricted using video metadata.
Flow

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
           └──────────► Supabase Vector Store
                              │
                              ▼
                       Metadata Filter
                              │
                              ▼
                       Relevant Chunks
                              │
                              ▼
                         AI Response

3.1 Video Insights Form
The user provides:
- Video Title
- Query
The selected video title is used as retrieval metadata.
3.2 RAG Agent 2
RAG Agent 2 receives the user's query together with the selected video title.
The agent is instructed to answer only using retrieved transcript chunks and their associated metadata.
3.3 Metadata Filtering
The Supabase Vector Store applies a metadata filter using:
videoTitle
This prevents transcript chunks from unrelated videos from entering the retrieval context.
Conceptually:


All Documents
      │
      ▼
Filter by Video Title
      │
      ▼
Selected Video Documents
      │
      ▼
Semantic Retrieval
      │
      ▼
Relevant Context
      │
      ▼
AI Response


This is particularly useful when multiple videos discuss similar subjects.
3.4 Source Attribution
The retrieved metadata allows the agent to reference the original source.
Responses can include:
- Video title
- Exact timestamp
- Original video URL
The agent is instructed to preserve timestamps and URLs exactly as supplied by the retrieval layer.
This provides traceability between the generated response and the original transcript.
4. Transcript Deletion & Cleanup
The deletion pipeline provides lifecycle management for indexed transcript data.
An indexed video can be removed from the vector database by changing its status in Google Sheets.
Flow

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
   Delete Vectors
        │
        ▼
 Update Google Sheets
        │
        ▼
    Status = Removed

4.1 Google Sheets Trigger
The workflow monitors updates to the Google Sheets Status column.
The deletion process is activated when the status becomes:
Remove
4.2 Filter
The Filter node ensures that only rows explicitly marked:
Remove
continue through the deletion workflow.
This prevents unrelated spreadsheet updates from triggering deletion.
4.3 Loop Over Items
The Loop Over Items node processes deletion items individually.
This provides controlled execution when multiple rows need to be processed.
4.4 Edit Fields
The Edit Fields node extracts the video URL from the selected spreadsheet row.
The URL is then used as the identifier for locating the associated transcript vectors.
4.5 Vector Deletion
The Supabase deletion step removes documents whose metadata contains the matching video URL.
Conceptually:

Google Sheets URL
       │
       ▼
Match videoURL metadata
       │
       ▼
Delete transcript vectors

This removes the transcript chunks associated with the selected video.
4.6 Status Update
After the deletion process, the Google Sheets row is updated to:
Removed
This provides a simple lifecycle indicator and makes the deletion state visible to the user.
5. Data Lifecycle
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
     ├───────────────► General RAG
     │
     └───────────────► Metadata-Filtered RAG
     
     
Google Sheets
     │
     │ Status = Remove
     ▼
Vector Deletion
     │
     ▼
Status = Removed

This architecture provides a complete ingestion, retrieval, and deletion lifecycle.
6. Technology Stack
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
Metadata Filtering	Supabase Vector Store
Data Processing	n8n Code Nodes


7. Key Design Principles
Grounded Generation
The RAG agents are instructed to rely on retrieved transcript content instead of generating unsupported information.
Source Traceability
Transcript chunks retain video title, timestamp, and URL metadata so generated answers can be connected to their original source.
Metadata-Aware Retrieval
The architecture supports restricting retrieval to a specific YouTube video when required.
Retrieval Quality
The general RAG pipeline uses reranking to improve the relevance of retrieved transcript chunks.
Data Lifecycle Management
Indexed videos can be removed from the vector database through a controlled Google Sheets workflow.
Modular Architecture
Each major responsibility is isolated into a dedicated workflow section, making the system easier to understand, maintain, debug, and extend.
8. Architecture Summary
The system combines:

YouTube
   │
   ▼
Apify
   │
   ▼
n8n Processing
   │
   ▼
Gemini Embeddings
   │
   ▼
Supabase Vector Store
   │
   ├── General RAG
   │       └── Qwen + Reranking
   │
   └── Metadata-Filtered RAG
           └── Qwen + Video Filter

Google Sheets
   │
   └── Lifecycle Control
           │
           ▼
      Vector Deletion

The result is a modular YouTube knowledge system capable of transforming video transcripts into searchable AI knowledge while maintaining source traceability and controlled data lifecycle management.
