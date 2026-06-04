# AI/LLM Chat with SMC Data Platform — Tài liệu PoC (AC1–AC4)

**Project:** AI/LLM Chat with SMC Data Platform  
**Mentor:** Phạm Văn Dương (v.duongpv15@vinsmartfuture.tech)  
**Intern 1:** Nguyễn Hoàng Khải Minh (26ai.minhnhk@vinuni.edu.vn)  
**Intern 2:** Nguyễn Quốc Nam (26ai.namnq@vinuni.edu.vn)  
**Thời gian:** 6 tuần

---

## AC1 — Product Backlog cho PoC (Must-have / Should-have / Could-have)

### 1.1 Bối cảnh lý luận và phân loại hệ thống
Dự án AI/LLM Chat with SMC Data Platform thuộc nhóm **Natural Language Interface for Databases (NLIDB)**. Theo khảo sát của Liu & Xu (2025) *"NLI4DB: A Systematic Review of Natural Language Interfaces for Databases"* (arXiv:2503.02435), các hệ thống NLIDB hiện đại dựa trên LLM bao gồm các lớp cốt lõi: Natural Language Understanding, Schema Linking, SQL Generation & Execution, và Answer Synthesis.

Backlog Must-have bao phủ đầy đủ các thành phần theo thiết kế NL2SQL production-ready của Mahakali (2025).

### 1.2 Nền tảng domain: ISO Smart City & ESG Indicators
SMC Data Platform được xây dựng theo:
- ISO 37120:2018 (104 KPIs trên 19 chủ đề)
- ISO 37122:2019 (Smart City Indicators)
- ISO 37125:2024 (ESG Indicators for Cities)

### 🔴 Must-have (15 items)

#### Nhóm A — Input & Pipeline Core
| ID | Chức năng | Mô tả | Tuần |
|----|-----------|-------|------|
| M-01 | Nhận câu hỏi tiếng Việt | Web UI (Streamlit/React) hoặc API `/ask` | T3 |
| M-02 | Sinh SQL từ câu hỏi (Text-to-SQL) | LLM sinh SQL cho StarRocks với schema context | T3 |
| M-03 | Whitelist bảng/cột | File YAML quản lý bảng/cột được phép | T2 |
| M-04 | Validate SQL trước khi thực thi | Kiểm tra cú pháp, whitelist, nguy hiểm commands | T3–4 |
| M-05 | Thực thi truy vấn trên StarRocks | Kết nối JDBC/MySQL protocol | T4 |
| M-06 | Trả lời tự nhiên từ kết quả DB | LLM tổng hợp kết quả thành tiếng Việt | T3 |
| M-07 | Logging toàn pipeline | Log JSON chi tiết từng bước | T3–4 |
| M-08 | Xử lý lỗi & fallback | Xử lý ≥6 loại lỗi phổ biến | T3–4 |

#### Nhóm B — Intelligence & Quality
| ID | Chức năng | Mô tả | Tuần |
|----|-----------|-------|------|
| M-09 | Semantic Layer (Business Term Mapping) | YAML mapping thuật ngữ nghiệp vụ | T5 |
| M-10 | System prompt chuẩn hóa cho Smart City | Prompt domain-specific + few-shot | T5 |
| M-11 | Báo cáo so sánh trước/sau cải tiến | So sánh 30 câu hỏi baseline vs improved | T5 |

#### Nhóm C — UX & Security
| ID | Chức năng | Mô tả | Tuần |
|----|-----------|-------|------|
| M-12 | Lịch sử hội thoại (multi-turn) | Giữ context 5 lượt | T4 |
| M-13 | Hiển thị SQL sinh ra (Transparency) | Toggle xem SQL | T3–4 |
| M-14 | Phân quyền dữ liệu theo role | RBAC (viewer/analyst/admin) | T4 |
| M-15 | Hỗ trợ đa ngôn ngữ (VN/EN) | Detect và trả lời đúng ngôn ngữ | T3 |

### 🟡 Should-have (5 items)
- S-01: Giao diện quản trị whitelist
- S-02: Caching kết quả truy vấn
- S-03: Phát hiện câu hỏi ngoài phạm vi
- S-04: Gợi ý câu hỏi liên quan
- S-05: Export kết quả (CSV/Excel)

### 🟢 Could-have (5 items)
- C-01: Monitoring dashboard
- C-02: Đánh giá câu trả lời (thumbs up/down)
- C-03: Alert KPI vượt ngưỡng
- C-04: Trực quan hóa biểu đồ
- C-05: Tích hợp SSO

---

## AC2 — Kế hoạch triển khai 6 tuần & Phân công vai trò

**Intern 1 (Minh):** AI/LLM Engineer (Prompt, Text-to-SQL, Semantic Layer)  
**Intern 2 (Nam):** Backend/Data Engineer (StarRocks, API, Validator, Logging)

### Tuần 1–2: Nền tảng & Kiến trúc
- Tuần 1: Phân tích nghiệp vụ, khảo sát schema, xây dựng 30 câu hỏi test
- Tuần 2: Thiết kế kiến trúc, API spec, prompt v1, test plan

### Tuần 3–4: Xây dựng PoC Core
- Tuần 3: Web UI, Text-to-SQL pipeline, API, Whitelist
- Tuần 4: Kết nối StarRocks, Validator, Logging, Baseline Evaluation

### Tuần 5–6: Cải tiến & Hoàn thiện
- Tuần 5: Semantic Layer, Prompt v2, Post-improvement Evaluation
- Tuần 6: Tài liệu, API docs, Roadmap, Demo

---

## AC3 — Bộ tiêu chí đánh giá PoC

**5 Tiêu chí chính:**

1. **SQL Accuracy** (Execution Accuracy) → Target: ≥80% (Post)
2. **Answer Quality** (RAGAS-based) → Target: ≥3.8/5
3. **Response Time** (p90 End-to-End) → Target: <10s
4. **Error Handling** → Target: ≥80%
5. **Extensibility** → Target: ≥7/10

Đánh giá 2 lần: Baseline (tuần 4) và Post-improvement (tuần 5).

---

## AC4 — Bộ 30 câu hỏi test

**Nhóm 1: Môi trường & Chất lượng không khí** (5 câu)  
**Nhóm 2: Năng lượng & Tiêu thụ điện** (5 câu)  
**Nhóm 3: Giao thông & Di chuyển** (5 câu)  
**Nhóm 4: Nước & Vệ sinh** (5 câu)  
**Nhóm 5: Dân số & Dịch vụ xã hội** (5 câu)  
**Nhóm 6: Kinh tế & ESG** (5 câu)

*(Chi tiết 30 câu hỏi + ground truth SQL mô tả nằm trong tài liệu gốc)*

---

**Tài liệu này được tạo phục vụ PoC — Tháng 6/2025**
