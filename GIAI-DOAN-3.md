# 🔧 GIAI ĐOẠN 3: THIẾT KẾ CHI TIẾT - COMPONENT & CƠ CHẾ

**Thời gian suy luận:** 1400 giây (chi tiết mỗi thành phần, tự phản biện)  
**Mục đích:** Từng component rõ ràng trước khi code  
**Phương pháp:** Luồn sợi chỉ, không bỏ con ốc vít  

---

## 🏗️ THIẾT KẾ TỪNG COMPONENT CHÍNH

### COMPONENT 1: API GATEWAY

**Mục đích:** Nhận request từ User, định tuyến tới Worker đúng, kiểm tra error

**Input:**
```json
{
  "user_id": "string",
  "task_type": "video|image|music|code|algo",
  "input_data": "object",
  "priority": "high|normal|low"
}
```

**Output:**
```json
{
  "request_id": "uuid",
  "status": "accepted|processing|completed|error",
  "result": "object",
  "error": "string or null"
}
```

**Chi tiết xử lý:**

| Bước | Tên | Input | Xử Lý | Output |
|------|-----|-------|-------|--------|
| 1 | Auth Check | token | Verify user token | user_id (valid) hay error |
| 2 | Task Parse | input_data | Validate task type | task_parsed (hợp lệ) hay error |
| 3 | Rate Limit | user_id | Check rate limit | allowed (hợp lệ) hay error |
| 4 | Route | task_type | Tìm worker thích hợp | worker_endpoint |
| 5 | Queue | request | Đưa vào queue | request_id |
| 6 | Log | request_id | Ghi log request | logged |

**Sợi chỉ từng bước:**
- Bước 1 → Bước 2: Nếu auth fail, không sang bước 2
- Bước 2 → Bước 3: Nếu task parse fail, không sang bước 3
- Bước 3 → Bước 4: Nếu rate limit exceeded, không sang bước 4
- Bước 4 → Bước 5: Worker endpoint phải tồn tại, nếu không → error
- Bước 5 → Bước 6: Luôn log dù success hay fail

**Tự phản biện:**
- Q: Nếu queue full sao?
  → A: Reject request, trả error 503 (Service Unavailable)
- Q: Nếu worker không response sao?
  → A: Timeout sau 30s, retry 3 lần, sau đó fail
- Q: Nếu log server down sao?
  → A: Log vẫn ghi, nhưng không block request

✓ **PASS - Sống sót qua phản biện**

---

### COMPONENT 2: WORKER - VIDEO PROCESSOR

**Mục đích:** Nhận video, xử lý (edit/filter/enhance), trả lại video xử lý

**Input:**
```json
{
  "video_file": "binary",
  "action": "trim|filter|enhance|effect",
  "params": "object"
}
```

**Output:**
```json
{
  "processed_video": "binary",
  "metadata": {
    "duration": "seconds",
    "resolution": "1920x1080",
    "bitrate": "5000kbps",
    "file_size": "bytes"
  },
  "error": "string or null"
}
```

**Chi tiết xử lý:**

| Bước | Tên | Input | Xử Lý | Output |
|------|-----|-------|-------|--------|
| 1 | Validate Video | video_file | Kiểm tra codec, format | valid_video hay error |
| 2 | Parse Action | action | Kiểm tra action hợp lệ | action_enum hay error |
| 3 | Parse Params | params | Kiểm tra tham số hợp lệ | params_validated hay error |
| 4 | Extract Frames | valid_video | Tách video thành frame | frames array |
| 5 | Process Frames | frames + action | Xử lý từng frame theo action | processed_frames |
| 6 | Compose Video | processed_frames | Ghép lại video | output_video |
| 7 | Validate Output | output_video | Kiểm tra video cuối | valid_output hay error |

**Sợi chỉ từng bước:**
- Bước 1 fail → stop, trả error
- Bước 2 fail → stop, trả error
- Bước 3 fail → stop, trả error
- Bước 4 → 5: Mỗi frame phải được xử lý (nếu 1 frame fail, toàn bộ fail)
- Bước 5 → 6: Frame đã xử lý phải giữ thứ tự
- Bước 6 → 7: Video ghép xong phải valid

