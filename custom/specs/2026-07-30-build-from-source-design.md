# Chuyển PhotoPrism từ image pull sang build từ source

**Ngày:** 2026-07-30
**Trạng thái:** Đã duyệt thiết kế, chờ kế hoạch implement

## Mục tiêu

Môi trường PhotoPrism duy nhất tại `/Volumes/BKM/photoprism` hiện chạy image
`photoprism/photoprism:latest` pull từ Docker Hub. Chuyển nó sang chạy code build từ
repo `/Volumes/BKM/git/photoprism`, sao cho:

1. Cập nhật được source mới nhất từ upstream mà không mất code tự sửa.
2. Sửa được cả frontend (Vue) và backend (Go) với vòng lặp đủ nhanh để thử nghiệm.
3. Giữ nguyên dữ liệu đang có: ảnh gốc, DB, thumbnail cache.

## Bối cảnh hiện tại

| Hạng mục | Giá trị |
| --- | --- |
| Deployment dir | `/Volumes/BKM/photoprism` (không phải git repo) |
| Source repo | `/Volumes/BKM/git/photoprism` |
| Image đang chạy | `photoprism/photoprism:latest` = `260601-a7d098548-Linux-ARM64-Plus` |
| Repo trước khi bắt đầu | branch `develop`, **1258 commit sau** `origin/develop` |
| Tag release mới nhất | `260728-bbde8f452` (28/07/2026) |
| Ảnh gốc | `/Volumes/BKM/memories` → `/photoprism/originals` |
| Storage | `./storage` → `/photoprism/storage` |
| DB | named volume `database`, MariaDB 11, db `photoprism` |
| Host | macOS arm64, Docker 29.6.2, 10 CPU, 8 GB RAM cho Docker |
| Disk nội bộ trống | ~20 GB (Docker đã dùng ~19 GB) |
| Disk BKM trống | ~30 GB / 932 GB |

Đường build có sẵn trong repo (đã xác minh trên tag `260728-bbde8f452`):

- `make docker-local` → `docker-local-resolute` → pull `photoprism/develop:resolute` +
  `ubuntu:resolute`, chạy `scripts/docker/build.sh photoprism ce-resolute /resolute
  "-t photoprism/photoprism:local"`.
- `docker/photoprism/resolute/Dockerfile` là multi-stage: stage build dùng
  `photoprism/develop:resolute` chạy `make all install DESTDIR=/opt/photoprism`, stage
  production dùng `photoprism/develop:resolute-slim`.
- Không cần Go trên host (host chưa có Go, chỉ có Node 22).
- `make all` = `dep build-js`; `make build-go` = `scripts/build.sh develop photoprism`.

## Quyết định thiết kế

### Hướng: kết hợp dev-container + bake image (hướng C)

Lý do: `scripts/docker/build.sh` chạy `docker build --no-cache`, và `make all install`
nằm trong **một** RUN layer — sửa một dòng Go cũng rebuild toàn bộ Go + npm, ước tính
25–50 phút trên 10 CPU / 8 GB. Không dùng được cho vòng lặp thử nghiệm.

Nên có hai chế độ trên **cùng một stack, cùng một data**:

- **Chế độ dev** — service `photoprism` chạy `photoprism/develop:resolute`, mount source
  từ repo. Sửa Go → `make build-go` (~1–3 phút). Sửa Vue → `make watch-js` (vài giây).
- **Chế độ image** — `make docker-local` bake `photoprism/photoprism:local`, compose chạy
  image đó. Gọn, giống cấu trúc image chính thức, dùng cho chạy thường ngày.

Chuyển qua lại bằng compose override, không sửa file chính:

```bash
docker compose up -d                                  # chế độ image
docker compose -f compose.yaml -f compose.dev.yaml up -d   # chế độ dev
```

Hai chế độ dùng chung base image `photoprism/develop:resolute` nên chỉ tải một lần.

### Base version: tag `260728-bbde8f452`

Không dùng `develop` HEAD. Lý do: chuyển từ 260601 → bản mới sẽ chạy DB migration một
chiều; tag release đã qua QA upstream, `develop` HEAD thì chưa.

### Git: fork trên GitHub

```
origin    → github.com/chaien0609/photoprism   (fork, tạo bằng gh — đã auth)
upstream  → github.com/photoprism/photoprism   (chỉ fetch)
```

- Branch `custom`: tạo từ tag `260728-bbde8f452`, chứa toàn bộ code sửa. **Đã tạo.**
- Branch `develop`: giữ làm mirror upstream, không commit lên.
- Mọi file thuộc fork đặt trong `custom/` ở gốc repo. Thư mục này không tồn tại trên
  upstream nên không bao giờ conflict khi rebase. (Không dùng `docs/` — upstream đã
  `.gitignore` đường dẫn đó.)

## Kiến trúc file

```
/Volumes/BKM/photoprism/           # deployment (không phải git repo)
  compose.yaml                     # SỬA: image → :local, SITE_URL → port 8098
  compose.dev.yaml                 # MỚI: override bật chế độ dev
  dev.sh                           # MỚI: wrapper các lệnh hay dùng
  CLAUDE.md                        # SỬA: ghi lại workflow mới
  storage/                         # giữ nguyên

/Volumes/BKM/git/photoprism/       # source, branch custom
  custom/specs/                    # MỚI: spec này, và mọi doc thuộc fork
```

