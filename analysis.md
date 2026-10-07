# Báo cáo Phân tích Toàn diện: Tool Use & Skill Use (Block 1 & Block 2)

**Học phần:** Kỹ thuật xây dựng hệ thống Agentic AI (SE373)  
**Mô hình sử dụng (LLM):** `qwen/qwen3.8-27b` (thông qua Groq API, Base URL: `https://api.groq.com/openai/v1`)  
**Môi trường thực thi:** Windows 11, Python 3.14.6, Git Bash (`C:/Program Files/Git/bin/bash.exe`), uv venv, LangChain & LangGraph  

---

# PHẦN I: BLOCK 1 - TRA CỨU CHÍNH SÁCH HOÀN TIỀN ĐÚNG PHIÊN BẢN

## 1. Tổng quan và Thay đổi Thực hiện (Block 1)

Trong Block 1, hệ thống được nâng cấp từ phiên bản cơ bản (chỉ có `read_file` và `write_file`) lên phiên bản hỗ trợ công cụ liệt kê thư mục `list_files` cùng Skill chuyên biệt `refund-policy` để xử lý bài toán tra cứu chính sách hoàn tiền theo mốc thời gian áp dụng và cấu trúc file động.

### Bảng tóm tắt các tệp được thêm mới và cập nhật

| Tệp tin | Thư mục | Mô tả thay đổi |
|---|---|---|
| `tools/files.py` | `stage-01-files`, `stage-02-skills` | Bổ sung tool `list_files(directory)`: kiểm tra đường dẫn tương đối an toàn trong workspace, chống traversal/symlink escape, trả về danh sách file dạng JSON UTF-8. |
| `tools/__init__.py` | `stage-01-files`, `stage-02-skills` | Export `list_files` cùng với `read_file`, `write_file`. |
| `config.py` | `stage-01-files`, `stage-02-skills` | Cập nhật `CAPABILITY_TEXT` công bố công cụ `list_files`. |
| `prompts.py` | `stage-01-files`, `stage-02-skills` | Cập nhật hướng dẫn trong System Prompt về việc sử dụng `list_files` khi cần khám phá cấu trúc thư mục trước khi đọc file. Không hardcode tên file hay nội dung chính sách. |
| `agent.py` | `stage-01-files`, `stage-02-skills` | Thêm `list_files` vào mảng `TOOLS = [read_file, write_file, list_files]`. Cấu hình `max_tokens=4096`. |
| `policy-before-oct.md` | `workspace/data/policies/`, `fixtures/data/policies/` | Chính sách áp dụng cho đơn hàng trước ngày 01/10/2026: hoàn tiền trong vòng 07 ngày, bất kể trạng thái kích hoạt. |
| `policy-from-oct.md` | `workspace/data/policies/`, `fixtures/data/policies/` | Chính sách mới từ ngày 01/10/2026: hoàn tiền trong 14 ngày nếu chưa kích hoạt; nếu đã kích hoạt chỉ hoàn trong 03 ngày với phí xử lý 10%. |
| `SKILL.md` | `workspace/skills/refund-policy/`, `fixtures/skills/refund-policy/` | Hướng dẫn agent quy trình 4 bước: (1) Dùng `list_files` liệt kê `data/policies/` để phát hiện file thực tế; (2) Đọc chính sách tương ứng mốc mua; (3) Kiểm tra điều kiện (ngày yêu cầu, trạng thái kích hoạt) và hỏi lại nếu thiếu; (4) Trả lời theo mẫu chuẩn. |
| `answer-template.md` | `workspace/skills/refund-policy/references/`, `fixtures/...` | Mẫu phản hồi chuẩn gồm: Kết luận, Căn cứ chính sách, Chi tiết tính toán số ngày và Trạng thái kích hoạt. |

---

## 2. Kết quả Kiểm thử Tự động (Automated Unit Tests - Block 1)

Toàn bộ các bài kiểm thử đơn vị của `stage-01-files` và `stage-02-skills` đã được chạy qua `pytest`:

| Bộ kiểm thử | Tổng số test | Đạt (Passed) | Thất bại (Failed) | Ghi chú |
|---|---|---|---|---|
| `stage-01-files/tests` | 42 | 40 | 2 | 2 test symlink thất bại do hệ điều hành Windows yêu cầu quyền Administrator (`WinError 1314: privilege not held`). Toàn bộ 40 test logic file, bảo mật traversal, path resolution đều PASS. |
| `stage-02-skills/tests` | 49 | 47 | 2 | Tương tự, 47/49 test PASS; toàn bộ test skill catalog, observer, agent capabilities đều hoạt động chuẩn xác. |

---

## 3. Phân tích Chi tiết Các Ca Thử nghiệm Thực tế (Block 1)

