# PaperGuide AI — Complete Architecture Analysis

## Table of Contents
1. Frontend Flow
2. Backend Flow
3. Agents / Checkers Flow
4. Orchestration Flow
5. End-to-End Workflow
6. Architecture Diagram Data (Layers)
7. Mermaid Flowchart
8. Summary for Presentation

---

## 1. Frontend Flow

### Pages / Screens

| Page | Route | Purpose |
|------|-------|---------|
| **Home** | `/` (`page.tsx`) | Single-page wizard: conference selection -> upload -> analysis -> package generation |
| **Dashboard** | `/dashboard/page.tsx` | Redirects to Home |
| **Upload** | `/upload/page.tsx` | Standalone paper upload page |
| **Report** | `/report/page.tsx` | Displays compliance report with score, issues, passed checks |
| **Packages** | `/packages/page.tsx` | Package generation trigger and download |

### Components Used

- **layout/AppShell** -- sidebar + header wrapper
- **layout/PageContainer** -- max-width content container
- **layout/Sidebar** -- navigation sidebar
- **common/Button** -- primary/secondary/danger/ghost variants
- **common/Card** -- rounded bordered container
- **common/Loader** -- spinner
- **common/ErrorMessage** -- error banner
- **common/Badge** -- status badge (success/warning/error/info)
- **common/EmptyState** -- no-data placeholder
- **dashboard/HeroSection** -- gradient hero with CTA
- **dashboard/WorkflowStepper** -- 5-step progress indicator
- **dashboard/StatsPreview** -- score/issue counts overview
- **conference/ConferenceSelector** -- grid of ConferenceCard components
- **conference/ConferenceCard** -- individual conference card with guidelines
- **conference/GuidelineSummary** -- guideline details display
- **upload/FileUploadBox** -- drag-and-drop / file picker
- **upload/UploadProgress** -- upload/processing progress bar
- **package/PackageGenerationPanel** -- generate package trigger UI

### State Flow

```
Conference selection (stored in localStorage + state)
  -> File selected (client-side validation: type + size)
  -> Upload button -> POST /api/v1/papers/upload
    -> paper_id stored in localStorage
  -> Auto-trigger POST /api/v1/compliance/analyze
    -> complianceReport stored in localStorage + state
  -> User clicks "Generate Package" -> POST /api/v1/packages/generate
    -> package_id stored in localStorage + state
  -> User clicks "Download" -> GET /api/v1/packages/{id}/download
    -> Blob download triggered
```

### API Calls Made per Screen

| Screen | API Call | Method | Endpoint |
|--------|----------|--------|----------|
| Home | Load conferences | GET | /api/v1/conferences |
| Home | Get conference details | GET | /api/v1/conferences/{id} |
| Home/Upload | Upload paper | POST | /api/v1/papers/upload |
| Home/Report | Analyze compliance | POST | /api/v1/compliance/analyze |
| Home/Packages | Generate package | POST | /api/v1/packages/generate |
| Home/Packages | Download package | GET | /api/v1/packages/{id}/download |

---

## 2. Backend Flow

### Routes and Services

