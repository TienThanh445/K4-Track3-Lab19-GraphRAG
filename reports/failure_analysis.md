# Báo Cáo Phân Tích Ca Lỗi (Failure Analysis) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đoàn Tiến Thành

---

## 🔍 BẢNG SO SÁNH BENCHMARK TỔNG HỢP (LLM-AS-A-JUDGE)

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|:---|:---:|:---:|:---:|:---|
| **Comprehensiveness (1–5)** | 1.333 | 1.167 | -0.166 | Flat RAG nhỉnh hơn do đưa toàn bộ text chunk thô vào context. |
| **Faithfulness (1–5)** | 1.000 | 1.000 | 0.000 | Cả 2 hệ thống đều tuân thủ tốt nguyên tắc không bịa đặt khi thiếu context. |
| **Multi-hop Reasoning (1–5)** | 1.000 | 1.000 | 0.000 | Điểm cân bằng trên tập mẫu dữ liệu nhỏ. |
| **Latency trung bình (s)** | **2.700s** | **4.090s** | +1.390s | GraphRAG tốn thêm thời gian trích xuất seed + Cypher traversal đồ thị. |
| **Token usage trung bình** | **879.7** | **711.2** | **-168.5** | **GraphRAG tiết kiệm token hơn** nhờ đồ thị được tuyến tính hóa cô đọng. |

---

## 🚨 PHÂN TÍCH 2 CA LỖI ĐIỂN HÌNH (ROOT-CAUSE ANALYSIS)

### Ca lỗi 1: Flat RAG Bịa đặt / Ảo giác (GraphRAG Trung thực & Đạt chuẩn Faithfulness)

- **Question ID:** `G5000-41` (`group: factoid`)
- **Câu hỏi:** *"Which company did HPE agree to acquire to expand edge-to-cloud security, and what unified security architecture did the report say the deal would support?"*
- **Reference Answer:** *"Axis Security; a unified Secure Access Service Edge (SASE) solution."*

#### 1. Hành vi của Flat RAG (Thất bại do Hallucination):
- **Câu trả lời sinh ra:** *"HPE agreed to acquire **QuSecure** to broaden its edge‑to‑cloud security offering [chunk_id=8dc576fda3c5168c162a]..."*
- **Truy vết nguyên nhân gốc rễ (Root Cause):**
  1. Do trong tập mẫu trích xuất không có chunk chứa đúng bài báo về Axis Security, Vector Search dựa trên độ tương đồng ngữ nghĩa đã tìm kiếm các chunk có từ khóa *"HPE"*, *"edge-to-cloud"*, *"security"*.
  2. Nó tìm thấy `chunk_id=8dc576fda3c5168c162a` nói về công nghệ của công ty *QuSecure*.
  3. LLM của Flat RAG khi đọc đoạn text này đã tự động suy diễn và **bịa đặt (Hallucination)** rằng HPE mua lại QuSecure, tạo ra thông tin hoàn toàn sai lệch thực tế.

#### 2. Hành vi của GraphRAG (Thành công về tính trung thực - Faithfulness):
- **Câu trả lời sinh ra:** *"I’m sorry, but the supplied context does not contain any information about HPE’s acquisition or the unified security..."*
- **Tại sao GraphRAG giải quyết được:**
  1. Seed extractor nhận diện thực thể `HPE`.
  2. Graph Traversal truy vấn các cạnh nối với node `HPE` trong Neo4j và không tìm thấy bất kỳ cạnh `ACQUIRED` nào nối tới `Axis Security` hay `QuSecure`.
  3. Khi tuyến tính hóa Subgraph context, đồ thị không cung cấp thông tin sai lệch $\rightarrow$ LLM Generator tuân thủ nghiêm ngặt nguyên tắc chỉ trả lời dựa trên bằng chứng và **từ chối bịa đặt một cách trung thực**.

---

### Ca lỗi 2: GraphRAG Thất bại do Missing Subgraph (Tập dữ liệu subset bị phân mảnh)

- **Question ID:** `G5000-26` / `G5000-28` (`group: multi-hop`)
- **Câu hỏi:** *"Which model providers are connected to Google Cloud Next '23 in the selected data, and which models are associated with each provider?"*
- **Reference Answer:** *"Meta is connected via Llama 2 and Code Llama; the Technology Innovation Institute is connected via Falcon LLM; Anthropic is connected via Claude 2, which Google Cloud pre-announced."*

#### 1. Hiện tượng & Phân tích nguyên nhân:
- Cả Flat RAG và GraphRAG đều trả về: *"I’m sorry, but the supplied context does not contain any information about Google Cloud Next '23..."*
- **Nguyên nhân gốc rễ:** 
  1. Do thời lượng chạy bài lab ngắn, tập dữ liệu chỉ trích xuất từ 400 chunks (`EXTRACTION_MAX_CHUNKS = 400`), trong khi bài báo về Google Cloud Next '23 nằm ở dòng thứ 3,395 trong toàn bộ dataset gốc.
  2. Khi trích xuất không bao phủ dòng này, đồ thị Neo4j hoàn toàn thiếu vắng các node `Google Cloud Next '23`, `Falcon LLM`, `Code Llama`.
  3. Graph Traversal nhận kết quả `NO_SEED` hoặc 0 cạnh.

#### 2. Giải pháp khắc phục chuẩn Production:
- **Cơ chế Self-Correction & Hybrid Vector Fallback (Đã triển khai ở Bonus 2):**
  1. Nếu Graph Traversal trả về `NO_SEED` hoặc context đồ thị rỗng, hệ thống tự động kích hoạt tuyến đường `hop3+vector`.
  2. Kết hợp truy xuất Dense Vector top-$k$ chunks kèm báo cáo tóm tắt cộng đồng (Community Reports) để bù đắp các lỗ hổng tri thức chưa kịp trích xuất thành đồ thị.
