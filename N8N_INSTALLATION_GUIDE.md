# Hướng dẫn cài đặt n8n Production trên Ubuntu 24.04 với Docker

## Thông tin Server
- **OS:** Ubuntu 24.04 LTS
- **CPU:** 2 cores
- **RAM:** 3GB
- **Disk:** 40GB
- **IP:** 103.216.117.22
- **Domain:** n8n.socialmate.site
- **SSL:** Let's Encrypt (Certbot) + Cloudflare Full (strict)

---

## Bước 1: Cấu hình DNS trên Cloudflare

1. Đăng nhập vào Cloudflare Dashboard
2. Chọn domain `socialmate.site`
3. Vào tab **DNS** → **Records**
4. Thêm record mới:

| Type | Name | Content | Proxy status | TTL |
|------|------|---------|--------------|-----|
| A | n8n | 103.216.117.22 | Proxied (orange cloud) | Auto |

5. Vào tab **SSL/TLS** → chọn mode **Full (strict)** (recommended với Let's Encrypt)

---

## Bước 2: Chuẩn bị thư mục trên VPS

SSH vào server và tạo cấu trúc thư mục:

```bash
# SSH vào server
ssh root@103.216.117.22

# Tạo thư mục cho n8n
mkdir -p /opt/n8n
cd /opt/n8n

# Tạo thư mục data với đúng permissions
# n8n container chạy với user node (UID 1000)
mkdir -p data postgres-data redis-data
chown -R 1000:1000 data
```

> **⚠️ Quan trọng:** Thư mục `data` phải thuộc sở hữu của UID 1000 (user node trong container), nếu không n8n sẽ báo lỗi `EACCES: permission denied`.

---

## Bước 3: Tạo file Docker Compose

Tạo file `docker-compose.yml`:

```bash
nano /opt/n8n/docker-compose.yml
```

Nội dung file:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: n8n-postgres
    restart: always
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
      POSTGRES_NON_ROOT_USER: n8n
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U n8n -d n8n']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      # Database
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}

      # General
      N8N_HOST: ${N8N_HOST}
      N8N_PORT: 5678
      N8N_PROTOCOL: https
      WEBHOOK_URL: https://${N8N_HOST}/

      # Security
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}

      # Execution settings - optimized for 3GB RAM
      EXECUTIONS_MODE: regular
      EXECUTIONS_PROCESS: main
      EXECUTIONS_TIMEOUT: 3600
      EXECUTIONS_TIMEOUT_MAX: 7200
      EXECUTIONS_DATA_SAVE_ON_ERROR: all
      EXECUTIONS_DATA_SAVE_ON_SUCCESS: all
      EXECUTIONS_DATA_SAVE_ON_PROGRESS: true
      EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS: true
      EXECUTIONS_DATA_PRUNE: true
      EXECUTIONS_DATA_MAX_AGE: 336

      # Queue mode for concurrent executions
      QUEUE_BULL_REDIS_HOST: redis
      QUEUE_HEALTH_CHECK_ACTIVE: true

      # Performance tuning for 3GB RAM
      N8N_DEFAULT_BINARY_DATA_MODE: filesystem
      N8N_AVAILABLE_BINARY_DATA_MODES: filesystem
      GENERIC_TIMEZONE: Asia/Ho_Chi_Minh
      TZ: Asia/Ho_Chi_Minh

      # Logging
      N8N_LOG_LEVEL: info
      N8N_LOG_OUTPUT: console

      # Disable Python Task Runner (tránh warning nếu không dùng Python nodes)
      N8N_RUNNERS_DISABLED: true

      # Disable diagnostics (optional)
      N8N_DIAGNOSTICS_ENABLED: false
      N8N_VERSION_NOTIFICATIONS_ENABLED: true

    volumes:
      - ./data:/home/node/.n8n
    ports:
      - "127.0.0.1:5678:5678"
    networks:
      - n8n-network

  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: always
    volumes:
      - ./redis-data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

networks:
  n8n-network:
    driver: bridge
```

---

## Bước 4: Tạo file Environment

Tạo file `.env`:

```bash
nano /opt/n8n/.env
```

Nội dung:

```bash
# Domain
N8N_HOST=n8n.socialmate.site

# Database Password - THAY ĐỔI PASSWORD NÀY!
POSTGRES_PASSWORD=YourSecurePassword123!@#

# Encryption Key - Tạo key ngẫu nhiên, QUAN TRỌNG: LƯU GIỮ CẨN THẬN!
N8N_ENCRYPTION_KEY=your-32-character-encryption-key
```

**Tạo Encryption Key ngẫu nhiên:**

```bash
# Chạy lệnh này để tạo key ngẫu nhiên
openssl rand -hex 32

