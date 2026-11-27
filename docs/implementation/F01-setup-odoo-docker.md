# F1. Setup Odoo 19 + PostgreSQL + Mailpit (Docker) & Konfigurasi Fungsional

| | |
|---|---|
| **Requirement FSD** | FR-SAL-01…05, FR-MRP-01…06, FR-INV-01…03, FR-PUR-01…03, FR-MNT-01…03, FR-INVC-01…02, FR-EML-04 |
| **Minggu** | 1 (5–11 Okt 2026) |
| **Hasil** | Odoo berjalan di Docker, semua modul standar terkonfigurasi, alur MTO teruji manual |

---

## 1. Tujuan & Konsep

### 1.1 Odoo vs Frappe: cara berjalan

| Frappe/ERPNext | Odoo |
|---|---|
| `bench` mengelola site, app, worker, redis, socketio | Satu proses `odoo-bin` (atau image Docker) + PostgreSQL. Tidak butuh Redis |
| Site = folder + database MariaDB | Database PostgreSQL; satu instance bisa melayani banyak DB (`dbfilter`) |
| `bench install-app` | `-i module` atau tombol *Activate* di Apps |
| `bench migrate` | `-u module` (update/upgrade module) |
| `bench start` (dev) | `odoo --dev=xml,qweb` (reload XML/QWeb tanpa restart; Python tetap perlu restart) |
| Background job (RQ) | `ir.cron` dijalankan oleh thread/worker cron |
| Realtime (socketio) | Bus + websocket (port 8072 saat `workers > 0`) |

### 1.2 Kenapa Mailpit
Semua email (SMTP port 1025) ditangkap dan ditampilkan di UI `http://localhost:8025`, jadi tidak ada email yang benar-benar terkirim ke pelanggan.

---

## 2. Prasyarat

Di Mac kamu sudah terpasang **Docker Desktop** (Docker 28.1, Compose v2.34). Yang perlu dilakukan hanya **menjalankan aplikasinya**:

```bash
open -a Docker               # start Docker Desktop (wait until the whale icon says "running")
docker info | grep -E "Architecture|Total Memory"   # expect: aarch64, >= 6 GiB
```

Docker Desktop → **Settings → Resources**: Memory **≥ 6 GB**, CPU ≥ 4. Settings → General: pastikan **"Use Virtualization framework"** dan file sharing **VirtioFS** aktif (default di versi baru) agar bind mount folder addons cepat.

> ✅ **Mac M3 (Apple Silicon) didukung native.** Sudah dicek 6 Okt 2026: image resmi `odoo:19.0` di Docker Hub tersedia untuk **arm64** (juga amd64), dan wkhtmltopdf versi arm64 sudah terpasang di image. **Tidak perlu** `platform: linux/amd64` / Rosetta.

> **Kenapa Docker, bukan instal langsung di macOS?** Dokumentasi resmi Odoo 19 hanya membahas instalasi *source* untuk macOS (Python, PostgreSQL, wkhtmltopdf, dan dependensi dipasang manual via Homebrew), dan itu rawan masalah di Apple Silicon (library C untuk `psycopg2`, `lxml`, `python-ldap`; wkhtmltopdf versi patched). Image Docker resmi sudah membawa semua dependensi yang benar, jadi untuk pemula ini jalur paling aman.
> Kalau ingin instal source untuk belajar: Python **3.10–3.14** dan PostgreSQL **≥ 13** (dari `odoo/release.py` 19.0); ikuti <https://www.odoo.com/documentation/19.0/administration/on_premise/source.html>.

---

## 3. Langkah

### Langkah 1 — Struktur folder

```bash
cd ~/Documents/odoo-project
mkdir -p odoo/config odoo/addons nginx gateway monitor docs/screenshots
git init
```

### Langkah 2 — `.gitignore`

`odoo-project/.gitignore`
```gitignore
# env & secrets
.env
*.local

# python
__pycache__/
*.pyc

# node
monitor/node_modules/
monitor/.next/

# go
gateway/bin/

# OS / editor
.DS_Store
.idea/
.vscode/

# backups
*.dump
*.zip
```

