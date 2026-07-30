# Kế hoạch implement: chạy PhotoPrism từ source repo local

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Chuyển deployment PhotoPrism tại `/Volumes/BKM/photoprism` từ image pull sẵn sang chạy code build từ `/Volumes/BKM/git/photoprism`, giữ nguyên 78k ảnh + 134 GB cache + DB, và có vòng lặp sửa code đủ nhanh.

**Architecture:** Hai chế độ trên cùng một compose project `photoprism`, cùng data. Chế độ dev override service `photoprism` sang `photoprism/develop:resolute` với source mount vào — sửa Go rồi `make build-go` (~1–3 phút). Chế độ image chạy `photoprism/photoprism:local` bake bằng `make docker-local`. Data (`storage/` 134 GB, originals, volume `photoprism_database`) không di chuyển, chỉ trỏ bằng đường dẫn tuyệt đối.

**Tech Stack:** Docker Compose, Go (trong container), Vue 3 + npm (trong container), MariaDB 11, macOS arm64.

**Spec:** `custom/specs/2026-07-30-build-from-source-design.md`

## Global Constraints

- Deployment dir: `/Volumes/BKM/photoprism` — **không phải git repo**, không đưa vào git (password không được vào git).
- Source repo: `/Volumes/BKM/git/photoprism`, branch `custom`. Mọi file thuộc fork đặt trong `custom/`.
- Compose project name phải là `photoprism` để giữ volume `photoprism_database`. Luôn chạy `docker compose` từ cwd `/Volumes/BKM/photoprism`.
- **Không** chạy `docker system prune` hay bất kỳ lệnh dọn disk nào — người dùng tự quản lý. Nếu thiếu chỗ thì dừng và báo.
- **Không** copy hay move `storage/` (134 GB, BKM chỉ còn ~30 GB trống).
- Base image: `photoprism/develop:resolute` (đã xác minh có arm64). Prod stage dùng `photoprism/develop:resolute-slim`.
- Version string sinh từ `scripts/build.sh`: `$(date -u +%y%m%d)-$(git describe --always)-Linux-ARM64`, thêm hậu tố `-DEVELOP` khi build bằng `make build-go`. Tag `260728-bbde8f452` là lightweight nên `git describe` trả SHA trần, **không** trả tên tag.
- `git describe` chạy trong container cần config `safe.directory`. Truyền qua env `GIT_CONFIG_COUNT=1 / GIT_CONFIG_KEY_0=safe.directory / GIT_CONFIG_VALUE_0=/go/src/github.com/photoprism/photoprism` — không sửa global git config trong container.
- `photoprism/develop:resolute` đã có `ENTRYPOINT ["/init"]` + `CMD ["/scripts/cmd.sh", "tail", "-f", "/dev/null"]` → container tự sống, **không** set `command:`.
- Trong chế độ dev phải override `PHOTOPRISM_INIT: ""` (develop image đã có TensorFlow, không cần cài lại mỗi lần start) và `PHOTOPRISM_INDEX_SCHEDULE: ""` (tránh auto-index 134k file khi đang thử nghiệm).
- **BẮT BUỘC** đặt `PHOTOPRISM_STORAGE_PATH: "/photoprism/storage"` trong `compose.dev.yaml`. Image `photoprism/photoprism` đặt biến này trong ENV của image; image `photoprism/develop` **không** có, và `compose.yaml` cũng không đặt. Thiếu nó thì PhotoPrism fallback về default `<originals>/.photoprism/storage`, dẫn tới: không thấy sidecar thật (video/HEIC preview trả về SVG icon lỗi kèm **HTTP 200**, rất dễ tưởng là bình thường), bỏ qua thumbnail cache 134 GB, và ghi rác vào thư mục ảnh gốc. Đặt kèm `PHOTOPRISM_ORIGINALS_PATH`, `PHOTOPRISM_IMPORT_PATH`, và nhóm thumbnail (`THUMB_SIZE` 1920, `THUMB_SIZE_UNCACHED` 7680, `THUMB_UNCACHED` true, `JPEG_SIZE`/`PNG_SIZE` 7680) vì code default lệch với ENV của image prod (5120 / false).
- Đừng dùng `docker compose run <service> photoprism config` để lấy config tham chiếu từ image prod: entrypoint `/init` xoá sạch biến `PHOTOPRISM_*` trước khi CMD chạy, nên kết quả là code default chứ không phải cấu hình thật. Muốn đọc ENV của image thì bypass entrypoint: `docker run --rm --entrypoint /bin/bash <image> -c 'echo $PHOTOPRISM_STORAGE_PATH'`.
- Git identity repo-local đã set: `chaien0609 <cuongnh0609@gmail.com>`.

---

### Task 1: Baseline, backup, và cổng kiểm tra disk

Chuyển 260601 → bản mới sẽ chạy DB migration một chiều. Task này tạo đường lùi trước khi có bất kỳ thay đổi nào.

**Files:**
- Create: `/Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz` (dump DB)
- Create: `custom/notes/2026-07-30-migration-log.md` (baseline + log tiến trình)

**Interfaces:**
- Produces: file dump ở đường dẫn trên; 3 con số baseline (photos / files / albums) ghi trong `custom/notes/2026-07-30-migration-log.md` — Task 6 và Task 8 sẽ so lại với chúng.

- [ ] **Step 1: Cổng disk — dừng nếu không đủ chỗ**

```bash
df -h / | tail -1
df -h /Volumes/BKM | tail -1
```

Yêu cầu: cột `Avail` của `/` **≥ 15Gi** và của `/Volumes/BKM` **≥ 5Gi`.
Nếu không đạt: **DỪNG toàn bộ kế hoạch**, báo người dùng con số thực tế và để họ tự dọn. Không tự chạy lệnh dọn nào.

- [ ] **Step 2: Ghi lại số liệu baseline**

```bash
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL) AS photos, \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL) AS files, \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL) AS albums;"
docker exec photoprism-photoprism-1 photoprism --version
```

Kỳ vọng: 3 số nguyên, và version `260601-a7d098548-Linux-ARM64-Plus`.
Số đo lúc viết kế hoạch để đối chiếu: **78715 photos / 134383 files / 325 albums**. Số thực tế lúc chạy có thể cao hơn (auto-index mỗi 5 phút) — dùng số vừa đo, không dùng số này.

- [ ] **Step 3: Dump DB ra file gzip**

```bash
docker exec photoprism-mariadb-1 mariadb-dump -u root -pinsecure \
  --single-transaction --quick --routines --events photoprism \
  | gzip > /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz
```

DB khoảng 2.2 GB nên bước này mất vài phút.

- [ ] **Step 4: Kiểm chứng dump không hỏng**

```bash
gunzip -t /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz && echo "GZIP OK"
gunzip -c /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz | grep -c 'CREATE TABLE'
ls -lh /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz
```

Kỳ vọng: in ra `GZIP OK`, số `CREATE TABLE` **≥ 30**, file size **≥ 100M**.
Nếu số `CREATE TABLE` < 30 hoặc file < 100M → dump lỗi, làm lại Step 3, không đi tiếp.

- [ ] **Step 5: Backup albums + metadata YAML**

```bash
docker exec photoprism-photoprism-1 photoprism backup --albums --force
ls /Volumes/BKM/photoprism/storage/backup/albums | head -5
ls /Volumes/BKM/photoprism/storage/backup/albums | wc -l
```

Kỳ vọng: số file YAML **≥ 300** (baseline có 325 album).

- [ ] **Step 6: Ghi log và commit**

Tạo `custom/notes/2026-07-30-migration-log.md` với nội dung (thay 3 số và version bằng số thật vừa đo ở Step 2):

```markdown
# Log chuyển sang build từ source — 2026-07-30

