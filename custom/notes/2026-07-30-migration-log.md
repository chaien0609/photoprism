# Log chuyển sang build từ source — 2026-07-30

## Baseline trước khi chuyển

| Chỉ số | Giá trị |
| --- | --- |
| photos | 78715 |
| files | 134383 |
| albums | 325 |
| migrations đã chạy | 39 |
| version đang chạy | `260601-a7d098548-Linux-ARM64-Plus` |
| DB size | 2.2 GB (dump gzip 526 MB) |
| storage size | 134 GB |
| disk `/` trống | 19 Gi |
| disk `/Volumes/BKM` trống | 30 Gi |

## Đường lùi

1. Sửa `image:` trong `/Volumes/BKM/photoprism/compose.yaml` về `photoprism/photoprism:260601-a7d098548`
2. `docker compose down`
3. `docker exec -i photoprism-mariadb-1 mariadb -u root -pinsecure photoprism < <(gunzip -c /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz)`
4. `docker compose up -d`

Dump: `/Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz` (526 MB, gzip OK, 39 `CREATE TABLE`)
Albums YAML: `/Volumes/BKM/photoprism/storage/backup/albums/` (325 file)

## Tiến trình

- [x] Task 1 — baseline + backup
- [x] Task 2 — fork `chaien0609/photoprism` (PUBLIC), `origin` → fork, `upstream` → photoprism/photoprism, branch `custom` đã push (SHA khớp)
- [x] Task 3 — pull `photoprism/develop:resolute` (arm64, **11.1 GB**, mất 4m53s), tạo volume `photoprism-gocache`. Toolchain: Go 1.26.5 / Node 24.18.0 / npm 12.0.1. Disk `/` còn 17 Gi sau khi pull.
- [x] Task 4 — tạo `compose.dev.yaml` + `dev.sh`, sửa `PHOTOPRISM_SITE_URL` từ port 2342 → 8098. Verify merge: project name `photoprism`, volume DB `photoprism_database` giữ nguyên, photoprism `command: None`, đủ 4 mount. Instance cũ vẫn chạy `photoprism/photoprism:latest`, API 200 — không downtime.
- [x] Task 5 — build xong trong container tạm. `make dep`: 4 model (229 MB) + npm ci 1240 package. `make build-js build-go`: binary 92 MB, frontend output 21 MB. Version `260730-2ed240c89-Linux-ARM64-DEVELOP`, edition `ce`. Instance cũ vẫn Up 22h, API 200 — **Task 1–5 không downtime**.

  Ghi chú: `frontend/node_modules` không tồn tại và đó là đúng — `package.json` khai `workspaces: ['frontend']` nên deps gom về `node_modules/` ở root.
- [x] Task 6 — chuyển sang chế độ dev, migration 39 → 43 (4 migration mới, 5 giây, 0 error). Dữ liệu sau migration: 78715 / 134383 / 325, khớp chính xác mốc trước khi chuyển.

  **Sự cố đã xử lý — thiếu `PHOTOPRISM_STORAGE_PATH` trong `compose.dev.yaml`:**

  Image `photoprism/photoprism` đặt biến này trong ENV của image; image `photoprism/develop` không có, và `compose.yaml` cũng không đặt (tôi tưởng nó đến từ compose). PhotoPrism fallback về default `<originals>/.photoprism/storage`, gây:
  - `sidecar-path` trỏ sai → 23 preview video/HEIC (20 `.mov.jpg`, 2 `.heic.jpg`, 1 `.mp4.jpg`) trả về SVG icon lỗi
  - `thumb-cache-path` trỏ sai → bỏ qua thumbnail cache 134 GB
  - ghi 40 KB / 10 file rác vào `/Volumes/BKM/memories/.photoprism/`

  Điểm khó phát hiện: `internal/api/thumbnails.go:168` trả **HTTP 200 kèm SVG icon lỗi**, nên log toàn 200 và kiểm chứng bằng status code không thấy gì sai.

  Thiệt hại dữ liệu: **không có**. photos/files/albums không đổi; 0 dòng `file_missing` bị ghi hôm nay (23 file đó đã bị đánh missing từ 11/07 bởi setup cũ nên `Update` là no-op).

  Fix: đặt `PHOTOPRISM_STORAGE_PATH`, `PHOTOPRISM_ORIGINALS_PATH`, `PHOTOPRISM_IMPORT_PATH` và nhóm thumbnail (`THUMB_SIZE` 1920, `THUMB_SIZE_UNCACHED` 7680, `THUMB_UNCACHED` true, `JPEG_SIZE`/`PNG_SIZE` 7680 — code default lệch prod ở 5120 / false).

  Kiểm chứng sau fix: mọi đường dẫn khớp (`storage-path /photoprism/storage`, `sidecar-path /photoprism/storage/sidecar`, `thumb-cache-path /photoprism/storage/cache/thumbnails`, `backup-path /photoprism/storage/backup`), và **23/23** file trước đó lỗi giờ trả `image/jpeg` thật, 0 file trả SVG. Log app 0 error.

  Sai sót trong quá trình debug, ghi lại để không lặp: tôi dùng `docker compose run <service> photoprism config` để lấy config tham chiếu từ image prod, nhưng entrypoint `/init` xoá sạch biến `PHOTOPRISM_*` trước khi CMD chạy → kết quả là code default, không phải cấu hình thật. Cách đúng để đọc ENV của image: `docker run --rm --entrypoint /bin/bash <image> -c 'echo $PHOTOPRISM_STORAGE_PATH'`.
