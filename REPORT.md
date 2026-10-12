# REPORT — Lab Day 9

> Điền **bằng lời và số liệu của chính bạn** (lấy từ đầu ra của `scripts/lab.sh step ...` và từ `report/evidence/*.json`). Bài làm công khai trên GitHub; giảng viên đối chiếu với lịch sử commit và với bài của các bạn khác. Mỗi câu 3 đến 8 dòng là đủ.

| | |
|---|---|
| Họ và tên | Vũ Đức Thiện |
| MSSV (đúng bằng `LAB_STUDENT_ID` trong `.env`) | `2A202602437` |
| GitHub | `vuthien3002-sys` |
| Trip dùng cho bài tập sập và chạy lại (xem GUIDE, bước 5) | `trip-03904` |

> Cách tính trip: MSSV có chữ `A` nên `int(MSSV)` của GUIDE không chạy được; tôi bỏ chữ `A` rồi tính `2202602437 % 26 = 9`, phần tử thứ 9 (đếm từ 0) của `scripts/lab.sh cli list-trips` là `trip-03904`.
>
> Môi trường: máy tôi dùng (máy phòng net, có phần mềm đóng băng ổ đĩa) không bật được WSL2 nên không cài được Docker Desktop. Tôi chạy toàn bộ lab trên **GitHub Codespaces** (4 core, 16 GB RAM, Docker có sẵn). Log đầu ra của từng bước được lưu ở `report/run_logs/`.

## Kết quả cuối

- `dataset_version` của bạn: `247152d8c2e5`  (phải khớp `report/evidence/run_summary.json`)
- Số khung hình train / val / test của bạn: `689 / 140 / 190` (tổng 1019 khung; 18 / 4 / 4 trip)
- `report/verify.txt`: tạo bằng `python3 scripts/check_submission.py --save-verify` (phải có `27/27 checks passed`).
- Ảnh chụp trong `report/screenshots/`: `airflow_graph.png`, `minio_lake.png`, `metabase_dashboard.png`; bài tập sập/chạy lại: `crash_task_log_do.png` (log task đỏ, thấy `trip-03904`), `crash_graph_do.png`, `crash_clear_dialog.png`, `crash_graph_xanh_sau_clear.png`.

## Bước 1 · Bronze

### Câu 1
Liệt kê **ít nhất 4 khác biệt** giữa telemetry nguồn A và nguồn B mà bạn thấy ở bronze. Vì sao pipeline giữ **mỗi nguồn một bảng bronze** và để mọi cột ở dạng chữ, thay vì gộp và ép kiểu ngay?

Khác biệt tôi thấy ở hai dòng đầu của `raw_telemetry_a` (3021 dòng) và `raw_telemetry_b` (3234 dòng):
1. **Tên cột khác**: `ts_utc / speed_mps / heading_deg / lat / lon / alt_m / acc_x / gas / brake` của A so với `timestamp_ms / spd_kmh / hdg / latitude / longitude / altitude / ax / throttle_pct / brake_pct` của B.
2. **Kiểu thời gian**: A là chuỗi ISO UTC (`2025-10-07T08:00:00.014Z`), B là epoch mili giây (`1759933801509`).
3. **Đơn vị tốc độ**: A là m/s (`1.213`), B là km/h (`55.11`).
4. **Bàn đạp**: A là tỉ lệ 0–1 (`gas 0.028`), B là phần trăm (`throttle_pct 2.0`).
5. **Đồng hồ thiết bị**: `bronze_video_files` ghi `telemetry_clock_offset_ms` = 0 cho drive_a và **1500** cho drive_b, tức đồng hồ của B chạy sớm 1,5 s so với video.

