# **🚀 AI-Powered Resume Parser and Job Matcher**

## **Project Overview**

### This project delivers a robust, enterprise-grade solution for resume processing and candidate matching. We have implemented a highly scalable architecture to overcome the traditional bottlenecks of slow processing time, format variability, and the lack of semantic context in older HR systems.

### The core achievement is the integration of a Decoupled Asynchronous Processing Pipeline using modern AI/ML techniques (Hugging Face Transformers) to perform complex text analysis while keeping the user-facing API instantaneous.

## **🏗️ Technical Architecture: Decoupled for Scale**

### The solution runs as a Microservice Architecture orchestrated by Docker Compose, ensuring environmental consistency and horizontal scalability. This is key to achieving high throughput.

### The Four Core Services ->

### 1) API Gateway (FastAPI):

### This is the user interface layer. It handles all incoming HTTP requests, performs initial file validation, and manages the connection to the database.

### Crucially, it returns a 202 Accepted response almost immediately, outsourcing the heavy lifting to the worker.

### 2) Worker Engine (Celery):

### This is the compute-intensive powerhouse. It operates completely asynchronously in the background.

### It executes complex tasks like multi-format text extraction (including OCR) and runs the PyTorch/Transformer models for data structuring and matching.

### 3) Database (PostgreSQL):

### Provides persistent storage for all resume metadata and the resulting structured JSON data. We utilize the native JSONB data type for flexibility, ensuring the schema can easily adapt to evolving AI output without costly database migrations.

### 4) Broker (Redis):

### Acts as the high-speed message queue and result backend, bridging communication between the API (producer) and the Worker (consumer). This ensures no job is lost and the system can handle sudden spikes in resume uploads.

## **⚙️ Setup and Installation**

### Prerequisites

### You must have Docker Desktop (or Docker Engine) and Docker Compose installed and running on your system.

### One-Command Setup ->

### Navigate to the project root directory where setup.sh is located and execute the single setup script.

### chmod +x setup.sh
### ./setup.sh


## **🚀 Live Demo Flow and Core Endpoints**
### The API documentation is compliant with OpenAPI 3.1.1 and available via the interactive Swagger UI at http://localhost:8000/docs.

### Demo Flow: The 3-Step Candidate Assessment
### Step 1: Upload and Queue
### Endpoint: POST /resumes/upload

### Action: Upload a resume file (PDF, DOCX, or Image).

<img width="1284" height="945" alt="Screenshot 2025-11-05 122733" src="https://github.com/user-attachments/assets/269ee6b0-b19a-4c25-978a-d1f901578812" />


### Result: The API instantly returns a 202 Accepted response with a unique Resume ID.

### Step 2: Parse and Retrieve Data
### Function: The Celery Worker extracts raw text (including running Tesseract OCR if needed), feeds it into the Hugging Face NER pipeline, and saves the structured JSON to PostgreSQL.

### Endpoint: GET /resumes/{id}

### Action: Check the database using the unique ID.

<img width="1277" height="853" alt="Screenshot 2025-11-05 122816" src="https://github.com/user-attachments/assets/0f53454a-95dd-40a0-98a3-13a10ab4a6bf" />


### Result: Retrieves the full candidate profile, including skills, experience, and contact details, demonstrating high-accuracy data extraction.

### Step 3: Advanced Job Matching (High-Value Feature)
### Endpoint: POST /resumes/{id}/match

### Action: Submit a Job Description JSON to this endpoint, using the Resume ID obtained in Step 1.

<img width="1274" height="745" alt="Screenshot 2025-11-05 122902" src="https://github.com/user-attachments/assets/14649054-2d3c-42f6-902d-86a7472e415f" />


### Logic: The system runs the Sentence Transformer model (all-MiniLM-L6-v2) to calculate the semantic closeness between the resume and the job description.

### Result: Returns the final Overall Score and the Gap Analysis, demonstrating intelligence beyond simple keyword matching.

<img width="1276" height="895" alt="Screenshot 2025-11-05 122918" src="https://github.com/user-attachments/assets/88f9afc9-96bd-45ef-85fb-6702892f21b4" />


## **🧠 AI/ML Implementation Deep Dive**
### The intelligence of the application is handled by two distinct models:

### Data Extraction (Core NER):

### We use a pre-trained Hugging Face NER model (dslim/bert-base-NER) to identify entities like names, roles, and dates, achieving high baseline accuracy.

### This is combined with robust multi-format processing logic to handle diverse layouts.

### Contextual Matching (Advanced Feature):

### This is the innovative core. We utilize Sentence Embedding via the all-MiniLM-L6-v2 model.

### This model converts all text into numerical vectors (embeddings), allowing the system to understand that 'DevOps practices' and 'Docker/Kubernetes' are conceptually related, leading to highly accurate, contextual match scores.

### The matching logic is wrapped in defensive coding to prevent a single model failure from crashing the entire web API (ensuring the submission criteria for proper error handling are met).

### Deployment: Fully containerized and deployed using docker-compose.yml and the working ./setup.sh.

### presentation slides -> https://drive.google.com/file/d/1AASKFCT2fu8jQY_ObgsgvLqRrbiIHVvS/view?usp=drive_link
