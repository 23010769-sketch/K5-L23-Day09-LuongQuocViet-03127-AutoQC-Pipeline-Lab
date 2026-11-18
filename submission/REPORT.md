# Báo cáo Day 09 — Lương Quốc Việt, 2A202603127

## 1. Kết luận trước
Pipeline kết luận **HOLD**: quét 224 ảnh, loại 31, nhận và xuất 193/224 ảnh. Sau sửa tự động và duyệt mô phỏng 80/121 ảnh trong hàng đợi, còn 0 lỗi bắt buộc nhưng 34/193 ảnh có cảnh báo chưa duyệt (17,62% ≥ 10%). `submission/export/clean_v1` là **bản nháp để thử nghiệm, chưa được phát hành**, đúng với quyết định trong `08_summary.json` và `manifest.json`.

## 2. Lỗi và phân việc
Nguồn: `02_error_list_v1.csv` gồm 216 issue: 31 lỗi cửa vào + 36 lỗi bắt buộc của QC vòng đầu + 149 cảnh báo. “Ảnh / issue” dưới đây đếm ảnh duy nhất và số bản ghi, không suy diễn mọi issue là một lỗi thật.

| Mã lỗi | Số ảnh/dòng | Ai sửa (máy/người) | Vì sao |
|---|---:|---|---|
| `BAD_SIZE` | 3 ảnh / 3 issue | Máy (intake) | Cạnh ngắn dưới 250 px; loại ảnh trước QC. |
| `CORRUPT_IMAGE` | 2 / 2 | Máy (intake) | Ảnh không mở/đọc được. |
| `DUP_IMAGE_EXACT` | 8 / 8 | Máy (intake) | Trùng từng byte. |
| `DUP_IMAGE_NEAR` | 7 / 7 | Máy (intake), cần người đối chiếu | pHash ≤ 16; có bằng chứng loại nhầm ở mục 3. |
| `LEAK_TEST` | 11 / 11 | Máy (intake) | Cờ trùng kho test/val; không dùng test để chọn ngưỡng. |
| `BAD_CLASS_ID` | 6 / 6 | Máy | Xoá dòng có ID ngoài danh sách lớp. |
| `BAD_CLASS_NAME` | 1 file `data.yaml` / 2 issue | Máy | Viết lại tên lớp; log có 1 lần `rename_classes`. |
| `BAD_FORMAT` | 3 / 3 | Máy | Xoá dòng không đúng 5 cột số. |
| `BOX_OUT_OF_IMAGE` | 8 / 8 | Máy | Cắt box; nếu hết diện tích thì xoá dòng. |
| `BOX_ZERO` | 4 / 4 | Máy | Xoá box có cạnh không dương. |
| `DUP_BOX` | 7 / 7 | Máy | Xoá dòng box lặp phía sau. |
| `MISSING_LABEL_FILE` | 6 / 6 | Người | Máy chỉ ghi `needs_label`, không tạo nhãn rỗng. |
| `EMPTY_LABEL` | 11 / 11 | Người | Xác nhận ảnh nền hay thiếu nhãn. |
| `BOX_TINY` | 4 / 6 | Người | Xem lại vật nhỏ theo guideline. |
| `BOX_HUGE` | 3 / 3 | Người | Kiểm tra box chiếm gần hết ảnh. |
| `CLASS_RARE` | Cấp dataset / 4 issue | Người | Xem phân bố lớp; không bịa/thêm box để cân bằng. |
| `POSSIBLE_MISSING` | 52 / 53 | Người | Model nghi thiếu nhãn. |
| `POSSIBLE_WRONG_CLASS` | 17 / 17 | Người | Model và nhãn khác lớp. |
| `POSSIBLE_LOOSE_BOX` | 23 / 27 | Người | Model nghi box lỏng/lệch. |
| `POSSIBLE_FALSE_POS` | 26 / 28 | Người | Nhãn không khớp dự đoán đủ tin cậy; model cũng có thể sai. |

