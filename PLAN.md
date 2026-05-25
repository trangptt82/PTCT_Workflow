# PLAN: Web App Sinh Đề Thi từ Question Bank (SinhDeDoAI)

> **Version:** 1.1
> **Ngày cập nhật:** 2026-05-21
> **Changelog v1.1:** Bổ sung so sánh tech stack & chi phí | Đổi role từ 4 → 3 (admin/mod/reviewer) | Bỏ tự đăng ký, admin cấp phát tài khoản | Thêm phần Audit Logging toàn hệ thống
> **Tech stack đề xuất:** Next.js 14 (App Router) + Supabase (PostgreSQL + Auth + Storage) + Vercel
> **Mục tiêu MVP:** Hoàn thiện full flow upload → duyệt → tinh chỉnh → sinh đề ngẫu nhiên → xuất Word/PDF, mọi thao tác đều có lưu vết
> **Output:** Đề thi (.docx/.pdf) + Hướng dẫn chấm (HDC) (.docx/.pdf)

---

## 1. TỔNG QUAN HỆ THỐNG

### 1.1. Bài toán
Xây dựng web app quản lý ngân hàng câu hỏi (question bank) và sinh đề thi tự động theo cấu trúc cấu hình được. Hệ thống nội bộ (closed system) — không cho phép tự đăng ký, admin là người tạo và cấp phát tài khoản. Gồm 3 vai trò: **Admin / Mod / Reviewer** (giáo viên duyệt). Toàn bộ thao tác được lưu vết để audit khi cần.

### 1.2. Các đối tượng chính (Domain Entities)
- **Subject** (môn học): Toán, Lý, Hóa, Anh…
- **Question Bank** (kho câu hỏi): chia 2 trạng thái — *Raw* (thô) và *Refined* (tinh chỉnh)
- **Question** (câu hỏi): có loại, độ khó, lời giải, lịch sử chỉnh sửa
- **Exam Structure** (cấu trúc đề): cấu hình số lượng câu × độ khó × phần
- **Exam** (đề thi): output sau khi sinh
- **Template** (mẫu đề Word): file .docx mẫu để merge câu hỏi vào
- **Audit Log** (nhật ký hệ thống): mọi hành động ghi nhận đầy đủ

### 1.3. Loại câu hỏi hỗ trợ
| Loại | Mô tả | Cấu trúc data |
|------|-------|---------------|
| MCQ_SINGLE | Trắc nghiệm 1 đáp án (A/B/C/D) | options[], correct: 1 index |
| MCQ_MULTI | Trắc nghiệm nhiều đáp án | options[], correct: [indexes] |
| TRUE_FALSE | Đúng/Sai (có thể nhiều ý a/b/c/d) | statements[], correct: [bool] |
| SHORT_ANSWER | Tự luận / Điền đáp án ngắn | answer_text, rubric (HDC) |

---

## 2. SO SÁNH & LỰA CHỌN TECH STACK

### 2.1. Bảng so sánh chi tiết

| Tiêu chí | **Next.js + Supabase + Vercel** ⭐ | Streamlit + Postgres | Lovable / Bolt.new | Replit + Flask | Django + Postgres (self-host VPS) |
|----------|-----------------------------------|----------------------|--------------------|----------------|-----------------------------------|
| **Chi phí/tháng (MVP)** | $0-5 | $0-7 | $20-50 | $7-25 | $5-10 (VPS) |
| **Chi phí khi scale (50-500 user)** | $25-50 | $20-40 | $50-100+ | $25-50 | $10-30 |
| **Tốc độ ra MVP** | 4-6 tuần | 2-3 tuần | 3-7 ngày (cho prototype) | 3-4 tuần | 4-6 tuần |
| **UX/UI chất lượng** | ⭐⭐⭐⭐⭐ | ⭐⭐ (đơn giản, hạn chế custom) | ⭐⭐⭐⭐ (auto-generated) | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Phân quyền phức tạp** | Tuyệt vời (RLS) | Khó (phải tự code) | Hạn chế | Trung bình | Tốt (built-in admin) |
| **File upload + parse | Tốt | Tốt | Trung bình | Tốt | Tốt |
| **Generate Word/PDF** | Tốt (cần worker riêng cho PDF) | Tuyệt vời (python-docx, weasyprint) | Hạn chế (export khó) | Tuyệt vời (Python ecosystem) | Tuyệt vời |
| **Audit log + Triggers DB** | Tuyệt vời (Postgres triggers) | Tuyệt vời (Postgres triggers) | Hạn chế | Tốt | Tuyệt vời |
| **Khả năng scale** | Tuyệt vời | Trung bình (single process) | Phụ thuộc nền tảng | Trung bình | Tuyệt vời |
| **Vendor lock-in** | Thấp (chuyển self-host được) | Rất thấp | **Cao** | Trung bình | Không |
| **Bảo trì/Maintenance** | Trung bình | Thấp | Phụ thuộc Lovable | Thấp | Cao (tự lo VPS) |
| **Khả năng customize** | Cao | Trung bình | Thấp (no-code limit) | Cao | Cao |

⭐ = lựa chọn được đề xuất

### 2.2. Phân tích từng option

#### Option A: Next.js + Supabase + Vercel ⭐ (Đề xuất)
**Ưu:**
- Free tier rất tốt: Supabase 500MB DB, Vercel hobby free → MVP ~$0
- RLS (Row Level Security) ở DB-level giải quyết phân quyền 3 role gọn gàng
- Postgres triggers cực mạnh cho audit logging tự động
- UI/UX chuyên nghiệp với shadcn/ui
- Scale lên 500+ users mà chi phí vẫn dưới $50

