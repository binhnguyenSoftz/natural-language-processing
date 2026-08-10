# Đề tài & Pipeline nghiên cứu
## 1. Tên đề tài
**Xây dựng khung sinh dữ liệu hội thoại tổng hợp có kiểm soát ngữ cảnh, kết hợp tri thức nền và truy xuất ngữ nghĩa, phục vụ huấn luyện hệ thống NLU và mô hình ngôn ngữ nhỏ cho trợ lý gọi món tiếng Việt.**
## 2. Câu hỏi nghiên cứu (Research Questions)

|Mã|Nội dung|
|---|---|
|**RQ1**|Dữ liệu hội thoại tổng hợp sinh theo ma trận ngữ cảnh có kiểm soát (tâm trạng/hành vi × thời gian/không gian × loại giao dịch) có cải thiện độ chính xác và độ bao phủ của mô hình NLU so với dữ liệu sinh không kiểm soát ngữ cảnh hay không?|
|**RQ2**|Việc grounding LLM bằng tri thức nền (menu, quy trình nghiệp vụ theo từng quán) khi sinh dữ liệu có giảm tỷ lệ hallucination thực thể so với sinh không grounding hay không?|
|**RQ3**|Với cùng ngân sách dữ liệu và tài nguyên tính toán, kiến trúc pipeline-based (NER chuyên biệt + Intent Classifier) có đạt độ chính xác trích xuất thực thể cao hơn kiến trúc end-to-end (LLM nhỏ tự sinh JSON) hay không?|
## 3. Ma trận ngữ cảnh
Ma trận ngữ cảnh gồm **3 chiều**, mỗi lần sinh hội thoại sẽ chọn ngẫu nhiên có trọng số 1 tổ hợp giá trị từ 3 chiều này:

|Chiều|Ví dụ giá trị|Tín hiệu quan sát được trong văn bản|
|---|---|---|
|**Tâm trạng/hành vi**|Vội vàng, khó tính, thân thiện, do dự|Độ dài lượt thoại, số lần ngắt lời/hỏi lại, mức độ lịch sự|
|**Thời gian/không gian**|Giờ cao điểm, cuối tuần, giao hàng, tại quán|Entity liên quan (địa chỉ, giờ hẹn), độ dài hội thoại|
|**Loại giao dịch**|Đặt món mới, đổi món, hủy, hỏi thông tin|Quyết định trực tiếp intent|
## 4. Cấu trúc Vector DB