`05_review_queue.csv` có 121 ảnh, xếp giảm dần theo tổng trọng số, có nhân `(1 + conf)` cho cảnh báo có confidence; hoà điểm thì theo tên ảnh. `img_0020` đứng đầu với 11,524 = 3×(1+0,9474) + 2×(1+0,8408) + 2, từ `WRONG_CLASS`, `MISSING` và `FIXED_DROP_LINE`; tiếp theo là 6 ảnh `NEEDS_LABEL`, mỗi ảnh 10 điểm. Duyệt mô phỏng 80/121 ảnh: gồm cả 6 ảnh thiếu nhãn và 74 ảnh mang 94/120 cảnh báo ngữ nghĩa ở v2 (52 MISSING, 17 WRONG_CLASS, 13 FALSE_POS, 12 LOOSE_BOX). Đây là số cảnh báo được đưa tới bước duyệt, không phải 94 lỗi thật đã xác nhận. V2 có 100 ảnh mang cảnh báo ngữ nghĩa; 41/121 ảnh hàng đợi chưa được duyệt.

## 3. Calibration (2 ngưỡng)
| Tham số | Khởi đầu | Đã chọn | Precision / Recall (trước → sau) | Lý do (đánh đổi) |
|---|---|---|---|---|
| `duplicates.phash_dist` | 2 | **16** | NEAR: 1,00 / 0,50 → 1,00 / 0,83; EXACT: 1,00 / 1,00 → 1,00 / 1,00 | Sweep: 4–14 chỉ đạt recall NEAR 0,67; 16 bắt 5/6 ca chèn, F1 0,91. Tuy nhiên raw có loại nhầm: giữ 16 để báo cáo lần chạy này, đề xuất thử lại 14. |
| `second_opinion.conf` | 0,25 | **0,6** | WRONG_CLASS: 0,56 / 0,83 → 0,62 / 0,83; MISSING: 0,50 / 1,00 → 0,57 / 1,00; LOOSE: 0,29 / 0,40 → 0,20 / 0,20 | So với 0,5, precision WRONG_CLASS tăng; so với 0,7–0,8, vẫn giữ recall MISSING 1,00. Đánh đổi là giảm khả năng bắt box lỏng. FALSE_POS giữ 0,31 / 1,00 vì dùng `fp_conf` riêng. |

Nguồn P/R là **các dòng sweep** trong `calibration/calibration_report.md`. Không lấy bảng mở đầu “Với cấu hình hiện tại” làm số đo sau đổi ngưỡng: các số ở đó trùng hàng 2 và 0,25 của sweep. Bộ calib chỉ có 39 ảnh, gồm 6 ca NEAR, 5 ca LOOSE và không có ca LEAK_TEST; precision 1,00 trên calib không bảo đảm an toàn trên raw.

| Chỉ số | Trước: 2 / 0,25 | Sau: 16 / 0,6 |
|---|---:|---:|
| Ảnh nhận / quét | 205/224 | 193/224 |
| Ảnh loại / quét | 19/224 | 31/224 |
| Cảnh báo QC đầu | 273 issue | 149 issue |
| Hàng đợi duyệt | 145 ảnh | 121 ảnh |
| Lỗi bắt buộc sau duyệt | 0 | 0 |
| Ảnh cảnh báo chưa duyệt / ảnh nhận | 60/205 (29,27%) | 34/193 (17,62%) |
| Quyết định | HOLD | HOLD |

Cột trước do học viên cung cấp; chưa có báo cáo lần trước để đối chiếu độc lập. Cột sau đã đối chiếu `01_intake.*`, `02_error_list_v1.csv`, `05_review_queue.csv`, `06_review_log.csv`, `07_error_list_v3.csv`, `08_summary.json` và manifest xuất. Hai ngưỡng đổi **cùng lúc** nên bảng chỉ thể hiện tác động tổng hợp, không quy mức giảm cảnh báo/hàng đợi cho riêng tham số nào. Hash cấu hình trong summary khớp file hiện tại: `b8c69ed7653ad6ea2bd1ebe1e9cc5279b0a05b11`.

Đã mở trực tiếp cả 7 cặp NEAR trong `data/raw/images/` bằng công cụ xem ảnh của AI và tính lại pHash, khớp khoảng cách trong `01_intake.csv` (cột trái là ảnh bị loại):