Các thử nghiệm được thực thi với mô hình `qwen/qwen3.8-27b` và ghi nhận trace đầy đủ vào thư mục `traces/`.

### 3.1. Baseline: Stage 00 (Chat đơn thuần, không tool)
- **Tập tin trace:** `stage-00-chat/traces/20261007-214441_12fea2b4_turn01_47763c81.jsonl`
- **Câu hỏi người dùng:** *"Chính sách hoàn tiền của công ty hiện tại như thế nào?"*
- **Hành vi quan sát được:**
  - Mô hình không được cấp bất kỳ công cụ nào (`tools: []`).
  - Phản hồi của mô hình dựa hoàn toàn vào tri thức huấn luyện chung (parametric knowledge): đưa ra các điều khoản phỏng đoán (ví dụ: hoàn tiền trong 7-30 ngày thông thường của thương mại điện tử), hoàn toàn không thể truy cập tài liệu nội bộ trong thư mục `data/policies/`.

### 3.2. Stage 01: Bổ sung công cụ `list_files`
- **Tập tin trace:**
  - Turn 1: `stage-01-files/traces/20261007-214452_ab3faf3e_turn01_49f565d1.jsonl`
  - Turn 2: `stage-01-files/traces/20261007-214502_ab3faf3e_turn02_945882c8.jsonl`
- **Câu hỏi người dùng:**
  - Turn 1: *"Trong thư mục data/policies có những file chính sách nào?"*
  - Turn 2: *"Đọc nội dung file chính sách áp dụng từ tháng 10/2026."*
- **Hành vi quan sát được:**
  - Tại Turn 1, Agent gọi tool `list_files(directory="data/policies")`. Tool trả về danh sách gồm `policy-before-oct.md` và `policy-from-oct.md`. Agent trả lời chính xác tên hai tệp.
  - Tại Turn 2, Agent tiếp tục gọi `read_file(path="data/policies/policy-from-oct.md")` và tóm tắt chính xác nội dung chính sách mới (14 ngày chưa kích hoạt, 3 ngày nếu đã kích hoạt).

### 3.3. Stage 02 - Case A: Đơn mua trước tháng 10/2026 (Từ chối hoàn tiền)
- **Tập tin trace:** `stage-02-skills/traces/20261007-214523_49142ae9_turn01_8cb8f127.jsonl`
- **Câu hỏi người dùng:**
  > *"Tôi mua ngày 28/09/2026, yêu cầu hoàn ngày 06/10/2026, chưa kích hoạt. Tôi có được hoàn không?"*
- **Hành vi chi tiết trong trace:**
  - Agent nhận diện yêu cầu khớp với skill `refund-policy`.
  - Bước 1: Gọi `read_file("skills/refund-policy/SKILL.md")` để nạp quy trình xử lý.
  - Bước 2: Gọi `list_files(directory="data/policies")` để quét các tệp chính sách hiện có.
  - Bước 3: Đơn mua ngày 28/09/2026 thuộc giai đoạn trước 01/10/2026, Agent gọi `read_file("data/policies/policy-before-oct.md")`.
  - Bước 4: Gọi `read_file("skills/refund-policy/references/answer-template.md")` để định dạng câu trả lời.
  - **Kết quả tính toán:** Từ 28/09/2026 đến 06/10/2026 là 8 ngày. Chính sách cũ quy định tối đa 7 ngày. Trạng thái kích hoạt không ảnh hưởng đến thời hạn 7 ngày của chính sách cũ.
  - **Kết luận của Agent:** **Từ chối hoàn tiền** do đã quá hạn 8 ngày (> 7 ngày).

### 3.4. Stage 02 - Case B: Đơn mua từ tháng 10/2026 (Chấp nhận hoàn tiền)
- **Tập tin trace:** `stage-02-skills/traces/20261007-214539_8ae6a2c0_turn01_52f313dd.jsonl`
- **Câu hỏi người dùng:**
  > *"Tôi mua ngày 02/10/2026, yêu cầu hoàn ngày 12/10/2026, chưa kích hoạt. Tôi có được hoàn không?"*
- **Hành vi chi tiết trong trace:**
  - Đơn mua ngày 02/10/2026 thuộc giai đoạn từ 01/10/2026 trở đi.
  - Agent gọi `list_files` và chọn đọc file `data/policies/policy-from-oct.md`.
  - **Kết quả tính toán:** Khoảng cách thời gian là 10 ngày (12/10 - 02/10). Khóa học/dịch vụ **chưa kích hoạt**, chính sách mới cho phép hoàn trong 14 ngày (10 ngày <= 14 ngày).
  - **Kết luận của Agent:** **Đồng ý hoàn tiền 100%** do thỏa mãn điều kiện dưới 14 ngày và chưa kích hoạt dịch vụ.