## Baseline trước khi chuyển

| Chỉ số | Giá trị |
| --- | --- |
| photos | <số từ Step 2> |
| files | <số từ Step 2> |
| albums | <số từ Step 2> |
| version đang chạy | <version từ Step 2> |
| DB size | 2.2 GB |
| storage size | 134 GB |

## Đường lùi

1. Sửa `image:` trong `/Volumes/BKM/photoprism/compose.yaml` về `photoprism/photoprism:260601-a7d098548`
2. `docker compose down`
3. `docker exec -i photoprism-mariadb-1 mariadb -u root -pinsecure photoprism < <(gunzip -c /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz)`
4. `docker compose up -d`

Dump: `/Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz`
Albums YAML: `/Volumes/BKM/photoprism/storage/backup/albums/`

## Tiến trình

- [x] Task 1 — baseline + backup
```

```bash
cd /Volumes/BKM/git/photoprism
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Record baseline and rollback path before source migration"
```

---

### Task 2: Fork GitHub và cấu hình remote

**Files:**
- Modify: `.git/config` (qua lệnh `git remote`, không sửa tay)
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: `custom/notes/2026-07-30-migration-log.md` từ Task 1.
- Produces: remote `origin` = fork của user, remote `upstream` = photoprism/photoprism. Task 9 (runbook) dựa vào đúng hai tên này.

- [ ] **Step 1: Xác nhận trạng thái remote hiện tại trước khi đổi**

```bash
cd /Volumes/BKM/git/photoprism
git remote -v
git branch --show-current
git log --oneline -3
```

Kỳ vọng: `origin` trỏ `https://github.com/photoprism/photoprism.git`, branch hiện tại là `custom`, có commit `Custom: Record baseline...` ở trên cùng.

- [ ] **Step 2: Tạo fork trên GitHub**

```bash
gh repo fork photoprism/photoprism --clone=false
```

Kỳ vọng: in ra URL `https://github.com/chaien0609/photoprism` (hoặc "already exists" nếu đã có — cũng OK).

Không thêm `--remote=false`: gh 2.96 báo lỗi `the --remote flag is unsupported when a repository argument is provided`. Khi truyền tên repo, gh không tự thêm remote nên không cần flag đó.

- [ ] **Step 3: Kiểm chứng fork tồn tại**

```bash
gh repo view chaien0609/photoprism --json name,isFork,parent,visibility
```

Kỳ vọng: JSON có `"isFork": true`, `"parent"` là `photoprism/photoprism`.
Ghi nhận `visibility` — fork của repo public thì cũng public. Không sao vì compose (chứa password) không vào git.

- [ ] **Step 4: Đổi tên remote và thêm origin mới**

```bash
git remote rename origin upstream
git remote add origin https://github.com/chaien0609/photoprism.git
git remote -v
```

Kỳ vọng: `upstream` → photoprism/photoprism, `origin` → chaien0609/photoprism, mỗi cái 2 dòng fetch/push.

- [ ] **Step 5: Push branch custom lên fork**

```bash
git push -u origin custom
```

Kỳ vọng: push thành công, `branch 'custom' set up to track 'origin/custom'`. Chỉ vài commit được gửi đi vì fork đã có toàn bộ history upstream.

- [ ] **Step 6: Kiểm chứng push**

```bash
git rev-parse HEAD origin/custom
```

Kỳ vọng: hai dòng SHA **giống nhau**.

- [ ] **Step 7: Cập nhật log và commit**

Thêm vào cuối phần `## Tiến trình` của `custom/notes/2026-07-30-migration-log.md`:

```markdown
- [x] Task 2 — fork `chaien0609/photoprism`, origin/upstream đã đổi, branch `custom` đã push
```

```bash
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Point origin at personal fork, upstream at photoprism/photoprism"
git push
```

---

### Task 3: Pull base image và kiểm chứng môi trường build

**Files:** không tạo/sửa file. Task này chỉ tải image và xác nhận toolchain trong đó chạy được.

**Interfaces:**
- Produces: image `photoprism/develop:resolute` có sẵn local; Docker volume `photoprism-gocache`. Task 5 và 6 dùng cả hai.

- [ ] **Step 1: Pull image dev**

```bash
docker pull photoprism/develop:resolute
```

Mất khoảng 5–15 phút tùy mạng (image lớn, có TensorFlow + libheif + darktable).

- [ ] **Step 2: Kiểm chứng image và disk sau khi pull**

```bash
docker image inspect photoprism/develop:resolute --format '{{.Architecture}} {{.Os}} {{.Config.Entrypoint}} {{.Config.Cmd}}'
docker image ls photoprism/develop --format '{{.Repository}}:{{.Tag}} {{.Size}}'
df -h / | tail -1
```

Kỳ vọng: `arm64 linux [/init] [/scripts/cmd.sh tail -f /dev/null]`. Nếu `Architecture` không phải `arm64` → dừng, báo người dùng (đã build sai platform).
Nếu `Avail` của `/` tụt xuống **dưới 8Gi** → dừng, báo người dùng, không tự dọn.

- [ ] **Step 3: Kiểm chứng toolchain trong image chạy được với source mount**

```bash
docker run --rm \
  -v /Volumes/BKM/git/photoprism:/go/src/github.com/photoprism/photoprism \
  -w /go/src/github.com/photoprism/photoprism \
  -e GIT_CONFIG_COUNT=1 \
  -e GIT_CONFIG_KEY_0=safe.directory \
  -e GIT_CONFIG_VALUE_0=/go/src/github.com/photoprism/photoprism \
  --entrypoint /bin/bash \
  photoprism/develop:resolute -c \
  'go version && node --version && npm --version && git describe --always && echo TOOLCHAIN_OK'
```

Kỳ vọng: in go version, node version, npm version, một SHA ngắn, rồi `TOOLCHAIN_OK`.
Nếu `git describe` báo `detected dubious ownership` → biến `GIT_CONFIG_*` chưa có tác dụng; kiểm tra lại đúng chính tả 3 biến rồi chạy lại.

- [ ] **Step 4: Tạo volume cache cho Go**

Bind mount trên macOS (VirtioFS) chậm, và BKM chỉ còn ~30 GB — nên Go build cache đặt trong Docker volume.

```bash
docker volume create photoprism-gocache
docker volume ls --format '{{.Name}}' | grep photoprism-gocache
```

Kỳ vọng: in ra `photoprism-gocache`.

- [ ] **Step 5: Cập nhật log và commit**

Thêm vào `## Tiến trình`:

```markdown
- [x] Task 3 — pull `photoprism/develop:resolute` (arm64), tạo volume `photoprism-gocache`, toolchain OK
```

```bash
cd /Volumes/BKM/git/photoprism
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Verify build toolchain in develop:resolute image"
git push
```

---

### Task 4: Viết cấu hình chế độ dev

Chỉ viết file, **chưa** đổi container nào — instance hiện tại vẫn chạy bình thường suốt task này.

**Files:**
- Create: `/Volumes/BKM/photoprism/compose.dev.yaml`
- Create: `/Volumes/BKM/photoprism/dev.sh` (chmod +x)
- Modify: `/Volumes/BKM/photoprism/compose.yaml` (sửa `PHOTOPRISM_SITE_URL`)
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: image `photoprism/develop:resolute` và volume `photoprism-gocache` từ Task 3.
- Produces: `dev.sh` với các subcommand `up|down|shell|build-go|build-js|watch-js|run|bake|prod|logs|status`. Task 5–8 gọi các subcommand này.

- [ ] **Step 1: Sửa SITE_URL sai port trong compose.yaml**