| Cặp bị loại ↔ đối chiếu | pHash | Quan sát |
|---|---:|---|
| `img_0011` ↔ `img_0008` | 16 | Tennis trên sân xanh ↔ hai người trượt tuyết trên núi: **khác ảnh, loại nhầm rõ ràng**. |
| `img_0084` ↔ `img_0074` | 16 | Cùng người cầm ô giữa đám đông, khác khung/cỡ: phù hợp ảnh gần trùng. |
| `img_0177` ↔ `img_0113` | 16 | Người lướt sóng ↔ người ngồi bàn: **khác ảnh, loại nhầm rõ ràng**. `img_0113` còn bị BAD_SIZE (240×226), nên sau intake cũng không được giữ. |
| `img_0076` ↔ `img_0042` | 0 | Cùng xe buýt vàng “103 ROJA”, khác cỡ: phù hợp gần trùng. |
| `img_0143` ↔ `img_0037` | 0 | Cùng xe buýt trên phố và cần cẩu đỏ, khác cỡ: phù hợp gần trùng. |
| `img_0186` ↔ `img_0061` | 10 | Cùng người trên ghế công viên, khác khung/cỡ: phù hợp gần trùng. |
| `img_0223` ↔ `img_0077` | 6 | Cùng tháp đồng hồ và xe buýt, khác khung/cỡ: phù hợp gần trùng. |

Như vậy ít nhất 2/7 cờ NEAR ở raw là sai qua quan sát, dù sweep calib báo precision 1,00. Đề xuất **thử 14** vì loại hai phép ghép có distance 16 trên, nhưng cũng có thể bỏ sót cặp đúng `img_0084`/`img_0074` và giảm recall calib từ 5/6 xuống 4/6. Chưa chạy lại ở 14; cần kiểm tra lại toàn bộ cặp NEAR, số ảnh nhận và các cờ LEAK_TEST vì thuật toán duyệt theo thứ tự có thể đổi ảnh đối chiếu. **Giữ nguyên 16 và 0,6 trong lượt này**, không dùng tập test để chọn ngưỡng.

## 4. Cửa chặn và điều pipeline KHÔNG bắt được
Đã chạy kiểm tra ngân sách 0 trong thư mục riêng để giữ nguyên lần chạy chính:

```text
python -m pipeline.run --budget 0 --work work/post_calibration_nb_20261118_41c8e6/run --reports work/post_calibration_nb_20261118_41c8e6/reports --export-dir work/post_calibration_nb_20261118_41c8e6/export
```

Kết quả: nhận 193/224 ảnh, còn **6 lỗi MISSING_LABEL_FILE**, duyệt 0 ảnh, **Rework**, mã thoát **20**; không có `export/clean_v1` trong thư mục kiểm tra. Điều này xác nhận thiếu nhãn không được máy bịa/che đi và lỗi bắt buộc ưu tiên hơn cảnh báo. Báo cáo lần kiểm tra nằm trong thư mục `reports/` cùng gốc `work/post_calibration_nb_20261118_41c8e6/`; không ghi đè `submission/reports/` hay `work/run/`.

Hai điểm mù có bằng chứng: (1) pHash không hiểu nội dung nên có thể gộp nhầm cảnh khác nhau (hai cặp ở mục 3); (2) checker bỏ sót box lỏng: ở conf 0,6, recall LOOSE trên calib chỉ 0,20 = 1/5 ca, tức bỏ sót 4/5 ca chèn. Ngoài ra FALSE_POS chỉ có precision 0,31 trên calib, nên không tự xoá nhãn theo model. Không suy diễn rằng hết cảnh báo nghĩa là nhãn sạch; cần người kiểm tra độc lập.

## 5. Train: raw vs clean (tùy chọn, chấm riêng)
Đã nhập đủ **4 lượt train Colab** từ `D:/Download/results.zip`: `raw_seed0.json`, `raw_seed1.json`, `clean_seed0.json`, `clean_seed1.json`, giữ nguyên từng byte JSON. Bản tải `results (1).zip` có cùng nội dung bốn JSON khi đối chiếu, không phải bộ train khác cần lựa chọn. Đã chạy `python scripts/summarize_results.py` để sinh lại `comparison.md`/`comparison.json`; biểu đồ `comparison.png` lấy từ cùng ZIP sau khi xác nhận hai bảng tổng hợp trong ZIP khớp bản sinh lại.

