# Lab 17 - Multi-Memory Agent Submission

## 3 Câu Phân Tích (Mục 5.2)

### 1. Layer Quan Trọng Nhất
**Long-term memory** là layer quan trọng nhất trong bộ test này vì chiếm 4/11 cases (E02, E03, E08, E09) = 20đ. Layer này lưu preferences, facts và open loops qua các thread khác nhau. Không có long-term, agent không nhớ được user thích Python hay Java, deadline của dự án, hay stack công nghệ bắt buộc cho từng project (ví dụ: BLUEBIRD-42 phải dùng TypeScript/NestJS, không phải Python).

### 2. Trade-off: Context Block / Zep vs Redis + Qdrant
**Context Block của Zep** cung cấp relevance-ranked context tự động mà không cần query thủ công. Redis+Qdrant yêu cầu vector search riêng, nhưng cho phép custom scoring và local-only deployment không phụ thuộc cloud. Zep tốt hơn cho rapid prototyping và multi-user isolation tự động; Redis+Qdrant linh hoạt hơn nhưng cần implement thêm user-scoped namespace.

### 3. Guardrail Chống Memory Poisoning
Lab này sử dụng **consent opt-in** (consent.json) trước khi ingest vào Zep, **redact PII** (email/phone) trong message content, và **heartbeat read-only** không tự thêm instruction mới. Right-to-be-forgotten (`src.forget`) xóa user-scoped data nhưng giữ shared semantic KB. Guardrail chính là policy-protected trimming trong ContextBudgetManager đảm bảo token budget không bị abuse.

## 4 Câu Phân Tích Benchmark

### 1. Layer Có Hit Rate Thấp Nhất
Không có layer nào thấp trong implementation này (11/11 PASS). Tuy nhiên, **no-memory baseline** cho thấy episodic và semantic đều 0% hit rate khi không có durable memory.

### 2. Case Retrieve Nhiều Token Nhất
**E08** (long_term: BLUEBIRD-42 stack) retrieve 1410 tokens - cao nhất trong 11 cases vì phải load đầy đủ context để trả lời về project-specific constraint.

### 3. E07 Mixed Cần Kết Hợp Memory Nào?
E07 yêu cầu cả **long-term** (để biết Minh thích Python) và **semantic** (để tìm retry policy với Idempotency-Key). Evidence bắt buộc: `Python` từ user preference và `Idempotency-Key` từ payment KB.

### 4. Token Reduction và Hit Rate
No-memory baseline có token reduction 81.8% nhưng hit rate chỉ 18.2% vì nó retrieve NOTHING. Token reduction cao mà không đi kèm hit rate tốt là false optimization - agent nhanh nhưng sai.

## E08 Recency và E10 Compaction

### E08 - Recency/Conflict
Khi Minh nói "không thích Python cho BLUEBIRD-42" (project mới), đây là **project-specific override** không xóa preference Python cho ORCHID-27 (demo cá nhân). Context Block phải trả về TypeScript/NestJS cho BLUEBIRD-42 và Python cho ORCHID-27. Recency wins nhưng scope được respect.

### E10 - Compaction
Short-term memory sử dụng **sliding window + durable notes**. 16 turns được compact thành: session summary + durable notes (REVIEW-DEADLINE-1600) + last 6 turns. Buffer không đủ vì token tăng tuyến tính; compaction giữ constraint quan trọng dù raw turn đã bị evict. Compaction ưu tiên **state, decision, TODO, constraint** chứ không phải "tóm tắt văn hóa".
