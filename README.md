# ATLIZINTLA — Financial Management Platform

> **Architecture Case Study · End-to-End Systems Integration**
> Designed & delivered by **Daniel Almazán** · IT Project Manager & Technical Architect

---

![PHP](https://img.shields.io/badge/Backend-PHP_8.x-777BB4?style=flat-square&logo=php&logoColor=white)
![Python](https://img.shields.io/badge/Services-Python_3.x-3776AB?style=flat-square&logo=python&logoColor=white)
![JavaScript](https://img.shields.io/badge/Frontend-JavaScript_ES6+-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![MySQL](https://img.shields.io/badge/Database-MySQL_/_MariaDB-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Bootstrap](https://img.shields.io/badge/UI-Bootstrap_5-7952B3?style=flat-square&logo=bootstrap&logoColor=white)
![Google Cloud](https://img.shields.io/badge/OCR-Google_Document_AI-4285F4?style=flat-square&logo=google-cloud&logoColor=white)
![PHPMailer](https://img.shields.io/badge/Email-PHPMailer_SMTP-EA4335?style=flat-square&logo=gmail&logoColor=white)
![ReportLab](https://img.shields.io/badge/PDF_Engine-ReportLab_+_pdfrw-CC0000?style=flat-square&logo=adobe-acrobat-reader&logoColor=white)

---

*A Fintech web application that digitizes the full client lifecycle — from KYC onboarding and financial product quoting, through legally-binding digital contract signing, to portfolio tracking and executive reporting — replacing fragmented manual workflows with a unified, role-secured platform.*

---

## Table of Contents

1. [The Business Problem](#1-the-business-problem)
2. [My Role](#2-my-role)
3. [Solution Architecture](#3-solution-architecture)
4. [Technology Stack Deep Dive](#4-technology-stack-deep-dive)
5. [Core Modules Breakdown](#5-core-modules-breakdown)
6. [Security Architecture (Zero Trust)](#6-security-architecture-zero-trust)
7. [Database Design](#7-database-design)
8. [Integration Points](#8-integration-points)
9. [Impact & Results](#9-impact--results)
10. [Lessons Learned & Roadmap](#10-lessons-learned--roadmap)

---

> **⚠️ Confidentiality Notice**
> This repository is a **public architecture case study only**. Per NDA agreements with the client organization, no proprietary source code, credentials, student data, or internal business logic is included. All references describe system capabilities at a design level to demonstrate architectural thinking, systems integration skills, and end-to-end project delivery.

---

## 1. The Business Problem

**Cortés Capital** is a financial services firm managing savings accounts and investment products for a growing client base. Before this platform existed, the operation faced critical bottlenecks:

- **Fragmented Tooling** — Client records, quotations, and contract documents lived across spreadsheets, email threads, and paper files. There was no single source of truth for a client's financial profile.
- **Manual Contract Lifecycle** — Generating contracts required manually filling PDF templates, printing them, coordinating in-person signatures, scanning the signed documents, and filing them. A single contract could take days to formalize.
- **No Digital KYC** — Client onboarding (Know Your Customer) required executives to manually transcribe data from government-issued IDs (INE, CURP, RFC) into forms — a slow and error-prone process.
- **Weak Access Control** — There was no centralized system for managing who could see what. Executives had unrestricted access to all client data, and there was no audit trail for sensitive operations.
- **Zero Operational Visibility** — Management lacked real-time dashboards to monitor portfolio performance, contract pipeline status, or executive productivity.

**The mandate was clear**: build a production-grade platform that could digitize the entire client and contract lifecycle, enforce role-based security, and provide operational intelligence — all within a lean, self-hosted infrastructure.

---

## 2. My Role

**IT Project Manager & Technical Architecture Lead**

I owned this project end-to-end — from requirements gathering with business stakeholders (Operations Director, Compliance, Branch Managers) through architecture design, hands-on development, and deployment to a production hosting environment.

Key responsibilities included:

- Translating business workflows (client onboarding, contract signing, portfolio management) into system requirements and data models.
- Designing the full application architecture: MVC backend, relational database schema, Python microservice for OCR, PDF generation pipeline, and SMTP email integration.
- Implementing the complete codebase: backend controllers, data models, frontend modules, security middleware, and external service integrations.
- Defining and enforcing the security model: TOTP-based 2FA, RBAC, session hardening, CSRF protection, rate limiting, and a remote kill switch for compromised accounts.
- Managing the deployment on a shared hosting environment (Apache/cPanel) with custom cron jobs, Python virtual environments, and Let's Encrypt SSL.

---

## 3. Solution Architecture

A monolithic MVC application with a Python sidecar service, designed for operational simplicity and full control over the data layer.

### 3.1 High-Level Architecture

> **`[Insert Architecture Diagram Here]`**
>
> *Recommended: A block diagram showing the Browser → Apache/.htaccess → PHP Router → Controllers → Models → MySQL flow, with lateral blocks for the Python OCR Service, SMTP Gateway, PDF Generation Pipeline, and File Storage layer.*

### 3.2 Architectural Decisions

| Decision | Rationale |
|:---|:---|
| **PHP Native MVC (no framework)** | Full control over request lifecycle and security middleware. No framework upgrade debt. Minimal server requirements for shared hosting deployment. |
| **Python Sidecar for OCR** | Google Document AI SDK has first-class Python support. Flask microservice runs on a separate port (`5001`), decoupled from the PHP application. Health-check cron ensures uptime. |
| **PDF Generation via Python (ReportLab + pdfrw)** | Overlaying dynamic data and digital signatures onto a pre-designed PDF template required precise coordinate-based rendering. ReportLab's canvas API provided pixel-level control. |
| **Session-Based Auth with TOTP 2FA** | Stateless JWT was unnecessary for this use case. PHP native sessions with hardened cookie configuration, combined with Google Authenticator TOTP, provided bank-grade security with minimal complexity. |
| **Direct File System Storage** | For an MVP serving a single-branch operation, local disk storage (with directory-per-client structure) was pragmatic. The architecture allows migration to S3/Azure Blob by swapping the `FileUploader` service class. |

### 3.3 Request Lifecycle

```
Browser (JS Fetch)
    → Apache mod_rewrite → index.php (Front Controller)
        → Router::dispatch()
            → Controller (validates session, CSRF, RBAC)
                → Model (PDO Prepared Statements → MySQL)
                    ← JSON Response
            ← Rendered View / AJAX Response
```

For contract generation, the flow extends:

```
PHP Controller
    → Writes JSON data file to /tmp
        → shell_exec() → Python generate_pdf.py
            → Reads JSON + PDF Template
            → ReportLab Canvas overlay (text, signatures, metadata)
            → pdfrw merge → Final PDF
        ← File path returned to PHP
    → PHP updates DB record with PDF path
    → PHP sends email notification via SMTP
```

---

## 4. Technology Stack Deep Dive

### Backend (PHP 8.x)
- **Architecture Pattern**: Custom MVC with a lightweight `Router.php` (method + path matching), centralized `Database` singleton (PDO), and `ApiResponse` helper for consistent JSON output.
- **Key Libraries**: PHPMailer 7.x (SMTP with Office 365), Monolog 3.x (structured logging), vlucas/phpdotenv (environment configuration).
- **Security Middleware**: `security.php` acts as a gatekeeper included at the top of every protected endpoint — verifying session validity, account status, 2FA compliance, RBAC permissions, and the remote kill switch flag.

### Services Layer (Python 3.x)
- **OCR Microservice** (`process_image.py`): Flask API wrapping Google Document AI. Accepts image/PDF uploads, extracts text via cloud OCR, then applies regex + spaCy NLP to parse Mexican government IDs (CURP, RFC, INE) and addresses (street, postal code → reverse geocode via Google Maps API).
- **PDF Generation Engine** (`generate_pdf.py`): Reads a JSON payload with client data, quotation terms, beneficiary info, and signature image paths. Uses ReportLab to render text at precise coordinates on each page of a 7-page contract template, then merges overlays onto the original PDF using pdfrw. Generates an 8th metadata page with forensic signature evidence (IP, User-Agent, timestamp, geolocation).

### Frontend (Vanilla JavaScript ES6+)
- **Module Pattern**: Class-based JavaScript (`ClientManager.js`, `accounting.js`) with `async/await` for all server communication. No build step — shipped as ES6 modules loaded with `<script type="module">`.
- **UI Framework**: Bootstrap 5 with a custom "Black Card" design system (dark sidebar, serif display typography via Cinzel, rectangular components with `rounded-0`, silver/platinum accents).
- **Interactive Components**: Signature Pad (canvas-based digital signature capture), Chart.js (dashboard KPIs), Marked.js (internal documentation viewer with Mermaid diagram rendering), Toastify (non-blocking notifications).

### Database (MySQL / MariaDB)
- **Schema**: 18+ tables with full referential integrity (foreign keys, cascading deletes), catalog/lookup tables for normalized dropdowns, and audit tables for forensic evidence.
- **Access Pattern**: 100% Prepared Statements via PDO. Transactional writes (`beginTransaction/commit`) for multi-table operations (client creation, contract formalization).

### Infrastructure
- **Hosting**: Apache on shared hosting (cPanel) with `mod_rewrite` for clean URLs.
- **Email**: Office 365 SMTP via PHPMailer with STARTTLS.
- **SSL**: Let's Encrypt (auto-renewed).
- **Process Management**: Custom bash watchdog script (`keep_alive_ocr.sh`) + cron job to monitor and restart the Python OCR service.

---

## 5. Core Modules Breakdown

### 5.1 Client Management (KYC Onboarding)

The client module handles the full lifecycle from prospect registration to validated profile.

**Key Capabilities:**
- Multi-step registration form (General Info → Identity/Address → Banking) with real-time duplicate detection.
- **OCR-Assisted Data Entry**: Executives upload a photo of a client's INE (national ID) or utility bill. The system sends it to the Python OCR service, which extracts CURP, RFC, address fields, and auto-populates the form — reducing data entry time from ~15 minutes to ~2 minutes per client.
- Digital document repository (`uploads/clientes/{id}/`) with categorized file management (ID documents, proof of address, banking statements).
- Inline document viewer (PDF/image) without requiring file downloads.
- Role-aware visibility: Executives see only their assigned clients; Managers and Admins see the full portfolio.

### 5.2 Savings Products (Quotation → Contract → Portfolio)

The financial core of the platform, managing the complete product lifecycle.

**Quotation Engine:**
- Interactive calculator: executives input principal amount, term (months), periodicity (monthly/biweekly), and annual interest rate. The system computes projected returns in real-time.
- Upsert logic: if a client already has a pending quotation, the system updates it rather than creating duplicates.
- Automatic product-client binding: saving a quotation immediately creates a `tabla_producto_cliente` record with a unique account key (`AHORRO-YYYYMMDD-XXXX`).

**Digital Contract Lifecycle (State Machine):**

```
BORRADOR → ENVIADO → FIRMADO_CLIENTE → FIRMADO_GERENTE → FINALIZADO
                ↓                              ↓
          CORRECCION_REQ ←←←←←←←←←←←←←← RECHAZADO
                ↓
           (Regenerate)
```

- **Draft Generation**: PHP assembles client data, quotation terms, beneficiary details, and manager info into a JSON payload → invokes the Python PDF engine → stores the generated draft at `uploads/contratos/`.
- **Client Signing (External)**: System generates a cryptographic token, emails a unique signing link to the client. The external-facing page (`firmar_externo.php`) displays the PDF, captures a canvas-drawn signature, and records forensic evidence: IP address, User-Agent, UTC timestamp, and GPS coordinates (with user consent).
- **Manager Signing (External)**: Separate token-authenticated page for managerial review and counter-signature. Includes CLABE (interbank key) capture for fund disbursement.
- **Contract Versioning**: Rejected contracts are archived in `contrato_versiones`. The system supports regeneration (creating a clean draft from updated data) and version restoration.
- **Final PDF Stamping**: Once both signatures exist, the Python engine re-renders the contract with both signature images overlaid at precise coordinates on each page, plus an Annex page documenting the full chain of custody.

**Payment Tracking:**
- Period-by-period payment schedule generation based on contract terms.
- Movement registration (deposits, withdrawals, yields, adjustments) with optional proof-of-payment upload.
- Real-time dashboard showing current balance, overdue payments, and coverage status.
- CSV export for accounting reconciliation.

### 5.3 Accounting & Reporting

- **Global Ledger View**: Cross-client movement history with date range, client, and transaction type filters.
- **KPI Dashboard**: Total inflows, outflows, net cash flow — computed server-side from `cuentaahorro_movimientos`.
- **Executive Dashboard**: Chart.js visualizations for client distribution, product mix, and contract pipeline status.
- **Print-Ready Report**: Dedicated `report_print.php` generates a browser-printable executive summary with embedded charts (auto-rendered via Chart.js before `window.print()`).

### 5.4 User Administration

- Role-based user management (Admin, Manager, Executive, Director).
- Automated credential provisioning: system generates a temporary password, emails it to the new user via the branded HTML template.
- Password reset with remote kill switch: resetting a user's password simultaneously sets `force_logout = 1`, terminating any active session.
- Logical deletion (soft delete via `activo` flag) to preserve referential integrity.

### 5.5 Secure Document Vault

A secondary security layer within the documentation module for storing sensitive operational documents (licenses, internal keys).

- Master password authentication (separate from user login).
- 5-minute auto-lock timer with manual lock capability.
- 2FA-gated password regeneration (requires Google Authenticator code to generate a new 15-character master password).
- Full audit trail: every access, upload, download, and failed attempt is logged with user ID, IP, and timestamp.

---

## 6. Security Architecture (Zero Trust)

The platform was designed under a **"never trust, always verify"** principle, critical for a system handling financial data and legally-binding documents.

| Layer | Implementation |
|:---|:---|
| **Authentication** | Username/password (bcrypt hashed via `password_hash`) → Mandatory TOTP 2FA (Google Authenticator) via `PHPGangsta/GoogleAuthenticator`. Two-phase login: credentials create a temporary session (`temp_user_id`); only a valid TOTP code promotes it to a real session (`user_id`). |
| **Session Hardening** | HttpOnly cookies, `SameSite=Lax`, Secure flag on HTTPS, custom session name (`ATLIZINTLA_SESSID`), 15-day `gc_maxlifetime`. Session ID regeneration on privilege escalation. |
| **2FA Re-verification** | Every page load checks `last_2fa_login`. If >15 days since last TOTP verification, the user is forced to re-authenticate — even with a valid session. A 10-second "VIP pass" window after verification prevents redirect loops on concurrent AJAX requests. |
| **Remote Kill Switch** | The `force_logout` flag in the `usuarios` table is checked on every request by `security.php`. Setting it to `1` immediately destroys the target user's session — useful for compromised accounts or terminations. |
| **RBAC** | Three-tier role model enforced both server-side (controller-level checks on `$_SESSION['rol']`) and client-side (conditional UI rendering based on role). Executives cannot access user management or global reports. |
| **CSRF Protection** | Token generated per session (`bin2hex(random_bytes(32))`), injected via `<meta>` tag, validated on all state-changing requests. |
| **Brute Force Protection** | `login_attempts` table tracks failed logins by IP. 4 failures within 20 minutes triggers a temporary lockout (HTTP 429). |
| **SQL Injection Prevention** | 100% PDO Prepared Statements across all models. No raw string interpolation in queries. |
| **File Security** | `view_document.php` acts as a secure proxy: validates session or token, checks file path against `realpath()` to prevent directory traversal, verifies the requesting user has permission to access the specific contract, then streams the file with appropriate MIME headers. |

---

## 7. Database Design

The schema follows a normalized relational model with 18+ tables organized into four domains.

> **`[Insert ERD Diagram Here]`**
>
> *Recommended: An Entity-Relationship Diagram generated from the `db_structure.txt` schema, highlighting the relationships between `tabla_cliente`, `tabla_producto_cliente`, `cuentaahorro_cotizacion`, `cuentaahorro_contrato`, and `contrato_firmas_evidencia`.*

### Key Design Patterns

- **Catalog Tables**: Seven `catalogo_*` tables normalize dropdown values (nationalities, banks, document types, relationship types, payment periodicities, request statuses, contract statuses). This enables UI flexibility without schema changes.
- **Pivot Table for Products**: `tabla_producto_cliente` serves as the junction between clients and products, generating unique account keys and enabling a client to hold multiple products.
- **Forensic Evidence Chain**: `contrato_firmas_evidencia` stores immutable records for each signature event — signer identity, IP, User-Agent, UTC timestamp, document hash snapshot, and a JSON metadata field for geolocation data.
- **Contract Versioning**: `contrato_versiones` archives superseded PDF versions with timestamps and comments, enabling a complete document history.
- **Audit Logging**: `contratos_bitacora` records every state transition in the contract lifecycle (sent, signed, rejected, regenerated) with actor identification and timestamps.

---

## 8. Integration Points

| External System | Integration Method | Purpose |
|:---|:---|:---|
| **Google Document AI** | REST API via Python SDK (`google-cloud-documentai`) | OCR processing of government IDs and utility bills for automated KYC data extraction. |
| **Google Maps Geocoding API** | HTTP GET from Python | Reverse geocode postal codes to auto-fill city and state fields during OCR processing. |
| **Google Authenticator (TOTP)** | PHP library (`GoogleAuthenticator.php`) | Time-based one-time password generation and verification for 2FA. QR code generation via `api.qrserver.com`. |
| **Office 365 SMTP** | PHPMailer with STARTTLS on port 587 | Transactional email: contract signing links, credential provisioning, recovery codes, and notification alerts. Branded HTML templates with "Black Card" design. |
| **spaCy NLP** | Python (`es_core_news_md` model) | Named Entity Recognition on OCR-extracted text to identify addresses, organizations, and person names in Spanish-language documents. |

---

## 9. Impact & Results

| Metric | Before | After |
|:---|:---|:---|
| **Client Onboarding Time** | ~45 min (manual transcription from IDs, paper forms) | ~10 min (OCR-assisted, digital forms with validation) |
| **Contract Formalization** | 3–5 business days (print → sign → scan → file) | < 24 hours (digital generation → email link → e-signature → auto-archive) |
| **Data Entry Errors** | Frequent (manual transcription of CURP, RFC, CLABE) | Near-zero (OCR extraction + format validation) |
| **Operational Visibility** | None (spreadsheet-based, retrospective) | Real-time dashboards with KPIs, payment tracking, and pipeline status |
| **Security Posture** | Single-factor, shared credentials | Mandatory 2FA, RBAC, session hardening, audit trails, remote kill switch |
| **Document Management** | Physical files, no versioning | Digital repository per client, contract versioning, forensic signature evidence |
| **Compliance Readiness** | Manual, no audit trail | Immutable signature evidence (IP, device, timestamp, geolocation, document hash) |

### Scalability Achieved

- **Platform handles the full product lifecycle** from initial prospect to active portfolio management in a single system.
- **Modular product architecture**: the savings module (`CTAH`) is fully operational; the schema and UI are pre-built for three additional product types (Flexible Savings, Current Account, Term Investment) — toggled on when business is ready.
- **Self-documenting system**: an embedded Markdown-based documentation portal (`modules/docs/`) with Mermaid diagram rendering ensures institutional knowledge persists beyond any single team member.

---

## 10. Lessons Learned & Roadmap

### What Worked Well
- **Python sidecar pattern**: keeping the OCR and PDF generation in a separate Flask service allowed independent scaling, debugging, and dependency management without polluting the PHP environment.
- **Token-based external signing**: the contract signing pages (`firmar_externo.php`, `firmar_gerente_externo.php`) require no user account — just a cryptographic token. This dramatically simplified the client experience and eliminated the need for client-side account provisioning.
- **Security-first design**: building 2FA, RBAC, and session hardening into the architecture from day one (rather than bolting them on later) resulted in a coherent security model that business stakeholders could trust with financial data.

### Technical Debt Acknowledged
- **Debug artifacts in production root**: multiple `debug_*.php` and `test_*.php` files need removal before any public-facing deployment.
- **Hardcoded product ID**: the savings product key `CTAH001` is embedded in the model layer rather than externalized to configuration.
- **Signature hash is simulated**: the current `hash_snapshot` uses a time-based hash rather than computing SHA-256 over the actual PDF binary. A future iteration should hash the real document content.
- **Frontend state management**: client-side state relies on global variables and `innerHTML` rendering, which introduces XSS surface area and makes complex UI state difficult to manage. A migration to a lightweight reactive library or Web Components is recommended.

### Future Roadmap
- [ ] Migrate file storage to S3-compatible object storage with pre-signed URLs.
- [ ] Implement real document hashing (SHA-256 of PDF binary) for legally defensible digital signatures.
- [ ] Add WebSocket-based real-time notifications for contract status changes.
- [ ] Build a client-facing portal for self-service account viewing and document downloads.
- [ ] Containerize the application (Docker) for environment parity and simplified deployment.

---

*Specializing in end-to-end systems integration, fintech workflows, and secure web application architecture.*

## Contact

**Daniel Almazán**
IT Project Manager · Technical Architect · Systems Integration Specialist

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/daniel-almazan-l/)
[![Email](https://img.shields.io/badge/Email-Contact_Me-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:daniel.almazan.lopez16@gmail.com)
[![Portfolio](https://img.shields.io/badge/Portfolio-View_More-000000?style=for-the-badge&logo=github&logoColor=white)](https://github.com/danielalmazan-manager?tab=repositories)

---

<sub>This case study was authored for portfolio purposes. No proprietary code, credentials, or protected data is included in this repository. All technical descriptions reflect architectural decisions and system capabilities at a design level.</sub>
