# AI/LLM Chat with SMC Data Platform — Tài liệu PoC (AC1–AC3)

**Project:** AI/LLM Chat with SMC Data Platform
**Mentor:** Phạm Văn Dương (v.duongpv15@vinsmartfuture.tech)
**Intern 1:** Nguyễn Hoàng Khải Minh (26ai.minhnhk@vinuni.edu.vn)
**Intern 2:** Nguyễn Quốc Nam (26ai.namnq@vinuni.edu.vn)
**Thời gian:** 6 tuần

---

## AC1 — Product Backlog cho PoC (Must-have / Should-have / Could-have)

### Cơ sở lý thuyết

Hệ thống tuân theo thiết kế NL2SQL được đề xuất bởi Mahakali (2025) trên Medium *"NL2SQL System Design Guide 2025"*, một hệ thống production-ready cần tối thiểu: input interface, context-rich prompt builder, LLM SQL generator, schema service, SQL validator và DB connector. Backlog Must-have của dự án này bao phủ đầy đủ các thành phần này trong phạm vi PoC 6 tuần.

Về domain dữ liệu: SMC Data Platform được xây dựng theo khung **ISO 37120** (104 KPI trên 19 chủ đề cho đô thị bền vững, World Council on City Data), **ISO 37122:2019** (chỉ số Smart City), và **ISO 37125:2024** (bộ chỉ số ESG mới nhất cho thành phố). Đây là nền tảng để xác định nhóm câu hỏi nghiệp vụ phù hợp cho chatbot.

---

### 🔴 Must-have — Bắt buộc hoàn thành trong PoC

> Các chức năng này tạo thành **pipeline end-to-end tối thiểu** để chatbot có thể hoạt động và được đánh giá. Thiếu bất kỳ item nào, PoC không thể chạy được.

| ID | Chức năng | Mô tả chi tiết | Lý do Must-have | Tuần hoàn thành |
|----|-----------|----------------|-----------------|-----------------|
| M-01 | Nhận câu hỏi tiếng Việt | Giao diện Web UI (Streamlit/React) hoặc REST API endpoint `/ask` nhận input dạng văn bản tự nhiên tiếng Việt, tối đa 500 ký tự | Đây là điểm đầu vào duy nhất của hệ thống; không có UI/API thì không có gì để test | Tuần 3 |
| M-02 | Sinh SQL từ câu hỏi (Text-to-SQL) | LLM dịch câu hỏi thành truy vấn SQL đúng cú pháp StarRocks (MySQL-compatible). Prompt phải chứa schema context, few-shot examples, và ràng buộc whitelist | Core value proposition của toàn hệ thống. Theo benchmark BIRD (Li et al., 2023), đây là module quyết định >80% chất lượng toàn pipeline | Tuần 3 |
| M-03 | Whitelist bảng/cột | File cấu hình (YAML/JSON) liệt kê danh sách bảng và cột được phép truy vấn. LLM chỉ được sinh SQL trên schema này. Validator từ chối mọi tham chiếu ngoài whitelist | Bảo vệ dữ liệu nhạy cảm và kiểm soát phạm vi PoC. Theo NL2SQL System Design (Mahakali, 2025), schema restriction là yêu cầu bắt buộc cho môi trường enterprise | Tuần 2 |
| M-04 | Validate SQL trước khi thực thi | Kiểm tra: (1) cú pháp SQL hợp lệ (parse AST), (2) chỉ dùng bảng/cột trong whitelist, (3) không chứa lệnh DDL/DML nguy hiểm (DROP, DELETE, UPDATE, INSERT), (4) không có subquery vượt quá độ phức tạp cho phép | Ngăn SQL injection và bảo vệ DB. Theo NL2SQL design pattern (Mahakali, 2025): *"Prevents SQL injection or destructive queries (like DROP, DELETE, etc.)"* | Tuần 3–4 |
| M-05 | Thực thi truy vấn trên StarRocks | Kết nối StarRocks qua MySQL protocol/JDBC, thực thi SQL đã validate, nhận kết quả dạng tabular, xử lý timeout (mặc định 10s), connection pooling | Bằng chứng kết nối thực tế với DB của SMC Platform — yêu cầu cốt lõi của PoC | Tuần 4 |
| M-06 | Trả lời tự nhiên từ kết quả DB | LLM tổng hợp kết quả tabular từ DB thành câu trả lời tiếng Việt, dễ hiểu với người dùng phi kỹ thuật. Bao gồm đơn vị đo, ngữ cảnh và giải thích ngắn nếu cần | Kết quả raw (JSON/table) không có giá trị với người dùng nghiệp vụ. Đây là lớp Answer Synthesis phân biệt chatbot với tool query thông thường | Tuần 3 |
| M-07 | Logging toàn pipeline | Ghi log có cấu trúc (JSON) cho mỗi request: câu hỏi gốc, SQL sinh ra, kết quả DB, câu trả lời cuối, timestamp từng bước (T1/T2/T3), mã lỗi nếu có. Lưu vào file hoặc SQLite | Bắt buộc để đo lường 5 tiêu chí AC3. Không có log, không thể tính SQL Accuracy hay Response Time | Tuần 3–4 |
| M-08 | Xử lý lỗi & fallback | Xử lý ít nhất 6 loại lỗi: SQL syntax error, bảng không tồn tại, DB timeout, kết quả rỗng, câu hỏi ngoài phạm vi, input rỗng. Mỗi loại có thông báo thân thiện bằng tiếng Việt và không để hệ thống crash | Hệ thống không có error handling = không thể demo. Tiêu chí AC3-4 yêu cầu ≥80% error case được xử lý đúng | Tuần 3–4 |
| M-09 | Semantic Layer (Business Term Mapping) | File mapping (YAML/JSON) ánh xạ thuật ngữ nghiệp vụ sang tên kỹ thuật trong DB. Ví dụ: `"chỉ số AQI"` → `air_quality.aqi_index`, `"mức PM2.5"` → `air_quality.pm25_avg`, `"tiêu thụ điện"` → `energy.consumption_kwh`. Mapping được inject vào prompt trước khi gọi LLM | Theo nghiên cứu của Denodo Platform (2024): *"Additional semantics can have a very significant impact on text-to-SQL accuracy."* dbt Labs (2026) xác nhận semantic layer giúp accuracy tiệm cận 100% cho các query được định nghĩa. Đây là cải tiến chính của tuần 5 | Tuần 5 |
| M-10 | System prompt chuẩn hóa cho Smart City domain | Prompt template được thiết kế riêng cho SMC Data Platform, bao gồm: mô tả domain (Smart City / ESG), schema context, mapping ISO indicators, few-shot examples với câu hỏi tiếng Việt, ràng buộc output format | Theo thiết kế NL2SQL (Mahakali, 2025): prompt builder phải *"enrich with relevant database schema, sample SQL examples, user permissions or roles"*. System prompt yếu là nguyên nhân hàng đầu gây hallucination | Tuần 5 |
| M-11 | Báo cáo so sánh trước/sau cải tiến | Document so sánh kết quả chạy 30 câu hỏi test ở tuần 4 (baseline) vs tuần 5 (sau semantic layer + system prompt v2): SQL Accuracy, Answer Quality, Response Time. Kèm phân tích nguyên nhân cải thiện/suy giảm | Yêu cầu bắt buộc của PO/BA để đánh giá giá trị của semantic layer. Là output chứng minh PoC có giá trị tiếp tục đầu tư | Tuần 5 |
| M-12 | Lịch sử hội thoại (multi-turn context) | Lưu tối thiểu 5 lượt hội thoại gần nhất trong session, inject vào prompt để LLM hiểu ngữ cảnh câu trước. Ví dụ: sau câu hỏi về PM2.5 ở Hà Nội, câu tiếp theo "Còn quý trước thì sao?" phải hiểu đúng ngữ cảnh | Theo benchmark SParC (extending Spider with conversational context), multi-turn là yêu cầu thiết yếu cho chatbot thực tế, không phải UX nâng cao | Tuần 4 |
| M-13 | Hiển thị SQL sinh ra (Transparency mode) | Cho phép người dùng bật/tắt chế độ xem câu SQL được sinh ra phía dưới câu trả lời. Khi bật: hiển thị SQL đã format, tên bảng và kết quả raw (số dòng, thời gian thực thi) | Tăng tính minh bạch và tin cậy — yếu tố quan trọng với người dùng kỹ thuật trong SMC Platform. Cũng cần thiết cho nhóm đánh giá SQL Accuracy trong AC3 | Tuần 3–4 |
| M-14 | Phân quyền dữ liệu theo role | Whitelist phân tầng theo role: `viewer` chỉ xem aggregated data, `analyst` xem chi tiết theo district, `admin` xem toàn bộ. Role được truyền vào API header, validator áp dụng whitelist tương ứng | ISO 37120 bao gồm nhiều chỉ số nhạy cảm (tài chính công, an ninh). Phân quyền là yêu cầu bảo mật tối thiểu để deploy trong môi trường Smart City thực tế | Tuần 4 |
| M-15 | Hỗ trợ đa ngôn ngữ (Tiếng Việt & Tiếng Anh) | Pipeline xử lý được câu hỏi bằng cả tiếng Việt lẫn tiếng Anh. LLM tự detect ngôn ngữ và trả lời cùng ngôn ngữ với câu hỏi. SQL vẫn là English (không đổi) | SMC Platform phục vụ cả chuyên gia quốc tế. Hầu hết LLM hiện đại (GPT-4o, Gemini) xử lý tốt cả hai ngôn ngữ mà không cần code thêm — chi phí implement thấp | Tuần 3 |

