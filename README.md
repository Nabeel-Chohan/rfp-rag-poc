# RFP RAG PoC — AI Coding Agent Prompt

## Project
**Goal:** Build a self‑contained Proof‑of‑Concept (PoC) for a human‑centric Retrieval‑Augmented Generation (RAG) system for RFP handling. The PoC runs in Google Colab, uses **OpenAI 3.5 Turbo** for model calls, persists Knowledge Modules to a **JSON file** (`module_store.json`), accepts up to **two** uploaded past successful RFPs for ingestion, and exposes a minimal FastAPI backend + simple HTML UI. The system must be **provenance‑first, deterministic, inspectable, and refuse to guess** when knowledge is missing.

---

## Quick start (what the AI agent must produce first)
1. Create repository skeleton matching the layout in **Required files & layout** below.  
2. Implement three plain Python agent functions (no orchestration frameworks). Each agent must accept explicit inputs and return structured JSON only.  
3. Implement a FastAPI app with endpoints listed in **API endpoints**. Include a minimal HTML UI that shows: Question, Suggested Modules (with provenance), Draft Answer (with sentence citations), and controls for Approve/Purge.  
4. Persist modules to `module_store.json` and logs to `logs.txt`. Provide a `/purge` endpoint that deletes persisted files when confirmed.  
5. Constrain OpenAI calls to JSON-only outputs and include the model name `gpt-3.5-turbo` in prompts. Log every model call input and output.

---

## Required files & layout
Create these files with minimal, focused content (stubs where appropriate):

```
/rfp-rag-poc
├─ README.md                # this file (prompt for the coding agent)
├─ LICENSE
├─ .gitignore
├─ specs/API_SPEC.md
├─ specs/AGENTS_SPEC.md
├─ specs/MODULE_SCHEMA.json
├─ notebooks/colab_poc_notebook.ipynb  # placeholder cells list
├─ samples/sample_rfp_1.txt
├─ samples/sample_rfp_2.txt
├─ server/app.py            # FastAPI entrypoint
├─ server/agents.py         # agent implementations
├─ logs.txt                 # append-only agent logs (created at runtime)
├─ module_store.json        # created at runtime
└─ docs/DEMO_SCRIPT.md
```

**Keep each file minimal and explicit.** The notebook is the primary runtime; server files must run inside Colab.

---

## Agents and exact signatures (implement as plain functions)
Implement these three agents in `server/agents.py`. Each must return structured JSON and append a JSON line to `logs.txt` describing inputs/outputs.

**1. Distillation agent**
```python
def distill_document_agent(raw_document_text: str, source_document_id: str) -> List[dict]:
    """
    Returns: List of Knowledge Module dicts following MODULE_SCHEMA.json.
    Determinism: module_id must be deterministic (e.g., KM-{sha1(content)[:8]}-{YYYYMMDDHHMMSS}).
    Behavior: Prefer deterministic paragraph-split + keyword heuristics; only call OpenAI for JSON-only extraction when needed.
    """
```

**2. Retrieval agent**
```python
def retrieve_modules_agent(question: str, module_store: List[dict]) -> dict:
    """
    Returns: {
      "selected_module_ids": [...],
      "rationales": {module_id: rationale_str, ...},
      "status": "found" | "none",
      "message": "No relevant modules found"  # when none
    }
    Behavior: deterministic matching first (tag match, exact phrase, keyword overlap). Only call OpenAI to rank ambiguous candidates; model must return JSON-only rationales.
    """
```

**3. Assembly agent**
```python
def assemble_answer_agent(question: str, selected_modules: List[dict]) -> dict:
    """
    Returns: {
      "draft_answer": str,
      "sentence_traces": [{"sentence": str, "module_ids": [str,...]}, ...],
      "used_module_ids": [...],
      "status": "ok" | "insufficient",
      "reason": str  # when insufficient
    }
    Behavior: Use ONLY provided modules. Each sentence must be traceable to one or more module_ids. If any sentence lacks a module citation, return status "insufficient".
    """
```

---

## Knowledge Module schema (specs/MODULE_SCHEMA.json)
Use this exact minimal schema for every module (values types shown):

```json
{
  "module_id": "string",
  "title": "string",
  "content": "string",
  "source_document_id": "string",
  "source_excerpt": "string",
  "confidence_level": 0.0,
  "created_at": "ISO8601 string",
  "tags": ["string"]
}
```

**Rules:** `module_id` stable and human‑readable; `source_excerpt` ≤ 300 chars; `confidence_level` float 0.0–1.0.

