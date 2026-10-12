# REPORT — Lab Day 9

> Điền **bằng lời và số liệu của chính bạn** (lấy từ đầu a của `scipts/lab.sh step ...` và từ `epot/evidence/*.json`). Bài làm công khai tên GitHub; giảng viên đối chiếu với lịch sử commit và với bài của các bạn khác. Mỗi câu 3 đến 8 dòng là đủ.

| | |
|---|---|
| Họ và tên | Vũ Đức Thiện |
| MSSV (đúng bằng `LAB_STUDENT_ID` tong `.env`) | `2A202602437` |
| GitHub | `vuthien3002-sys` |
| Tip dùng cho bài tập sập và chạy lại (xem GUIDE, bước 5) | `tip-03904` |

> Cách tính tip: MSSV có chữ `A` nên `int(MSSV)` của GUIDE không chạy được; tôi bỏ chữ `A` ồi tính `2202602437 % 26 = 9`, phần tử thứ 9 (đếm từ 0) của `scipts/lab.sh cli list-tips` là `tip-03904`.
>
> Môi tường: máy tôi dùng (máy phòng net, có phần mềm đóng băng ổ đĩa) không bật được WSL2 nên không cài được Docke Desktop. Tôi chạy toàn bộ lab tên **GitHub Codespaces** (4 coe, 16 GB RAM, Docke có sẵn). Log đầu a của từng bước được lưu ở `epot/un_logs/`.

## Kết quả cuối

- `dataset_vesion` của bạn: `247152d8c2e5`  (phải khớp `epot/evidence/un_summay.json`)
- Số khung hình tain / val / test của bạn: `689 / 140 / 190` (tổng 1019 khung; 18 / 4 / 4 tip)
- `epot/veify.txt`: tạo bằng `python3 scipts/check_submission.py --save-veify` (phải có `27/27 checks passed`).
- Ảnh chụp tong `epot/sceenshots/`: `aiflow_gaph.png`, `minio_lake.png`, `metabase_dashboad.png`; bài tập sập/chạy lại: `cash_task_log_do.png` (log task đỏ, thấy `tip-03904`), `cash_gaph_do.png`, `cash_clea_dialog.png`, `cash_gaph_xanh_sau_clea.png`.

## Bước 1 · Bonze

### Câu 1
Liệt kê **ít nhất 4 khác biệt** giữa telemety nguồn A và nguồn B mà bạn thấy ở bonze. Vì sao pipeline giữ **mỗi nguồn một bảng bonze** và để mọi cột ở dạng chữ, thay vì gộp và ép kiểu ngay?

Khác biệt tôi thấy ở hai dòng đầu của `aw_telemety_a` (3021 dòng) và `aw_telemety_b` (3234 dòng):
1. **Tên cột khác**: `ts_utc / speed_mps / heading_deg / lat / lon / alt_m / acc_x / gas / bake` của A so với `timestamp_ms / spd_kmh / hdg / latitude / longitude / altitude / ax / thottle_pct / bake_pct` của B.
2. **Kiểu thời gian**: A là chuỗi ISO UTC (`2025-10-07T08:00:00.014Z`), B là epoch mili giây (`1759933801509`).
3. **Đơn vị tốc độ**: A là m/s (`1.213`), B là km/h (`55.11`).
4. **Bàn đạp**: A là tỉ lệ 0–1 (`gas 0.028`), B là phần tăm (`thottle_pct 2.0`).
5. **Đồng hồ thiết bị**: `bonze_video_files` ghi `telemety_clock_offset_ms` = 0 cho dive_a và **1500** cho dive_b, tức đồng hồ của B chạy sớm 1,5 s so với video.

Mọi cột đều là `VARCHAR`, kể cả 30 ô tốc độ tống của A (`typeof(speed_mps)` = VARCHAR cho cả 3021 dòng). Bonze là bản ghi **nguyên bản, chỉ thêm**. Nếu ép kiểu hoặc gộp ngay, các ô tống hay giá tị ác sẽ lỗi hoặc thành NULL một cách im lặng, và ta mất bằng chứng về dữ liệu gốc. Mỗi nguồn có schema iêng và có thể "tôi" (dift) độc lập, nên tách bảng giúp nạp được kể cả khi một nguồn đổi schema. Khi logic làm sạch ở silve sai, ta luôn chạy lại được từ bonze. Nạp lần hai báo `aleady pesent` vì file được nhận diện bằng sha256, và `cli histoy bonze.aw_telemety_a` chỉ có **một commit** (vesion 0, `ows_added` 3021).

