RAG Sales Chatbot
A retrieval-augmented generation (RAG) system that answers natural language questions about a B2B SaaS company's sales history using call transcripts, deal notes, churn reports, and quarterly business reviews.

Built with n8n, Pinecone, and OpenAI — no custom code beyond lightweight data processing nodes.
What it does
My RAG app provides sales leadership with answers about sales, renewal, discounting and churn  from call transcripts, deal notes, churn reports, QBR summaries in a chatbot with [90%] faithfulness and/or [90%] relevance.
A sales leader can ask questions like:

"Why did Crestline Manufacturing churn?"
"Which AE tends to offer the most discounts and what reasons do they give?"
"What feedback did Meridian Health Systems give in their 2024 QBR?"

The system retrieves relevant document chunks from a vector store, surfaces the source metadata (document type, date, customer, account executive), and generates a cited answer grounded in the actual records.

Field
Answers
Use case
(one line)
My RAG app helps Sales leadership answer sales, renewal, discounting and churn questions from call transcripts, deal notes, churn reports, QBR summaries in a chatbot with [90 %] faithfulness and/or [90%] relevance
Corpus
Structured data: 
10 customers, 3 industries
3 account executives
2 products, 2 pricing tiers each
Unstructured data
Call transcripts per customer 
Deal notes
churn reports
QBR summaries (top 3 customers only)
Ingestion + cleaning
Documents are JSON files stored in Google Drive. n8n's Search Files node lists all 62 files in the corpus folder, then a Download node pulls each one. A Code node parses each JSON into content and metadata fields. 
Cleaning is minimal since the corpus is synthetic plain text — no HTML, no OCR, no encoding issues. 
The only cleaning step is header stripping: a Code node checks doc_type in metadata and strips everything above the first blank line for all document types except QBR summaries, which keep their section headers (Business Performance, Platform Usage, etc.) because they aid chunking. Whitespace is normalized by collapsing triple+ newlines into double.
Ingestion + freshness
Currently manual — you click "Test workflow" to run the ingestion pipeline. There's no refresh cadence or freshness SLA because this is a demo corpus with static data.
Chunking + embedding
Recursive Character Text Splitter at 800 tokens with 100 token overlap. It splits on paragraphs first (double newlines), falls back to sentences if a paragraph exceeds the cap. This is a hybrid of semantic and fixed — paragraph boundaries are natural meaning boundaries, but the token cap prevents oversized chunks. Embedding model is OpenAI text-embedding-3-small (1536 dimensions). Chosen for cost efficiency and speed — for a 62-doc corpus the quality difference vs text-embedding-3-large is negligible, and it keeps latency low on the query side.
Retrieve
Pinecone as the vector store. Dense retrieval only — no sparse or hybrid (BM25). Top-k is set to 5. Hybrid wasn't needed because metadata filtering handles the exact-match cases (specific customer, date, contract ID) that keyword search would normally cover. A separate Code Tool handles structured queries (revenue, dates, calculations) that vector search can't answer, so the system has two retrieval paths rather than one hybrid retriever.



Why RAG and not just an LLM?
These questions could be answered by a general-purpose LLM if you pasted all the documents into the prompt — but that doesn't scale. RAG earns its place here because:

Recency: the corpus can be updated without retraining a model.
Exact citations: answers reference specific documents with dates and metadata, not paraphrased training data.
Scoped retrieval: the system searches a curated, verified dataset — not the open internet.

Structured questions (revenue totals, contract expiry dates) are better served by database queries. A hybrid retrieval path combining vector search with tool calls against structured data is also 
Architecture
Ingestion Pipeline


Recursive Text Splitter: 800 tokens with 100overlap
OpenAI Embeddings

Query Pipeline

AI Agent - GPT-4mini


Corpus
62 synthetic documents simulating 3 years (2023–2025) of sales activity for a fictional B2B SaaS company selling two products (DataSync Pro and InsightIQ) to 12 customers.