### 3.5. Stage 02 - Case C: Thiếu thông tin cần thiết (Chủ động hỏi lại)
- **Tập tin trace:** `stage-02-skills/traces/20261007-214648_22198475_turn01_73345cb9.jsonl`
- **Câu hỏi người dùng:**
  > *"Khách mua ngày 02/10/2026, muốn hoàn tiền. Được không?"*
- **Hành vi chi tiết trong trace:**
  - Agent đọc `SKILL.md` và kiểm tra bộ thông tin bắt buộc: Ngày mua (`02/10/2026`), Ngày yêu cầu hoàn tiền (Thiếu), Trạng thái kích hoạt (Thiếu).
  - Vì chính sách từ tháng 10 có sự phân nhánh rõ ràng (chưa kích hoạt: 14 ngày; đã kích hoạt: 3 ngày kèm phí 10%), Agent không tự phỏng đoán hay đưa ra kết luận vội vã.
  - **Phản hồi của Agent:** Yêu cầu người dùng cung cấp bổ sung: (1) Ngày khách hàng gửi yêu cầu hoàn tiền; (2) Sản phẩm/dịch vụ đã được kích hoạt hay chưa.

### 3.6. Stage 02 - Case D: Thư mục chính sách bị đổi tên file (Khả năng thích ứng động)
- **Thực nghiệm:** Đổi tên `policy-before-oct.md` thành `chinhsachcu.md` và `policy-from-oct.md` thành `chinhsachmoi.md`.
- **Tập tin trace:** `stage-02-skills/traces/20261007-214715_0b6ced0a_turn01_495d5752.jsonl`
- **Câu hỏi người dùng:** Tương tự Case B (Mua 02/10/2026, yêu cầu 12/10/2026, chưa kích hoạt).
- **Hành vi chi tiết trong trace:**
  - Agent thực hiện quy tắc trong `SKILL.md`: bắt buộc gọi `list_files("data/policies")` trước.
  - Tool trả về: `["chinhsachcu.md", "chinhsachmoi.md"]`.
  - Nhờ có danh sách file thực tế, Agent nhận ra `chinhsachmoi.md` chính là chính sách mới, gọi `read_file("data/policies/chinhsachmoi.md")`.
  - **Kết quả:** Agent vẫn đọc đúng nội dung chính sách mới, tính toán đúng 10 ngày <= 14 ngày, và đưa ra quyết định **Đồng ý hoàn tiền** một cách chuẩn xác mà không hề bị lỗi `File Not Found` (ENOENT).

---

## 4. Trả lời Câu hỏi cuối bài (Block 1)

> **Câu hỏi cuối bài:** *Vì sao cần tool để tìm file và skill để hướng dẫn chọn chính sách? Nếu agent chưa có tool tìm file, việc sửa prompt có giải quyết được yêu cầu đổi tên file không? Giải thích.*

### 4.1. Vì sao cần tool để tìm file và skill để hướng dẫn chọn chính sách?
1. **Vai trò của Tool tìm file (`list_files`):**
   - Cung cấp năng lực quan sát thế giới thực (grounding perception). Nếu không có `list_files`, Agent bị mù về cấu trúc thư mục, chỉ có thể đoán mò tên tệp hoặc đọc các tệp được hardcode sẵn. Khi cấu trúc tệp thay đổi, Agent hoàn toàn bất lực.
   - `list_files` cho phép Agent thăm dò môi trường động trước khi thực hiện hành động đọc file.

2. **Vai trò của Skill hướng dẫn chọn chính sách (`refund-policy`):**
   - Tool `list_files` chỉ trả về danh sách thô các chuỗi tên file, nhưng không thể cho mô hình biết:
     - Logic nghiệp vụ: Căn cứ vào dòng `"Áp dụng cho ngày mua..."` bên trong nội dung tệp thay vì phán đoán cảm tính dựa trên tên tệp.
     - Cơ chế Guardrail: Khi người dùng đưa thiếu thông tin (như thiếu ngày yêu cầu hoặc trạng thái kích hoạt trong Case C), Agent phải dừng lại để hỏi làm rõ thay vì tự bịa kịch bản.
     - Chuẩn hóa đầu ra: Hướng dẫn Agent trình bày theo cấu trúc chuẩn (Căn cứ, Tính toán số ngày, Kết luận, Phí hoàn) theo `answer-template.md`.

### 4.2. Nếu agent chưa có tool tìm file, việc sửa prompt có giải quyết được yêu cầu đổi tên file không? Giải thích.
**Trả lời:** **KHÔNG THỂ giải quyết được.**

