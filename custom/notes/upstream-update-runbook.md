# Runbook: cập nhật code mới nhất từ upstream

Repo: `/Volumes/BKM/git/photoprism`, branch `custom`.
Remote: `origin` = fork cá nhân `chaien0609/photoprism`, `upstream` = `photoprism/photoprism`.
Deployment: `/Volumes/BKM/photoprism` (compose + `dev.sh`, không phải git repo).

## 0. Trước khi update — backup và ghi mốc

```bash
cd /Volumes/BKM/photoprism
docker exec photoprism-mariadb-1 mariadb-dump -u root -pinsecure \
  --single-transaction --quick --routines --events photoprism \
  | gzip > /Volumes/BKM/photoprism/backup-$(date +%Y%m%d).sql.gz
gunzip -t /Volumes/BKM/photoprism/backup-$(date +%Y%m%d).sql.gz && echo "GZIP OK"

docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.migrations);"
```

Ghi lại 4 số đó. Sau khi update, 3 số đầu **phải khớp**; số migration có thể tăng.

Kiểm disk: `df -h /` cần **≥ 10Gi** trống cho lần bake. Nếu thiếu thì dọn trước, đừng bắt đầu.

## 1. Rebase lên tag release mới

```bash
cd /Volumes/BKM/git/photoprism
git fetch upstream --tags
git tag --sort=-creatordate | head -5
git rebase <tag-mới> custom
```

Rebase lên **tag release**, không lên `develop` HEAD: tag đã qua QA upstream, migration ổn định hơn.

Conflict chỉ xảy ra ở file đã sửa. `git status` để xem, sửa xong `git add` rồi `git rebase --continue`. Muốn huỷ: `git rebase --abort`.

Xem toàn bộ code riêng đang có: `git log --oneline <tag-đang-dùng>..custom`

## 2. So ENV của image mới với cấu hình đang dùng

Upstream **có** thay đổi ENV giữa các bản. Ví dụ thật: bản 260728 (resolute) đã bỏ `THUMB_SIZE`, `THUMB_SIZE_UNCACHED`, `JPEG_SIZE`, `PNG_SIZE` khỏi ENV và đổi `THUMB_UNCACHED` từ `true` sang `false` so với bản 260601. Nếu không để ý, hành vi thumbnail đổi âm thầm.

```bash
docker image inspect photoprism/photoprism:local \
  --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^PHOTOPRISM_' | sort > /tmp/env-old.txt
# sau khi bake image mới, chạy lại lệnh trên vào /tmp/env-new.txt rồi:
diff /tmp/env-old.txt /tmp/env-new.txt
```

Biến nào bị bỏ mà mình đang dựa vào thì **ghim tường minh vào `compose.yaml`**. Hiện `compose.yaml` đã ghim 5 biến thumbnail vì lý do này.

Muốn đọc ENV thật của một image thì **bypass entrypoint** — entrypoint `/init` xoá sạch biến `PHOTOPRISM_*` trước khi CMD chạy:

```bash
docker run --rm --entrypoint /bin/bash <image> -c 'echo $PHOTOPRISM_STORAGE_PATH'
```

## 3. Bake và chạy

```bash
cd /Volumes/BKM/photoprism
./dev.sh bake        # ~15 phút (đo thực tế), docker build --no-cache
./dev.sh prod
```

## 4. Kiểm chứng

```bash
cd /Volumes/BKM/photoprism
./dev.sh status
docker exec photoprism-photoprism-1 photoprism --version
docker exec photoprism-photoprism-1 photoprism edition
docker exec photoprism-photoprism-1 photoprism config | grep -E 'storage-path|sidecar-path|thumb-cache-path'
docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT (SELECT COUNT(*) FROM photoprism.photos WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.files WHERE deleted_at IS NULL), \
          (SELECT COUNT(*) FROM photoprism.albums WHERE deleted_at IS NULL);"
docker logs photoprism-photoprism-1 2>&1 | grep -iE 'fatal|panic' | head
docker logs photoprism-photoprism-1 2>&1 | grep 'custom: running build'
```

Yêu cầu:
- 3 số khớp mốc ở bước 0
- `storage-path` = `/photoprism/storage`, `sidecar-path` = `/photoprism/storage/sidecar`, `thumb-cache-path` = `/photoprism/storage/cache/thumbnails` — **không** được là `/photoprism/originals/.photoprism/storage/...`
- không có `fatal` / `panic`
- có dòng `custom: running build from local source (...)` → xác nhận code riêng còn trong build

Kiểm thumbnail qua HTTP (cần đăng nhập UI để có preview token còn hiệu lực):

```bash
TOK=$(docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT preview_token FROM photoprism.auth_sessions WHERE preview_token<>'' ORDER BY created_at DESC LIMIT 1;")
H=$(docker exec photoprism-mariadb-1 mariadb -u root -pinsecure -N -B -e \
  "SELECT file_hash FROM photoprism.files WHERE file_name LIKE '%.mov.jpg' LIMIT 1;")
curl -s -o /dev/null -w 'HTTP %{http_code} %{content_type}\n' "http://localhost:8098/api/v1/t/$H/$TOK/tile_500"
```

Kỳ vọng `HTTP 200` **và** `image/jpeg`. Luôn in cả status code: token hết hiệu lực trả **403 kèm SVG**, rất dễ đọc nhầm thành lỗi thumbnail. Chọn file `.mov.jpg` vì preview video nằm trong sidecar — đúng chỗ vỡ nếu storage path sai; ảnh JPEG thường nằm trong originals nên vẫn hiện dù cấu hình sai.

Cuối cùng mở `http://localhost:8098` xem bằng mắt: thư viện hiện ảnh, thumbnail render, mở được một ảnh.

## 5. Push code riêng lên fork

```bash
cd /Volumes/BKM/git/photoprism
gh auth switch --user chaien0609      # push cần active account là chaien0609
git push --force-with-lease origin custom
gh auth switch --user core-cuong      # trả lại nếu trước đó đang dùng account này
```

Cần `--force-with-lease` vì rebase viết lại history. Dùng `--force-with-lease` (không phải `--force`) để không ghi đè mất commit nếu fork có thay đổi lạ.

Nếu push báo `Permission to chaien0609/photoprism.git denied to core-cuong` thì đúng là do active account sai — switch rồi push lại.

## Nếu vỡ — đường lùi

```bash
cd /Volumes/BKM/photoprism
docker compose down
# đổi image: trong compose.yaml về tag còn nằm local, xem bằng:
docker image ls photoprism/photoprism
# restore DB:
gunzip -c /Volumes/BKM/photoprism/backup-<ngày>.sql.gz | \
  docker exec -i photoprism-mariadb-1 mariadb -u root -pinsecure photoprism
docker compose up -d
```

Lưu ý: migration là **một chiều**. Quay về image cũ mà không restore DB thì schema mới có thể không tương thích.

## Giữ code sửa dễ rebase

- Ưu tiên thêm **file mới** trong `custom/` hơn là sửa file upstream.
- Khi buộc phải sửa file upstream, sửa càng ít dòng càng tốt, gom vào một commit riêng có prefix `Custom:`.
- Trước khi sửa, kiểm anchor còn tồn tại (ví dụ `grep -c '// Initialize the index database.' internal/commands/start.go` phải ra 1) — upstream có thể đã đổi.