---

## API endpoints (implement in server/app.py)
Implement these endpoints and return the structured JSON shapes shown in `specs/API_SPEC.md`.

- **POST /ingest/upload**  
  - Accept up to 2 files (text or PDF). Extract text (use `pdfplumber` for PDFs). For each file call `distill_document_agent`, append modules to `module_store.json`, and return created module IDs and provenance.

- **GET /modules**  
  - Return full module store JSON array.

- **POST /query**  
  - Body: `{"question":"..."}`  
  - Flow: call `retrieve_modules_agent`. If `status == "none"`, return retrieval result and `"assembly": null`. If found, load selected modules and call `assemble_answer_agent`. Return retrieval, assembly, and logs.

- **POST /approve**  
  - Body: `{"question":"...", "approved": true|false, "finalize": true|false}`  
  - If `approved` and `finalize` true, return `final_answer` and `status: "finalized"`. Append approval to logs.

- **POST /purge**  
  - Body: `{"confirm": true}`  
  - Delete `module_store.json`, `logs.txt`, and uploaded sample files. Return list of deleted files.

- **GET /**  
  - Minimal HTML UI with three panels (Question, Suggested Modules with provenance, Draft Answer with citations) and controls for upload, approve, purge. UI must display raw agent inputs/outputs and rationales.

---

## Determinism, constraints, and OpenAI usage
- **Model:** use `gpt-3.5-turbo` (OpenAI 3.5 Turbo). All model calls must be constrained to return **JSON only**. Include explicit system instructions in prompts to enforce JSON output.  
- **Determinism:** implement deterministic matching rules (tag match, exact phrase, keyword overlap) before any model call. `module_id` generation must be deterministic.  
- **No hallucination:** assembly agent must refuse to add facts not present in selected modules. If model output contains unreferenced sentences, mark `status: "insufficient"`.  
- **One agent = one responsibility:** agents are plain functions; control flow lives in FastAPI endpoints.  
- **Persistence:** store modules in `module_store.json` as an array; never use embeddings or vector DB for this PoC.

---

## Logging, observability, and purge
- **Logging:** append JSON lines to `logs.txt` for every agent call with keys: `agent`, `input_summary`, `output_summary`, `timestamp`. Also include model prompt and model response when a model call occurs. Return logs in API responses for transparency.  
- **Purge:** `/purge` must require explicit confirmation and remove `module_store.json`, `logs.txt`, and uploaded files. Provide a clear JSON list of deleted files.

---

## Testing, demo, and acceptance criteria
- **Smoke tests:** provide curl examples for each endpoint in `specs/API_SPEC.md`.  
- **Demo script:** implement `docs/DEMO_SCRIPT.md` with a 5‑minute flow: ingest two sample RFPs, query a supported question (show retrieval + assembly), query an unsupported question (show explicit refusal), approve and finalize, then purge.  
- **Acceptance criteria:**  
  - Every sentence in final draft is traceable to one or more `module_id`s.  
  - System returns `"No relevant modules found"` when no modules match.  
  - All agent inputs, outputs, and rationales are visible in API responses and `logs.txt`.  
  - The system can be reset via `/purge`.

---

## Minimal development steps for the AI agent to start coding
1. Create repo skeleton and files listed above.  
2. Implement `server/agents.py` with the three agent functions and deterministic `module_id` generator. Add simple deterministic heuristics (tag match, exact phrase, keyword overlap). Add fallback OpenAI JSON-only extraction/ranking where needed.  
3. Implement `server/app.py` FastAPI endpoints and minimal HTML UI. Ensure endpoints return the structured JSON shapes.  
4. Add two short synthetic RFPs to `samples/` for initial testing.  
5. Add `specs/*` files with exact examples and curl commands.  
6. Run in Colab: install dependencies, set `OPENAI_API_KEY`, run FastAPI and expose via pyngrok or Colab port forwarding. Verify demo flow.  
7. Add unit tests that validate JSON shapes for each endpoint and agent.

---

## Deliverables for first PR
- `server/agents.py` with working agent implementations (deterministic rules + OpenAI JSON-only calls).  
- `server/app.py` with all endpoints and minimal UI.  
- `samples/` with two RFPs.  
- `specs/` populated (`API_SPEC.md`, `AGENTS_SPEC.md`, `MODULE_SCHEMA.json`).  
- `docs/DEMO_SCRIPT.md` and `logs.txt` created during demo.  
- README (this file) present at repo root.

---

**Start coding now using this README as the single source of truth.**