**Giải thích chi tiết:**
- System Prompt là thông tin **tĩnh** được nạp vào lúc khởi tạo phiên làm việc của Agent. Nếu sửa prompt để chỉ định một tên file mới (ví dụ ghi trong prompt là đọc `data/policies/chinhsachmoi.md`), điều này chỉ giải quyết được tạm thời cho đúng một tên file đó (hardcoding).
- Khi người quản trị tiếp tục đổi tên file thành `policy_2026_v2.md`, `chinh-sach-thang-10.md` hoặc bổ sung phiên bản mới `policy-2027.md`, Agent không có cách nào biết được tên file mới này nếu không sửa lại mã nguồn / prompt một lần nữa.
- Việc sửa prompt vi phạm nguyên tắc tách biệt giữa logic điều khiển và dữ liệu môi trường. Chỉ khi Agent được trang bị công cụ `list_files` kết hợp với quy tắc trong Skill ("quét thư mục trước, sau đó đọc nội dung file để kiểm tra ngày áp dụng"), hệ thống mới đạt được tính thích ứng động (dynamic adaptation) và hoạt động bền vững trước mọi thay đổi về tên tệp.

---
---

# PHẦN II: BLOCK 2 - KIỂM TRA QUÁ TẢI KHỐI LƯỢNG CÔNG VIỆC THEO NGƯỜI

## 1. Tóm tắt Thay đổi Thực hiện (Block 2)

Block 2 mở rộng công cụ và kỹ năng kiểm tra chất lượng file CSV sang bài toán: phân tích khối lượng công việc, tính tổng giờ theo người phụ trách (owner), phát hiện dòng vi phạm, lọc công việc bị trùng lặp mã (`task_id`), và xác định người bị quá tải dựa trên ngưỡng giờ linh hoạt `--max-hours`.

### Bảng các tệp tin được cập nhật và bổ sung

| Tệp tin | Vị trí | Mô tả chi tiết thay đổi |
|---|---|---|
| `scripts/check_csv.py` | `stage-04-script-skill/workspace/skills/csv-quality/` và bản sao trong `fixtures/skills/csv-quality/` | - Bổ sung tham số bắt buộc `--max-hours` (float >= 0), trả về exit code 2 và thông báo stderr nếu thiếu hoặc sai.<br>- Chuẩn hóa trường: `strip()` khoảng trắng đầu/cuối của `task_id`, `owner`, `hours`.<br>- Thuật toán khử trùng `task_id`: Mỗi `task_id` không rỗng chỉ giữ lần xuất hiện đầu tiên; tất cả các lần xuất hiện sau đều bị loại vào `excluded_rows` với lý do `duplicate_id` (kể cả khi lần đầu có dữ liệu không hợp lệ).<br>- Tính `hours_by_owner`: chỉ cộng dồn những dòng hợp lệ (đủ 3 cột, task_id không rỗng, owner không rỗng, hours là số hữu hạn không âm).<br>- Xác định `overloaded_owners`: lọc các owner có `total_hours > max_hours` (bằng ngưỡng không tính là quá tải).<br>- Thu thập `excluded_rows`: mỗi dòng bị loại xuất hiện đúng 1 lần với mảng `reasons` được sắp xếp theo thứ tự quy ước chuẩn: `wrong_field_count`, `missing_task_id`, `duplicate_id`, `missing_owner`, `invalid_hours`.<br>- Cấu hình `sys.stdout.reconfigure(encoding="utf-8")` để hỗ trợ xuất tiếng Việt không lỗi trên Windows. |
| `SKILL.md` | `stage-04-script-skill/workspace/skills/csv-quality/` và `fixtures/` | - Cập nhật quy tắc: Khi người dùng yêu cầu kiểm tra quá tải, phải tìm ngưỡng trong câu hỏi để truyền vào `--max-hours`.<br>- **Guardrail quan trọng:** Nếu người dùng chưa nêu ngưỡng giờ, Agent bắt buộc phải dừng lại và hỏi lại người dùng; tuyệt đối không tự đoán ngưỡng mặc định hoặc kết luận vội vã.<br>- Hướng dẫn Agent cách đọc và chuyển đổi JSON đầu ra của script thành báo cáo Markdown theo mẫu chuẩn. |
| `references/report-template.md` | `stage-04-script-skill/workspace/skills/csv-quality/` và `fixtures/` | Cập nhật cấu trúc báo cáo gồm: Tổng quan chỉ số, Tổng giờ theo owner (chỉ dòng hợp lệ), Danh sách người quá tải, Bảng chi tiết các dòng bị loại kèm bảng tra cứu mã lý do chuẩn, Chi tiết lỗi chất lượng dữ liệu, Đánh giá và Khuyến nghị. |
| `data/workload.csv` | `stage-03-bash/workspace/` và `stage-04-script-skill/workspace/` | File dữ liệu thử nghiệm chứa 6 dòng dữ liệu: có dòng lỗi `hours` ('abc'), dòng lặp `T02`, và dòng trống `owner`. |
| `data/workload-edge.csv` | `stage-04-script-skill/workspace/` và `fixtures/` | File dữ liệu kiểm tra trường hợp biên: `E01` xuất hiện đầu tiên với `hours='abc'`, dòng sau lặp lại `E01` với `hours=5`. Quy tắc yêu cầu loại bỏ dòng sau vì lặp ID, không được cộng dồn cho Lan. |
| `tests/test_check_csv.py` | `stage-04-script-skill/tests/` | Bổ sung test tự động kiểm thử toàn diện: tham số CLI `--max-hours`, logic lọc trùng ID đầu tiên không hợp lệ (`test_first_duplicate_invalid_hours_rejects_second`), thứ tự lý do trong `excluded_rows`, v.v. |
| `tools/bash.py` | `stage-03-bash/tools/` và `stage-04-script-skill/tools/` | Thích ứng môi trường Windows: tự động trỏ đến Git Bash (`C:/Program Files/Git/bin/bash.exe`), thiết lập biến môi trường `PYTHONIOENCODING=utf-8` và xử lý ngắt tiến trình an toàn. |