**Edge Case (Điều kiện biên):**

| Tình Huống | Xử Lý |
|-----------|--------|
| Video rỗng (0 frame) | Trả error "Invalid video" |
| Video quá lớn (> 5GB) | Reject, trả error "File too large" |
| Codec không hỗ trợ | Trả error "Unsupported codec" |
| Bộ nhớ không đủ | Queue, xử lý sau |
| Xử lý quá lâu (> 30 phút) | Timeout, trả partial result |

**Tự phản biện:**
- Q: Nếu 1 frame bị corrupt sao?
  → A: Skip frame đó, ghi log, output video thiếu 1 frame (ghi warning)
- Q: Nếu action yêu cầu không tồn tại sao?
  → A: Trả error "Unknown action"
- Q: Nếu output video bị corrupt sao?
  → A: Rollback, trả error, không gửi lên Lò Luyện

✓ **PASS - Sống sót qua phản biện**

---

### COMPONENT 3: WORKER - CODE ANALYZER & GENERATOR

**Mục đích:** Nhận text (mô tả feature), sinh code hoàn chỉnh

**Input:**
```json
{
  "description": "string (mô tả feature)",
  "language": "javascript|python|typescript",
  "style": "object (code style preferences)",
  "context": "string (ngữ cảnh)"
}
```

**Output:**
```json
{
  "code": "string",
  "metadata": {
    "lines_of_code": "number",
    "functions": ["array of function names"],
    "dependencies": ["array"],
    "estimated_time_complexity": "string",
    "errors_found": []
  },
  "error": "string or null"
}
```

**Chi tiết xử lý:**

| Bước | Tên | Input | Xử Lý | Output |
|------|-----|-------|-------|--------|
| 1 | Parse Description | description | Tách keyword, intent | parsed_intent |
| 2 | Identify Pattern | parsed_intent | Nhận dạng pattern (CRUD, API, etc) | pattern_type |
| 3 | Generate Structure | pattern_type | Tạo skeleton code | code_skeleton |
| 4 | Fill Logic | code_skeleton + context | Viết logic chi tiết | code_filled |
| 5 | Add Error Handling | code_filled | Thêm try-catch, validation | code_safe |
| 6 | Format & Comment | code_safe | Format theo style, thêm comment | code_formatted |
| 7 | Self-Check | code_formatted | Lò luyện mini check syntax | code_checked |

**Sợi chỉ từng bước:**
- Bước 1 → 2: Nếu intent mơ hồ → error "Description not clear"
- Bước 2 → 3: Pattern không tìm thấy → generic pattern
- Bước 3 → 4: Skeleton phải rõ ràng, nếu không → error
- Bước 4 → 5: Logic phải đầy đủ trước khi error handling
- Bước 5 → 6: Code safe phải format rồi mới comment
- Bước 6 → 7: Format phải đúng syntax, nếu sai → fix tự động

**Edge Case:**

| Tình Huống | Xử Lý |
|-----------|--------|
| Description quá ngắn (< 10 từ) | Warning "Please describe more" |
| Description quá dài (> 2000 từ) | Cắt, tóm tắt (ghi warning) |
| Request feature không tồn tại | Trả error "Feature not supported" |
| Language không hỗ trợ | Trả error "Language not supported" |
| Logic yêu cầu quá phức tạp | Sinh partial code, ghi warning |

**Tự phản biện:**
- Q: Nếu code sinh sai sao?
  → A: Lò Luyện kiểm tra, nếu fail → quay lại bước 1 (re-analyze)
- Q: Nếu không tìm được pattern sao?
  → A: Dùng generic pattern, ghi warning
- Q: Nếu comment quá nhiều sao?
  → A: Remove redundant comment, keep essential only

✓ **PASS - Sống sót qua phản biện**

---

