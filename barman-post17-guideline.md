# Barman Backup Configuration Guideline
## PostgreSQL 17 — Hybrid Mode (rsync + Streaming WAL)

---

## 📋 Thông Số Cấu Hình

| Tham số | Giá trị |
|---|---|
| **Barman user** | `barman` |
| **Barman password** | `123` |
| **Barman server IP** | `192.168.56.126` |
| **PostgreSQL server IP** | `192.168.56.124` |
| **PostgreSQL port** | `5432` |
| **PostgreSQL version** | `17` |
| **Barman version** | `3.x` |
| **Backup mode** | rsync (base backup) + streaming (WAL) |
| **Backup directory** | `/data/backup/post-17` |
| **Barman config file** | `post-17.conf` |
| **PostgreSQL instance name** | `post-17` |
| **OS** | Oracle Linux 9 |
| **Retention policy** | ⚠️ *Chưa xác định — xem ghi chú cuối file* |

---

## 1. Cài Đặt Barman (trên Barman Server — 192.168.56.126)

```bash
# Cài đặt EPEL và PostgreSQL repo
sudo dnf install -y epel-release
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Cài Barman và PostgreSQL client 17
sudo dnf install -y barman barman-cli postgresql17
```

> ⚠️ **Ghi chú (nội suy):** Package name `barman` trên Oracle Linux 9 thường đến từ EPEL hoặc PGDG repo. Nếu không tìm thấy, thử: `sudo dnf install -y python3-barman`. Kiểm tra lại bằng `dnf search barman`.

---

## 2. Cấu Hình SSH Key (cho rsync — Base Backup)

Barman dùng SSH để rsync data từ PostgreSQL server. Cần thiết lập SSH key trust 2 chiều.

### 2.1 Tạo SSH key trên Barman Server

```bash
# Chạy với user barman
sudo -u barman ssh-keygen -t rsa -b 4096 -N "" -f /var/lib/barman/.ssh/id_rsa
```

### 2.2 Copy public key sang PostgreSQL Server

```bash
sudo -u barman ssh-copy-id postgres@192.168.56.124
# Nhập password của user postgres trên PostgreSQL server khi được hỏi
```

### 2.3 Tạo SSH key trên PostgreSQL Server (chiều ngược lại)

```bash
# Chạy trên PostgreSQL server (192.168.56.124), với user postgres
sudo -u postgres ssh-keygen -t rsa -b 4096 -N "" -f /var/lib/pgsql/.ssh/id_rsa
sudo -u postgres ssh-copy-id barman@192.168.56.126
```

### 2.4 Kiểm tra SSH

```bash
# Từ Barman server — phải login được mà không cần password
sudo -u barman ssh postgres@192.168.56.124 "echo OK"

# Từ PostgreSQL server — phải login được mà không cần password
sudo -u postgres ssh barman@192.168.56.126 "echo OK"
```

---

## 3. Cấu Hình PostgreSQL (trên PostgreSQL Server — 192.168.56.124)

### 3.1 Chỉnh postgresql.conf

```bash
sudo -u postgres vi /var/lib/pgsql/17/data/postgresql.conf
```

Thêm / chỉnh các tham số sau:

```ini
# WAL configuration
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 512MB

# Archiving (dùng barman-wal-archive để ship WAL qua SSH)
archive_mode = on
archive_command = 'barman-wal-archive 192.168.56.126 post-17 %p'
```

> ⚠️ **Ghi chú:** `barman-wal-archive` là script có trong package `barman-cli`, cần được cài trên **PostgreSQL server** (192.168.56.124), không phải Barman server.

```bash
# Cài barman-cli trên PostgreSQL server
sudo dnf install -y barman-cli
```

### 3.2 Chỉnh pg_hba.conf — Cho phép replication connection

```bash
sudo -u postgres vi /var/lib/pgsql/17/data/pg_hba.conf
```

Thêm dòng sau:

```
# Barman streaming replication
host    replication     barman          192.168.56.126/32       md5

# Barman superuser connection (dùng pg_connect_string trong barman config)
host    all             barman          192.168.56.126/32       md5
```

### 3.3 Tạo user barman trong PostgreSQL

