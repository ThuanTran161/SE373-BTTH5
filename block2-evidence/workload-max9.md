# Báo cáo chất lượng dữ liệu và khối lượng công việc: `data/workload.csv`

Công cụ: `skills/csv-quality/scripts/check_csv.py` | ngưỡng `--max-hours`: 9.0 | exit code: 0

## Tổng quan
| Chỉ số | Giá trị |
|---|---|
| Số dòng dữ liệu (không tính header) | 6 |
| Dòng thiếu owner | 1 |
| Dòng hours không hợp lệ | 1 |
| Số task_id bị lặp (distinct) | 1 (T02) |
| Số dòng bị loại khỏi tổng giờ | 3 |

## Tổng giờ theo owner
Chỉ cộng các dòng hợp lệ (không nằm trong mục "Dòng bị loại").

| Owner | Tổng giờ | Quá tải (> 9.0 giờ)? |
|---|---|---|
| Lan | 9.0 | Không |
| Minh | 3.0 | Không |

## Người quá tải
Không có ai quá tải (tổng giờ bằng ngưỡng không tính là quá tải).

## Dòng bị loại
| Line | task_id | Lý do |
|---|---|---|
| 5 | T04 | `invalid_hours` – hours không phải số hữu hạn không âm |
| 6 | T02 | `duplicate_id` – task_id đã xuất hiện ở dòng trước (lần đầu ở line 3) |
| 7 | T05 | `missing_owner` – owner trống |

Bảng mã lý do:
| Mã | Ý nghĩa |
|---|---|
| `wrong_field_count` | Số trường khác số cột của header |
| `missing_task_id` | task_id trống |
| `duplicate_id` | task_id đã xuất hiện ở dòng trước (chỉ giữ lần đầu, kể cả khi lần đầu không hợp lệ) |
| `missing_owner` | owner trống |
| `invalid_hours` | hours không phải số hữu hạn không âm |

## Chi tiết lỗi chất lượng
| Line | Cột | Loại | task_id | Mô tả |
|---|---|---|---|---|
| 5 | hours | invalid_hours | T04 | hours 'abc' không phải số hữu hạn không âm. |
| 6 | task_id | duplicate_id | T02 | task_id T02 đã xuất hiện ở line 3. |
| 7 | owner | missing_owner | T05 | owner trống. |

## Đánh giá
- Tổng giờ ở trên chỉ phản ánh các dòng hợp lệ; 3 dòng bị loại chưa được tính.
- Dòng T05 (line 7) thiếu owner nên khối lượng của nó chưa được gán cho ai.
- Dòng T04 (line 5) có hours không hợp lệ; nếu giờ thực sự lớn, kết luận quá tải có thể thay đổi sau khi sửa dữ liệu.

## Khuyến nghị
- Line 5 (T04): sửa giá trị `hours` thành số hữu hạn không âm.
- Line 6 (T02): đặt task_id mới, khác với task_id ở line 3 (nếu là 2 công việc riêng).
- Line 7 (T05): bổ sung `owner` cho dòng này.