| Route File | Method | Path | Service Called | Schema (Request -> Response) |
|------------|--------|------|----------------|------------------------------|
| conferences.py | GET | /api/v1/conferences | ConferenceService.get_all_conferences() | -> list[ConferenceBrief] |
| conferences.py | GET | /api/v1/conferences/search | ConferenceService.filter_conferences() | query params -> list[ConferenceBrief] |
| conferences.py | GET | /api/v1/conferences/{id} | ConferenceService.get_conference_by_id() | -> ConferenceDetailed |
| conferences.py | GET | /api/v1/conferences/{id}/guidelines | ConferenceService.get_guidelines() | -> ConferenceGuidelines |
| upload.py | POST | /api/v1/papers/upload | (inline) | UploadFile + Form -> PaperUploadResponse |
| papers.py | POST | /api/v1/papers | PaperService.create_paper() | PaperCreate -> dict |
| papers.py | GET | /api/v1/papers/{id} | PaperService.get_paper_by_id() | -> dict |
| papers.py | GET | /api/v1/papers/{id}/metadata | PaperService.get_paper_metadata() | -> PaperMetadata |
| papers.py | PUT | /api/v1/papers/{id}/status | PaperService.update_paper_status() | query param -> dict |
| papers.py | DELETE | /api/v1/papers/{id} | PaperService.delete_paper() | -> 204 |
| papers.py | GET | /api/v1/papers/list/conference/{id} | PaperService.list_papers_by_conference() | -> list[dict] |
| compliance.py | POST | /api/v1/compliance/analyze | (inline orchestrator) | ComplianceAnalyzeRequest -> ComplianceReportResponse |
| packages.py | POST | /api/v1/packages/generate | PackageService.generate_package() | PackageGenerationRequest -> PackageGenerationResponse |
| packages.py | GET | /api/v1/packages/{id}/download | (inline) | -> FileResponse (ZIP) |
| generation.py | (all routes) | /papers/{id}/generate, /papers/{id}/generation/status, /papers/{id}/download | **NOT IMPLEMENTED** -- all TODOs | -- |
| projects.py | (all routes) | /projects, /projects/{id}, /projects/{id}/resubmit | **NOT IMPLEMENTED** -- all TODOs | -- |

### ID Flow

```
upload.py: paper_id = uuid4 -> stored in PAPER_STORAGE dict (in-memory)
compliance.py: paper_id reused -> reads from PAPER_STORAGE -> issues generated -> stored in COMPLIANCE_REPORT_STORAGE
packages.py: package_id = uuid4 -> reads from COMPLIANCE_PAPER_STORAGE + COMPLIANCE_REPORT_STORAGE -> ZIP created -> stored in PACKAGE_STORAGE
```

### Database Models

- **Conference** -- id, abbr, name, start_date, submission_deadline, location, flag, conference_type, topics, description, url, acceptance_rate
- **ConferenceGuidelines** -- id, conference_id, max_pages, min_pages, requires_anonymity, reference_format, margins, required_sections, forbidden_topics, special_rules
- **Paper** -- id, conference_id, original_filename, storage_path, title, authors, abstract, page_count, section_structure, status, current_report_id
- **ComplianceIssue** -- id, report_id, paper_id, issue_type, severity, title, description, guideline_reference, suggested_fix, paper_location, detected_by
- **ComplianceReport** -- id, paper_id, conference_id, total_issues, critical_count, warning_count, compliance_score, status
- **Project** -- id, paper_id, conference_id, status, current_report_id

---

## 3. Agents / Checkers Flow

### Currently Connected and Working (5 checkers)

| Checker | File | Input | Processing | Output | Status | Runs |
|---------|------|-------|-----------|--------|--------|------|
| **AnonymityChecker** | checkers/anonymity_checker.py | extracted_text, parsed_paper | Regex for email addresses + keyword detection (author, affiliation, acknowledgment, etc.) | List of dicts with issue_id, category="anonymity", severity, message, location, suggested_fix | **IMPLEMENTED** | Sequential |
| **PageLimitChecker** | checkers/page_limit_checker.py | parsed_paper.page_count, guidelines.max_pages | Compare page count against max pages from guidelines | List of dicts with category="page_limit", severity="critical" if exceeded | **IMPLEMENTED** | Sequential |
| **ReferenceChecker** | checkers/reference_checker.py | parsed_paper.references_found, guidelines | Check if references_found flag is True in parsed output | List with category="references", severity="warning" if missing | **IMPLEMENTED** | Sequential |
| **CitationChecker** | checkers/citation_checker.py | extracted_text, parsed_paper | Regex search for \cite{}, [N], or (Author, Year) patterns | List with category="citations", severity="warning" if none found | **IMPLEMENTED** | Sequential |
| **ClaimsChecker** | checkers/claims_checker.py | extracted_text, parsed_paper | Regex for percentages, decimals in benchmark context, speedup claims | List with category="claims_integrity", severity="warning", needs_verification=True | **IMPLEMENTED** | Sequential |

