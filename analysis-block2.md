# Báo cáo Phân tích Block 2: Kiểm tra Quá tải Khối lượng Công việc theo Người

**Học phần:** Kỹ thuật xây dựng hệ thống Agentic AI (SE373)  
**Mô hình sử dụng (LLM):** `qwen/qwen3.8-27b` (Groq API, Base URL: `https://api.groq.com/openai/v1`)  
**Môi trường thực thi:** Windows 11, Python 3.14.6, Git Bash (`C:/Program Files/Git/bin/bash.exe`), LangChain & LangGraph  

---

## 1. Tóm tắt Thay đổi Thực hiện

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

## 2. Kết quả Kiểm thử Tự động (Automated Unit Tests)

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

## 3. Kết quả Chạy Trực tiếp Script `check_csv.py` (JSON Evidence)

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

## 4. Thí nghiệm Stage 03: Tính tổng giờ bằng Python qua Bash

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

## 5. Thí nghiệm Stage 04: Mở rộng Script và Cập nhật Skill `csv-quality`

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
  - Agent chạy lệnh: `python skills/csv-quality/scripts/check_csv.py --input data/khong-ton-tai.csv --max-hours 8``.
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
