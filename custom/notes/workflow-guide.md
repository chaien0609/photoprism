# Hướng dẫn customize PhotoPrism và deploy

Dành cho người chưa quen codebase này. Mọi lệnh và ví dụ trong đây đã chạy thật, không phải phỏng đoán.

---

## 1. Bức tranh tổng thể

### Ảnh đi từ đâu tới đâu

```
iCloud
  │  icloudpd (Docker, 5 phút/lần)          ← cần đăng nhập ở http://localhost:8070
  ▼
/Volumes/BKM/tools/icloudpd/Photos
  │  sort-media (launchd, 300 giây)          ← đổi tên + phân thư mục theo năm
  ▼
/Volumes/BKM/memories                        ← PhotoPrism đọc chỗ này (originals)
  │  PhotoPrism index (5 phút/lần)
  ▼
MariaDB + storage/cache (thumbnail)
```

Bạn **không** sửa gì trong pipeline này khi customize. Nó chỉ liên quan khi ảnh không xuất hiện.

### Code đi từ đâu tới đâu

```
/Volumes/BKM/git/photoprism          (source, branch "custom")
  │
  ├── chế độ DEV: mount trực tiếp vào container, build tại chỗ (36 giây)
  │     └─► container photoprism/develop:resolute ──► http://localhost:8098
  │
  └── chế độ IMAGE: bake thành image (15 phút)
        └─► photoprism/photoprism:local ──► http://localhost:8098
```

Cả hai chế độ dùng **cùng** ảnh gốc, cùng storage, cùng database. Chỉ khác cách binary được tạo ra.

### Vì sao cần hai chế độ

`./dev.sh bake` chạy `docker build --no-cache`, và toàn bộ `make all install` nằm trong **một** RUN layer — sửa một dòng Go cũng rebuild lại từ đầu, 15 phút. Không dùng được để thử nghiệm.

| | Chế độ dev | Chế độ image |
| --- | --- | --- |
| Sửa Go → thấy kết quả | ~40 giây | ~15 phút |
| Sửa Vue → thấy kết quả | vài giây | ~15 phút |
| Giống production | gần giống | giống hệt |
| Dùng khi | đang code | chạy thường ngày |

**Nguyên tắc:** code ở chế độ dev, khi xong thì bake sang chế độ image.

---

## 2. Cái gì ở đâu

### Hai thư mục, đừng lẫn

| Thư mục | Là gì | Có trong git? |
| --- | --- | --- |
| `/Volumes/BKM/git/photoprism` | **source code**. Sửa code ở đây. | Có, branch `custom` |
| `/Volumes/BKM/photoprism` | **deployment**: compose, `dev.sh`, storage 134 GB | Không |

Mọi lệnh vận hành chạy từ `/Volumes/BKM/photoprism` (nơi có `dev.sh`).
Mọi lệnh git chạy từ `/Volumes/BKM/git/photoprism`.

### Frontend — `frontend/src/`

| Đường dẫn | Chứa gì |
| --- | --- |
| `page/` | Một file `.vue` cho mỗi trang: `photos.vue`, `albums.vue`, `labels.vue`, `places.vue`, `people.vue`, `settings/`, `admin.vue`, `auth/login.vue`… |
| `component/` | Component tái dùng: `lightbox/`, `album/`, `label/`, `input/`, `icon/`, `dialogs.vue`… |
| `model/` | Lớp dữ liệu, gọi API và map sang object: `photo.js`, `album.js`, `label.js`, `file.js`… |
| `common/` | Hạ tầng: `api.js` (HTTP client), `config.js`, `session.js`, `util.js` |
| `app/routes.js` | Định nghĩa URL → trang. Ví dụ `path: "/about"` |
| `locales/*.po` | 45 ngôn ngữ. Bản dịch. |
| `css/` | Style |

### Backend — `internal/`

| Đường dẫn | Chứa gì |
| --- | --- |
| `api/` | 108 file handler HTTP. **Chỉ là lớp keo**, không chứa business logic. |
| `server/routes.go` | Nơi khai báo mọi endpoint. Thêm API mới phải đăng ký ở đây. |
| `commands/` | Lệnh CLI: `start.go`, `index.go`, `import.go`, `migrate.go`… |
| `photoprism/` | **Logic cốt lõi**: index, import, convert, thumbnail, faces, cleanup |
| `entity/` | Model GORM (bảng DB), query, migration |
| `config/` | Cấu hình, đọc flag/env, client config |
| `workers/` | Job chạy nền theo lịch: index, backup, meta, vision |
| `thumb/`, `ffmpeg/`, `meta/` | Xử lý ảnh, video, metadata |
| `ai/vision/` | Pipeline computer vision (nasnet, facenet, nsfw) |
| `auth/` | ACL, session, OIDC |