Trong `/Volumes/BKM/photoprism/compose.yaml`, đổi đúng một dòng:

```yaml
      PHOTOPRISM_SITE_URL: "http://localhost:2342/"  # server URL in the format "http(s)://domain.name(:port)/(path)"
```

thành:

```yaml
      PHOTOPRISM_SITE_URL: "http://localhost:8098/"  # server URL in the format "http(s)://domain.name(:port)/(path)"
```

Port host thật là 8098 (mapping `8098:2342`); giá trị 2342 cũ làm link sinh trong app, share link và WebDAV trỏ sai cổng.

- [ ] **Step 2: Kiểm chứng compose.yaml vẫn hợp lệ**

```bash
cd /Volumes/BKM/photoprism
docker compose config --quiet && echo "COMPOSE OK"
docker compose config | grep PHOTOPRISM_SITE_URL
```

Kỳ vọng: `COMPOSE OK` và dòng SITE_URL hiện `http://localhost:8098/`.

- [ ] **Step 3: Tạo compose.dev.yaml**

Tạo `/Volumes/BKM/photoprism/compose.dev.yaml`:

```yaml
## Chế độ dev: chạy PhotoPrism từ source tại /Volumes/BKM/git/photoprism
##
## Dùng qua ./dev.sh, hoặc trực tiếp:
##   docker compose -f compose.yaml -f compose.dev.yaml up -d
##
## Data dùng chung với chế độ image: cùng originals, cùng storage, cùng volume
## photoprism_database. Chỉ khác: binary chạy từ source thay vì từ image bake sẵn.

services:
  photoprism:
    image: photoprism/develop:resolute
    ## KHÔNG set command: image đã có ENTRYPOINT ["/init"] + CMD tail -f /dev/null
    ## nên container tự sống, app được chạy thủ công bằng ./dev.sh run
    working_dir: "/go/src/github.com/photoprism/photoprism"
    environment:
      ## Assets nằm trong source (frontend build output + TensorFlow model)
      PHOTOPRISM_ASSETS_PATH: "/go/src/github.com/photoprism/photoprism/assets"
      ## BẮT BUỘC: image develop không có các biến này (image prod có trong ENV).
      ## Thiếu -> fallback "<originals>/.photoprism/storage" -> mất sidecar + bỏ qua cache 134 GB.
      PHOTOPRISM_STORAGE_PATH: "/photoprism/storage"
      PHOTOPRISM_ORIGINALS_PATH: "/photoprism/originals"
      PHOTOPRISM_IMPORT_PATH: "/photoprism/import"
      ## Không set PHOTOPRISM_BACKUP_PATH: suy ra từ STORAGE_PATH thành
      ## /photoprism/storage/backup — khớp thư mục đang có trên disk.
      ## Khớp hành vi thumbnail với image prod (code default khác: 5120 / false)
      PHOTOPRISM_THUMB_SIZE: 1920
      PHOTOPRISM_THUMB_SIZE_UNCACHED: 7680
      PHOTOPRISM_THUMB_UNCACHED: "true"
      PHOTOPRISM_JPEG_SIZE: 7680
      PHOTOPRISM_PNG_SIZE: 7680
      ## develop image đã có TensorFlow sẵn — để rỗng, không cài lại mỗi lần start
      PHOTOPRISM_INIT: ""
      ## Tắt auto-index khi đang thử nghiệm (134k file). Bật lại bằng cách xoá dòng này.
      PHOTOPRISM_INDEX_SCHEDULE: ""
      PHOTOPRISM_DEBUG: "true"
      ## Go build cache nằm trong Docker volume, không nằm trên bind mount (chậm)
      GOCACHE: "/go/cache"
      ## Cho phép git đọc repo mounted mà không cần sửa global git config
      GIT_CONFIG_COUNT: "1"
      GIT_CONFIG_KEY_0: "safe.directory"
      GIT_CONFIG_VALUE_0: "/go/src/github.com/photoprism/photoprism"
    volumes:
      - "/Volumes/BKM/git/photoprism:/go/src/github.com/photoprism/photoprism"
      - "gocache:/go/cache"

volumes:
  gocache:
    name: photoprism-gocache
    external: true
```

- [ ] **Step 4: Kiểm chứng merge hai compose file ra cấu hình đúng**

```bash
cd /Volumes/BKM/photoprism
docker compose -f compose.yaml -f compose.dev.yaml config --quiet && echo "DEV COMPOSE OK"
docker compose -f compose.yaml -f compose.dev.yaml config --format json | python3 -c "
import json,sys
c=json.load(sys.stdin)
print('project name:', c.get('name'))
for svc in ('photoprism','mariadb'):
    s=c['services'][svc]
    print(f'--- {svc}')
    print('  image   :', s.get('image'))
    print('  command :', s.get('command'))
    for v in s.get('volumes',[]):
        print('  vol     :', v.get('source'),'->',v.get('target'), v.get('type'))
print('--- volumes top-level:', {k:(v or {}).get('name') for k,v in (c.get('volumes') or {}).items()})
"
```

Đọc field đã merge thay vì `grep` text: `grep -c 'command:'` sẽ đếm cả `command` của mariadb (nó có 8 flag tuning) nên không phân biệt được service nào.

Kỳ vọng:
- `DEV COMPOSE OK`
- `project name: photoprism` — **bắt buộc**. Nếu ra tên khác → volume DB thành `<tên>_database` và DB trông như trống; phải thêm `name: photoprism` ở cấp trên cùng của compose.dev.yaml rồi kiểm lại.
- photoprism: image `photoprism/develop:resolute`, **`command : None`**
- photoprism có đủ 4 mount: `/Volumes/BKM/memories` → `/photoprism/originals`, `/Volumes/BKM/photoprism/storage` → `/photoprism/storage`, `/Volumes/BKM/git/photoprism` → `/go/src/github.com/photoprism/photoprism`, và `gocache` → `/go/cache`
- `volumes top-level` có `'database': 'photoprism_database'` — đúng volume DB đang dùng
- mariadb vẫn `image: mariadb:11` và giữ nguyên `command` 8 flag tuning

- [ ] **Step 6: Tạo dev.sh**

Tạo `/Volumes/BKM/photoprism/dev.sh`:

```bash
#!/usr/bin/env bash
# Wrapper cho hai chế độ chạy PhotoPrism.
#   Chế độ dev   : chạy binary build từ source ở /Volumes/BKM/git/photoprism
#   Chế độ image : chạy photoprism/photoprism:local bake bằng `make docker-local`
set -euo pipefail

cd "$(dirname "$0")"

REPO="/Volumes/BKM/git/photoprism"
DEV=(docker compose -f compose.yaml -f compose.dev.yaml)
IMG=(docker compose -f compose.yaml)

case "${1:-}" in
  up)       "${DEV[@]}" up -d ;;
  down)     "${DEV[@]}" down ;;
  shell)    "${DEV[@]}" exec photoprism bash ;;
  build-go) "${DEV[@]}" exec photoprism make build-go ;;
  build-js) "${DEV[@]}" exec photoprism make build-js ;;
  watch-js) "${DEV[@]}" exec photoprism make watch-js ;;
  run)      "${DEV[@]}" exec photoprism ./photoprism start ;;
  version)  "${DEV[@]}" exec photoprism ./photoprism --version ;;
  logs)     "${DEV[@]}" logs -f photoprism ;;
  bake)     ( cd "$REPO" && make docker-local ) ;;
  prod)     "${DEV[@]}" down && "${IMG[@]}" up -d ;;
  status)
    docker compose ls | grep -E 'NAME|photoprism' || true
    docker ps --filter name=photoprism --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
    curl -s -o /dev/null -w 'API HTTP %{http_code}\n' http://localhost:8098/api/v1/status || true
    ;;
  *)
    cat >&2 <<'USAGE'
usage: ./dev.sh <command>

  up        bật chế độ dev (container dev + mariadb), app CHƯA chạy
  run       chạy app từ source (foreground, Ctrl-C để dừng app, container vẫn sống)
  build-go  compile lại Go sau khi sửa code (~1-3 phút)
  build-js  build lại frontend một lần
  watch-js  build frontend liên tục khi file thay đổi (chạy nền, Ctrl-C để dừng)
  shell     vào bash trong container dev
  version   in version của binary đã build
  logs      xem log container dev
  down      tắt stack

  bake      build photoprism/photoprism:local từ source (25-50 phút)
  prod      tắt chế độ dev, chạy lại bằng image :local
  status    xem container nào đang chạy và API có sống không

Vòng lặp sửa Go:  Ctrl-C app  ->  ./dev.sh build-go  ->  ./dev.sh run
Vòng lặp sửa Vue: ./dev.sh watch-js (tab khác), reload browser
USAGE
    exit 1
    ;;
esac
```

