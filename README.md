# n8n VPS Installation Guide

Hướng dẫn chi tiết cài đặt n8n Production trên Ubuntu 24.04 với Docker.

## Features

- Docker Compose setup với PostgreSQL + Redis
- Nginx reverse proxy với SSL (Let's Encrypt)
- Cloudflare integration
- Tối ưu cho VPS 3GB RAM
- Backup script tự động
- Health check và monitoring

## System Requirements

- **OS:** Ubuntu 24.04 LTS
- **CPU:** 2 cores (minimum)
- **RAM:** 3GB (minimum)
- **Disk:** 40GB
- **Domain:** Configured with Cloudflare (optional)

## Quick Start

1. Clone repository này về VPS
2. Copy `.env.example` thành `.env` và cấu hình
3. Làm theo hướng dẫn trong [N8N_INSTALLATION_GUIDE.md](N8N_INSTALLATION_GUIDE.md)

## Stack

| Service    | Image                  | Purpose           |
|------------|------------------------|-------------------|
| n8n        | n8nio/n8n:latest       | Workflow Engine   |
| PostgreSQL | postgres:16-alpine     | Database          |
| Redis      | redis:7-alpine         | Queue/Cache       |
| Nginx      | System package         | Reverse Proxy     |

## Directory Structure

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

## Documentation

- [Installation Guide (Vietnamese)](N8N_INSTALLATION_GUIDE.md)

## Security Notes

- **NEVER** commit `.env` file to git
- **SAVE** `N8N_ENCRYPTION_KEY` securely - losing this key means losing all credentials
- Use strong passwords for `POSTGRES_PASSWORD`
- Consider using Cloudflare Access or VPN for additional protection

## License

MIT License - see [LICENSE](LICENSE) for details.
