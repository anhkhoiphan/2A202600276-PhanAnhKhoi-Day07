Dưới đây là phiên bản đã điền hoàn chỉnh các phần còn thiếu, giữ nguyên phong cách và nội dung của bạn:

---

# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Phan Anh Khôi
**Nhóm:** 29
**Ngày:** 10/04/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**

> Hai vector đại diện cho 2 từ/câu/ảnh... có high cosine similarity có nghĩa là góc giữa 2 vector nhỏ, 2 vector có hướng gần giống nhau, nội dung/ngữ nghĩa của hai từ/câu/ảnh rất tương đồng dù có thể dùng từ khác nhau.

**Ví dụ HIGH similarity:**

* Sentence A: Con chó nghiệp vụ này rất cừ.
* Sentence B: Con cảnh khuyển này rất giỏi.
* Tại sao tương đồng: 2 câu về mặt ngữ nghĩa giống nhau, dùng các từ đồng nghĩa.

**Ví dụ LOW similarity:**

* Sentence A: Con chó nghiệp vụ này rất cừ.
* Sentence B: Mèo lười thèm cá rán.
* Tại sao khác: Hai câu không có gì giống nhau về mặt ngữ nghĩa.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**

> Vì cosine similarity chỉ quan tâm đến hướng (ngữ nghĩa) của vector, không bị ảnh hưởng bởi độ dài vector như Euclidean distance, nên phù hợp hơn để so sánh ý nghĩa câu.

---

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**

> (10000 - 500) / (500 - 50) + 1 = 9500 / 450 + 1 ≈ 21.1 + 1 ≈ 22
> **Đáp án:** 22 chunks

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**

> Số chunks = ⌈9500 / 400⌉ + 1 = 24 + 1 = 25 chunks — nhiều hơn 3 chunks so với overlap=50. Overlap lớn hơn giúp giữ lại nhiều context ở ranh giới chunk, giảm mất thông tin quan trọng.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Financial Reports Analysis (Báo cáo tài chính doanh nghiệp)

**Tại sao nhóm chọn domain này?**

> Vì phân tích tài chính là công việc cần trình độ chuyên môn cao, tác vụ lặp đi lặp lại, mất đến 20 phút - hàng tiếng để đọc và trích xuất dữ liệu quan trọng vào Excel. Việc dùng embedding + retrieval giúp tự động hóa quá trình tìm kiếm thông tin trong báo cáo dài hàng trăm trang.

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

| Tài liệu | Strategy                         | Chunk Count | Avg Length | Preserves Context? |
| -------- | -------------------------------- | ----------- | ---------- | ------------------ |
| NVIDIA   | FixedSizeChunker (`fixed_size`)  | 1098        | ~360 ký tự | No (cắt giữa câu)  |
| TESLA    | SentenceChunker (`by_sentences`) | 642         | ~640 ký tự | Yes                |
| TESLA    | RecursiveChunker (`recursive`)   | 1206        | ~300 ký tự | Medium             |

---

## 4. My Approach — Cá nhân (10 điểm)

### EmbeddingStore

**`search_with_filter` + `delete_document` — approach:**

> `search_with_filter` thực hiện filter metadata trước khi tính similarity để giảm số lượng vector cần so sánh, giúp tăng hiệu năng. Filter dựa trên các field như `company`, `fiscal_year`.
> `delete_document` hoạt động bằng cách xóa tất cả record có cùng `doc_id` trong store in-memory và nếu dùng Chroma thì gọi delete theo metadata filter tương ứng.

---

### KnowledgeBaseAgent

**`answer` — approach:**

> Prompt được thiết kế theo dạng: system instruction + context + user query. Context được inject từ top-k retrieved chunks (thường k=3) và được format rõ ràng để model dễ đọc. Model được yêu cầu trả lời dựa *chỉ trên context*, tránh hallucination.

---

### Test Results

```
============================= test session starts =============================
collected 12 items

tests/test_chunking.py .........                              [ 75%]
tests/test_embedding_store.py ...                             [100%]

============================== 12 passed in 1.42s ==============================
```

**Số tests pass:** 12 / 12

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A                      | Sentence B                           | Dự đoán | Actual Score | Đúng? |
| ---- | ------------------------------- | ------------------------------------ | ------- | ------------ | ----- |
| 1    | Tesla produces EV cars          | Tesla manufactures electric vehicles | High    | 0.91         | Đúng  |
| 2    | Dog is running in park          | Cat sleeps on sofa                   | Low     | 0.23         | Đúng  |
| 3    | Revenue increased this year     | Company profits grew                 | High    | 0.78         | Đúng  |
| 4    | Stock price dropped             | Weather is sunny                     | Low     | 0.12         | Đúng  |
| 5    | Tesla builds factories in Texas | Tesla HQ is in Austin                | Low     | 0.55         | Sai   |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**

> Pair 5 khá bất ngờ vì mình dự đoán low nhưng score lại trung bình. Điều này cho thấy embeddings không chỉ dựa vào nghĩa trực tiếp mà còn học được mối liên hệ ngữ cảnh (Tesla + location), nên vẫn coi là somewhat related.

---

## 6. Results — Cá nhân (10 điểm)

### Kết Quả Của Tôi

| # | Query                                         | Top-1 Retrieved Chunk (tóm tắt) | Score  | Relevant? | Agent Answer (tóm tắt)           |
| - | --------------------------------------------- | ------------------------------- | ------ | --------- | -------------------------------- |
| 1 | Tesla employees end of 2025?                  | Headcount 134,785               | 0.6829 | Yes       | Tesla had 134,785 employees      |
| 2 | Tesla HQ & primary manufacturing locations?   | List factories                  | 0.6400 | Yes       | Liệt kê factories nhưng thiếu HQ |
| 3 | Tesla pay dividends?                          | Board discretion statement      | 0.6462 | Partial   | Trả lời chưa rõ ràng             |
| 4 | Tesla autonomous vehicle for robotaxi market? | Robotaxi + Cybercab             | 0.7130 | Yes       | Trả lời chính xác                |
| 5 | Tesla IP protection & EV growth?              | Patent pledge                   | 0.6442 | Yes       | Trả lời đúng                     |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**

> RecursiveChunker có thể linh hoạt hơn mình nghĩ, đặc biệt khi xử lý các tài liệu không có cấu trúc rõ ràng. Tuy chunk count cao nhưng đôi khi lại giúp retrieval chính xác hơn trong các câu hỏi chi tiết.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**

> Một số nhóm sử dụng metadata filtering rất tốt (ví dụ filter theo year + doc_type trước khi search), giúp giảm noise đáng kể và cải thiện độ chính xác của retrieval.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**

> Mình sẽ kết hợp SentenceChunker với semantic chunking (dựa trên embedding similarity) thay vì chỉ dựa vào dấu câu. Ngoài ra, sẽ enrich metadata nhiều hơn (section title, topic) để hỗ trợ retrieval tốt hơn.

---

## Tự Đánh Giá

| Tiêu chí                    | Loại    | Điểm tự đánh giá |
| --------------------------- | ------- | ---------------- |
| Warm-up                     | Cá nhân | 5 / 5            |
| Document selection          | Nhóm    | 9 / 10           |
| Chunking strategy           | Nhóm    | 14 / 15          |
| My approach                 | Cá nhân | 9 / 10           |
| Similarity predictions      | Cá nhân | 5 / 5            |
| Results                     | Cá nhân | 9 / 10           |
| Core implementation (tests) | Cá nhân | 28 / 30          |
| Demo                        | Nhóm    | 5 / 5            |
| **Tổng**                    |         | **94 / 100**     |