---

### 🟡 Should-have — Nên có, ưu tiên nếu hoàn thành Must-have trước tuần 5

> Các chức năng này nâng cao đáng kể chất lượng sản phẩm nhưng không block việc demo và đánh giá PoC.

| ID | Chức năng | Mô tả chi tiết | Lý do Should-have | Tuần dự kiến |
|----|-----------|----------------|-------------------|--------------|
| S-01 | Giao diện quản trị whitelist (Admin UI) | Web UI đơn giản cho phép admin thêm/xóa/sửa bảng và cột trong whitelist mà không cần sửa file cấu hình thủ công. Thay đổi có hiệu lực ngay (hot reload) | Cần thiết cho vận hành thực tế khi schema DB thay đổi, nhưng trong PoC nhóm dev có thể sửa file YAML trực tiếp | Tuần 5–6 |
| S-02 | Caching kết quả truy vấn | Cache cặp (câu hỏi → kết quả) trong bộ nhớ hoặc Redis với TTL 1 giờ. Câu hỏi giống nhau trong session không gọi lại LLM và DB. Cache invalidation khi data cập nhật | Giảm latency và chi phí API call với câu hỏi lặp lại thường gặp (dashboard-style queries). Không cần thiết cho 30 câu hỏi test trong PoC | Tuần 5–6 |
| S-03 | Phát hiện câu hỏi ngoài phạm vi | Classifier nhẹ (rule-based hoặc LLM zero-shot) phát hiện câu hỏi không liên quan đến dữ liệu SMC Platform (thời tiết, tin tức, câu hỏi cá nhân...) trước khi gọi Text-to-SQL pipeline | Tránh hệ thống sinh SQL vô nghĩa hoặc trả lời sai. Nhưng trong PoC, lỗi này đã được xử lý ở M-08 ở mức cơ bản | Tuần 5 |
| S-04 | Gợi ý câu hỏi liên quan | Sau mỗi câu trả lời, hệ thống đề xuất 2–3 câu hỏi tiếp theo liên quan (generated bởi LLM dựa trên context). Hiển thị dạng chip/button để click nhanh | Cải thiện UX đáng kể cho người dùng không biết nên hỏi gì tiếp theo, đặc biệt với domain ISO Smart City phức tạp | Tuần 6 |
| S-05 | Export kết quả truy vấn | Nút download kết quả dạng CSV hoặc Excel. Bao gồm tên cột, dữ liệu và metadata (câu hỏi gốc, timestamp, tên bảng nguồn) | Người dùng nghiệp vụ thường cần đưa dữ liệu vào báo cáo. Không cần thiết cho việc đánh giá PoC | Tuần 6 |

---

### 🟢 Could-have — Có thể có nếu PoC vượt tiến độ kế hoạch

> Các chức năng này có giá trị rõ ràng nhưng không phù hợp với phạm vi 6 tuần PoC. Được đưa vào Roadmap sau PoC.

| ID | Chức năng | Mô tả chi tiết | Lý do Could-have | Roadmap |
|----|-----------|----------------|-----------------|---------|
| C-01 | Monitoring & observability dashboard | Dashboard (Grafana/Metabase) theo dõi real-time: số lượt hỏi, tỷ lệ SQL error, phân phối latency (p50/p90/p99), top câu hỏi phổ biến, tỷ lệ fallback | Cần thiết cho production nhưng quá tốn thời gian setup trong PoC 6 tuần | Giai đoạn production |
| C-02 | Đánh giá câu trả lời (Thumbs up/down) | Người dùng có thể đánh giá câu trả lời tốt/xấu. Feedback được lưu kèm câu hỏi và SQL để làm dữ liệu cải tiến model sau này | Thu thập human feedback quan trọng cho RLHF hoặc fine-tuning sau này, nhưng không ảnh hưởng đến chất lượng PoC | Giai đoạn production |
| C-03 | Alert & thông báo KPI vượt ngưỡng | Hệ thống tự động detect và gửi alert (email/Slack) khi chỉ số KPI trong DB vượt ngưỡng định sẵn (AQI > 150, tiêu thụ điện tăng >20%...) | Tăng giá trị vận hành đáng kể nhưng đây là tính năng chủ động (push), khác bản chất so với chatbot thụ động (pull) của PoC | Giai đoạn production |
| C-04 | Trực quan hóa kết quả bằng biểu đồ | Tự động sinh biểu đồ (line chart, bar chart, heatmap) từ kết quả truy vấn dạng chuỗi thời gian hoặc so sánh nhiều chiều | Cải thiện UX đáng kể nhưng yêu cầu thêm logic phân loại loại câu hỏi và thư viện chart. Phù hợp giai đoạn v2 | Sau PoC — v2 |
| C-05 | Tích hợp SSO / xác thực tổ chức | Đăng nhập qua hệ thống SSO nội bộ (LDAP, OAuth2, SAML). Tự động map user → role → whitelist tương ứng | Bắt buộc cho production nhưng cần phối hợp với team IT infrastructure của tổ chức, nằm ngoài phạm vi PoC | Giai đoạn production |