**Nhược:**
- PDF generation cần worker riêng (LibreOffice không chạy native trên Vercel) → thêm $5 trên Railway
- Cần biết React/TypeScript

**Kết luận:** Tối ưu nhất cho yêu cầu — chi phí cực thấp + đầy đủ tính năng + audit trail mạnh.

#### Option B: Streamlit + PostgreSQL (Phương án rẻ thay thế)
**Ưu:**
- **Rẻ nhất** nếu chạy trên Streamlit Community Cloud (free) hoặc Hugging Face Spaces (free)
- Python ecosystem mạnh cho generate Word/PDF (`python-docx`, `docxtpl`, `weasyprint`)
- Code nhanh nếu team quen Python
- Có thể tích hợp `supabase-py` để vẫn dùng Supabase free tier

**Nhược:**
- UI/UX **hạn chế** — Streamlit là dashboard framework, không phải app full-fledged
- Phân quyền 3 role phải tự code middleware (Streamlit không có built-in auth)
- Khó làm rich text editor cho câu hỏi (chỉ có textarea cơ bản)
- Single-process → chậm khi 20+ user đồng thời
- Trang admin/reviewer dùng chung 1 layout → UX không tách bạch

**Khi nào nên chọn:** Team chỉ có Python skill + chấp nhận UX cơ bản + chỉ <20 user dùng đồng thời + cần ra prototype trong 2 tuần.

**Chi phí:** $0 (Streamlit Cloud) + $0 (Supabase free) = **$0/tháng** cho MVP.

#### Option C: Lovable / Bolt.new (No-code AI)
**Ưu:**
- Ra prototype cực nhanh (1-3 ngày)
- AI tự generate UI/code → không cần biết code

**Nhược:**
- **Chi phí cao** ($20-50/tháng) vì subscription
- **Vendor lock-in** mạnh — khó export code chuẩn để self-host
- Phân quyền phức tạp (3 role + RLS) **làm rất khó** trên no-code
- Audit logging tự động (DB triggers) **không hỗ trợ** trực tiếp
- Generate Word/PDF custom layout phức tạp **không làm được**
- Khó scale > 50 user

**Kết luận:** Chỉ phù hợp demo nhanh, **không khuyến nghị** cho production có audit + phân quyền chặt.

#### Option D: Replit + Flask/FastAPI + PostgreSQL
**Ưu:**
- Code Python + deploy trên 1 nền tảng
- Free tier có giới hạn, plan Hacker $7/tháng

**Nhược:**
- Phải tự code rất nhiều (auth, RBAC, UI)
- Replit deploy không ổn định bằng Vercel
- DB free chỉ 1GB

**Khi nào nên chọn:** Học sinh/personal project, không khuyến nghị cho hệ thống nội bộ cấp trường.

#### Option E: Self-host Django + PostgreSQL trên VPS (DigitalOcean/Hetzner)
**Ưu:**
- Django Admin **đã có sẵn** giao diện quản lý + audit log (`django-simple-history`)
- Full control, không vendor lock-in
- VPS Hetzner CX11 chỉ ~$5/tháng

**Nhược:**
- Cần biết DevOps cơ bản (nginx, backup, SSL)
- Bảo trì server tự lo
- Django Admin UI hơi cũ, custom mất công

**Khi nào nên chọn:** Có người DevOps trong team + muốn sở hữu hoàn toàn data + ngân sách rất hạn chế.

### 2.3. Khuyến nghị cuối cùng

| Tình huống | Lựa chọn |
|-----------|----------|
| **Mặc định (recommend)** | Next.js + Supabase + Vercel |
| Ngân sách $0, chấp nhận UX cơ bản, ra nhanh | Streamlit + Supabase free |
| Cần demo trong 3 ngày để chốt requirement | Lovable (sau đó migrate sang Next.js) |
| Có sysadmin + muốn tự sở hữu | Django + VPS Hetzner |

**Quyết định:** Tiếp tục với **Next.js + Supabase + Vercel** cho plan này, nhưng kiến trúc DB schema và logic có thể tái sử dụng nếu sau này chuyển sang stack khác (Postgres-native).

---

## 3. KIẾN TRÚC HỆ THỐNG

```
┌─────────────────────────────────────────────────────────────┐
│                    Next.js 14 (App Router)                  │
│  ┌───────────────┐  ┌───────────────┐  ┌────────────────┐  │
│  │  Login UI     │  │  Admin UI     │  │  Reviewer UI   │  │
│  │  (no signup)  │  │  (manage all) │  │  (review Qs)   │  │
│  └───────────────┘  └───────────────┘  └────────────────┘  │
│                    Server Actions / API Routes              │
│              + Audit middleware (mọi mutation)              │
└──────────────────────────┬──────────────────────────────────┘
                           │
       ┌───────────────────┼─────────────────────┐
       ▼                   ▼                     ▼
┌─────────────┐    ┌──────────────┐      ┌──────────────┐
│  Supabase   │    │   Supabase   │      │   Vercel     │
│  Postgres   │    │   Storage    │      │   (Serverless│
│  + RLS      │    │  (raw files, │      │    + Edge)   │
│  + Triggers │    │   templates) │      │              │
│  + Audit    │    │              │      │              │
└─────────────┘    └──────────────┘      └──────────────┘
       │
       ▼
┌─────────────────────────────────┐
│  Document generation worker     │
│  (docxtemplater + libreoffice)  │
│  → Word + PDF                   │
└─────────────────────────────────┘
```