|Collection|Nội dung|Mục đích|
|---|---|---|
|Store Profile|Tên quán, loại hình, giờ hoạt động|Xác định phạm vi truy vấn cho các collection khác|
|Menu Items (theo quán)|Tên món, giá, size, topping, tên gọi thay thế thông dụng|Grounding sinh dữ liệu (RQ2) + Entity Normalization|
|Business Rules (theo quán)|Quy trình đặt món, chính sách đổi/hủy/giao hàng|Đảm bảo hội thoại đúng logic nghiệp vụ|
|Generated Dialogues (theo quán)|Hội thoại đã sinh, gắn nhãn ma trận ngữ cảnh|Diversity check / dedup|
|Few-shot Prompt Examples|Cặp (ngữ cảnh, hội thoại mẫu chất lượng cao)|Retrieval-augmented prompting khi sinh dữ liệu|
|Real Conversations _(tuỳ chọn)_|Hội thoại thật thu thập được|Đối chiếu độ tự nhiên của dữ liệu tổng hợp + hỗ trợ xây dựng ma trận ngữ cảnh (mục 3)|
## 5. Pipeline tổng thể
```
┌─────────────────────────────────────────────────────────┐
│                 VECTOR DB (multi-tenant)                  │
│  Store Profile | Menu Items | Business Rules |            │
│  Generated Dialogues | Few-shot Examples | Real Convos     │
└─────────────────────────────────────────────────────────┘
        │ retrieve (theo store_id)      │ dedup check
        ▼                                ▼
┌───────────────────────────────────────────────────────┐
│ GIAI ĐOẠN 1 — SINH DỮ LIỆU                              │
│                                                          │
│ [A]  Chọn quán (store_id) → retrieve Menu + Business Rules│
│ [B]  Pick point trong ma trận ngữ cảnh:                   │
│      tâm trạng/hành vi × thời gian/không gian ×           │
│      loại giao dịch (có trọng số xác suất)                │
│      → build thành mô tả ngữ cảnh (natural language)      │
│ [C]  Search similarity: retrieve few-shot examples         │
│      gần giống ngữ cảnh                                    │
│ [D]  Sinh hội thoại bằng LLM lớn (GPT-4o/Claude)           │
│      — grounded bằng menu/rules + few-shot                │
│ [D''] Context Validation:                                  │
│      • Loại giao dịch ↔ Intent: so khớp trực tiếp          │
│      • Thời gian/không gian ↔ Entity bắt buộc: rule-based  │
│      • Tâm trạng/hành vi: LLM-as-judge chấm điểm khớp      │
│      → tính Context Adherence Rate, loại nếu không đạt      │
│ [D']  Diversity check (embed → so similarity trong          │
│       Vector DB cùng quán → loại nếu trùng lặp)             │
│ [E]  Gán nhãn: Intent + NER (BIO)                          │
│ [F]  Entity Normalization (semantic search trong Menu      │
│      Items của đúng quán → chuẩn hóa món/thuộc tính,       │
│      đo entity hallucination rate)                         │
│ [G]  TTS + Noise Augmentation (nhạc nền, tiếng máy xay...) │
└───────────────────────────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────┐
│ GIAI ĐOẠN 2 — HUẤN LUYỆN & ĐÁNH GIÁ (2 nhánh song song)  │
│                                                          │
│  Nhánh A: NER (PhoBERT/GLiNER) + Intent Classifier        │
│  Nhánh B: LLM nhỏ (Qwen/Llama) end-to-end sinh JSON        │
│           │                    │                         │
│           └────────┬───────────┘                         │
│                     ▼                                     │
│     ĐÁNH GIÁ: Intent F1, Entity F1, Hallucination rate,    │
│     Entity Linking accuracy, Latency inference (server)    │
│     (RQ1–RQ3)                                              │
│                     ▼                                     │
│     LLM nhỏ (NLG) sinh phản hồi từ (intent, entities)       │
│     → Demo Proof-of-Concept (phụ lục)                      │
└───────────────────────────────────────────────────────┘
```
## 6. Chỉ số đánh giá bổ sung — Context Adherence Rate
Đo lường mức độ hội thoại sinh ra khớp đúng với ngữ cảnh đã pick ở bước [B], tính riêng cho từng chiều:
```
Context Adherence Rate (chiều X) = 
    số hội thoại khớp đúng chiều X / tổng số hội thoại được validate
```

|Chiều|Phương pháp đo|
|---|---|
|Loại giao dịch|Automatic/rule-based — so khớp intent với loại giao dịch đã pick|
|Thời gian/không gian|Automatic/rule-based — kiểm tra entity bắt buộc có/không xuất hiện đúng|
|Tâm trạng/hành vi|LLM-as-judge — chấm điểm mức độ phù hợp (thang điểm hoặc nhị phân)|
Chỉ số này phục vụ trực tiếp RQ1: chứng minh dữ liệu sinh theo ma trận không chỉ _được gắn nhãn ngữ cảnh_, mà _thực sự phản ánh đúng ngữ cảnh đó_ — làm căn cứ tin cậy khi so sánh với baseline dữ liệu sinh ngẫu nhiên.
## 7. Baseline so sánh

|Baseline|Dùng để kiểm chứng|
|---|---|
|Dữ liệu sinh ngẫu nhiên (không dùng ma trận ngữ cảnh)|RQ1|
|Dữ liệu sinh không grounding (không dùng menu/rules)|RQ2|
|Kiến trúc end-to-end bằng LLM nhỏ|RQ3|
|Zero-shot prompting (không retrieval few-shot)|Đánh giá bổ sung chất lượng sinh dữ liệu|
## 8. Phạm vi loại trừ
- Không triển khai sản phẩm/robot hoàn chỉnh — chỉ dừng ở proof-of-concept (phụ lục)
- Thử nghiệm trên 2–3 quán thuộc loại hình khác nhau, không cần scale lớn
- 1 mô hình NER (PhoBERT) + 1 LLM nhỏ (chọn 1 trong 2: Qwen2.5 hoặc Llama 3.1)
- Vector DB dùng FAISS/Chroma — đủ dùng cho luận văn, không cần production-grade
- Không xử lý nhận diện nhân khẩu học (già/trẻ, nghề nghiệp...) — việc này cần kết hợp thị giác máy tính, nằm ngoài phạm vi đề tài
- Mô hình ngôn ngữ nhỏ được huấn luyện/triển khai/đánh giá trên server — không phải mô hình nhúng trên thiết bị phần cứng hạn chế.