---

### Tóm tắt phân loại backlog

| Phân loại | Số items | Tuần hoàn thành | Mục tiêu |
|-----------|----------|-----------------|----------|
| 🔴 Must-have | 15 | Tuần 2–5 | Demo & đánh giá PoC đầy đủ |
| 🟡 Should-have | 5 | Tuần 5–6 | Nâng chất lượng nếu còn thời gian |
| 🟢 Could-have | 5 | Sau PoC | Roadmap production |

**Tài liệu tham khảo AC1:**
- Liu, M. & Xu, J. (2025). *NLI4DB: A Systematic Review of Natural Language Interfaces for Databases*. arXiv:2503.02435. https://arxiv.org/html/2503.02435v1
- Mahakali, A. (2025). *NL2SQL System Design Guide 2025*. Medium. https://medium.com/@adityamahakali/nl2sql-system-design-guide-2025-c517a00ae34d
- Getwren.ai (2024). *How we design our semantic engine for LLMs*. https://www.getwren.ai/post/how-we-design-our-semantic-engine-for-llms-the-backbone-of-the-semantic-layer-for-llm-architecture
- Denodo Platform (2024). *Improving the Accuracy of LLM-Based Text-to-SQL Generation with a Semantic Layer*. https://www.datamanagementblog.com/improving-the-accuracy-of-llm-based-text-to-sql-generation-with-a-semantic-layer-in-the-denodo-platform/
- dbt Labs (2026). *Semantic Layer vs. Text-to-SQL: 2026 Benchmark Update*. https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026
- World Council on City Data. *ISO 37120 — 104 KPIs across 19 themes*. https://www.dataforcities.org/iso-37120
- ISO 37122:2019. *Indicators for Smart Cities*. https://www.iso.org/standard/69050.html
- ISO 37125:2024. *ESG Indicators for Cities*. https://www.iso.org/standard/85198.html

---

## AC2 — Kế hoạch triển khai 6 tuần & Phân công vai trò

### Vai trò hai intern

| Intern | Vai trò chính | Trách nhiệm |
|--------|---------------|-------------|
| **Nguyễn Hoàng Khải Minh** | **AI/LLM Engineer** | Thiết kế prompt, Text-to-SQL pipeline, semantic layer, đánh giá chất lượng câu trả lời |
| **Nguyễn Quốc Nam** | **Backend/Data Engineer** | Kết nối StarRocks, API backend, validate SQL, logging, tài liệu kỹ thuật |

> Cả hai cùng tham gia: phân tích nghiệp vụ (tuần 1–2), kiểm thử bộ 30 câu hỏi, chuẩn bị demo (tuần 6).

---

### Giai đoạn 1: Nền tảng & Kiến trúc (Tuần 1–2)

#### Tuần 1 — Phân tích nghiệp vụ & Khảo sát dữ liệu

| # | Công việc | Người thực hiện | Output |
|---|-----------|-----------------|--------|
| 1.1 | Phân tích nghiệp vụ theo khung ISO Smart City Indicators | Cả hai | Danh sách nhóm câu hỏi theo domain |
| 1.2 | Khảo sát schema StarRocks: bảng, cột, kiểu dữ liệu, quan hệ | Nam | Schema document + whitelist draft |
| 1.3 | Xây dựng bộ 30 câu hỏi test (theo 6 nhóm KPI/ESG) | Minh | File `test_questions_v1.xlsx` |
| 1.4 | Nghiên cứu LLM phù hợp (GPT-4o / Gemini / local model) | Minh | Báo cáo so sánh model ngắn |
| 1.5 | Xác định stack kỹ thuật & môi trường phát triển | Cả hai | Tech stack document |

#### Tuần 2 — Thiết kế kiến trúc & Đặc tả chức năng

| # | Công việc | Người thực hiện | Output |
|---|-----------|-----------------|--------|
| 2.1 | Thiết kế sơ đồ kiến trúc tổng thể (end-to-end) | Minh | Architecture diagram (draw.io / Mermaid) |
| 2.2 | Đặc tả API (endpoint, request/response schema) | Nam | API spec (OpenAPI/Swagger draft) |
| 2.3 | Thiết kế cơ chế whitelist bảng/cột | Nam | Whitelist config file (JSON/YAML) |
| 2.4 | Thiết kế prompt template ban đầu cho Text-to-SQL | Minh | Prompt v1 document |
| 2.5 | Kế hoạch kiểm thử chi tiết & định nghĩa tiêu chí đánh giá | Cả hai | Test plan document |

---

### Giai đoạn 2: Xây dựng PoC (Tuần 3–4)

#### Tuần 3 — Phát triển core (Frontend + Text-to-SQL)

| # | Công việc | Người thực hiện | Output |
|---|-----------|-----------------|--------|
| 3.1 | Phát triển Web UI chatbot đơn giản (React/Streamlit) | Minh | Giao diện nhận câu hỏi & hiển thị trả lời |
| 3.2 | Tích hợp LLM API + prompt Text-to-SQL | Minh | Pipeline câu hỏi → SQL hoạt động |
| 3.3 | Xây dựng backend FastAPI với endpoint `/ask` | Nam | API endpoint nhận câu hỏi |
| 3.4 | Implement whitelist validator | Nam | Module validate bảng/cột |
| 3.5 | Unit test Text-to-SQL với 10 câu hỏi mẫu | Cả hai | Kết quả test tuần 3 |

#### Tuần 4 — Kết nối DB + Validate + Logging

| # | Công việc | Người thực hiện | Output |
|---|-----------|-----------------|--------|
| 4.1 | Kết nối StarRocks (JDBC/MySQL protocol) | Nam | Connection pool hoạt động |
| 4.2 | Thực thi SQL & trả kết quả về API | Nam | End-to-end flow hoạt động |
| 4.3 | Implement SQL validator nâng cao (parse AST) | Nam | Validator từ chối SQL nguy hiểm |
| 4.4 | Hệ thống logging (câu hỏi, SQL, kết quả, latency, lỗi) | Nam | Log file / database |
| 4.5 | Chạy toàn bộ 30 câu hỏi test — baseline evaluation | Cả hai | Baseline report (SQL accuracy, answer quality) |

---

### Giai đoạn 3: Cải tiến & Hoàn thiện (Tuần 5–6)

#### Tuần 5 — Cải tiến chất lượng & Semantic Layer

