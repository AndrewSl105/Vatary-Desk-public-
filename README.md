<div align="center">

# Vatary Desk

**A document workspace with AI built in.**
<img width="1269" height="952" alt="image" src="https://github.com/user-attachments/assets/cf2059a0-6108-4e67-ab61-dbb1b604b88b" />

Open almost any file, edit PDFs in place, keep everything in folders with full version history, and share it with your team. All in one browser tab.

[Website](https://vatarydesk.com) · [Architecture](#architecture) · [Author](https://andrii.tech)

</div>

---

## The idea

Work documents are spread across too many tools. One app opens the PDF, another edits it, a third converts it to Word, a cloud drive stores it, and a chat app is where people send it around. Each move between them loses context, versions and access rules.

Vatary Desk puts that whole loop on one desk. You upload a file once. From then on you can read it, edit it, convert it, ask AI about it, see who changed what, and share it, all without leaving the workspace or downloading a copy.

## What you can do

### 📄 Edit PDFs, not just view them
- **Edit text in place.** Change words in an existing PDF and keep the original fonts, size and layout. Text reflows inside its block.
- **Move and replace content.** Swap images, move objects and adjust the page layout.
- **Organize pages.** Reorder, rotate and delete pages, split one PDF into parts or merge several into one.
- **Compose a new PDF** from pages of other files on your desk.
- **Respect protection.** Owner-protected PDFs open normally. When you save, you choose to keep the original protection or remove it, so it is never stripped by accident.
- **Built for big files.** Files go straight to object storage, and processing runs in a separate service, so a 500-page manual does not slow the app down.

### 🗂 Open almost any format
| Kind | Formats |
|---|---|
| PDF | view, edit, organize, split, merge, protect |
| Documents | DOCX, DOC, ODT, RTF, TXT, Markdown |
| Spreadsheets | XLSX, XLS, ODS, CSV, TSV |
| Presentations | PPTX |
| E-books | EPUB, FB2 |
| Images | PNG, JPG, WebP, AVIF, GIF, SVG, **HEIC/HEIF** |
| Audio and video | MP3, WAV, M4A, OGG, FLAC, MP4, WebM, MOV and more |

**Convert between them** without leaving the desk: DOCX → PDF, PDF → DOCX with real editable paragraphs and tables, PDF → Markdown, HEIC → JPG, and more. Tables and images survive the conversion.

### 📁 A real file system
- Nested **folders**, a file tree and a **trash** you can restore from.
- **Tags**, links and fast search across your files and pages.
- Fast uploads of large files in parts, directly to storage.
- An installable **PWA** that caches what you opened, in **Ukrainian and English**.

### 🕓 History you can trust
- Every page and table keeps a **version timeline**.
- **Compare any two versions** side by side with highlighted changes and a change list.
- **Restore** an earlier version in one click, or bring back only the part you need.

### 🤝 Sharing and teamwork
- Share a file, folder, page or whole desk with **roles**: Owner, Manager, Editor, Viewer.
- Access set on a folder applies to everything inside it.
- **Public links** for people outside the workspace.
- Email notification when someone gives you access.
- **Real-time co-editing** of pages: see who else is in the document, with changes merged by CRDTs (Yjs).
- Organizations and teams with a shared workspace.

### ✍️ Pages and widgets
- A rich **page editor** with tables, images, outlines and import from Word or HTML. Export back to DOCX or PDF.
- Put **widgets** on your desk: notes, to-dos, kanban boards, tables with formulas, flowcharts, galleries, bookmarks and more.

### 🤖 AI that works on your documents
- Ask questions about a document and get answers grounded in its content, including very long files. The service reads the document outline first and pulls only the sections it needs.
- Turn a source into a new page or document from an outline.
- Save repeatable AI **recipes** and run them on any file.
- Speak instead of typing: voice input with speech-to-text.
- Semantic search with embeddings.
- AI usage is metered in credits per plan, with budgets that keep spend predictable.

---

## Architecture

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture-dark.png">
  <img alt="Vatary Desk architecture: React PWA behind nginx, NestJS API, Go orchestrator calling a Python document service and a Go AI runtime over gRPC, PostgreSQL with pgvector, Cloudflare R2, LLM providers, and Sentry, Microsoft Clarity, Grafana and Loki for observability" src="docs/architecture-light.png">
</picture>

Five application containers on one host, written in three languages, each in the language that suits its job.

| Service | Stack | Role |
|---|---|---|
| **client** | React, TypeScript, Vite, MUI, TanStack Query, Zustand, Tiptap, Yjs, pdf.js, Konva | The web app and PWA, served by **nginx**, which also proxies the API and WebSockets |
| **server** | NestJS, Prisma, PostgreSQL (Neon) + pgvector | Accounts, files, folders, versions, sharing, billing, real-time collaboration, AI gateway |
| **orchestrator** | Go, gRPC `workflow.v1` | Runs long jobs as workflows with steps, retries and progress |
| **docsvc** | Python, PyMuPDF, LibreOffice, pdf2docx, pikepdf/qpdf, ffmpeg | Conversion, PDF editing, extraction, outlines and chunking for AI |
| **ai-runtime** | Go, gRPC `ai.v1`, SQLite | LLM gateway for 9 providers, with prompt registry, fallbacks, schema checks, cost tracking and an operations console |

### Engineering decisions

- **Blobs, not bytes.** Services pass presigned R2 links (`BlobRef`) to each other and verify the sha256. File contents never pass through the API process.
- **Contracts first.** Each service has a `.proto` contract. Go and Python code is generated from it, and Node loads it at runtime, so all three always use the same contract.
- **Provider keys in one place.** Only `ai-runtime` holds LLM API keys. It picks the model per prompt, falls back to another on `429` or `5xx`, validates the answer against a JSON schema and returns the exact cost of every call.
- **One correlation id per user action.** It travels through every hop, so a click can be followed through Sentry, the logs and the AI console.
- **Tests run inside the Docker build.** A failing build never produces an image.

### Observability

| Tool | What it covers |
|---|---|
| **Sentry** | Errors and performance in the browser (`@sentry/react`) and the API (`@sentry/nestjs`) |
| **Microsoft Clarity** | Session replays and heatmaps, with document content masked |
| **Grafana + Loki** | Logs from every container, dashboards and alerts |
| **AI Runtime console** | Per-call traces, latency p50/p95, cost, fallback and schema-failure rates, and a graph of all model calls behind one user action |

---

## Repository layout

```
client/          React PWA
server/          NestJS API
workflow-runtime/  Go orchestrator
docsvc/          Python document service
ai-runtime/      Go LLM gateway + console
proto/           gRPC contracts shared by all services
docs/            architecture notes and ADRs
```

## Running locally

```bash
# the whole stack
docker compose -f docker-compose.dev.yml up --build

# a single service on its own
cd docsvc && docker compose up --build       # OpenAPI UI on http://localhost:8000/docs
cd ai-runtime && docker compose up --build   # console on http://localhost:8080
```

Without provider keys, `ai-runtime` falls back to a built-in mock provider, so the console and playground still work offline.

---

<div align="center">

Built by **[Andrii Slipchenko](https://andrii.tech)**, product engineer, Kyiv.
[GitHub](https://github.com/AndrewSl105) · [LinkedIn](https://www.linkedin.com/in/andrii-s-284690117/)

</div>
