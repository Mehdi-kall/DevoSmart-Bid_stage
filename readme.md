\# cv-platform



An end-to-end intelligent platform for automating CV processing and consultant matching in response to client RFPs (\*appels d'offres\*), built during an AI internship at \*\*Devoteam Tunisie\*\*.



The platform ingests CVs in multiple formats, extracts structured candidate data using LLMs, deduplicates and normalizes records across a growing candidate database, and matches consultants to client requirements using semantic search — then auto-generates tailored, client-ready CV decks.



\---



\## Table of Contents



\- \[Overview](#overview)

\- \[Screenshots](#screenshots)

\- \[Architecture](#architecture)

\- \[Services \& Tech Stack](#services--tech-stack)

\- \[Models Used](#models-used)

\- \[Core Pipeline](#core-pipeline)

\- \[Project Structure](#project-structure)

\- \[Getting Started](#getting-started)

\- \[Environment Variables](#environment-variables)

\- \[Running the Project](#running-the-project)

\- \[Known Limitations](#known-limitations)

\- \[Roadmap](#roadmap)



\---



\## Overview



Consulting firms responding to RFPs need to quickly identify which consultants match a client's required skill set, and present them in a standardized, professional format. Doing this manually — sifting through hundreds of CVs in different formats, layouts, and languages — is slow and error-prone.



\*\*cv-platform\*\* solves this by:

1\. Extracting structured data from any CV format (PDF, DOCX, PPTX)

2\. Deduplicating and merging candidate records from multiple sources

3\. Matching candidates to a client requirement using semantic search, not just keyword matching

4\. Generating a polished, branded CV deck (PPTX/DOCX) automatically, with AI-adapted content per client



\---



\## Screenshots



> \_Add screenshots below to showcase the platform in action.\_



\*\*Dashboard / Candidate Search\*\*



`\[screenshot placeholder — candidate search \& filtering UI]`



\*\*Candidate Matching \& Scoring\*\*



`\[screenshot placeholder — semantic match results with scores]`



\*\*CV Selection \& Reordering (drag-and-drop)\*\*



`\[screenshot placeholder — CandidateSidebar + drag-and-drop experience reordering]`



\*\*Generated CV Output (PPTX)\*\*



`\[screenshot placeholder — before/after: raw CV vs. generated branded deck]`



\---



\## Architecture



```

┌─────────────┐      ┌──────────────┐      ┌───────────────┐

│   Frontend   │◄────►│   FastAPI     │◄────►│   MongoDB      │

│ React + TS   │      │   Backend     │      │ (candidates,   │

│ Vite/Tailwind│      │               │      │  merged\_data)  │

└─────────────┘      └───────┬───────┘      └───────────────┘

&#x20;                             │

&#x20;             ┌───────────────┼───────────────┐

&#x20;             ▼               ▼               ▼

&#x20;      ┌─────────────┐ ┌─────────────┐ ┌──────────────┐

&#x20;      │  ChromaDB    │ │  Gemini API  │ │  python-pptx │

&#x20;      │ (vector DB,  │ │  (LLM        │ │  (CV deck    │

&#x20;      │  SBERT)      │ │  extraction/ │ │  generation) │

&#x20;      │              │ │  adaptation) │ │              │

&#x20;      └─────────────┘ └─────────────┘ └──────────────┘

```



\---



\## Services \& Tech Stack



\### Backend

| Service | Role |

|---|---|

| \*\*FastAPI\*\* | REST API layer, request validation via Pydantic |

| \*\*MongoDB\*\* | Primary datastore — `candidatesV2` (raw extracted candidates), `merged\_candidates` (deduplicated/merged records) |

| \*\*ChromaDB\*\* | Vector database for semantic search over CV sections (experience, summary, skills) |

| \*\*Sentence-Transformers (SBERT)\*\* | Generates embeddings locally — kept on-premise for GDPR compliance instead of external embedding APIs |

| \*\*Gemini API\*\* (`google.genai`) | LLM-based structured extraction, CV content adaptation, anti-hallucination rewriting |

| \*\*Groq API\*\* | Alternative/fallback LLM provider for faster inference on certain tasks |

| \*\*DeepL API\*\* | Automatic translation for multilingual CVs |

| \*\*python-pptx + lxml\*\* | Direct manipulation of PPTX XML for template duplication, dynamic layouts, and notes-slide handling in multi-candidate merges |

| \*\*pdfplumber\*\* | PDF text/layout extraction |

| \*\*rapidfuzz\*\* | Fast fuzzy string matching, used as a prefilter before semantic deduplication |

| \*\*pycountry / langdetect\*\* | Country and language normalization |



\### Frontend

| Service | Role |

|---|---|

| \*\*React + TypeScript\*\* | UI layer |

| \*\*Vite\*\* | Build tool / dev server |

| \*\*Tailwind CSS\*\* | Styling |

| \*\*@dnd-kit\*\* | Drag-and-drop reordering of candidate experiences (order is preserved through to the backend during generation) |

| \*\*framer-motion\*\* | UI animations |



\---



\## Models Used



\- \*\*Embedding model\*\*: `paraphrase-multilingual-mpnet-base-v2` (SBERT) — chosen for multilingual support (French/English CVs) and strong semantic similarity performance on short professional text

\- \*\*LLM (extraction \& adaptation)\*\*: `gemini-3.5-flash` via `google.genai` — used for structured field extraction and CV experience rewriting

&#x20; - ⚠️ Note: `gemini-3.1-flash-live-preview` is a streaming/Live API model and is \*\*not\*\* compatible with `generateContent` — do not substitute it here

\- \*\*Fuzzy matching\*\*: `rapidfuzz` (Levenshtein-based) — used as a cheap prefilter before running SBERT cosine similarity, to avoid comparing every candidate pair

\- \*\*Semantic similarity thresholds\*\* (tuned empirically):

&#x20; - Experience sections: cosine distance ≤ `0.35`

&#x20; - Summary sections: cosine distance ≤ `0.6`



\---



\## Core Pipeline



1\. \*\*Ingestion \& Extraction\*\*

&#x20;  CVs (PDF/DOCX/PPTX) are parsed and routed to a prompt template based on detected format (D2C, TECH-6, generic). Long CVs are chunked before extraction to stay within LLM context limits. Extraction combines regex pre-processing with LLM structuring, validated against Pydantic schemas.



2\. \*\*Storage \& Deduplication\*\*

&#x20;  Extracted candidates are stored in `candidatesV2`. A Union-Find clustering algorithm groups likely-duplicate profiles using SBERT cosine similarity on experience text, pre-filtered by company-name similarity (rapidfuzz) — this prefilter is essential, since generic consulting vocabulary alone can produce false-positive similarity scores above 0.9 between unrelated companies. The richest version of each section across duplicates is selected and merged into `merged\_candidates`.



3\. \*\*Normalization\*\*

&#x20;  Skills, countries, and languages are normalized against canonical/alias tables to make downstream search and filtering reliable.



4\. \*\*Semantic Search \& Matching\*\*

&#x20;  Candidate sections are embedded and indexed in ChromaDB. Given a client requirement, the platform computes a multi-criteria match score — experience similarity is weighted as the dominant factor, with skills and certifications as secondary criteria using dynamic weight normalization.



5\. \*\*CV Generation\*\*

&#x20;  Selected candidates/experiences are adapted for the client context using Gemini (with anti-hallucination prompting to avoid fabricated details), then rendered into a branded PPTX deck by duplicating and populating template slides via `python-pptx`/`lxml`. Multi-candidate decks are merged with correct handling of `notesSlideN.xml` relationships.



\---



\## Project Structure



```

cv-platform/

├── app/

│   ├── normalize\_sections/     # Skills, countries, languages normalization

│   ├── extraction/              # Multi-format parsing + LLM extraction

│   ├── matching/                 # global\_score.py — multi-criteria scoring

│   ├── generation/               # CV JSON building + PPTX/DOCX rendering

│   └── main.py                   # FastAPI entrypoint

├── frontend/

│   ├── src/

│   │   ├── components/           # CandidateSidebar, drag-and-drop UI, etc.

│   │   └── ...

│   └── vite.config.ts

├── start-all.bat / stop-all.bat  # One-click local service management (Windows)

└── README.md

```



\---



\## Getting Started



\### Prerequisites



\- Python 3.10+

\- Node.js 18+

\- MongoDB running locally (`mongodb://localhost:27017`)

\- API keys: Gemini, (optionally) Groq and DeepL



\### Installation



```bash

\# Clone the repo

git clone https://github.com/<your-username>/cv-platform.git

cd cv-platform



\# Backend setup

python -m venv .venv

.venv\\Scripts\\activate        # Windows

\# source .venv/bin/activate   # macOS/Linux

pip install -r requirements.txt



\# Frontend setup

cd frontend

npm install

cd ..

```



\---



\## Environment Variables



Create a `.env` file at the project root:



```env

MONGO\_URI=mongodb://localhost:27017/cv\_platform

GEMINI\_API\_KEY=your\_gemini\_api\_key

GROQ\_API\_KEY=your\_groq\_api\_key

DEEPL\_API\_KEY=your\_deepl\_api\_key

```



\---



\## Running the Project



\*\*Option 1 — One-click (Windows):\*\*



```bash

start-all.bat

```



This launches MongoDB (if not already running), the FastAPI backend, and the React frontend together. Use `stop-all.bat` to shut everything down.



\*\*Option 2 — Manual:\*\*



```bash

\# Terminal 1 — Backend

.venv\\Scripts\\activate

uvicorn app.main:app --reload --port 8000



\# Terminal 2 — Frontend

cd frontend

npm run dev

```



The frontend will be available at `http://localhost:5173` (default Vite port), and the API at `http://localhost:8000`.



\---



\## Known Limitations



\- \*\*Multi-candidate merge bug\*\*: Gemini occasionally returns malformed JSON (`No valid JSON found in model response`) during `adapt\_selected\_experiences` when merging multiple candidates into one deck — root cause under investigation.

\- Extraction quality depends on CV template consistency; heavily unstructured CVs may require manual correction after extraction.



\---



\## Roadmap



\- \[ ] Resolve Gemini JSON parsing failure in multi-candidate merge

\- \[ ] Backend refactor: consolidate logic into `CandidateService`, `generation\_service.py`, and `matching\_service.py`

\- \[ ] Expand template support beyond D2C / TECH-6 / generic