| # | Công việc | Người thực hiện | Output |
|---|-----------|-----------------|--------|
| 5.1 | Xây dựng system prompt chuẩn hóa cho Smart City domain | Minh | System prompt v2 (tối ưu) |
| 5.2 | Xây dựng semantic layer (business term mapping) | Minh + Nam | Semantic mapping file |
| 5.3 | Tích hợp semantic layer vào pipeline | Minh | Pipeline v2 với semantic layer |
| 5.4 | Chạy lại 30 câu hỏi test — post-improvement evaluation | Cả hai | Improvement report (so sánh trước/sau) |
| 5.5 | Xử lý các edge case & lỗi thường gặp | Cả hai | Danh sách edge cases & cách xử lý |

#### Tuần 6 — Tài liệu & Demo

| # | Công việc | Người thực hiện | Output |
|---|-----------|-----------------|--------|
| 6.1 | Viết tài liệu kiến trúc & hướng dẫn cài đặt/chạy PoC | Nam | `ARCHITECTURE.md` + `SETUP.md` |
| 6.2 | Viết tài liệu API (Swagger/Postman collection) | Nam | `API_DOCS.md` / Swagger UI |
| 6.3 | Đề xuất Roadmap sau PoC | Minh | `ROADMAP.md` |
| 6.4 | Chuẩn bị slide trình bày (bài toán, kiến trúc, kết quả, hạn chế) | Cả hai | Slide deck (8–12 slides) |
| 6.5 | Demo script & rehearsal | Cả hai | Demo script + buổi chạy thử |

---

### Tóm tắt tiến độ theo tuần

```
Tuần 1: [Phân tích nghiệp vụ] [Khảo sát schema] [Bộ 30 câu hỏi]
Tuần 2: [Kiến trúc] [Đặc tả API] [Prompt v1] [Test plan]
Tuần 3: [Web UI] [Text-to-SQL pipeline] [API endpoint] [Whitelist]
Tuần 4: [Kết nối StarRocks] [Logging] [BASELINE EVALUATION]
Tuần 5: [Semantic layer] [Prompt v2] [POST-IMPROVEMENT EVALUATION]
Tuần 6: [Tài liệu] [API docs] [Roadmap] [DEMO]
```

---

## AC3 — Bộ tiêu chí đánh giá PoC

### Bối cảnh & Cơ sở lý luận

Đánh giá hệ thống Text-to-SQL chatbot đòi hỏi phối hợp nhiều loại metric vì pipeline gồm nhiều bước: **câu hỏi → SQL → kết quả DB → câu trả lời tự nhiên**. Lỗi có thể xảy ra ở bất kỳ bước nào và ảnh hưởng khác nhau đến trải nghiệm cuối cùng.

Hai benchmark học thuật phổ biến nhất trong lĩnh vực này là:
- **Spider** (Yu et al., 2018): 10,181 câu hỏi trên 200 DB, 138 domain. Dùng hai metric chính: **Exact Match (EM)** và **Execution Accuracy (EX)**. Các hệ thống tốt nhất hiện nay đạt ~83% EX trên Spider (Promethium, 2025).
- **BIRD** (Li et al., 2023): 12,751 cặp Text-SQL trên 95 DB quy mô lớn (~33.4 GB). Bổ sung thêm metric **Valid Efficiency Score (VES)** để đo hiệu quả truy vấn, không chỉ độ đúng. Theo BIRD-Ent benchmark (OpenReview, 2025), hệ thống enterprise thực tế chỉ đạt 39.1 EX — cho thấy khoảng cách lớn giữa nghiên cứu và thực tế.

Bên cạnh đó, để đánh giá chất lượng câu trả lời tự nhiên (Answer Quality), framework **RAGAS** (Es et al., 2023) cung cấp bộ metric tiêu chuẩn: **Faithfulness**, **Answer Relevancy**, **Context Recall**, **Context Precision**. Theo nghiên cứu của MDPI Electronics (2025), RAGAS phù hợp hơn các metric NLP truyền thống (BLEU, ROUGE) vì đánh giá được sự liên kết ngữ nghĩa với context, không chỉ lexical overlap.

Bộ 5 tiêu chí dưới đây được thiết kế để **phù hợp với quy mô PoC 6 tuần** (đo thủ công, không cần infrastructure phức tạp) trong khi vẫn bám sát các metric học thuật chuẩn quốc tế.

---

### Tổng quan phương pháp đánh giá

Đánh giá được thực hiện **2 lần** trên cùng bộ 30 câu hỏi test:

| Đợt | Thời điểm | Mục tiêu |
|-----|-----------|---------|
| **Baseline Evaluation** | Cuối tuần 4 | Đo hiệu suất hệ thống cơ bản (chưa có semantic layer, prompt v1) |
| **Post-improvement Evaluation** | Cuối tuần 5 | Đo sau khi thêm semantic layer + system prompt v2. So sánh delta |

Kết quả được ghi vào **Improvement Report** (output tuần 5, M-11).

---

### Tiêu chí 1 — Độ đúng SQL (SQL Accuracy)

**Định nghĩa:** Tỷ lệ câu SQL được sinh ra cho kết quả đúng khi thực thi trên StarRocks, so với ground truth.

**Cơ sở khoa học:** Trong cộng đồng nghiên cứu, metric chuẩn là **Execution Accuracy (EX)** — chạy cả SQL dự đoán và SQL ground truth, so sánh kết quả trả về. Theo Promethium (2025) và CallSphere (2026): *"Execution Accuracy (EX) runs both the predicted and reference SQL against the database and compares results. This is the standard metric because it correctly handles multiple valid SQL formulations."* Điều này quan trọng vì nhiều câu SQL khác nhau về cú pháp nhưng cho kết quả đúng đều nên được tính là đúng — metric Exact Match (EM) sẽ bỏ sót những trường hợp này.

Ngoài ra, BIRD benchmark (Li et al., 2023) bổ sung **Valid Efficiency Score (VES)** — đánh giá hiệu quả thực thi (query time) bên cạnh độ đúng. Trong PoC này, VES được đo gián tiếp qua Tiêu chí 3 (Response Time).

**Thang đánh giá (theo từng câu hỏi):**

Metric chính được dùng để tính điểm là **Execution Accuracy (EX)**. Các metric tham chiếu còn lại (SM, CM, EM, VES, QVT) được liệt kê để đối chiếu với chuẩn học thuật, nhưng không tính vào điểm PoC do yêu cầu công cụ đo phức tạp hơn.