Mọi cột đều là `VARCHAR`, kể cả 30 ô tốc độ trống của A (`typeof(speed_mps)` = VARCHAR cho cả 3021 dòng). Bronze là bản ghi **nguyên bản, chỉ thêm**. Nếu ép kiểu hoặc gộp ngay, các ô trống hay giá trị rác sẽ lỗi hoặc thành NULL một cách im lặng, và ta mất bằng chứng về dữ liệu gốc. Mỗi nguồn có schema riêng và có thể "trôi" (drift) độc lập, nên tách bảng giúp nạp được kể cả khi một nguồn đổi schema. Khi logic làm sạch ở silver sai, ta luôn chạy lại được từ bronze. Nạp lần hai báo `already present` vì file được nhận diện bằng sha256, và `cli history bronze.raw_telemetry_a` chỉ có **một commit** (version 0, `rows_added` 3021).

## Bước 2 · Silver telemetry

### Câu 2
Pipeline đưa hai nguồn về một schema như thế nào (đơn vị, kiểu thời gian, **độ lệch đồng hồ**)? Bằng chứng nào trong kết quả bước 2 cho thấy đã đúng? Dán bảng quarantine của bạn (số dòng theo từng lý do, từng drive) và giải thích vì sao dòng lỗi vào `rejects` thay vì bị xoá.

`stg_telemetry_b.sql` đổi tên cột về schema của A, tính `spd_kmh / 3.6` → m/s, `throttle_pct / 100` → 0–1. Thời gian được tính bằng `to_timestamp((timestamp_ms − offset_ms) / 1000)`, trong đó `offset_ms` = 1500 lấy từ `bronze.video_files` (gốc ở `sources.json`). Như vậy độ lệch đồng hồ được trừ bằng **giá trị đã khai báo**, không đoán.

Bằng chứng: thời điểm bắt đầu của B là `2025-10-08 14:30:00.009`, khớp giờ bắt đầu video `14:30:00` (verify yêu cầu lệch < 0,5 s). Tốc độ tối đa sau chuẩn hoá là 71 km/h (A) và 100 km/h (B), hợp lý cho xe chạy đường phố. Đối soát số dòng: sạch + cách ly = nhận về, tức 2968 + 53 = 3021 (A) và 3181 + 53 = 3234 (B).

| drive | duplicate | speed_missing | speed_out_of_range | tổng |
|---|---|---|---|---|
| drive_a | 20 | 30 | 3 | 53 |
| drive_b | 20 | 30 | 3 | 53 |

Dòng lỗi vào `silver_telemetry_rejects` kèm `reject_reason`, không bị xoá. Mỗi dòng bị cách ly giữ nguyên `_batch_id` và `_row_no`, nên truy ngược được về đúng dòng trong file gốc. Ví dụ, `select * from silver.silver_telemetry_rejects limit 5` cho thấy dòng `_row_no 400` của drive_a có `speed_mps = 250.0` (900 km/h, lý do `speed_out_of_range`), dòng 330 bị `duplicate`, còn dòng 1204 và 1211 bị `speed_missing`. Giữ dòng lỗi là để còn **đối soát được** (verify kiểm tra đúng phương trình trên) và **truy được nguồn lỗi** (ví dụ thiết bị nào hay gửi trùng). Nếu một luật lọc sai, ta xem lại hoặc khôi phục được các dòng đó. Xoá im lặng sẽ làm mất dữ liệu mà không ai biết.

### Câu 3
Vì sao test `max_gap_ms` là `warn` còn `in_range` là `error`? Kết quả thí nghiệm `break-gate` của bạn (`break_gate.json`): test nào bắt được, bao nhiêu dòng, và điều đó cho biết gì về vai trò của "cổng chất lượng"?

`max_gap_ms` (khoảng trống > 1000 ms giữa hai mẫu) đo hiện tượng **có thật** của cảm biến: mất tín hiệu, xe vào hầm. Dữ liệu vẫn dùng được, và các khung không có telemetry gần đã bị gắn cờ `sync_ok = false` riêng (7 khung mỗi drive). Dừng cả pipeline vì một điều kiện tự nhiên là quá tay, nên test này để mức `warn`: báo cho người vận hành nhưng vẫn chạy tiếp (bước 2 cho `PASS=6 WARN=1`). Ngược lại, `in_range` (0–70 m/s) kiểm một giới hạn **vật lý**: tốc độ > 70 m/s (252 km/h) chắc chắn là dữ liệu hỏng, và nếu lọt vào gold nó sẽ làm sai nhãn huấn luyện. Vì vậy test này để mức `error`, và trong DAG nó chặn mọi task phía sau.