### Placeholder / Not Yet Connected (5 checkers)

| Checker | File | Status | Reason |
|---------|------|--------|--------|
| **ClaimsIntegrityChecker** | checkers/claims_integrity_checker.py | **PLACEHOLDER** -- all TODOs, returns [] | Uses BaseChecker ABC; not imported in compliance.py |
| **MarginChecker** | checkers/margin_checker.py | **PLACEHOLDER** -- all TODOs, returns [] | Uses BaseChecker ABC; not imported in compliance.py |
| **ReferenceFormatChecker** | checkers/reference_format_checker.py | **PLACEHOLDER** -- all TODOs, returns [] | Uses BaseChecker ABC; not imported in compliance.py |
| **SectionChecker** | checkers/section_checker.py | **PLACEHOLDER** -- all TODOs, returns [] | Uses BaseChecker ABC; not imported in compliance.py |
| **TemplateChecker** | checkers/template_checker.py | **PLACEHOLDER** -- all TODOs, returns [] | Uses BaseChecker ABC; not imported in compliance.py |

### Key Distinction

There are **two separate checker patterns** in the codebase:
1. **Simple dict-based checkers** (used): AnonymityChecker, PageLimitChecker, ReferenceChecker, CitationChecker, ClaimsChecker -- all have a check() method that returns List[Dict] and are **directly instantiated in compliance.py**.
2. **Abstract BaseChecker-based checkers** (not used): ClaimsIntegrityChecker, MarginChecker, ReferenceFormatChecker, SectionChecker, TemplateChecker -- all inherit from BaseChecker with async check() -> List[ComplianceIssue] but are **never imported or instantiated**.

### Execution Model

All 5 working checkers run **sequentially**, one after another in the analyze_compliance function (compliance.py:69-117). The orchestrator does not use parallelism or async gathering.

---

## 4. Orchestration Flow

### Orchestrator

**There is no dedicated orchestrator service.** The orchestration is performed **inline** in the analyze_compliance() function in compliance.py:50-140.

### Orchestration Steps

```
POST /api/v1/compliance/analyze
  -> Validate paper_id exists in PAPER_STORAGE (in-memory dict)
  -> Read file from disk at storage_path
  -> Instantiate ParserService() and call .parse(storage_path, file_type)
    -> Routes to PDFParser / DOCXParser / LatexZipParser
  -> Fetch ConferenceGuidelines from DB via ConferenceService(db).get_guidelines()
  -> Extract "extracted_text" from parsed_paper dict
  -> Initialize empty issues[] and passed_checks[]

  -> Run AnonymityChecker.check(extracted_text, parsed_paper)
    -> If guidelines.requires_anonymity is True
    -> Append issues; if none, add "anonymity" to passed_checks

  -> Run PageLimitChecker.check(parsed_paper, guidelines_dict)
    -> Append issues; if none, add "page_limit" to passed_checks

  -> Run ReferenceChecker.check(parsed_paper, guidelines_dict)
    -> Append issues; if none, add "references" to passed_checks

  -> Run CitationChecker.check(extracted_text, parsed_paper)
    -> Append issues; if none, add "citations" to passed_checks

  -> Run ClaimsChecker.check(extracted_text, parsed_paper)
    -> Append issues; if none, add "claims_integrity" to passed_checks

  -> Calculate readiness_score (100 - 20 per critical - 8 per warning - 2 per info)
  -> Determine overall_status from score + critical_count
  -> Build ComplianceReportResponse with issues, passed_checks, counters
  -> Store report in COMPLIANCE_REPORT_STORAGE[paper_id] (in-memory)
  -> Return ComplianceReportResponse
```

### Score Calculation

```
readiness_score = 100
  critical -> -20 each
  warning -> -8 each
  info    -> -2 each
clamped to [0, 100]

Status mapping:
  score >= 90 AND critical_count == 0 -> "submission_ready"
  score >= 75 AND critical_count == 0 -> "needs_minor_fixes"
  score >= 50                        -> "needs_major_fixes"
  score < 50                         -> "not_ready"
```