### Langkah 3 — `.env.example` lalu salin ke `.env`

`odoo-project/.env.example`
```dotenv
# ---------- PostgreSQL (Odoo) ----------
POSTGRES_USER=odoo
POSTGRES_PASSWORD=change-me-odoo-db
POSTGRES_DB=postgres

# ---------- Odoo ----------
ODOO_DB_NAME=keripik
ODOO_PORT=8069

# ---------- Mailpit ----------
MAILPIT_UI_PORT=8025

# ---------- Diisi di fase berikutnya ----------
# F6
TELEGRAM_BOT_TOKEN=
# F7
MONITOR_PASSWORD=
MONITOR_SESSION_SECRET=
ODOO_MONITOR_TOKEN=
NGROK_DOMAIN=
# F9
SENSOR_DB_PASSWORD=change-me-sensor-db
ODOO_GATEWAY_TOKEN=
GATEWAY_INBOUND_TOKEN=
GATEWAY_MONITOR_TOKEN=
MACHINE_TOKEN=
```

```bash
cp .env.example .env
```

### Langkah 4 — `odoo/config/odoo.conf`

```ini
[options]
; --- database ---
db_host = db
db_port = 5432
db_user = odoo
db_password = change-me-odoo-db
; In dev we allow the database manager; disable in production (F7)
list_db = True
; Master password for the database manager (change it!)
admin_passwd = change-me-master

; --- paths ---
addons_path = /mnt/extra-addons
data_dir = /var/lib/odoo

; --- server ---
; workers = 0 -> threaded mode, websocket served on 8069 (simple for dev)
workers = 0
proxy_mode = False
log_level = info

; --- mail (Mailpit) ---
smtp_server = mailpit
smtp_port = 1025
smtp_ssl = False
email_from = noreply@keripik.local
```

> `db_password` harus sama dengan `POSTGRES_PASSWORD` di `.env`. File `.conf` tidak membaca env var, jadi jangan commit password produksi (untuk produksi, buat `odoo.prod.conf` yang di-ignore, lihat F7).

### Langkah 5 — `docker-compose.yml`

`odoo-project/docker-compose.yml`
```yaml
name: keripik

services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - odoo-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 10

  odoo:
    image: odoo:19.0          # multi-arch: runs natively (arm64) on Apple Silicon
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      HOST: db
      USER: ${POSTGRES_USER}
      PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./odoo/config:/etc/odoo
      - ./odoo/addons:/mnt/extra-addons
      - odoo-web-data:/var/lib/odoo
    ports:
      - "${ODOO_PORT:-8069}:8069"

  mailpit:
    image: axllent/mailpit:latest   # pin a version tag once you verify it
    restart: unless-stopped
    ports:
      - "${MAILPIT_UI_PORT:-8025}:8025"
    environment:
      MP_SMTP_AUTH_ACCEPT_ANY: "1"
      MP_SMTP_AUTH_ALLOW_INSECURE: "1"

volumes:
  odoo-db-data:
  odoo-web-data:
```

`odoo-project/docker-compose.override.yml` (otomatis dipakai saat dev)
```yaml
services:
  odoo:
    # --dev=xml,qweb reloads views/templates from disk without restart
    # (the image entrypoint accepts "odoo <args>" and appends db host/user/password)
    command: ["odoo", "--dev=xml,qweb"]
  db:
    ports:
      - "5433:5432"   # optional: connect with DBeaver/psql from host
```

### Langkah 6 — `Makefile` (shortcut)