- [ ] **Step 7: Kiểm chứng dev.sh**

```bash
chmod +x /Volumes/BKM/photoprism/dev.sh
/Volumes/BKM/photoprism/dev.sh 2>&1 | head -3
/Volumes/BKM/photoprism/dev.sh status
bash -n /Volumes/BKM/photoprism/dev.sh && echo "SYNTAX OK"
```

Kỳ vọng: không tham số thì in usage; `status` in container đang chạy và `API HTTP 200` (instance cũ vẫn đang sống); `SYNTAX OK`.

- [ ] **Step 8: Kiểm chứng instance hiện tại chưa bị ảnh hưởng**

```bash
docker ps --filter name=photoprism-photoprism-1 --format '{{.Image}} {{.Status}}'
curl -s http://localhost:8098/api/v1/status
```

Kỳ vọng: vẫn là `photoprism/photoprism:latest`, status `Up ...`, API trả `{"status":"operational"}`. Task này không được làm gián đoạn gì.

- [ ] **Step 9: Cập nhật log và commit**

Thêm vào `## Tiến trình`:

```markdown
- [x] Task 4 — tạo compose.dev.yaml + dev.sh, sửa SITE_URL về port 8098. Instance cũ vẫn chạy.
```

```bash
cd /Volumes/BKM/git/photoprism
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Add dev-mode compose override and dev.sh wrapper"
git push
```

---

### Task 5: Build lần đầu trong container tạm

Build trong container `--rm` riêng, **không** đụng service đang chạy → instance cũ vẫn phục vụ suốt 15–35 phút build này. Downtime chỉ xảy ra ở Task 6.

**Files:**
- Create (do build sinh ra, đều đã gitignored): `photoprism` (binary), `node_modules/`, `frontend/node_modules/`, `assets/models/*`, `assets/static/build/*`
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: image + volume từ Task 3.
- Produces: binary `/Volumes/BKM/git/photoprism/photoprism` chạy được, frontend build output trong `assets/static/build/`. Task 6 chạy binary này.

- [ ] **Step 1: Xác nhận điểm khởi đầu sạch**

```bash
cd /Volumes/BKM/git/photoprism
git status --short
ls photoprism 2>&1
```

Kỳ vọng: `git status --short` rỗng; `ls photoprism` báo `No such file or directory` (chưa build lần nào).

- [ ] **Step 2: Tải model TensorFlow/ONNX và npm deps**

```bash
cd /Volumes/BKM/git/photoprism
docker run --rm \
  -v /Volumes/BKM/git/photoprism:/go/src/github.com/photoprism/photoprism \
  -v photoprism-gocache:/go/cache \
  -w /go/src/github.com/photoprism/photoprism \
  -e GOCACHE=/go/cache \
  -e GIT_CONFIG_COUNT=1 \
  -e GIT_CONFIG_KEY_0=safe.directory \
  -e GIT_CONFIG_VALUE_0=/go/src/github.com/photoprism/photoprism \
  --entrypoint /bin/bash \
  photoprism/develop:resolute -c 'make dep'
```

`make dep` = `dep-tensorflow dep-onnx dep-js`: tải facenet + nasnet + nsfw + scrfd vào `assets/models/`, rồi `npm ci`. Khoảng 5–15 phút.

- [ ] **Step 3: Kiểm chứng model và npm deps đã có**

```bash
cd /Volumes/BKM/git/photoprism
ls assets/models/
du -sh assets/models/
ls -d node_modules >/dev/null 2>&1 && echo "node_modules OK"
python3 -c "import json; print('workspaces:', json.load(open('package.json')).get('workspaces'))"
```

Kỳ vọng: `assets/models/` có đúng 4 thư mục `facenet nasnet nsfw scrfd`, tổng khoảng 229 MB, và `node_modules OK`.

**Không** kỳ vọng `frontend/node_modules` tồn tại: `package.json` khai `workspaces: ['frontend']` nên npm ci gom toàn bộ dependency về `node_modules/` ở root. Bằng chứng frontend deps đủ dùng là `make build-js` chạy được ở Step 4, không phải sự tồn tại của thư mục đó.

- [ ] **Step 4: Build frontend và Go binary**

```bash
cd /Volumes/BKM/git/photoprism
docker run --rm \
  -v /Volumes/BKM/git/photoprism:/go/src/github.com/photoprism/photoprism \
  -v photoprism-gocache:/go/cache \
  -w /go/src/github.com/photoprism/photoprism \
  -e GOCACHE=/go/cache \
  -e GIT_CONFIG_COUNT=1 \
  -e GIT_CONFIG_KEY_0=safe.directory \
  -e GIT_CONFIG_VALUE_0=/go/src/github.com/photoprism/photoprism \
  --entrypoint /bin/bash \
  photoprism/develop:resolute -c 'make build-js build-go'
```

10–25 phút (lần đầu compile toàn bộ dependency Go).

- [ ] **Step 5: Kiểm chứng binary chạy được**

```bash
cd /Volumes/BKM/git/photoprism
ls -lh photoprism
docker run --rm \
  -v /Volumes/BKM/git/photoprism:/go/src/github.com/photoprism/photoprism \
  -w /go/src/github.com/photoprism/photoprism \
  --entrypoint /bin/bash \
  photoprism/develop:resolute -c './photoprism --version && ./photoprism edition'
```

Kỳ vọng:
- binary tồn tại, kích thước vài chục MB
- version dạng `<YYMMDD>-<sha ngắn>-Linux-ARM64-DEVELOP`, ví dụ `260730-446600a33-Linux-ARM64-DEVELOP`. Hậu tố `-DEVELOP` là đúng (`make build-go` → `scripts/build.sh develop`). Version **không** chứa tên tag vì `260728-bbde8f452` là lightweight tag.
- `edition` in ra `ce`

- [ ] **Step 6: Kiểm chứng frontend build output**

```bash
cd /Volumes/BKM/git/photoprism
ls assets/static/build/ | head
du -sh assets/static/build/
```

Kỳ vọng: có các file `.js` / `.css` đã bundle, dung lượng vài MB.

- [ ] **Step 7: Kiểm chứng build không làm bẩn git**

```bash
cd /Volumes/BKM/git/photoprism
git status --short
```

Kỳ vọng: **rỗng**. Binary `photoprism`, `node_modules/`, `assets/models/`, `assets/static/build/` đều đã nằm trong `.gitignore`. Nếu có file lạ hiện ra → xem lại, đừng commit output build.

- [ ] **Step 8: Kiểm chứng instance cũ vẫn chạy**