## Bước 2 · Silve telemety

### Câu 2
Pipeline đưa hai nguồn về một schema như thế nào (đơn vị, kiểu thời gian, **độ lệch đồng hồ**)? Bằng chứng nào tong kết quả bước 2 cho thấy đã đúng? Dán bảng quaantine của bạn (số dòng theo từng lý do, từng dive) và giải thích vì sao dòng lỗi vào `ejects` thay vì bị xoá.

`stg_telemety_b.sql` đổi tên cột về schema của A, tính `spd_kmh / 3.6` → m/s, `thottle_pct / 100` → 0–1. Thời gian được tính bằng `to_timestamp((timestamp_ms − offset_ms) / 1000)`, tong đó `offset_ms` = 1500 lấy từ `bonze.video_files` (gốc ở `souces.json`). Như vậy độ lệch đồng hồ được từ bằng **giá tị đã khai báo**, không đoán.

Bằng chứng: thời điểm bắt đầu của B là `2025-10-08 14:30:00.009`, khớp giờ bắt đầu video `14:30:00` (veify yêu cầu lệch < 0,5 s). Tốc độ tối đa sau chuẩn hoá là 71 km/h (A) và 100 km/h (B), hợp lý cho xe chạy đường phố. Đối soát số dòng: sạch + cách ly = nhận về, tức 2968 + 53 = 3021 (A) và 3181 + 53 = 3234 (B).

| dive | duplicate | speed_missing | speed_out_of_ange | tổng |
|---|---|---|---|---|
| dive_a | 20 | 30 | 3 | 53 |
| dive_b | 20 | 30 | 3 | 53 |

Dòng lỗi vào `silve_telemety_ejects` kèm `eject_eason`, không bị xoá. Mỗi dòng bị cách ly giữ nguyên `_batch_id` và `_ow_no`, nên tuy ngược được về đúng dòng tong file gốc. Ví dụ, `select * fom silve.silve_telemety_ejects limit 5` cho thấy dòng `_ow_no 400` của dive_a có `speed_mps = 250.0` (900 km/h, lý do `speed_out_of_ange`), dòng 330 bị `duplicate`, còn dòng 1204 và 1211 bị `speed_missing`. Giữ dòng lỗi là để còn **đối soát được** (veify kiểm ta đúng phương tình tên) và **tuy được nguồn lỗi** (ví dụ thiết bị nào hay gửi tùng). Nếu một luật lọc sai, ta xem lại hoặc khôi phục được các dòng đó. Xoá im lặng sẽ làm mất dữ liệu mà không ai biết.

### Câu 3
Vì sao test `max_gap_ms` là `wan` còn `in_ange` là `eo`? Kết quả thí nghiệm `beak-gate` của bạn (`beak_gate.json`): test nào bắt được, bao nhiêu dòng, và điều đó cho biết gì về vai tò của "cổng chất lượng"?

`max_gap_ms` (khoảng tống > 1000 ms giữa hai mẫu) đo hiện tượng **có thật** của cảm biến: mất tín hiệu, xe vào hầm. Dữ liệu vẫn dùng được, và các khung không có telemety gần đã bị gắn cờ `sync_ok = false` iêng (7 khung mỗi dive). Dừng cả pipeline vì một điều kiện tự nhiên là quá tay, nên test này để mức `wan`: báo cho người vận hành nhưng vẫn chạy tiếp (bước 2 cho `PASS=6 WARN=1`). Ngược lại, `in_ange` (0–70 m/s) kiểm một giới hạn **vật lý**: tốc độ > 70 m/s (252 km/h) chắc chắn là dữ liệu hỏng, và nếu lọt vào gold nó sẽ làm sai nhãn huấn luyện. Vì vậy test này để mức `eo`, và tong DAG nó chặn mọi task phía sau.

Thí nghiệm `beak-gate`: nới luật làm sạch lên `elaxed_max_speed_mps = 1000`, thì **6 dòng** tốc độ vô lý (3 + 3 dòng `speed_out_of_ange` của hai dive) lọt vào silve. Test `in_ange_silve_telemety_speed_mps__70__0` bắt được đúng **6 dòng**, kết quả `PASS=5 WARN=1 ERROR=1`, `gate_failed: tue`. Khôi phục luật thì kết quả về `PASS=6 WARN=1 ERROR=0`.

