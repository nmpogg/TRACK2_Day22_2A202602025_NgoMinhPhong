# Bằng chứng — Day 22: LangSmith + Prompt Versioning

Thư mục này chứa bằng chứng cho 4 nhiệm vụ của lab. Dưới đây là danh mục file và
phần **phân tích so sánh Prompt V1 vs V2** dựa trên kết quả RAGAS.

## Danh mục bằng chứng

| File | Nhiệm vụ | Nội dung |
|------|----------|----------|
| `01_langsmith_traces.png` | 1 | Giao diện LangSmith với ≥ 50 traces |
| `02_prompt_hub.png` | 2 | Prompt Hub hiển thị 2 phiên bản được đặt tên |
| `02_ab_routing_log.txt` | 2 | Log console A/B routing (50 câu, nhãn v1/v2) |
| `03_ragas_scores.png` | 3 | Bảng so sánh điểm RAGAS V1 vs V2 |
| `03_ragas_report.json` | 3 | Bản sao của `data/ragas_report.json` |
| `04_pii_demo_log.txt` | 4 | Output các test case phát hiện & che PII |
| `04_json_demo_log.txt` | 4 | Output các test case sửa JSON |

## Định nghĩa 2 prompt version

- **V1 — Ngắn gọn:** "trợ lý AI hữu ích", trả lời trực tiếp **2–4 câu**, chỉ dùng
  context, nói "không biết" nếu thiếu thông tin.
- **V2 — Có cấu trúc:** "chuyên gia AI", đọc kỹ context → xác định facts liên quan
  → viết câu trả lời rõ ràng, có tổ chức **3–5 câu**.

Phân bổ A/B routing (tất định theo `hash(request_id) % 2`): **V1 = 19 câu, V2 = 31 câu**
trên tổng 50 câu.

## Kết quả RAGAS

| Chỉ số | V1 (ngắn gọn) | V2 (có cấu trúc) | Cao hơn |
|--------|:-------------:|:----------------:|:-------:|
| faithfulness       | **0.9463** | **0.9465** | ≈ ngang (V2 +0.0002) |
| answer_relevancy   | **0.9129** | 0.8902 | **V1** (+0.0227) |
| context_recall     | 1.0000 | 1.0000 | ngang |
| context_precision  | **0.9417** | 0.9350 | **V1** (+0.0067) |

> **Mục tiêu faithfulness ≥ 0.8: ĐẠT ở cả 2 phiên bản** — và thực tế **cả hai đều ≥ 0.9**.

## Phân tích: vì sao V1 nhỉnh hơn?

**Kết luận: V1 (ngắn gọn) tốt hơn V2 (có cấu trúc) một chút về tổng thể**, dù khác biệt nhỏ.

1. **`answer_relevancy` — V1 thắng rõ nhất (+0.023).**
   RAGAS đo answer_relevancy bằng cách sinh ngược câu hỏi từ câu trả lời rồi so
   với câu hỏi gốc. Prompt V1 yêu cầu **trả lời trực tiếp 2–4 câu**, nên câu trả
   lời bám sát đúng câu hỏi. V2 yêu cầu **3–5 câu, có tổ chức**, dễ thêm câu dẫn
   nhập / bối cảnh phụ → làm "loãng" độ liên quan trực tiếp, kéo điểm xuống.

2. **`faithfulness` — gần như ngang (chênh 0.0002).**
   Cả 2 prompt đều ràng buộc "**chỉ dùng context**", nên tỉ lệ claim suy ra được
   từ context là như nhau. Việc dài hơn (V2) không làm model bịa thêm — chứng tỏ
   ràng buộc grounding hiệu quả ở cả hai.

3. **`context_recall` = 1.0 ở cả hai.**
   Đây là chỉ số của **retriever**, không phụ thuộc prompt sinh câu trả lời. Cùng
   một vectorstore FAISS (k=3) nên recall giống hệt nhau — đúng như kỳ vọng.

4. **`context_precision` — V1 nhỉnh nhẹ (+0.007).**
   Cũng chủ yếu do retriever; chênh lệch nhỏ đến từ dao động của LLM-judge, không
   phải khác biệt có ý nghĩa thống kê.

### Nhận định

- Với knowledge base dạng **factual Q&A** (định nghĩa ML/NLP/RAG…), **câu trả lời
  ngắn gọn và trực tiếp (V1) phù hợp hơn**: đúng trọng tâm, ít lan man → relevancy cao hơn.
- V2 (dài, có cấu trúc) sẽ có lợi thế hơn ở các câu hỏi **mở / cần giải thích nhiều
  bước**, nhưng ở bộ câu hỏi định nghĩa này thì sự "có tổ chức" không tạo thêm giá trị
  mà còn làm giảm nhẹ độ liên quan.
- Khác biệt nhìn chung **nhỏ** → cả hai prompt đều là lựa chọn sản xuất hợp lý;
  nếu ưu tiên độ ngắn gọn & bám câu hỏi thì **chọn V1**.
