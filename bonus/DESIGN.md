# Bonus — Phiên Brainstorm: Pipeline dữ liệu cho Trợ lý Bồi thường Bảo hiểm Sức khỏe (VN)

## Bài toán + ràng buộc thực

**Ai dùng.** Một công ty bảo hiểm sức khỏe ở Việt Nam (nghĩ Bảo Việt / PVI / một
insurtech) muốn một trợ lý AI trả lời hai nhóm câu hỏi: (a) cho **khách hàng** —
"hợp đồng của tôi có chi trả nội soi dạ dày không?", "quyền lợi nha khoa còn bao
nhiêu?"; (b) cho **giám định viên** — "yêu cầu bồi thường này có hợp lệ theo điều
khoản loại trừ không?". Đây không phải chatbot FAQ: câu trả lời sai về quyền lợi
là rủi ro tài chính và pháp lý thật.

**Dữ liệu gì, vì sao khó.**
- **Hợp đồng + bộ quy tắc quyền lợi**: hàng nghìn PDF tiếng Việt — nhiều bản
  **scan** (ảnh, có dấu), nhiều cột, **bảng quyền lệ lồng nhau**, phụ lục sửa đổi
  (endorsement) chồng lên hợp đồng gốc. Schema **trôi** theo từng năm sản phẩm.
- **Hồ sơ bồi thường**: dữ liệu có cấu trúc (số tiền, mã ICD-10, ngày) + ảnh hóa
  đơn/chỉ định.
- **Trace sản xuất**: mỗi lượt hỏi của trợ lý sinh cây span `gen_ai.*` (giống
  `data/traces/` của lab) — vàng để làm eval + fine-tune.

**Ràng buộc cứng.** PDPL (Luật 91/2025) — dữ liệu sức khỏe là **dữ liệu nhạy cảm**,
phải tối thiểu hóa và có cơ sở xử lý; sai quyền lợi → khiếu nại + phạt; ngân sách
token có hạn vì hỏi nhiều, biên lợi nhuận mỏng.

---

## Sơ đồ kiến trúc đề xuất

```
                         ┌─────────────────── BATCH (nightly / theo sản phẩm) ──────────────────┐
  PDF hợp đồng (scan,    │                                                                       │
  nhiều cột, tiếng Việt) │   OCR(VN)  ─▶  layout parse ─▶ extract field ─▶ GATE(Pandera) ─┬─clean─▶ Silver (điều khoản đã typed)
  + endorsement          │   (Tesseract-vi/         (bảng→rows)  (số tiền, mã,          │            │
                         │    Google DocAI)                       hiệu lực)          quarantine     ├─▶ chunk → embed → VECTOR store
                         │                                          ▲                   (DLQ)        └─▶ triples → KNOWLEDGE GRAPH
                         │                                          │ con người review                  (quyền lợi ⊂ gói ⊂ hợp đồng)
                         └──────────────────────────────────────────┘                                            │
                                                                                                                  ▼
  Câu hỏi khách / giám định ─────────────────────────────▶  Agent (retrieve: graph 1–2 hop + vector) ─▶ trả lời
                                                                          │
                         ┌──────────── NEAR-REAL-TIME (flywheel) ─────────┘
  trace gen_ai.* ───▶ Bronze spans ─▶ trace_summary (cost/latency/outcome)
                         │
                         ├─▶ eval golden set  (lượt giám định-đã-duyệt = nhãn vàng)
                         └─▶ DPO pairs (đúng vs sai cùng prompt) ─▶ DECONTAMINATE ─▶ Ngày-22 SFT/DPO
                                                                       ▲
                                            point-in-time (ASOF): quyền lợi/limit còn lại TẠI thời điểm hỏi
```

---

## Các câu hỏi mở tôi chọn (quyết định + đánh đổi)

**1) Nguồn & hình dạng — OCR + trích field thế nào?** PDF scan tiếng Việt nhiều cột
là phần khó nhất. **Quyết định: tách "đọc chữ" khỏi "hiểu cấu trúc".** Chạy
OCR có dấu (Google Document AI hoặc Tesseract-vie tinh chỉnh) → rồi một bước
layout/table parse → rồi trích field. **Đánh đổi: pipeline OCR + parser tất định
(X) vs nhét cả trang ảnh vào một VLM (Y).** Chọn X vì: rẻ hơn ~10× mỗi trang,
**deterministic → reproducible/backfill được**, và lỗi khu trú (sai 1 bảng, không
sai cả hợp đồng). Y dùng làm **fallback** cho trang OCR confidence thấp, không
làm đường chính. Schema trôi theo năm sản phẩm → giữ extractor **versioned theo
product_code**, không hard-code một layout.

**2) Hợp đồng & chất lượng — validate gì trước khi vào model?** **Quyết định:
một quality gate kiểu Pandera (đúng như `pipeline/validate.py` của lab) đứng giữa
extract và store.** Kiểm: số tiền > 0, ngày hiệu lực ≤ ngày hết hạn, mã quyền lợi
∈ từ vựng, tổng sub-limit ≤ limit gói. Dòng rớt → **quarantine/DLQ cho giám định
viên review**, không bao giờ tự vào index. **Đánh đổi: chặn cứng (X) vs cảnh báo
rồi vẫn nạp (Y).** Chọn X cho dữ liệu quyền lợi: một điều khoản trích sai âm thầm
vào RAG sẽ tạo câu trả lời sai-tự-tin — tệ hơn nhiều so với "tôi chưa có thông tin
này". Người được báo khi tỷ lệ quarantine vọt: data on-call + nghiệp vụ sản phẩm.

