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

- [x] Task 8 — bake `photoprism/photoprism:local` (**~15 phút**, không phải 25–50 như ước tính), 3.27 GB, đúng bằng size image chính thức. 3 tag: `local`, `ce-resolute`, `260730-ce-resolute`. Version `260730-576b598e2-Linux-ARM64` (không có `-DEVELOP` vì target `install` build bằng `scripts/build.sh prod`), edition `ce`.

  Sau khi switch sang chế độ image: dữ liệu 78715 / 134383 / 325, migrations 43 — khớp. Log có `custom: running build from local source (260730-576b598e2-Linux-ARM64)` → thay đổi Go ở Task 7 đã vào image. 0 fatal/panic. Đường dẫn đúng: `storage-path /photoprism/storage`, `sidecar-path /photoprism/storage/sidecar`, `thumb-cache-path /photoprism/storage/cache/thumbnails`.

  **ENV image drift giữa 260601 và 260728** (thay đổi upstream, không phải lỗi setup): image `:latest` (260601) đặt `THUMB_SIZE=1920`, `THUMB_SIZE_UNCACHED=7680`, `THUMB_UNCACHED=true`, `JPEG_SIZE=7680`, `PNG_SIZE=7680` trong ENV; bản resolute (260728) đã **bỏ** các biến này và đổi `THUMB_UNCACHED` sang `false`. Đã ghim tường minh 5 biến đó vào `compose.yaml` để hành vi giữ nguyên như setup cũ và để hai chế độ giống nhau.

  Sai sót của tôi khi kiểm chứng, ghi lại để không lặp: tôi curl thumbnail và chỉ in `content_type`, thấy `image/svg+xml` nên kết luận "chế độ image bị lỗi thumbnail". Thực tế là **HTTP 403** — preview token `a48sm2dk` đã hết hiệu lực (session biến mất khỏi `auth_sessions`), và API trả 403 kèm SVG. Ảnh thường cũng 403. Bài học: luôn in `%{http_code}` cùng `%{content_type}`.

## Số liệu vòng lặp dev (đo thực tế)

| Việc | Thời gian thật | Ước tính trong spec |
| --- | --- | --- |
| `make build-go` | **36 giây** | 1–3 phút |
| `make build-js` | **31–49 giây** | vài giây (sai — đó là watch-js incremental) |
| `make dep` lần đầu | ~2 phút | 15–25 phút |
| pull `develop:resolute` | 4m53s | 5–15 phút |

## Việc còn tồn (quyết định để sau)

- **58 file `file_missing=1`** đánh từ 11/07 bởi setup cũ, gồm 23 file preview video/HEIC mà preview giờ đã render được. Chúng bị ẩn khỏi kết quả tìm kiếm. Cách xử lý đúng: `photoprism index` (app tự đọc lại disk); cách nhanh: UPDATE có điều kiện kiểm file tồn tại trên disk. **Chưa làm.**
- **`hub.yml` bị app ghi lại lúc 17:13 ngày 30/07** (trước đó mtime 29/07). Key/Secret/Session/Serial đều còn. Trường `Status` rỗng — **không xác minh được** vốn đã rỗng hay từng có trạng thái membership, vì không có bản cũ để so. Binary cũ báo edition `Plus`, bản build từ source là `ce`. Cần kiểm tab Places/maps trong UI để biết có ảnh hưởng thật không.
- **`gh` active account**: push lên fork cần active account là `chaien0609`. Trong phiên làm việc này active account là `core-cuong` nên push bị 403. Xử lý: `gh auth switch --user chaien0609` → push → switch lại.
- Rác `/Volumes/BKM/memories/.photoprism/` (40 KB, 10 file scaffolding, 0 file ảnh) **đã xoá**. Config thật trong `/Volumes/BKM/photoprism/storage/config/` không bị đụng (`settings.yml` mtime 01/02, `serial` mtime 30/01).

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
- [x] Task 7 — vòng lặp customize xác nhận hoạt động.

  **Go**: thêm `log.Infof("custom: running build from local source (%s)", conf.Version())` vào `internal/commands/start.go:92`, trước anchor `// Initialize the index database.`. Sau `make build-go` (36 giây) và restart, log in ra `custom: running build from local source (260730-c10a98cd0-Linux-ARM64-DEVELOP)`. Giữ lại làm dấu hiệu nhận biết đang chạy bản build riêng.

  **Frontend**: sửa tạm một chuỗi trong `frontend/src/common/config.js`, build, marker xuất hiện trong `app.38906a59abc6d5835960.js` + `share.*.js` với **hash filename mới**. Đã revert và build lại, marker về 0.

  Hai phép thử sai của tôi, phát hiện trong lúc chạy:
  1. **mtime không dùng được để kiểm chứng build frontend** — webpack mặc định `output.compareBeforeEmit: true`, nội dung không đổi thì không ghi lại file nên mtime giữ nguyên dù build đã chạy thành công. (Và `ls --time-style` không có trên macOS.)
  2. **Dừng app phải làm bên trong container** — giết client `docker compose exec` ở host không dừng tiến trình trong container; nó vẫn giữ cổng 2342, khiến `run` lần sau lỗi `bind: address already in use` rồi shutdown, trong khi cổng vẫn do binary **cũ** phục vụ. Đã thêm `./dev.sh stop-app` và `./dev.sh restart` để xử lý. Cũng lưu ý: `ps aux | grep -c "[p]hotoprism start"` cho kết quả sai (đếm cả command line của chính nó).
