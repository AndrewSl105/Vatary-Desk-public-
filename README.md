<div align="center">

# Vatary Desk

**A document workspace with AI built in.**
<img width="1269" height="952" alt="image" src="https://github.com/user-attachments/assets/cf2059a0-6108-4e67-ab61-dbb1b604b88b" />

Open almost any file, edit PDFs in place, keep everything in folders with full version history, and share it with your team. All in one browser tab.

**[Open Vatary Desk →](https://vatarydesk.com)** · **[AI console live demo →](https://demo-ai-staging.vatarydesk.com/)**

[Architecture](#architecture) · [Author](https://andrii.tech)

</div>

---

## Try it

| | |
|---|---|
| 🖥 **The product** | [vatarydesk.com](https://vatarydesk.com): the live workspace |
| 📊 **AI operations console** | [demo-ai-staging.vatarydesk.com](https://demo-ai-staging.vatarydesk.com/): explore the LLM gateway's console with sample traffic. No account needed |

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
- **Built for big files.** Uploads go straight to storage and processing runs in the background, so a 500-page manual does not slow the app down.

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
- **Real-time co-editing** of pages: see who else is in the document, with simultaneous changes merged automatically.
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
  <img alt="Vatary Desk architecture: a React PWA behind nginx, a NestJS API, a workflow orchestrator calling a document service and an AI gateway over gRPC, PostgreSQL with pgvector, Cloudflare R2, LLM providers, and Sentry, Microsoft Clarity, Grafana and Loki for observability" src="docs/architecture-light.png">
</picture>

A service-oriented system in **TypeScript, Go and Python**, each language used where it is strongest. Everything runs in Docker containers behind a single reverse proxy.

| Layer | Technology | Responsibility |
|---|---|---|
| **Web app** | React, TypeScript, Vite, MUI, TanStack Query, Zustand, Tiptap, Yjs, pdf.js, Konva | Installable PWA with offline cache, rich editors and real-time collaboration |
| **API** | NestJS, Prisma | Accounts, files, folders, versions, sharing and roles, billing, real-time sync, AI features |
| **Orchestrator** | Go, gRPC | Long-running jobs as workflows with steps, retries and progress |
| **Document service** | Python, PyMuPDF, LibreOffice, pdf2docx, qpdf, ffmpeg | Conversion, PDF editing, extraction and document structure for AI |
| **AI gateway** | Go, gRPC | One entry point to 9 LLM providers: prompt registry, model fallbacks, schema validation, cost tracking |
| **Data** | PostgreSQL + pgvector, Cloudflare R2 | Relational data and vector search, object storage for files and versions |
| **Infrastructure** | Docker Compose, nginx | Containerised services, TLS, reverse proxy for HTTP and WebSockets |

### Engineering decisions

- **Files never pass through the API.** The browser uploads straight to object storage with signed links, and services exchange links instead of bytes. Large PDFs never sit in application memory, and the API stays fast under heavy uploads.
- **Heavy work runs outside the request path.** Conversion, PDF edits and AI pipelines run as orchestrated workflows with retries and progress reporting, so a slow job never blocks the interface.
- **Strict contracts between services.** Every service boundary is a versioned gRPC contract shared by the TypeScript, Go and Python code, so the services cannot drift apart.
- **AI under control.** Only one service holds provider keys. It picks a model per task, falls back to another provider on errors, validates every answer against a schema and records the exact cost of each call. Usage is metered per plan.
- **Long documents without huge prompts.** The AI first reads a document's outline, then pulls only the sections it needs. Tables reach the model whole.
- **Collaboration without conflicts.** Pages sync in real time through CRDTs, so simultaneous edits merge instead of overwriting each other.
- **Quality gates in the build.** Over 1,100 test files across the stack. Tests run inside the Docker build, so a failing build never produces an image.

### Observability

Every user action gets one correlation id that travels through every service, so a single click can be traced end to end.

| Tool | What it covers |
|---|---|
| **Sentry** | Errors, performance and releases in the web app and the API |
| **Microsoft Clarity** | Session replays and heatmaps, with document content masked |
| **Grafana + Loki** | Logs from every container, dashboards and alerts |
| **AI operations console** | Per-call traces, latency p50/p95, cost, fallback and error rates, and a graph of all model calls behind one user action. [Live demo →](https://demo-ai-staging.vatarydesk.com/) |

---

<div align="center">

Built by **[Andrii Slipchenko](https://andrii.tech)**, product engineer, Kyiv.
[GitHub](https://github.com/AndrewSl105) · [LinkedIn](https://www.linkedin.com/in/andrii-s-284690117/)

</div>