Document type
Count
Description
Call transcripts
24
2 per customer, spread across years
Deal notes
16
New business, renewals, and upsells
Churn reports
4
Crestline, Redstone, Irongate, Castlepoint
QBR summaries
18
Top 3 customers, semi-annual, 3 years


All documents are JSON files with a content field (the text) and a metadata field (customer ID, name, account executive, document type, date, year, contract ID). The metadata is stored alongside the vector in Pinecone and used for filtering and citation.

A structured data spreadsheet (sales_corpus_source_of_truth.xlsx) served as the single source of truth during corpus generation. Every name, number, and date in the unstructured documents was written against this spreadsheet to prevent conflicting information.
Cleaning logic
Header stripping: all document types except QBR summaries have their header block (document type, customer name, date, etc.) removed before chunking — that information already lives in metadata and would pollute retrieval if left in the text.
QBR summaries retain their headers because the section labels (Business Performance, Platform Usage, etc.) provide useful structure for the chunker.
Whitespace normalization: consecutive blank lines collapsed to a single paragraph break.
Tech stack
Component
Tool
Purpose
Orchestration
n8n (cloud)
Workflow automation for both pipelines
Vector store
Pinecone
Stores embeddings + metadata
Embeddings
OpenAI text-embedding-3-small
1536-dimension vectors
LLM
OpenAI gpt-4o-mini
Answer generation
File storage
Google Drive
Corpus hosting for ingestion

Repo structure
rag-sales-chatbot/

├── README.md

├── workflows/

│   └── Sales_Leadership_RAG_Chatbot.json   # n8n workflow (import into n8n)

├── corpus/                                  # 62 JSON documents

│   ├── call_transcript_C001_2023-05-15.json

│   ├── churn_report_C004.json

│   ├── deal_note_C001_2023.json

│   ├── qbr_C001_2023-07-18.json

│   └── ...

├── data/

│   └── sales_corpus_source_of_truth.xlsx    # Structured data (source of truth)

└── evals/

    └── rag_evals.xlsx                       # 18 eval questions with scoring
Setup
Prerequisites
n8n account (cloud or self-hosted)
Pinecone account with an index (dimension: 1536, metric: cosine)
OpenAI API key
Steps
Import the workflow: in n8n, go to Workflows → Import → upload Sales_Leadership_RAG_Chatbot.json.
Configure credentials: add your Google Drive OAuth2, Pinecone API key, and OpenAI API key in n8n's credential manager.
Upload corpus: unzip the corpus folder and upload all 62 JSON files to a Google Drive folder. Update the Search Files node to point at your folder.
Run ingestion: click "Test workflow" on the ingestion pipeline. All 62 documents will be chunked, embedded, and stored in Pinecone.
Test queries: open the chat interface on the query pipeline and ask a question.
Eval framework
18 evaluation questions across three retrieval paths:

Path
Count
Description
Vector Store
8
Answered from unstructured documents
Tool Call
6
Require calculation against structured data (not yet implemented)
Hybrid
4
Require both vector search and structured data


Each eval is scored on correctness (did it get the facts right?), completeness (did it find all relevant information?), and source accuracy (did it pull from the right documents?).
Sample output
Question: "Why did Crestline Manufacturing churn?"

Answer: According to the churn report dated June 10, 2024 and the March 18, 2024 renewal call transcript for Crestline Manufacturing (C004), account executive Tom Rivera:

Primary reason: low platform adoption — fewer than 4 of 12 licensed users were logging in regularly by Q1 2024.
Operations Manager Dale Pruitt said the interface was too complex for their day-to-day workflows and that they needed a purpose-built scheduling/inventory tool.
Tom Rivera offered a downgrade and a 15% discount (~$5,500), which Crestline declined.
Secondary factor: budget pressure after investing in new CNC equipment; CFO trimmed SaaS spend.
No direct competitor named — Crestline reverted to spreadsheets and their ERP.
Final ARR lost: $6,480.

License
This project uses synthetic data generated for demonstration purposes. No real customer or company data is included.