### Package Generation Trigger

```
POST /api/v1/packages/generate
  -> Validates paper exists
  -> Validates compliance report exists (MUST run analyze first)
  -> Validates conference_id matches
  -> Parses the file again (second parse!)
  -> Calls PackageService.generate_package():
    -> Instantiates ReportGenerator -> generates compliance_report.json
    -> Instantiates LatexPackageGenerator -> generates main.tex, references.bib, README
    -> Assembles files_dict from parsed_paper.source_files + generated files
    -> Instantiates ZipUtils -> creates .zip from files_dict
    -> Stores package metadata in PACKAGE_STORAGE (in-memory)
  -> Returns PackageGenerationResponse
```

---

## 5. End-to-End Workflow

```
1. User opens homepage
   -> Frontend calls GET /api/v1/conferences
   -> ConferenceSelector shows 6 conference cards (NeurIPS, CVPR, ICML, ACL, EMNLP, etc.)
   -> Only NeurIPS 2025 is marked as "Available"; others are "Coming Soon"

2. User selects NeurIPS 2025
   -> Frontend calls GET /api/v1/conferences/neurips-2025
   -> Backend returns ConferenceDetailed with guidelines (max_pages=9, requires_anonymity=true, reference_format=bibtex)
   -> Selected conference stored in localStorage + component state
   -> FileUploadBox becomes visible

3. User selects a PDF/DOCX/ZIP file
   -> Client-side validation: file extension (.pdf/.docx/.zip), file size (<=50MB)

4. User clicks "Upload & Run Compliance Check"
   -> Frontend creates FormData with file + conference_id
   -> POST /api/v1/papers/upload
   -> Backend validates file, generates uuid paper_id, saves to ./uploads/{paper_id}.{ext}
   -> Returns PaperUploadResponse {paper_id, file_name, file_type, file_size}

5. Frontend auto-triggers compliance analysis
   -> POST /api/v1/compliance/analyze {paper_id, conference_id}
   -> Backend: ParserService.parse(file_path, file_type)
     -> PDFParser: extracts text via PyPDF2, counts pages, detects references
     -> DOCXParser: extracts text via python-docx
     -> LatexZipParser: unzips and parses .tex files
   -> Backend: loads ConferenceGuidelines from SQLite DB
   -> Backend: runs 5 checkers sequentially
     -> AnonymityChecker -> emails + author keywords
     -> PageLimitChecker -> page count vs max_pages
     -> ReferenceChecker -> references_found flag
     -> CitationChecker -> citation patterns
     -> ClaimsChecker -> numeric claims (%, decimals, speedups)
   -> Backend: aggregates issues, calculates readiness_score, determines status
   -> Stores report in COMPLIANCE_REPORT_STORAGE (in-memory dict)
   -> Returns ComplianceReportResponse

6. Frontend displays report
   -> Readiness score with color coding (green >=80, amber >=50, red <50)
   -> Overall status badge
   -> Critical Issues section (red) -- each with message + suggested_fix
   -> Warnings section (amber)
   -> Passed Checks section (green) -- badges for each passed category
   -> StatsPreview updates with counts

7. User clicks "Generate Overleaf Package"
   -> POST /api/v1/packages/generate {paper_id, conference_id, package_type: "neurips_overleaf"}
   -> Backend validates paper + compliance report exist
   -> Backend parses file again (second parse)
   -> PackageService.generate_package():
     -> LatexPackageGenerator generates main.tex (with documentclass, abstract, content)
     -> LatexPackageGenerator generates references.bib (template)
     -> LatexPackageGenerator generates README_OVERLEAF_INSTRUCTIONS.md
     -> ReportGenerator generates compliance_report.json
     -> LatexPackageGenerator generates COMPLIANCE_SUMMARY.txt
     -> ZipUtils.create_package_zip() bundles all files into overleaf_{package_id}.zip
   -> Returns PackageGenerationResponse {package_id, generated_files[], status}

8. Frontend shows success state with file list and Download button

9. User clicks "Download Package"
   -> GET /api/v1/packages/{package_id}/download
   -> Backend returns FileResponse (application/zip)
   -> Frontend creates Blob URL and triggers download as overleaf_{package_id}.zip
```

