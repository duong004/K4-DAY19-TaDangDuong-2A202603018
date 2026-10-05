# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Tạ Đăng Dương  **MSSV:** 2A202603018  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:


```

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     48.5
graph       196     91958     4606   0.00927    107.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     1.48
graph       0.89   1.67     4935       84   0.00078     2.39

```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00927 | ×8.28 |
| Indexing giây | 48.5 | 107.3 | ×2.21 |
| Mỗi câu: USD | $0.00013 | $0.00078 | ×6.00 |
| Mỗi câu: giây | 1.48 | 2.39 | ×1.61 |
| Mỗi câu: in_tok | 694 | 4935 | ×7.11 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Ở giai đoạn Indexing, chi phí USD tăng gấp 8.28 lần và thời gian tăng gấp 2.21 lần do GraphRAG phải gọi thêm 20 cuộc gọi LLM có cấu trúc JSON để bóc tách thực thể và quan hệ từ các bài báo. Khi Querying, chi phí mỗi câu hỏi của GraphRAG cao gấp 6 lần và `in_tok` cao gấp 7.11 lần do hệ thống phải nạp thêm trung bình 15–25 facts từ đồ thị Neo4j vào prompt ngữ cảnh nhằm phục vụ suy luận đa bước.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều lấy được định nghĩa tiền chất trong Luật Phòng, chống ma túy 2021. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều trích xuất đúng 2 bị cáo nhận án tử hình từ một bài báo đơn lẻ. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG bị đứt gãy thông tin chéo; GraphRAG đi chính xác từ Lê Minh Thành qua tội danh đến Điều 251 khoản 1 BLHS. |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph | Flat RAG trả về không đủ thông tin; GraphRAG tìm đúng hành vi tổ chức sử dụng và Điều 255 BLHS. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Flat RAG nhầm sang "khoản b)", GraphRAG kết nối tang vật 9,6kg MDMA xác định đúng khoản 4 Điều 250 và án tử hình. |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 1 | Graph | Flat RAG chỉ tóm tắt 3 vụ lẻ tẻ; GraphRAG gom đủ 5 vụ việc liên quan đến MDMA trên toàn bộ dữ liệu tin tức. |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E5: LLM lệch với graph (nhầm lẫn giữa các đồng phạm)

- **Hiện tượng:** Ở phiên bản baseline (`ket_qua_benchmark_kg.hint.txt`), tại câu Q3, hệ thống trả lời: *"Lê Minh Thành bị tuyên 24 tháng tù về tội tàng trữ trái phép chất ma túy. Tội này được quy định tại Điều 249..."* (Judge = 0), dù thực tế Thành bị tuyên 36 tháng tù tội mua bán ma túy.
- **Bằng chứng:** 

```cypher
MATCH (p:Person {name:'Lê Minh Thành'})-[r:INVOLVED_IN]->(k:Case)
RETURN p.name, r.role, r.charge, r.sentence, k.name;

```

```
p.name: "Lê Minh Thành"
r.role: "bị cáo"
r.charge: "mua bán trái phép chất ma túy"
r.sentence: "36 tháng tù"
k.name: "Vụ góp tiền mua ma túy tại Hà Nội"

```

* **Nguyên nhân:** Đồ thị baseline chỉ liên kết `Person -> Case`. Khi `seed_facts` quét quanh vụ án, toàn bộ đồng phạm trong vụ (trong đó có đối tượng bị 24 tháng tù tội tàng trữ) đều bị gom vào facts. Ngữ cảnh quá nhiều thông tin khiến LLM bị hoán đổi thuộc tính giữa các đối tượng.


* **Đề xuất sửa:** Bổ sung quan hệ trực tiếp `(:Person)-[:CHARGED_WITH {sentence, role}]->(:Crime)`. Khi câu hỏi nhắc đích danh tên người, truy vấn trực tiếp trách nhiệm cá nhân để cô lập thông tin, giúp câu Q3 đạt điểm tuyệt đối (Judge = 2, Recall = 1.00) ở bản cải tiến.



### Lỗi E2: Thiếu ngữ cảnh luật (sai khung hình phạt do quy tắc lọc khoản)

* **Hiện tượng:** Tại câu Q4, khi hỏi về mức phạt tù tối đa của 'Hoàng Nato', GraphRAG trả lời: *"có thể bị phạt tù tối đa 7 năm theo Điều 255 BLHS khoản 1"* (Judge = 1), trong khi khung cao nhất là 20 năm hoặc chung thân.


* **Bằng chứng:**

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS number, cl.penalty AS penalty ORDER BY number;

```

```
number: 1, penalty: "phạt tù từ 02 năm đến 07 năm"
number: 2, penalty: "phạt tù từ 07 năm đến 15 năm"
number: 3, penalty: "phạt tù từ 15 năm đến 20 năm"
number: 4, penalty: "phạt tù 20 năm hoặc tù chung thân"

```

* **Nguyên nhân:** Để tiết kiệm context window, logic `context()` ưu tiên lấy `cl.number = 1` hoặc các khoản có tang vật trùng khớp. Vì vụ việc Hoàng Nato không nêu rõ khối lượng ma túy cụ thể, hệ thống fallback về khoản 1, dẫn tới việc bỏ sót khung hình phạt tăng nặng tối đa ở khoản 4.


* **Đề xuất sửa:** Thêm quan hệ `HAS_MAX_CLAUSE` tới khoản có hình phạt cao nhất của Điều luật. Khi câu hỏi chứa từ khóa "tối đa / cao nhất", bổ sung khoản này vào facts. Đánh đổi: tăng thêm khoảng 100–150 input tokens cho mỗi câu hỏi dạng này.



## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.

> * **Khi nào Flat RAG là đủ:** Đối với các câu hỏi single-hop (Q1, Q2) mà thông tin nằm gọn trong một văn bản, Flat RAG đạt điểm tuyệt đối (Recall = 1.00, Judge = 2) nhưng chi phí vận hành rẻ hơn gấp 6 lần ($0.00013 so với $0.00078) và độ trễ thấp hơn (1.48s so với 2.39s). Flat RAG phù hợp cho tra cứu tài liệu đơn lẻ hoặc hệ thống cần tiết kiệm tối đa chi phí.
> 
> 
> * **Khi nào nên dùng KG:** Với các câu hỏi cross-KB (Q3, Q4, Q5) hoặc aggregation (Q6), Flat RAG đứt gãy hoàn toàn (Recall chỉ đạt 0.00 – 0.60, Judge chỉ 0 – 1). GraphRAG vượt trội hoàn toàn khi nâng Recall lên 0.67 – 1.00 và Judge đạt 1 – 2 nhờ khả năng đi đa bước (multi-hop) liên kết giữa các miền dữ liệu. Knowledge Graph đặc biệt đáng tiền trong các nghiệp vụ pháp lý, y tế hoặc điều tra phức tạp.
> 
> 
> 
> 

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................ [100%]
48 passed in 1.15s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 158 node / 322 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 15 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.

```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.

> Không có. Hệ thống đã vượt qua toàn bộ unit tests và hợp đồng kiểm tra kỹ thuật.