Thí nghiệm `break-gate`: nới luật làm sạch lên `relaxed_max_speed_mps = 1000`, thì **6 dòng** tốc độ vô lý (3 + 3 dòng `speed_out_of_range` của hai drive) lọt vào silver. Test `in_range_silver_telemetry_speed_mps__70__0` bắt được đúng **6 dòng**, kết quả `PASS=5 WARN=1 ERROR=1`, `gate_failed: true`. Khôi phục luật thì kết quả về `PASS=6 WARN=1 ERROR=0`.

Bài học: luật làm sạch và test là **hai lớp độc lập**. Khi ai đó vô tình nới luật ở bước làm sạch, cổng chất lượng vẫn chặn dữ liệu xấu trước khi tới gold.

## Bước 3 · Silver frames

### Câu 4
Dựa vào `idempotency.json`: số dòng trước và sau hai lần chạy lại, và khi ghi kiểu `append`. Vì sao Delta `overwrite` theo partition (một trip) an toàn hơn xoá rồi ghi lại, nhất là khi giữa chừng có lỗi?

Tôi chạy lại partition `trip-00003` (60 khung) hai lần:
- **Trước khi chạy lại**: bảng có **1246** dòng, dấu vân tay `6c7480dfde7e`.
- **Sau hai lần chạy lại**: vẫn **1246** dòng, dấu vân tay vẫn `6c7480dfde7e` (`unchanged: true`).
- **Ghi kiểu `append`**: trip đó thành **120** dòng, tức nhân đôi (2x).

`cli history silver.frames` cho thấy mỗi lần chạy lại là **một commit** `WRITE` với `predicate: trip_id = 'trip-00003'` và `rows_added: 60` (version 26 và 27). Overwrite theo predicate thay phần dữ liệu của trip đó **trong một giao dịch nguyên tử**: việc gỡ file cũ và thêm file mới được ghi chung vào một entry của transaction log. Người đọc chỉ thấy phiên bản cũ hoặc phiên bản mới, không bao giờ thấy trạng thái nửa vời. Nếu tiến trình chết giữa chừng thì chưa có commit nào, bảng vẫn ở version trước.

Với cách "xoá rồi ghi" thì có hai bước tách rời. Nếu sập sau khi xoá, trip đó mất dữ liệu, và trong khoảng giữa hai bước người đọc thấy bảng thiếu trip. Delta còn giữ lịch sử nên có thể time travel về version trước nếu cần.

### Câu 5
Khung hình **đen** và khung hình **tối** khác nhau thế nào, pipeline xử lý khác nhau ra sao, bạn có đồng ý không? Dựa vào `threshold.json`: nếu chọn ngưỡng 0, 2, 6 thì bao nhiêu khung bị coi là trùng ở mỗi drive? Bạn chọn ngưỡng nào, vì sao, và ngưỡng đó phụ thuộc vào tốc độ xe thế nào?

| dHash khác tối đa | số khung trùng, drive A | số khung trùng, drive B |
|---|---|---|
| 0 | 61 | 8 |
| 2 | 149 | 53 |
| 6 | 363 | 244 |

Khung **đen** có độ sáng trung bình < 12 (`dark_luma`). Đó là camera hỏng hoặc che, gần như không có thông tin, nên bị **loại** (A có 6 khung, B có 4, verify bắt đủ 6/6 và 4/4). Khung **tối** có độ sáng < 40 (`dark_soft`): đường hầm, bóng râm, chiều tối. Đây là **điều kiện lái thật** mà mô hình phải học, nên chỉ **gắn cờ và giữ lại** (B giữ 61 khung, A giữ 4). Tôi đồng ý với cách này: loại khung tối sẽ khiến mô hình kém đúng ở những tình huống khó nhất. Cờ `dark` vẫn hữu ích để kiểm tra mỗi split có đủ khung tối hay không.

