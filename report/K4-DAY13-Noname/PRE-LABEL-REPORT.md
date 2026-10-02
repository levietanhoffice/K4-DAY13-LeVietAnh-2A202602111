# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm Noname / Lớp K4-Day13
- Thành viên: xem `TEAMMATES.md` (Lê Việt Anh - MSSV: 2A202602111 - Thực hiện cá nhân).
- Trạng thái: `executed-by-group` (thực hiện độc lập trên máy cá nhân)
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Việt Anh; 2026-10-02 09:42:00 (UTC 02:41:21); Windows 11 x86_64, Docker Desktop Linux containers (amd64)
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` (ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); Repo Git commit: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (frame_id: `demo`, 17238 points, KITTI format); SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image `/opt/PointPillars/pretrained/epoch_160.pth` (SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`)
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ tư dùng hằng số (bỏ reflectance gốc KITTI); `z_ground = 0.075 m` (ước lượng tự động từ PCD)

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ nhận diện được duy nhất 1 hộp `vehicles` tại x≈13.15m, y≈-0.45m với score rất thấp (0.322). Đáy hộp bị chìm sát đất do không bù trừ delta. |
| B | 1.73 | 0.16 | 13 | 1.034 | `boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | Phát hiện 13 hộp gồm 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`. Hộp bám sát cụm điểm LiDAR, cao độ tâm z trung bình ~1.034m hợp lý so với mặt đất. |
| C | 1.73 | 0.32 | 6 | 1.091 | `boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Chỉ phát hiện được 6 hộp và toàn bộ đều là `pedestrian`, hoàn toàn mất lớp `vehicles` và `two-wheels` do kích thước pillar tăng gấp đôi làm loãng đặc trưng cụm điểm lớn. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side cho thấy lượt A gần như toàn bộ cụm điểm xe không được đóng hộp; lượt B các cụm điểm xe ô tô trên lòng đường đều có hộp bao quanh với score cao (>0.8-0.9). Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là: tại sao với delta=0 model vẫn bắt được 1 hộp yếu mà không trượt hoàn toàn.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh Side và file JSON cho thấy sự biến đổi hoàn toàn về class: lượt B có 10 vehicles, 1 two-wheels, 2 pedestrian; lượt C chỉ còn lại 6 pedestrian, mất sạch vehicles. Có đủ bằng chứng để kết luận tốt hơn không? Chưa đủ cơ sở để nói C tốt hơn, ngược lại C làm mất hoàn toàn nhận diện ô tô do model pretrained được huấn luyện tối ưu với voxel size 0.16m.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Hình chiếu Side (mặt phẳng x-z) nhìn từ cạnh xe gộp tất cả các vật thể ở các tọa độ y khác nhau đè lên nhau, do đó không thể thấy được độ xoay yaw hoặc kiểm tra chồng lấn tâm theo trục y, và không phân biệt được vật bị miss do ngoài ROI hay do model bỏ sót.
- JSON nào còn chưa đủ cơ sở để import? JSON của lượt A và C hoàn toàn không đủ cơ sở để dùng. Ngay cả JSON lượt B cũng chỉ là pre-label gợi ý ban đầu từ model KITTI, chưa thể coi là nhãn chuẩn để import nếu chưa qua rà soát và đối chiếu ảnh camera.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm từng hộp bình thường | Giữ nguyên từ lượt B; tất cả 13 hộp có đáy bám sát vệt điểm mặt đất trên ảnh `side-correct.png`. |
| case-batch-z | 13 / 13 | -1.805 m (bị trừ `z_ground + delta`) | Không đổi (class, x, y, size, yaw, score giữ nguyên) | **DỪNG BATCH NGAY** | Trên ảnh `side-batch-z.png`, toàn bộ 13 hộp đều bị chìm sâu xuống dưới mặt đất (tâm z âm ~-0.8m đến -1.1m). Đây là lỗi pipeline (thiếu bước cộng ngược z). |
| case-one-box-z | 1 / 13 | -1.805 m (chỉ hộp đầu tiên ID 0) | Không đổi (các hộp còn lại giữ nguyên z gốc) | Kiểm tra từng hộp (không dừng batch) | Trên `side-one-box-z.png`, 12 hộp vẫn nằm chuẩn trên mặt đất, duy nhất hộp tại x≈8.09m bị chìm. Đây là lỗi cục bộ, không phải do pipeline chuyển đổi cả frame. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Thành viên: Lê Việt Anh (MSSV: 2A202602111)
- **Vai trò đã làm**: Tự thực hiện toàn bộ quy trình: Vận hành lệnh chạy A/B/C và QC cases, kiểm tra cấu hình/JSON, quan sát ảnh Side hình học, phân tích số liệu và ghi báo cáo.
- **Quan sát A/B/C**: Ở lượt A (delta=0), model chỉ ra 1 hộp xe tại x=13.15m với score chỉ 0.322. Khi đổi delta=1.73m ở lượt B, số hộp tăng lên 13 với 10 xe ô tô có score rất cao (>0.9). Điều này chứng minh việc đưa đúng giả định sensor height vào model PointPillars là tối quan trọng. Khi tăng voxel_size lên 0.32m ở lượt C, mất sạch lớp xe ô tô, chỉ còn 6 hộp người đi bộ.
- **Diễn giải phép z thuận/ngược**: Pipeline trừ `(z_ground + delta)` trước khi inference để đưa dữ liệu điểm về hệ quy chiếu mặt phẳng chuẩn mà model KITTI được học. Sau khi model dự đoán, bắt buộc phải cộng ngược lại `(z_ground + delta)` để đưa hộp về tọa độ thực của point cloud gốc.
- **Quyết định lỗi batch và hành động**: Khi gặp `case-batch-z`, tất cả 13 hộp đều chìm xuống lòng đất đúng 1.805m $\rightarrow$ Đây là lỗi pipeline toàn hệ thống, hành động đúng là dừng sửa thủ công ngay lập tức và báo kỹ thuật kiểm tra lại code transform z của batch.
- **Điều chưa chắc**: Khi chuyển sang voxel 0.32m (lượt C), vì sao model lại phân loại nhầm các cụm điểm thành `pedestrian` thay vì chỉ đơn giản là bỏ sót hộp? Cần tìm hiểu thêm về anchor size và cơ chế gán nhãn của PointPillars khi thay đổi độ phân giải pillar.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
