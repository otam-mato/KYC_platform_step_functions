# KYC_platform_step_functions

![PostgreSQL](https://img.shields.io/badge/db-PostgreSQL-blue)
![AWS Lambda](https://img.shields.io/badge/runtime-AWS_Lambda-orange)
![Textract](https://img.shields.io/badge/AWS-Textract-purple)
![Rekognition](https://img.shields.io/badge/AWS-Rekognition-purple)
![SQS](https://img.shields.io/badge/messaging-SQS-green)
![Apify](https://img.shields.io/badge/external-Apify-black)



End‑to‑end Know‑Your‑Customer (KYC) flow built on **AWS Lambda, SQS, Textract, Rekognition, RDS (PostgreSQL)** and **Apify**.  This repository contains:

* Live architecture and data‑flow diagrams
* A step‑by‑step processing table with example payloads
* Patch SQL to bring the schema up

---

## 📁 Quick Links

| Artifact                          | File                                                 |
| --------------------------------- | ---------------------------------------------------- |
| AWS CloudFormation template | [`infrastructure.yml`](./infrastructure.yml)             |
| Schema patch SQL                  | [`Kyc_Schema_Patch.sql`](./Kyc_Schema_Patch.sql) |

---

## 1  High‑Level Component Diagram

```mermaid
%%  KYC platform – component diagram
graph TD

    %% ─────────  EXTERNAL ACTORS  ─────────
    subgraph "External Actors"
        CSUI["CS Review UI"]
        User["User / Browser"]
    end

    %% ─────────  AWS KYC PLATFORM  ───────
    subgraph "KYC Platform (AWS account)"
        direction TB

        S3[(S3 Bucket<br/>kyc-raw/<br/>SSE-KMS + ObjectLock + VPC Endpoint)]:::store

        KycJobsQ[[kyc-jobs SQS<br/>+ DLQ + Alarm]]:::queue

        DocScan[[doc_scan_lambda<br/>+ X-Ray + Logs]]:::lambda
        ReqMaker[[reg_request_lambda<br/>+ X-Ray + Logs]]:::lambda
        FaceMatch[[face_match_lambda<br/>+ X-Ray + Logs]]:::lambda
        RegCheck[[reg_check_lambda<br/>+ X-Ray + Logs]]:::lambda
        Decision[[decision_lambda<br/>+ X-Ray + Logs]]:::lambda
        Expiry[[expiry_reminder_lambda<br/>+ X-Ray + Logs]]:::lambda

        StepFn["KYC State&nbsp;Machine<br/>(Step&nbsp;Functions)"]:::stepfn

        DB[(RDS PostgreSQL<br/>w/ Proxy & Read Replica)]:::store
        AuditLog[(Audit Trail DB<br/>Structured Logs to OpenSearch)]:::store
    end

    %% ─────────  SQL02 ON-PREM / EDGE  ─────────
    subgraph "SQL02 On-prem"
        OnPremSvc["onprem_upload_service"]:::lambda
        DataSyncAgent["AWS DataSync Agent"]:::source
    end

    %% ─────────  REGISTERS / APIFY  ───────
    subgraph APIFY_BOX["Registers / Apify scrapers"]
        direction TB
        ApifyGDC["GDC Register Scraper"]
        ApifyNMC["NMC Register Scraper"]
        ApifyGMC["GMC Register Scraper"]
        ApifyGPC["GPC Register Scraper"]
        RegCache["Register Cache (DynamoDB + TTL)"]:::store
    end

    %% ─────────  AWS AI SERVICES  ─────────
    subgraph "AWS AI services"
        direction TB
        Textract[(Amazon Textract<br/>via VPC Endpoint)]:::ai
        Rekog[(Amazon Rekognition<br/>via VPC Endpoint)]:::ai
        Textract -. "layout (no data)" .-> Rekog  
    end

    %% ─────────  NOTIFICATION  ───────────
    subgraph "Notifications"
        direction TB
        NotifySNS["SNS / Webhook"]
        NotifyEmail["Email to CS"]
        NotifySNS -. "layout (no data)" .-> NotifyEmail
    end

    %% ─────────  FEEDBACK LOOP  ─────────
    subgraph "Manual Review Feedback"
        FeedbackSvc["Review Outcome Capture Lambda"]
        FeedbackDB["Feedback Store (DynamoDB)"]:::store
    end

    %% ─────────  FLOWS ─────────
    User -->|"PUT ID & selfie"| OnPremSvc
    OnPremSvc -- "copy files" --> DataSyncAgent
    DataSyncAgent -->|"PUT objects"| S3

    S3 -- "ObjectCreated" --> DocScan
    DocScan --> Textract
    Textract --> DocScan
    DocScan --> DB
    DocScan --> KycJobsQ
    KycJobsQ --> StepFn

    StepFn --> FaceMatch
    FaceMatch --> Rekog
    Rekog --> FaceMatch
    FaceMatch --> DB
    FaceMatch --> StepFn

    StepFn --> ReqMaker
    ReqMaker -- "type = GDC" --> ApifyGDC
    ReqMaker -- "type = NMC" --> ApifyNMC
    ReqMaker -- "type = GMC" --> ApifyGMC
    ReqMaker -- "type = GPC" --> ApifyGPC

    %% each scraper writes downward into the cache
    ApifyGDC --> RegCache
    ApifyNMC --> RegCache
    ApifyGMC --> RegCache
    ApifyGPC --> RegCache
    RegCache --> ReqMaker
    ReqMaker --> StepFn

    StepFn --> RegCheck
    RegCheck --> DB
    RegCheck --> StepFn

    StepFn --> Decision
    Decision --> DB
    Decision --> NotifySNS
    Decision --> NotifyEmail
    Decision --> CSUI
    Decision --> AuditLog

    CSUI --> FeedbackSvc
    FeedbackSvc --> FeedbackDB

    Expiry --> DB
    Expiry --> NotifyEmail
    Expiry --> User

    %% ─────────  STYLING  ─────────
    classDef lambda fill:#004B76,stroke:#fff,color:#fff;
    classDef queue  fill:#C0D4E4,stroke:#004B76,color:#000;
    classDef store  fill:#F8F8F8,stroke:#555,color:#000;
    classDef source fill:#FFF4CE,stroke:#C09,color:#000;
    classDef ai     fill:#7A5FD0,stroke:#fff,color:#fff;
    classDef stepfn fill:#0D5C63,stroke:#fff,color:#fff,font-weight:bold;

    class OnPremSvc,DocScan,ReqMaker,FaceMatch,RegCheck,Decision,Expiry,FeedbackSvc lambda;
    class KycJobsQ queue;
    class S3,DB,AuditLog,FeedbackDB,RegCache store;
    class DataSyncAgent source;
    class Textract,Rekog ai;
    class StepFn stepfn;
```

---

## 2  Detailed Flow Diagram

```mermaid
%%  KYC flow – full round-trip (Step Functions, feedback path added ✓)
graph TD
    %% ─────────── Sources & Triggers ───────────
    USER_UPLOAD[User uploads<br/>ID + selfie]
    FOLDER[(On-prem images<br/>folder)]
    DATASYNC[AWS DataSync<br/>agent]
    S3[(S3 bucket<br/>kyc-raw)]
    APIFY[Apify<br/>register scrapers]
    CRON[Weekly<br/>expiry scheduler]
    CSUI[CS Review UI]

    %% ─────────── AI services ───────────
    Textract[(Amazon Textract)]
    Rekog[(Amazon Rekognition)]

    %% ─────────── Lambda workers / StepFn tasks ───────────
    DOC[doc_scan_lambda]
    FACE[face_match_lambda]
    REQ[reg_request_lambda]
    REG[reg_check_lambda]
    DEC[decision_lambda]
    REM[expiry_reminder_lambda]
    FEED[review_outcome_capture_lambda]

    %% ─────────── Queue & Workflow ───────────
    JOBS[[kyc-jobs SQS<br/>DLQ + alarm]]:::queue
    STEP["KYC State&nbsp;Machine<br/>(Step Functions)"]:::stepfn

    %% ─────────── Notifications ───────────
    CSMAIL[Email to CS]
    NOTIFY_SNS[SNS / Webhook]

    %% ─────────── Relational DB (tables) ───────────
    subgraph DB[Relational DB]
        USERS[(users)]
        IDDOC[(id_documents)]
        SCANS[(doc_scans)]
        SELFIES[(selfies)]
        FACES[(face_checks)]
        REGCHK[(reg_checks)]
        KYC[(kyc_decisions)]
    end

    %% ─────────── Feedback DB ───────────
    FEEDDB[(Feedback Store<br/>DynamoDB)]:::store

    %% ───── 0  Upload → folder → S3
    USER_UPLOAD -- "write images" --> FOLDER
    FOLDER -->|DataSync job| DATASYNC
    DATASYNC -->|PUT objects| S3
    S3 -- ObjectCreated --> DOC

    %% ───── 1  doc_scan_lambda
    DOC -. "Textract OCR" .-> Textract
    Textract -. JSON .-> DOC
    DOC -- "INSERT doc_scans" --> SCANS
    DOC -- "INSERT selfies"   --> SELFIES
    DOC -- "UPDATE id_documents<br/>(status='OCR_DONE')" --> IDDOC
    DOC -->|msg user_id=12345| JOBS

    %% ───── 2  Step Functions orchestration
    JOBS --> STEP

    STEP --> FACE
    FACE -. "compare + liveness" .-> Rekog
    Rekog -. JSON .-> FACE
    FACE -- "INSERT face_checks" --> FACES
    FACE --> STEP

    STEP --> REQ
    REQ -- "HTTP type=GDC" --> APIFY
    APIFY -- JSON --> REG
    REG -- "INSERT reg_checks" --> REGCHK
    REG --> STEP

    STEP --> DEC
    DEC -- "INSERT kyc_decisions" --> KYC
    DEC -- "UPDATE users" --> USERS
    DEC --|PASS|--> NOTIFY_SNS
    DEC --|MANUAL&nbsp;REVIEW|--> CSMAIL
    DEC --|WebSocket|--> CSUI
    DEC --> STEP    

    %% ───── 5  Manual review feedback
    CSUI -- "approve / reject" --> FEED
    FEED -- "INSERT outcome" --> FEEDDB
    FEED -- "UPDATE kyc_decisions" --> KYC

    %% ───── 6  Expiry reminders
    CRON --> REM
    REM -- "expiry <90/30/7d → SNS" --> NOTIFY_SNS

    %% ─────────── Styling ───────────
    classDef lambda fill:#004B76,stroke:#fff,color:#fff;
    classDef queue  fill:#C0D4E4,stroke:#004B76,color:#000;
    classDef stepfn fill:#0D5C63,stroke:#fff,color:#fff,font-weight:bold;
    classDef ai     fill:#7A5FD0,stroke:#fff,color:#fff;
    classDef source fill:#FFF4CE,stroke:#C09,color:#000;
    classDef store  fill:#F8F8F8,stroke:#555,color:#000;

    class DOC,FACE,REQ,REG,DEC,REM,FEED lambda;
    class JOBS queue;
    class Textract,Rekog ai;
    class USER_UPLOAD,FOLDER,DATASYNC,APIFY,CRON,CSUI,CSMAIL,NOTIFY_SNS source;
    class USERS,IDDOC,SCANS,SELFIES,FACES,REGCHK,KYC,FEEDDB store;
    class STEP stepfn;
```

---

## 3  Process Step Descriptions (Enum-Aligned)

<details>
<summary>Click to expand step‑by‑step table</summary>

| Step     | Trigger / Source                        | Service (State-Machine Task)                                 | Action                                                         | DB Writes                                                                             | Example Columns / Notes                                                                     | Next                                    |     |       |                   |
| -------- | --------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------- | --- | ----- | ----------------- |
| **0**    | User uploads ID & selfie → on-prem → S3 | `onprem_upload_service`                                      | Accept files, create user/doc rows                             | `users` INSERT<br>`id_documents` INSERT (`status='NEW'`)                              | `email=dr.jane@example.com`<br>`reg_no=6143219`<br>`doc_type=passport`                      | —                                       |     |       |                   |
| **1**    | **`S3:ObjectCreated`**                  | `doc_scan_lambda`                                            | Run Textract OCR, parse metadata                               | `doc_scans` INSERT<br>`selfies` INSERT<br>`id_documents` UPDATE (`status='OCR_DONE'`) | `parsed_name=JANE ANN DOE`<br>`parsed_expiry=2032-05-01`                                    | → **put message** on **`kyc-jobs` SQS** |     |       |                   |
| **2**    | **`kyc-jobs` SQS**                      | **KYC State Machine (Step Functions)**<br>*Execution starts* | Orchestrate KYC checks with retries, time-outs & audit history | —                                                                                     | Input contains `user_id`, parsed doc data                                                   | **2a** & **2b** run in parallel         |     |       |                   |
| **2a**   | Step Functions task                     | `face_match_lambda`                                          | Rekognition face-match & liveness                              | `face_checks` INSERT                                                                  | `match_score=0.93`<br>`liveness_pass=true`                                                  | Wait for **2b**                         |     |       |                   |
| **2b-1** | Step Functions task                     | `reg_request_lambda`                                         | Choose scraper based on `type`, invoke Apify (async)           | —                                                                                     | \`type=GDC                                                                                  | NMC                                     | GMC | GPC\` | Wait for callback |
| **2b-2** | Apify HTTP callback → Step Functions    | `reg_check_lambda`                                           | Store register result                                          | `reg_checks` INSERT                                                                   | `matched_name=true`<br>`matched_status=true`                                                | Wait for **2a**                         |     |       |                   |
| **2c**   | Step Functions task                     | `decision_lambda`                                            | Aggregate all signals, decide PASS / MANUAL / FAIL             | `kyc_decisions` INSERT<br>`users` UPDATE                                              | `decision=PASS`<br>`status=VERIFIED`                                                        | SNS → 3 channels                        |     |       |                   |
| **2d**   | Step Functions task                     | (integrated)                                                 | Publish result                                                 | —                                                                                     | Sends to:<br>• **SNS / Webhook**<br>• **Email to CS**<br>• **CS Review UI** (WebSocket/SSE) | —                                       |     |       |                   |
| **3**    | Manual override in CS UI                | —                                                            | CS agent approves / rejects                                    | `kyc_decisions` UPDATE<br>`users` UPDATE                                              | —                                                                                           | —                                       |     |       |                   |
| **4**    | Weekly CloudWatch rule                  | `expiry_reminder_lambda`                                     | Email users whose IDs expire in 90 / 30 / 7 days               | *(read-only)*                                                                         | —                                                                                           | Notify topic                            |     |       |                   |

</details>

---

## 4  Entity‑Relationship Diagram

```mermaid
erDiagram
    %% direction LR   %% ← Uncomment if your renderer supports L-to-R layout

    users {
        BIGSERIAL        id PK
        VARCHAR(254)     email
        VARCHAR(50)      reg_no
        reg_type_enum    reg_type
        user_status_enum status
        TIMESTAMPTZ      created_at
    }

    id_documents {
        BIGSERIAL         id PK
        BIGINT            user_id FK
        TEXT              s3_key_original
        VARCHAR(40)       doc_type
        id_doc_status_enum status
        DATE              expiry_date
        TIMESTAMPTZ       created_at
    }

    selfies {
        BIGSERIAL      id PK
        BIGINT         user_id FK
        TEXT           s3_key
        TIMESTAMPTZ    created_at
    }

    doc_scans {
        BIGSERIAL      id PK
        BIGINT         id_document_id FK
        JSONB          textract_json
        TEXT           parsed_name
        DATE           parsed_dob
        DATE           parsed_expiry
        VARCHAR(40)    parser_version
        TIMESTAMPTZ    completed_at
    }

    face_checks {
        BIGSERIAL      id PK
        BIGINT         user_id FK
        BIGINT         selfie_id FK
        BIGINT         id_document_id FK
        NUMERIC        match_score
        BOOLEAN        liveness_pass
        VARCHAR(100)   rekognition_job_id
        TIMESTAMPTZ    completed_at
    }

    reg_checks {
        BIGSERIAL      id PK
        BIGINT         user_id FK
        DATE           snapshot_date
        BOOLEAN        matched_name
        BOOLEAN        matched_status
        JSONB          raw_response_json
        TIMESTAMPTZ    checked_at
    }

    kyc_decisions {
        BIGSERIAL        id PK
        BIGINT           user_id FK
        kyc_decision_enum decision
        TEXT[]           reasons
        TIMESTAMPTZ      decided_at
    }

    %% ─────── Relationships (each FK exactly once) ───────
    users        ||--o{ id_documents  : "users.id → id_documents.user_id"
    users        ||--o{ selfies       : "users.id → selfies.user_id"
    users        ||--o{ face_checks   : "users.id → face_checks.user_id"
    users        ||--o{ reg_checks    : "users.id → reg_checks.user_id"
    users        ||--o{ kyc_decisions : "users.id → kyc_decisions.user_id"

    id_documents ||--o{ doc_scans     : "id_documents.id → doc_scans.id_document_id"
    id_documents ||--o{ face_checks   : "id_documents.id → face_checks.id_document_id"

    selfies      ||--o{ face_checks   : "selfies.id → face_checks.selfie_id"
```

---

### Running Locally

```bash
psql $DB_URL -f "Kyc_Schema_Patch.sql"
```

---

### Contributing

PRs welcome – please update diagrams + docs if queue names, enum values, or DB tables change.