```bash
curl -s http://localhost:8098/api/v1/status
docker ps --filter name=photoprism-photoprism-1 --format '{{.Image}} {{.Status}}'
```

Kỳ vọng: `{"status":"operational"}`, image vẫn `photoprism/photoprism:latest`. Task này không được gây downtime.

- [ ] **Step 9: Cập nhật log và commit**

Thêm vào `## Tiến trình` (thay version bằng version thật ở Step 5):

```markdown
- [x] Task 5 — build xong trong container tạm, binary `<version thật>`, edition `ce`. Chưa downtime.
```

```bash
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Build app from source in throwaway container"
git push
```

---

### Task 6: Chuyển sang chế độ dev, chạy migration, kiểm chứng dữ liệu

Đây là task duy nhất có downtime (vài phút, cộng thời gian migration).

**Files:**
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: binary từ Task 5, `dev.sh` + `compose.dev.yaml` từ Task 4, 3 số baseline từ Task 1.
- Produces: DB đã migrate lên schema của bản build từ source; app chạy được từ source trên port 8098.

- [ ] **Step 1: Ghi lại số liệu ngay trước khi dừng**

```bash
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL);"
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT COUNT(*) FROM photoprism.migrations;"
```

Ghi lại 3 số này (có thể lớn hơn baseline Task 1 do auto-index) và số lượng migration đã chạy. Đây là mốc so sánh chính xác nhất.

- [ ] **Step 2: Bật chế độ dev**

```bash
cd /Volumes/BKM/photoprism
./dev.sh up
docker ps --filter name=photoprism --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
```

Kỳ vọng: `photoprism-photoprism-1` giờ chạy image `photoprism/develop:resolute`, `photoprism-mariadb-1` vẫn `mariadb:11` và **không bị recreate** (Status vẫn là uptime cũ, không phải "Up 5 seconds").
Nếu mariadb bị recreate: không mất data (volume `photoprism_database` giữ nguyên) nhưng hãy kiểm chứng ở Step 6 kỹ hơn.

- [ ] **Step 3: Kiểm chứng container dev thấy đúng source và data**

```bash
cd /Volumes/BKM/photoprism
./dev.sh version
docker compose -f compose.yaml -f compose.dev.yaml exec photoprism bash -c \
  'ls /photoprism/originals | head -3; ls /photoprism/storage; ls assets/static/build | head -3'
```

Kỳ vọng: version dạng `<YYMMDD>-<sha>-Linux-ARM64-DEVELOP`; thấy thư mục ảnh gốc (2013, 2017, 2020, 2026...); thấy `backup cache config sidecar users` trong storage; thấy file build của frontend.

- [ ] **Step 4: Chạy migration tường minh trước khi start app**

```bash
cd /Volumes/BKM/photoprism
docker compose -f compose.yaml -f compose.dev.yaml exec photoprism ./photoprism migrate 2>&1 | tail -30
```

Chạy migrate riêng (thay vì để `start` tự làm) để log migration không lẫn với log khởi động.
Kỳ vọng: kết thúc không có dòng `level=error`, `level=fatal`, hay `panic`.
Nếu có error: **DỪNG**, không start app, báo người dùng kèm log, và dùng đường lùi trong `custom/notes/2026-07-30-migration-log.md`.

- [ ] **Step 5: Kiểm chứng migration đã thêm bản ghi và không mất dữ liệu**

```bash
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.migrations);"
```

Kỳ vọng: 3 số đầu **bằng đúng** số ghi ở Step 1. Số migration **≥** số ở Step 1.
Nếu số photos/files/albums giảm → **DỪNG**, restore theo đường lùi, báo người dùng.

- [ ] **Step 6: Chạy app từ source**

Mở một terminal riêng (lệnh này chạy foreground):

```bash
cd /Volumes/BKM/photoprism
./dev.sh run
```

Kỳ vọng: log khởi động, kết thúc bằng dòng cho biết server lắng nghe trên `:2342`. Không có `panic` hay `fatal`.

- [ ] **Step 7: Kiểm chứng API và số liệu qua HTTP**

Ở terminal khác:

```bash
curl -s -o /dev/null -w 'HTTP %{http_code} in %{time_total}s\n' http://localhost:8098/api/v1/status
curl -s http://localhost:8098/api/v1/status
```

Kỳ vọng: `HTTP 200`, body `{"status":"operational"}`.

- [ ] **Step 7b: Kiểm chứng thumbnail trả JPEG thật, không phải icon lỗi**

`internal/api/thumbnails.go:168` trả về **HTTP 200 kèm SVG icon lỗi** khi không resolve được file — nên chỉ xem status code sẽ không phát hiện được lỗi. Phải kiểm `content_type`:

```bash
TOK=$(docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT preview_token FROM photoprism.auth_sessions WHERE preview_token<>'' ORDER BY created_at DESC LIMIT 1;")
H=$(docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT file_hash FROM photoprism.files WHERE file_name LIKE '%.mov.jpg' LIMIT 1;")
curl -s -o /dev/null -w 'HTTP %{http_code} content_type: %{content_type}\n' "http://localhost:8098/api/v1/t/$H/$TOK/tile_500"
```

**Phải in cả `http_code`, không chỉ `content_type`.** Preview token hết hiệu lực thì API trả **403 kèm SVG** — nhìn `content_type: image/svg+xml` một mình sẽ tưởng là lỗi thumbnail. Token trong `auth_sessions` biến mất khi session của người dùng hết hạn; lấy token mới bằng cách đăng nhập lại trên UI rồi query lại.

Kỳ vọng: `HTTP 200` **và** `image/jpeg`.
- `HTTP 403` → token hết hiệu lực, không liên quan thumbnail. Đăng nhập lại rồi lấy token mới.
- `HTTP 200` + `image/svg+xml` → đường dẫn sidecar sai thật, kiểm lại `PHOTOPRISM_STORAGE_PATH`.

Kiểm chứng không cần token (dùng khi không có session): xác nhận container thấy đúng file cache và sidecar:

```bash
docker exec photoprism-photoprism-1 photoprism config | grep -E 'storage-path|sidecar-path|thumb-cache-path'
docker exec photoprism-photoprism-1 ls -l /photoprism/storage/sidecar/2026/<tên file>.mov.jpg
```

Chọn file `.mov.jpg` vì preview video nằm trong sidecar — đúng chỗ vỡ khi storage path sai. Ảnh JPEG thường nằm trong originals nên vẫn render dù cấu hình sai, không phát hiện được gì.

- [ ] **Step 8: Kiểm chứng UI thật trên browser**

Mở `http://localhost:8098` trong browser, đăng nhập bằng tài khoản admin của bạn, rồi xác nhận **bằng mắt**:

Lưu ý: `PHOTOPRISM_ADMIN_PASSWORD` trong compose chỉ áp dụng ở lần setup đầu tiên. Nếu password đã đổi sau đó thì giá trị trong compose không còn đúng.
- Thư viện hiện ảnh, thumbnail render đúng (không phải ô xám)
- Mở một ảnh ở chế độ xem lớn được
- Trang Albums hiện đúng số album như Step 5

Nếu thumbnail trắng/xám hàng loạt: `PHOTOPRISM_ASSETS_PATH` hoặc storage cache mount sai — kiểm lại Step 3 trước khi đi tiếp.

- [ ] **Step 9: Cập nhật log và commit**

Thêm vào `## Tiến trình` (điền số thật):

```markdown
- [x] Task 6 — chạy từ source OK. Sau migration: <photos>/<files>/<albums>, khớp với trước khi chuyển. UI + thumbnail đã kiểm bằng mắt.
```

