# SƠ ĐỒ FLOW & KIẾN TRÚC — SinhDeDoAI

> **Version:** 1.1 | **Cập nhật:** 2026-05-21
> Thay đổi v1.1: 3 role (admin/mod/reviewer), bỏ public signup, thêm audit logging xuyên suốt.

## 1. Luồng nghiệp vụ tổng thể (Business Flow)

```mermaid
flowchart TD
    A0[Admin tạo tài khoản<br/>mod & reviewer] --> A1[User đăng nhập<br/>đổi password lần đầu]
    A1 --> A[Mod/Admin upload<br/>Question Bank thô]
    A --> B[(Storage: raw-uploads)]
    B --> C[Parser trích xuất câu hỏi]
    C --> D[(DB: questions<br/>status=pending_review<br/>bank type=raw)]
    D --> E[Admin/Mod phân quyền<br/>reviewer duyệt]
    E --> F[Reviewer duyệt L1:<br/>sửa nội dung, đáp án,<br/>điều chỉnh độ khó]
    F --> G{Status: reviewed}
    G --> H[Admin/Mod phê duyệt L2]
    H -->|Approve| I[(DB: questions<br/>status=approved<br/>bank type=refined)]
    H -->|Reject| F
    I --> J[Admin tạo/chọn<br/>Exam Structure]
    J --> K[Admin chọn Template Word]
    K --> L[Sinh đề ngẫu nhiên<br/>theo cấu trúc]
    L --> M[Render docx<br/>via docxtemplater]
    M --> N[Convert PDF<br/>via LibreOffice]
    N --> O[(Storage: exams/)]
    L --> P[Sinh HDC<br/>Hướng dẫn chấm]
    P --> M
    O --> Q[Admin/Mod tải xuống<br/>.docx + .pdf]

    %% Audit log overlay
    A0 -.log.- AL[(audit_logs)]
    A1 -.log.- AL
    A -.log.- AL
    E -.log.- AL
    F -.log.- AL
    H -.log.- AL
    L -.log.- AL
    Q -.log.- AL

    style A0 fill:#ffe1f0
    style A fill:#e1f5ff
    style F fill:#fff4e1
    style H fill:#ffe1e1
    style L fill:#e8ffe1
    style Q fill:#f0e1ff
    style AL fill:#fff9c4
```

---

## 2. Kiến trúc hệ thống (System Architecture)

```mermaid
flowchart LR
    subgraph Client["🌐 Client - Browser"]
        UI_Admin[Admin UI]
        UI_Mod[Mod UI]
        UI_Rev[Reviewer UI]
    end

    subgraph Vercel["▲ Vercel - Next.js 14"]
        Login[Login page<br/>NO SIGNUP]
        SSR[Server Components<br/>+ Server Actions]
        API[API Routes]
        Audit[Audit middleware]
        Middleware[Auth + role check]
    end

    subgraph Supabase["🟢 Supabase"]
        Auth[Auth<br/>signup DISABLED<br/>admin creates users]
        DB[(PostgreSQL<br/>+ RLS<br/>+ Audit triggers)]
        AuditDB[(audit_logs<br/>append-only)]
        Storage[Storage<br/>raw / templates / exams]
        Realtime[Realtime]
    end

    subgraph Worker["🐳 Doc Worker - Railway/Fly.io"]
        Docx[docxtemplater]
        LibreOffice[LibreOffice<br/>headless]
    end

    subgraph External["🤖 External APIs - optional"]
        LLM[Claude / OpenAI<br/>parse free-form .docx]
        Email[Resend<br/>welcome + reset]
    end

    UI_Admin --> Login
    UI_Mod --> Login
    UI_Rev --> Login
    Login --> Auth

    UI_Admin --> SSR
    UI_Mod --> SSR
    UI_Rev --> SSR

    SSR --> Middleware
    Middleware --> Audit
    Audit --> AuditDB
    SSR --> DB
    DB -.trigger.- AuditDB
    API --> DB
    API --> Storage
    SSR -.subscribe.- Realtime

    API -->|generate| Worker
    Worker --> Docx
    Docx --> LibreOffice
    LibreOffice --> Storage

    API -.fallback parse.- LLM
    API -.send mail.- Email

    style Client fill:#e8f4ff
    style Vercel fill:#f0f0f0
    style Supabase fill:#e8ffe8
    style Worker fill:#fff4e8
    style AuditDB fill:#fff9c4
```

---

## 3. ER Diagram (rút gọn, v1.1)