# Copy output và paste vào N8N_ENCRYPTION_KEY trong file .env
```

**Tạo Database Password:**

```bash
# Chạy lệnh này để tạo password ngẫu nhiên
openssl rand -base64 24
```

---

## Bước 5: Cấu hình Nginx Reverse Proxy

> **⚠️ Important:** Chúng ta sẽ tạo config HTTP-only đơn giản trước, sau đó để Certbot tự động thêm SSL config. Cách này tránh lỗi "no ssl_certificate" khi test nginx.

### 5.1 Cài đặt Nginx và Certbot:

```bash
# Update packages
apt update

# Cài đặt Nginx
apt install -y nginx

# Cài đặt Certbot (Let's Encrypt)
apt install -y certbot python3-certbot-nginx

# Verify installations
nginx -v
certbot --version
```

### 5.2 Tạo file cấu hình Nginx (HTTP-only ban đầu):

```bash
nano /etc/nginx/sites-available/n8n
```

Nội dung (HTTP-only, Certbot sẽ add HTTPS sau):

```nginx
# n8n Configuration - n8n.socialmate.site
# Initial HTTP-only config - Certbot will add HTTPS automatically

# Upstream for n8n
upstream n8n_backend {
    server 127.0.0.1:5678;
    keepalive 64;
}

server {
    listen 80;
    listen [::]:80;
    server_name n8n.socialmate.site;

    # Logging
    access_log /var/log/nginx/n8n.access.log;
    error_log /var/log/nginx/n8n.error.log;

    # Client body size for file uploads
    client_max_body_size 100M;

    # Let's Encrypt challenge (for Certbot)
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # Timeout settings for long-running workflows
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
    proxy_read_timeout 300;
    send_timeout 300;

    # Main location
    location / {
        proxy_pass http://n8n_backend;
        proxy_http_version 1.1;

        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;

        # WebSocket support (required for n8n)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Buffering
        proxy_buffering off;
        proxy_cache off;

        # Chunked transfer
        chunked_transfer_encoding on;
    }

    # Webhook endpoints - no rate limiting
    location ~ ^/webhook {
        proxy_pass http://n8n_backend;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        client_max_body_size 100M;
    }

    # Health check endpoint
    location /healthz {
        proxy_pass http://n8n_backend/healthz;
        proxy_http_version 1.1;
        access_log off;
    }
}
```

### 5.3 Kích hoạt site Nginx:

```bash
# Tạo symlink
ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/

# Kiểm tra cấu hình (should pass - no SSL errors)
nginx -t

# Reload Nginx
systemctl reload nginx
systemctl enable nginx
```

### 5.4 Test HTTP Access:

```bash
# Test that Nginx is proxying correctly (n8n chưa chạy nên sẽ 502, nhưng nginx hoạt động)
curl -I http://n8n.socialmate.site

# Nếu DNS đúng, sẽ thấy response từ nginx
```

---

## Bước 6: Cấu hình SSL với Let's Encrypt (Certbot)

> **⚠️ Important về Cloudflare:** Nếu bạn đang dùng Cloudflare với proxy (orange cloud ☁️), bạn có 2 options:
> 1. **Tạm thời disable proxy** (orange → gray cloud) để issue certificate, sau đó bật lại
> 2. **Dùng Cloudflare SSL mode "Full"** - giữ proxy enabled

### 6.1 Chuẩn bị cho SSL Certificate:

```bash
# Tạo thư mục cho Let's Encrypt challenges
mkdir -p /var/www/html

# Verify DNS resolution (should point to your VPS)
nslookup n8n.socialmate.site

# Should return: 103.216.117.22
```

**Nếu dùng Cloudflare với proxy (orange cloud):**

**Option A: Tạm thời disable proxy (Recommended cho lần đầu)**
1. Vào Cloudflare DNS settings
2. Click orange cloud → gray cloud (DNS only)
3. Đợi 1-2 phút cho DNS propagation
4. Tiếp tục với Certbot bên dưới
5. Bật lại proxy (gray → orange) sau khi có certificate

**Option B: Giữ proxy enabled**
- Cloudflare Settings → SSL/TLS → Set to **"Full"**
- Certbot có thể hoạt động với proxy enabled

### 6.2 Lấy SSL Certificate:

```bash
# Chạy Certbot cho n8n domain (sẽ tự động modify Nginx config)
certbot --nginx -d n8n.socialmate.site