```bash
cd /Volumes/BKM/git/photoprism
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Switch running instance to source build, migrate database"
git push
```

---

### Task 7: Kiểm chứng vòng lặp customize hoạt động

Chứng minh bằng một thay đổi code thật rằng vòng lặp sửa → build → thấy kết quả chạy được. Thay đổi Go được giữ lại làm dấu hiệu nhận biết đang chạy bản build riêng.

**Files:**
- Modify: `internal/commands/start.go` (thêm một dòng log)
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: chế độ dev đang chạy từ Task 6.
- Produces: xác nhận vòng lặp Go và vòng lặp frontend đều hoạt động; thời gian thực đo của `make build-go`.

- [ ] **Step 1: Xác nhận anchor còn đúng**

```bash
cd /Volumes/BKM/git/photoprism
grep -c 'Initialize the index database' internal/commands/start.go
grep -n 'Initialize the index database' -A 1 internal/commands/start.go
```

Kỳ vọng: đếm được **đúng 1**, và hai dòng đó là:

```go
	// Initialize the index database.
	conf.InitDb()
```

Chọn chỗ này vì nó chạy vô điều kiện mỗi lần start (không nằm trong nhánh `if`), và biến `conf` đã có trong scope của `startAction`.

- [ ] **Step 2: Thêm dòng log đánh dấu bản build riêng**

Trong `internal/commands/start.go`, thay:

```go
	// Initialize the index database.
	conf.InitDb()
```

bằng:

```go
	log.Infof("custom: running build from local source (%s)", conf.Version())

	// Initialize the index database.
	conf.InitDb()
```

`log` đã có sẵn trong package `commands` (dùng ở dòng 86, 106, 118 của cùng file), `conf.Version()` là method có thật trong `internal/config` — không cần thêm import.

Kiểm chứng sửa đúng chỗ:

```bash
grep -n 'custom: running build' -B 1 -A 3 internal/commands/start.go
```

Kỳ vọng: thấy dòng log mới nằm ngay trên comment `// Initialize the index database.`

- [ ] **Step 3: Đo thời gian build và build lại**

```bash
cd /Volumes/BKM/photoprism
time ./dev.sh build-go
```

Kỳ vọng: build thành công, in kích thước binary. Ghi lại thời gian thực (`real`) — đây là số liệu thật cho vòng lặp, so với ước tính 1–3 phút trong spec.

- [ ] **Step 4: Restart app và kiểm chứng log mới xuất hiện**

```bash
cd /Volumes/BKM/photoprism
./dev.sh stop-app     # BẮT BUỘC trước khi run lại
./dev.sh run
```

Phải `stop-app` vì Ctrl-C (hoặc kill) ở host chỉ giết client `docker compose exec`; tiến trình app **trong container** vẫn giữ cổng 2342, nên lần `run` sau sẽ thất bại với `server: listen tcp 0.0.0.0:2342: bind: address already in use` rồi tự shutdown — và cổng vẫn do binary **cũ** phục vụ, rất dễ tưởng là đã restart thành công.

Kiểm tiến trình khi cần: `docker exec photoprism-photoprism-1 bash -c 'ps -eo pid,args | grep [p]hotoprism'` (đừng dùng `grep -c`, nó đếm cả command line của chính lệnh đang chạy).

Kỳ vọng: trong log khởi động có dòng `custom: running build from local source (<version>)`.
Đây là bằng chứng thay đổi Go đã vào binary đang chạy.

- [ ] **Step 5: Kiểm chứng vòng lặp frontend**

**Không** dùng mtime của file build để kiểm chứng: webpack mặc định `output.compareBeforeEmit: true`, nội dung không đổi thì nó **không ghi lại file**, mtime giữ nguyên dù build đã chạy. `ls --time-style` cũng không tồn tại trên macOS.

Cách đúng — sửa thật một chuỗi, build, grep trong bundle, rồi revert:

```bash
cd /Volumes/BKM/git/photoprism
# đổi tạm chuỗi log trong nhánh debug (không ảnh hưởng người dùng)
sed -i '' 's/console.log("config: new settings"/console.log("config: new settings CUSTOMBUILDTEST"/' frontend/src/common/config.js
grep -n 'CUSTOMBUILDTEST' frontend/src/common/config.js

cd /Volumes/BKM/photoprism && ./dev.sh build-js
grep -l 'CUSTOMBUILDTEST' /Volumes/BKM/git/photoprism/assets/static/build/*.js | xargs -n1 basename
```

Kỳ vọng: ít nhất `app.<hash>.js` và `share.<hash>.js` chứa marker, và **tên file có hash mới** so với trước (content hash đổi theo nội dung).

Revert và build lại:

```bash
cd /Volumes/BKM/git/photoprism
sed -i '' 's/console.log("config: new settings CUSTOMBUILDTEST"/console.log("config: new settings"/' frontend/src/common/config.js
cd /Volumes/BKM/photoprism && ./dev.sh build-js
grep -l 'CUSTOMBUILDTEST' /Volumes/BKM/git/photoprism/assets/static/build/*.js | wc -l
```

Kỳ vọng: `0` — không còn marker. `git status --short` chỉ còn `internal/commands/start.go`.

(`./dev.sh watch-js` là bản chạy liên tục của cùng cơ chế; không cần test riêng.)

- [ ] **Step 6: Kiểm chứng app vẫn phục vụ bình thường sau khi sửa code**

```bash
curl -s http://localhost:8098/api/v1/status
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL;"
```

Kỳ vọng: `{"status":"operational"}` và số photos vẫn khớp Task 6 Step 5.

- [ ] **Step 7: Commit thay đổi code đầu tiên**

```bash
cd /Volumes/BKM/git/photoprism
git add internal/commands/start.go
git commit -m "Custom: Log a marker at startup when running a local source build"
git push
```

- [ ] **Step 8: Cập nhật log và commit**

Thêm vào `## Tiến trình` (điền thời gian thật đo được ở Step 3):

```markdown
- [x] Task 7 — vòng lặp Go OK (`make build-go` mất <thời gian thật>), vòng lặp frontend OK. Commit code sửa đầu tiên.
```

```bash
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Record measured dev loop timings"
git push
```

---

### Task 8: Bake image `:local` và chuyển sang chế độ image

**Files:**
- Modify: `/Volumes/BKM/photoprism/compose.yaml` (dòng `image:`)
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: source đã có thay đổi từ Task 7; `dev.sh bake` và `dev.sh prod` từ Task 4.
- Produces: image `photoprism/photoprism:local` chạy được; compose.yaml mặc định dùng image đó.

- [ ] **Step 1: Kiểm tra disk trước khi build**

```bash
df -h / | tail -1
```

Yêu cầu: `Avail` **≥ 10Gi**. Nếu thiếu: **DỪNG**, báo người dùng con số thực tế, không tự dọn.

- [ ] **Step 2: Bake image**

```bash
cd /Volumes/BKM/photoprism
time ./dev.sh bake
```

Chạy `make docker-local` → `docker-local-resolute`: pull `photoprism/develop:resolute` + `ubuntu:resolute`, rồi build `docker/photoprism/resolute/Dockerfile` với `--no-cache`. 25–50 phút. Ghi lại thời gian thực.

Build này sinh 3 tag: `photoprism/photoprism:ce-resolute`, `photoprism/photoprism:<YYMMDD>-ce-resolute`, và `photoprism/photoprism:local`.

- [ ] **Step 3: Kiểm chứng image vừa build**

```bash
docker image ls photoprism/photoprism --format '{{.Repository}}:{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}'
docker run --rm --entrypoint /opt/photoprism/bin/photoprism photoprism/photoprism:local --version
docker run --rm --entrypoint /opt/photoprism/bin/photoprism photoprism/photoprism:local edition
```