Tôi mở ảnh trong `silver/frames/` để đối chiếu (ảnh ghép `report/screenshots/silver_frames_mau.png`):
- Khung đen `drive_a-01001` có độ sáng 0.0, độ nét 0: không có gì để học.
- Khung mờ `drive_a-01998` có độ sáng 117.8 nhưng độ nét chỉ 2: cảnh bị nhoè hoàn toàn.
- Khung tối `drive_b-00030` là cửa hầm, độ sáng 23.1 nhưng độ nét 244: vẫn thấy rõ đường và thành hầm, giữ lại là đúng.
- Khung trùng `drive_a-00100` gần như giống hệt khung liền trước `drive_a-00095`, vì xe chỉ đi 3 km/h.

Độ trễ đồng bộ frame–telemetry: trung bình 66 ms (A) và 73 ms (B), trung vị 18 / 81 ms. Mức tối đa 2884 / 3112 ms rơi vào các đoạn cảm biến mất tín hiệu, và mỗi drive có 7 khung `sync_lost`.

Tôi chọn ngưỡng **2**:
- Ngưỡng 0 chỉ bắt khung giống hệt, nên bỏ sót nhiều khung gần trùng do nhiễu nén. Drive B chỉ bắt được 8 khung.
- Ngưỡng 6 coi tới 363/602 khung của A (60%) là trùng, tức là loại cả những khung khác nhau thật.
- Ngưỡng 2 bắt được đoạn camera đứng hình của A (verify 11/12) mà vẫn giữ phần lớn khung.

Ngưỡng phụ thuộc tốc độ: xe càng chậm thì hai khung liên tiếp (cách nhau 0,5 s) càng giống nhau. Drive A chậm hơn (tốc độ trung bình của các split 32–42 km/h) nên có 149 khung trùng, nhiều gấp gần 3 lần B (64–86 km/h, 53 khung). Với dữ liệu nhiều xe dừng đèn đỏ, có thể cần ngưỡng thích ứng theo tốc độ, hoặc xét thêm `speed_mps ≈ 0`.

## Bước 4 · Gold

### Câu 6
Dựa vào `leakage.json`: chia theo khung hình làm bao nhiêu trip bị chia cắt, so với chia theo trip? Vì sao điều này làm điểm kiểm thử đẹp một cách giả tạo?

Với 1019 khung đáng dùng thuộc 26 trip:
- **Chia ngẫu nhiên theo khung hình** (70/15/15): **26/26 trip** nằm ở nhiều hơn một split, và cả **26 trip có khung ở cả train lẫn test**.
- **Chia theo trip** (gold của pipeline): **0 trip** bị chia cắt. Dbt test chống rò rỉ và verify "no trip is in two splits" đều PASS.

Các khung trong cùng một trip là cùng con đường, cùng ánh sáng, cùng thời tiết, chỉ cách nhau 0,5 s. Bộ lọc trùng chỉ so hai khung **liền nhau**, nên khung t ở train và khung t+1 s ở test vẫn gần như giống hệt. Khi chia theo khung, mô hình được "xem trước đề thi": nó nhớ cảnh thay vì học cách tổng quát. Điểm test vì vậy cao giả tạo, rồi tụt khi gặp chuyến xe mới ngoài thực tế. Chia theo trip (theo cảnh) làm test giống điều kiện triển khai: toàn cảnh mô hình chưa từng thấy.

### Câu 7
Vì sao `dataset_version` là **băm nội dung** chứ không phải ngày giờ hay số đếm? Hai lần chạy liên tiếp cho kết quả gì? Vì sao `dataset_version` của bạn **khác** bạn bè dù cùng dữ liệu (liên hệ MSSV)?

Băm nội dung cho cùng một tên với **cùng nội dung**: cùng khung, cùng split thì ra cùng version, bất kể chạy lúc nào, trên máy nào. Vì vậy version vừa là định danh vừa là phép kiểm tra tái lập. Nếu dùng ngày giờ thì mỗi lần chạy lại cho tên mới dù dữ liệu y hệt. Nếu dùng số đếm thì hai máy có thể cho cùng số "v3" mà nội dung khác nhau. Cả hai cách đều không trả lời được câu "hai dataset này có giống nhau không".