Repo có `CODEMAP.md` ở gốc — bản đồ chi tiết hơn, đọc khi cần đào sâu.

**Quy tắc kiến trúc của upstream:** `pkg/*` không bao giờ được import `internal/*`. Business logic nằm ở `internal/photoprism`, không nằm ở `internal/api`.

---

## 3. Quy tắc vàng: giữ code sửa dễ rebase

Mỗi lần cập nhật upstream, bạn sẽ `git rebase <tag-mới> custom`. Conflict chỉ xảy ra ở **file bạn đã sửa**. Nên:

1. **Ưu tiên thêm file mới** hơn là sửa file có sẵn. File mới không bao giờ conflict.
   - Thêm API? Tạo `internal/api/custom_xxx.go` mới, đừng nhét vào file có sẵn.
   - Thêm component? Tạo file `.vue` mới trong `component/`.
2. **Khi buộc phải sửa file upstream**, sửa ít dòng nhất có thể.
3. **Gom code riêng vào commit riêng** với prefix `Custom:` để dễ nhìn:
   ```bash
   git log --oneline 260728-bbde8f452..custom
   ```
4. **Kiểm anchor trước khi sửa.** Upstream có thể đã đổi đoạn code bạn dựa vào:
   ```bash
   grep -c '// Initialize the index database.' internal/commands/start.go   # phải ra 1
   ```
5. Tài liệu, spec, ghi chú của bạn đặt trong `custom/` — thư mục này không tồn tại trên upstream.

Hiện code riêng của bạn chỉ có **một** thay đổi vào file upstream: dòng log marker trong `internal/commands/start.go:92`.

---

## 4. Bật môi trường dev

```bash
cd /Volumes/BKM/photoprism
./dev.sh status      # xem đang ở chế độ nào
./dev.sh up          # chuyển sang chế độ dev — app CHƯA chạy
./dev.sh run         # chạy app (foreground, giữ terminal này)
```

Mở http://localhost:8098 — vẫn là thư viện thật của bạn, 78k ảnh.

Kiểm chứng đang chạy đúng bản từ source:

```bash
./dev.sh version
# → photoprism version 260730-<sha>-Linux-ARM64-DEVELOP
```

Hậu tố `-DEVELOP` cho biết đây là bản `make build-go` (chế độ dev). Bản bake ra image **không** có hậu tố này.

Lần đầu tiên trên máy mới cần thêm một bước tải dependency (~2 phút):

```bash
./dev.sh shell
make dep          # tải 4 model TensorFlow/ONNX (229 MB) + npm ci (1240 package)
exit
```

---

## 5. Sửa frontend (Vue)

### Vòng lặp

Mở **hai** terminal:

```bash
# Terminal 1 — app chạy
cd /Volumes/BKM/photoprism && ./dev.sh run

# Terminal 2 — tự build lại frontend khi bạn lưu file
cd /Volumes/BKM/photoprism && ./dev.sh watch-js
```

Sửa file `.vue` → lưu → watch-js build lại → **reload browser** là thấy. Không cần restart app, vì Go chỉ serve file tĩnh từ `assets/static/build/`.

Nếu không muốn chạy watch liên tục thì build một lần: `./dev.sh build-js` (31–49 giây).

### Cách tìm code của một thứ trên UI

Cách nhanh nhất là grep chuỗi chữ bạn thấy trên màn hình:

```bash
cd /Volumes/BKM/git/photoprism
grep -rn "Recently added" frontend/src/ | head
```

Hoặc đi từ URL: URL `/albums` → tra trong `frontend/src/app/routes.js` xem `path: "/albums"` trỏ tới component nào → mở file đó trong `page/`.

### Ví dụ: đổi một chuỗi trên trang About

File: `frontend/src/page/about/about.vue`

```vue
{{ $gettext("Our mission is to provide the most user- and privacy-friendly solution to keep your pictures organized and accessible.") }}
```

Đổi thành:

```vue
{{ $gettext("Thư viện ảnh gia đình của tôi.") }}
```

Rồi build và reload browser.

### Về i18n — `$gettext` hoạt động thế nào

Chuỗi trong `$gettext("...")` là **chuỗi gốc tiếng Anh**, đồng thời là khoá tra bản dịch. Khi UI đang ở tiếng Việt:

- Nếu chuỗi có bản dịch trong `frontend/src/locales/vi.po` → hiện bản dịch.
- Nếu **không** có → hiện đúng chuỗi gốc bạn viết.

Nên nếu bạn viết trực tiếp tiếng Việt vào `$gettext(...)` thì nó hiện tiếng Việt ở mọi ngôn ngữ. Muốn làm đúng bài bản thì viết tiếng Anh rồi thêm bản dịch:

```bash
./dev.sh shell
make gettext-extract     # quét code, cập nhật file .po với chuỗi mới
# sửa frontend/src/locales/vi.po, thêm msgstr cho msgid mới
make gettext-compile     # biên dịch .po → .json để frontend dùng
exit
./dev.sh build-js
```

Với thư viện cá nhân dùng một mình, viết thẳng tiếng Việt vào `$gettext` là chấp nhận được.

### Cách frontend gọi API

```js
import $api from "common/api";

// GET
const res = await $api.get("custom/hello");
console.log(res.data);

// POST
await $api.post("photos/" + uid + "/like");
```

`$api` đã tự thêm tiền tố `/api/v1/` và tự gắn token session. Xem `frontend/src/common/api.js`.

### Cạm bẫy đã gặp thật

Đừng kiểm chứng build frontend bằng **mtime** của file trong `assets/static/build/`. Webpack mặc định `output.compareBeforeEmit: true` — nội dung không đổi thì nó **không ghi lại file**, mtime giữ nguyên dù build đã chạy xong. Kiểm bằng cách grep nội dung:

```bash
grep -l 'chuỗi bạn vừa thêm' /Volumes/BKM/git/photoprism/assets/static/build/*.js
```

Khi nội dung thật sự đổi, tên file cũng đổi (hash theo nội dung): `app.bb16352a...js` → `app.38906a59...js`.

---

## 6. Sửa backend (Go)

### Vòng lặp

```bash
cd /Volumes/BKM/photoprism
# sửa code trong /Volumes/BKM/git/photoprism/internal/...
./dev.sh restart     # = stop-app + build-go (36 giây) + run
```

`restart` là một lệnh gộp. Nếu muốn tách:

```bash
./dev.sh stop-app    # BẮT BUỘC trước khi run lại
./dev.sh build-go
./dev.sh run
```

**Vì sao bắt buộc `stop-app`:** Ctrl-C ở terminal chỉ giết client `docker compose exec`, còn tiến trình app **trong container** vẫn sống và giữ cổng 2342. Lần `run` sau sẽ báo `bind: address already in use` rồi tự tắt — và cổng vẫn do binary **cũ** phục vụ. Rất dễ tưởng đã restart thành công trong khi code mới chưa chạy.

### Ví dụ: thêm một API endpoint mới

Đây là ví dụ đã được compile và `go vet` kiểm chứng.

**Bước 1** — tạo file mới `internal/api/custom_hello.go` (file mới → không bao giờ conflict khi rebase):

```go
package api

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// GetCustomHello responds with a custom greeting, as an example endpoint.
//
//	@Summary	responds with a custom greeting
//	@Id			GetCustomHello
//	@Tags		Debug
//	@Produce	json
//	@Success	200	{object}	gin.H
//	@Router		/api/v1/custom/hello [get]
func GetCustomHello(router *gin.RouterGroup) {
	router.GET("/custom/hello", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"hello": "from my own build"})
	})
}
```

Khối comment `@Summary`/`@Router` là annotation Swagger — không bắt buộc để chạy, nhưng giữ đúng quy ước upstream và giúp `make swag` sinh tài liệu API.

**Bước 2** — đăng ký trong `internal/server/routes.go`, mục `// Technical Endpoints.`:

```go
	api.GetSvg(APIv1)
	api.GetStatus(APIv1)
	api.GetCustomHello(APIv1)      // ← thêm dòng này
```

**Bước 3** — build và chạy:

```bash
cd /Volumes/BKM/photoprism
./dev.sh restart
```

**Bước 4** — kiểm chứng:

```bash
curl -s http://localhost:8098/api/v1/custom/hello
# → {"hello":"from my own build"}
```

Mẫu này copy từ `internal/api/status.go` — handler đơn giản nhất trong repo (20 dòng), đáng đọc để hiểu quy ước.

### Ví dụ: sửa logic có sẵn

Giả sử muốn thay đổi cách index xử lý file. Đường đi để tìm code:

```bash
cd /Volumes/BKM/git/photoprism
# 1. Tìm từ thông điệp log bạn thấy
grep -rn "could not create preview image" internal/ | head

# 2. Hoặc tìm từ điểm vào: lệnh CLI
#    internal/commands/index.go → internal/photoprism/index*.go
ls internal/photoprism/index*.go
```

Business logic nằm trong `internal/photoprism/`, không nằm trong `internal/api/`.

### Chạy test

```bash
./dev.sh shell
go test ./internal/api/            # test một package
go vet ./internal/api/             # kiểm lỗi tĩnh
make lint-go                       # golangci-lint, in cảnh báo, không fail
exit
```

Full suite `go test ./...` cần MariaDB test instance riêng và rất lâu — chỉ chạy khi thật cần.

### Xem log app

Log hiện ngay trên terminal đang chạy `./dev.sh run`. Muốn debug chi tiết hơn, chế độ dev đã bật `PHOTOPRISM_DEBUG: "true"` nên có cả log `level=debug`.

---

## 7. Deploy — bake thành image

Khi thay đổi đã ổn:

```bash
cd /Volumes/BKM/photoprism
df -h /              # cần ≥ 10Gi trống, KHÔNG tự chạy lệnh dọn disk
./dev.sh bake        # ~15 phút
```

Bake sinh 3 tag cùng lúc: `photoprism/photoprism:local`, `:ce-resolute`, `:260730-ce-resolute` (tag ngày). `compose.yaml` trỏ vào `:local`.

Trong lúc bake, app ở chế độ dev **vẫn phục vụ bình thường** — bake chạy trên host, không đụng container đang chạy.

Chuyển sang chạy bằng image:

```bash
./dev.sh stop-app    # dừng app dev trước
./dev.sh prod        # tắt chế độ dev, chạy image :local
```

Kiểm chứng sau deploy:

```bash
docker exec photoprism-photoprism-1 photoprism --version
# → 260730-<sha>-Linux-ARM64   (KHÔNG có "-DEVELOP")

curl -s http://localhost:8098/api/v1/status
# → {"status":"operational"}

docker logs photoprism-photoprism-1 2>&1 | grep 'custom: running build'
# → phải thấy dòng này, xác nhận code riêng đã vào image

docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL;"
# → 78715 (hoặc hơn nếu có ảnh mới)
```

Dòng `custom: running build from local source (...)` là marker chủ ý — nó ở `internal/commands/start.go:92` để bạn luôn phân biệt được đang chạy bản của mình hay bản upstream.

### Commit và push code riêng

```bash
cd /Volumes/BKM/git/photoprism
git add internal/api/custom_hello.go internal/server/routes.go
git commit -m "Custom: Add hello endpoint"

gh auth switch --user chaien0609     # BẮT BUỘC, xem mục 9
git push origin custom
gh auth switch --user core-cuong     # trả lại account cũ
```

---

## 8. Cập nhật code mới từ upstream

Xem `custom/notes/upstream-update-runbook.md` cho quy trình đầy đủ. Rút gọn:

```bash
cd /Volumes/BKM/git/photoprism
git fetch upstream --tags
git tag --sort=-creatordate | head -5      # chọn tag release mới nhất
git rebase <tag-mới> custom                # rebase lên TAG, không phải develop HEAD
cd /Volumes/BKM/photoprism && ./dev.sh bake && ./dev.sh prod
```

Hai điều bắt buộc trước khi update:
1. **Backup DB** — migration là một chiều, không lùi được nếu không có backup.
2. **So ENV của image mới.** Upstream có đổi ENV giữa các bản: bản 260728 đã bỏ `THUMB_SIZE`, `JPEG_SIZE`, `PNG_SIZE`, `THUMB_SIZE_UNCACHED` khỏi ENV image và đổi `THUMB_UNCACHED` từ `true` sang `false`. Vì vậy 5 biến đó giờ được ghim tường minh trong `compose.yaml`. Nếu không để ý, hành vi thumbnail đổi âm thầm.

---

## 9. Cạm bẫy — tất cả đều đã xảy ra thật

### `compose.dev.yaml` bắt buộc có `PHOTOPRISM_STORAGE_PATH`

Image `photoprism/photoprism` đặt biến này trong ENV của image; image `photoprism/develop` **không** có. Thiếu nó, PhotoPrism fallback về `<originals>/.photoprism/storage`, dẫn tới: mất sidecar (preview video/HEIC thành icon lỗi), bỏ qua cache 134 GB, ghi rác vào thư mục ảnh gốc.