### 3.1. Lý do chọn stack (recap)
- **Next.js 14**: SSR cho trang admin/reviewer, Server Actions giảm boilerplate API, type-safe TypeScript.
- **Supabase**: PostgreSQL có RLS cho 3 role + Postgres triggers tự động ghi audit log mỗi khi data thay đổi. Auth không bật public signup → admin tạo user qua Service Role API.
- **Vercel**: Deploy 1-click từ GitHub, có Cron Jobs cho cleanup log cũ.
- **docxtemplater + LibreOffice**: Word từ template, convert Word → PDF bằng `libreoffice --headless`.

---

## 4. DATABASE SCHEMA (PostgreSQL / Supabase)

### 4.1. Bảng chính

```sql
-- ============================================
-- USERS - chỉ admin được tạo, không có public signup
-- ============================================
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  email TEXT NOT NULL UNIQUE,
  full_name TEXT,
  role TEXT NOT NULL CHECK (role IN ('admin', 'mod', 'reviewer')),
  is_active BOOLEAN DEFAULT TRUE,
  must_change_password BOOLEAN DEFAULT TRUE,   -- bắt đổi mật khẩu lần đầu
  created_by UUID REFERENCES profiles(id),     -- admin nào tạo
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_login_at TIMESTAMPTZ,
  deactivated_at TIMESTAMPTZ
);

-- ============================================
-- SUBJECTS & CHAPTERS
-- ============================================
CREATE TABLE subjects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT UNIQUE NOT NULL,        -- "MATH12", "PHYS11"
  name TEXT NOT NULL,
  description TEXT,
  created_by UUID REFERENCES profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE chapters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  order_idx INT DEFAULT 0
);

-- ============================================
-- QUESTION BANKS
-- ============================================
CREATE TABLE question_banks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id UUID REFERENCES subjects(id),
  name TEXT NOT NULL,
  type TEXT CHECK (type IN ('raw', 'refined')),
  description TEXT,
  created_by UUID REFERENCES profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- QUESTIONS
-- ============================================
CREATE TABLE questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  bank_id UUID REFERENCES question_banks(id) ON DELETE CASCADE,
  chapter_id UUID REFERENCES chapters(id),
  question_type TEXT NOT NULL CHECK (question_type IN
    ('MCQ_SINGLE','MCQ_MULTI','TRUE_FALSE','SHORT_ANSWER')),
  content TEXT NOT NULL,
  options JSONB,
  correct_answer JSONB,
  explanation TEXT,
  rubric JSONB,
  difficulty TEXT NOT NULL CHECK (difficulty IN ('NB','TH','VD','VDC')),
  tags TEXT[],
  status TEXT NOT NULL DEFAULT 'pending_review' CHECK (status IN
    ('pending_review','reviewed','approved','rejected')),
  source_file TEXT,
  created_by UUID REFERENCES profiles(id),
  reviewed_by UUID REFERENCES profiles(id),
  approved_by UUID REFERENCES profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_questions_bank ON questions(bank_id);
CREATE INDEX idx_questions_status ON questions(status);
CREATE INDEX idx_questions_difficulty ON questions(difficulty);
CREATE INDEX idx_questions_chapter ON questions(chapter_id);

-- ============================================
-- REVIEWER_ASSIGNMENTS (đổi từ teacher_assignments)
-- ============================================
CREATE TABLE reviewer_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  reviewer_id UUID REFERENCES profiles(id),
  bank_id UUID REFERENCES question_banks(id),
  chapter_id UUID REFERENCES chapters(id),   -- nullable
  assigned_by UUID REFERENCES profiles(id),
  assigned_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(reviewer_id, bank_id, chapter_id)
);

-- ============================================
-- EXAM STRUCTURES, TEMPLATES, EXAMS (giống v1.0)
-- ============================================
CREATE TABLE exam_structures (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  subject_id UUID REFERENCES subjects(id),
  total_questions INT,
  total_time_min INT,
  config JSONB NOT NULL,
  created_by UUID REFERENCES profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  file_path TEXT NOT NULL,
  template_type TEXT CHECK (template_type IN ('exam', 'answer_key')),
  created_by UUID REFERENCES profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE exams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT UNIQUE,
  title TEXT NOT NULL,
  structure_id UUID REFERENCES exam_structures(id),
  template_id UUID REFERENCES templates(id),
  question_ids UUID[],
  shuffle_seed INT,
  exam_file_path TEXT,
  exam_pdf_path TEXT,
  answer_key_docx_path TEXT,
  answer_key_pdf_path TEXT,
  generated_by UUID REFERENCES profiles(id),
  generated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- AUDIT LOG - LƯU VẾT MỌI HÀNH ĐỘNG (xem section 12)
-- ============================================
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  actor_id UUID REFERENCES profiles(id),
  actor_email TEXT,                     -- snapshot tại thời điểm action
  actor_role TEXT,                      -- snapshot role
  action TEXT NOT NULL,                 -- xem danh sách action ở section 12
  entity_type TEXT,                     -- 'question', 'exam', 'user', 'bank'...
  entity_id UUID,
  before_value JSONB,
  after_value JSONB,
  diff JSONB,                           -- chỉ các field thay đổi
  request_ip INET,
  user_agent TEXT,
  session_id UUID,
  note TEXT,
  metadata JSONB                        -- flexible cho action đặc thù
);

CREATE INDEX idx_audit_ts ON audit_logs(ts DESC);
CREATE INDEX idx_audit_actor ON audit_logs(actor_id);
CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_logs(action);

-- Partition theo tháng (khi >1M records)
-- CREATE TABLE audit_logs_2026_05 PARTITION OF audit_logs
--   FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### 4.2. Row Level Security (RLS) — ví dụ rule
```sql
-- Reviewer chỉ sửa được câu hỏi trong bank được phân công
CREATE POLICY reviewer_edit_assigned_questions ON questions
  FOR UPDATE USING (
    EXISTS (
      SELECT 1 FROM reviewer_assignments ra
      WHERE ra.reviewer_id = auth.uid()
        AND ra.bank_id = questions.bank_id
        AND (ra.chapter_id IS NULL OR ra.chapter_id = questions.chapter_id)
    )
    AND (SELECT role FROM profiles WHERE id = auth.uid()) = 'reviewer'
  );