Kết quả của tôi:
- Chạy `cli export-dataset` lại, rồi `step all --reset` nhiều lần (cả sau bài tập sập/chạy lại), đều ra **`247152d8c2e5`** và cùng `manifest_sha256 71cd35ac…`.
- Ở lần chạy cuối, `silver.frames` đang ở version 25 thay vì 27 như lần trước, nhưng `dataset_version` không đổi. Như vậy version phụ thuộc **nội dung**, không phụ thuộc lịch sử commit.

MSSV được trộn vào thứ tự xếp trip (`md5(trip_id || LAB_STUDENT_ID)` trong `gold_dataset_frames.sql`), nên mỗi MSSV cho cách gán trip vào train/val/test khác nhau, dẫn đến nội dung khác và băm khác. Tôi thử đổi MSSV thành `2A202602438`: version thành `0e05ca3103f7`, split thành 693 / 177 / 149 (của tôi là 689 / 140 / 190), tổng vẫn 1019 khung. Đặt lại `2A202602437` thì version quay về `247152d8c2e5`.

### Câu 8
Đọc phần `lineage` trong `manifest.json` của dataset bạn. Trả lời "dataset này được tạo từ dữ liệu nào?" bằng chính nội dung đó. Nếu một người hỏi 3 tháng sau, thông tin nào giúp tái tạo lại đúng dataset này?

Trong MinIO, thư mục `lake/gold/datasets/247152d8c2e5/` gồm `manifest.json`, `manifest.parquet` và ba thư mục `train/` (689 ảnh), `val/` (140), `test/` (190). Số ảnh khớp đúng `by_split` của manifest (ảnh `report/screenshots/minio_gold_dataset.png`). Theo `manifest.json` (lần chạy cuối), dataset được tạo từ:
- **Bronze**: `raw_telemetry_a` v0, `raw_telemetry_b` v0, `trips_raw` v0, `video_files` v1.
- **Silver**: `frames` v25, `telemetry` v0, `frames_flagged` v0, `frame_telemetry` v0.
- **Video đầu vào**: drive_a sha256 `636ba1a98dfc…e7e9aa`, drive_b sha256 `bfa62703f645…ee9fdc`.
- **Luật**: loại `is_black, is_blurry, is_near_dup, not sync_ok`; chia "per trip, stratified by drive, md5 order".
- **Kết quả**: 1019 khung (A 436, B 583), `manifest_sha256 71cd35ac…fa624a1`.

3 tháng sau, để tái tạo đúng dataset này cần:
1. Hai file video có đúng sha256 trên, cùng file telemetry gốc (bronze giữ nguyên bản).
2. Các version Delta ở trên để time travel. Điều này chỉ được nếu chưa `VACUUM` mất file cũ.
3. MSSV `2A202602437`, vì nó là "muối" của split.
4. Các tham số trong `dbt_project.yml` (`dark_luma`, ngưỡng dHash 2, `sync_tol_ms`, `split_train/val`).

Kiểm chứng: tạo lại rồi so `dataset_version` / `manifest_sha256`. Điểm còn thiếu: manifest **không ghi commit git của mã** (pipeline/dbt) và tham số dbt. Đổi code mà không đổi dữ liệu thì lineage không cho thấy, nên tôi sẽ thêm `git_sha` và `vars` vào manifest.

## Bước 5 · Điều phối và phục vụ

### Câu 9
Mô tả chính xác điều xảy ra khi trip của bạn sập rồi chạy lại: bao nhiêu task chuyển đỏ, bao nhiêu task `upstream_failed`, bạn clear gì, dữ liệu có nhân đôi không, `dataset_version` có đổi không? Bằng chứng (ảnh, số) là gì?