| Metric / Mức Đánh Giá | Điểm | Loại Metric | Mô tả Chi Tiết |
|---|---|---|---|
| **Execution Match** | 1.0 | Execution Accuracy (EX) — **Dùng để tính điểm** | SQL thực thi thành công, kết quả khớp hoàn toàn với ground truth (giá trị, số dòng, thứ tự nếu có ORDER BY) |
| **Partial Execution Match** | 0.5 | Execution Accuracy (EX) — **Dùng để tính điểm** | SQL thực thi thành công, đúng bảng/metric chính nhưng sai điều kiện lọc nhỏ (sai tháng, sai threshold, sai khoảng thời gian) |
| **Syntax / Runtime Error** | 0.0 | Execution Accuracy (EX) — **Dùng để tính điểm** | SQL sai cú pháp, tham chiếu bảng/cột không tồn tại, hoặc gặp exception khi thực thi |
| **Invalid / Refused** | N/A | Execution Accuracy (EX) | Hệ thống từ chối sinh SQL (câu hỏi ngoài phạm vi) — không tính vào accuracy |
| **String-Match Accuracy (SM)** | — | String Comparison — *Tham chiếu* | So sánh chuỗi SQL ground truth và SQL dự đoán có giống hệt nhau không (dễ phạt nặng dù kết quả thực thi đúng) |
| **Component-Match Accuracy (CM)** | — | Component Level — *Tham chiếu* | So sánh chi tiết từng thành phần SQL (SELECT, WHERE, GROUP BY, ORDER BY, v.v.) |
| **Exact-Match Accuracy (EM)** | — | Component Level — *Tham chiếu* | Yêu cầu tất cả thành phần SQL phải khớp hoàn toàn với ground truth (dựa trên CM) |
| **Valid Efficiency Score (VES)** | — | Efficiency + Accuracy — *Tham chiếu* | Đánh giá cả độ đúng và hiệu suất thực thi (thời gian chạy query); đo gián tiếp qua Tiêu chí 3 |
| **Query Variance Testing (QVT)** | — | Robustness — *Tham chiếu* | Đánh giá độ robust của hệ thống khi câu hỏi tự nhiên thay đổi cách diễn đạt |

**Công thức:**
```
SQL Execution Accuracy (EX) = Σ điểm từng câu / (Tổng số câu có SQL) × 100%
```

**Ngưỡng chấp nhận:**

| Đợt | Ngưỡng EX | Mức kỳ vọng |
|-----|-----------|-------------|
| Baseline (tuần 4) | ≥ 60% | Khả thi với LLM tốt + prompt cơ bản |
| Post-improvement (tuần 5) | ≥ 80% | Sau semantic layer + prompt v2. BIRD-Ent baseline của enterprise là 39.1 EX, nên 80% là mục tiêu tham vọng nhưng hợp lý với domain hẹp |

**Cách đo:** Người đánh giá (Mentor hoặc cả hai intern) chạy thủ công 30 câu hỏi, ghi SQL sinh ra, so sánh với ground truth SQL đã chuẩn bị trong AC4. Dùng log từ M-07 để truy xuất SQL tự động.

**Phân tích lỗi:** Với mỗi câu sai, ghi nhận loại lỗi: (a) sai schema linking, (b) sai aggregation, (c) sai filter condition, (d) missing JOIN, (e) syntax error thuần túy → dùng để prioritize cải tiến.

---

### Tiêu chí 2 — Độ đúng câu trả lời (Answer Quality)

**Định nghĩa:**  
Mức độ câu trả lời tự nhiên cuối cùng của chatbot phản ánh **đúng, đầy đủ, trung thực, liên quan** và **tương đồng** với dữ liệu trả về từ cơ sở dữ liệu cũng như ý định của người dùng.

**Cơ sở khoa học:**  
Framework **RAGAS** (Es et al., 2023) cung cấp bộ metrics chuẩn để đánh giá chất lượng câu trả lời trong các hệ thống dựa trên LLM. PoC này sử dụng 4 metrics chính: **Faithfulness**, **Answer Relevancy**, **Answer Correctness** và **Answer Similarity**. 

Theo Confident AI (2025): *"No single metric tells the whole story."* Việc kết hợp nhiều metrics giúp đánh giá toàn diện hơn.

**Bốn metrics đánh giá:**

| Metric                  | Ký hiệu | Trọng số | Định nghĩa |
|-------------------------|---------|----------|----------|
| **Faithfulness**        | F       | 30%      | Câu trả lời có trung thực với dữ liệu từ DB hay không (không hallucinate số liệu, đơn vị, tên chỉ số). |
| **Answer Relevancy**    | R       | 25%      | Câu trả lời có đúng trọng tâm, liên quan trực tiếp đến câu hỏi và không lan man. |
| **Answer Correctness**  | AC      | 25%      | Câu trả lời có chính xác về mặt sự kiện so với ground truth. |
| **Answer Similarity**   | AS      | 20%      | Mức độ tương đồng ngữ nghĩa giữa câu trả lời sinh ra và câu trả lời mong đợi. |

**Thang điểm chi tiết (1–5):**

| Điểm | Faithfulness                                      | Answer Relevancy                                      | Answer Correctness                                   | Answer Similarity                                    |
|------|---------------------------------------------------|-------------------------------------------------------|------------------------------------------------------|------------------------------------------------------|
| **5** | Hoàn toàn trung thực, không hallucinate           | Rất liên quan, tập trung cao vào câu hỏi             | Hoàn toàn chính xác với ground truth                 | Tương đồng ngữ nghĩa gần như tuyệt đối              |
| **4** | Trung thực cao, chỉ sai sót nhỏ về format         | Liên quan tốt, thiếu rất ít thông tin phụ            | Chính xác phần lớn, sai sót nhỏ                      | Tương đồng cao về ý nghĩa                            |
| **3** | Có một số sai sót nhỏ hoặc thiếu dẫn chứng        | Đúng chủ đề nhưng hơi chung chung                    | Đúng một phần, có lỗi trung bình                     | Tương đồng trung bình                                |
| **2** | Có hallucinate hoặc sai lệch đáng kể              | Lạc đề một phần                                       | Sai khá nhiều so với ground truth                    | Tương đồng thấp                                      |
| **1** | Hallucinate nghiêm trọng, mâu thuẫn với DB       | Hoàn toàn không liên quan                             | Sai hoàn toàn hoặc bịa đặt                           | Gần như không tương đồng                             |
|**Ngưỡng chấp nhận:**| 3| 3| 3 | 3|

| Đợt | Ngưỡng Weighted Score | Ghi chú |
|-----|----------------------|---------|
| Baseline (tuần 4) | ≥ 3.0 / 5 | Peer review bởi 2 người đánh giá, lấy trung bình |
| Post-improvement (tuần 5) | ≥ 3.8 / 5 | Sau semantic layer + prompt v2 |

**Cách đo:** Hai người đánh giá độc lập chấm điểm từng câu trả lời theo 3 chiều trên thang 1–5, sau đó tính weighted score. Lấy trung bình của hai người. Trường hợp lệch ≥ 1.0 điểm ở bất kỳ chiều nào → thảo luận để thống nhất.

---

### Tiêu chí 3 — Thời gian phản hồi (Response Time / Latency)

**Định nghĩa:** Thời gian từ khi người dùng gửi câu hỏi đến khi chatbot trả về câu trả lời hoàn chỉnh (end-to-end latency).