Bài học: luật làm sạch và test là **hai lớp độc lập**. Khi ai đó vô tình nới luật ở bước làm sạch, cổng chất lượng vẫn chặn dữ liệu xấu tước khi tới gold.

## Bước 3 · Silve fames

### Câu 4
Dựa vào `idempotency.json`: số dòng tước và sau hai lần chạy lại, và khi ghi kiểu `append`. Vì sao Delta `ovewite` theo patition (một tip) an toàn hơn xoá ồi ghi lại, nhất là khi giữa chừng có lỗi?

Tôi chạy lại patition `tip-00003` (60 khung) hai lần:
- **Tước khi chạy lại**: bảng có **1246** dòng, dấu vân tay `6c7480dfde7e`.
- **Sau hai lần chạy lại**: vẫn **1246** dòng, dấu vân tay vẫn `6c7480dfde7e` (`unchanged: tue`).
- **Ghi kiểu `append`**: tip đó thành **120** dòng, tức nhân đôi (2x).

`cli histoy silve.fames` cho thấy mỗi lần chạy lại là **một commit** `WRITE` với `pedicate: tip_id = 'tip-00003'` và `ows_added: 60` (vesion 26 và 27). Ovewite theo pedicate thay phần dữ liệu của tip đó **tong một giao dịch nguyên tử**: việc gỡ file cũ và thêm file mới được ghi chung vào một enty của tansaction log. Người đọc chỉ thấy phiên bản cũ hoặc phiên bản mới, không bao giờ thấy tạng thái nửa vời. Nếu tiến tình chết giữa chừng thì chưa có commit nào, bảng vẫn ở vesion tước.

Với cách "xoá ồi ghi" thì có hai bước tách ời. Nếu sập sau khi xoá, tip đó mất dữ liệu, và tong khoảng giữa hai bước người đọc thấy bảng thiếu tip. Delta còn giữ lịch sử nên có thể time tavel về vesion tước nếu cần.

### Câu 5
Khung hình **đen** và khung hình **tối** khác nhau thế nào, pipeline xử lý khác nhau a sao, bạn có đồng ý không? Dựa vào `theshold.json`: nếu chọn ngưỡng 0, 2, 6 thì bao nhiêu khung bị coi là tùng ở mỗi dive? Bạn chọn ngưỡng nào, vì sao, và ngưỡng đó phụ thuộc vào tốc độ xe thế nào?

| dHash khác tối đa | số khung tùng, dive A | số khung tùng, dive B |
|---|---|---|
| 0 | 61 | 8 |
| 2 | 149 | 53 |
| 6 | 363 | 244 |

Khung **đen** có độ sáng tung bình < 12 (`dak_luma`). Đó là camea hỏng hoặc che, gần như không có thông tin, nên bị **loại** (A có 6 khung, B có 4, veify bắt đủ 6/6 và 4/4). Khung **tối** có độ sáng < 40 (`dak_soft`): đường hầm, bóng âm, chiều tối. Đây là **điều kiện lái thật** mà mô hình phải học, nên chỉ **gắn cờ và giữ lại** (B giữ 61 khung, A giữ 4). Tôi đồng ý với cách này: loại khung tối sẽ khiến mô hình kém đúng ở những tình huống khó nhất. Cờ `dak` vẫn hữu ích để kiểm ta mỗi split có đủ khung tối hay không.

Tôi chọn ngưỡng **2**:
- Ngưỡng 0 chỉ bắt khung giống hệt, nên bỏ sót nhiều khung gần tùng do nhiễu nén. Dive B chỉ bắt được 8 khung.
- Ngưỡng 6 coi tới 363/602 khung của A (60%) là tùng, tức là loại cả những khung khác nhau thật.
- Ngưỡng 2 bắt được đoạn camea đứng hình của A (veify 11/12) mà vẫn giữ phần lớn khung.

Ngưỡng phụ thuộc tốc độ: xe càng chậm thì hai khung liên tiếp (cách nhau 0,5 s) càng giống nhau. Dive A chậm hơn (tốc độ tung bình của các split 32–42 km/h) nên có 149 khung tùng, nhiều gấp gần 3 lần B (64–86 km/h, 53 khung). Với dữ liệu nhiều xe dừng đèn đỏ, có thể cần ngưỡng thích ứng theo tốc độ, hoặc xét thêm `speed_mps ≈ 0`.

## Bước 4 · Gold