---

## 6. Architecture Diagram Data (Layers)

### Layer 1: Frontend UI

| Node | Type | Description |
|------|------|-------------|
| [Home Page] | Screen | Main wizard: conference -> upload -> analysis -> package |
| [Upload Page] | Screen | Standalone upload |
| [Report Page] | Screen | Compliance report viewer |
| [Packages Page] | Screen | Package generation + download |

**Connections**: Home -> Upload (nav) -> Report (auto after upload) -> Packages (nav)

### Layer 2: Frontend API Services

| Node | File | Calls |
|------|------|-------|
| conferenceApi | services/conferenceApi.ts | GET /api/v1/conferences, GET /api/v1/conferences/{id} |
| paperApi | services/paperApi.ts | POST /api/v1/papers/upload |
| complianceApi | services/complianceApi.ts | POST /api/v1/compliance/analyze |
| packageApi | services/packageApi.ts | POST /api/v1/packages/generate, GET /api/v1/packages/{id}/download |

**Connections**: All services -> apiClient (axios instance with baseURL from env)

### Layer 3: Backend Routes

| Node | File |
|------|------|
| ConferenceRoutes | routes/conferences.py |
| UploadRoutes | routes/upload.py |
| PaperRoutes | routes/papers.py |
| ComplianceRoutes | routes/compliance.py |
| PackageRoutes | routes/packages.py |

### Layer 4: Backend Services / Orchestrator

| Node | File | Responsibility |
|------|------|---------------|
| ConferenceService | services/conference_service.py | CRUD conferences, search, guidelines fetch |
| PaperService | services/paper_service.py | CRUD papers, status management |
| **ComplianceAnalyzer** | routes/compliance.py (inline) | **Orchestrator**: parse -> checkers -> score -> report |
| ParserService | services/parser_service.py | Routes to PDF/DOCX/LaTeX parser |
| PackageService | services/package_service.py | Generate ZIP from parsed paper + report |
| ReportGenerator | services/report_generator.py | JSON report serialization |
| LatexPackageGenerator | services/latex_package_generator.py | main.tex, references.bib, README generation |
| ZipUtils | utils/zip_utils.py | ZIP creation |

### Layer 5: Parsers

| Node | File | Input | Output Keys |
|------|------|-------|-------------|
| PDFParser | parsers/pdf_parser.py | .pdf | extracted_text, page_count, references_found, citation_patterns_found, sections, title, abstract_found |
| DOCXParser | parsers/docx_parser.py | .docx | Same structure as PDFParser |
| LatexZipParser | parsers/latex_zip_parser.py | .zip | source_files{}, main_tex_content, bib_files{}, plus same structure |

### Layer 6: Agents / Checkers

| Node | File | Status |
|------|------|--------|
| AnonymityChecker | checkers/anonymity_checker.py | **Active** |
| PageLimitChecker | checkers/page_limit_checker.py | **Active** |
| ReferenceChecker | checkers/reference_checker.py | **Active** |
| CitationChecker | checkers/citation_checker.py | **Active** |
| ClaimsChecker | checkers/claims_checker.py | **Active** |
| ClaimsIntegrityChecker | checkers/claims_integrity_checker.py | **Placeholder** |
| MarginChecker | checkers/margin_checker.py | **Placeholder** |
| ReferenceFormatChecker | checkers/reference_format_checker.py | **Placeholder** |
| SectionChecker | checkers/section_checker.py | **Placeholder** |
| TemplateChecker | checkers/template_checker.py | **Placeholder** |

### Layer 7: Generators

