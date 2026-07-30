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