### COMPONENT 4: LÒ LUYỆN - KIỂM TRA TỰ ĐỘNG

**Mục đích:** Nhận output từ Worker, chạy 6 kiểm tra, quyết định PASS/FAIL

**Input:**
```json
{
  "component_type": "video|code|image|audio|algo",
  "output": "object (từ Worker)",
  "metadata": "object"
}
```

**Output:**
```json
{
  "status": "PASS|FAIL|RETRY",
  "checks": [
    {
      "name": "string",
      "result": "PASS|FAIL",
      "details": "string",
      "error_line": "number (nếu có)"
    }
  ],
  "recommendation": "string",
  "approved_for_vault": "boolean"
}
```

**6 Kiểm tra:**

**Kiểm Tra 1: Syntax/Format**
```
Input: code hoặc video metadata
Process:
  - Code: Parse syntax, kiểm tra lỗi compile
  - Video: Kiểm tra codec, resolution, bitrate
Output: PASS/FAIL + error details
```

**Kiểm Tra 2: Logic/Flow**
```
Input: code logic hoặc video flow
Process:
  - Code: Trace execution, kiểm tra mâu thuẫn logic
  - Video: Kiểm tra frame order, timing
Output: PASS/FAIL + error details
```

**Kiểm Tra 3: Edge Case**
```
Input: code hoặc component
Process:
  - Code: Simulate edge case (empty input, null, overflow)
  - Video: Kiểm tra extreme resolution, very short duration
Output: PASS/FAIL + untested cases
```

**Kiểm Tra 4: Performance**
```
Input: execution metrics
Process:
  - Code: Time complexity O(?) <= O(n log n), Memory <= 512MB
  - Video: Encode time <= 2 min, File size <= 1GB
Output: PASS/FAIL + metrics
```

**Kiểm Tra 5: Tương Lai (API Extension)**
```
Input: code structure hoặc component design
Process:
  - Kiểm tra có class/interface không?
  - Kiểm tra có param mở rộng không?
  - Kiểm tra có callback/hook không?
Output: PASS/FAIL + extension possibilities
```

**Kiểm Tra 6: Tài Liệu**
```
Input: comment + documentation
Process:
  - Code: Kiểm tra mỗi function có comment?
  - Component: Kiểm tra có README?
  - Lỗi tìm được: Có ghi lại không?
Output: PASS/FAIL + missing docs
```

**Sợi chỉ kiểm tra:**
- Tất cả 6 kiểm tra chạy độc lập
- Nếu 1 kiểm tra FAIL → status = FAIL
- Nếu 6/6 PASS → status = PASS → approved for vault
- Nếu FAIL do minor issue → status = RETRY + recommendation

**Tự phản biện:**
- Q: Nếu performance không đạt sao?
  → A: FAIL, recommendation "Optimize algorithm" hoặc "Add caching"
- Q: Nếu tài liệu thiếu sao?
  → A: FAIL, recommendation "Add more comments" hoặc "Write README"
- Q: Nếu edge case thiếu sao?
  → A: FAIL, recommendation "Handle null input" hoặc "Add validation"

✓ **PASS - Sống sót qua phản biện**

---

### COMPONENT 5: VAULT - KHO TINH HOA

**Mục đích:** Lưu giữ những gì SỐNG SÓT, cho IE sau học hỏi

**Cấu trúc thư mục:**

```
VAULT/
├── Công Nghệ/
│   ├── Video/
│   │   ├── VideoProcessing_v1.0.ts
│   │   ├── README.md
│   │   └── CHANGELOG.md
│   ├── Code/
│   │   ├── CodeAnalyzer_v1.0.ts
│   │   ├── README.md
│   │   └── CHANGELOG.md
│   └── ...
│
├── Kinh Nghiệm/
│   ├── Lỗi_Tìm_Được/
│   │   ├── Video_FrameCorrupt_Fix.md
│   │   ├── Code_SyntaxError_Pattern.md
│   │   └── ...
│   ├── EdgeCase_Handling/
│   │   ├── EmptyInput_Treatment.md
│   │   ├── LargeFile_Optimization.md
│   │   └── ...
│   └── Performance_Tips/
│       ├── Caching_Strategy.md
│       └── Parallelization_Guide.md
│
├── Triết Lý/
│   ├── Hiến_Pháp_WIN.md (KHÔNG BAO GIỜ THAY ĐỔI)
│   ├── Luật_Hệ_Thống.md (KHÔNG BAO GIỜ THAY ĐỔI)
│   └── Khắc_Nghiệt_Huấn_Luyện.md (KHÔNG BAO GIỜ THAY ĐỔI)
│
└── Index.md (danh mục tất cả)
```