| Node | File | Output |
|------|------|--------|
| LatexPackageGenerator | services/latex_package_generator.py | main.tex, references.bib, README_OVERLEAF_INSTRUCTIONS.md, COMPLIANCE_SUMMARY.txt |
| ReportGenerator | services/report_generator.py | compliance_report.json |
| ZipUtils | utils/zip_utils.py | overleaf_{package_id}.zip |

### Layer 8: Storage

| Node | Type | Location |
|------|------|----------|
| SQLite DB | Database | paperguide-ai/backend/paperguide.db |
| PAPER_STORAGE | In-memory dict | routes/compliance.py |
| COMPLIANCE_REPORT_STORAGE | In-memory dict | routes/compliance.py |
| PACKAGE_STORAGE | In-memory dict | services/package_service.py |
| ./uploads/ | Filesystem | uploads/{paper_id}.{ext} |
| ./generated/packages/ | Filesystem | generated/packages/overleaf_{package_id}.zip |
| ./data/conferences.json | JSON file | Static seed data |
| ./data/guidelines/ | JSON files | Guidelines per conference |

### Layer 9: Final Outputs

| Output | Format | Description |
|--------|--------|-------------|
| ComplianceReportResponse | JSON (API) | Readiness score, issues list, passed checks, status |
| overleaf_{id}.zip | ZIP | Overleaf-ready LaTeX project with main.tex, references.bib, README, compliance_report.json, COMPLIANCE_SUMMARY.txt |

---

## 7. Mermaid Flowchart

```
flowchart TB
    subgraph Frontend["Frontend (Next.js + Tailwind)"]
        direction TB
        Home["Home Page\n(conference -> upload -> analysis -> package)"]
        Upload["/upload\nFileUploadBox\n+ UploadProgress"]
        Report["/report\nCompliance Report\nScore + Issues + Passed Checks"]
        Packages["/packages\nPackage Generation\n+ Download"]
    end

    subgraph Services["Frontend API Services (Axios)"]
        ConfAPI["conferenceApi.ts"]
        PaperAPI["paperApi.ts"]
        CompAPI["complianceApi.ts"]
        PkgAPI["packageApi.ts"]
        Axios["apiClient.ts\nbaseURL = NEXT_PUBLIC_API_BASE_URL"]
    end

    subgraph Backend["Backend (FastAPI + Uvicorn)"]
        direction TB
        ConfRoutes["GET /api/v1/conferences\nGET /api/v1/conferences/{id}"]
        UploadRoute["POST /api/v1/papers/upload"]
        CompRoute["POST /api/v1/compliance/analyze\n<-- COMPLIANCE ORCHESTRATOR"]
        PkgRoute["POST /api/v1/packages/generate\nGET /api/v1/packages/{id}/download"]
    end

    subgraph Services2["Backend Services"]
        ConfSvc["ConferenceService"]
        PaperSvc["PaperService"]
        ParserSvc["ParserService"]
        PkgSvc["PackageService"]
        ReportGen["ReportGenerator"]
        LatexGen["LatexPackageGenerator"]
        ZipUtil["ZipUtils"]
    end

    subgraph Parsers["Parsers"]
        PDF["PDFParser\n(PyPDF2)"]
        DOCX["DOCXParser\n(python-docx)"]
        LaTeX["LatexZipParser\n(zipfile + regex)"]
    end

    subgraph Checkers["Active Checkers (Sequential)"]
        Anon["AnonymityChecker\nemails + author keywords"]
        Page["PageLimitChecker\npage_count vs max_pages"]
        Ref["ReferenceChecker\nreferences_found flag"]
        Cite["CitationChecker\n\\cite{}, [N], (Author, Year)"]
        ClaimsC["ClaimsChecker\n%, decimals, speedups"]
    end

    subgraph Placeholders["Placeholder Checkers (Not Connected)"]
        CI["ClaimsIntegrityChecker"]
        Margin["MarginChecker"]
        RefFmt["ReferenceFormatChecker"]
        Section["SectionChecker"]
        Template["TemplateChecker"]
    end

    subgraph Storage["Storage"]
        SQLite[("SQLite\npaperguide.db")]
        MemPAPER[("PAPER_STORAGE\nin-memory dict")]
        MemCOMP[("COMPLIANCE_REPORT_STORAGE\nin-memory dict")]
        MemPKG[("PACKAGE_STORAGE\nin-memory dict")]
        FS_UPLOAD[("./uploads/\n{paper_id}.{ext}")]
        FS_PKG[("./generated/packages/\noverleaf_{id}.zip")]
        Seed[("./data/conferences.json\n+ guidelines/*.json")]
    end

    subgraph Output["Outputs"]
        ZIP["Overleaf ZIP\nmain.tex\nreferences.bib\nREADME.md\ncompliance_report.json\nCOMPLIANCE_SUMMARY.txt"]
        API_JSON["ComplianceReportResponse\n{score, issues, status}"]
    end

    Home --> ConfAPI
    Home --> Upload
    Upload --> Report
    Report --> Packages

    ConfAPI --> Axios --> ConfRoutes
    PaperAPI --> Axios --> UploadRoute
    CompAPI --> Axios --> CompRoute
    PkgAPI --> Axios --> PkgRoute

    ConfRoutes --> ConfSvc --> SQLite
    UploadRoute --> PaperSvc
    UploadRoute --> FS_UPLOAD
    UploadRoute --> MemPAPER

    CompRoute --> ParserSvc
    ParserSvc --> PDF
    ParserSvc --> DOCX
    ParserSvc --> LaTeX
    PDF & DOCX & LaTeX --> MemPAPER

    CompRoute --> ConfSvc --> SQLite
    CompRoute --> MemPAPER

    CompRoute --> Anon --> Page --> Ref --> Cite --> ClaimsC
    CompRoute --> MemCOMP
    CompRoute --> API_JSON

    PkgRoute --> MemPAPER
    PkgRoute --> MemCOMP
    PkgRoute --> ParserSvc
    PkgRoute --> PkgSvc

    PkgSvc --> LatexGen
    PkgSvc --> ReportGen
    PkgSvc --> ZipUtil
    ZipUtil --> FS_PKG
    PkgSvc --> MemPKG
    PkgRoute --> ZIP

    Seed --> SQLite
```

