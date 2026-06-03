# Phân tích Kiến trúc Hệ thống WrenAI (với GitNexus)

Dựa trên dữ liệu từ **GitNexus**, dưới đây là bản phác họa chi tiết về kiến trúc, luồng thực thi và các công cụ được sử dụng bên trong lõi của WrenAI.

## 1. Trả lời về Framework AI Agent
WrenAI **KHÔNG** sử dụng Haystack. Phân tích thư mục `sdk/` và mã nguồn cho thấy WrenAI hiện đang hỗ trợ chính thức hai framework AI Agent:
*   **LangChain**: Tích hợp thông qua `sdk/wren-langchain`
*   **Pydantic AI**: Tích hợp thông qua `sdk/wren-pydantic`

---

## 2. Tương tác Giữa Các Lớp (Layer Interactions)

Kiến trúc WrenAI được thiết kế theo 3 lớp lõi. Dưới đây là sơ đồ chi tiết về cách các lớp này làm việc với nhau, các cấu trúc dữ liệu, và công cụ cụ thể được phân tích từ codebase:

```mermaid
flowchart TB
    %% External
    LLM((LLM / AI Agent\nLangChain / Pydantic))
    DB[(Data Warehouse\nBigQuery, Postgres, ...)]

    %% Lớp 1: Agent SDK
    subgraph AgentSDK [Lớp 1: Agent SDK - Giao tiếp LLM]
        direction TB
        wren_query["wren_query() (Tool)"]
        wren_dry_plan["wren_dry_plan() (Tool)"]
        WrenToolkit["WrenToolkit"]
        
        wren_query & wren_dry_plan --> WrenToolkit
    end

    %% Lớp 2: Python Orchestration
    subgraph PythonLayer [Lớp 2: Python Orchestration - Điều phối logic]
        direction TB
        WrenEngine["WrenEngine (Class trung tâm)"]
        ConnectionManager["ConnectorABC / BaseConnectionInfo"]
        MDLParser["MDL Parser (Đọc JSON schema)"]
        SQLGlot["SQLGlot (Transpile AST)"]
        
        WrenToolkit -->|"Chuyển tiếp câu lệnh SQL / Intent"| WrenEngine
        WrenEngine <--> MDLParser
        WrenEngine <--> ConnectionManager
        WrenEngine <--> SQLGlot
    end

    %% Lớp 3: Semantic Engine
    subgraph RustLayer [Lớp 3: Semantic Engine - Lõi tính toán]
        direction TB
        DataFusion["Apache DataFusion (Query Execution)"]
        Dialects["Dialects (Generic, BigQuery, MySQL...)"]
        CTERewriter["CTERewriter (Phân tích Star Schema)"]
        
        DataFusion <--> CTERewriter
        DataFusion --> Dialects
    end

    %% Connections
    LLM <-->|Input: Natural Language / SQL\nOutput: Dữ liệu / Lỗi| AgentSDK
    PythonLayer <-->|"Gọi Rust bindings qua wren-core-py\nInput: SQL Draft & MDL\nOutput: SQL Chuẩn"| RustLayer
    ConnectionManager <-->|"Truy vấn vật lý\nInput: SQL Dialect\nOutput: Raw Data"| DB
    
    classDef sdk fill:#4f46e5,stroke:#3730a3,stroke-width:2px,color:#fff
    classDef py fill:#0ea5e9,stroke:#0369a1,stroke-width:2px,color:#fff
    classDef rust fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff
    
    class AgentSDK sdk
    class PythonLayer py
    class RustLayer rust
```

---

## 3. Luồng Thực thi Cốt lõi (`wren_query` Workflow)

Từ kết quả của GitNexus (`Wren_query → _detect_star_models`, Step Count: 8), luồng xử lý truy vấn đi qua các bước rất khắt khe để đảm bảo AI sinh ra mã chính xác. Dưới đây là Workflow tương tác (Sequence Diagram) thể hiện Input/Output chi tiết:

```mermaid
sequenceDiagram
    autonumber
    actor User as 👤 Người Dùng (User)
    participant UI as 🖥️ Wren UI (Frontend)
    participant Service as ⚙️ Wren AI Service (Backend/RAG)
    participant LLM as 🧠 LLM (Claude/OpenAI)
    
    box Lớp 1: Agent SDK (Python)
    participant Tool as wren_query() / WrenToolkit
    end
    
    box Lớp 2: Python Orchestration
    participant Engine as WrenEngine (engine.py)
    participant Connector as ConnectorABC (base.py)
    participant SQLGlot as SQLGlot (Transpiler)
    end
    
    box Lớp 3: Semantic Engine (Rust)
    participant RustCore as wren-core (Rust / DataFusion)
    end
    
    participant DB as 🗄️ Data Warehouse (BigQuery/Postgres)

    %% --- PHASE 1: USER INTENT & RAG ---
    User->>UI: Gõ: "Doanh thu tháng này theo khu vực?"
    UI->>Service: API POST /ask
    
    Service->>Service: Vector Search Context từ kho MDL
    Note over Service,LLM: Service đóng gói (Câu hỏi + Schema MDL + Lịch sử) thành Prompt
    Service->>LLM: Gửi Prompt
    
    LLM-->>Service: Function Call: wren_query(sql="SELECT ...")
    Note right of LLM: LLM suy luận ra Draft SQL và quyết định dùng công cụ.

    %% --- PHASE 2: AGENT SDK & ENGINE ENTRY ---
    Service->>Tool: Khởi chạy Tool: wren_query()
    Tool->>Engine: Gọi hàm WrenEngine.query(sql)

    %% --- PHASE 3: SEMANTIC VALIDATION & PLANNING ---
    Engine->>Engine: Gọi nội bộ WrenEngine.dry_plan(sql)
    Note over Engine,RustCore: Python đẩy Draft SQL xuống tầng Rust để lập kế hoạch
    
    Engine->>RustCore: Gửi SQL + MDL Schema
    RustCore->>RustCore: _collect_user_cte_names()
    Note right of RustCore: Bóc tách các CTE (Common Table Expressions) người dùng tạo
    RustCore->>RustCore: Resolve_model_name()
    Note right of RustCore: Kiểm tra xem các bảng/cột có thực sự tồn tại trong file MDL không
    RustCore->>RustCore: _detect_star_models()
    Note right of RustCore: Thuật toán dò tìm lõi: Dựng quan hệ Star Schema giữa các Fact & Dimension
    RustCore->>RustCore: _check_functions()
    Note right of RustCore: Chặn các hàm SQL không được hỗ trợ (chống Hallucination)
    
    RustCore-->>Engine: Trả về Logical Plan (AST) đã xác thực và an toàn

    %% --- PHASE 4: SQL TRANSPILING ---
    Engine->>Engine: _get_connector() (Tìm loại Database hiện tại)
    Engine->>SQLGlot: Chuyển Logical Plan sang Dialect của DB (VD: BigQuery)
    SQLGlot-->>Engine: Trả về Physical SQL query string

    %% --- PHASE 5: EXECUTION ---
    Engine->>Connector: Gọi hàm Connector.query(physical_sql)
    Connector->>DB: Execute Query (Qua mạng)
    DB-->>Connector: Raw Data (Dữ liệu thô từ Warehouse)
    Connector-->>Engine: Trả dữ liệu thô

    %% --- PHASE 6: POST-PROCESSING & RETURN ---
    Engine->>Engine: Áp dụng giới hạn Cap_size (tránh OOM)
    Engine->>Engine: Chuyển đổi an toàn Json_safe()
    Engine-->>Tool: Trả về dữ liệu dạng JSON
    
    Tool->>Tool: format_query_content()
    Tool-->>Service: Trả về make_success(Dữ liệu JSON)

    %% --- PHASE 7: EXPLANATION & RENDER ---
    Service->>LLM: Nhờ LLM đọc JSON và sinh câu trả lời giải thích bằng chữ
    LLM-->>Service: Lời giải thích tự nhiên
    Service-->>UI: Trả về Data JSON + Lời giải thích
    
    UI->>UI: Render biểu đồ / Bảng dữ liệu từ JSON
    UI-->>User: Hiển thị kết quả trực quan trên màn hình
```

### Phân tích Sâu từng Nút thắt (Deep-dive Analysis):
1.  **Từ User đến Engine (Bước 1 - 2):** Người dùng không bao giờ chạm trực tiếp vào Engine. Mọi yêu cầu được Wren AI Service (hệ thống backend tách rời chứa UI) bọc lại bằng RAG và ép LLM phải sử dụng Tool `wren_query`.
2.  **Bức tường lửa (Bước 3):** Hàm `dry_plan` của Python Engine lập tức ném câu SQL còn non nớt của LLM xuống tầng Rust. Tại đây, Rust dùng các thuật toán như `_detect_star_models` để vẽ lại cấu trúc bảng (đảm bảo đúng ngữ nghĩa kinh doanh) chứ không tin tưởng hoàn toàn vào câu SQL do LLM viết.
3.  **Dịch ngôn ngữ (Bước 4):** Lõi Rust không sinh ra mã SQL trực tiếp. Nó sinh ra một cái cây trừu tượng (AST). Sau đó Python lấy cây này, đút vào thư viện `SQLGlot` để phiên dịch ra đúng "tiếng địa phương" của cơ sở dữ liệu (Dialect).
4.  **Kiểm soát bộ nhớ (Bước 6):** Nếu DB trả về 1 triệu dòng, việc nạp thẳng vào LLM sẽ làm nổ (Out of Context Window). Do đó Engine có hàm `Cap_size` để chặt bớt dữ liệu trước khi format thành JSON.
5.  **Dựng UI (Bước 7):** Kết quả cuối cùng đi lên UI luôn gồm 2 phần: Dữ liệu JSON (để thư viện Frontend tự dựng biểu đồ) và Lời giải thích do LLM tạo ra. User sẽ nhìn thấy biểu đồ chứ không chỉ thấy chữ.

---

## 4. Tổng kết công cụ (Tech Stack)
*   **Ngôn ngữ:** Python (Điều phối & Agent), Rust (Lõi Engine tính toán), TypeScript (WebAssembly).
*   **Core Engine:** Apache DataFusion (Dùng để xử lý dữ liệu và logic SQL nhanh chóng trên RAM).
*   **Parsing/Transpiling:** SQLGlot (dùng để dịch qua lại giữa các ngôn ngữ SQL).
*   **Agent Integration:** LangChain, Pydantic AI.
*   **Semantic Layer:** MDL (Modeling Definition Language - định dạng schema của WrenAI).