-- Admin & Mod thấy hết
CREATE POLICY admin_mod_full_access ON questions
  FOR ALL USING (
    (SELECT role FROM profiles WHERE id = auth.uid()) IN ('admin', 'mod')
  );

-- Audit log: chỉ admin xem được, không ai sửa/xóa
CREATE POLICY audit_admin_only_select ON audit_logs
  FOR SELECT USING (
    (SELECT role FROM profiles WHERE id = auth.uid()) = 'admin'
  );
CREATE POLICY audit_no_update ON audit_logs FOR UPDATE USING (false);
CREATE POLICY audit_no_delete ON audit_logs FOR DELETE USING (false);
```

---

## 5. VAI TRÒ & QUYỀN HẠN (3 ROLE)

| Role | Mô tả | Quyền chính |
|------|-------|-------------|
| **Admin** | Quản trị toàn hệ thống | Tạo user, gán role, mọi quyền dữ liệu, xem audit log, cấu hình hệ thống |
| **Mod** | Quản lý nội dung | Upload bank thô, phân reviewer, approve/reject câu hỏi, sinh đề |
| **Reviewer** | Giáo viên duyệt L1 | Chỉ sửa được câu hỏi/đáp án/độ khó trong bank được phân công |

**Đã bỏ vai trò Viewer & Teacher** (so với v1.0). Mọi user phải có tài khoản — không có ai truy cập "công khai".

### Permission Matrix chi tiết

| Hành động | Admin | Mod | Reviewer |
|----------|:-----:|:---:|:--------:|
| **Quản lý user** | | | |
| Tạo tài khoản user mới | ✅ | ❌ | ❌ |
| Đặt/đổi role của user | ✅ | ❌ | ❌ |
| Reset password user | ✅ | ❌ | ❌ |
| Vô hiệu hóa (deactivate) user | ✅ | ❌ | ❌ |
| **Nội dung** | | | |
| Tạo Subject / Chapter | ✅ | ✅ | ❌ |
| Upload bank thô | ✅ | ✅ | ❌ |
| Phân quyền reviewer | ✅ | ✅ | ❌ |
| Sửa câu hỏi (duyệt L1) | ✅ | ✅ | ✅ (assigned only) |
| Approve / Reject L2 | ✅ | ✅ | ❌ |
| **Sinh đề** | | | |
| Tạo / sửa Exam Structure | ✅ | ❌ | ❌ |
| Upload Template | ✅ | ✅ | ❌ |
| Sinh đề | ✅ | ✅ | ❌ |
| Tải đề đã sinh | ✅ | ✅ | ❌ |
| **Audit & cấu hình** | | | |
| Xem audit log (toàn hệ thống) | ✅ | ❌ | ❌ |
| Xem audit log của chính mình | ✅ | ✅ | ✅ |
| Cấu hình retention log | ✅ | ❌ | ❌ |
| Export audit log | ✅ | ❌ | ❌ |

---

## 6. CƠ CHẾ TẠO & QUẢN LÝ TÀI KHOẢN

### 6.1. Nguyên tắc
- **Không có public signup**: trang `/signup` không tồn tại
- **Chỉ admin tạo user**: thông qua trang `/admin/users`
- Login form duy nhất: `/login` (email + password)
- Lần đầu đăng nhập: bắt đổi password (flag `must_change_password = true`)
- Có "Quên mật khẩu" → gửi reset link qua email (Supabase Auth có sẵn)

### 6.2. Cách tắt public signup trong Supabase
Trong Supabase Dashboard → Authentication → Settings:
- **Disable email signup** (nếu Supabase hỗ trợ) HOẶC
- Set `auth.enable_signup = false` trong config
- Bật **Confirm email** để chặn auto-signup qua API

Backend phải dùng **Service Role Key** (không expose client) để gọi `auth.admin.createUser()`:

```typescript
// app/api/admin/users/route.ts (chỉ admin call được)
import { createClient } from '@supabase/supabase-js'

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // server-only
)

