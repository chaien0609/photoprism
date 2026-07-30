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