`odoo-project/Makefile`
```makefile
DB ?= keripik
MOD ?= mrp_keripik
ODOO_EXEC = docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d $(DB) --no-http --stop-after-init

.PHONY: up down logs restart shell psql update install test backup

up:
	docker compose up -d

down:
	docker compose down

logs:
	docker compose logs -f --tail=200 odoo

restart:
	docker compose restart odoo

shell:
	docker compose exec odoo odoo shell -c /etc/odoo/odoo.conf -d $(DB) --no-http

psql:
	docker compose exec db psql -U odoo -d $(DB)

install:
	$(ODOO_EXEC) -i $(MOD)

update:
	$(ODOO_EXEC) -u $(MOD)

test:
	$(ODOO_EXEC) -u $(MOD) --test-tags /$(MOD) --log-level=test

backup:
	docker compose exec db pg_dump -U odoo -Fc $(DB) > backup-$(DB)-$$(date +%Y%m%d%H%M).dump
```

> `--no-http` mencegah proses kedua bentrok port 8069 dengan Odoo yang sedang berjalan. Tanpa itu muncul error `Address already in use`.

### Langkah 7 — Jalankan

```bash
docker compose pull
docker compose up -d
docker compose ps          # semua harus "running"/"healthy"
docker compose logs -f odoo
```

Buka:
- Odoo → <http://localhost:8069>
- Mailpit → <http://localhost:8025>

### Langkah 8 — Buat database

**Opsi A (UI):** <http://localhost:8069/web/database/manager>
- Master Password: `change-me-master`
- Database Name: `keripik`
- Email: `admin@keripik.local`, Password: (catat)
- Language: **Indonesian / Bahasa Indonesia**
- Country: Indonesia
- Demo data: **tidak dicentang**

**Opsi B (CLI):**
```bash
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d keripik -i base \
  --no-http --stop-after-init --load-language=id_ID
```
> ✅ Terverifikasi di source 19.0 (`odoo/tools/config.py`): **demo data tidak dipasang secara default** (`--without-demo` adalah default). Untuk database latihan berisi data contoh, tambahkan `--with-demo`. Database `keripik` sengaja dibuat **tanpa** demo.
> Kode bahasa Indonesia: `id_ID`.

> 💡 Bekerja dalam **Bahasa Inggris** di backend memudahkan mencocokkan dengan dokumentasi dan source. Rekomendasi: user admin pakai English, user operator pakai Indonesia.

### Langkah 9 — Install modul standar

```bash
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d keripik --no-http --stop-after-init \
  -i sale_management,website_sale,im_livechat,website_livechat,mrp,sale_mrp,purchase,purchase_stock,product_expiry,maintenance,hr,account,stock_account,mrp_account
docker compose restart odoo
```

> ✅ Terverifikasi 6 Okt 2026: semua modul di atas ada di repo Community `odoo/odoo` branch 19.0. `mrp_maintenance` **tidak ada** (Enterprise), jadi tidak dipasang; equipment cukup diberi nama sesuai work center.

---

## 4. Konfigurasi Fungsional (tanpa coding)

> Menu ditulis dalam bahasa Inggris (backend admin). Bila memakai bahasa Indonesia, nama menu mengikuti terjemahan.

### 4.1 Perusahaan
*Settings → Companies → (company)*: Nama "CV Keripik Tempe Nusantara", alamat, mata uang **IDR**, logo. *Settings → Users → admin → Preferences → Timezone* **Asia/Jakarta**.

### 4.2 Settings modul

| Menu | Aktifkan |
|---|---|
| *Inventory → Configuration → Settings* | Storage Locations, Multi-Step Routes (untuk lihat route), Lots & Serial Numbers, Expiration Dates, Units of Measure, Packages (opsional) |
| *Manufacturing → Configuration → Settings* | Work Orders, By-Products, (Quality: tidak ada di CE), Unlock MO (opsional) |
| *Purchase → Configuration → Settings* | Purchase Agreements (opsional) |
| *Sales → Configuration → Settings* | Unit of Measure; Invoicing policy default: Delivered quantities |
| *Website → Configuration → Settings* | Live Chat channel |

### 4.3 Lokasi & putaway
*Inventory → Configuration → Locations* → buat di bawah `WH/Stock`:
- `WH/Stock/Bahan Baku`
- `WH/Stock/Produk Jadi`

*Inventory → Configuration → Product Categories*:

| Kategori | Removal Strategy |
|---|---|
| All / Bahan Baku | FEFO |
| All / Setengah Jadi | FEFO |
| All / Produk Jadi | FEFO |
| All / By-product | FIFO |
| All / Kemasan | FIFO |

*Inventory → Configuration → Putaway Rules*: kategori Bahan Baku & Kemasan → `WH/Stock/Bahan Baku`; Produk Jadi → `WH/Stock/Produk Jadi`.

### 4.4 Route MTO
*Inventory → Configuration → Routes* → filter **Archived** → **Replenish on Order (MTO)** → *Unarchive*.

### 4.5 Vendor
*Purchase → Orders → Vendors*:
- **CV Tempe Sejahtera** (tempe mentah)
- **UD Sumber Minyak** (minyak goreng)
- **Toko Bumbu Jaya** (tepung bumbu, bumbu)
- **PT Plastik Kemas** (plastik kemasan)

### 4.6 Produk

| Produk | Kategori | Tipe / Track Inventory | UoM | Tracking | Expiry (hari) / Alert (hari) | Routes | Harga beli / jual | Vendor |
|---|---|---|---|---|---|---|---|---|
| Tempe Mentah | Bahan Baku | Goods, tracked | kg | Lot | 3 / 1 | Buy | 12.000 / – | CV Tempe Sejahtera, lead 1 hari |
| Tepung Bumbu | Bahan Baku | Goods, tracked | kg | Lot | 180 / 14 | Buy | 12.000 | Toko Bumbu Jaya |
| Minyak Goreng | Bahan Baku | Goods, tracked | L | Lot | 365 / 14 | Buy | 18.000 | UD Sumber Minyak |
| Bumbu Keripik | Bahan Baku | Goods, tracked | kg | Lot | 180 / 14 | Buy | 60.000 | Toko Bumbu Jaya |
| Plastik Kemasan 200 g | Kemasan | Goods, tracked | Units | – | – | Buy | 500 | PT Plastik Kemas |
| Tempe Iris | Setengah Jadi | Goods, tracked | kg | Lot | 1 / 0 | Manufacture + MTO | (dihitung) | – |
| Keripik Siap Kemas | Setengah Jadi | Goods, tracked | kg | Lot | 60 / 14 | Manufacture + MTO | (dihitung) | – |
| **Keripik Tempe 200 g** | Produk Jadi | Goods, tracked | Units | Lot | 90 / 14 | Manufacture + MTO | – / **15.000** | – |
| Minyak Bekas | By-product | Goods, tracked | L | – | – | – | 0 | – |
| Remahan Keripik | By-product | Goods, tracked | kg | – | – | – | 0 / 5.000 | – |

Catatan:
- Odoo 18+: tidak ada lagi "Storable Product". Pilih **Goods** lalu centang **Track Inventory** ⚠️ (cek di 19).
- Expiry diatur di tab Inventory → *Expiration Date* (centang) → *Expiration Time*, *Alert Time*.
- Keripik Tempe 200 g: *Sales → Invoicing Policy = Delivered quantities*; *Website*: Published; aktifkan "Continue selling when out of stock" (MTO).
- Tempe Iris & Keripik Siap Kemas: `Can be Sold` dimatikan.

### 4.7 Work center
*Manufacturing → Configuration → Work Centers*

| Nama | Kode | Capacity | Time efficiency | Cost per hour (Rp) | Setup / Cleanup (menit) |
|---|---|---|---|---|---|
| Mesin Pengiris | WC-IRIS | 1 | 100% | 25.000 | 5 / 5 |
| Penggoreng 1 | WC-GORENG-1 | 1 | 100% | 40.000 | 10 / 10 |
| Penggoreng 2 | WC-GORENG-2 | 1 | 95% | 40.000 | 10 / 10 |
| Meja Pembumbuan | WC-BUMBU | 2 | 100% | 20.000 | 0 / 5 |
| Mesin Kemas | WC-KEMAS | 1 | 100% | 30.000 | 5 / 5 |