export async function POST(req: Request) {
  // 1. Verify caller is admin
  const { user } = await verifyAdminRole(req)
  if (!user) return new Response('Forbidden', { status: 403 })

  const { email, full_name, role } = await req.json()

  // 2. Tạo user với password tạm
  const tempPassword = generateTempPassword(12)
  const { data, error } = await supabaseAdmin.auth.admin.createUser({
    email,
    password: tempPassword,
    email_confirm: true,
    user_metadata: { full_name }
  })

  // 3. Insert profile
  await supabaseAdmin.from('profiles').insert({
    id: data.user.id,
    email,
    full_name,
    role,
    must_change_password: true,
    created_by: user.id
  })

  // 4. Gửi email chứa password tạm (hoặc magic link reset)
  await sendWelcomeEmail(email, tempPassword)

  // 5. Audit log (tự động qua trigger, xem section 12)
  return Response.json({ ok: true })
}
```

### 6.3. Trang `/admin/users` — UI gợi ý
- Bảng list user: email, full_name, role, status (active/deactivated), last_login, created_at, created_by
- Filter theo role, status
- Action: ➕ Tạo user mới | ✏️ Edit role | 🔄 Reset password | ⛔ Deactivate
- Modal tạo user: email + full_name + role (select) → click Submit → user nhận email

### 6.4. Vô hiệu hóa thay vì xóa
- Không cho phép xóa user vĩnh viễn (mất audit trail)
- Chỉ set `is_active = false` + `deactivated_at = NOW()`
- User bị deactivate không login được; mọi `auth.uid()` query trả null trong RLS

---

## 7. CHI TIẾT FLOW NGHIỆP VỤ

### Bước 1: Admin tạo tài khoản & cấp phát role
- Admin vào `/admin/users` → tạo tài khoản cho mod & reviewer
- User nhận email với link đăng nhập + password tạm
- Lần đầu login → bị bắt đổi password

### Bước 2: Upload Question Bank thô
- **Actor**: Mod / Admin
- **Input**: File `.docx` / `.xlsx` / `.csv` / `.txt`
- **Output**: Records trong `questions` với `status = pending_review`, gắn vào 1 `question_banks(type='raw')`
- Background parser trích câu hỏi → insert DB

### Bước 3: Admin/Mod phân quyền reviewer
- UI: Bảng matrix `reviewer × bank/chapter`, tick để phân
- Insert vào `reviewer_assignments`

### Bước 4: Reviewer duyệt L1
- Trang `/reviewer/review` — danh sách câu hỏi `status='pending_review'` thuộc bank được giao
- Hành động: sửa nội dung / đáp án / độ khó → click "Đã duyệt" → `status = 'reviewed'`

### Bước 5: Admin/Mod phê duyệt L2
- Trang `/approve` — danh sách câu hỏi `status='reviewed'`
- Approve → `status='approved'`, di chuyển sang bank refined
- Reject + ghi chú → `status='rejected'`, reviewer chỉnh lại

### Bước 6: Admin tạo Exam Structure
- Form builder kéo-thả các section, mỗi section có rule `(chapter, difficulty, count)`
- Validate khả thi trước khi save

### Bước 7: Sinh đề ngẫu nhiên
- Input: structure + template + số mã đề
- Thuật toán random sampling theo rule, có seed để reproducible
- Render `.docx` qua docxtemplater, convert PDF qua LibreOffice
- Sinh đồng thời file đề + file HDC

### Bước 8: Tải xuống
- Download `.docx` hoặc `.pdf`, hoặc zip toàn bộ mã đề + HDC

> **Mọi bước trên đều được tự động ghi vào `audit_logs` — xem chi tiết ở section 12.**

---

## 8. PARSING QUESTION BANK THÔ

(Giữ nguyên nội dung từ v1.0)

### 8.1. Excel/CSV (khuyến nghị cho MVP)
Mỗi dòng = 1 câu hỏi với schema cố định.

### 8.2. Word (.docx)
Mammoth.js extract text + regex match pattern, có LLM fallback.

### 8.3. Plain text / Markdown
Parser regex + hỗ trợ LaTeX inline.

---

## 9. DOCUMENT GENERATION

(Giữ nguyên từ v1.0 — docxtemplater + LibreOffice convert PDF)

---

## 10. CẤU TRÚC THƯ MỤC PROJECT

```
sinhdedoai/
├── app/                          # Next.js App Router
│   ├── login/page.tsx            # CHỈ có login, KHÔNG có signup
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   ├── admin/
│   │   │   ├── users/            # tạo & quản lý user
│   │   │   ├── subjects/
│   │   │   ├── banks/
│   │   │   ├── structures/
│   │   │   ├── templates/
│   │   │   ├── approve/
│   │   │   └── audit-logs/       # XEM AUDIT LOG
│   │   ├── reviewer/
│   │   │   └── review/
│   │   ├── exams/
│   │   │   ├── generate/
│   │   │   └── history/
│   │   └── dashboard/page.tsx
│   ├── api/
│   │   ├── admin/users/route.ts  # tạo user (service role)
│   │   ├── upload/route.ts
│   │   ├── parse/route.ts
│   │   ├── generate-exam/route.ts
│   │   └── audit/route.ts        # query log
│   └── layout.tsx
├── lib/
│   ├── supabase/
│   ├── audit/
│   │   ├── logger.ts             # central audit logger
│   │   └── events.ts             # enum các action
│   ├── parsers/
│   ├── generators/
│   └── auth.ts
├── supabase/
│   ├── migrations/
│   │   ├── 001_schema.sql
│   │   ├── 002_rls.sql
│   │   ├── 003_audit_triggers.sql  # triggers tự động log
│   │   └── 004_seed_admin.sql      # tạo admin đầu tiên
│   └── functions/                  # Supabase Edge Functions
├── docker/
│   └── pdf-worker/
└── ...
```

---

## 11. LỘ TRÌNH PHÁT TRIỂN (6 SPRINTS - ~6 TUẦN)

### Sprint 1: Foundation + User Management
- [ ] Khởi tạo Next.js + Supabase
- [ ] Schema DB + RLS policies
- [ ] Tắt public signup, tạo admin đầu tiên qua seed
- [ ] Trang `/login` (no signup) + role-based redirect
- [ ] Trang `/admin/users` (create/edit/deactivate)
- [ ] Flow đổi password lần đầu

### Sprint 2: Subjects & Banks
- [ ] CRUD subjects + chapters
- [ ] CRUD question banks
- [ ] Upload Excel/CSV → parser cơ bản
- [ ] Question editor (rich text + LaTeX preview KaTeX)
- [ ] **Audit triggers cho các bảng chính**

### Sprint 3: Duyệt L1 & L2
- [ ] Trang reviewer review filter theo assignment
- [ ] Edit inline + autosave
- [ ] Trang admin approve với diff viewer
- [ ] Notifications

### Sprint 4: Exam Structure & Sinh đề
- [ ] Structure builder UI
- [ ] Validate khả thi
- [ ] Upload template .docx
- [ ] Random sampling logic
- [ ] Render .docx

### Sprint 5: PDF, HDC, Polish
- [ ] PDF generation (LibreOffice worker)
- [ ] Sinh đồng thời đề + HDC
- [ ] Trang download có preview
- [ ] Export zip

### Sprint 6: Audit UI + Testing + Deploy
- [ ] **Trang `/admin/audit-logs`**: filter, search, export CSV
- [ ] Unit tests + E2E tests
- [ ] Backup strategy (Supabase daily)
- [ ] Documentation cho admin/reviewer
- [ ] Deploy production

---

## 12. AUDIT LOGGING — LƯU VẾT TOÀN HỆ THỐNG ⭐ MỚI

### 12.1. Mục tiêu
- Mọi hành động ghi vào DB **không thể bị xóa hoặc sửa**
- Trả lời được câu hỏi: *"Ai đã làm gì, khi nào, trên dữ liệu nào, kết quả trước/sau ra sao?"*
- Truy vết được lỗi nghiệp vụ (vd: câu hỏi đáp án sai bị ai sửa)
- Đáp ứng yêu cầu compliance/báo cáo nội bộ

### 12.2. Cấp độ ghi log (Logging tiers)

**Tier 1 — Database trigger (tự động, không thể bỏ sót)**
- Postgres trigger `AFTER INSERT/UPDATE/DELETE` trên các bảng quan trọng
- Ghi `before` (OLD), `after` (NEW), `diff` chỉ các field thay đổi
- Capture: actor_id (qua `current_setting('app.current_user_id')`), timestamp

**Tier 2 — Application-level log (cho action không gắn 1 dòng DB cụ thể)**
- Login / logout
- Failed login attempts
- Upload file (kèm filename + size)
- Generate exam (kèm structure + số mã đề)
- Export audit log
- Reset password người khác

**Tier 3 — System log (debug/error)**
- Lưu riêng (vd: Sentry/Logtail) — không vào `audit_logs`
- Không phải audit business action

### 12.3. Danh sách Action được log (action enum)

```typescript
// lib/audit/events.ts
export const AUDIT_ACTIONS = {
  // Auth
  AUTH_LOGIN: 'auth.login',
  AUTH_LOGOUT: 'auth.logout',
  AUTH_LOGIN_FAILED: 'auth.login_failed',
  AUTH_PASSWORD_CHANGED: 'auth.password_changed',
  AUTH_PASSWORD_RESET_REQUESTED: 'auth.password_reset_requested',

  // User management
  USER_CREATED: 'user.created',
  USER_ROLE_CHANGED: 'user.role_changed',
  USER_DEACTIVATED: 'user.deactivated',
  USER_REACTIVATED: 'user.reactivated',
  USER_PASSWORD_RESET_BY_ADMIN: 'user.password_reset_by_admin',

  // Subject / Chapter
  SUBJECT_CREATED: 'subject.created',
  SUBJECT_UPDATED: 'subject.updated',
  SUBJECT_DELETED: 'subject.deleted',
  CHAPTER_CREATED: 'chapter.created',
  CHAPTER_UPDATED: 'chapter.updated',

  // Question Bank
  BANK_CREATED: 'bank.created',
  BANK_FILE_UPLOADED: 'bank.file_uploaded',
  BANK_PARSED: 'bank.parsed',          // metadata: số câu thành công/lỗi

  // Question
  QUESTION_CREATED: 'question.created',
  QUESTION_UPDATED: 'question.updated',  // before/after content, options, difficulty
  QUESTION_REVIEWED: 'question.reviewed', // đánh dấu duyệt L1
  QUESTION_APPROVED: 'question.approved',
  QUESTION_REJECTED: 'question.rejected',  // có note
  QUESTION_DELETED: 'question.deleted',

  // Reviewer assignment
  REVIEWER_ASSIGNED: 'reviewer.assigned',
  REVIEWER_UNASSIGNED: 'reviewer.unassigned',

  // Exam structure
  STRUCTURE_CREATED: 'structure.created',
  STRUCTURE_UPDATED: 'structure.updated',
  STRUCTURE_DELETED: 'structure.deleted',

  // Template
  TEMPLATE_UPLOADED: 'template.uploaded',
  TEMPLATE_DELETED: 'template.deleted',

  // Exam generation
  EXAM_GENERATED: 'exam.generated',     // metadata: structure_id, count, seed
  EXAM_DOWNLOADED: 'exam.downloaded',
  EXAM_DELETED: 'exam.deleted',

  // Audit
  AUDIT_LOG_VIEWED: 'audit.viewed',
  AUDIT_LOG_EXPORTED: 'audit.exported',
} as const;
```

### 12.4. PostgreSQL Trigger function (tự động)

```sql
-- Function chung capture old/new diff
CREATE OR REPLACE FUNCTION fn_audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
  actor UUID;
  actor_email TEXT;
  actor_role TEXT;
  diff_value JSONB;