# Follow prompts:
# - Enter email: your-email@gmail.com
# - Agree to Terms of Service: Yes (A)
# - Share email with EFF: No (N)
# - Redirect HTTP to HTTPS: Yes (2) ← Certbot sẽ add redirect tự động
```

**Certbot sẽ tự động:**
- ✅ Lấy SSL certificate từ Let's Encrypt
- ✅ Modify Nginx config để thêm HTTPS server block
- ✅ Thêm SSL certificate paths
- ✅ Thêm HTTP → HTTPS redirect
- ✅ Setup auto-renewal cron job

**Expected output:**

```
Congratulations! You have successfully enabled HTTPS on n8n.socialmate.site

IMPORTANT NOTES:
 - Your certificate and chain have been saved at:
   /etc/letsencrypt/live/n8n.socialmate.site/fullchain.pem
 - Your key file has been saved at:
   /etc/letsencrypt/live/n8n.socialmate.site/privkey.pem
 - Your certificate will expire on [DATE]. To obtain a new or
   tweaked version, simply run certbot again with the "certonly" option.
```

### 6.3 Bật lại Cloudflare Proxy (nếu đã disable ở 6.1):

Nếu bạn đã tạm disable Cloudflare proxy:
1. Quay lại Cloudflare DNS settings
2. Click gray cloud → orange cloud (Proxied)
3. SSL mode nên là **"Full (strict)"**

### 6.4 Cấu hình Auto-Renewal:

Certbot đã tự động setup cron job cho auto-renewal, nhưng hãy verify:

```bash
# Check Certbot timer
systemctl status certbot.timer

# Test renewal (dry-run)
certbot renew --dry-run

# Nếu thành công, certificates sẽ tự động renew mỗi 60 ngày
```

### 6.5 Verify SSL Configuration:

```bash
# Test HTTPS (sau khi n8n đã chạy)
curl -I https://n8n.socialmate.site

# Expected: HTTP/2 200 OK
```

**SSL Labs Test (Recommended):**

Truy cập: https://www.ssllabs.com/ssltest/
Nhập domain: `n8n.socialmate.site`
**Target: A+ rating** ✅

---

## Bước 7: Khởi động n8n

```bash
cd /opt/n8n

# Tạo thư mục redis-data
mkdir -p redis-data

# Pull images
docker compose pull

# Khởi động services
docker compose up -d

# Kiểm tra logs
docker compose logs -f

