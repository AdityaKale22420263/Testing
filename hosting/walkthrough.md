## Architecture Overview

The project is a full-stack application:
- **Frontend:** React + TypeScript (Vite), using TailwindCSS and shadcn/ui components
- **Backend:** Python Flask REST API, using Google Gemini AI for intelligent document processing
- **AI Engine:** Gemini 2.5 (Pro/Flash) for content enrichment, image captioning, register extraction, and image-to-SVG conversion

---

## 🔁 The Agentic AI Pipeline

The system implements an **agentic AI architecture** where the AI doesn't just respond to a single prompt — it acts as an autonomous agent that orchestrates a multi-step document transformation pipeline with decision-making at each stage. Here's how the agentic behavior manifests:

1. **Autonomous Document Analysis Agent:** When a document is uploaded, the system's AI agent autonomously analyzes the content, identifies document structure (headings, paragraphs, tables, images), and classifies each block. It decides what is a heading vs body text based on font size, style, and contextual clues — without human intervention.

2. **Register Detection Agent:** A specialized sub-agent (`RegsTableDetector`) scans documents using pattern-matching heuristics to autonomously identify semiconductor register specification tables. It scores each candidate on multiple signals (bit diagrams, offset patterns, field indicators) and makes an independent decision on whether a section is a register spec — a judgment call, not a simple rule.

3. **Quota-Aware Self-Regulating Agent:** The system includes a self-regulating agent layer (`GeminiRateLimiter` + quota checks in `MultiIpConverter`) that monitors API quota in real-time and autonomously decides which operations to prioritize, skip, or defer. For example, if quota is low, it will skip optional image-to-SVG conversion but prioritize register extraction. This is goal-directed autonomous behavior — the agent adapts its plan based on resource constraints.

4. **AI Enrichment Agent:** After segmentation, an enrichment agent passes through the document structure and autonomously decides which images need AI-generated captions, which tables need AI-generated titles, and optionally simplifies complex paragraphs for non-expert audiences. Each decision is made per-content-block.

5. **Multi-Step Orchestration with Feedback Loops:** The `MultiIpConverter` acts as a master orchestrating agent that runs a multi-phase pipeline (Extract → Detect → Segment → Enrich → Write → Convert → Package), monitoring results after each phase, logging progress, and adjusting downstream behavior based on upstream outcomes. If extraction yields no blocks, it short-circuits. If AI enrichment fails, it gracefully degrades. This is classic agentic loop behavior: Observe → Decide → Act → Observe.

6. **Iterative Retry with Self-Correction:** The `GeminiProcessor` implements retry logic with multiple model fallbacks and self-correcting JSON parsing — if the AI returns malformed output, it retries with a different model automatically, exhibiting autonomous error recovery.

---

## Backend Files

### Core Application

**`backend/app.py`** — The main Flask application entry point. Registers all API route blueprints (DITA conversion, REGS conversion, image conversion, multi-IP processing, spec editor, history), configures CORS for frontend communication, and sets up upload/output directories.

**`backend/config.py`** — Centralized configuration for the entire backend. Stores Gemini API settings (model name, temperature, token limits, safety settings), file upload limits, allowed file extensions, and folder paths. All settings are loaded from environment variables with sensible defaults.