Cả bốn JSON xác nhận model khởi tạo `yolo26n.pt`, **20 epoch**, batch **16**, `imgsz=640`, seed **0 và 1**, `device="0"` (không xác định được loại GPU thực tế từ JSON). `train_images` là **224 ảnh raw / 193 ảnh clean** theo script; không có log Ultralytics để xác nhận số ảnh/nhãn raw bị bỏ qua khi train. Notebook dùng chung val `data/gold_val` và test `data/test` cho hai biến thể. Hai JSON clean có `manifest_sha1=afe11c0275f80b2a0f689290b357c5bdb6152ca2`, khớp manifest hiện tại sau chuẩn hoá CRLF→LF; cả bốn có hash cấu hình `b8c69ed7653ad6ea2bd1ebe1e9cc5279b0a05b11`, cũng khớp repo.

Kết quả từng lượt (giá trị gốc trong JSON):

| Biến thể | Seed | Ảnh train | mAP50 | mAP50–95 | Precision | Recall |
|---|---:|---:|---:|---:|---:|---:|
| raw | 0 | 224 | 0.7256 | 0.5144 | 0.8068 | 0.6291 |
| raw | 1 | 224 | 0.7259 | 0.5090 | 0.7203 | 0.6689 |
| clean | 0 | 193 | 0.7277 | 0.4994 | 0.7735 | 0.6226 |
| clean | 1 | 193 | 0.7311 | 0.4998 | 0.8138 | 0.6159 |

Bảng tổng hợp nguyên số liệu từ `submission/results/comparison.md` (TB ± độ lệch chuẩn mẫu giữa hai seed):

| biến thể | #ảnh train | số seed | map50 (TB ± độ lệch) | map50_95 (TB ± độ lệch) | precision (TB ± độ lệch) | recall (TB ± độ lệch) |
|---|---|---|---|---|---|---|
| raw | 224 | 2 | 0.726 ± 0.000 | 0.512 ± 0.004 | 0.764 ± 0.061 | 0.649 ± 0.028 |
| clean | 193 | 2 | 0.729 ± 0.002 | 0.500 ± 0.000 | 0.794 ± 0.028 | 0.619 ± 0.005 |

Chênh mAP50 (clean − raw) theo bảng là **+0.004**. Đối chiếu cùng seed: **+0.0021** ở seed 0 và **+0.0052** ở seed 1, tức tăng nhất quán về chiều qua hai seed. Tính trực tiếp từ bốn JSON, chênh trung bình là +0.00365 (0,365 điểm phần trăm); độ lệch chuẩn mAP50 raw khoảng 0.0002, clean khoảng 0.0024. Số 0.000 trong bảng là làm tròn, không có nghĩa bằng 0. Mức tăng còn nhỏ, chỉ có hai seed nên chưa đủ kết luận cải thiện ổn định hay có ý nghĩa thống kê.

Các thước đo không cùng cải thiện: mAP50–95 clean giảm ở cả hai seed (**−0.0150**, **−0.0092**, trung bình **−0.0121**); recall cũng giảm (**−0.0065**, **−0.0530**, trung bình **−0.02975**). Precision tăng trung bình khoảng +0.0301 nhưng không nhất quán: seed 0 giảm 0.0333, seed 1 tăng 0.0935. Vì vậy chỉ có bằng chứng clean tăng nhẹ mAP50 trong hai lượt này, không kết luận clean tốt hơn raw trên mọi mặt.

Hai giả thuyết cần kiểm chứng: (1) clean ít hơn 31 ảnh và có ít nhất hai ảnh bị loại nhầm do pHash (mục 3), có thể giảm độ đa dạng/recall; kiểm bằng thí nghiệm kiểm soát chỉ thay cách lọc trùng, giữ cùng model/siêu tham số/seed. (2) raw có 11 cờ LEAK_TEST ở intake; nếu là rò rỉ thật, phép đo raw có thể được lợi từ ảnh gần/trùng test/val; cần đối chiếu các cờ và so sánh với bản train đã loại rò rỉ theo quy tắc cố định, không chọn ngưỡng bằng kết quả test. Hai seed và tập test 100 ảnh theo notebook vẫn hạn chế khả năng khái quát; nên thêm seed khi có điều kiện, không bịa thêm lượt chạy.