---

## 2. Kết quả Kiểm thử Tự động (Automated Unit Tests - Block 2)

Toàn bộ 22 bài kiểm tra chức năng của script `check_csv.py` đều đạt kết quả xuất sắc:

```text
stage-04-script-skill/tests/test_check_csv.py ...................... [100%]
22 passed in 1.25s
```

### Bảng chi tiết một số kiểm thử then chốt:
| Kiểm thử | Mục tiêu kiểm tra | Kết quả |
|---|---|---|
| `test_max_hours_required` | Kiểm tra bắt buộc phải có tham số `--max-hours` (exit code khác 0 nếu thiếu) | PASSED |
| `test_max_hours_negative_rejected` | Từ chối giá trị ngưỡng âm (`--max-hours -1`) | PASSED |
| `test_first_duplicate_invalid_hours_rejects_second` | Dòng đầu của mã lặp bị lỗi giờ, dòng sau dù hợp lệ vẫn bị loại vì trùng ID | PASSED |
| `test_overloaded_strict_greater` | So sánh quá tải: `total_hours > max_hours` (bằng ngưỡng không tính là quá tải) | PASSED |
| `test_reason_codes_ordered` | Thứ tự các mã lý do trong `excluded_rows` luôn đúng chuẩn quy định | PASSED |

---

## 3. Kết quả Chạy Trực tiếp Script `check_csv.py` (JSON Evidence - Block 2)

Script được thực thi trực tiếp bằng CLI từ thư mục `stage-04-script-skill/workspace`:

### 3.1. Chạy với `data/workload.csv`, ngưỡng `--max-hours 8`
- **Tập tin lưu kết quả:** `block2-evidence/workload-max8.json`
- **Trích đoạn kết quả JSON:**
```json
{
  "input": "workspace/data/workload.csv",
  "row_count": 6,
  "missing_owner_count": 1,
  "invalid_hours_count": 1,
  "duplicate_id_count": 1,
  "duplicate_ids": ["T02"],
  "max_hours": 8.0,
  "hours_by_owner": [
    {"owner": "Lan", "total_hours": 9.0},
    {"owner": "Minh", "total_hours": 3.0}
  ],
  "overloaded_owners": [
    {"owner": "Lan", "total_hours": 9.0}
  ],
  "excluded_rows": [
    {"line": 5, "task_id": "T04", "reasons": ["invalid_hours"]},
    {"line": 6, "task_id": "T02", "reasons": ["duplicate_id"]},
    {"line": 7, "task_id": "T05", "reasons": ["missing_owner"]}
  ]
}
```
*Nhận xét:* Tổng giờ của Lan là 9.0h (T01: 4h + T02 dòng 3: 5h). Dòng 6 (T02 lặp lại) bị loại. Với ngưỡng 8.0h, Lan bị quá tải (9.0 > 8.0).

### 3.2. Chạy với `data/workload.csv`, ngưỡng `--max-hours 9`
- **Tập tin lưu kết quả:** `block2-evidence/workload-max9.json`
- **Trích đoạn kết quả JSON:**
```json
{
  "max_hours": 9.0,
  "hours_by_owner": [
    {"owner": "Lan", "total_hours": 9.0},
    {"owner": "Minh", "total_hours": 3.0}
  ],
  "overloaded_owners": []
}
```
*Nhận xét:* Tổng giờ của Lan là 9.0h, bằng đúng ngưỡng 9.0h. Theo quy định `total_hours > max_hours`, Lan **không bị quá tải**; danh sách `overloaded_owners` là rỗng `[]`.