# Kiểm tra status
docker compose ps
```

---

## Bước 8: Cấu hình Cloudflare Security

### 8.1 SSL/TLS Settings:
- Vào SSL/TLS → Overview → chọn **Full (strict)** (recommended với Let's Encrypt)

### 8.2 Edge Certificates:
- Always Use HTTPS: **ON**
- Automatic HTTPS Rewrites: **ON**
- Minimum TLS Version: **1.2**

### 8.3 Page Rules (Optional):
Tạo Page Rule cho `n8n.socialmate.site/*`:
- SSL: Full (strict)
- Cache Level: Bypass

---

## Bước 9: Thiết lập n8n User

1. Truy cập https://n8n.socialmate.site
2. Tạo owner account đầu tiên
3. Thiết lập email và password mạnh

---

## Bước 10: Cấu hình Production bổ sung

### 10.1 Tự động khởi động khi reboot:

Docker Compose với `restart: always` đã xử lý điều này.

### 10.2 Backup Script:

```bash
nano /opt/n8n/backup.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/opt/n8n/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup PostgreSQL
docker exec n8n-postgres pg_dump -U n8n n8n > $BACKUP_DIR/n8n_db_$DATE.sql

# Backup n8n data
tar -czf $BACKUP_DIR/n8n_data_$DATE.tar.gz -C /opt/n8n data

# Giữ lại 7 ngày backup
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

```bash
chmod +x /opt/n8n/backup.sh

# Thêm cronjob backup hàng ngày lúc 2h sáng
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/n8n/backup.sh >> /var/log/n8n-backup.log 2>&1") | crontab -
```

### 10.3 Monitoring:

```bash
# Kiểm tra health
curl -s http://127.0.0.1:5678/healthz

# Kiểm tra resource usage
docker stats --no-stream
```

---

## Bước 11: Tối ưu hóa cho 3GB RAM

### 11.1 Giới hạn memory cho containers:

Cập nhật `docker-compose.yml`, thêm vào mỗi service:

```yaml
services:
  postgres:
    # ... existing config ...
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  n8n:
    # ... existing config ...
    deploy:
      resources:
        limits:
          memory: 1536M
        reservations:
          memory: 512M

  redis:
    # ... existing config ...
    deploy:
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M
```

### 11.2 Cấu hình Swap (khuyến nghị):

```bash
# Kiểm tra swap hiện tại
free -h

# Tạo swap file 2GB
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Thêm vào fstab để tự động mount
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Tối ưu swappiness
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p
```

---

## Các lệnh quản lý hữu ích

```bash
# Vào thư mục n8n
cd /opt/n8n

# Xem logs
docker compose logs -f n8n
docker compose logs -f postgres

# Restart services
docker compose restart

# Stop services
docker compose down

# Start services
docker compose up -d

# Update n8n lên version mới
docker compose pull
docker compose up -d

# Vào PostgreSQL shell
docker exec -it n8n-postgres psql -U n8n -d n8n

# Vào n8n container
docker exec -it n8n /bin/sh

# Xem disk usage
docker system df
```

---

## Troubleshooting

### Lỗi nginx -t fails với "no ssl_certificate is defined" ⚠️ COMMON

**Error message:**
```
[emerg] nginx: no "ssl_certificate" is defined for the "listen ... ssl" directive
nginx: configuration file /etc/nginx/nginx.conf test failed
```

**Nguyên nhân:** Bạn tạo config với `listen 443 ssl` nhưng chưa chạy Certbot để lấy SSL certificates.

**Giải pháp:** Sử dụng config HTTP-only trong hướng dẫn này. Certbot sẽ tự động thêm HTTPS.

```bash
# Nếu đã có config với HTTPS block, chạy Certbot ngay:
certbot --nginx -d n8n.socialmate.site
```

### Lỗi Certbot không thể verify domain

**Nguyên nhân:** Cloudflare proxy đang enabled hoặc DNS chưa propagate.

**Giải pháp:**
```bash
# Tạm thời disable Cloudflare proxy (orange → gray cloud)
# Đợi 1-2 phút, rồi chạy lại:
certbot --nginx -d n8n.socialmate.site
```

### Lỗi Let's Encrypt rate limit

**Nguyên nhân:** Đã request quá nhiều certificates (5 per week per domain).

**Giải pháp:** Đợi 1 tuần, hoặc dùng staging certificates cho testing:

```bash
# Dùng --staging flag cho testing
certbot --nginx --staging -d n8n.socialmate.site
```

### Lỗi 502 Bad Gateway:
```bash
# Kiểm tra n8n có chạy không
docker compose ps
docker compose logs n8n
```

### Lỗi EACCES: permission denied ⚠️ COMMON

**Error message:**
```
Error: EACCES: permission denied, open '/home/node/.n8n/config'
```

**Nguyên nhân:** Thư mục `./data` được tạo bởi root, nhưng n8n container chạy với user `node` (UID 1000).

**Giải pháp:**
```bash
cd /opt/n8n

# Thay đổi ownership cho thư mục data
chown -R 1000:1000 ./data

# Restart n8n
docker compose restart n8n
```

### Lỗi database connection:
```bash
# Kiểm tra postgres
docker compose logs postgres
docker exec -it n8n-postgres pg_isready -U n8n -d n8n
```

### Lỗi WebSocket:
- Kiểm tra Cloudflare có enable WebSocket không (mặc định đã bật)
- Kiểm tra nginx config có `proxy_set_header Upgrade` và `Connection "upgrade"`

### Out of memory:
```bash
# Kiểm tra memory
free -h
docker stats

# Restart để giải phóng memory
docker compose restart
```

---

## Cấu trúc thư mục cuối cùng

```
/opt/n8n/
├── docker-compose.yml
├── .env
├── backup.sh
├── data/                  # n8n data
├── postgres-data/         # PostgreSQL data
├── redis-data/            # Redis data
└── backups/               # Backup files
```

---

## Checklist sau khi cài đặt

- [ ] DNS đã trỏ đúng về IP VPS
- [ ] Nginx config đã tạo và test thành công (`nginx -t`)
- [ ] Certbot đã lấy SSL certificate thành công
- [ ] Cloudflare SSL mode = Full (strict)
- [ ] Cloudflare proxy đã bật lại (orange cloud)
- [ ] n8n accessible tại https://n8n.socialmate.site
- [ ] Owner account đã được tạo
- [ ] Auto-renewal đã test thành công (`certbot renew --dry-run`)
- [ ] Backup script đã được thiết lập
- [ ] Swap đã được cấu hình
- [ ] Tất cả containers đang chạy (`docker compose ps`)

---

## Thông tin bảo mật quan trọng

⚠️ **LƯU Ý:**
1. **KHÔNG** commit file `.env` lên git
2. **LƯU GIỮ** `N8N_ENCRYPTION_KEY` cẩn thận - mất key = mất toàn bộ credentials
3. Thay đổi `POSTGRES_PASSWORD` thành password mạnh
4. Sử dụng Cloudflare Access hoặc VPN để bảo vệ thêm (optional)