```sql
-- Kết nối vào PostgreSQL với user postgres
sudo -u postgres psql

-- Tạo user barman với quyền superuser và replication
CREATE ROLE barman WITH LOGIN SUPERUSER REPLICATION PASSWORD '123';

-- Kiểm tra
\du barman
```

> ⚠️ **Ghi chú:** SUPERUSER là cách đơn giản nhất để Barman hoạt động đầy đủ. Nếu muốn least-privilege, có thể cấp quyền từng cái (REPLICATION + pg_read_all_settings + pg_monitor...), nhưng cần test kỹ hơn — đây là nội suy từ best practice Barman docs.

### 3.4 Reload PostgreSQL

```bash
sudo systemctl reload postgresql-17
# hoặc
sudo -u postgres /usr/pgsql-17/bin/pg_ctl reload -D /var/lib/pgsql/17/data
```

---

## 4. Cấu Hình Barman (trên Barman Server — 192.168.56.126)

### 4.1 Tạo thư mục backup

```bash
sudo mkdir -p /data/backup/post-17
sudo chown -R barman:barman /data/backup/post-17
```

### 4.2 Kiểm tra / chỉnh file barman.conf chính

```bash
sudo vi /etc/barman.conf
```

Đảm bảo có các dòng cơ bản:

```ini
[barman]
barman_user = barman
configuration_files_directory = /etc/barman.d
barman_home = /var/lib/barman
log_file = /var/log/barman/barman.log
log_level = INFO
```

### 4.3 Tạo file cấu hình cho instance post-17

```bash
sudo vi /etc/barman.d/post-17.conf
```

Nội dung file `post-17.conf`:

```ini
[post-17]
description = "PostgreSQL 17 - post-17"

; ---- Kết nối đến PostgreSQL ----
conninfo = host=192.168.56.124 user=barman password=123 dbname=postgres port=5432
streaming_conninfo = host=192.168.56.124 user=barman password=123 port=5432

; ---- SSH (dùng cho rsync base backup) ----
ssh_command = ssh postgres@192.168.56.124

; ---- Backup method ----
backup_method = rsync
reuse_backup = link

; ---- WAL streaming ----
streaming_archiver = on
slot_name = barman_post17
create_slot = auto

; ---- Archiving (backup qua archive_command từ PostgreSQL) ----
archiver = on

; ---- Thư mục lưu backup ----
basebackups_directory = /data/backup/post-17/base
wals_directory = /data/backup/post-17/wals
errors_directory = /data/backup/post-17/errors
streaming_wals_directory = /data/backup/post-17/streaming

; ---- Retention Policy ----
; ⚠️ Chưa xác định — đặt tạm, cần điều chỉnh theo yêu cầu thực tế
retention_policy = RECOVERY WINDOW OF 7 DAYS
retention_policy_mode = auto
wal_retention_policy = main

; ---- Compression (tuỳ chọn) ----
; compression = gzip
```

> ⚠️ **Ghi chú (nội suy):** Các tham số `basebackups_directory`, `wals_directory`, `errors_directory`, `streaming_wals_directory` là sub-path conventions thông thường. Barman 3.x mặc định sẽ tạo các thư mục này dưới `barman_home/<server_name>/`. Nếu bạn override bằng đường dẫn tuyệt đối như trên, hãy đảm bảo tất cả đều có quyền `barman:barman`.
>
> Nếu muốn đơn giản hơn, chỉ cần set `basebackups_directory` và để Barman tự quản lý phần còn lại theo `barman_home`.

---

## 5. Tạo Replication Slot

```bash
# Chạy trên Barman server
sudo -u barman barman receive-wal --create-slot post-17
```

Hoặc nếu đã set `create_slot = auto` trong config, slot sẽ tự tạo khi chạy `barman switch-wal`.

---

## 6. Kiểm Tra Cấu Hình

### 6.1 Chạy check

```bash
sudo -u barman barman check post-17
```

Tất cả các mục phải là `OK`. Các mục thường gặp cần chú ý:

| Check item | Mô tả |
|---|---|
| `ssh` | SSH từ Barman sang PostgreSQL |
| `PostgreSQL` | Kết nối psql được |
| `replication slot` | Slot tồn tại |
| `streaming` | Streaming WAL hoạt động |
| `archive_mode` | `archive_mode = on` |
| `continuous archiving` | `archive_command` hoạt động |
| `backup maximum age` | Còn trong retention window |

