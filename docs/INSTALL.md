# INSTALL — Step by Step Instalasi

Ringkasan semua langkah instalasi dari F1, F5, F7, F9, dan F10 dalam satu tempat. Detail dan penjelasan ada di file fase masing-masing (link di setiap bagian).

---

## 0. Prasyarat (macOS)

| Tool | Perintah | Cek |
|---|---|---|
| Docker Desktop | sudah terpasang | `docker compose version` (punyamu v2.34) |
| Git | bawaan / `brew install git` | `git --version` |
| Go | `brew install go` | `go version` (terpasang: 1.27.1) |
| Node.js LTS | `brew install node@22` | `node -v` |
| jq (opsional) | `brew install jq` | – |
| ngrok | `brew install ngrok/ngrok/ngrok` (atau pakai image Docker) | `ngrok version` |

Docker Desktop sudah terpasang di Mac-mu (Docker 28.1, Compose v2.34). Sebelum menjalankan perintah apa pun, buka aplikasinya: `open -a Docker`.
Docker Desktop → Settings → Resources: RAM **≥ 6 GB**.

> ✅ Terverifikasi untuk Mac M3 (6 Okt 2026): image `odoo:19.0` tersedia native **arm64**; PostgreSQL 16 memenuhi syarat minimum Odoo 19 (PG ≥ 13); semua modul yang dipasang ada di Community 19.0; demo data default **tidak** dipasang.

---

## 1. Odoo + PostgreSQL + Mailpit (dev) → [F1](implementation/F01-setup-odoo-docker.md)

```bash
cd ~/Documents/odoo-project
mkdir -p odoo/config odoo/addons nginx gateway monitor docs/screenshots scripts
git init
# create: .gitignore, .env.example, odoo/config/odoo.conf, docker-compose.yml,
#         docker-compose.override.yml, Makefile   (content in F1 §3)
cp .env.example .env
docker compose pull
docker compose up -d
docker compose ps
```

- Odoo: <http://localhost:8069>
- Mailpit: <http://localhost:8025>

Buat database `keripik` (tanpa demo) lalu install modul standar:
```bash
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d keripik -i base --no-http --stop-after-init --load-language=id_ID
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d keripik --no-http --stop-after-init \
  -i sale_management,website_sale,im_livechat,website_livechat,mrp,sale_mrp,purchase,purchase_stock,product_expiry,maintenance,hr,account,stock_account,mrp_account
docker compose restart odoo
```

Konfigurasi fungsional (produk, BoM, work center, route MTO, reorder, SMTP, live chat, user): **F1 §4**.
Uji alur MTO manual: **F1 §5**. Lalu backup:
```bash
make backup
```

---

## 2. Module `mrp_keripik` → [F2](implementation/F02-module-quality-check.md)–[F6](implementation/F06-telegram-odoo.md)

```bash
docker compose exec odoo odoo scaffold mrp_keripik /mnt/extra-addons
# follow F2 to restructure the module
make install MOD=mrp_keripik
make update MOD=mrp_keripik     # after each change
make test MOD=mrp_keripik
```

Setelah F5:
1. Buat user teknis `gateway_bot` & `monitor_bot`.
2. *Settings → Technical → Keripik → API Tokens*: generate token gateway & monitor → simpan di `.env` (`ODOO_GATEWAY_TOKEN`, `ODOO_MONITOR_TOKEN`).
3. Tambahkan `dbfilter = ^keripik$` di `odoo.conf` bila ada lebih dari satu DB.

Setelah F6 (Telegram):
1. Buat bot di @BotFather, simpan token di `.env` (`TELEGRAM_BOT_TOKEN`).
2. *System Parameters*: `mrp_keripik.telegram_bot_token`, `mrp_keripik.telegram_bot_username`, `mrp_keripik.telegram_poller = odoo`.

---

## 3. Next.js Monitor (dev) → [F7](implementation/F07-nextjs-monitor-deploy-v1.md)

```bash
cd ~/Documents/odoo-project
npx create-next-app@latest monitor --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm
cd monitor
npm install recharts server-only
# add files from F7 §2
cp .env.local.example .env.local      # fill ODOO_MONITOR_TOKEN, MONITOR_PASSWORD, MONITOR_SESSION_SECRET
npm run dev
```
→ <http://localhost:3000/monitor>

