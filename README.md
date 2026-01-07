This repository contains a self-contained **Proof-of-Concept (PoC)** for a human-centric Retrieval-Augmented Generation (RAG) system specifically designed for **RFP (Request for Proposal)** handling. The system automates the extraction of knowledge from past successful bids to streamline the generation of new, traceable RFP responses.

---

## 🚀 Overview

The application leverages a multi-agent architecture to transform raw documents into structured "Knowledge Modules". These modules are then retrieved and assembled into draft answers using OpenAI's GPT-3.5 Turbo, ensuring every sentence in the response is traceable to a specific source module.

### Key Features

* 
**Modular Ingestion:** Processes up to two uploaded RFP documents (PDF or Text) and distills them into discrete Knowledge Modules.


* 
**Intelligent Retrieval:** Uses keyword-based filtering to select the most relevant knowledge components for a given query.


* 
**Traceable Assembly:** Generates draft responses where each sentence is linked to a `module_id` for easy verification.


* 
**Lightweight Backend:** Built with **FastAPI** for high performance and **JSON-based storage** for simplicity.


* 
**Integrated UI:** Includes a minimal HTML/JavaScript interface for document uploads, querying, and data management.



---

## 🛠 Tech Stack

* 
**Core Logic:** Python 3.12+ 


* 
**LLM Integration:** OpenAI API (GPT-3.5 Turbo) 


* 
**API Framework:** FastAPI & Uvicorn 


* 
**PDF Processing:** pdfplumber 


* 
**Persistence:** JSON (`module_store.json`) and text logs (`logs.txt`) 



---

## 🧬 Agent Architecture

The system utilizes three specialized functional agents:

1. 
**Distill Document Agent:** Breaks down raw text into schema-validated Knowledge Modules with unique IDs and metadata.


2. 
**Retrieve Modules Agent:** Conducts a keyword-overlap analysis to identify relevant modules for the user's question.


3. 
**Assemble Answer Agent:** Instructs the LLM to synthesize a response using *only* the provided context, returning a JSON object with the draft and sentence-level traces.