**3) Phi cấu trúc → RAG hay KG?** **Quyết định: lai — KG cho quan hệ bao phủ,
vector cho diễn giải ngôn ngữ.** Câu hỏi quyền lợi vốn **multi-hop**: "nội soi"
→ thuộc nhóm "chẩn đoán hình ảnh" → nằm trong "gói nâng cao" → có điều khoản chờ
30 ngày. Vector phẳng hay **tách các fact này qua nhiều chunk** và không nối được
(đúng như foil trong `kg_demo.py`). **Đánh đổi: KG (X) vs chỉ vector + chunk lớn
(Y).** Chọn X cho suy luận bao phủ/loại trừ; nhưng vector vẫn cần cho "diễn giải
điều khoản X bằng tiếng Việt dễ hiểu". Chi phí: xây KG đắt hơn (cần trích triple),
nên chỉ graph-hóa **quan hệ quyền lợi**, phần văn xuôi để vector.

**4) Train/serve parity — point-in-time ở đâu?** **Quyết định: bắt buộc ASOF
join cho mọi feature "hạn mức còn lại".** Khi build eval/training từ trace, một
lượt hỏi tháng 3 phải thấy **hạn mức nha khoa còn lại TẠI tháng 3**, không phải
giá trị hôm nay. **Đánh đổi: ASOF point-in-time (X) vs join "giá trị mới nhất"
cho gọn (Y).** Chọn X vì Y **rò rỉ tương lai**: model học rằng "khách này về sau
hết hạn mức" và ăn gian — eval đẹp lúc offline, sai khi serve. Đây chính xác là
cái `pipeline/features.py` minh họa — ở đây hậu quả là tiền thật.

**5) Flywheel — biến trace thành dữ liệu mà không tự đầu độc?** **Quyết định:
nhãn vàng = lượt đã được giám định viên DUYỆT, + decontamination cứng.** Eval set
lấy từ các quyết định bồi thường con người đã phê duyệt (ground truth thật).
DPO pairs ghép một câu trả lời được duyệt (chosen) với một câu bị giám định bác
(rejected) cho **cùng câu hỏi** — y hệt `build_preference_pairs`. **Đánh đổi: tin
nhãn người (X) vs tự-gán nhãn bằng LLM-judge (Y).** Chọn X làm nguồn vàng vì miền
pháp lý/tài chính không chịu được judge ảo giác; Y chỉ để **pre-rank** giảm tải.
Bắt buộc `decontaminate`: bỏ mọi cặp train có prompt trùng eval — bỏ bước này là
cách số 1 khiến eval **nói dối âm thầm** (điểm cao giả vì học thuộc đề).

**6) Bối cảnh Việt Nam (PDPL + tiếng Việt).** **Quyết định: PII sức khỏe không
bao giờ rời pipeline ở dạng thô để train.** Trước khi trace vào dataset fine-tune:
**redact/pseudonymize** tên, số thẻ BHYT, địa chỉ; tách định danh khỏi nội dung
y khoa. **Đánh đổi: redact tại nguồn (X) vs lọc lúc xuất (Y).** Chọn X — tối thiểu
hóa dữ liệu theo Luật 91/2025 nghĩa là **không thu cái không cần ngay từ đầu**;
lọc-lúc-xuất để sót một lần là vi phạm vĩnh viễn (model đã nhớ). Tiếng Việt có dấu
ảnh hưởng OCR và chunk: phải chuẩn hóa Unicode (tổ hợp dấu) trước embed, nếu không
"nội soi" và "nội soi" (khác mã dấu) thành hai token khác nhau.

---

## Một phương án bị loại (kèm lý do)

**Loại: "đổ thẳng PDF vào một LLM context dài, bỏ luôn lớp structured + KG".**
Nghe gọn (không cần extract, không cần gate), nhưng **bị loại** vì ba lý do:
(1) **Chi phí** — nhồi cả bộ hợp đồng + endorsement vào context mỗi câu hỏi tốn
token gấp nhiều lần một truy vấn graph/vector có chọn lọc, ở quy mô hàng nghìn
câu/ngày là không kham nổi với biên lợi nhuận bảo hiểm. (2) **Không truy vết** —
khi trợ lý nói "không chi trả", giám định viên cần **trích dẫn điều khoản nào**;
context-dump không cho ra citation ổn định. (3) **Endorsement** — phụ lục sửa đổi
phải *ghi đè* hợp đồng gốc; một LLM đọc cả hai dễ trộn lẫn điều khoản cũ/mới,
trong khi lớp structured + point-in-time giải quyết việc "điều khoản nào đang hiệu
lực" một cách tất định.

## Failure semantics & chi phí (tóm tắt)

- **Idempotent backfill**: extract khóa theo `(contract_id, page, extractor_version)`;
  chạy lại OCR không nhân đôi điều khoản. Re-embed **tăng dần theo content-hash** —
  chỉ embed lại chunk đã đổi (re-OCR một sản phẩm không phải re-embed toàn kho).
- **80% chi phí** nằm ở OCR/VLM trên ảnh scan, không phải ở LLM trả lời. Cắt bằng:
  cache OCR theo hash trang + chỉ gọi VLM-fallback cho trang confidence thấp.
- **Side-effect không đảo ngược**: dữ liệu PII đã vào checkpoint fine-tune — nên
  redact phải ở *thượng nguồn* (xem Q6), không phải bước cuối.
