# Báo Cáo Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án (Reflection & Action Plan)

**Học viên:** Đoàn Tiến Thành

---

## 📌 PHẦN 1: MAPPING BÀI GIẢNG VÀO CODE

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|:---|:---:|:---|:---|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Giảm thiểu false pronoun resolution; bảo vệ các thực thể số và ngày tháng. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Chặn đứng các relation rác do LLM tự sinh; đảm bảo schema Neo4j luôn sạch. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND` nạp 58 edges và 98 nodes chỉ trong vài mili-giây, không bị lock DB. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp thực thể chính xác, chặn thành công ca lỗi `David Gardner` vs `Tom Gardner` ($0.8288$). |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Cắt tỉa `degree > 100 -> cap 50`, bảo vệ context window của LLM không bị quá tải. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `robust_judge_answer()` | Chấm điểm tự động khách quan trên 3 tiêu chí với định dạng JSON chuẩn. |

---

## 📌 PHẦN 2: QUÁ TRÌNH DEBUGGING & BÀI HỌC KỸ THUẬT

### 1. Lỗi kỹ thuật phức tạp nhất gặp phải:
1. **Lỗi Rate Limit 429 & JSON Mode 400 trên Groq:** Trong bước Judge Evaluation, model `openai/gpt-oss-120b` bị cạn hạn ngạch token trong ngày (TPD Limit 200k), đồng thời cờ `json_mode=True` bị crash khi context dài do server-side validation của Groq.
2. **Lỗi lệch môi trường Local IDE vs Remote Colab Kernel:** File `.env` lưu ở máy local không tự động đồng bộ lên máy ảo Linux của Google Colab.

### 2. Cách xử lý thành công:
1. Chuyển sang model **`openai/gpt-oss-20b`** có hạn ngạch dồi dào, tắt strict `json_mode` phía server và dùng hàm bóc tách Regex `parse_json_object` kết hợp cơ chế Safe Fallback scoring giúp vòng lặp chạy 100% không bị dừng lại.
2. Cấu hình biến môi trường trực tiếp trên Colab Secrets và đồng bộ dữ liệu chuẩn mực.

---

## 📌 PHẦN 3: KẾ HOẠCH ÁP DỤNG VÀO ĐỒ ÁN THỰC TẾ (ACTION PLAN)

- **Tên đồ án dự kiến:** Hệ thống Trợ lý Pháp lý & Tra cứu Quan hệ Sở hữu Doanh nghiệp (Enterprise Knowledge & Legal GraphRAG).
- **Lý do chọn GraphRAG:** Các vụ án pháp lý hoặc điều tra sở hữu chéo công ty đòi hỏi khả năng **truy vết đa chặng (multi-hop traversal qua 3-5 cấp cổ đông/công ty con)** mà Vector Search phẳng hoàn toàn bất lực vì thông tin bị chia nhỏ ở nhiều điều khoản/hợp đồng khác nhau.
- **Cấu trúc Node & Relation dự kiến:**
  - *Nodes:* `DoanhNghiep`, `CaNhanDaiDien`, `VanBanPhapLuat`, `DuAn`, `CoPhan`.
  - *Relations:* `SO_HUU_CO_PHAN`, `DAI_DIEN_PHAP_LUAT`, `CONG_TY_ME_CON`, `VI_PHAM_QUY_DINH`, `THUC_HIEN_DU_AN`.
- **Chiến lược xử lý Super-node & Entity Resolution:**
  - Sử dụng Mã số thuế (MST) / Số CCCD làm Unique Canonical Key thay vì chỉ so khớp tên chuỗi.
  - Áp dụng Super-node degree cap cho các tập đoàn lớn (như Vingroup, Viettel) kết hợp phân cấp Community Detection theo ngành nghề.

---

## 🎁 PHẦN 4: THUYẾT MINH THỰC HIỆN BONUS (+10 ĐIỂM)

1. **Bonus 1 — Near-Deduplication (+3 Điểm):** Thuật toán MinHash + LSH ($M=128$, $b=16$, $r=8$, Jaccard threshold $0.82$, length guard $\ge 0.60$). Lọc sạch 33k exact dedup + 3 bài near-dedup trong mẫu 1,500 bài.
2. **Bonus 2 — Global Search via Community Detection (+5 Điểm):** Phân cụm và gán nhãn thành công **40 cụm cộng đồng** vào Neo4j bằng NetworkX Modularity.
3. **Bonus 3 — Self-Correction Graph Retrieval (+5 Điểm):** Tự động mở rộng bán kính truy xuất và kích hoạt thành công tuyến đường **`hop3+vector`** với độ dài ngữ cảnh đạt **2,790 ký tự**.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|:---|:---:|:---|
| **Mức độ hiểu bài giảng GraphRAG** | 5/5 | Nắm vững toàn bộ pipeline từ Deduplication, Coreference, NER/RE, Entity Resolution đến Hybrid Retrieval. |
| **Khả năng kiểm soát AI Coding Agent** | 5/5 | Bác bỏ thuật toán $O(N^2)$, chủ động chỉ đạo Agent thiết kế MinHash LSH và Lexical Guard. |
| **Chất lượng đồ thị tri thức xây dựng** | 5/5 | Schema chuẩn, 100% provenance đầy đủ, kiểm soát tốt Super-nodes, có phân cụm Community. |
| **Khả năng phân tích và debug hệ thống** | 5/5 | Tự xử lý triệt để các lỗi dependency, môi trường Colab, và Rate limit LLM. |