```mermaid
erDiagram
    profiles ||--o{ reviewer_assignments : "assigned to"
    profiles ||--o{ audit_logs : "performs action"
    profiles ||--o| profiles : "created by"
    subjects ||--o{ chapters : "has"
    subjects ||--o{ question_banks : "contains"
    subjects ||--o{ exam_structures : "for"
    question_banks ||--o{ questions : "contains"
    chapters ||--o{ questions : "categorize"
    reviewer_assignments }o--|| question_banks : "scope"
    reviewer_assignments }o--o| chapters : "scope (optional)"
    exam_structures ||--o{ exams : "generates"
    templates ||--o{ exams : "rendered with"

    profiles {
        uuid id PK
        text email UK
        text full_name
        text role "admin/mod/reviewer"
        bool is_active
        bool must_change_password
        uuid created_by FK
        timestamptz last_login_at
    }
    subjects {
        uuid id PK
        text code UK
        text name
    }
    chapters {
        uuid id PK
        uuid subject_id FK
        text name
        int order_idx
    }
    question_banks {
        uuid id PK
        uuid subject_id FK
        text name
        text type "raw/refined"
    }
    questions {
        uuid id PK
        uuid bank_id FK
        uuid chapter_id FK
        text question_type
        text content
        jsonb options
        jsonb correct_answer
        text explanation
        text difficulty "NB/TH/VD/VDC"
        text status
    }
    reviewer_assignments {
        uuid id PK
        uuid reviewer_id FK
        uuid bank_id FK
        uuid chapter_id FK
    }
    exam_structures {
        uuid id PK
        uuid subject_id FK
        text name
        int total_questions
        jsonb config
    }
    templates {
        uuid id PK
        text file_path
        text template_type "exam/answer_key"
    }
    exams {
        uuid id PK
        text code
        uuid structure_id FK
        uuid template_id FK
        uuid_array question_ids
        text exam_file_path
        text exam_pdf_path
        text answer_key_docx_path
        text answer_key_pdf_path
    }
    audit_logs {
        bigserial id PK
        timestamptz ts
        uuid actor_id FK
        text actor_email
        text actor_role
        text action
        text entity_type
        uuid entity_id
        jsonb before_value
        jsonb after_value
        jsonb diff
        inet request_ip
    }
```

---

## 4. State Machine của một Question

```mermaid
stateDiagram-v2
    [*] --> pending_review: Upload bởi Mod/Admin
    pending_review --> reviewed: Reviewer sửa & đánh dấu duyệt
    reviewed --> approved: Admin/Mod approve (vào bank refined)
    reviewed --> rejected: Admin/Mod từ chối
    rejected --> pending_review: Reviewer chỉnh lại
    approved --> [*]: Sẵn sàng dùng sinh đề

    note right of approved
        Mỗi chuyển trạng thái
        đều ghi audit_logs
    end note
```

---

## 5. Sequence Diagram — Tạo tài khoản (Admin only)

```mermaid
sequenceDiagram
    actor Admin
    participant UI as Next.js /admin/users
    participant API as API /api/admin/users
    participant Auth as Supabase Auth Admin API
    participant DB as Supabase DB
    participant Email as Resend
    actor NewUser as New User

    Admin->>UI: Form: email + name + role (mod/reviewer)
    UI->>API: POST {email, name, role}
    API->>API: Verify caller.role == 'admin'
    API->>Auth: createUser(email, tempPassword)
    Auth-->>API: user.id
    API->>DB: INSERT profiles (id, email, role,<br/>must_change_password=true,<br/>created_by=admin.id)
    DB-->>API: ok
    DB-->>DB: trigger → audit_logs<br/>(user.created)
    API->>Email: send welcome(email, tempPassword)
    Email-->>NewUser: 📧 Email với link login
    API-->>UI: ✅
    UI-->>Admin: User đã tạo

    NewUser->>UI: Login lần đầu
    UI->>UI: must_change_password=true<br/>→ redirect /change-password
    NewUser->>UI: Đổi password mới
    UI->>Auth: updatePassword
    UI->>DB: UPDATE profiles SET<br/>must_change_password=false
    DB-->>DB: trigger → audit_logs<br/>(auth.password_changed)
```

---

## 6. Sequence Diagram — Sinh đề thi (có audit)

```mermaid
sequenceDiagram
    actor Admin
    participant UI as Next.js UI
    participant API as API Route
    participant DB as Supabase DB
    participant W as Doc Worker
    participant S as Supabase Storage
    participant AL as audit_logs

    Admin->>UI: Chọn Structure + Template<br/>+ Số mã đề = 4
    UI->>API: POST /api/generate-exam
    API->>DB: SET app.current_user_id = admin.id
    API->>DB: Validate khả thi
    DB-->>API: OK

    loop Với mỗi mã đề (1..4)
        API->>DB: SELECT câu hỏi theo rules
        DB-->>API: question_ids[]
        API->>W: render-docx
        W->>S: GET template.docx
        S-->>W: file binary
        W->>W: docxtemplater render
        W->>W: LibreOffice convert PDF
        W->>S: PUT exam-001.docx + pdf<br/>+ key-001.docx + pdf
        W-->>API: paths
        API->>DB: INSERT exams row
        DB->>AL: trigger log<br/>(exam.insert)
    end

    API->>AL: app log<br/>(exam.generated,<br/>metadata: count=4, seed)
    API-->>UI: 4 đề đã sinh
    UI-->>Admin: Danh sách + nút Download

    Admin->>UI: Click Download
    UI->>API: GET /api/exams/{id}/download
    API->>S: get signed URL
    API->>AL: log (exam.downloaded)
    API-->>UI: redirect signed URL
```

---

## 7. Phân quyền theo Role (Permission Matrix v1.1)

