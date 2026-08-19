# Báo Cáo Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG
---

## 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Tại `chunk_id = 55a09cbc43c41ffb2dd9::c0000` (Tin tức về sự hợp tác giữa *Transact Campus* và *Talkiatry*):  
  *Nguyên văn:* "The partnership will provide students with access to quality psychiatric services... Transact said Transact will provide partner institutions... **it** plans to roll out across campus apps."
- **Hiện tượng:** Đại từ *"it"* nằm ở câu sau có thể bị hiểu nhầm là đại diện cho *Talkiatry* thay vì chủ ngữ chính của nền tảng phân phối là *Transact*. Trong các văn bản tin tức tài chính/M&A nhiều thực thể cùng xuất hiện (bên mua, bên bán, bên tư vấn), LLM dễ gán nhầm đại từ *"the company"*, *"it"*, *"the firm"* cho thực thể được nhắc tới gần nhất thay vì thực thể thực hiện hành động.
- **Hậu quả đối với Graph:** Tạo ra **False Edge (Cạnh quan hệ sai lệch)**. Ví dụ: Gán nhầm quan hệ `(Talkiatry)-[DEVELOPED]->(Campus App)` thay vì `(Transact)-[DEVELOPED]->(Campus App)`. Trong đồ thị tri thức, một false edge sẽ làm sai lệch toàn bộ đường đi suy luận đa chặng (multi-hop traversal) sau này.
- **Biện pháp kiểm soát:** Áp dụng nguyên tắc **Conservative Coreference**: Chỉ phân giải khi quan hệ tiền ngữ (antecedent) hoàn toàn rõ ràng trong cùng chunk; nếu có sự mơ hồ (ambiguity), giữ nguyên thực thể và đưa vào danh sách `unresolved_mentions` thay vì đoán mò.

---

## 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.80$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.80` (kết hợp `top_k = 5` trên FAISS `IndexFlatIP`).
- **Cặp thực thể thực tế bị Guard chặn từ Audit Log:**
  - `Thực thể Left:` **David Gardner** (`type: Person`)
  - `Thực thể Right:` **Tom Gardner** (`type: Person`)
  - `Độ tương đồng Vector Cosine:` **`0.8288`**
  - `Quyết định & Lý do từ Guard:` `REJECT_GUARD` với mã `REJECT_PERSON_DIFFERENT_FIRST_NAME`.
- **Lý do ngữ nghĩa tại sao chặn:**  
  Cả hai đều là chuyên gia đầu tư tài chính công nghệ và là hai anh em đồng sáng lập *The Motley Fool*. Vì cùng xuất hiện trong ngữ cảnh đầu tư công nghệ và có chung họ *Gardner*, vector embedding của 2 cái tên này đạt độ tương đồng cosine rất cao ($0.8288$). Nếu chỉ dùng Vector ANN thuần túy, 2 người này sẽ bị **gộp nhầm thành 1 node duy nhất (False Merge)**. Nhờ có **Lexical Guard của Challenge B**, hệ thống nhận diện được phần tên riêng khác nhau (`David` $\neq$ `Tom`) và từ chối gộp, bảo toàn tính chính xác của thực thể con người.

---

## 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes thực tế trong đồ thị:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|:---:|:---|:---:|:---:|
| **1** | **Apple** | Company | **5** |
| **2** | **Railergy** | Company | **5** |
| **3** | **Meeno** / **Unacademy** | Company | **3** |

- **Ưu điểm & Rủi ro của Temporal Mitigation (`published_date DESC LIMIT 50`):**
  - *Ưu điểm:* 
    1. Ngăn chặn hiện tượng **Bùng nổ ngữ cảnh (Context Explosion)**: Các Big Tech như Apple/Google/Microsoft có thể có hàng nghìn kết nối trong đồ thị lớn; việc giới hạn 50 cạnh giúp prompt không bị tràn token context của LLM.
    2. **Độ tươi mới của dữ liệu (Recency):** Ưu tiên các sự kiện M&A, quan hệ đối tác, và sản phẩm công nghệ mới nhất, phản ánh đúng tình trạng kinh doanh hiện tại của doanh nghiệp.
  - *Rủi ro:* 
    1. **Mất dấu thông tin lịch sử:** Đối với các câu hỏi điều tra nguồn gốc (ví dụ: *"Ai là nhà sáng lập ban đầu?"* hoặc *"Thương vụ thâu tóm đầu tiên của Apple là gì?"*), các cạnh quan hệ lịch sử (thường có `published_date` từ nhiều năm trước) sẽ bị cắt tỉa hoàn toàn khỏi retrieval context.
    2. **Độ nhạy với dữ liệu thiếu ngày (`NULL published_date`):** Các bài báo không parse được ngày xuất bản có thể bị đẩy xuống cuối và bị loại bỏ oan uổng.

---

## 4. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:**

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:**
  - *Flat RAG:* Chi phí indexing rất rẻ ($O(N)$ embedding), truy vấn nhanh ($< 0.5s$), nhưng ngữ cảnh cồng kềnh, dễ bị phân mảnh khi trả lời câu hỏi đa chặng (multi-hop) và dễ bị ảo giác khi gặp chunk gần giống.
  - *GraphRAG:* Chi phí indexing ban đầu cao (chi phí gọi LLM trích xuất NER/RE), độ trễ truy vấn cao hơn $\approx 1.5s$ (do seed extraction + graph traversal), nhưng **tiết kiệm token prompt (ít hơn $\approx 20\%$)**, cấu trúc quan hệ tường minh và kiểm soát nguồn gốc (provenance) cực kỳ chặt chẽ.

- **Quyết định từ chối AI Coding Agent:**
  - Trong quá trình triển khai Challenge A (Near-Dedup), AI Coding Agent từng đề xuất tính ma trận tương đồng $O(N^2)$ Pairwise Cosine Similarity trên toàn bộ 212,212 bài báo.
  - **Lý do từ chối:** Với $N = 212,212$, số phép tính so sánh lên tới $\frac{N(N-1)}{2} \approx 22.5$ tỷ phép tính, chắc chắn gây tràn RAM (OOM) và treo máy. Tôi đã yêu cầu Agent chuyển sang kiến trúc **MinHash + Locality Sensitive Hashing (LSH)** với $M=128$ hàm băm và $b=16$ bands, giảm độ phức tạp xuống $O(N \cdot M)$ tuyến tính, xử lý lọc 1,500 bài trong chưa đầy 1 giây.

- **Giải pháp Scale lên toàn bộ 350MB (~100,000 bài báo / 7 triệu dòng):**
  1. *Bottleneck 1 (LLM Extraction):* Trích xuất 100k bài báo qua LLM sẽ bị nghẽn I/O và rate limit. **Giải pháp:** Sử dụng hàng đợi bất đồng bộ (Celery + Redis / Ray workers) gọi song song đa luồng qua các model chuyên dụng nhỏ hơn (vLLM / Llama-3-8B local) với cơ chế JSON Schema enforcement.
  2. *Bottleneck 2 (Neo4j Ingestion):* Insert từng node sẽ gây lock database. **Giải pháp:** Bắt buộc dùng Cypher `UNWIND` theo batch 5,000 records kèm Unique Constraints trên `(Entity.id)`.
  3. *Bottleneck 3 (Entity Resolution):* **Giải pháp:** Áp dụng thuật toán Blocking (theo First Letter + Soundex) trước khi chạy FAISS HNSW ANN, kết hợp Disjoint-Set Union (Union-Find) song song.