### 3.3. Chạy trường hợp biên với `data/workload-edge.csv`, ngưỡng `--max-hours 0`
- **Tập tin lưu kết quả:** `block2-evidence/workload-edge-max0.json`
- **Trích đoạn kết quả JSON:**
```json
{
  "input": "workspace/data/workload-edge.csv",
  "row_count": 3,
  "max_hours": 0.0,
  "hours_by_owner": [
    {"owner": "Minh", "total_hours": 0.0}
  ],
  "overloaded_owners": [],
  "excluded_rows": [
    {"line": 2, "task_id": "E01", "reasons": ["invalid_hours"]},
    {"line": 3, "task_id": "E01", "reasons": ["duplicate_id"]}
  ]
}
```
*Nhận xét:* Dòng 2 mang mã E01 bị lỗi hours 'abc'. Dòng 3 cũng mang mã E01 dù có hours=5 nhưng xuất hiện sau nên bị loại do `duplicate_id`. Lan không được cộng bất kỳ giờ nào. Minh có 0 giờ hợp lệ, bằng ngưỡng 0 nên không quá tải.

---

## 4. Thí nghiệm Stage 03: Tính tổng giờ bằng Python qua Bash (Block 2)

- **Tập tin trace:** `stage-03-bash/traces/20261007-221041_e2b9a49e_turn01_e81a5976.jsonl`
- **Yêu cầu của người dùng:**
  > *"Dùng Python qua Bash để tính tổng hours theo owner trong data/workload.csv. Không chỉnh sửa file đầu vào. Cho biết cách xử lý dòng lỗi và task_id trùng."*

### 4.1. Lệnh và Kết quả Tool trong Trace
Agent đã sử dụng công cụ `bash` để chạy một đoạn mã Python trực tiếp:
```python
import csv
from collections import defaultdict
totals = defaultdict(float)
seen = set()

with open("data/workload.csv", newline="", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for i, row in enumerate(reader, start=2):
        if len(row) != 3: continue
        task_id, owner, hours_raw = row
        if task_id in seen:
            continue  # Lọc task_id trùng: chỉ tính lần đầu tiên
        seen.add(task_id)
        try:
            hours = float(hours_raw)
        except ValueError:
            continue  # Bỏ qua dòng lỗi hours không phải số
        owner = owner.strip() or "(chưa gán owner)"
        totals[owner] += hours
```

### 4.2. Phân tích Xử lý và So sánh Kết quả
- **Kết quả tính toán:**
  - Lan: 9.0 giờ (T01 = 4h, T02 = 5h).
  - Minh: 3.0 giờ (T03 = 3h).
  - Chưa gán owner (T05): 2.0 giờ.
- **Xử lý dòng lỗi:** Dòng 5 (`T04,Minh,abc`) chứa giá trị hours không thể chuyển sang số thực. Mô hình bỏ qua hoàn toàn dòng này, không gán mặc định nhằm tránh làm lệch tổng.
- **Xử lý dòng trùng:** Dòng 3 (`T02,Lan,5`) và dòng 6 (`T02,Lan,5`) có cùng `task_id` là `T02`. Mô hình sử dụng tập hợp `seen` để ghi nhớ các `task_id` đã duyệt. Khi gặp lại `T02` ở dòng 6, câu lệnh `if task_id in seen: continue` đã loại bỏ dòng này.
- **Đối chiếu nguy cơ cộng dồn:**
  - Nếu mã không lọc trùng `task_id`, dòng 6 sẽ được cộng thêm vào tài khoản của Lan, dẫn đến tổng giờ của Lan là **14 giờ** (4 + 5 + 5).
  - Do mô hình áp dụng cơ chế lọc trùng theo ID, tổng giờ của Lan là **9 giờ**, phản ánh chính xác khối lượng thực tế.

---

## 5. Thí nghiệm Stage 04: Mở rộng Script và Cập nhật Skill `csv-quality` (Block 2)

Các thử nghiệm trong Stage 04 đánh giá khả năng kết hợp công cụ thực thi mã định sẵn và quy tắc Skill khi có tham số người dùng cung cấp.