**Sập.** Tôi đặt `LAB_FAIL_ONCE=trip-03904`, trigger `day09_reset` rồi `day09_video_pipeline` (run `manual__2026-10-12T15:55:41+00:00`).
- **1 task đỏ**: `silver_frames` map_index **9**. Log ghi `RuntimeError: simulated crash while extracting trip-03904; run it again` (ảnh `crash_task_log_do.png`).
- **7 task `upstream_failed`**: `dbt_run_silver`, `dbt_test_silver`, `dbt_run_gold`, `dbt_test_gold`, `publish_lake`, `export_dataset`, `publish_warehouse`.
- **29 task success**: 3 bronze, `trip_commands` và 25 trip còn lại (ảnh `crash_graph_do.png`).

**Clear.** Trên giao diện, tôi chỉ **Clear task** đúng instance `silver_frames` Map Index 9, bật Downstream. Hộp thoại báo **Affected Tasks: 8** (ảnh `crash_clear_dialog.png`): 1 task đỏ và 7 task phía sau. 25 trip đã xong không bị chạy lại.

**Sau khi chạy lại.**
- DAG chuyển **xanh 37/37** (ảnh `crash_graph_xanh_sau_clear.png`).
- `silver.frames` có **1246** dòng, cả **1246** `frame_id` đều khác nhau. `trip-03904` có **22** dòng, đúng bằng số khung của trip, không phải 44, tức **không nhân đôi**, vì trip được ghi bằng overwrite theo predicate (commit `rows_added: 22, predicate trip_id = 'trip-03904'`).
- `scripts/lab.sh verify` vẫn **27/27**, `dataset_version` vẫn **`247152d8c2e5`**.

### Câu 10
Nâng từ 2 drive lên **500 xe mỗi ngày**: ba thành phần nào của pipeline này sẽ vỡ đầu tiên, và bạn thay bằng gì (nối với slide 12, 22, 23)?

Với 500 xe × nhiều giờ video × 7 camera mỗi ngày, ba chỗ sẽ vỡ trước:
1. **Giải mã video bằng FFmpeg trên một container Airflow** (LocalExecutor, `PARALLELISM=2`). Decode là nút cổ chai về CPU. Tôi sẽ chuyển sang một cụm worker riêng (Kubernetes/Ray/Spark) có giải mã bằng GPU (NVDEC), và cân nhắc lấy mẫu hoặc lọc khung ngay trên xe để giảm dữ liệu gửi về.
2. **DuckDB + dbt chạy trên một tiến trình, một máy**, cộng với một node MinIO. Dữ liệu sẽ vượt RAM và đĩa của một máy, và Delta sinh rất nhiều file nhỏ (mỗi trip một commit). Tôi sẽ thay bằng engine phân tán (Spark/Trino trên Delta/Iceberg), object storage dạng cụm (S3), partition theo ngày/xe và chạy `OPTIMIZE`/compaction định kỳ.
3. **Điều phối và nạp theo lô**: mỗi trip một mapped task sẽ thành hàng chục nghìn task mỗi ngày, làm quá tải scheduler và DB metadata của Airflow. Tôi sẽ chuyển sang ingest hướng sự kiện (xe upload thì đẩy thông báo qua hàng đợi Kafka/SQS), xử lý theo micro-batch, và để Airflow chỉ điều phối ở mức lô ngày/giờ.

Ngoài ra, mart `mart_frame_gallery` mỗi khung một dòng trong Postgres cũng sẽ phình to. BI nên đọc bảng tổng hợp thay vì từng khung.

## Câu hỏi riêng của bạn trên Metabase

Câu SQL bạn thêm vào dashboard và điều nó cho biết:

Tôi thêm 2 câu hỏi vào dashboard "Day 9: video data pipeline" (ảnh `metabase_dashboard.png`, hai ô cuối):

