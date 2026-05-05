# Reflection — Lab 19

**Tên:** Đào Anh Quân
**Cohort:** A20-K1
**Path đã chạy:** lite

---

## Câu hỏi (≤ 200 chữ)

> Trên golden set 50 queries, mode nào thắng ở loại query nào (`exact` /
> `paraphrase` / `mixed`), và tại sao? Khi nào bạn **không** dùng hybrid
> (i.e. khi nào pure BM25 hoặc pure vector là lựa chọn đúng)?

Kết quả Precision@10 trên 50 golden queries:

| Mode    | exact (n=15) | paraphrase (n=15) | mixed (n=20) | Overall |
|---------|-------------|-------------------|--------------|---------|
| BM25    | **96.7%**   | 33.3%             | 97.0%        | 77.8%   |
| Vector  | 88.7%       | 24.0%             | 98.5%        | 73.2%   |
| Hybrid  | **96.7%**   | 32.0%             | **100.0%**   | **78.6%** |

**Exact queries:** BM25 và Hybrid đồng hạng nhất (96.7%). BM25 khớp term chính xác nên không cần vector; hybrid không hại nhưng cũng không giúp thêm.

**Paraphrase queries:** Cả ba mode đều kém (~24–33%) vì embedding model (`bge-small-en-v1.5`) được huấn luyện tiếng Anh, biểu diễn ngữ nghĩa tiếng Việt chưa tốt. Hybrid không cứu được khi cả hai nhánh đều yếu.

**Mixed queries:** Hybrid thắng rõ (100%) nhờ RRF kết hợp được điểm BM25 (từ khóa kỹ thuật) và vector (ý nghĩa ngữ cảnh).

**Khi không dùng hybrid:** (1) Chỉ dùng BM25 khi latency cực kỳ nhạy cảm (P99 BM25 = 4.7ms vs hybrid 19.7ms) hoặc corpus toàn thuật ngữ kỹ thuật cố định; (2) Chỉ dùng vector khi query hoàn toàn diễn đạt lại, không có từ khóa chung với tài liệu gốc, và embedding model đủ mạnh cho ngôn ngữ đích.

---

## Điều ngạc nhiên nhất khi làm lab này

Hybrid (RRF k=60) đạt 100% Precision@10 trên mixed queries — cao hơn cả pure vector (98.5%) lẫn pure BM25 (97.0%), cho thấy RRF fusion thực sự cộng hưởng hai tín hiệu bù trừ nhau thay vì chỉ lấy max. Điều bất ngờ khác là paraphrase queries thất bại đồng đều ở tất cả modes (~24–33%), nhắc nhở rằng chất lượng embedding model theo ngôn ngữ mới là nút thắt cổ chai thực sự.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: _<tên đồng đội nếu có>_