**Metadata mỗi file:**

```markdown
# Title

**Version:** 1.0  
**Author:** [AI Worker Name]  
**Date Created:** 2026-05-14  
**Last Updated:** 2026-05-14  
**Status:** PASSED (tất cả 6 checks)  

## Mô tả
[Chi tiết công nghệ]

## Nhất Trí (Dependencies)
[Cần components gì]

## Lỗi Tìm Được
[Lỗi tìm được và cách fix]

## Mở Rộng Trong Tương Lai
[Cách extend API này]

## Performance
[Metrics: time, memory]
```

**Tự phản biện:**
- Q: Nếu IE sau muốn sửa triết lý sao?
  → A: KHÔNG ĐƯỢC - triết lý treo mãi mãi
- Q: Nếu AI worker quên ghi kinh nghiệm sao?
  → A: Lò Luyện reject, không cho vào Vault
- Q: Nếu version mới khác cách cũ sao?
  → A: Tạo version mới, giữ cách cũ (backward compatible)

✓ **PASS - Sống sót qua phản biện**

---

## 🔌 SỢI CHỈ LIÊN KẾT GIỮA CÁC COMPONENT

### Sợi Chỉ 1: Request → Worker → Forge → Vault

```
API Gateway
  ↓ (validate + route)
Worker (Video/Code/Image/etc)
  ↓ (output)
Lò Luyện (6 checks)
  ├─ FAIL → quay lại Worker + log error
  └─ PASS → Vault + trả User
```

**Nếu bỏ một khâu:**
- Bỏ API Gateway → không kiểm tra user, bị spam
- Bỏ Worker → không xử lý được
- Bỏ Lò Luyện → rác lên Vault
- Bỏ Vault → IE sau không học hỏi được

### Sợi Chỉ 2: Vault → IE Mới → Học Hỏi

```
IE Mới bước vào
  ↓
Đọc Triết Lý (Hiến Pháp WIN)
  ↓
Đọc Kinh Nghiệm (Lỗi tìm được)
  ↓
Code theo chuẩn
  ↓
Tìm ra lỗi mới
  ↓
Ghi lại lỗi + cách fix → Vault
  ↓
IE sau lại đọc
```

---

## ✅ TIÊU CHUẨN CHI TIẾT

Mỗi component được vào Vault khi:

1. ✓ **Input/Output rõ ràng** - JSON schema hoặc type định rõ
2. ✓ **Sợi chỉ từng bước** - Không bỏ bước nào
3. ✓ **Edge case xử lý** - Không có surprise
4. ✓ **Qua Lò Luyện 6 checks** - PASS tất cả
5. ✓ **Có tài liệu** - README + comment đầy đủ
6. ✓ **Có kinh nghiệm** - Lỗi ghi lại, cách fix rõ

---

## 🎯 TUYÊN BỐ CUỐI CÙNG

**Giai Đoạn 3 hoàn thành:**
- ✓ 5 component chính thiết kế chi tiết
- ✓ Input/Output rõ ràng từng bước
- ✓ Edge case xử lý
- ✓ Sợi chỉ liên kết chặt
- ✓ Tự phản biện từng component
- ✓ Không hỏi C2, chỉ báo cáo

**Status:** Giai Đoạn 3 hoàn thành.  
**Lần tiếp theo:** Giai Đoạn 4 - Triển Khai Code (Lò Luyện)