---

## 8. Summary for Presentation

**PaperGuide AI** is a conference-aware research paper submission assistant with a Next.js/TypeScript frontend and FastAPI/Python backend.

The **frontend** provides a 5-step wizard (Select Conference -> Upload Paper -> Run Compliance Checks -> Review Issues -> Generate Overleaf ZIP) with 4 pages at /, /upload, /report, and /packages. State flows through React component state and localStorage.

The **backend** exposes 7 route groups: conferences retrieval, paper upload/CRUD, compliance analysis, and package generation/download. The compliance analysis endpoint in routes/compliance.py acts as the **inline orchestrator** -- it instantiates the ParserService to parse PDF/DOCX/LaTeX files, then runs **5 checkers sequentially**: AnonymityChecker, PageLimitChecker, ReferenceChecker, CitationChecker, and ClaimsChecker. These produce structured issue lists that are aggregated into a ComplianceReportResponse with a calculated readiness score (0-100) and overall status. An additional **5 checkers** (ClaimsIntegrityChecker, MarginChecker, ReferenceFormatChecker, SectionChecker, TemplateChecker) exist as abstract BaseChecker subclasses but are **placeholder-only** and not connected.

The **package generator** (PackageService) takes the parsed paper + compliance report and generates a ZIP containing main.tex, references.bib, README_OVERLEAF_INSTRUCTIONS.md, compliance_report.json, and COMPLIANCE_SUMMARY.txt -- ready for direct upload to Overleaf.

Execution is **purely sequential** with no parallelism. All storage is file-based with SQLite for persistence and in-memory dicts (PAPER_STORAGE, COMPLIANCE_REPORT_STORAGE, PACKAGE_STORAGE) for session data. The final output is a downloadable Overleaf-ready ZIP package.