**`backend/run_server.py`** — Production server launcher using Waitress (instead of Flask's dev server) with extended timeouts (600 seconds) and multi-threading (8 threads). This was needed because document conversion can take several minutes and Flask's dev server would drop the connection.

### Extractors (`backend/extractors/`)

**`docx_extractor.py`** — Extracts structured content from Word documents using python-docx. Pulls out paragraphs with heading styles, embedded images (via OOXML blip references), and tables with full merge/span support (handles both horizontal gridSpan and vertical vMerge). Returns a flat list of typed content blocks.

**`pdf_extractor.py`** — Extracts content from PDFs using PyMuPDF (fitz). Captures text with font metadata (size, flags, font name) which is later used to infer heading levels. Also extracts embedded images as PNG bytes. Each block is tagged with its source page number.

**`regs_base_extractor.py`** — Abstract base class that defines the interface for all register document extractors. Enforces that every extractor implements an `extract(filepath) -> str` method.

**`regs_excel_extractor.py`** — Extracts register data from Excel spreadsheets using pandas. Features advanced table detection, register pattern recognition (looking for offset addresses, bit fields, access types), layout-preserving data extraction, and numeric pattern analysis across multiple sheets.

**`regs_word_extractor.py`** — Extracts register specifications from Word documents. Parses paragraphs and tables, detects section breaks at headings, and applies register pattern detection (offset addresses, bit ranges, access types) across the full document text.

**`regs_pdf_extractor.py`** — Extracts register specification data from PDF files. Similar to the general PDF extractor but specialized for identifying and formatting register tables with enhanced table detection.

### Pipeline (`backend/pipeline/`)

**`segmenter.py`** — The document structure builder. Takes a flat list of extracted content blocks and builds a hierarchical topic tree based on heading levels. Uses font size and style information to determine heading depth (H1–H6). The output is a nested tree where each heading becomes a node with its content and child sections — this directly maps to DITA's topic structure.

**`enrich.py`** — The AI enrichment stage of the pipeline. Iterates over the document structure and uses Gemini AI to: (1) generate captions for images that don't have them, and (2) generate descriptive titles for tables. This is where the AI agent enhances the raw extracted content with semantic understanding.

### Services (`backend/services/`)

**`dita_converter.py`** — The main orchestrator service for single-document DITA conversion. Chains together the full pipeline: extract content → segment into topic tree → enrich with AI → write DITA files. Handles both PDF and DOCX inputs and produces a complete DITA topic set with a ditamap.

**`multi_ip_converter.py`** — The most complex service — handles converting multiple IP specification documents in one session. For each IP, it runs the full pipeline (extract → detect registers → segment → enrich → write DITA → convert images to SVG → process register tables → generate REGS files). Then it assembles everything into an organized PDF and ZIP archive. Implements quota-aware processing that intelligently skips non-essential operations when API limits are running low.

**`any_to_regs_converter.py`** — Converts any supported document format (PDF, Word, Excel) to NXP-format .REGS XML. Uses the appropriate extractor to pull raw text, sends it to Gemini AI for structured register data extraction, then generates standardized .REGS XML through the XML generator.

**`image_to_svg_converter.py`** — Converts raster images (PNG, JPG) to SVG using Gemini AI. Includes specialized prompts for different diagram types: flowcharts, waveform diagrams, block diagrams, and power architecture diagrams. Each prompt instructs the AI to preserve connection geometry, signal fidelity, and component accuracy. Includes built-in rate limiting to avoid API quota exhaustion.

**`regs_pdf_converter.py`** — Converts .REGS XML files into professional PDF documents using ReportLab. Parses the register XML, extracts register definitions (name, offset, width, reset value, bit fields), and renders them as formatted PDF pages with bit-field diagrams and field description tables.

### Utilities (`backend/utils/`)

**`gemini_client.py`** — A wrapper around the Google Gemini API that provides high-level methods: `caption_image()` for generating image descriptions, `title_table()` for generating table titles, and `simplify_text()` for rewriting paragraphs. Supports both the new `google-genai` and legacy `google-generativeai` SDKs with automatic fallback.

**`gemini_processor.py`** — Specialized Gemini processor for register data extraction. Sends formatted document text to the AI with a detailed prompt asking it to extract register names, offsets, widths, reset values, and bit field definitions. Includes retry logic with multiple model fallbacks, structured JSON parsing with error correction, and debug output saving.

**`gemini_rate_limiter.py`** — A singleton rate limiter that globally tracks all Gemini API calls across the application. Enforces per-minute and per-day limits (5 RPM, 20 RPD for free tier), implements minimum delay between calls, and supports API-requested forced delays. Prevents quota exhaustion during batch processing.

**`xml_generator.py`** — Generates NXP-standard .REGS XML from structured register data. Produces compliant XML with proper namespace declarations, register definitions, bit field specifications (including reserved fields), and enumerated values. Handles XML entity escaping and formatting.

**`regs_table_detector.py`** — Heuristic-based detector for register specification tables within documents. Looks for signature patterns specific to semiconductor register docs: "Bits 31" headers, offset patterns like "CTRL 4h", bit range notations, access descriptors ("0b0 (read):"), and field/RESERVED indicators. Scores candidates and returns confidence levels.

**`helpers.py`** — Utility functions used across the backend. Includes `slugify()` for generating clean DITA-compatible filenames from titles (with domain-specific keyword mapping for semiconductor terms), `is_bullet()` for detecting list items, and `make_safe_filename()` for filesystem-safe naming.

### Writers (`backend/writers/`)

**`dita_writer.py`** — Generates DITA XML output files from the document structure. Creates individual `.dita` topic files for each section (with proper DITA XML structure: topic → title → body → paragraphs/tables/images) and a `.ditamap` file that defines the topic hierarchy with `topicref` elements. Handles table spans, image references, and nested sections.

**`pdf_writer.py`** — Generates PDF documents from the document structure using ReportLab. Supports both single-topic and multi-topic PDF generation, table rendering with span support, image embedding, list formatting, and optional AI-powered text summarization. The `write_multi_ip_pdf()` variant handles the multi-chapter SOC documentation format with REGS page images.

### Routes (`backend/routes/`)

**`conversion.py`** — REST endpoint for PDF/DOCX → DITA conversion. Accepts file upload via multipart form data, validates file type, saves to uploads directory, invokes `DITAConverter`, and returns session ID with conversion stats. Also provides download endpoints for the generated ZIP of DITA files.

**`regs_conversion.py`** — REST endpoint for .REGS → PDF conversion. Accepts .regs file upload, invokes `RegsToPDFConverter`, and returns the generated PDF for download.

**`any_to_regs_conversion.py`** — REST endpoint for PDF/Word/Excel → .REGS conversion. Accepts any supported document, invokes `AnyToRegsConverter` with Gemini AI processing, and returns the generated .REGS file for download.

**`image_conversion.py`** — REST endpoint for Image → SVG conversion. Accepts image upload with optional diagram type hint (flowchart, waveform, block_diagram, power_architecture, or auto-detect), invokes `ImageToSVGConverter`, and returns the generated SVG.

**`multi_ip_conversion.py`** — REST endpoint for the Multi-IP Documentation Generator. Accepts multiple file uploads simultaneously, invokes `MultiIpConverter` for batch processing, and provides download endpoints for the organized PDF and ZIP archive.

**`spec_source_editor.py`** — REST endpoints for the in-browser spec source editor. Provides APIs to: browse the folder structure of generated spec_source files, read file contents (DITA XML, REGS XML, ditamaps), save edited files back, and trigger PDF regeneration from modified sources.

**`spec_to_pdf_conversion.py`** — REST endpoint for converting existing spec_source ZIP archives directly to PDF. Accepts a ZIP containing DITA topics, ditamaps, REGS files, and images, unpacks it, and generates an organized PDF document.

**`history.py`** — REST endpoints for conversion history management. Scans the outputs and spec_outputs directories to discover past conversions (DITA, REGS, SVG, multi-IP), returns metadata (filename, date, type, size), and provides delete functionality.

---

## Frontend Files

### Core

**`src/App.tsx`** — The root React component. Sets up routing (React Router), global providers (React Query, Tooltip, Toast notifications), and wraps everything in the sidebar layout. Defines all application routes mapping URL paths to page components.

**`src/main.tsx`** — Application bootstrap file. Mounts the React app to the DOM with StrictMode enabled.

**`src/index.css`** — Global styles including TailwindCSS imports, custom CSS variables for theming (light/dark mode), and custom utility classes like `cyber-glow` effects for the dark theme.

### Layout Components (`src/components/layout/`)

**`Layout.tsx`** — The main application shell component. Provides the sidebar + top nav + content area layout structure using shadcn's SidebarProvider.

**`AppSidebar.tsx`** — The navigation sidebar with links to all modules: Home, Multi-IP Doc Generator, Spec Source → PDF, PDF → DITA, DOCX → DITA, .REGS → PDF, Any → .REGS, Image → SVG, Upload History, and About. Supports collapsed icon-only mode.

**`TopNav.tsx`** — The top navigation bar with the sidebar toggle trigger and application branding.

### Shared Components (`src/components/`)

**`FileUpload.tsx`** — Reusable drag-and-drop file upload component with progress bar. Supports both single and multiple file selection, drag-over visual feedback, file type filtering via the `accept` prop, and displays selected file names. Used across all conversion modules.

**`LogViewer.tsx`** — Real-time process log display component. Shows timestamped log entries in a scrollable terminal-style card, with color-coded entries (green for success, red for errors, yellow for warnings, gray for info). Gives users visibility into what the backend is doing.

**`ModuleCard.tsx`** — Card component used on the Home page to display each conversion module with its icon, title, description, and an "Open Module" button that navigates to the module's route.

**`SpecSourceEditor.tsx`** — A full in-browser file editor dialog. After Multi-IP generation, users can open this to browse the generated folder structure (DITA topics, ditamaps, REGS files, images), edit XML/DITA content in a textarea, save changes back to the server, and trigger PDF regeneration — all without leaving the browser.

**`NavLink.tsx`** — A styled wrapper around React Router's NavLink that applies active-state CSS classes for sidebar navigation highlighting.

**`ThemeToggle.tsx`** — Dark/light mode toggle button for switching the application theme.

### Module Pages (`src/pages/modules/`)

**`MultiIpDocGen.tsx`** — The Multi-IP Documentation Generator page. Allows uploading multiple IP specification documents at once, shows real-time processing logs, displays progress, and provides download buttons for the organized PDF and ZIP archive. Also integrates the Spec Source Editor for post-generation editing.

**`SpecToPdf.tsx`** — Spec Source to PDF converter page. Accepts a ZIP file containing pre-existing spec_source folders and converts them to an organized PDF document.

**`PdfToDita.tsx`** — PDF to DITA conversion page. Single file upload with progress tracking, log viewing, and download of the generated DITA ZIP package.

**`DocxToDita.tsx`** — Word document to DITA conversion page. Identical in structure to PdfToDita but accepts .docx/.doc files.

**`RegsToPdf.tsx`** — .REGS to PDF conversion page. Upload a .regs XML file and download the rendered PDF with register bit diagrams.

**`AnyToRegs.tsx`** — Any-format to .REGS conversion page. Upload PDF, Word, or Excel documents containing register specifications and get back a structured .REGS XML file.

**`ImageToSvg.tsx`** — Image to SVG conversion page. Upload a raster image, optionally select the diagram type (flowchart, waveform, block diagram, power architecture), and get back an AI-generated SVG. Includes error handling for quota exhaustion with retry support.

### Other Pages (`src/pages/`)

**`Home.tsx`** — The landing page displaying a grid of all available conversion modules as clickable cards.

**`History.tsx`** — Conversion history page. Fetches and displays all past conversions from the server with download and delete actions.

**`Settings.tsx`** — Settings page with toggles for auto-convert on upload and notification preferences.

**`About.tsx`** — About page with application information.

**`Index.tsx` / `NotFound.tsx`** — Index redirect and 404 page.

### Hooks & Utilities (`src/hooks/`, `src/lib/`)

**`use-mobile.tsx`** — Custom hook for detecting mobile viewport breakpoints.

**`use-toast.ts`** — Custom hook wrapping the toast notification system.

**`utils.ts`** — Utility function `cn()` for merging Tailwind CSS class names using clsx and tailwind-merge.

---

## Config Files

**`vite.config.ts`** — Vite build configuration with React plugin and path aliases (@/ → src/).

**`tailwind.config.ts`** — TailwindCSS configuration with custom colors (neon-green, cyber theme), dark mode support, and shadcn/ui integration.

**`package.json`** — Frontend dependencies including React 18, React Router, TanStack React Query, shadcn/ui components, Lucide icons, and Sonner toast notifications.

**`requirements.txt`** — Backend Python dependencies including Flask, PyMuPDF, python-docx, pandas, ReportLab, google-generativeai, Pillow, lxml, and Waitress.
