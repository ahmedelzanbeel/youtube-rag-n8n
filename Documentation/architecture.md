# System Architecture

## Overview

YouTube RAG is an AI-powered Retrieval-Augmented Generation (RAG) system built with n8n.

The system transforms YouTube video transcripts into a searchable AI knowledge base by extracting transcript data, processing timestamp-aware chunks, generating vector embeddings, and storing the resulting documents in Supabase.

The architecture supports two retrieval modes:

- General knowledge-base retrieval across all indexed videos
- Metadata-filtered retrieval for a specific YouTube video

The system also includes Cohere reranking, source attribution, Google Sheets management, and controlled transcript deletion.

The system is organized into four main pipelines:

1. Transcript Ingestion & Storage
2. General YouTube RAG Agent
3. Metadata-Filtered RAG Agent
4. Transcript Deletion & Cleanup

---

# 1. Transcript Ingestion & Storage

The ingestion pipeline converts a YouTube video into structured, searchable knowledge.

## 1.1 Pipeline Flow

    YouTube Video
         │
         ▼
    n8n Form Trigger
    (Video Title + URL)
         │
         ▼
    Get Transcript
         │
         ▼
    Apify Transcript Scraper
         │
         ├──────────────────► Transcript Processing
         │
         └──────────────────► Timestamp Processing
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

## 1.2 Video Submission

The workflow starts with an n8n Form Trigger.

The user provides:

- Video Title
- YouTube URL

These values are used throughout the workflow for processing, metadata enrichment, storage, retrieval, source attribution, and deletion.

## 1.3 Transcript Extraction

The YouTube URL is sent to an Apify transcript scraper.

The scraper returns timestamped transcript segments containing information such as:

- Transcript text
- Start time
- Duration

The extracted transcript becomes the primary knowledge source for the RAG system.

## 1.4 Transcript Processing

The workflow uses dedicated n8n Code nodes to process the extracted transcript.

The processing stage normalizes the transcript before it is converted into searchable documents.

Processing includes:

- Combining transcript segments
- Removing unnecessary newlines
- Trimming whitespace
- Normalizing repeated spaces
- Preparing transcript content for chunking

The normalized transcript can also be recorded in Google Sheets for management and tracking purposes.

## 1.5 Timestamp Processing

The timestamp processing stage groups transcript segments into structured chunks.

The workflow groups transcript segments into chunks of 20 items.

Each generated chunk preserves information such as:

- Combined transcript text
- Start timestamp
- End timestamp
- Duration
- Formatted timestamps
- Chunk number
- Number of transcript segments

This timestamp-aware structure allows retrieved information to remain connected to its original location in the YouTube video.

## 1.6 Merge

The Merge node combines the processed transcript information with the timestamp information.

The resulting documents contain both transcript content and corresponding temporal context.

These documents are then prepared for embedding and vector storage.

## 1.7 Metadata Enrichment

Each stored document contains metadata used throughout the retrieval and lifecycle-management processes.

Important metadata includes:

- videoTitle
- timestamp
- videoURL

This metadata enables:

- Source attribution
- Timestamp references
- Video-specific retrieval
- Metadata filtering
- Controlled deletion
- Traceability between generated answers and original content

## 1.8 Embeddings

Google Gemini converts each transcript chunk into a vector representation.

The process can be represented as:

    Transcript Chunk
          │
          ▼
    Gemini Embedding Model
          │
          ▼
    Vector Representation

The resulting vectors allow the system to perform semantic similarity search instead of relying only on exact keyword matching.

## 1.9 Supabase Vector Storage

The generated embeddings, transcript content, and metadata are stored in the Supabase documents table.

Supabase acts as the persistent vector knowledge layer of the system.

The vector store supports:

- Semantic retrieval
- Metadata filtering
- RAG tool access
- Vector deletion
- Persistent document storage

## 1.10 Google Sheets Management

Google Sheets acts as a lightweight management layer for indexed videos.

The workflow records information such as:

- Video Title
- YouTube URL
- Processing Status
- Transcript

The spreadsheet is also used to control the deletion lifecycle of indexed videos.

---

# 2. General YouTube RAG Agent

The General YouTube RAG pipeline allows users to query the entire indexed YouTube knowledge base.

## 2.1 Pipeline Flow

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
                           AI Response

## 2.2 Chat Trigger

The workflow starts when a user sends a message through the n8n chat interface.

The user can ask questions about any indexed YouTube transcript.

Examples include:

- Summarizing concepts discussed across videos
- Finding explanations of a specific topic
- Comparing information across indexed videos
- Asking factual questions about transcript content

## 2.3 YouTube RAG Agent

The YouTube RAG Agent acts as the main reasoning layer.

Its responsibilities include:

- Understanding the user's request
- Querying the vector database
- Retrieving relevant transcript chunks
- Using transcript metadata
- Maintaining contextual relationships between chunks
- Producing grounded responses

The agent is designed to rely on retrieved transcript content rather than unsupported information.

## 2.4 Qwen Cloud

Qwen Cloud provides the language model used by the RAG agent.

Configured model:

qwen3.7-flash

