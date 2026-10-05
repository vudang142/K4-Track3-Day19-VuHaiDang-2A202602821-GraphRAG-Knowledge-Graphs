# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Vũ Hải Đăng  **MSSV:** 2A202602821  **Ngày:** 5/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     80.8
graph       196     91958     4700   0.00932    147.2

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.98
graph       0.63   1.33     3374       82   0.00055     3.00
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00932 | ×8.3 |
| Indexing giây | 80.8s | 147.2s | ×1.8 |
| Mỗi câu: USD | $0.00013 | $0.00055 | ×4.2 |
| Mỗi câu: giây | 1.98s | 3.00s | ×1.5 |
| Mỗi câu: in_tok | 694 | 3374 | ×4.9 |

**Chi phí tăng thêm đến từ đâu?**
1. **Indexing:** GraphRAG gọi LLM 20 lần để trích xuất entities từ 20 bài báo (mỗi bài ~$0.0004), tăng chi phí 8 lần so với Flat RAG chỉ embed.
2. **Querying:** Prompt dài hơn do thêm dữ kiện từ graph (4.9× token), nhưng thời gian chỉ tăng 1.5× vì phần lớn là LLM inference.
3. **Chất lượng tăng:** recall từ 0.43 → 0.63 (+46%), judge từ 1.00 → 1.33 (+33%).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / - | 1.00 / - | = | Cả hai đều trả lời được vì câu hỏi chỉ cần 1 đoạn |
| Q2 | single-hop-news | 1.00 / - | 1.00 / - | = | Cả hai đều trả lời được vì câu hỏi chỉ cần 1 bài |
| Q3 | cross-kb | 0.00 / - | 0.67 / - | **Graph** | Cần thông tin từ cả tin tức (tên bị cáo, mức án) và luật (Điều 251, khung phạt) |
| Q4 | cross-kb | 0.00 / - | 0.00 / - | = | Cả hai đều không trả lời được |
| Q5 | cross-kb-multi-hop | 0.60 / - | 0.80 / - | **Graph** | Graph nối được vụ với luật và khoản phù hợp với chất MDMA |
| Q6 | aggregation | 0.00 / - | 0.33 / - | **Graph** | Graph truy vấn được các vụ liên quan đến MDMA qua Substance node |

**Quy luật:** GraphRAG thắng rõ ở các câu cross-KB (Q3, Q5, Q6) vì nó kết nối được 2 KB qua node `Crime`.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy

- **Hiện tượng:** Q4 có recall = 0.00 cho cả 2 pipeline, GraphRAG không cải thiện
- **Bằng chứng:** Bench cho thấy "Giang hồ 'Hoàng Nato'" recall = 0.00

```
Q4 flat  recall=0.00 1.11s $0.00007
Q4 graph recall=0.00 1.35s $0.00031
```

- **Nguyên nhân:** Có thể tên "Hoàng Nato" không khớp với tên trong bài báo gốc (Dương Minh Tuấn), nên vector search không tìm được doc_id đúng → graph context không có seed → không trả lời được.
- **Đề xuất sửa:** Cải thiện `link_entity` hoặc bổ sung aliases của Person vào seed_facts (hiện tại đã có aliases nhưng có thể chưa khớp).

### Lỗi E4: Phép đo recall không phản ánh đúng chất lượng

- **Hiện tượng:** Q1 recall = 1.00 cho cả 2 pipeline, nhưng câu trả lời của Flat RAG có thể thiếu ngữ cảnh đầy đủ về tiền chất
- **Bằng chứng:** Recall chỉ kiểm tra từ khóa "điều chế", "sản xuất", "danh mục tiền chất" có mặt hay không, không kiểm tra câu trả lời **đầy đủ và chính xác**

```
Q1 flat  recall=1.00 2.50s $0.00012
Q1 graph recall=1.00 3.39s $0.00038
```

- **Nguyên nhân:** `recall` chỉ đếm từ khóa, không đánh giá độ chính xác. Flat RAG có thể trả lời đúng từ khóa nhưng thiếu chi tiết từ Điều 2 PCMT.
- **Đề xuất sửa:** Dùng `judge` score (LLM chấm) thay vì chỉ recall để đánh giá. Thêm yêu cầu so sánh với đáp án chuẩn đầy đủ.

## 4. Kết luận (5 điểm)

**Flat RAG đủ khi:**
- Câu hỏi single-hop: Q1 (luật), Q2 (tin tức) → recall = 1.00, chi phí thấp hơn 4×
- Không cần kết hợp thông tin từ nhiều nguồn
- Thời gian và chi phí quan trọng hơn độ chính xác tuyệt đối

**GraphRAG đáng dùng khi:**
- Câu hỏi cross-KB: Q3, Q5, Q6 → GraphRAG recall cao hơn rõ rệt (0.00→0.67, 0.60→0.80, 0.00→0.33)
- Cần multi-hop: người → vụ → tội → điều luật → khoản
- Chi phí indexing tăng 8× nhưng chỉ làm 1 lần, sau đó mỗi câu hỏi tăng 4×

**Điểm hòa vốn:** Với ~20 câu hỏi, chi phí GraphRAG = $0.00932 + 20×$0.00055 = $0.02032, so với Flat = $0.00112 + 20×$0.00013 = $0.00372 (gấp ~5.5 lần). Chi phí tăng thêm là $0.0166. Nếu câu hỏi quan trọng và cần cross-KB, GraphRAG đáng để đánh đổi.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................ [100%]
48 passed in 0.17s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] KG-2 build_graph: 206 node / 386 cạnh
[OK] KG-3 context: trả về 13 dữ kiện cho câu hỏi về Lê Minh Thành
[OK] KG-4 GraphRAG: 2 pipeline đều chạy được
[OK] KG-4 judge: Flat 1.00 / Graph 1.00 cho Q1
[OK] KG-4 judge: Flat 1.00 / Graph 1.00 cho Q2
[OK] KG-4 judge: Flat 0.00 / Graph 0.67 cho Q3
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Ngô Việt Dung

## Vấn đề gặp phải (không tính điểm)

- Docker pull neo4j:5 bị lỗi mạng ở lần đầu → chạy lại thành công
- Không có vấn đề khác