Đường dẫn binary `/opt/photoprism/bin/photoprism` lấy từ Makefile target `install`: `./scripts/build.sh prod "$(DESTDIR)/bin/$(BINARY_NAME)"` với `DESTDIR=/opt/photoprism` và `BINARY_NAME=photoprism`.

Kỳ vọng: có tag `local` vừa tạo; version dạng `<YYMMDD>-<sha>-Linux-ARM64` (**không** có hậu tố `-DEVELOP` — target `install` build bằng `scripts/build.sh prod`, không phải `develop`); edition `ce`.

- [ ] **Step 4: Trỏ compose.yaml sang image local**

Trong `/Volumes/BKM/photoprism/compose.yaml`, đổi:

```yaml
    image: photoprism/photoprism:latest
```

thành:

```yaml
    ## Image build từ source local bằng `./dev.sh bake` (make docker-local)
    image: photoprism/photoprism:local
```

- [ ] **Step 5: Kiểm chứng compose hợp lệ**

```bash
cd /Volumes/BKM/photoprism
docker compose config --quiet && echo "COMPOSE OK"
docker compose config | grep 'image:.*photoprism'
```

Kỳ vọng: `COMPOSE OK`, và image của service photoprism là `photoprism/photoprism:local`.

- [ ] **Step 6: Chuyển sang chế độ image**

Ở terminal đang chạy `./dev.sh run`: Ctrl-C để dừng app trước.

```bash
cd /Volumes/BKM/photoprism
./dev.sh prod
docker ps --filter name=photoprism --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
```

Kỳ vọng: `photoprism-photoprism-1` chạy image `photoprism/photoprism:local`, status `Up`.

- [ ] **Step 7: Kiểm chứng đầy đủ sau khi chuyển**

```bash
sleep 30
docker exec photoprism-photoprism-1 photoprism --version
docker exec photoprism-photoprism-1 photoprism edition
curl -s http://localhost:8098/api/v1/status
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL);"
docker logs --tail 40 photoprism-photoprism-1 2>&1 | grep -iE 'custom:|error|fatal|panic' | head -20
```

Kỳ vọng:
- version dạng `<YYMMDD>-<sha>-Linux-ARM64`, edition `ce`
- API `{"status":"operational"}`
- 3 số **khớp** Task 6 Step 5
- log có dòng `custom: running build from local source (...)` → xác nhận thay đổi Task 7 đã vào image
- không có `fatal` / `panic`

- [ ] **Step 8: Kiểm chứng UI lần cuối**

Mở `http://localhost:8098`, đăng nhập, xác nhận bằng mắt: thư viện hiện ảnh, thumbnail render đúng, mở được một ảnh, số album khớp.

- [ ] **Step 9: Cập nhật log và commit**

Thêm vào `## Tiến trình` (điền số thật):

```markdown
- [x] Task 8 — bake `photoprism/photoprism:local` (<thời gian thật>), compose trỏ sang image đó, chạy OK. Version `<version>`, edition `ce`. Data khớp.
```

```bash
cd /Volumes/BKM/git/photoprism
git add custom/notes/2026-07-30-migration-log.md
git commit -m "Custom: Bake local image and switch deployment to it"
git push
```

---

### Task 9: Runbook update upstream và cập nhật CLAUDE.md

**Files:**
- Create: `custom/notes/upstream-update-runbook.md`
- Modify: `/Volumes/BKM/photoprism/CLAUDE.md`
- Modify: `custom/notes/2026-07-30-migration-log.md`

**Interfaces:**
- Consumes: tên remote `origin`/`upstream` từ Task 2; các subcommand `dev.sh` từ Task 4.
- Produces: tài liệu để lần sau mở session là có ngay context, không phải suy luận lại.

- [ ] **Step 1: Viết runbook update upstream**

Tạo `custom/notes/upstream-update-runbook.md`:

```markdown
# Runbook: cập nhật code mới nhất từ upstream

Repo: `/Volumes/BKM/git/photoprism`, branch `custom`.
Remote: `origin` = fork cá nhân, `upstream` = photoprism/photoprism.

## Trước khi update

```bash
cd /Volumes/BKM/photoprism
docker exec photoprism-mariadb-1 mariadb-dump -u root -pinsecure \
  --single-transaction --quick --routines --events photoprism \
  | gzip > /Volumes/BKM/photoprism/backup-$(date +%Y%m%d).sql.gz
gunzip -t /Volumes/BKM/photoprism/backup-$(date +%Y%m%d).sql.gz && echo "GZIP OK"

docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL);"
```

Ghi lại 3 số đó — sau khi update phải khớp.

## Update

```bash
cd /Volumes/BKM/git/photoprism
git fetch upstream --tags
git tag --sort=-creatordate | head -5          # chọn tag release mới nhất
git rebase <tag-mới> custom
```

Rebase lên **tag release**, không lên `develop` HEAD: tag đã qua QA upstream, migration ổn định hơn.

Nếu conflict: chỉ xảy ra ở file đã sửa. `git status` để xem, sửa xong `git add` rồi `git rebase --continue`. Muốn huỷ: `git rebase --abort`.

## Build lại và chạy

```bash
cd /Volumes/BKM/photoprism
./dev.sh bake                    # 25-50 phút
./dev.sh prod
sleep 30
docker exec photoprism-photoprism-1 photoprism --version
curl -s http://localhost:8098/api/v1/status
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL);"
docker logs --tail 50 photoprism-photoprism-1 2>&1 | grep -iE 'error|fatal|panic' | head
```

3 số phải khớp số ghi trước khi update. Nếu không khớp hoặc có `fatal`/`panic`: restore dump vừa tạo, đổi `image:` về tag image cũ đang còn local (`docker image ls photoprism/photoprism`).

## Push code riêng lên fork

```bash
cd /Volumes/BKM/git/photoprism
git push --force-with-lease origin custom
```

Cần `--force-with-lease` vì rebase viết lại history. `--force-with-lease` (không phải `--force`) để không ghi đè mất commit nếu fork có thay đổi lạ.

## Giữ code sửa dễ rebase

- Ưu tiên thêm **file mới** trong `custom/` hơn là sửa file upstream.
- Khi buộc phải sửa file upstream, sửa càng ít dòng càng tốt và gom vào một commit riêng có prefix `Custom:`.
- Xem toàn bộ code riêng đang có: `git log --oneline <tag-đang-dùng>..custom`
```

- [ ] **Step 2: Kiểm chứng runbook không có lệnh sai cú pháp**

```bash
cd /Volumes/BKM/git/photoprism
grep -c 'docker exec photoprism-mariadb-1' custom/notes/upstream-update-runbook.md
git log --oneline 260728-bbde8f452..custom | head -10
```

Kỳ vọng: grep ra **≥ 3**; `git log` liệt kê các commit `Custom:` đã tạo (chứng minh lệnh cuối trong runbook chạy được thật).

- [ ] **Step 3: Viết lại CLAUDE.md ở deployment dir**

Ghi `/Volumes/BKM/photoprism/CLAUDE.md`:

```markdown
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Đây là thư mục **deployment** của PhotoPrism, không phải source code. App được build từ
source ở `/Volumes/BKM/git/photoprism` (branch `custom`, fork của `photoprism/photoprism`),
**không** dùng image pull từ Docker Hub nữa.

- **Web UI**: http://localhost:8098 (map từ container port 2342)
- **Originals**: `/Volumes/BKM/memories` → `/photoprism/originals`
- **Storage** (cache 134 GB, sidecar, config): `./storage` → `/photoprism/storage`
- **Database**: volume `photoprism_database` (MariaDB 11)
- **Source**: `/Volumes/BKM/git/photoprism`, branch `custom`
- **Spec + runbook**: `/Volumes/BKM/git/photoprism/custom/`

## Hai chế độ chạy

| Chế độ | Chạy bằng | Khi nào dùng |
| --- | --- | --- |
| image | `photoprism/photoprism:local` bake từ source | chạy thường ngày |
| dev | `photoprism/develop:resolute` + source mount | khi sửa code |

Cả hai dùng **cùng** originals, storage, và database. Mọi lệnh qua `./dev.sh`:

```bash
./dev.sh status      # container nào đang chạy, API có sống không
./dev.sh up          # bật chế độ dev (app CHƯA chạy)
./dev.sh run         # chạy app từ source (foreground)
./dev.sh build-go    # compile lại Go sau khi sửa (~1-3 phút)
./dev.sh watch-js    # build frontend liên tục khi sửa Vue
./dev.sh shell       # vào bash trong container dev
./dev.sh bake        # build photoprism/photoprism:local (25-50 phút)
./dev.sh prod        # tắt dev, chạy lại bằng image :local
./dev.sh down        # tắt stack
```

Vòng lặp sửa Go: `Ctrl-C` app → `./dev.sh build-go` → `./dev.sh run`
Vòng lặp sửa Vue: `./dev.sh watch-js` ở tab khác, rồi reload browser

## Cập nhật code mới từ upstream

Theo `/Volumes/BKM/git/photoprism/custom/notes/upstream-update-runbook.md`.
Tóm lại: `git fetch upstream --tags` → `git rebase <tag-mới> custom` → `./dev.sh bake` → `./dev.sh prod`.
Rebase lên **tag release**, không lên `develop` HEAD.

## PhotoPrism CLI

Chế độ image:

```bash
docker compose exec photoprism photoprism index
docker compose exec photoprism photoprism index --force
docker compose exec photoprism photoprism purge
docker compose exec photoprism photoprism backup --albums --force
docker compose exec photoprism photoprism config
```

Chế độ dev: thay `docker compose exec photoprism photoprism` bằng
`docker compose -f compose.yaml -f compose.dev.yaml exec photoprism ./photoprism`.

## MariaDB

```bash
docker compose exec mariadb mariadb -u photoprism -pinsecure photoprism
```

Backup DB:

```bash
docker exec photoprism-mariadb-1 mariadb-dump -u root -pinsecure \
  --single-transaction --quick --routines --events photoprism \
  | gzip > backup-$(date +%Y%m%d).sql.gz
```

## Cấu hình

- `compose.yaml` — chế độ image. Đổi setting runtime ở đây.
- `compose.dev.yaml` — override chế độ dev. Ghi đè `PHOTOPRISM_INIT=""` (develop image
  đã có TensorFlow) và `PHOTOPRISM_INDEX_SCHEDULE=""` (tránh auto-index 134k file khi
  đang thử nghiệm).
- `storage/config/settings.yml` — UI preferences và feature flags.

Project name compose **phải** là `photoprism` để giữ volume `photoprism_database`. Luôn
chạy `docker compose` từ thư mục này.

## Ràng buộc cần biết

- **Không** chạy `docker system prune` hay lệnh dọn disk — người dùng tự quản lý.
  Disk nội bộ mỏng (~20 GB trống lúc setup), BKM còn ~30 GB.
- **Không** copy/move `storage/` (134 GB, BKM không đủ chỗ).
- **Không** chạy `make start` từ repo source: đó là dev env của upstream, nó tạo storage
  riêng ở gốc repo và gây nhầm lẫn.
- `./dev.sh bake` chạy `docker build --no-cache` → mỗi lần build lại toàn bộ Go + npm.
  Sửa code thì dùng chế độ dev, chỉ bake khi thay đổi đã ổn.

## Vấn đề tồn đọng

- ~46 video không tạo được preview image (`.mov`, `.mp4`, phần lớn thuộc `2020/`)
- 62 file JPG bị skip khi index (`could not be identified`)
- 1 HEIC không convert được (`2026/20260618183700.heic`)
- 1 JPG corrupt gây vips panic-stack (`2017/`)
- Mật khẩu admin và DB vẫn là mặc định `insecure`
```

- [ ] **Step 4: Kiểm chứng CLAUDE.md khớp thực tế**

```bash
cd /Volumes/BKM/photoprism
grep -oE './dev.sh [a-z-]+' CLAUDE.md | sed 's|\./dev\.sh ||' | sort -u > /tmp/claude-cmds.txt
grep -oE '^  [a-z-]+\)' dev.sh | tr -d ' )' | sort -u > /tmp/devsh-cmds.txt
comm -23 /tmp/claude-cmds.txt /tmp/devsh-cmds.txt
wc -l < /tmp/claude-cmds.txt
```

`grep -oE ... [a-z-]+` (dấu `+`, không phải `*`) để câu văn xuôi có `./dev.sh` mà không kèm subcommand không sinh ra dòng rỗng làm sai kết quả `comm`.

Kỳ vọng: `comm` in ra **rỗng** — mọi subcommand nhắc trong CLAUDE.md đều tồn tại thật trong `dev.sh`; và `wc -l` **≥ 8** (chứng minh grep có bắt được lệnh thật, không phải rỗng nên mới "pass"). Nếu `comm` in ra dòng nào, đó là lệnh CLAUDE.md bịa ra; sửa lại.

- [ ] **Step 5: Kiểm chứng lần cuối toàn hệ thống**

```bash
cd /Volumes/BKM/photoprism
./dev.sh status
docker exec photoprism-photoprism-1 photoprism --version
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL);"
ls -lh /Volumes/BKM/photoprism/backup-pre-source-migration.sql.gz
df -h / | tail -1
```

Kỳ vọng: API HTTP 200, version là bản build từ source, 3 số khớp Task 6, dump backup còn nguyên, và ghi lại disk còn lại để báo người dùng.

- [ ] **Step 6: Đóng log và commit**

Thêm vào `## Tiến trình`:

```markdown
- [x] Task 9 — runbook update upstream + CLAUDE.md deployment đã cập nhật. Hoàn tất.
```

```bash
cd /Volumes/BKM/git/photoprism
git add custom/notes/
git commit -m "Custom: Add upstream update runbook and refresh deployment docs"
git push
```

---

## Ghi chú cho người thực thi

**Thứ tự downtime:** Task 1–5 không gây downtime (instance cũ vẫn phục vụ). Downtime bắt đầu ở Task 6 Step 2 và kéo dài đến hết Task 8. Nếu cần instance sống liên tục thì dừng sau Task 5 và làm tiếp Task 6–8 vào lúc thuận tiện.

**Điểm dừng bắt buộc — báo người dùng, không tự xử lý:**
- Disk không đủ ở Task 1 Step 1, Task 3 Step 2, hoặc Task 8 Step 1
- Migration có error ở Task 6 Step 4
- Số photos/files/albums giảm ở bất kỳ bước kiểm chứng nào
- `develop:resolute` không phải arm64 ở Task 3 Step 2

**Không tự chạy** `docker system prune`, `docker volume rm`, `docker image prune`, hay bất kỳ lệnh dọn disk nào — kể cả khi build fail vì hết chỗ.

**Ước tính thời gian:** Task 1 ~10 phút, Task 2 ~5 phút, Task 3 ~10–20 phút, Task 4 ~15 phút, Task 5 ~20–40 phút, Task 6 ~15–30 phút, Task 7 ~10 phút, Task 8 ~30–60 phút, Task 9 ~15 phút. Tổng khoảng 2–4 giờ, phần lớn là chờ build.