The model receives the retrieved context and generates the final response.

## 2.5 Supabase Retrieval

The Supabase Vector Store is exposed to the agent as a retrieval tool.

The system performs semantic similarity search against the indexed transcript chunks.

The retrieval stage uses a Top-K strategy to identify multiple potentially relevant chunks before reranking.

## 2.6 Cohere Reranking

Retrieved candidate chunks are passed through a Cohere reranker.

The reranker evaluates the relevance of each candidate against the user's query and improves the quality of the context provided to the language model.

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

Reranking is especially useful when multiple transcript chunks contain similar concepts or terminology.

## 2.7 Source Attribution

Retrieved transcript metadata allows the system to preserve source information.

Depending on the retrieved context, responses can reference:

- Video title
- Timestamp
- Original YouTube URL

This makes the generated answer easier to verify against the original source material.

---

# 3. Metadata-Filtered RAG Agent

The Metadata-Filtered RAG pipeline is designed for questions about a specific YouTube video.

Instead of searching the complete knowledge base, retrieval is restricted using the selected video's metadata.

## 3.1 Pipeline Flow

    Video Insights Form
    (Video Title + Query)
              │
              ▼
          RAG Agent 2
              │
              ├──────────────► Qwen Cloud
              │
              └──────────────► Supabase Vector Store
                                      │
                                      ▼
                               Metadata Filter
                                      │
                                      ▼
                            videoTitle = selected
                                      │
                                      ▼
                             Semantic Retrieval
                                      │
                                      ▼
                              Relevant Chunks
                                      │
                                      ▼
                               AI Response

## 3.2 Video Insights Form

The user provides:

- Video Title
- Query

The selected video title becomes the metadata filter used during retrieval.

## 3.3 RAG Agent 2

RAG Agent 2 receives the user's query together with the selected video title.

The agent is instructed to generate answers only from retrieved transcript content.

This prevents unrelated videos from contributing context to the answer.

## 3.4 Metadata Filtering

The Supabase Vector Store applies a metadata filter using:

videoTitle

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

This approach is useful when multiple videos contain similar subjects, keywords, or terminology.

## 3.5 Source Attribution

The retrieved metadata allows the system to preserve the relationship between the generated response and the original video.

Relevant source information includes:

- Video title
- Exact timestamp
- Original video URL

The agent is instructed to preserve retrieved timestamps and URLs instead of inventing source information.

---

# 4. Transcript Deletion & Cleanup

The deletion pipeline provides lifecycle management for indexed transcript data.

An indexed video can be removed from the vector database by changing its status in Google Sheets.

## 4.1 Pipeline Flow

    Google Sheets
          │
          │ Status = "Remove"
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
      Edit Fields
          │
          ▼
    Delete Matching Supabase Vectors
          │
          ▼
    Update Google Sheets
          │
          ▼
    Status = "Removed"

## 4.2 Google Sheets Trigger

The workflow monitors changes to the Google Sheets status field.

The deletion process is activated when the status becomes:

Remove

## 4.3 Filter

The Filter node ensures that only rows explicitly marked:

Remove

continue through the deletion workflow.

This prevents unrelated spreadsheet changes from triggering vector deletion.

## 4.4 Loop Over Items

The Loop Over Items node processes deletion requests individually.

This provides controlled execution when multiple indexed videos require removal.

## 4.5 Edit Fields

The Edit Fields node extracts the video URL from the selected spreadsheet row.

The URL becomes the identifier used to locate the associated transcript documents in Supabase.

## 4.6 Vector Deletion

The deletion stage removes transcript documents whose metadata contains the matching videoURL.

Conceptually:

    Google Sheets URL
           │
           ▼
    Match videoURL metadata
           │
           ▼
    Delete Transcript Vectors

This removes the indexed transcript chunks associated with the selected video.

## 4.7 Status Update

After the deletion process completes, the Google Sheets row is updated from:

Remove

to:

Removed

This provides a visible lifecycle indicator and confirms that the deletion workflow has processed the selected item.

---

# 5. Data Lifecycle

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

This provides a complete lifecycle covering ingestion, indexing, retrieval, source traceability, management, and controlled deletion.

---

# 6. Retrieval Architecture

The system uses a multi-stage retrieval architecture.

## 6.1 General Retrieval

    User Query
        │
        ▼
    Semantic Vector Search
        │
        ▼
    Candidate Documents
        │
        ▼
    Cohere Reranking
        │
        ▼
    Relevant Context
        │
        ▼
    Qwen Generation
        │
        ▼
    Grounded Answer

## 6.2 Metadata-Filtered Retrieval

    User Query + Video Title
               │
               ▼
         Metadata Filter
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
        Qwen Generation
               │
               ▼
        Grounded Answer

The two retrieval modes serve different use cases:

| Retrieval Mode | Scope | Main Purpose |
| --- | --- | --- |
| General RAG | All indexed videos | Cross-video knowledge retrieval |
| Metadata-Filtered RAG | One selected video | Precise video-specific analysis |

---

# 7. Metadata Strategy

Metadata plays an important role in both retrieval and lifecycle management.