---

## 4. Deploy v1 (Nginx + ngrok) → [F7 §3](implementation/F07-nextjs-monitor-deploy-v1.md)

```bash
cp odoo/config/odoo.prod.conf.example odoo/config/odoo.prod.conf   # edit passwords
# create nginx/default.conf (F7 version) and docker-compose.prod.yml
# .env: NGROK_AUTHTOKEN, NGROK_DOMAIN, MONITOR_PASSWORD, MONITOR_SESSION_SECRET, ODOO_MONITOR_TOKEN
docker compose down
make prod-up
make prod-logs
```
Setelah online: `web.base.url = https://<NGROK_DOMAIN>` dan `web.base.url.freeze = True`.

---

## 5. Go Gateway + sensor_db + simulator → [F9](implementation/F09-machine-data-gateway.md)

```bash
cd ~/Documents/odoo-project/gateway
go mod init github.com/<your-github>/odoo-project/gateway
go get github.com/gofiber/fiber/v3 github.com/jackc/pgx/v5 github.com/google/uuid
# create files from F9 §2 (migrations, config, domain, main) and §3 (your implementation)
go vet ./... && go test ./... -race

# tokens
for v in GATEWAY_INBOUND_TOKEN GATEWAY_MONITOR_TOKEN MACHINE_TOKEN; do echo "$v=$(openssl rand -hex 24)"; done >> ../.env
# add sensor-db, gateway, simulator services to docker-compose.yml (F9 §5)
cd .. && docker compose up -d --build sensor-db gateway
curl -s localhost:8080/api/v1/healthz
```

Odoo *System Parameters*:
- `mrp_keripik.gateway_url = http://gateway:8080`
- `mrp_keripik.gateway_inbound_token = <GATEWAY_INBOUND_TOKEN>`
- `mrp_keripik.telegram_poller = gateway`

Work center: `machine_code` FRY-01, FRY-02, SCL-01.

Simulator:
```bash
docker compose --profile demo up -d simulator           # normal scenario
# or from host:
cd gateway && go run ./cmd/simulator -token "$MACHINE_TOKEN" -scenario overheat -overheat-after 30s -speed 5
```

---

## 6. Deploy lengkap + data demo → [F10](implementation/F10-streaming-deploy-readme.md)

```bash
# replace nginx/default.conf with the F10 version; update docker-compose.prod.yml
make prod-down
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile public --profile demo up -d --build

# demo data
docker compose exec -T odoo odoo shell -c /etc/odoo/odoo.conf -d keripik --no-http < scripts/demo_data.py
```

---

## 7. Perintah Harian

| Tujuan | Perintah |
|---|---|
| Start dev | `make up` |
| Log Odoo | `make logs` |
| Update module | `make update MOD=mrp_keripik` |
| Test Odoo | `make test MOD=mrp_keripik` |
| Odoo shell | `make shell` |
| psql Odoo | `make psql` |
| psql sensor_db | `docker compose exec sensor-db psql -U sensor -d sensor_db` |
| Test Go | `cd gateway && go test ./... -race` |
| Backup | `make backup` |
| Start produksi | `make prod-up` |
| Stop semua | `make down` / `make prod-down` |

## 8. Reset Total (⚠️ menghapus data)
```bash
docker compose down -v      # removes volumes: odoo DB, filestore, sensor_db
```

## 9. Port

| Layanan | Dev (host) | Prod |
|---|---|---|
| Odoo HTTP | 8069 | via Nginx `/` |
| Odoo websocket | – (workers=0) | via Nginx `/websocket` → 8072 |
| Mailpit UI | 8025 | 127.0.0.1:8025 |
| PostgreSQL Odoo | 5433 (opsional) | – |
| sensor_db | 5434 (opsional) | – |
| Gateway | 8080 | via Nginx `/gateway/` |
| Next.js | 3000 (`npm run dev`) | via Nginx `/monitor` |
| Nginx | – | 80 (→ ngrok) |