Kết quả train **không thay đổi quyết định QC: HOLD**, vì vẫn còn **34/193 ảnh cảnh báo chưa duyệt (17,62% ≥ 10%)**. `clean_v1` tiếp tục là bản nháp để thử nghiệm.

## 6. Quyết định và ghi chú AI
Quyết định cuối: **HOLD**, giữ bản xuất là bản nháp. Cần rà soát các trường hợp loại nhầm, thử lại ngưỡng theo đề xuất trong một lượt được cho phép, duyệt/sửa cảnh báo còn lại theo quy trình và chạy lại QC. Nếu vẫn giữ mẫu số 193, phải còn tối đa 19 ảnh cảnh báo chưa duyệt (9,84%, tức giảm ít nhất 15 ảnh so với 34) và 0 lỗi bắt buộc mới đủ điều kiện Release; không thay ngưỡng gate 10% hay ngân sách quy định để ép kết luận.

- **Prompt chính đã dùng AI:** yêu cầu hoàn thiện lần lượt 4 module, đọc docstring/test trước, chỉ sửa file được chỉ định, giữ chữ ký và chạy test; sau calibration yêu cầu đối chiếu số liệu, xem cặp NEAR và hoàn thiện báo cáo mà không đổi cấu hình/dữ liệu.
- **Kiểm chứng có bằng chứng:** các lượt kiểm thử riêng đạt 14 test `label_format`, 10 test `gate`, 11 test `autofix`, 15 test `second_opinion`; file tạm autofix có dòng trống ở dòng 2 đã cắt đúng dòng 3 theo log và chạy lại không đổi; ảnh tạm có hai dự đoán chỉ sinh một `POSSIBLE_MISSING` cho dự đoán chưa có nhãn. Lượt này đã xem 7 cặp NEAR, đối chiếu CSV/summary/manifest và kiểm tra budget 0 như mục 4.
- **Một lần AI sai có ghi nhận:** thao tác PowerShell kiểm tra thư mục tạm dùng `Split-Path -LiteralPath ... -Parent` gây `ParameterBindingException`; sau đó đổi sang thuộc tính `DirectoryInfo.Parent.FullName`/`Name` và kiểm tra được. Đây là lỗi thao tác công cụ, chưa có bằng chứng để khẳng định AI từng sai thuật toán module. **Cần học viên xác nhận/bổ sung:** đã tự đọc hiểu từng dòng code, tự xem các cặp ảnh và ghi thêm một lỗi logic AI nếu thực sự đã gặp; không bịa ví dụ để điền mẫu.

Kiểm tra thực tế bằng `python scripts/check_submission.py` **gồm cả phần train** (đặt thư mục tạm pytest trong workspace): **79 passed in 83.09s**; script trả mã 1 với **2 lỗi, 0 cảnh báo**, không có lỗi kiểm tra kết quả train. Hai lỗi còn lại báo file `pipeline/calibrate.py` nhắc `MANIFEST` và `data/calib`: đã xác minh đây là lỗi đối chiếu đường dẫn trên Windows của bộ kiểm tra, vì `str(relative_to(...))` dùng dấu gạch chéo ngược còn danh sách bảo vệ dùng `/`. File `pipeline/calibrate.py` vẫn đúng bản đề, SHA1 chuẩn hoá khớp giá trị ghim `cebe2e719ae48d21f990dea95ea99739db68cfbc`; file calibration được phép đọc tập calib theo GUIDE/RULES. Không sửa script hay file bảo vệ để bỏ qua lỗi. **Cần xác nhận với giảng viên hoặc chạy lại trên Linux/Colab** để có kết quả bộ kiểm tra không vướng lỗi đường dẫn; chưa tuyên bố kiểm tra nộp bài hết lỗi.

**Cần học viên xác nhận trước khi nộp:** lưu bằng chứng lần chạy trước nếu có; link/tên repo public theo `SUBMISSION.md`, thông tin tác giả commit và nộp LMS. Chưa thực hiện commit/push trong lượt hỗ trợ này. CVAT (tùy chọn): chưa có bằng chứng đã thực hiện.
