# Reflection — Lab 19

**Tên:** Nguyễn Cao Nam
**Cohort:** A20
**Path đã chạy:** lite

---

## Câu hỏi (≤ 200 chữ)

> Trên golden set 50 queries, mode nào thắng ở loại query nào (`exact` /
> `paraphrase` / `mixed`), và tại sao? Khi nào bạn **không** dùng hybrid
> (i.e. khi nào pure BM25 hoặc pure vector là lựa chọn đúng)?

- **Exact queries:** BM25 thắng hoặc ngang ngửa do khớp chính xác từ khóa đặc thù (mã lỗi, tên riêng, thuật ngữ kỹ thuật) mà embedding model có thể làm mờ nghĩa.
- **Paraphrase queries:** Semantic (Vector) thắng áp đảo nhờ nắm bắt được ngữ nghĩa ngầm và ý định người dùng khi không trùng từ vựng gốc.
- **Mixed queries & Tổng thể:** Hybrid (RRF k=60) thắng vì kết hợp được cả tín hiệu từ vựng chính xác lẫn ngữ nghĩa, bù trừ điểm mù của từng phương pháp.

**Khi không dùng Hybrid:**
1. **Dùng pure BM25:** Khi tìm kiếm mã định danh, số serial, log code, hoặc cần độ trễ cực thấp (sub-millisecond) với tài nguyên tối thiểu (không cần tính vector embedding).
2. **Dùng pure Vector:** Khi tìm kiếm đa ngôn ngữ (cross-lingual), dữ liệu đa phương tiện, hoặc khi truy vấn trừu tượng không có từ khóa cố định.

---

## Điều ngạc nhiên nhất khi làm lab này

Thuật toán RRF tuy đơn giản nhưng mang lại hiệu quả vượt trội cho độ chính xác tìm kiếm; việc chạy Qdrant in-memory và Feast SQLite giúp toàn bộ stack hoạt động trơn tru và nhanh chóng trên môi trường local.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: _<tên đồng đội nếu có>_
