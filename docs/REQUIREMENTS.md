# AI Agent for Health Management

## Requirements & Technical Specification Document

# **1\. Project Overview**

The Health Companion Agent is a conversational artificial intelligence agent designed to help individuals manage the health of themselves and their pets. Through integration with messaging services, the agent allows users to register appointments, medications, schedules, prescriptions, and exams in a conversational and natural way.

The system combines structured storage (PostgreSQL) with semantic search (pgvector) to provide precise responses both in structured queries via Text-to-SQL and in contextual similarity searches.

## **1.1 Main Objectives**

* Centralize health information for all individuals in a single conversational agent
* Enable registration and querying of medical appointments, medications, and schedules via text
* Generate exportable reports and calendars in PDF format for offline use
* Offer semantic search on appointment history and structured search on tabular data
* Keep operational costs low by using AI models via OpenRouter
* Be accessible via WhatsApp or a lower-cost alternative

## **1.2 Target Audience**

* People who want to organize personal health information in a practical way
* Pet owners who need to manage vaccines, appointments, and medications
* Caregivers and family members who monitor the health of others

# **2\. Technology Stack**

| Layer | Technology | Rationale |
| :---- | :---- | :---- |
| Language | Python 3.12+ | Robust ecosystem for AI and automation |
| Agent Framework | LangChain / LangGraph | Orchestration of complex stateful flows with tools |
| Observability | LangSmith | Tracing, debugging, and production monitoring |
| Database | PostgreSQL 16+ | Structured data with JSON and full-text search support |
| Semantic Search | pgvector | Embeddings and similarity search integrated with PostgreSQL |
| Containerization | Docker / Docker Compose | Reproducible environment and simplified deployment |
| Package Manager | uv | Fast installation and deterministic dependency resolution |
| Linter / Formatter | Ruff | Fast linting and formatting in a single tool |
| Automation | Makefile | Shortcuts for common dev, test, and deploy commands |
| AI Models | OpenRouter | Access to multiple models with free and low-cost options |
| Testing | pytest \+ pytest-asyncio | Testing framework with async support and fixtures |
| PDF Generation | WeasyPrint / ReportLab | PDF generation from HTML/CSS templates or programmatically |
| Messaging (WhatsApp) | Twilio WhatsApp Business API | Official provider with robust API and sandbox for testing |
| Messaging (Telegram) | Telegram Bot API | Free channel for development and testing |

## **2.1 Environment Setup**

### **2.1.1 Directory Structure**