### 6.2 Xem trạng thái

```bash
sudo -u barman barman status post-17
```

### 6.3 Test kết nối streaming

```bash
sudo -u barman barman switch-wal --force --archive post-17
```

---

## 7. Thực Hiện Backup Lần Đầu

```bash
sudo -u barman barman backup post-17
```

Theo dõi tiến trình:

```bash
sudo -u barman barman list-backup post-17
```

---

## 8. Cấu Hình Tự Động — Cron Job

```bash
sudo -u barman crontab -e
```

Thêm các dòng sau:

```cron
# Backup full hàng ngày lúc 1:00 AM
0 1 * * * /usr/bin/barman backup post-17

# WAL archiving check mỗi 5 phút
*/5 * * * * /usr/bin/barman cron
```

> ⚠️ **Ghi chú:** `barman cron` là lệnh bắt buộc phải chạy định kỳ — nó quản lý WAL streaming, copy WAL files, và maintenance. Nên để frequency 1–5 phút.

---

## 9. Kiểm Tra WAL và Recovery

### 9.1 Kiểm tra WAL được archive

```bash
sudo -u barman barman check-backup post-17 latest
```

### 9.2 Test restore (dry-run)

```bash
# Xem thông tin backup để lấy BACKUP_ID
sudo -u barman barman list-backup post-17

# Thực hiện restore vào thư mục test
sudo -u barman barman recover post-17 latest /tmp/pg17-restore-test --remote-ssh-command "ssh postgres@192.168.56.124"
```

> ⚠️ **Ghi chú (nội suy):** Lệnh `recover` ở trên restore **trực tiếp sang PostgreSQL server** qua SSH. Nếu muốn restore sang server khác hoặc locally trên Barman server, bỏ `--remote-ssh-command`. Đây là hành động nguy hiểm nếu làm trên production — luôn test trên server riêng.

---

## 10. Lệnh Vận Hành Hàng Ngày

| Lệnh | Mô tả |
|---|---|
| `barman check post-17` | Kiểm tra toàn bộ cấu hình |
| `barman status post-17` | Xem trạng thái instance |
| `barman list-backup post-17` | Liệt kê các backup |
| `barman backup post-17` | Chạy backup thủ công |
| `barman show-backup post-17 latest` | Chi tiết backup mới nhất |
| `barman delete post-17 <backup-id>` | Xoá backup cụ thể |
| `barman switch-wal post-17` | Buộc switch WAL (test archive) |
| `barman cron` | Chạy maintenance tasks |

---

## ⚠️ Các Ghi Chú Tổng Hợp

1. **Retention policy chưa được xác định** — File này dùng `RECOVERY WINDOW OF 7 DAYS` làm giá trị mặc định tạm thời. Cần xác nhận yêu cầu nghiệp vụ (RPO/RTO) để điều chỉnh.

2. **Password trong config file** — Password `123` được lưu plaintext trong `post-17.conf`. Trong môi trường production, nên dùng `.pgpass` hoặc `pg_service.conf` thay thế.

3. **Đường dẫn PostgreSQL data** — Guideline này giả định data directory là `/var/lib/pgsql/17/data` (mặc định của PGDG package trên RHEL/OL). Nếu bạn cài custom, cần điều chỉnh lại.

4. **Firewall** — Đảm bảo port `5432` và SSH (22) được mở giữa 2 server:
   ```bash
   sudo firewall-cmd --permanent --add-port=5432/tcp
   sudo firewall-cmd --permanent --add-service=ssh
   sudo firewall-cmd --reload
   ```

5. **SELinux** — Oracle Linux 9 mặc định bật SELinux. Nếu gặp lỗi permission khi rsync hoặc SSH, kiểm tra:
   ```bash
   sudo ausearch -m avc -ts recent
   sudo setsebool -P rsync_client 1
   ```

---

*Tài liệu được tạo cho: Barman 3.x + PostgreSQL 17 + Oracle Linux 9*
*Ngày tạo: 2026-06-13*