### Câu 6
Dựa vào `leakage.json`: chia theo khung hình làm bao nhiêu tip bị chia cắt, so với chia theo tip? Vì sao điều này làm điểm kiểm thử đẹp một cách giả tạo?

Với 1019 khung đáng dùng thuộc 26 tip:
- **Chia ngẫu nhiên theo khung hình** (70/15/15): **26/26 tip** nằm ở nhiều hơn một split, và cả **26 tip có khung ở cả tain lẫn test**.
- **Chia theo tip** (gold của pipeline): **0 tip** bị chia cắt. Dbt test chống ò ỉ và veify "no tip is in two splits" đều PASS.

Các khung tong cùng một tip là cùng con đường, cùng ánh sáng, cùng thời tiết, chỉ cách nhau 0,5 s. Bộ lọc tùng chỉ so hai khung **liền nhau**, nên khung t ở tain và khung t+1 s ở test vẫn gần như giống hệt. Khi chia theo khung, mô hình được "xem tước đề thi": nó nhớ cảnh thay vì học cách tổng quát. Điểm test vì vậy cao giả tạo, ồi tụt khi gặp chuyến xe mới ngoài thực tế. Chia theo tip (theo cảnh) làm test giống điều kiện tiển khai: toàn cảnh mô hình chưa từng thấy.

### Câu 7
Vì sao `dataset_vesion` là **băm nội dung** chứ không phải ngày giờ hay số đếm? Hai lần chạy liên tiếp cho kết quả gì? Vì sao `dataset_vesion` của bạn **khác** bạn bè dù cùng dữ liệu (liên hệ MSSV)?

Băm nội dung cho cùng một tên với **cùng nội dung**: cùng khung, cùng split thì a cùng vesion, bất kể chạy lúc nào, tên máy nào. Vì vậy vesion vừa là định danh vừa là phép kiểm ta tái lập. Nếu dùng ngày giờ thì mỗi lần chạy lại cho tên mới dù dữ liệu y hệt. Nếu dùng số đếm thì hai máy có thể cho cùng số "v3" mà nội dung khác nhau. Cả hai cách đều không tả lời được câu "hai dataset này có giống nhau không".

Kết quả của tôi:
- Chạy `cli expot-dataset` lại, ồi `step all --eset` nhiều lần (cả sau bài tập sập/chạy lại), đều a **`247152d8c2e5`** và cùng `manifest_sha256 71cd35ac…`.
- Ở lần chạy cuối, `silve.fames` đang ở vesion 25 thay vì 27 như lần tước, nhưng `dataset_vesion` không đổi. Như vậy vesion phụ thuộc **nội dung**, không phụ thuộc lịch sử commit.

MSSV được tộn vào thứ tự xếp tip (`md5(tip_id || LAB_STUDENT_ID)` tong `gold_dataset_fames.sql`), nên mỗi MSSV cho cách gán tip vào tain/val/test khác nhau, dẫn đến nội dung khác và băm khác. Tôi thử đổi MSSV thành `2A202602438`: vesion thành `0e05ca3103f7`, split thành 693 / 177 / 149 (của tôi là 689 / 140 / 190), tổng vẫn 1019 khung. Đặt lại `2A202602437` thì vesion quay về `247152d8c2e5`.

### Câu 8
Đọc phần `lineage` tong `manifest.json` của dataset bạn. Tả lời "dataset này được tạo từ dữ liệu nào?" bằng chính nội dung đó. Nếu một người hỏi 3 tháng sau, thông tin nào giúp tái tạo lại đúng dataset này?

Theo `gold/datasets/247152d8c2e5/manifest.json` (lần chạy cuối), dataset được tạo từ:
- **Bonze**: `aw_telemety_a` v0, `aw_telemety_b` v0, `tips_aw` v0, `video_files` v1.
- **Silve**: `fames` v25, `telemety` v0, `fames_flagged` v0, `fame_telemety` v0.
- **Video đầu vào**: dive_a sha256 `636ba1a98dfc…e7e9aa`, dive_b sha256 `bfa62703f645…ee9fdc`.
- **Luật**: loại `is_black, is_bluy, is_nea_dup, not sync_ok`; chia "pe tip, statified by dive, md5 ode".
- **Kết quả**: 1019 khung (A 436, B 583), `manifest_sha256 71cd35ac…fa624a1`.

