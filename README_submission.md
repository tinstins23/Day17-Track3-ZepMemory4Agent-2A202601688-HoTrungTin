# Lab 17 — Báo cáo nộp bài

## 3 câu thực hành

**Layer quan trọng nhất trong bộ test:** long-term (Zep Context Block + fact edges), vì 4/11 case (E02, E03, E08, E09) và phần preference của E07 phụ thuộc recall cross-session; baseline no-memory fail toàn bộ nhóm này.

**Trade-off Zep Context Block vs Redis+Qdrant:** Zep gom user graph, relevance ranking và validity range sẵn; Redis/Qdrant baseline phải tự ingest, embed, TTL và conflict rule. Đổi lại Zep là managed API (latency ~1.3s/case long-term), Redis/Qdrant cho phép kiểm soát schema chi tiết nhưng tốn công vận hành.

**Guardrail chống memory poisoning:** consent opt-in (`consent.json`), redact PII trước ingest, mọi retrieval user-scoped đúng `user_id`, heartbeat dry-run không tự thêm instruction/quyền mới, recency + provenance giữ fact cũ chỉ để audit.

## 4 câu phân tích benchmark

1. **Layer hit rate thấp nhất:** practice set đạt 11/11 (100% mọi layer). Baseline no-memory chỉ 2/11 (18.2%) — fail toàn bộ long-term, episodic, semantic, mixed.
2. **Case retrieve nhiều token nhất:** E02 long-term (~1432 token) do Context Block trả USER_SUMMARY + FACTS + edges đầy đủ cho thread mới.
3. **E07 mixed:** cần long-term (`Python` preference ORCHID-27) + semantic (`Idempotency-Key`, retry policy PAYMENT-RULE-3); budget 10/4/3/3 giữ cả hai marker trong merged context.
4. **Token reduction:** no-memory giảm ~81.8% token nhưng hit 18.2%; memory-enabled giảm ~14.2% nhưng hit 100% — cắt hết context rẻ nhưng sai evidence.

## E08 recency & E10 compaction

**E08:** sau session stage 3, BLUEBIRD-42 bắt buộc TypeScript + NestJS; preference Python vẫn đúng cho ORCHID-27. Recency + project scope override fact cũ mà không xóa lịch sử.

**E10:** sliding `max_recent_messages=4` evict filler turns nhưng durable note giữ `REVIEW-DEADLINE-1600`, `Friday`, `16:00` — compaction ưu tiên constraint/TODO, không tóm tắt văn hoa.

## Minh chứng (ảnh chụp terminal)

Chụp **4 ảnh thật** từ terminal (Win+Shift+S), lưu vào `submission/`:

| File | Lệnh chạy | Phần cần chụp |
| --- | --- | --- |
| `long_term.png` | `docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded --only-layer long_term` | Bảng kết quả **E02, E03, E08, E09** đều `PASS` |
| `episodic.png` | `... --only-layer episodic` | **E04, E05** đều `PASS` |
| `semantic.png` | `... --only-layer semantic` | **E06, E11** đều `PASS` |
| `privacy.png` | Hai lệnh liên tiếp (xem bên dưới) | Dòng `Zep user absent: True` và `Redis user keys remaining: 0` |

Privacy (chụp **một ảnh** gộp cả hai lệnh, hoặc hai ảnh ghép):

```bash
docker compose run --rm app python -m src.forget --user-id minh-lab17
docker compose run --rm app python -m src.forget --user-id minh-lab17 --verify-only
```

Report text: `reports/benchmark.md`, `reports/comparison.md`
