# SE373 - Bài Thực Hành 5: Agent Tools & Skills Lab

Dự án triển khai và đánh giá kiến trúc Agent dựa trên LangChain, tích hợp cơ chế Tool Calling và Skill Execution qua 5 stages, giao diện Streamlit và panel State & Context.

---

## 📌 Tổng quan các Stage

| Project | Tools được cấp | Skills tích hợp | Trạng thái & Mô tả |
|---|---|---|---|
| [`stage-00-chat`](stage-00-chat/README.md) | Không có | Không có | Baseline Chatbot không có tool/skill |
| [`stage-01-files`](stage-01-files/README.md) | `read_file`, `write_file` | Không có | Agent thao tác file cục bộ, an toàn bảo mật |
| [`stage-02-skills`](stage-02-skills/README.md) | `read_file`, `write_file` | `weekly-report` | Prompt skill: Tạo báo cáo tuần theo template |
| [`stage-03-bash`](stage-03-bash/README.md) | `read_file`, `write_file`, `bash` | `weekly-report` | Cấp tool bash với sandbox an toàn |
| [`stage-04-script-skill`](stage-04-script-skill/README.md) | `read_file`, `write_file`, `bash` | `weekly-report`, `csv-quality` | Script skill: CLI `check_csv.py` lọc lỗi & kiểm tra tải |

---

## 📊 Báo cáo & Minh chứng thực nghiệm

- **Báo cáo Block 1:** [analysis.md](analysis.md) (Thực nghiệm Stage 00, 01, 02; so sánh token/latency, phân tích 5 rủi ro bảo mật file tools và giải pháp phòng chống).
- **Báo cáo Block 2:** [analysis-block2.md](analysis-block2.md) (Thực nghiệm Stage 03, 04; so sánh Prompt skill vs Script skill, phân tích rủi ro command injection/path traversal và giải pháp).
- **Minh chứng Block 2:** Thư mục [`block2-evidence/`](block2-evidence/) lưu trữ toàn bộ file JSON kết quả chạy CLI và log thực nghiệm.
- **Trace logs:** Lưu trữ chi tiết tại thư mục `traces/` của từng stage.

---

## 🚀 Hướng dẫn cài đặt & Chạy ứng dụng

### 1. Chuẩn bị môi trường
- Python 3.11+ và công cụ quản lý package [uv](https://docs.astral.sh/uv/) (hoặc `venv`).
- Cấu hình API key trong file `.env` ở thư mục gốc (hoặc từng stage):
```dotenv
OPENAI_API_KEY=your_api_key_here
MODEL_NAME=your_model_name
OPENAI_BASE_URL=https://api.groq.com/openai/v1   # nếu dùng Groq / endpoint tương thích
```

### 2. Chạy kiểm thử tự động (Unit Tests)
Toàn bộ unit test đều độc lập và chạy offline không cần API key:
```powershell
# Chạy test cho Stage 01
pytest stage-01-files/tests/

# Chạy test cho Stage 02
pytest stage-02-skills/tests/

# Chạy test cho Stage 04 (22/22 tests script check_csv & CLI)
pytest stage-04-script-skill/tests/test_check_csv.py
```

### 3. Chạy giao diện Web (Streamlit UI)
```powershell
# Khởi chạy Stage 04 (Script skill)
streamlit run stage-04-script-skill/app.py

# Khởi chạy Stage 02 (Prompt skill)
streamlit run stage-02-skills/app.py
```

---

## 🛠️ Cấu trúc thư mục

```text
SE373-BTTH5/
├── analysis.md              # Báo cáo phân tích Block 1
├── analysis-block2.md       # Báo cáo phân tích Block 2
├── block2-evidence/         # Dữ liệu JSON minh chứng chạy script Block 2
├── stage-00-chat/           # Baseline
├── stage-01-files/          # Files tools
├── stage-02-skills/         # Prompt skill (weekly-report)
├── stage-03-bash/           # Bash tool
├── stage-04-script-skill/   # Script skill (csv-quality + check_csv.py)
├── pyproject.toml           # Cấu hình dự án uv workspace
└── README.md                # Tài liệu hướng dẫn dự án
```