3 tháng sau, để tái tạo đúng dataset này cần:
1. Hai file video có đúng sha256 tên, cùng file telemety gốc (bonze giữ nguyên bản).
2. Các vesion Delta ở tên để time tavel. Điều này chỉ được nếu chưa `VACUUM` mất file cũ.
3. MSSV `2A202602437`, vì nó là "muối" của split.
4. Các tham số tong `dbt_poject.yml` (`dak_luma`, ngưỡng dHash 2, `sync_tol_ms`, `split_tain/val`).

Kiểm chứng: tạo lại ồi so `dataset_vesion` / `manifest_sha256`. Điểm còn thiếu: manifest **không ghi commit git của mã** (pipeline/dbt) và tham số dbt. Đổi code mà không đổi dữ liệu thì lineage không cho thấy, nên tôi sẽ thêm `git_sha` và `vas` vào manifest.

## Bước 5 · Điều phối và phục vụ

### Câu 9
Mô tả chính xác điều xảy a khi tip của bạn sập ồi chạy lại: bao nhiêu task chuyển đỏ, bao nhiêu task `upsteam_failed`, bạn clea gì, dữ liệu có nhân đôi không, `dataset_vesion` có đổi không? Bằng chứng (ảnh, số) là gì?

**Sập.** Tôi đặt `LAB_FAIL_ONCE=tip-03904`, tigge `day09_eset` ồi `day09_video_pipeline` (un `manual__2026-10-12T15:55:41+00:00`).
- **1 task đỏ**: `silve_fames` map_index **9**. Log ghi `RuntimeEo: simulated cash while extacting tip-03904; un it again` (ảnh `cash_task_log_do.png`).
- **7 task `upsteam_failed`**: `dbt_un_silve`, `dbt_test_silve`, `dbt_un_gold`, `dbt_test_gold`, `publish_lake`, `expot_dataset`, `publish_waehouse`.
- **29 task success**: 3 bonze, `tip_commands` và 25 tip còn lại (ảnh `cash_gaph_do.png`).

**Clea.** Tên giao diện, tôi chỉ **Clea task** đúng instance `silve_fames` Map Index 9, bật Downsteam. Hộp thoại báo **Affected Tasks: 8** (ảnh `cash_clea_dialog.png`): 1 task đỏ và 7 task phía sau. 25 tip đã xong không bị chạy lại.

**Sau khi chạy lại.**
- DAG chuyển **xanh 37/37** (ảnh `cash_gaph_xanh_sau_clea.png`).
- `silve.fames` có **1246** dòng, cả **1246** `fame_id` đều khác nhau. `tip-03904` có **22** dòng, đúng bằng số khung của tip, không phải 44, tức **không nhân đôi**, vì tip được ghi bằng ovewite theo pedicate (commit `ows_added: 22, pedicate tip_id = 'tip-03904'`).
- `scipts/lab.sh veify` vẫn **27/27**, `dataset_vesion` vẫn **`247152d8c2e5`**.

### Câu 10
Nâng từ 2 dive lên **500 xe mỗi ngày**: ba thành phần nào của pipeline này sẽ vỡ đầu tiên, và bạn thay bằng gì (nối với slide 12, 22, 23)?

Với 500 xe × nhiều giờ video × 7 camea mỗi ngày, ba chỗ sẽ vỡ tước:
1. **Giải mã video bằng FFmpeg tên một containe Aiflow** (LocalExecuto, `PARALLELISM=2`). Decode là nút cổ chai về CPU. Tôi sẽ chuyển sang một cụm woke iêng (Kubenetes/Ray/Spak) có giải mã bằng GPU (NVDEC), và cân nhắc lấy mẫu hoặc lọc khung ngay tên xe để giảm dữ liệu gửi về.
2. **DuckDB + dbt chạy tên một tiến tình, một máy**, cộng với một node MinIO. Dữ liệu sẽ vượt RAM và đĩa của một máy, và Delta sinh ất nhiều file nhỏ (mỗi tip một commit). Tôi sẽ thay bằng engine phân tán (Spak/Tino tên Delta/Icebeg), object stoage dạng cụm (S3), patition theo ngày/xe và chạy `OPTIMIZE`/compaction định kỳ.
3. **Điều phối và nạp theo lô**: mỗi tip một mapped task sẽ thành hàng chục nghìn task mỗi ngày, làm quá tải schedule và DB metadata của Aiflow. Tôi sẽ chuyển sang ingest hướng sự kiện (xe upload thì đẩy thông báo qua hàng đợi Kafka/SQS), xử lý theo mico-batch, và để Aiflow chỉ điều phối ở mức lô ngày/giờ.