> **3 role: Admin / Mod / Reviewer. Không có Viewer hay Public.**

| Hành động | Admin | Mod | Reviewer |
|----------|:-----:|:---:|:--------:|
| **Quản lý user** | | | |
| Tạo tài khoản | ✅ | ❌ | ❌ |
| Đổi role / Reset pw | ✅ | ❌ | ❌ |
| Vô hiệu hóa user | ✅ | ❌ | ❌ |
| **Nội dung** | | | |
| Tạo Subject / Chapter | ✅ | ✅ | ❌ |
| Upload bank thô | ✅ | ✅ | ❌ |
| Phân quyền reviewer | ✅ | ✅ | ❌ |
| Duyệt L1 (sửa nội dung) | ✅ | ✅ | ✅ (assigned only) |
| Approve / Reject L2 | ✅ | ✅ | ❌ |
| **Sinh đề** | | | |
| Tạo / sửa Exam Structure | ✅ | ❌ | ❌ |
| Upload Template | ✅ | ✅ | ❌ |
| Sinh đề | ✅ | ✅ | ❌ |
| Tải đề đã sinh | ✅ | ✅ | ❌ |
| **Audit** | | | |
| Xem audit log toàn hệ thống | ✅ | ❌ | ❌ |
| Xem audit log của chính mình | ✅ | ✅ | ✅ |
| Export audit log | ✅ | ❌ | ❌ |

---

## 8. Mô hình Audit Log Flow ⭐ MỚI

```mermaid
flowchart LR
    subgraph App["Application Layer"]
        Action1[User action:<br/>edit question]
        Action2[User action:<br/>login]
        Action3[User action:<br/>upload file]
        Action4[User action:<br/>generate exam]
    end

    subgraph DBLayer["Database Layer"]
        Tables[(questions<br/>profiles<br/>exams<br/>structures<br/>assignments)]
        Trigger{Postgres<br/>AFTER<br/>INSERT/UPDATE/DELETE<br/>trigger}
    end

    subgraph AppLog["App Logger"]
        Logger[lib/audit/logger.ts<br/>logAudit&#40;&#41;]
    end

    AuditTable[(audit_logs<br/>APPEND-ONLY<br/>no UPDATE/DELETE)]

    Action1 --> Tables
    Action3 --> Tables
    Action4 --> Tables
    Tables --> Trigger
    Trigger --> AuditTable

    Action2 --> Logger
    Action3 --> Logger
    Action4 --> Logger
    Logger --> AuditTable

    AuditTable --> UI[/admin/audit-logs<br/>UI viewer/]
    AuditTable --> Export[Export CSV<br/>by Admin]
    AuditTable -.monthly.- Archive[Archive<br/>to Storage<br/>after 12 months]

    style AuditTable fill:#fff9c4
    style Trigger fill:#ffcccc
```

**Tầng 1 (DB Trigger):** Mọi INSERT/UPDATE/DELETE trên bảng quan trọng tự động ghi log → không thể bypass.

**Tầng 2 (App Logger):** Login, upload, generate, download — những action không gắn 1 dòng DB cụ thể.

**Quy tắc bảo vệ:**
- Bảng `audit_logs` chỉ INSERT được, không UPDATE/DELETE (RULE level + RLS)
- Chỉ Admin SELECT được toàn bộ
- Mod/Reviewer chỉ SELECT row có `actor_id = chính mình`

---

## 9. Mô hình dữ liệu Exam Structure (JSONB config)

Ví dụ cấu hình một đề thi THPT QG Toán:

```json
{
  "exam_meta": {
    "subject": "Toán 12",
    "time_min": 90,
    "total_score": 10
  },
  "sections": [
    {
      "name": "Phần I — Trắc nghiệm 4 đáp án",
      "question_type": "MCQ_SINGLE",
      "instruction": "Chọn 1 đáp án đúng cho mỗi câu",
      "rules": [
        { "chapter": "Hàm số", "difficulty": "NB", "count": 4 },
        { "chapter": "Hàm số", "difficulty": "TH", "count": 4 },
        { "chapter": "Mũ - Logarit", "difficulty": "TH", "count": 3 },
        { "chapter": "Nguyên hàm", "difficulty": "VD", "count": 2 },
        { "chapter": "Hình học", "difficulty": "VDC", "count": 1 }
      ],
      "shuffle_options": true
    },
    {
      "name": "Phần II — Đúng/Sai",
      "question_type": "TRUE_FALSE",
      "rules": [
        { "chapter": "Hàm số", "difficulty": "TH", "count": 2 },
        { "chapter": "Hình học", "difficulty": "VD", "count": 2 }
      ]
    },
    {
      "name": "Phần III — Tự luận ngắn",
      "question_type": "SHORT_ANSWER",
      "rules": [
        { "chapter": "Tích phân", "difficulty": "VD", "count": 3 },
        { "chapter": "Hình học", "difficulty": "VDC", "count": 3 }
      ]
    }
  ]
}
```

---

*Sơ đồ v1.1 — Mở các file `.md` bằng VS Code (extension Markdown Preview Mermaid Support) hoặc GitHub để xem render trực tiếp.*    