Penggoreng 1 → *Alternative Workcenters*: Penggoreng 2.

### 4.8 BoM (*Manufacturing → Products → Bills of Materials*)

**BoM 1 — Tempe Iris** · qty **10 kg** · consumption *Warning*

| Komponen | Qty | Consumed in operation |
|---|---|---|
| Tempe Mentah | 10,5 kg | Pengirisan |

Operasi: **Pengirisan** · WC-IRIS · 30 menit.

**BoM 2 — Keripik Siap Kemas** · qty **6 kg** · consumption *Warning*

| Komponen | Qty | Operation |
|---|---|---|
| Tempe Iris | 10 kg | Penggorengan |
| Tepung Bumbu | 2 kg | Penggorengan |
| Minyak Goreng | 3 L | Penggorengan |
| Bumbu Keripik | 0,3 kg | Pembumbuan |

Operasi: **Penggorengan** · WC-GORENG-1 · 60 menit; **Pembumbuan** · WC-BUMBU · 20 menit.
By-products: Minyak Bekas 1,5 L (Penggorengan, cost share 0%); Remahan Keripik 0,3 kg (Pembumbuan, 0%).

**BoM 3 — Keripik Tempe 200 g** · qty **30 Units** · consumption *Warning*

| Komponen | Qty | Operation |
|---|---|---|
| Keripik Siap Kemas | 6 kg | Pengemasan |
| Plastik Kemasan 200 g | 30 Units | Pengemasan |

Operasi: **Pengemasan** · WC-KEMAS · 40 menit. By-product: Remahan 0,1 kg (0%).

> 6 kg / 30 bungkus = 200 g per bungkus. Di dunia nyata ada sedikit overfill, yang nanti terlihat di laporan susut.

> Dependensi operasi: di BoM 2, aktifkan *Operation Dependencies* (⚠️ cek nama di 19) dan atur Pembumbuan "Blocked by" Penggorengan.

### 4.9 Reorder rule
*Inventory → Operations → Replenishment* (atau *Configuration → Reordering Rules*):

| Produk | Lokasi | Min | Max | Trigger |
|---|---|---|---|---|
| Tempe Mentah | WH/Stock | 20 | 60 | Auto |
| Minyak Goreng | WH/Stock | 20 | 60 | Auto |
| Tepung Bumbu | WH/Stock | 10 | 40 | Auto |
| Bumbu Keripik | WH/Stock | 2 | 10 | Auto |
| Plastik Kemasan | WH/Stock | 200 | 1000 | Auto |

### 4.10 Maintenance
*Maintenance → Configuration → Maintenance Teams*: "Tim Teknik" (anggota: user Teknisi).
*Maintenance → Equipment*: Penggoreng 1, Penggoreng 2, Mesin Pengiris, Mesin Kemas, Timbangan SCL-01. Untuk Penggoreng: preventive tiap **30 hari** (⚠️ cara set recurrence di 19: field *Preventive Maintenance Frequency* atau *recurring* pada request); Timbangan 90 hari.

### 4.11 Website & Live Chat
1. *Website → Go to website* → pilih tema → publish produk Keripik Tempe 200 g, isi deskripsi & foto.
2. *Website → Configuration → Payment Providers* → aktifkan **Demo** (mode test) atau **Wire Transfer**.
3. *Live Chat → Channels* → "Keripik Support" → operator: Admin Sales → Channel Rules: tampil di semua halaman.
4. *Live Chat → Configuration → Chatbots* → buat "Asisten Keripik":

| # | Tipe step | Pesan | Jawaban |
|---|---|---|---|
| 1 | Question with options | "Halo! Ada yang bisa kami bantu?" | Harga & minimal order / Lama produksi / Bicara dengan admin |
| 2 | Text (jika Harga) | "Keripik tempe 200 g Rp15.000, minimal 10 bungkus." | – |
| 3 | Text (jika Lama produksi) | "Produksi 1–2 hari kerja setelah pesanan dikonfirmasi." | – |
| 4 | Forward to operator (jika admin) | "Menghubungkan ke admin…" | – |
| 5 | Question (email) | "Boleh minta email Anda untuk follow up?" | – |