```sql
-- Câu hỏi riêng 1: tỷ lệ khung bị loại theo từng trip, kèm tốc độ trung bình
select drive_id, trip_id, split, frames, frames - frames_kept as bi_loai,
       round(100.0 * (frames - frames_kept) / frames, 1) as ty_le_loai_pct,
       round(avg_speed_kmh::numeric, 1) as toc_do_tb_kmh
from marts.mart_trip_summary order by ty_le_loai_pct desc;

-- Câu hỏi riêng 2: tỷ lệ loại theo nhóm tốc độ
select case when avg_speed_kmh < 30 then '1. < 30 km/h' when avg_speed_kmh < 60 then '2. 30-60 km/h' else '3. >= 60 km/h' end as nhom_toc_do,
       count(*) as so_trip, sum(frames) as khung, sum(frames - frames_kept) as bi_loai,
       round(100.0 * sum(frames - frames_kept) / sum(frames), 1) as ty_le_loai_pct
from marts.mart_trip_summary group by 1 order by 1;
```

Kết quả câu 2: nhóm < 30 km/h có 6 trip, bị loại **29,3%** khung (75/256). Nhóm 30–60 km/h: 8 trip, **17,9%** (61/340). Nhóm ≥ 60 km/h: 12 trip, **14,0%** (91/650). Xe càng chậm thì càng bị loại nhiều, chủ yếu do khung gần trùng, khớp với nhận xét ở câu 5.

Câu 1 cho thấy một ngoại lệ đáng chú ý: `trip-04622` (drive_a) chạy trung bình 67 km/h nhưng bị loại **56,7%** (34/60 khung), cao nhất trong 26 trip. Tôi tra thêm `silver_frames_flagged`: cả 34 khung bị loại đều vì **gần trùng** (0 đen, 0 mờ). 34 khung ứng với khoảng 17 giây, dài hơn đoạn camera đứng hình 6 giây. Như vậy ở tốc độ cao mà khung vẫn bị coi là trùng, có thể do đoạn đường đơn điệu nên dHash 64 bit khó phân biệt, kèm đoạn đứng hình. Với người vận hành, câu hỏi này chỉ ra trip cần mở ảnh ra xem: hoặc camera có vấn đề, hoặc ngưỡng dHash đang loại nhầm khung tốt.

## Nhìn lại

Điều bạn thấy bất ngờ nhất, điều gì còn chưa rõ, và một lỗi dữ liệu mà bạn nghĩ pipeline **chưa** bắt được (nếu có). Tối đa 8 dòng.

- **Bất ngờ nhất**: chia ngẫu nhiên theo khung làm rò rỉ **cả 26/26 trip**, không phải "một vài". Lỗi này hoàn toàn vô hình nếu chỉ nhìn điểm test.
- **Về môi trường**: trên Codespaces, Airflow không kết nối được Postgres (timeout) vì bảng `iptables-legacy` có `FORWARD DROP` chặn mạng `day09_default`. Phải cho phép forward thì các container mới nói chuyện được. Mất khá lâu mới tìm ra đây là lỗi môi trường chứ không phải lỗi lab.
- **Chưa rõ**: vì sao verify đứng hình chỉ bắt 11/12 khung ở ngưỡng 2. Khung còn lại có lẽ khác khung trước hơn 2 bit do nhiễu nén, nhưng tôi chưa kiểm tra tận nơi.
- **Lỗi pipeline chưa bắt**:
  - Luật tốc độ 0–70 m/s (tới 252 km/h) quá rộng cho xe trong phố. Giá trị sai nhưng "có vẻ hợp lý", như 60 m/s giữa hai mẫu 20 m/s, vẫn lọt qua.
  - Không có test về **gia tốc hay độ nhảy** giữa hai mẫu liên tiếp, kể cả nhảy GPS hay đổi hướng đột ngột.
  - Cũng không có test chéo giữa tốc độ và chuyển động trong ảnh (xe báo 50 km/h nhưng khung hình đứng yên).
  - Ngược lại, có trường hợp pipeline **loại nhầm khung tốt**: khung đầu tiên `drive_a-00000` rõ nét (độ sáng 74, độ nét 613) nhưng bị `sync_lost`. Nguyên nhân là telemetry bắt đầu lúc `08:00:00.014`, tức 14 ms *sau* khung này, nên ASOF join (tìm mẫu telemetry gần nhất phía trước) không có dòng nào để ghép.