BEGIN
  -- Lấy user hiện tại từ app context
  actor := COALESCE(
    NULLIF(current_setting('app.current_user_id', true), '')::UUID,
    NULL
  );

  IF actor IS NOT NULL THEN
    SELECT email, role INTO actor_email, actor_role
    FROM profiles WHERE id = actor;
  END IF;

  -- Tính diff (chỉ field thay đổi)
  IF TG_OP = 'UPDATE' THEN
    SELECT jsonb_object_agg(key, value) INTO diff_value
    FROM jsonb_each(to_jsonb(NEW))
    WHERE to_jsonb(NEW)->key IS DISTINCT FROM to_jsonb(OLD)->key;
  END IF;

  INSERT INTO audit_logs(
    actor_id, actor_email, actor_role,
    action, entity_type, entity_id,
    before_value, after_value, diff
  ) VALUES (
    actor, actor_email, actor_role,
    TG_TABLE_NAME || '.' || lower(TG_OP),
    TG_TABLE_NAME,
    COALESCE((NEW).id, (OLD).id),
    CASE WHEN TG_OP <> 'INSERT' THEN to_jsonb(OLD) ELSE NULL END,
    CASE WHEN TG_OP <> 'DELETE' THEN to_jsonb(NEW) ELSE NULL END,
    diff_value
  );

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Apply cho các bảng quan trọng
CREATE TRIGGER trg_audit_questions
  AFTER INSERT OR UPDATE OR DELETE ON questions
  FOR EACH ROW EXECUTE FUNCTION fn_audit_trigger();