Pasang chatbot di Channel Rule (`Chatbot = Asisten Keripik`).

### 4.12 SMTP
*Settings → Technical → Outgoing Mail Servers* (aktifkan Developer Mode: *Settings → Activate developer mode*) → baru:
- Host `mailpit`, Port `1025`, Security *None*, Authentication *Username* kosong.
- **Test Connection** → sukses.

(Konfigurasi `smtp_server` di `odoo.conf` sudah menjadi fallback.)

### 4.13 Pengguna & karyawan uji

| Login | Nama | Grup (sementara, grup custom di F2) |
|---|---|---|
| `sales` | Rina (Admin Sales) | Sales: User; Live Chat: User; Invoicing: Billing |
| `gudang` | Joko (Admin Gudang) | Inventory: User |
| `purchase` | Dewi (Purchasing) | Purchase: User |
| `ppic` | Budi (PPIC) | Manufacturing: Administrator; Inventory: User |
| `operator1` | Sari (Operator) | Manufacturing: User |
| `qc` | Andi (QC) | Manufacturing: User (sementara) |
| `teknisi` | Eko (Teknisi) | Maintenance |
| `owner` | Pak Hadi (Pemilik) | Internal user, read-only |

Buat juga *Employees* yang terhubung ke user tersebut (dipakai Telegram di F6).

---

## 5. Uji Alur Make-to-Order Manual

> Simpan hasilnya (screenshot) di `docs/screenshots/f1-*.png`.

1. **Stok awal bahan** — *Inventory → Physical Inventory* atau buat PO lalu terima:
   - Buat PO ke CV Tempe Sejahtera: Tempe Mentah 30 kg → Confirm → Receive → isi lot `TM-20261006-A` → Validate.
   - Ulangi untuk Tepung 10 kg (`TB-…`), Minyak 20 L (`MG-…`), Bumbu 2 kg (`BK-…`), Plastik 100 (tanpa lot).
   - Cek AC-INV-02/03/05.
2. **Pesan dari website** (jendela incognito) — buka `/shop` → Keripik Tempe 200 g qty **30** → checkout sebagai pelanggan baru → bayar Demo.
3. **SO** — login `sales` → SO terkonfirmasi (atau Confirm) → smart button **Manufacturing** = 1 → cek AC-SAL-02.
4. **MO berantai** — login `ppic` → *Manufacturing → Manufacturing Orders* → harus ada 3 MO (Keripik 200 g, Keripik Siap Kemas, Tempe Iris) → AC-SAL-03.
5. **Kerjakan MO Tempe Iris** — buka MO → tab Work Orders → *Start* → *Done* → isi lot `TI-…` → *Produce All*.
6. **Kerjakan MO Keripik Siap Kemas** — Penggorengan → Pembumbuan (cek Pembumbuan terblokir dulu: AC-MRP-02) → ubah konsumsi minyak aktual 3,3 L (untuk bahan susut) → lot `KS-…` → done → cek by-product (AC-MRP-03).
7. **Kerjakan MO Keripik 200 g** → lot `KK-…` → done.
8. **Delivery** — SO → Delivery → status Ready → Validate.
9. **Invoice** — SO → Create Invoice → Confirm → Register Payment (AC-INVC-01/02).
10. **Email** — buka Mailpit, pastikan email konfirmasi SO & invoice ada.
11. **Reorder** — *Inventory → Operations → Replenishment* → *Run Scheduler* (atau *Order Once*) → RFQ dibuat untuk bahan di bawah min (AC-INV-01).
12. **Traceability standar** — buka lot `KK-…` → *Traceability* → lihat hubungan ke `KS-…`, `TI-…`, `TM-…` (ini baseline sebelum laporan custom F3).
13. **Backup**: `make backup` → simpan file `.dump`.