### 5.1. Case 1: Ngưỡng 8 giờ (Có người quá tải)
- **Tập tin trace:** `stage-04-script-skill/traces/20261007-220309_b63a05f2_turn01_127ca826.jsonl`
- **Tệp báo cáo sinh ra:** `stage-04-script-skill/workspace/output/workload-max8.md` (đã sao lưu tại `block2-evidence/workload-max8.md`)
- **Câu hỏi người dùng:** *"Kiểm tra data/workload.csv, người nào vượt 8 giờ? Ghi báo cáo vào output/workload-max8.md."*
- **Hành vi quan sát trong trace:**
  1. Agent nhận diện skill `csv-quality`, gọi `read_file("skills/csv-quality/SKILL.md")`.
  2. Phát hiện ngưỡng 8 giờ trong câu hỏi, Agent thực thi lệnh Bash:
     `python skills/csv-quality/scripts/check_csv.py --input data/workload.csv --max-hours 8`
  3. Lệnh trả về exit code 0 cùng JSON phân tích.
  4. Agent đọc `report-template.md` và ánh xạ chính xác toàn bộ dữ liệu JSON vào bảng Markdown.
  5. Gọi `write_file("output/workload-max8.md", ...)` và trả lời người dùng: **Lan là người quá tải (9.0h > 8.0h)**; có 3 dòng bị loại (`T04`, `T02` lặp, `T05` thiếu owner).

### 5.2. Case 2: Ngưỡng 9 giờ (Bằng ngưỡng, không ai quá tải)
- **Tập tin trace:** `stage-04-script-skill/traces/20261007-220544_db1be6f2_turn01_c348d5c9.jsonl`
- **Tệp báo cáo sinh ra:** `stage-04-script-skill/workspace/output/workload-max9.md` (đã sao lưu tại `block2-evidence/workload-max9.md`)
- **Câu hỏi người dùng:** *"Kiểm tra data/workload.csv, người nào vượt 9 giờ? Ghi báo cáo vào output/workload-max9.md."*
- **Hành vi quan sát trong trace:**
  - Agent thực thi lệnh với `--max-hours 9`.
  - JSON trả về `overloaded_owners: []`.
  - **Kết luận của Agent:** **Không có ai quá tải**. Agent giải thích rõ ràng: Lan đạt đúng 9.0 giờ, bằng với ngưỡng quy định (`total_hours == max_hours`), do đó theo quy tắc nghiêm ngặt chỉ tính quá tải khi lớn hơn ngưỡng (`> 9.0h`), Lan không bị coi là quá tải.

### 5.3. Case 3: Thiếu ngưỡng quá tải (Kích hoạt Guardrail)
- **Tập tin trace:** `stage-04-script-skill/traces/20261007-220737_40b16f16_turn01_8f4d3160.jsonl`
- **Câu hỏi người dùng:** *"Kiểm tra xem ai bị quá tải trong data/workload.csv."*
- **Hành vi quan sát trong trace:**
  - Agent đọc `SKILL.md` và kiểm tra câu hỏi của người dùng.
  - Nhận thấy người dùng không cung cấp số giờ làm căn cứ xác định quá tải.
  - Tuân thủ hướng dẫn nghiêm ngặt trong Skill, Agent **không tự ý gọi script với ngưỡng giả định (như 8 hay 40)**, mà dừng lại và phản hồi:
    > *"Theo quy trình, để xác định ai quá tải cần biết ngưỡng giờ quy ước của bạn. Mong bạn cho biết ngưỡng quá tải (số giờ, ví dụ 'quá 8 giờ là quá tải'). Hiện bạn chưa nêu ngưỡng nên tôi chưa chạy phân tích."*

### 5.4. Case 4: Tệp không tồn tại (Xử lý lỗi thực thi)
- **Tập tin trace:** `stage-04-script-skill/traces/20261007-220749_a02f0270_turn01_8bb976a5.jsonl`
- **Câu hỏi người dùng:** *"Kiểm tra data/khong-ton-tai.csv, người nào vượt 8 giờ? Ghi báo cáo vào output/workload.md."*
- **Hành vi quan sát trong trace:**
  - Agent chạy lệnh: `python skills/csv-quality/scripts/check_csv.py --input data/khong-ton-tai.csv --max-hours 8`.
  - Script kết thúc với `exit_code: 1`, `stderr: "File không tồn tại: data/khong-ton-tai.csv"`.
  - Agent phân biệt đây là **lỗi thực thi**, không bịa số liệu, thông báo rõ ràng cho người dùng về việc tệp không tồn tại và đề xuất các tệp CSV hiện có trong workspace (`tasks.csv`, `workload.csv`, `workload-edge.csv`).

---

## 6. Trả lời Câu hỏi cuối bài (Block 2)

> **Câu hỏi cuối bài:** *Phần nào do script tính, phần nào do model diễn giải? Nếu sửa script nhưng không cập nhật skill và reference, báo cáo có thể sai hoặc thiếu thông tin gì?*

