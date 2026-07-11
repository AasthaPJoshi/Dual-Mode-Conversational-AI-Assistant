# 🤖 Dual-Mode Conversational AI Assistant

A local-first, privacy-respecting conversational AI system that combines a **Retrieval-Augmented Generation (RAG)** pipeline with a **standard LLM chat interface**, running entirely on-device via [Ollama](https://ollama.ai). Conversations are streamed in real time to a clean Flask-powered web dashboard, with optional text-to-speech output and voice input support.
 
---
    
## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Design Decisions](#design-decisions)
- [Future Improvements](#future-improvements)

---

## Overview

This project was built to explore the practical integration of locally-run large language models with a retrieval layer over external documents, with no cloud APIs or data leaving the machine.

The system operates in two distinct modes:

- **Regular Mode:** A context-aware conversational assistant powered by a local Ollama model (Mistral or Gemma), tuned for domain-specific Q&A.
- **RAG Mode:** Users supply a URL pointing to a PDF or webpage. The system fetches, chunks, and retrieves the most relevant content, then grounds the LLM's response in that source material.

A lightweight Flask server acts as a bridge between the terminal-based AI engine and a browser-based conversation viewer, updating in real time via polling.

---

## Key Features

- **Dual conversation modes:** seamlessly switch between standard LLM chat and document-grounded RAG at runtime
- **On-device inference:** all LLM calls route through Ollama; no external API keys or data egress required
- **Dynamic document ingestion:** accepts live PDF URLs or web pages as RAG knowledge sources
- **Text chunking with overlap:** uses LangChain's `RecursiveCharacterTextSplitter` for clean, context-preserving segmentation
- **Relevance-based chunk retrieval:** lightweight keyword-matching retrieval selects the top-k chunks most relevant to the query
- **Real-time web dashboard:** Flask server syncs conversation history every 2 seconds via client-side polling; visually differentiates Regular vs RAG turns
- **Text-to-speech output:** `pyttsx3` speaks LLM responses aloud for accessibility and hands-free interaction
- **Voice input support:** Web Speech API integration enables microphone-based query input in the browser
- **ANSI/spinner output cleaning:** robust regex pipeline strips terminal control sequences from raw Ollama subprocess output
- **Modular architecture:** terminal interface, Flask API, and document processing layers are cleanly separated

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Terminal Interface                   │
│         (terminal_interaction.py / test.py)              │
│                                                         │
│  ┌──────────────┐          ┌──────────────────────────┐ │
│  │  Regular Mode │          │         RAG Mode         │ │
│  │               │          │                          │ │
│  │  Prompt +     │          │  URL → fetch/scrape →   │ │
│  │  History →    │          │  PDF/HTML text →         │ │
│  │  Ollama LLM   │          │  chunk → retrieve →      │ │
│  └──────┬────────┘          │  Ollama LLM              │ │
│         │                   └──────────────┬───────────┘ │
│         └──────────────┬───────────────────┘             │
│                        ▼                                 │
│              pyttsx3 TTS (speak response)                │
│                        │                                 │
│                        ▼                                 │
│         POST /history → Flask API (app.py)               │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Flask Web Server                       │
│                     (app.py)                            │
│                                                         │
│   Routes:  /         → serve index.html                 │
│            /history  → GET/POST conversation state      │
│            /mode     → GET/POST current mode            │
│            /full_history → both mode histories          │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                  Browser Dashboard                       │
│           (templates/index.html + static/)              │
│                                                         │
│   Polls /history every 2s → renders new messages        │
│   Color-codes turns: blue = Regular, red = RAG          │
│   Voice input via Web Speech API                        │
│   TTS playback via SpeechSynthesis API                  │
└─────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| LLM Runtime | [Ollama](https://ollama.ai) (Mistral, Gemma 2B) |
| Backend API | Python, Flask |
| RAG Pipeline | LangChain (`RecursiveCharacterTextSplitter`), PyPDF2, BeautifulSoup4 |
| TTS | pyttsx3 |
| Frontend | Vanilla HTML/CSS/JS, Web Speech API |
| HTTP Client | requests |
| Language | Python 3.10+ |

---

## Project Structure

```
.
├── app.py                    # Flask server: REST API for conversation state and mode
├── terminal_interaction.py   # Main AI engine: LLM chat, RAG pipeline, TTS, terminal I/O
├── test.py                   # Alternate entrypoint with LangChain chunking and cleaner RAG
├── templates/
│   └── index.html            # Real-time conversation dashboard
└── static/
    ├── css/
    │   └── styles.css        # Dashboard styling: mode-colour-coded message cards
    ├── js/
    │   └── script.js         # Voice input, TTS, and message submission logic
    ├── ai.gif                # Splash animation shown before conversation begins
    ├── ai1.gif
    └── ai2.gif
```

---

## Getting Started

### Prerequisites

- Python 3.10+
- [Ollama](https://ollama.ai/download) installed and running locally
- A pulled model, e.g. `ollama pull mistral` or `ollama pull gemma:2b`

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-username/dual-mode-ai-assistant.git
cd dual-mode-ai-assistant

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install flask requests pypdf2 beautifulsoup4 pyttsx3 langchain

# 4. Ensure Ollama is running with your chosen model
ollama serve
ollama pull mistral            # or: ollama pull gemma:2b
```

### Running

Open **two terminals**:

**Terminal 1 (Flask server):**
```bash
python app.py
```
The dashboard will be available at `http://127.0.0.1:5000`.

**Terminal 2 (AI engine):**
```bash
python terminal_interaction.py
# or, for the LangChain-enhanced version:
python test.py
```

---

## Usage

### Regular Mode (default)

```
Enter 'text' for text input (Current mode: regular): text
Enter your command or query. (Current mode: regular): What are the main challenges in social robotics?
LLM (Regular): ...
```

### Switching to RAG Mode

```
Enter 'text' for text input (Current mode: regular): text
Enter your command or query. (Current mode: regular): switch to RAG
Switched to RAG mode.

Please provide the link (PDF or Webpage) for the document.
Enter the link: https://example.com/research-paper.pdf

Enter your query: Summarise the key findings of this paper.
LLM (RAG): ...
```

### Switching back to Regular Mode

```
Enter your query or command: switch to regular
Switched to Regular mode.
```

The browser dashboard at `http://127.0.0.1:5000` will reflect all exchanges in real time, colour-coded by mode.

---

## Design Decisions

**Why Ollama over a cloud API?**
The project was designed with data privacy as a constraint. Running models locally via Ollama ensures no query content or document data is transmitted externally. It also removes API cost and rate-limit concerns during iterative development.

**Why a separate Flask server instead of embedding the UI in the terminal script?**
Decoupling the conversation engine from the presentation layer means either component can be replaced independently. The Flask server is intentionally minimal: it holds state and serves the frontend; all AI logic lives in the terminal module.

**Why polling instead of WebSockets?**
A 2-second polling interval is simple to implement, requires no additional dependencies, and is more than sufficient for conversational response cadences. WebSocket upgrades would be a straightforward future improvement.

**Why keyword-based chunk retrieval?**
The retrieval step in `test.py` uses word-overlap scoring rather than embedding similarity. This avoids a vector database dependency while still surfacing contextually relevant passages for shorter documents. Embedding-based retrieval (e.g. FAISS + sentence-transformers) is the natural upgrade path.

---

## Future Improvements

- **Embedding-based semantic retrieval:** replace keyword overlap with FAISS or ChromaDB vector search for more robust RAG performance
- **WebSocket push:** replace polling with server-sent events or WebSockets for lower-latency dashboard updates
- **Persistent conversation storage:** write history to SQLite so sessions survive server restarts
- **Model selection UI:** allow switching Ollama models from the browser without restarting the engine
- **Document upload:** add drag-and-drop PDF upload directly in the browser rather than requiring a URL
- **Streaming responses:** pipe Ollama token output to the frontend incrementally instead of waiting for full completion
- **Multi-document RAG:** maintain a persistent document store across multiple ingested sources per session
- **Containerisation:** Dockerise the full stack (Ollama + Flask + engine) for one-command deployment

---

## Learning Outcomes

This project provided hands-on experience with:

- Designing and implementing a complete RAG pipeline from document ingestion to grounded response generation
- Integrating a locally-run LLM into a Python application via subprocess orchestration
- Building a real-time web interface over a REST API without a frontend framework
- Text preprocessing challenges: chunking strategy, overlap tuning, and ANSI escape stripping from subprocess output
- Architectural separation between AI inference logic and web presentation concerns
- Practical trade-offs between retrieval quality and system complexity at different stages of development

---

## License

This project is released under the [MIT License](LICENSE).
