# PBI Automation — Report Pipeline

Sinh báo cáo Power BI (.pbip) từ một form yêu cầu (requirement form)

---

## 1. Mục tiêu (goal)

Khách điền form 10 ô → hệ thống trả về một dự án `.pbip` mở được trong Power BI Desktop, có measure đúng nghiệp vụ và bố cục (layout) khách đã duyệt qua ảnh preview.

- **Model**: bảng, quan hệ, measure — phần "tính toán".
- **Report**: trang, biểu đồ, bố cục — phần "hiển thị".


## 3. Các bước thực hiện (steps)

| # | Bước | Ai làm | Kết quả |
| --- | --- | --- | --- |
| 1 | Khách điền form 10 ô, chọn file dữ liệu | Khách | `form.json` |
| 2 | Quét file dữ liệu, lấy tên bảng, cột, kiểu dữ liệu | Python | danh sách cột |
| 3 | Hoàn thiện form: tự điền ô thiếu, ghi lý do | AI 1 | `requirement.json` |

| **Lớp Model** | | | |
| 4 | Thiết kế model: bảng, quan hệ, bảng Date, cột cần biến đổi, measure | AI 2 | `model_spec.json` |
| 5 | Ghi file model | Python | `SemanticModel/*.tmdl` |

| **Lớp Report** | | | |
| 6 | Đề xuất 2–3 phương án bố cục | AI 3 | `layout.json` |
| 7 | Vẽ ảnh preview, khách chọn 1 phương án | Python + Khách | `preview/*.png` |
| 8 | Ghi file report | Python | `Report/pages/...` |

| **Kiểm tra** | | | |
| 9 | Kiểm tra lỗi, mở Desktop, Refresh | Python  | báo cáo hoàn chỉnh |

Có lỗi ở bước nào thì quay lại AI của bước đó để sửa. Không ghi file khi còn lỗi.

## 4. Kiến trúc pipeline (architecture)

```
Form + file dữ liệu
      │
      ▼
[Python] quét cột ──► [AI 1] hoàn thiện form
                              │
             ┌────────────────┴────────────────┐
             ▼                                 ▼
      LỚP MODEL                          LỚP REPORT
[AI 2] thiết kế model              [AI 3] đề xuất bố cục
      │                                        │
[Python] ghi TMDL ──── tên cột, measure ──► [Python] vẽ preview → khách chọn
                                               │
                                        [Python] ghi PBIR
                                               │
                                               ▼
                              [Python] kiểm tra → mở Desktop → Refresh
```

Lớp Model làm trước. Lớp Report chỉ được dùng tên cột và measure mà lớp Model đã tạo.

## 5. Cấu trúc thư mục (folder structure)

```
pbi-automation/
├── README.md
├── src/                      ← code
├── templates/                ← khuôn mẫu lấy từ Desktop
├── samples/base_project/     ← dự án .pbip mẫu để test
└── runs/<job>/               ← kết quả mỗi lần chạy
```

| File | Để ở đâu | Làm gì | Bước |
| --- | --- | --- | --- |
| `models.py` | `src/` | Định nghĩa mẫu JSON cho form, model, layout | Tất cả |
| `scan_data.py` *(chưa có)* | `src/` | Quét file dữ liệu, lấy tên cột | 2 |
| `ai_steps.py` | `src/` | 3 lần gọi AI | 3, 4, 6 |
| `model_writer.py` *(chưa có)* | `src/` | Ghi bảng, quan hệ, code Power Query vào TMDL | 5 |
| `pbir_writer.py` | `src/` | Ghi measure; kiểm tra lỗi; ghi trang và biểu đồ | 5, 8, 9 |
| `preview.py` | `src/` | Vẽ ảnh preview | 7 |
| `pipeline.py` *(chưa có)* | `src/` | Chạy tất cả các bước bằng 1 lệnh | 1–9 |
| `visuals/*.json`, `page.json` | `templates/` | Khuôn biểu đồ và trang, copy từ Desktop | 8 |
| `form.json`, `requirement.json` | `runs/<job>/` | Yêu cầu của khách | 1, 3 |
| `model_spec.json`, `layout.json` | `runs/<job>/` | Thiết kế do AI đề xuất | 4, 6 |
| `preview/*.png` | `runs/<job>/` | Ảnh cho khách chọn | 7 |
| `output/` | `runs/<job>/` | Dự án `.pbip` hoàn chỉnh | 5, 8, 9 |

## 6. Chạy pipeline bằng notebook




Chạy lần lượt các notebook trong `notebooks/`:

1. `01_intake_scan.ipynb` — điền `FORM`, đặt `DATA_PATH`; quét bảng/cột và tạo prompt AI 1. Dán JSON AI trả về để lưu `requirement.json`.
2. `02_model_design.ipynb` — tạo prompt AI 2; dán kết quả để kiểm tra và lưu `model_spec.json`.
3. `03_report_layout_preview.ipynb` — tạo prompt AI 3; dán 2–3 phương án, chọn layout và lưu preview cùng `layout.json`.
4. `04_build_validate.ipynb` — tạo prompt yêu cầu file TMDL/PBIR, dán JSON `project_files` để lắp ráp `.pbip` trong `runs/<job>/output/`.
5. `05_build_PBI_app.ipynb` — sinh TMDL/PBIR từ `model_spec.json` và `layout.json`, sau đó đóng gói `.pbip`. Có thể dùng notebook này thay bước build thủ công ở notebook 04.

Các notebook không gọi dịch vụ AI tự động; người dùng chủ động chuyển prompt và phản hồi qua công cụ AI được phép. Notebook 05 sinh TMDL trực tiếp và dựng PBIR dựa trên visual templates từ Power BI Desktop. Cần kiểm tra giả định/DAX và mở project bằng Power BI Desktop để Refresh trước khi bàn giao; kiểm tra notebook không thay thế xác nhận trong Desktop.

Notebook 05 tạo semantic model TMDL trực tiếp. PBIR visual cần template đã export từ Power BI Desktop và lưu tại `templates/pbir/visuals/`; template phải chứa marker field theo hướng dẫn trong notebook. Các visual JSON mẫu hiện có chưa có field binding nên không dùng trực tiếp làm template hoàn chỉnh.