**health-companion-agent/**
```
├── src/
│   ├── agent/ — LangGraph graph definition, nodes and edges
│   ├── tools/ — agent tools (SQL, semantic search, PDF, etc.)
│   ├── models/ — SQLAlchemy models and Pydantic schemas
│   ├── services/ — business logic (medications, appointments, consultations)
│   ├── integrations/ — messaging connectors (WhatsApp, Telegram)
│   ├── templates/ — HTML/CSS templates for PDF generation
│   └── config.py — settings and environment variables
├── tests/ — unit and integration tests
├── migrations/ — Alembic migrations
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── pyproject.toml
└── ruff.toml
```

### **2.1.2 Makefile (Main Commands)**

| Command | Action |
| :---- | :---- |
| make setup | Installs dependencies with uv and configures environment |
| make dev | Starts Docker containers in development mode |
| make test | Runs test suite with pytest (unit \+ integration) |
| make lint | Runs ruff check and ruff format \--check |
| make format | Applies ruff format automatically |
| make migrate | Runs Alembic migrations |
| make build | Builds Docker images for production |
| make down | Stops all containers |

### **2.1.3 Ruff (Configuration)**

Ruff will be configured via ruff.toml in the project root with the following guidelines: target Python 3.12, line length 120, selected rules including pyflakes (F), pycodestyle (E/W), isort (I), flake8-bugbear (B), and flake8-simplify (SIM). Format will use double quote-style and spaces for indentation.

# **3\. System Architecture**

The architecture follows the agent-with-tools pattern (tool-calling agent) implemented with LangGraph. The agent graph has specialized nodes that are activated according to the intent detected in the user's message.

## **3.1 Main Agent Flow**

* User sends message via messaging service
* Webhook receives the message and sends it to the LangGraph graph
* Intent classification node identifies the type of request
* Routing to the specialized node (registration, query, PDF generation, etc.)
* Appropriate tool is called (SQL, pgvector, PDF generator)
* Response is formatted and sent back to the user

## **3.2 LangGraph Graph Nodes**

| Node | Responsibility | Tools |
| :---- | :---- | :---- |
| intent\_classifier | Classifies the intent of the user message | LLM (lightweight/free model) |
| consultation\_recorder | Records consultation reports with embeddings | pgvector, embedding model |
| consultation\_search | Semantic search on consultation history | pgvector similarity search |
| appointment\_manager | CRUD for appointments via Text-to-SQL | SQL tool, PostgreSQL |
| medication\_manager | CRUD for medications via Text-to-SQL | SQL tool, PostgreSQL |
| pdf\_generator | Generates PDFs for calendars and reports | WeasyPrint, templates |
| general\_chat | General responses and guidance | LLM |

## **3.3 Model Strategy (OpenRouter)**

To optimize costs, different models will be used depending on task complexity:

| Task | Model Type | OpenRouter Examples |
| :---- | :---- | :---- |
| Intent classification | Free / Ultra-lightweight | meta-llama/llama-3.1-8b-instruct:free, google/gemma-2-9b-it:free |
| Text-to-SQL | Low cost / Medium | mistralai/mistral-7b-instruct, deepseek/deepseek-coder |
| Embedding generation | Dedicated embedding | BAAI/bge-large-en-v1.5 (local) or via API |
| Complex responses | Medium | anthropic/claude-3.5-haiku, meta-llama/llama-3.1-70b-instruct |
| OCR and image analysis (future) | Vision | google/gemini-flash, meta-llama/llama-3.2-11b-vision |

# **4\. Detailed Features**

## **4.1 Medical Appointment Registration and Query**

Description: The user sends a descriptive text about what happened in a medical appointment (human or veterinary). The agent processes it, generates an embedding of the text, and stores it in the database.

### **Registration Flow**

* User sends: "I went to the cardiologist today, Dr. Silva. He ordered an electrocardiogram and said my blood pressure is a bit high. He prescribed losartan 50mg."
* Agent extracts metadata (date, specialty, doctor, type: human/animal)
* Full text is stored \+ embedding generated via embedding model
* Agent confirms: "Appointment recorded\! Cardiology with Dr. Silva on 04/17/2026."

### **Query Flow**

User asks: "What did the doctor say about my blood pressure?" → Semantic search via pgvector returns the most relevant appointments, and the agent synthesizes the response.

### **Data Model**

| Field | Type | Description |
| :---- | :---- | :---- |
| id | UUID | Unique identifier |
| user\_id | UUID | Reference to the user |
| subject\_type | ENUM | 'human' or 'animal' |
| subject\_name | VARCHAR | Patient/animal name |
| specialty | VARCHAR | Medical specialty |
| doctor\_name | VARCHAR | Professional name |
| consultation\_date | DATE | Appointment date |
| raw\_text | TEXT | Full text sent by user |
| summary | TEXT | Summary generated by agent |
| embedding | VECTOR(1536) | Text embedding for semantic search |
| created\_at | TIMESTAMP | Record creation date |

## **4.2 Appointment Management**

Description: The user informs future appointments via text. The agent extracts structured information and stores it in the database. Appointment queries are answered via Text-to-SQL.

### **Interaction Examples**

* Registration: "I have an appointment with the dermatologist on 04/25 at 2pm at Vida Clinic"
* "What appointments do I have next week?" → Text-to-SQL
* "Do I have any ophthalmology appointment scheduled?" → Text-to-SQL

### **Data Model**

| Field | Type | Description |
| :---- | :---- | :---- |
| id | UUID | Unique identifier |
| user\_id | UUID | Reference to the user |
| subject\_type | ENUM | 'human' or 'animal' |
| subject\_name | VARCHAR | Patient/animal name |
| appointment\_type | VARCHAR | Appointment/procedure type |
| specialty | VARCHAR | Specialty |
| doctor\_name | VARCHAR | Professional name (optional) |
| location | VARCHAR | Care location |
| appointment\_date | DATE | Appointment date |
| appointment\_time | TIME | Appointment time |
| notes | TEXT | Additional notes |
| status | ENUM | 'scheduled', 'completed', 'cancelled' |
| created\_at | TIMESTAMP | Record creation date |

## **4.3 Medication Management**

Description: The user informs the medications they take, schedules, dosages, and purpose. The agent records them in a structured way and answers questions via Text-to-SQL.

### **Interaction Examples**

* Registration: "I take losartan 50mg in the morning at 8am for high blood pressure"
* "What medications do I take in the morning?" → Text-to-SQL
* "What is the medication I take at night for?" → Text-to-SQL

### **Data Model**

| Field | Type | Description |
| :---- | :---- | :---- |
| id | UUID | Unique identifier |
| user\_id | UUID | Reference to the user |
| subject\_type | ENUM | 'human' or 'animal' |
| subject\_name | VARCHAR | Patient/animal name |
| medication\_name | VARCHAR | Medication name |
| dosage | VARCHAR | Dosage (e.g., 50mg) |
| frequency | VARCHAR | Frequency (e.g., 1x daily) |
| schedule\_times | JSONB | Schedules \[{"time": "08:00", "label": "morning"}\] |
| purpose | TEXT | Purpose / indication |
| prescriber | VARCHAR | Prescribing physician |
| start\_date | DATE | Treatment start date |
| end\_date | DATE | Treatment end date (null \= ongoing) |
| is\_active | BOOLEAN | Whether currently in use |
| notes | TEXT | Notes (e.g., take with food) |
| created\_at | TIMESTAMP | Record creation date |

## **4.4 PDF Generation**

Description: The agent generates formatted PDFs from HTML/CSS templates so users can export, print, or store their information without relying on the agent for queries.

### **Planned Templates**

| Template | Content | Period |
| :---- | :---- | :---- |
| Weekly Medication Calendar | Table with days of week x times, listing medications in each slot | Weekly |
| Appointment Calendar (Weekly) | List of scheduled appointments for the week with date, time, location, and doctor | Weekly |
| Appointment Calendar (Monthly) | Monthly view in grid format with appointments marked | Monthly |
| Health Summary | Consolidation of active medications, upcoming appointments, and recent records | On demand |

PDFs will follow clean, professional visual templates, optimized for A4 printing. Generation will be done with WeasyPrint, using Jinja2 \+ CSS templates for maximum design flexibility.

## **4.5 Messaging Service Integration**

| Service | Cost | Advantages | Disadvantages |
| :---- | :---- | :---- | :---- |
| Telegram Bot API | Free | No cost, robust API, bot command support, PDF file sending | Lower adoption outside Brazil |
| WhatsApp Business API (via Twilio/360dialog) | \~US$ 0.005–0.08/msg | High adoption in Brazil, familiar UX | Per-message cost, template approval |
| WhatsApp via Baileys/Evolution API | Free (self-hosted) | No message cost | Risk of ban, unofficial |
| Signal Bot | Free | Privacy, no cost | Low adoption |
| SMS (Twilio) | \~US$ 0.007/msg | Universal, works without internet | Cost, no rich media |

**Decision:** WhatsApp via Twilio as the main validation channel (high adoption in Brazil). Telegram Bot API as secondary/free channel for development testing. The architecture should abstract the messaging channel through a common interface (MessagingProvider) to facilitate adding new channels.

### **Webhook and Domain**

**Do I need my own domain to integrate with WhatsApp/Telegram?** No. What is needed is a public URL to receive webhooks. Railway automatically provides a URL in the format your-app.up.railway.app that works as a webhook endpoint for both Twilio and Telegram. A custom domain is optional and purely cosmetic at this stage.

Twilio Sandbox: For the testing phase, Twilio offers a free WhatsApp Sandbox that allows testing without needing a verified number. Just configure the Railway webhook URL in the Twilio panel.

# **5\. Future Features (Roadmap)**

## **5.1 Prescription and Exam OCR**

Users will be able to send photos of medical prescriptions and exam results. The agent will perform OCR (Optical Character Recognition) to extract the text, store it, and allow future queries.

### **Proposed Flow**

* User sends photo of prescription/exam via chat
* Agent uses vision model (e.g., Gemini Flash, LLaMA Vision) to extract text
* Extracted text is structured (medications, dosages, exam results)
* Data is stored in the database (structured \+ embedding for semantic search)
* Original image is stored in object storage (e.g., MinIO, S3)

## **5.2 Active Notifications and Reminders**

The agent will proactively send reminders to the user about medication schedules and upcoming appointments.

### **Technical Requirements**

* Scheduler: Celery Beat or APScheduler for periodic task scheduling
* Message queue: Redis as broker for Celery
* Logic: periodic job queries upcoming appointments and medication schedules
* Notification sent via API of the integrated messaging channel
* Lead time configuration (e.g., 1h before appointment, 15min before medication)
* Configurable sending window (e.g., do not send between 10pm and 7am)

## **5.3 Web Dashboard (Distant Future)**

Complementary web interface for health data visualization, medication adherence charts, and complete appointment history. Would be a separate frontend consuming the same agent API.

# **6\. Database Overview**

The PostgreSQL database will contain the following main tables:

| Table | Description | Search Type |
| :---- | :---- | :---- |
| users | User registry with basic data | Direct SQL |
| subjects | People/animals linked to the user | Direct SQL |
| consultations | Textual consultation records \+ embeddings | pgvector (semantic) |
| appointments | Future and past appointments | Text-to-SQL |
| medications | Active medications and history | Text-to-SQL |
| documents | Prescription/exam photos (future) | pgvector \+ SQL |
| reminders | Reminder settings (future) | Direct SQL |

The pgvector extension will be enabled in PostgreSQL for support to VECTOR type columns and similarity search indexes (IVFFlat or HNSW). Embeddings will be generated using a lightweight model, preferably local or low-cost via OpenRouter.

# **7\. Security and Privacy**

As the system handles sensitive health data, it must follow rigorous security practices:

* Data at rest encrypted (PostgreSQL TDE or application-level encryption)
* Communication always via HTTPS/TLS
* User authentication linked to messaging service identifier
* Data segregation by user\_id in all queries (Row-Level Security)
* Access logs and audit trail
* Sensitive variables managed via .env (never committed)
* Rate limiting for abuse prevention
* GDPR/LGPD compliance: consent, right to erasure, portability

# **8\. Deployment Strategy**

## **8.1 Development Environment**

* Docker Compose with services: app, postgres (with pgvector), redis (future)
* Hot-reload with uvicorn for fast development
* LangSmith connected for tracing in development

## **8.2 Production / Validation Environment**

* Backend: Railway — deploy of Python agent (API \+ webhook \+ worker)
* Frontend (future): Vercel — web dashboard when needed
* Database: Supabase (PostgreSQL \+ pgvector free on free tier) or Railway PostgreSQL addon
* CI/CD via GitHub Actions (lint, test, build, automatic deployment)
* No custom domain in testing phase — Railway and Vercel URLs are sufficient

## **8.3 Infrastructure as Code (IaC)**

**The chosen stack does not require IaC in the current phase.** Railway and Vercel are PaaS that abstract the infrastructure — configuration is done via railway.toml and vercel.json directly in the repository. Docker Compose is used only for local development. Terraform/Pulumi/CDK would make sense only in a future migration to AWS/GCP with multiple managed services.


# **9\. Operational Cost Estimate (MVP)**

| Item | Estimated Cost/month | Notes |
| :---- | :---- | :---- |
| Railway (backend) | US$ 0–5 | Hobby plan US$ 5/month with US$ 5 credit included |
| PostgreSQL DB (Supabase free) | US$ 0 | 500MB \+ pgvector, sufficient for initial phase |
| OpenRouter (models) | US$ 0–5 | Free models for intent \+ cheap models for SQL |
| Twilio WhatsApp (validation) | US$ 0–10 | Free sandbox for testing; production \~US$ 0.005/msg |
| Telegram Bot API (dev) | US$ 0 | Completely free, used for development |
| LangSmith (free tier) | US$ 0 | 5,000 traces/month |
| Vercel (future frontend) | US$ 0 | Generous free tier, only when dashboard is needed |
| Domain | US$ 0 (MVP phase) | Not required — Railway/Vercel URLs are sufficient |
| Total estimated | US$ 5–20/month | Testing and validation phase with real users |

# **10\. Next Steps**

* Set up repository with uv, ruff, pytest, Makefile, and Docker Compose
* Implement database models (SQLAlchemy \+ Alembic) and enable pgvector
* Create basic LangGraph graph with intent classification node
* Implement consultation registration with embeddings
* Implement CRUD for medications and appointments with Text-to-SQL
* Integrate with Telegram Bot API (free channel for development)
* Integrate with Twilio WhatsApp Sandbox for validation
* Implement PDF generation with templates
* Deploy on Railway \+ Supabase
* Add tests (pytest) and CI/CD via GitHub Actions
* Validate with real users and iterate
* Implement future features (OCR, notifications, Vercel dashboard)