Restore jika perlu:
```bash
docker compose exec -T db dropdb -U odoo keripik
docker compose exec -T db createdb -U odoo keripik
docker compose exec -T db pg_restore -U odoo -d keripik < backup-keripik-XXXX.dump
# filestore (attachments) is in the odoo-web-data volume; for full backup use the web database manager "Backup" (zip)
```

---

## 6. Error yang Sering Terjadi

| Gejala | Penyebab | Solusi |
|---|---|---|
| `could not translate host name "db"` | Odoo jalan sebelum DB siap / nama service salah | `depends_on` + healthcheck; cek `db_host = db` |
| `password authentication failed for user "odoo"` | Password `.env` ≠ `odoo.conf`, atau volume DB lama dengan password lama | Samakan; atau `docker compose down -v` (⚠️ hapus data) |
| `Address already in use` saat `make update` | Proses kedua membuka port 8069 | Tambah `--no-http` |
| Modul custom tidak muncul di Apps | `addons_path` salah / belum *Update Apps List* | Cek mount `/mnt/extra-addons`; Apps → Update Apps List (developer mode) |
| SO confirm tapi MO tidak dibuat | Route MTO belum aktif/belum dicentang di produk, atau BoM tidak ada | 4.4 & 4.6; cek *Replenishment* exceptions |
| MO turunan tidak dibuat | Produk setengah jadi tidak punya route MTO + Manufacture | Centang kedua route |
| Lot tidak diminta saat receipt | Tracking produk belum *By Lots* | 4.6 |
| Email tidak muncul di Mailpit | Outgoing server belum di-set / antrean belum diproses | *Technical → Emails* → *Send Now*; cron "Mail: Email Queue Manager" |
| `Cannot connect to the Docker daemon` | Docker Desktop belum dibuka | `open -a Docker`, tunggu sampai running |
| Odoo lambat / container restart sendiri di Mac | RAM Docker Desktop kecil | Settings → Resources → Memory ≥ 6 GB |
| `Bind for 0.0.0.0:8069 failed: port is already allocated` | Port dipakai proses lain | `lsof -i :8069`, hentikan proses atau ubah `ODOO_PORT` di `.env` |
| `odoo scaffold` membuat folder yang tidak bisa diedit dari Mac | Owner file dari container | `sudo chown -R $(whoami) odoo/addons/mrp_keripik` |
| Live chat tidak tampil | Channel rule belum diatur / website belum dipublikasi | 4.11 |

---

## 7. Pertanyaan Interview

1. **Apa beda MO dan WO di Odoo, dan padanannya di ERPNext?** MO = rencana produksi satu produk (≈ Work Order ERPNext). WO = pekerjaan per operasi di work center (≈ Job Card).
2. **Bagaimana MTO bekerja di Odoo?** Lewat route & stock rule: SO line memicu procurement; rule "Manufacture" dengan MTO membuat MO yang terhubung ke delivery (`move_dest_ids`). Untuk BoM multi-level, komponen yang juga MTO memicu MO turunan.
3. **Kenapa FEFO?** Bahan pangan cepat basi; yang kedaluwarsa duluan dikeluarkan duluan. FIFO berdasar tanggal masuk, bukan tanggal kedaluwarsa.
4. **Apa itu `consumption = warning`?** Mengizinkan konsumsi berbeda dari BoM tapi menampilkan peringatan. Strict memblokir, flexible diam. Kita pakai warning karena susut nyata perlu dicatat.
5. **Kenapa by-product cost share 0%?** Minyak bekas & remahan bernilai kecil; semua biaya dibebankan ke produk utama agar HPP konservatif.
6. **Kenapa Docker untuk Odoo?** Reproducible, versi terkunci, reviewer bisa menjalankan dengan satu perintah, mudah memisahkan DB.
7. **`workers = 0` vs `workers > 0`?** 0 = threaded (dev, satu proses). >0 = prefork multi-proses + gevent untuk websocket di port terpisah (8072) → butuh konfigurasi Nginx `/websocket` (F7).