### 6.1. Phần nào do script tính, phần nào do model diễn giải?

1. **Phần do Script tính toán (Deterministic Execution):**
   - **Thống kê chất lượng thô:** Đếm tổng số dòng dữ liệu (`row_count`), dòng thiếu owner (`missing_owner_count`), dòng có hours không hợp lệ (`invalid_hours_count`), và các `task_id` bị lặp (`duplicate_id_count`).
   - **Xử lý khử trùng và loại trừ:** Duyệt từng dòng, xác định dòng đầu tiên của `task_id` để giữ lại, gắn cờ các dòng vi phạm đưa vào `excluded_rows` và sắp xếp danh sách mã lý do theo thứ tự quy ước chuẩn.
   - **Tổng hợp số học:** Tính tổng số giờ cho từng owner (`hours_by_owner`), đảm bảo chỉ cộng dồn trên các dòng hoàn toàn hợp lệ.
   - **Phân loại quá tải:** So sánh chính xác điều kiện `total_hours > max_hours` để kết luận danh sách `overloaded_owners`.

2. **Phần do Model diễn giải (Orchestration & Natural Language Processing):**
   - **Nhận diện ý định và kích hoạt Skill:** Hiểu yêu cầu từ ngôn ngữ tự nhiên của người dùng để nạp skill `csv-quality`.
   - **Trích xuất tham số:** Phân tích câu hỏi để lấy đúng con số ngưỡng giờ truyền vào tham số dòng lệnh `--max-hours` của script.
   - **Xử lý tình huống biên (Guardrail):** Nhận biết trường hợp người dùng chưa đưa ngưỡng để dừng lại hỏi làm rõ thay vì tự ý giả định.
   - **Phân biệt loại lỗi:** Nhận diện mã thoát (exit code 0 vs exit code 1) để phân biệt giữa "dữ liệu có dòng lỗi" (báo cáo bình thường) và "lỗi thực thi tệp" (báo lỗi tệp không tồn tại).
   - **Biên soạn và cấu trúc báo cáo:** Ánh xạ dữ liệu JSON máy tính được vào cấu trúc Markdown theo mẫu `report-template.md`, giải thích ngữ nghĩa số liệu cho người đọc và đề xuất giải pháp xử lý dữ liệu nguồn.

### 6.2. Nếu sửa script nhưng không cập nhật skill và reference, báo cáo có thể sai hoặc thiếu thông tin gì?

1. **Lỗi thực thi do sai tham số dòng lệnh:**
   - Script mới yêu cầu bắt buộc tham số `--max-hours`. Nếu Skill không được cập nhật, Agent sẽ tiếp tục chạy lệnh cũ (`python check_csv.py --input ...`), dẫn đến script trả về exit code 2 và tiến trình hoàn toàn thất bại.

2. **Thiếu thông tin nghiêm trọng trong báo cáo:**
   - Nếu `report-template.md` không được cập nhật theo schema mới của script, báo cáo sinh ra sẽ thiếu:
     - Ngưỡng giờ áp dụng (`max_hours`).
     - Bảng tổng giờ hợp lệ theo từng nhân sự (`hours_by_owner`).
     - Danh sách chính xác những người quá tải (`overloaded_owners`).
     - Bảng chi tiết các dòng bị loại khỏi tổng giờ kèm các mã lý do chuẩn hóa (`excluded_rows`).

3. **Nguy cơ sai lệch logic do mô hình tự phỏng đoán:**
   - Nếu Skill không có quy tắc nghiêm ngặt về việc hỏi lại khi thiếu ngưỡng, Agent sẽ tự chọn một ngưỡng ngẫu nhiên (ví dụ 8h hoặc 40h) hoặc lấy lại ngưỡng từ cuộc trò chuyện trước, dẫn đến việc kết luận "ai quá tải" bị sai lệch hoàn toàn so với ý định thực sự của người dùng.
   - Model có thể tự cộng dồn cả những dòng vi phạm (như dòng trùng T02 hay dòng thiếu owner) vào tổng giờ nếu template không hướng dẫn rõ ràng việc chỉ lấy số liệu từ JSON của script.

---

## 7. Bằng chứng Thực thi (Traces & Evidence)

Toàn bộ các tệp bằng chứng thực tế được lưu trữ đầy đủ trong dự án:
- Traces Block 1: `stage-00-chat/traces/`, `stage-01-files/traces/`, `stage-02-skills/traces/`
- Traces Block 2: `stage-03-bash/traces/`, `stage-04-script-skill/traces/`
- JSON & Markdown Evidence: `block2-evidence/`
- **Bảo mật:** Không có bất kỳ API Key, chuỗi bí mật hay tệp `.env` nào được đưa vào báo cáo hoặc commit.