**Cơ sở khoa học:** BIRD benchmark giới thiệu **Valid Efficiency Score (VES)** để đo hiệu quả SQL song song với độ đúng, vì trong môi trường thực tế, một câu truy vấn đúng nhưng chạy 30 giây là không chấp nhận được. Theo VentureBeat (2025), semantic layer native (platform-native architecture) có ưu điểm *"zero-copy access, removing the need for a separate server"* — giảm latency đáng kể so với text-to-SQL thuần. Với SMC PoC dùng external LLM API, latency chủ yếu đến từ T1 (LLM call #1) và T3 (LLM call #2).

**Phân rã thành phần:**

| Thành phần | Ký hiệu | Mô tả | Benchmark kỳ vọng |
|------------|---------|-------|-------------------|
| LLM Inference — Text-to-SQL | T1 | Thời gian gọi LLM API, nhận SQL | 2–5s (GPT-4o) |
| StarRocks Execution | T2 | Thời gian DB thực thi SQL và trả kết quả | < 2s (query đơn giản–trung bình) |
| LLM Inference — Answer Synthesis | T3 | Thời gian gọi LLM API lần 2, nhận câu trả lời tự nhiên | 1–3s |
| **Tổng End-to-End** | **T = T1+T2+T3** | Thời gian người dùng cảm nhận | **< 10s target** |

**Ngưỡng đánh giá:**

| Mức | Thời gian (p90) | Đánh giá | Hành động |
|-----|-----------------|----------|-----------|
| ✅ Xuất sắc | < 5 giây | Tốt cho production | Giữ nguyên |
| ✅ Đạt | 5–10 giây | Chấp nhận được cho PoC | Document hạn chế |
| ⚠️ Cần cải thiện | 10–20 giây | Cần tối ưu prompt/query | Ghi vào roadmap |
| ❌ Không đạt | > 20 giây | Không thể demo | Điều tra ngay |

**Cách đo:**
- Log timestamp tại 4 điểm: `t_start` (nhận request), `t_sql` (nhận SQL từ LLM), `t_db` (nhận kết quả DB), `t_end` (gửi câu trả lời).
- Tính T1 = t_sql − t_start, T2 = t_db − t_sql, T3 = t_end − t_db, T = t_end − t_start.
- Chạy 30 câu hỏi test, lấy **p50** (median) và **p90** (90th percentile). P90 là số dùng để so sánh với ngưỡng.
- Phân tích phân phối theo độ phức tạp SQL (đơn giản / trung bình / phức tạp).

**Ngưỡng chấp nhận:**

| Đợt | p90 End-to-End | p90 T2 (StarRocks) |
|-----|---------------|-------------------|
| Baseline (tuần 4) | < 15 giây | < 3 giây |
| Post-improvement (tuần 5) | < 10 giây | < 2 giây |

---

### Tiêu chí 4 — Khả năng xử lý lỗi (Error Handling & Robustness)

**Định nghĩa:** Hệ thống phản ứng đúng, thân thiện và không crash với các tình huống đầu vào không hợp lệ hoặc lỗi nội bộ.

**Cơ sở khoa học:** Theo **Dr.Spider benchmark** (Chang et al., 2023), ngay cả các model tốt nhất cũng bị suy giảm hiệu suất **14–50%** khi gặp perturbation trong câu hỏi. Điều này cho thấy robustness là chiều đánh giá độc lập, không thể suy ra từ accuracy trên câu hỏi chuẩn.

Theo thiết kế **PHOTON system** (Zeng et al., 2020): *"The question corrector detects the untranslatable questions from user input, scans the confusion spans that need clarification or correction"* — xử lý câu hỏi ngoài phạm vi là thành phần thiết kế quan trọng.

**Ma trận test case lỗi:**

| Loại lỗi | ID | Câu hỏi test | Kết quả đúng | Kết quả sai (cần tránh) |
|---|---|---|---|---|
| Câu hỏi ngoài domain | E-01 | "Thời tiết Hà Nội hôm nay thế nào?" | Thông báo lịch sự, không sinh SQL | Sinh SQL sai hoặc trả lời bịa |
| Câu hỏi mơ hồ, thiếu ngữ cảnh | E-02 | "Cho tôi xem số liệu mới nhất" | Hỏi lại: mới nhất về cái gì? Khu vực nào? | Sinh SQL tùy tiện |
| Bảng/cột ngoài whitelist | E-03 | Câu hỏi tham chiếu bảng nhạy cảm | Từ chối, thông báo không có quyền | Thực thi và trả dữ liệu nhạy cảm |
| SQL syntax error từ LLM | E-04 | Câu hỏi phức tạp khiến LLM tạo SQL sai | Thông báo lỗi rõ ràng, gợi ý rephrasing | Crash hoặc hiện raw exception |
| DB timeout | E-05 | Query nặng vượt timeout 10s | Thông báo timeout, ghi log | Hang vô thời hạn |
| DB không khả dụng | E-06 | StarRocks ngừng hoạt động | Thông báo hệ thống đang bảo trì | Exception 500 không xử lý |
| Input rỗng | E-07 | Gửi string rỗng `""` | Yêu cầu nhập câu hỏi | Gọi LLM với prompt rỗng |
| Input quá dài | E-08 | Input > 1000 ký tự | Cắt bớt hoặc thông báo giới hạn | Gọi LLM, tốn token lãng phí |
| SQL injection attempt | E-09 | `'; DROP TABLE air_quality; --` | Validator từ chối | Thực thi câu SQL nguy hiểm |
| Kết quả DB rỗng | E-10 | Hỏi về dữ liệu chưa có trong DB | "Chưa có dữ liệu cho khoảng thời gian này" | "Kết quả là 0" (misleading) |

**Rubric chấm điểm từng test case:**

| Điểm | Điều kiện |
|---|---|
| 1.0 | Xử lý hoàn toàn đúng: đúng thông báo, không crash, ghi log |
| 0.5 | Xử lý được (không crash) nhưng thông báo không rõ ràng hoặc thiếu log |
| 0.0 | Crash, exception không xử lý, hoặc trả kết quả sai |

**Công thức:**
```
Error Handling Score = Σ điểm 10 test case / 10 × 100%
```

**Ngưỡng chấp nhận:**

| Đợt | Ngưỡng | Ghi chú |
|---|---|---|
| Baseline (tuần 4) | ≥ 60% (6/10) | Các lỗi cơ bản được xử lý |
| Post-improvement (tuần 5) | ≥ 80% (8/10) | Thêm SQL injection, timeout handling |

---

### Taxonomy lỗi NL2SQL

#### Nguyên tắc thiết kế taxonomy

Taxonomy được xây dựng theo bốn nguyên tắc cốt lõi:

- **Comprehensiveness** — Bao quát toàn bộ các lỗi có thể xảy ra trong quá trình chuyển đổi NL2SQL.
- **Mutual Exclusivity** — Mỗi loại lỗi được định nghĩa rõ ràng, không chồng chéo, tránh mơ hồ trong phân loại.
- **Extensibility** — Có khả năng mở rộng để tích hợp các loại lỗi mới khi công nghệ NL2SQL phát triển.
- **Practicality** — Ứng dụng được trong thực tế, hỗ trợ lập trình viên chẩn đoán và sửa lỗi hiệu quả.

#### Cấp độ 1 — Error Localization (Định vị lỗi trong SQL)

Cấp độ này xác định **vị trí cụ thể trong câu SQL** nơi lỗi xảy ra, giúp thu hẹp phạm vi cần kiểm tra và sửa chữa.

| Vị trí lỗi | Mô tả | Ví dụ lỗi điển hình |
|---|---|---|
| `SELECT` clause | Sai cột được chọn hoặc thiếu cột cần thiết | Chọn `temperature` thay vì `pm25` |
| `FROM` / `JOIN` clause | Sai bảng nguồn hoặc thiếu JOIN cần thiết | Bỏ sót bảng `station_info` khi cần tên trạm |
| `WHERE` clause | Sai điều kiện lọc — sai giá trị hoặc sai logic | `WHERE city = 'HCM'` thay vì `WHERE city_code = 'SGN'` |
| `GROUP BY` clause | Thiếu hoặc sai trường nhóm | Quên group theo `date` khi tính trung bình theo ngày |
| `ORDER BY` clause | Sai chiều sắp xếp hoặc sai trường | `ORDER BY station_id` thay vì `ORDER BY aqi DESC` |
| `HAVING` clause | Sai điều kiện lọc sau aggregation | `HAVING COUNT(*) > 10` thay vì `HAVING AVG(aqi) > 150` |
| Aggregation functions | Dùng sai hàm tổng hợp | `SUM(aqi)` thay vì `AVG(aqi)` |
| Subquery / CTE | Lỗi trong truy vấn lồng nhau | Subquery trả về nhiều hàng nhưng dùng `=` thay vì `IN` |
| Schema mapping | Sai tên bảng hoặc tên cột | Dùng tên cột không tồn tại trong schema |

#### Cấp độ 2 — Cause of Error (Nguyên nhân gốc rễ)

Cấp độ này xác định **lý do tại sao** model sinh ra SQL sai, cung cấp insight để cải thiện prompt hoặc fine-tuning.

##### 2.1 Lỗi hiểu ngữ nghĩa (Semantic Understanding Errors)

Lỗi phát sinh do model không nắm đúng ý nghĩa của câu hỏi tự nhiên.

| Loại | Mô tả | Ví dụ |
|---|---|---|
| Lexical ambiguity | Từ ngữ đa nghĩa không được giải quyết đúng | "mới nhất" → model dùng `MAX(id)` thay vì `MAX(timestamp)` |
| Scope misinterpretation | Hiểu sai phạm vi của câu hỏi | "tất cả các trạm" → chỉ lấy một tập con |
| Negation errors | Xử lý sai phủ định | "không vượt quá 100" → sinh `> 100` thay vì `<= 100` |
| Comparative errors | Sai khi xử lý so sánh tương đối | "cao hơn trung bình" → không tính subquery AVG |

##### 2.2 Lỗi hiểu giá trị dữ liệu (Value Understanding Errors)

Lỗi liên quan đến việc model không biết hoặc suy luận sai các giá trị cụ thể trong database.

| Loại | Mô tả | Ví dụ |
|---|---|---|
| Entity value mismatch | Sai giá trị thực thể trong điều kiện | `WHERE station = 'Hà Nội'` thay vì `WHERE station_code = 'HN001'` |
| Date/time format error | Sai định dạng hoặc khoảng thời gian | `WHERE date = '2024/01/01'` thay vì `'2024-01-01'` |
| Unit mismatch | Nhầm đơn vị đo lường | Lọc `pm25 > 50` nhưng DB lưu theo µg/m³ × 10 |
| Out-of-vocabulary value | Giá trị không có trong training data | Mã vùng, tên địa danh đặc thù |

##### 2.3 Lỗi logic và cấu trúc (Logical & Structural Errors)

Lỗi trong cách model tổ chức và liên kết các thành phần của câu SQL.

| Loại | Mô tả | Ví dụ |
|---|---|---|
| Join condition error | Sai điều kiện nối bảng | `JOIN ON a.id = b.station_id` thay vì `a.station_code = b.code` |
| Aggregation logic error | Sai thứ tự hoặc phạm vi aggregation | `AVG` tính trên toàn bảng thay vì trong từng nhóm |
| Subquery misuse | Dùng subquery không phù hợp | Dùng scalar subquery khi cần table subquery |
| Missing DISTINCT | Thiếu loại bỏ trùng lặp | Đếm stations bị trùng vì thiếu `DISTINCT` |

##### 2.4 Lỗi schema (Schema Grounding Errors)

Lỗi do model không ánh xạ đúng ngôn ngữ tự nhiên vào schema thực tế của database.

| Loại | Mô tả | Ví dụ |
|---|---|---|
| Table hallucination | Dùng bảng không tồn tại | `FROM air_data` thay vì `FROM air_quality_hourly` |
| Column hallucination | Dùng cột không tồn tại | `SELECT pollution_index` — cột không có trong schema |
| Wrong table selection | Chọn nhầm bảng có cấu trúc tương tự | Dùng bảng `daily` thay vì `hourly` khi cần dữ liệu giờ |
| Relationship misidentification | Không nhận ra mối quan hệ giữa các bảng | Không JOIN đúng bảng dimension |

#### Mapping: Test Case lỗi ↔ Taxonomy

Bảng dưới đây liên kết các test case trong ma trận (E-01 đến E-10) với taxonomy hai cấp để hỗ trợ chẩn đoán có hệ thống.

| Test Case | Error Localization | Cause of Error |
|---|---|---|
| E-01 (Ngoài domain) | N/A — không sinh SQL | Scope misinterpretation |
| E-02 (Mơ hồ) | `WHERE` clause | Lexical ambiguity |
| E-03 (Ngoài whitelist) | `FROM` clause | Table hallucination / Schema grounding |
| E-04 (SQL syntax error) | Toàn bộ câu SQL | Logical & Structural Errors |
| E-05 (DB timeout) | N/A — lỗi hạ tầng | N/A |
| E-06 (DB không khả dụng) | N/A — lỗi hạ tầng | N/A |
| E-07 (Input rỗng) | N/A — lỗi đầu vào | N/A |
| E-08 (Input quá dài) | N/A — lỗi đầu vào | N/A |
| E-09 (SQL injection) | `WHERE` clause | Value Understanding Errors |
| E-10 (Kết quả rỗng) | `WHERE` clause | Entity value mismatch / Date format error |

---

### Tiêu chí 5 — Khả năng mở rộng (Extensibility & Maintainability)

**Định nghĩa:** Mức độ dễ dàng để thêm dữ liệu mới, cải tiến model, hoặc mở rộng chức năng mà không cần refactor toàn bộ hệ thống.

**Cơ sở khoa học:** Theo NL2SQL System Design (Mahakali, 2025), yêu cầu non-functional quan trọng nhất là *"a modular system where we can swap Models or underlying database/Query Engine."* dbt Labs (2026) nhấn mạnh: trong text-to-SQL, *"text-to-SQL is flexible (any question is fair game) but also fragile"* — do đó cần thiết kế abstraction layer để giảm coupling giữa các thành phần. Đây là nền tảng cho việc scale lên production sau PoC.

**Checklist đánh giá định tính — 10 tiêu chí:**

| # | Hạng mục | Câu hỏi kiểm tra | Cách đánh giá | Điểm |
|---|----------|-----------------|---------------|------|
| 1 | Schema extensibility | Thêm 1 bảng mới vào whitelist có cần sửa code Python core không? | Thử thêm bảng test → chỉ sửa YAML = đạt | 0/1 |
| 2 | Prompt extensibility | Thêm domain mới (vd: logistics) có tái dùng được pipeline không? | Review code: system prompt có hardcode domain không? | 0/1 |
| 3 | LLM provider swappability | Đổi từ OpenAI sang Google Gemini có cần sửa >20 dòng code không? | Review code: có abstraction class/interface cho LLM không? | 0/1 |
| 4 | Semantic layer extensibility | Thêm 10 business term mới mà không sửa Python code | Thử thêm vào mapping YAML → reload → test | 0/1 |
| 5 | DB connector swappability | Đổi từ StarRocks sang PostgreSQL có isolate ở 1 module không? | Review code: DB layer có tách biệt với business logic không? | 0/1 |
| 6 | API versioning | API có versioning (/v1/, /v2/) để không break client cũ khi thêm field? | Kiểm tra endpoint definition | 0/1 |
| 7 | Structured logging | Log có format JSON chuẩn, có thể ingest vào ELK/Loki không? | Xem sample log output | 0/1 |
| 8 | Configuration management | Mọi config (API key, DB host, timeout) có qua env variable / config file không? | Không có hardcode trong source code | 0/1 |
| 9 | Test coverage | Có ít nhất unit test cho SQL validator và whitelist checker không? | Kiểm tra thư mục tests/ | 0/1 |
| 10 | README & onboarding | Người mới có thể chạy hệ thống trong < 30 phút theo README không? | Thử bởi người không quen codebase | 0/1 |

**Công thức:**
```
Extensibility Score = (Tổng điểm đạt) / 10 × 100%
```

**Ngưỡng chấp nhận:**

| Đợt | Ngưỡng | Ghi chú |
|-----|--------|---------|
| Baseline (tuần 4) | ≥ 5/10 | Các item cơ bản: env config, JSON log, structured code |
| Post-improvement (tuần 5) | ≥ 7/10 | Thêm abstraction layer, unit test, README hoàn chỉnh |

---

### Bảng tổng hợp tiêu chí đánh giá

| # | Tiêu chí | Metric chuẩn tham chiếu | Đơn vị đo | Baseline (Tuần 4) | Target (Tuần 5) | Phương pháp đo |
|---|----------|------------------------|-----------|-------------------|-----------------|----------------|
| 1 | SQL Accuracy | Execution Accuracy (EX) — Spider/BIRD | % câu đúng | ≥ 60% | ≥ 80% | So sánh thực thi với ground truth |
| 2 | Answer Quality | RAGAS Faithfulness (40%) + Answer Relevancy (35%) + Clarity (25%) | Điểm 1–5 (weighted) | ≥ 3.0 | ≥ 3.8 | Peer review 2 người (weighted) |
| 3 | Response Time | Valid Efficiency Score (VES) — BIRD | Giây (p90) | < 15s | < 10s | Logging timestamp 4 điểm |
| 4 | Error Handling | Robustness — Dr.Spider inspired | % xử lý đúng | ≥ 60% | ≥ 80% | 10 test case thủ công |
| 5 | Extensibility | Modularity checklist | Điểm /10 | ≥ 5/10 | ≥ 7/10 | Checklist review + thực hành |

---

### Score card mẫu (dùng khi đánh giá)

```
=== EVALUATION SCORECARD — [BASELINE / POST-IMPROVEMENT] — Ngày: _____ ===

Người đánh giá: _____________________

[TC1] SQL Accuracy
  Tổng số câu test:        30
  Execution Match (1.0):   ___
  Partial Match (0.5):     ___
  Error/Wrong (0.0):       ___
  EX Score:                ___% (ngưỡng: 60% / 80%)

[TC2] Answer Quality (trung bình 30 câu)
  Faithfulness avg (×0.40):      ___/5
  Answer Relevancy avg (×0.35):  ___/5
  Clarity avg (×0.25):           ___/5
  Weighted Score:                ___/5  (= F×0.40 + R×0.35 + C×0.25)
  Ngưỡng:                        3.0 / 3.8

[TC3] Response Time (p90 trên 30 câu)
  T1 LLM→SQL (p90):       ___s
  T2 StarRocks (p90):      ___s
  T3 LLM→Answer (p90):    ___s
  T End-to-End (p90):      ___s (ngưỡng: 15s / 10s)

[TC4] Error Handling (10 test cases)
  E-01: ___  E-02: ___  E-03: ___  E-04: ___  E-05: ___
  E-06: ___  E-07: ___  E-08: ___  E-09: ___  E-10: ___
  Score: ___/10 = ___% (ngưỡng: 60% / 80%)

[TC5] Extensibility (10 checklist items)
  Items đạt: ___ / 10 = ___% (ngưỡng: 5/10 / 7/10)

=== KẾT LUẬN ===
Số tiêu chí đạt ngưỡng: ___ / 5
Trạng thái PoC: [ ] PASS  [ ] CONDITIONAL PASS  [ ] FAIL
Ghi chú: _____________________
```

---

**Tài liệu tham khảo AC3:**
- Yu, T. et al. (2018). *Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task*. EMNLP 2018. (Spider benchmark — EM & EX metrics)
- Li, J. et al. (2023). *Can LLM Already Serve as A Database Interface? A BIg Bench for Large-Scale Database Grounded Text-to-SQLs (BIRD)*. NeurIPS 2023. (BIRD benchmark — VES metric)
- Chang, S. et al. (2023). *Dr.Spider: A Diagnostic Evaluation Benchmark towards Text-to-SQL Robustness*. arXiv:2301.08881. https://arxiv.org/pdf/2301.08881 (Robustness evaluation)
- Es, S. et al. (2023). *RAGAS: Automated Evaluation of Retrieval Augmented Generation*. (RAGAS — Faithfulness, Answer Relevancy)
- OpenReview (2025). *Text-to-SQL Benchmarks for Enterprise Realities: BIRD-Ent and Spider-Ent*. https://openreview.net/forum?id=gXkIkSN2Ha (Enterprise gap: 39.1 EX)
- Promethium (2025). *Text-to-SQL Leaderboard & Evaluation Metrics Guide*. https://promethium.ai/guides/text-to-sql-evaluation-benchmarks-metrics/
- CallSphere (2026). *Text-to-SQL Evaluation: Spider, BIRD, and Custom Benchmarks*. https://callsphere.ai/blog/text-to-sql-evaluation-spider-bird-benchmarks-accuracy-testing
- MDPI Electronics (2025). *Design and Performance Evaluation of LLM-Based RAG Pipelines for Chatbot Services*. https://www.mdpi.com/2079-9292/14/15/3095
- Confident AI (2025). *LLM Evaluation Metrics: The Ultimate LLM Evaluation Guide*. https://www.confident-ai.com/blog/llm-evaluation-metrics-everything-you-need-for-llm-evaluation
- Zeng, X. et al. (2020). *Photon: A Robust Cross-Domain Text-to-SQL System*. ACL 2020. arXiv:2007.15280 (Error handling in NLIDB)

---

## AC4 — Bộ 30 câu hỏi test đầu vào