Kiểm bất cứ lúc nào:

```bash
docker exec photoprism-photoprism-1 photoprism config | grep -E 'storage-path|sidecar-path|thumb-cache-path'
```

Phải ra `/photoprism/storage`, `/photoprism/storage/sidecar`, `/photoprism/storage/cache/thumbnails`.
Nếu thấy `/photoprism/originals/.photoprism/...` là sai.

### Thumbnail lỗi vẫn trả HTTP 200

`internal/api/thumbnails.go:168` trả **200 kèm SVG icon lỗi** khi không resolve được file. Còn token hết hiệu lực thì trả **403 kèm SVG**. Nên luôn in cả hai:

```bash
curl -s -o /dev/null -w 'HTTP %{http_code} %{content_type}\n' "http://localhost:8098/api/v1/t/$H/$TOK/tile_500"
```

- `200` + `image/jpeg` → đúng
- `200` + `image/svg+xml` → lỗi resolve file, kiểm storage path
- `403` + `image/svg+xml` → token hết hạn, không liên quan thumbnail

### `docker compose run <service> photoprism config` cho kết quả sai

Entrypoint `/init` xoá sạch biến `PHOTOPRISM_*` trước khi CMD chạy, nên bạn nhận về code default chứ không phải cấu hình thật. Muốn đọc ENV của image thì bypass entrypoint:

```bash
docker run --rm --entrypoint /bin/bash <image> -c 'echo $PHOTOPRISM_STORAGE_PATH'
```

### Push lên fork cần đúng account

Máy có 2 account `gh`. Nếu active account là `core-cuong` thì push báo:

```
remote: Permission to chaien0609/photoprism.git denied to core-cuong
```

Xử lý: `gh auth switch --user chaien0609` → push → switch lại.

### `PHOTOPRISM_ADMIN_PASSWORD` trong compose không phải password hiện tại

Biến đó chỉ áp dụng ở lần setup đầu tiên. Password admin đã đổi sau đó.

### Đừng thêm `tensorflow` vào `PHOTOPRISM_INIT`

Image đã có TensorFlow 2.18.0 trong `/opt/photoprism/lib/`, binary có `RUNPATH: $ORIGIN/../lib` nên tìm được. Thêm `tensorflow` vào `INIT` chỉ khiến mỗi lần start tải lại ~200 MB — khởi động từ 16 giây thành ~7 phút.

### Đừng chạy `make start` từ repo source

Đó là dev environment của upstream, nó kéo theo traefik/postgres/dummy-oidc và tạo storage riêng ở gốc repo. Dùng `./dev.sh` thay thế.

### Đừng chạy lệnh dọn disk

`docker system prune`, `docker volume rm`, `docker image prune` — không tự chạy. Disk mỏng (~20 GB nội bộ, ~33 GB trên BKM) nhưng việc dọn do bạn quyết.

---

## 10. Cheat sheet

```bash
cd /Volumes/BKM/photoprism      # mọi lệnh vận hành từ đây

# Xem tình hình
./dev.sh status

# Code frontend
./dev.sh up && ./dev.sh run      # terminal 1
./dev.sh watch-js                # terminal 2, rồi reload browser

# Code backend
./dev.sh restart                 # sau mỗi lần sửa Go, ~40 giây

# Vào container
./dev.sh shell

# Deploy
./dev.sh bake                    # ~15 phút
./dev.sh stop-app && ./dev.sh prod

# Về chế độ dev
./dev.sh up && ./dev.sh run
```

| Việc | Thời gian thật |
| --- | --- |
| `build-go` | 36 giây |
| `build-js` | 31–49 giây |
| `bake` | ~15 phút |
| khởi động chế độ image | 16 giây |
| `make dep` (lần đầu) | ~2 phút |

### Tài liệu liên quan

| File | Nội dung |
| --- | --- |
| `custom/specs/2026-07-30-build-from-source-design.md` | Thiết kế: vì sao làm như vậy |
| `custom/plans/2026-07-30-build-from-source.md` | Các bước setup ban đầu, có thể chạy lại |
| `custom/notes/2026-07-30-migration-log.md` | Log setup + mọi sự cố đã gặp và cách xử lý |
| `custom/notes/upstream-update-runbook.md` | Quy trình cập nhật upstream |
| `CODEMAP.md` (gốc repo) | Bản đồ backend chi tiết của upstream |
| `AGENTS.md` (gốc repo) | Quy ước code của upstream |
| `/Volumes/BKM/photoprism/CLAUDE.md` | Context deployment |