CREATE TRIGGER trg_audit_profiles
  AFTER INSERT OR UPDATE OR DELETE ON profiles
  FOR EACH ROW EXECUTE FUNCTION fn_audit_trigger();

CREATE TRIGGER trg_audit_exam_structures
  AFTER INSERT OR UPDATE OR DELETE ON exam_structures
  FOR EACH ROW EXECUTE FUNCTION fn_audit_trigger();

CREATE TRIGGER trg_audit_exams
  AFTER INSERT OR UPDATE OR DELETE ON exams
  FOR EACH ROW EXECUTE FUNCTION fn_audit_trigger();

CREATE TRIGGER trg_audit_reviewer_assignments
  AFTER INSERT OR UPDATE OR DELETE ON reviewer_assignments
  FOR EACH ROW EXECUTE FUNCTION fn_audit_trigger();

-- Prevent update/delete trên audit_logs
CREATE RULE no_update_audit AS ON UPDATE TO audit_logs DO INSTEAD NOTHING;
CREATE RULE no_delete_audit AS ON DELETE TO audit_logs DO INSTEAD NOTHING;
```

### 12.5. Set actor context trong app (Next.js)

```typescript
// lib/supabase/server.ts
export async function getServerSupabase() {
  const supabase = createServerClient(...)
  const { data: { user } } = await supabase.auth.getUser()

  if (user) {
    // Set actor_id để trigger biết ai đang thao tác
    await supabase.rpc('set_config', {
      key: 'app.current_user_id',
      value: user.id,
      is_local: true   // local = chỉ trong transaction này
    })
  }
  return supabase
}
```

### 12.6. Application-level logger (cho action không gắn DB row)

```typescript
// lib/audit/logger.ts
export async function logAudit(params: {
  action: string;
  entity_type?: string;
  entity_id?: string;
  metadata?: any;
  note?: string;
  req?: Request;
}) {
  const supabase = await getServerSupabase()
  const { data: { user } } = await supabase.auth.getUser()
  const profile = await supabase
    .from('profiles').select('email, role').eq('id', user?.id).single()

  await supabase.from('audit_logs').insert({
    actor_id: user?.id,
    actor_email: profile.data?.email,
    actor_role: profile.data?.role,
    action: params.action,
    entity_type: params.entity_type,
    entity_id: params.entity_id,
    metadata: params.metadata,
    note: params.note,
    request_ip: params.req?.headers.get('x-forwarded-for'),
    user_agent: params.req?.headers.get('user-agent')
  })
}