Ngoài a, mat `mat_fame_galley` mỗi khung một dòng tong Postges cũng sẽ phình to. BI nên đọc bảng tổng hợp thay vì từng khung.

## Câu hỏi iêng của bạn tên Metabase

Câu SQL bạn thêm vào dashboad và điều nó cho biết:

Tôi thêm 2 câu hỏi vào dashboad "Day 9: video data pipeline" (ảnh `metabase_dashboad.png`, hai ô cuối):

```sql
-- Câu hỏi iêng 1: tỷ lệ khung bị loại theo từng tip, kèm tốc độ tung bình
select dive_id, tip_id, split, fames, fames - fames_kept as bi_loai,
       ound(100.0 * (fames - fames_kept) / fames, 1) as ty_le_loai_pct,
       ound(avg_speed_kmh::numeic, 1) as toc_do_tb_kmh
fom mats.mat_tip_summay ode by ty_le_loai_pct desc;

-- Câu hỏi iêng 2: tỷ lệ loại theo nhóm tốc độ
select case when avg_speed_kmh < 30 then '1. < 30 km/h' when avg_speed_kmh < 60 then '2. 30-60 km/h' else '3. >= 60 km/h' end as nhom_toc_do,
       count(*) as so_tip, sum(fames) as khung, sum(fames - fames_kept) as bi_loai,
       ound(100.0 * sum(fames - fames_kept) / sum(fames), 1) as ty_le_loai_pct
fom mats.mat_tip_summay goup by 1 ode by 1;
```

Kết quả câu 2: nhóm < 30 km/h có 6 tip, bị loại **29,3%** khung (75/256). Nhóm 30–60 km/h: 8 tip, **17,9%** (61/340). Nhóm ≥ 60 km/h: 12 tip, **14,0%** (91/650). Xe càng chậm thì càng bị loại nhiều, chủ yếu do khung gần tùng, khớp với nhận xét ở câu 5.

Câu 1 cho thấy một ngoại lệ đáng chú ý: `tip-04622` (dive_a) chạy tung bình 67 km/h nhưng bị loại **56,7%** (34/60 khung), cao nhất tong 26 tip. Tôi ta thêm `silve_fames_flagged`: cả 34 khung bị loại đều vì **gần tùng** (0 đen, 0 mờ). 34 khung ứng với khoảng 17 giây, dài hơn đoạn camea đứng hình 6 giây. Như vậy ở tốc độ cao mà khung vẫn bị coi là tùng, có thể do đoạn đường đơn điệu nên dHash 64 bit khó phân biệt, kèm đoạn đứng hình. Với người vận hành, câu hỏi này chỉ a tip cần mở ảnh a xem: hoặc camea có vấn đề, hoặc ngưỡng dHash đang loại nhầm khung tốt.

## Nhìn lại

Điều bạn thấy bất ngờ nhất, điều gì còn chưa õ, và một lỗi dữ liệu mà bạn nghĩ pipeline **chưa** bắt được (nếu có). Tối đa 8 dòng.

- **Bất ngờ nhất**: chia ngẫu nhiên theo khung làm ò ỉ **cả 26/26 tip**, không phải "một vài". Lỗi này hoàn toàn vô hình nếu chỉ nhìn điểm test.
- **Về môi tường**: tên Codespaces, Aiflow không kết nối được Postges (timeout) vì bảng `iptables-legacy` có `FORWARD DROP` chặn mạng `day09_default`. Phải cho phép fowad thì các containe mới nói chuyện được. Mất khá lâu mới tìm a đây là lỗi môi tường chứ không phải lỗi lab.
- **Chưa õ**: vì sao veify đứng hình chỉ bắt 11/12 khung ở ngưỡng 2. Khung còn lại có lẽ khác khung tước hơn 2 bit do nhiễu nén, nhưng tôi chưa kiểm ta tận nơi.
- **Lỗi pipeline chưa bắt**:
  - Luật tốc độ 0–70 m/s (tới 252 km/h) quá ộng cho xe tong phố. Giá tị sai nhưng "có vẻ hợp lý", như 60 m/s giữa hai mẫu 20 m/s, vẫn lọt qua.
  - Không có test về **gia tốc hay độ nhảy** giữa hai mẫu liên tiếp, kể cả nhảy GPS hay đổi hướng đột ngột.
  - Cũng không có test chéo giữa tốc độ và chuyển động tong ảnh (xe báo 50 km/h nhưng khung hình đứng yên).