Each document is associated with metadata such as:

videoTitle
videoURL
timestamp

This metadata is used for:

- Source attribution
- Video-specific filtering
- Timestamp references
- Document identification
- Controlled vector deletion

The metadata strategy allows the system to maintain a connection between the vectorized transcript and its original YouTube source.

---

# 8. Source Traceability

Source traceability is a core architectural principle.

Instead of storing transcript text alone, the workflow preserves contextual metadata alongside each document.

A retrieved chunk can therefore be associated with:

- Video Title
- Timestamp
- YouTube URL
- Transcript Content

This enables users to understand where retrieved information originated.

It also helps reduce ambiguity when multiple videos discuss similar topics.

---

# 9. Data Management with Google Sheets

Google Sheets acts as a lightweight management interface for the indexed knowledge base.

A typical tracking structure is:

| Title | URL | Status | Transcript |
| --- | --- | --- | --- |

The spreadsheet provides a simple way to:

- Track indexed videos
- Store processed transcript information
- Monitor processing state
- Trigger controlled deletion

The deletion lifecycle is controlled through the Status field.

    Status = Remove
            │
            ▼
    Deletion Workflow
            │
            ▼
    Supabase Vector Deletion
            │
            ▼
    Status = Removed

---

# 10. Technology Stack

| Layer | Technology |
| --- | --- |
| Workflow Automation | n8n |
| Transcript Extraction | Apify |
| Language Model | Qwen Cloud |
| Chat Model | qwen3.7-flash |
| Embeddings | Google Gemini |
| Vector Database | Supabase + pgvector |
| Reranking | Cohere |
| Data Management | Google Sheets |
| Data Processing | JavaScript / n8n Code Nodes |
| Retrieval Architecture | Retrieval-Augmented Generation |
| Metadata Filtering | Supabase Vector Store |

---

# 11. Key Design Principles

## Grounded Generation

The RAG agents are designed to rely on retrieved transcript content instead of generating unsupported information.

## Source Traceability

Transcript chunks retain video title, timestamp, and URL metadata so generated answers can be connected to their original source.

## Metadata-Aware Retrieval

The architecture supports restricting retrieval to a specific YouTube video when precise video-level querying is required.

## Retrieval Quality

The general RAG pipeline uses Cohere reranking to improve the relevance of retrieved transcript chunks.

## Data Lifecycle Management

Indexed videos can be removed from the vector database through a controlled Google Sheets workflow.

## Modular Architecture

Each major responsibility is isolated into a dedicated workflow section, making the system easier to understand, maintain, debug, and extend.

## Credential Safety

The public workflow export is designed to be shared without exposing API keys, access tokens, or private credentials.

---

# 12. Security Considerations

The workflow repository is intended for public sharing.

Before publishing the workflow:

- API keys must be removed.
- OAuth credentials must not be included.
- Supabase secrets must not be included.
- Google service-account credentials must not be included.
- Private identifiers should be replaced with placeholders where necessary.
- Local environment files should be excluded using .gitignore.

The public workflow should contain only the configuration structure required to understand and reproduce the system.

Users importing the workflow must configure their own credentials inside n8n.

> Never commit API keys, access tokens, passwords, OAuth secrets, service-account files, or other sensitive credentials to GitHub.

---

# 13. Project Architecture Summary

The complete architecture can be summarized as:

                         YouTube
                            │
                            ▼
                          Apify
                            │
                            ▼
                     n8n Processing
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
          Transcript Text       Timestamp Chunks
                 │                     │
                 └──────────┬──────────┘
                            ▼
                    Metadata Enrichment
                            │
                            ▼
                    Gemini Embeddings
                            │
                            ▼
                 Supabase Vector Store
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
        General RAG              Metadata-Filtered RAG
              │                           │
              ▼                           ▼
      Vector Retrieval             Metadata Filter
              │                           │
              ▼                           ▼
      Cohere Reranking             Semantic Retrieval
              │                           │
              └─────────────┬─────────────┘
                            ▼
                     Qwen Cloud
                            │
                            ▼
                    Grounded Response


                    Google Sheets
                            │
                            │ Status = Remove
                            ▼
                    Deletion Pipeline
                            │
                            ▼
                  Supabase Vector Deletion
                            │
                            ▼
                     Status = Removed

---

# 14. Architecture Summary

YouTube RAG combines automated transcript ingestion, timestamp-aware processing, vector embeddings, semantic retrieval, reranking, metadata filtering, source attribution, and controlled deletion into a single n8n-based architecture.

The system transforms unstructured YouTube video content into a structured AI knowledge base that can be queried across the entire video library or restricted to a specific video.

The architecture is designed around four principles:

1. Reliable ingestion
2. High-quality retrieval
3. Source traceability
4. Controlled data lifecycle management

This modular approach makes the system suitable as a foundation for:

- Video intelligence systems
- Research assistants
- Educational knowledge bases
- AI-powered content analysis
- Internal knowledge systems
- Domain-specific RAG applications

The architecture can also be extended with additional retrieval strategies, reranking methods, AI models, data sources, and user interfaces as the system evolves.