// Cách dùng:
await logAudit({
  action: AUDIT_ACTIONS.EXAM_GENERATED,
  entity_type: 'exam_structure',
  entity_id: structureId,
  metadata: { num_versions: 4, seed: 12345 }
})
```

### 12.7. UI xem Audit Log (`/admin/audit-logs`)

**Filter:**
- Time range (preset: last 24h, 7d, 30d, custom)
- Actor (chọn user)
- Action (multi-select từ enum)
- Entity type
- Entity ID (search by UUID)
- Free-text search trong `note`/`metadata`

**Hiển thị (table):**

| Time | Actor | Role | Action | Entity | Diff preview | IP |
|------|-------|------|--------|--------|--------------|-----|
| 2026-05-21 10:23 | nguyen.an@... | reviewer | question.updated | Q-abc123 | difficulty: TH→VD | 192.168.1.5 |
| 2026-05-21 10:18 | admin@... | admin | exam.generated | Structure-xyz | 4 mã đề, seed 12345 | 10.0.0.1 |

**Click row** → modal hiển thị full before/after JSON, đẹp như git diff.

**Export:** Button "Export CSV" → tải log đã filter.

### 12.8. Retention Policy
- Mặc định **giữ vĩnh viễn** trong DB (chiếm dung lượng nhưng audit cần)
- Sau 12 tháng → tự động archive sang Supabase Storage `audit-archive/{year}-{month}.jsonl.gz` qua Vercel Cron monthly
- Bảng `audit_logs` chỉ giữ 12 tháng gần nhất để query nhanh
- Admin export được archive khi cần

### 12.9. Cảnh báo / Alerts (nice-to-have, Sprint 6+)
- > 5 lần `auth.login_failed` từ 1 IP trong 10 phút → alert email admin
- 1 user xóa > 50 câu hỏi trong 1 giờ → alert
- Role change từ reviewer → admin → alert ngay

### 12.10. Bảng tóm tắt: hành động nào được log ở đâu

| Hành động | Trigger DB | App logger |
|-----------|:----------:|:----------:|
| Tạo/sửa/xóa question | ✅ | |
| Tạo/sửa user, đổi role | ✅ | |
| Login / logout | | ✅ |
| Login failed | | ✅ |
| Upload file bank | ✅ (bank row) | ✅ (filename, size) |
| Parse bank xong | | ✅ (số câu success/fail) |
| Approve / Reject Q | ✅ (status change) | ✅ (kèm note reject) |
| Generate exam | ✅ (exam row) | ✅ (seed, count) |
| Download đề | | ✅ |
| Xem audit log | | ✅ |
| Export audit log | | ✅ |

---

## 13. RỦI RO & PHƯƠNG ÁN

| Rủi ro | Mức độ | Mitigation |
|--------|--------|------------|
| Audit log phình to nhanh | Trung | Partition theo tháng + archive sau 12 tháng |
| Trigger DB làm chậm write | Thấp-Trung | Trigger chỉ INSERT (không SELECT), index đầy đủ. Benchmark trước khi bật cho bảng lớn |
| Mất `app.current_user_id` context | Trung | Có wrapper `getServerSupabase()` set tự động, lint rule chặn dùng raw client |
| Parse Word ra câu hỏi sai format | Cao | Excel chuẩn trước; .docx free-form dùng LLM fallback |
| LibreOffice không chạy trên Vercel | Trung | Worker riêng trên Railway/Fly.io |
| Số câu/độ khó không đủ khi sinh đề | Cao | Validate trước, warning rõ ràng |
| RLS bug → reviewer thấy ngoài quyền | Cao | Test policy bằng `supabase test db` trước merge |
| User mất password tạm | Thấp | Magic link reset từ Supabase Auth |
| Admin duy nhất bị khóa tài khoản | Trung | Seed luôn 2 admin lúc init; có CLI script khôi phục admin từ service role key |

---

## 14. CHI PHÍ HẠ TẦNG (MVP) — SO SÁNH

| Stack | MVP/tháng | Khi 100 users | Khi 500 users |
|-------|-----------|---------------|---------------|
| **Next.js + Supabase + Vercel** ⭐ | $0 | $25-30 | $45-60 |
| Streamlit + Supabase | $0 | $25 | $40 (limit) |
| Lovable / Bolt.new | $20-50 | $50-100 | $100+ |
| Django + VPS Hetzner | $5 | $10 | $20 |

LLM API parser (optional): ~$5-20/tháng tuỳ usage (cho .docx phức tạp).

---

## 15. CÁC TÍNH NĂNG TƯƠNG LAI (POST-MVP)

- [ ] OCR file .pdf scan
- [ ] Plagiarism check giữa các câu
- [ ] Dashboard thống kê: câu nào sai nhiều, độ khó thực tế
- [ ] Tích hợp LMS (Moodle/Google Classroom)
- [ ] Mobile app cho reviewer
- [ ] AI gợi ý câu hỏi tương đương (vector embedding)
- [ ] Online exam mode (học sinh làm trực tiếp)
- [ ] Multi-tenant
- [ ] **SIEM integration** (gửi audit log sang Splunk/ELK)
- [ ] **2FA bắt buộc cho admin**

---

## 16. CHECKLIST KHỞI ĐỘNG

Trước khi bắt đầu code:
- [ ] Tạo project Supabase → tắt public signup
- [ ] Lấy URL + anon key + **service role key** (cho admin tạo user)
- [ ] Tạo project Vercel, link GitHub
- [ ] Cài Node.js 20+, pnpm
- [ ] Chuẩn bị 1 file Excel mẫu 50 câu hỏi
- [ ] Chuẩn bị 1 template .docx mẫu
- [ ] Thống nhất danh sách độ khó (NB/TH/VD/VDC?)
- [ ] Thống nhất loại câu hỏi (cần thêm ghép cặp/sắp xếp?)
- [ ] **Quyết định email service gửi welcome mail** (Resend/SendGrid free tier)
- [ ] **Tạo seed admin đầu tiên** (file `004_seed_admin.sql`)

---

## 17. LINKS THAM KHẢO

- Next.js: https://nextjs.org/docs
- Supabase Auth (no signup): https://supabase.com/docs/guides/auth/auth-helpers
- Supabase RLS: https://supabase.com/docs/guides/auth/row-level-security
- Postgres audit triggers: https://supabase.com/blog/postgres-audit
- docxtemplater: https://docxtemplater.com
- shadcn/ui: https://ui.shadcn.com
- KaTeX: https://katex.org
- Resend (email): https://resend.com (free 3k emails/tháng)

---

*Plan v1.1 — sẵn sàng review & điều chỉnh trước khi bắt tay vào code.*