### `compose.dev.yaml` — nội dung thiết kế

Override service `photoprism`:

- `image: photoprism/develop:resolute`
- `command: sleep infinity` — container sống độc lập với tiến trình app, để `make build-go`
  không bị kill khi app restart
- App được chạy **thủ công** bên trong container: `docker compose exec photoprism
  ./photoprism start`. Port mapping `8098:2342` giữ nguyên từ `compose.yaml` nên URL không
  đổi. Vòng lặp sửa Go = Ctrl-C app → `make build-go` → `./photoprism start` lại.
  `dev.sh` sẽ gói lại thành một lệnh
- Dùng chung db `photoprism` trên service `mariadb` hiện có, cùng user/password trong
  `compose.yaml` — không tạo db riêng
- Mount thêm `/Volumes/BKM/git/photoprism` → `/go/src/github.com/photoprism/photoprism`
- **Giữ nguyên** mount `/Volumes/BKM/memories` và `./storage`, dùng chung service
  `mariadb` đang chạy → cùng data, cùng thumbnail cache
- `PHOTOPRISM_ASSETS_PATH` trỏ vào `assets/` trong repo (frontend build output +
  TensorFlow model nằm ở đó)
- `PHOTOPRISM_STORAGE_PATH` và `PHOTOPRISM_ORIGINALS_PATH` giữ đúng đường dẫn cũ
  (`/photoprism/storage`, `/photoprism/originals`) để cache và sidecar dùng lại được
- `GOCACHE` trỏ vào **named volume** riêng, không để trong repo: bind mount trên macOS
  (VirtioFS) chậm, và disk BKM chỉ còn 30 GB

Không dùng `compose.yaml` dev gốc của repo — nó kéo theo traefik, postgres,
dummy-webdav, dummy-oidc và một bộ data riêng, không cần cho mục đích này.

## Vòng lặp làm việc

| Việc | Lệnh | Thời gian |
| --- | --- | --- |
| Setup lần đầu | `make dep build-all` trong container | ~15–25 phút (tải TF model + npm) |
| Sửa Go | `make build-go`, restart binary | ~1–3 phút |
| Sửa Vue | `make watch-js` chạy nền | vài giây |
| Bake image gọn | `make docker-local` trên host | 25–50 phút |

Cập nhật upstream định kỳ:

```bash
git fetch upstream --tags
git rebase <tag-mới> custom
make docker-local
```

Conflict chỉ xảy ra ở file đã sửa → nên tách code riêng ra file mới khi có thể.

## An toàn dữ liệu

Chuyển 260601 → 260728 chạy migration một chiều. Trước khi switch:

1. `mariadb-dump` db `photoprism` ra file gzip trong `/Volumes/BKM/photoprism/`
2. `photoprism backup -i` (albums + metadata YAML)
3. Ghi lại số photo / số file / số album để đối chiếu sau
4. Kiểm tra dung lượng disk còn đủ (~10–15 GB); nếu thiếu thì **báo lại, không tự dọn** —
   người dùng tự quản lý disk

Đường lùi nếu vỡ: đổi `image:` về `photoprism/photoprism:260601-a7d098548` và restore dump.

## Kiểm chứng

Sau khi switch, xác nhận bằng bằng chứng thật:

- `photoprism --version` → `<builddate>-260728-bbde8f452-Linux-ARM64`
  (bản `make build-go` có thêm hậu tố `-DEVELOP`). Lưu ý: `scripts/build.sh` **không** gắn
  hậu tố edition — `-Plus` của image chính thức do pipeline riêng của upstream chèn.
  `photoprism edition` in ra `ce`.
- `/api/v1/status` trả `{"status":"operational"}`
- Số photo / file / album **khớp** số ghi ở bước an toàn dữ liệu
- Log migration không có error
- Mở UI trên browser: xem được một ảnh, thumbnail render đúng

## Về edition CE vs Plus

Build từ repo này ra edition `ce`. Đã kiểm tra: `Config.Edition()` mặc định trả `"ce"`,
và giá trị này chỉ dùng để gắn nhãn (version string, bảng migration, client config, MCP
API). Chỉ edition `portal` mới gate hành vi thật (`config_cluster.go`). Không mất tính
năng khi chuyển từ Plus sang CE build.

## Ngoài scope (YAGNI)

- traefik / TLS reverse proxy
- dev compose gốc của repo (postgres, dummy-webdav, dummy-oidc)
- CI/CD
- multi-arch buildx
- Tách môi trường dev riêng khỏi môi trường đang chạy — người dùng xác nhận chỉ cần một

## Vấn đề tồn đọng, không thuộc scope lần này

Log 20 giờ gần nhất có 59 error, ghi lại để xử lý sau:

- ~46 video không tạo được preview image (`.mov`, `.mp4`, phần lớn thuộc `2020/`)
- 62 file JPG bị skip khi index (`could not be identified`)
- 1 HEIC không convert được (`2026/20260618183700.heic`)
- 1 JPG corrupt gây vips panic-stack (`2017/`)
- `PHOTOPRISM_SITE_URL` sai port (2342 thay vì 8098) — sửa trong lần này
- Mật khẩu admin và DB vẫn là mặc định `insecure`
