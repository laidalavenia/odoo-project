# D1. Gambaran Aplikasi — Sistem Produksi Pabrik Keripik

> Dokumen ini adalah pegangan untuk semua bagian berikutnya. Saat interview, bagian inilah yang menjawab **"kenapa proyek ini ada dan kenapa dirancang begini"**.

---

## 1. Masalah yang Diselesaikan

| # | Masalah | Dampak |
|---|---|---|
| P1 | Pesanan dicatat di WhatsApp/buku, produksi tidak tahu prioritas | Pesanan telat, produksi berlebih/kurang |
| P2 | Tidak ada catatan suhu minyak dan berat kemasan | Kualitas tidak konsisten (gosong, berat < 200 g), risiko komplain dan masalah regulasi |
| P3 | Susut bahan (tempe → keripik) tidak terukur | Harga pokok tidak akurat, kebocoran bahan tidak terdeteksi |
| P4 | Tidak ada lot & tanggal kedaluwarsa | Bahan basi terpakai; saat komplain tidak bisa recall per batch |
| P5 | Pemilik harus datang ke pabrik untuk tahu kondisi produksi | Keputusan lambat |
| P6 | Odoo Enterprise (Quality, Shop Floor, IoT) mahal untuk UMKM | Fitur penting tidak terpakai |

## 2. Solusi

- **Odoo 19 Community** = *system of record*: pesanan, MO, stok, lot, QC, biaya, laporan, notifikasi.
- **Custom module `mrp_keripik`** mengisi celah fitur Enterprise (Quality, Shop Floor, analisis biaya).
- **Go Machine Data Gateway** menangani data sensor berfrekuensi tinggi di database terpisah (`sensor_db`), dan hanya mengirim **ringkasan & kejadian penting** ke Odoo.
- **Next.js Monitor Produksi** + **bot Telegram** untuk pemilik dan tim internal.

> 💡 **Kalimat untuk interview:** *"Odoo menyimpan fakta bisnis — apa yang diproduksi, oleh siapa, lulus atau tidak. Go menyimpan telemetri — apa yang dibaca sensor tiap 2 detik. Keduanya terhubung lewat ringkasan dan event, bukan data mentah."*

---

## 3. Aktor & Hak Akses

Padanan di Frappe:

| Frappe | Odoo |
|---|---|
| Role | Group (`res.groups`) |
| Role Permission Manager (read/write/create/delete per DocType) | `ir.model.access.csv` (CRUD per model) |
| User Permission / permission query conditions | `ir.rule` (domain per record) |
| Disetel lewat UI / fixtures | Ditulis sebagai **data di dalam module** (XML/CSV), ikut ter-install |

| Aktor | Group Odoo (standar / **custom**) | Hak akses utama |
|---|---|---|
| Pelanggan | Portal / Public | Pesan di website, live chat, lihat pesanan sendiri di portal |
| Admin Sales | Sales / User, Live Chat / User | Buat & konfirmasi SO, balas chat, buat invoice |
| Admin Gudang | Inventory / User | Receipt, lot, tanggal kedaluwarsa, transfer |
| Purchasing | Purchase / User | RFQ/PO, vendor, aturan reorder |
| PPIC / Kepala Produksi | Manufacturing / Administrator + **Keripik / Production Manager** | CRUD MO, BoM, work center, semua laporan, terima Telegram |
| Operator Produksi | Manufacturing / User + **Keripik / Operator** | Layar operator: mulai/jeda/selesai WO, input hasil & scrap, isi QC. Tidak bisa hapus MO |
| Petugas QC | **Keripik / Quality User** | CRUD quality check, baca MO/WO, terima Telegram |
| Teknisi | Maintenance (grup standar)* | Kerjakan maintenance request |
| Pemilik Usaha | **Keripik / Owner** (read-only) | Next.js Monitor, Telegram `/produksi`, baca laporan |
| Go Gateway (sistem) | User teknis `gateway_bot` + API token scope `gateway` | Hanya endpoint REST tertentu |
| Next.js Monitor (sistem) | User teknis `monitor_bot` + API token scope `monitor` | Hanya endpoint baca |

\* Cek nama grup Maintenance di Odoo 19 (`addons/maintenance/security/maintenance.xml` di branch `19.0`).

---

## 4. Pembagian Tanggung Jawab Odoo vs Go

| Aspek | Odoo | Go Gateway |
|---|---|---|
| Jenis data | Transaksi bisnis (SO, MO, WO, lot, QC) | Telemetri mentah (suhu, berat) |
| Frekuensi tulis | Rendah: puluhan–ratusan record/hari | Tinggi: ribuan–puluhan ribu baris/hari |
| Database | `odoo` | `sensor_db` (PostgreSQL terpisah) |
| Logika | Aturan bisnis, workflow, security, laporan | Ingest, agregasi, deteksi anomali real-time, streaming |
| Keluar | Email pelanggan, Telegram (event bisnis) | Telegram (anomali, harus cepat), SSE ke Next.js, bot long-polling |

### Kenapa data sensor tidak ditulis ke Odoo? (pertanyaan interview #1)

Hitungan kasar: 2 penggoreng × 1 data/2 detik × 8 jam = **14.400 baris/hari** suhu, plus ±1.500 berat kemasan/hari → **±450 ribu baris/bulan**. Kecil untuk PostgreSQL, tapi **mahal kalau lewat ORM Odoo**:

1. **Setiap write ORM itu berat**: HTTP → auth → access rights & record rules → compute fields → (tracking chatter) → commit. Puluhan ms per record, vs. mikrodetik per baris untuk `INSERT` batch di Go.
2. **Worker Odoo terbatas** (prefork, beberapa worker). Banjir request sensor membuat layar operator & website lambat.
3. **DB bisnis jadi gemuk** → backup, restore, upgrade module, dan migrasi ke Odoo 20 lebih lambat.
4. **Kebutuhan beda**: telemetri butuh query time-series, retensi (hapus > 90 hari), streaming live. Odoo dirancang untuk transaksi.
5. **Isolasi kegagalan**: Odoo restart/upgrade → Go tetap menampung data dan menyinkronkan ringkasan nanti (retry/outbox).

Padanan Frappe: kamu juga tidak akan membuat DocType "Sensor Reading" lalu insert tiap 2 detik, karena setiap insert membawa hooks, versioning, dan permission check.

---

## 5. Daftar Fitur & Prioritas

**MVP** = wajib agar portofolio layak tampil (sampai F7). **Tambahan** = memperkuat profil Backend & Integration (F9–F10).

| ID | Fitur | Prioritas | Fase |
|---|---|---|---|
| FR-SAL | Sales + eCommerce + make-to-order | MVP | F1 |
| FR-MRP | BoM multi-level, work center, operasi, by-product | MVP | F1 |
| FR-INV / FR-PUR | Inventory (lokasi, lot, expiry), Purchase + reorder | MVP | F1 |
| FR-INVC | Invoicing, SMTP (Mailpit) | MVP | F1 |
| FR-SAL-LC | Live Chat + chatbot | Tambahan (mudah) | F1 |
| FR-MNT | Maintenance penggoreng | Tambahan (mudah) | F1 |
| FR-QC | Quality check + blokir MO | **MVP (inti)** | F2 |
| FR-YLD | Laporan yield & susut | **MVP (inti)** | F3 |
| FR-TRC | Traceability lot + simulasi recall | MVP | F3 |
| FR-OPS | Layar operator OWL | **MVP (inti)** | F4 |
| FR-API | REST controller + token | MVP | F5 |
| FR-CST | Biaya per batch | MVP | F5 |
| FR-EXP | Cron kedaluwarsa | MVP | F5 |
| FR-EML | Mail template pelanggan | MVP | F5 |
| FR-TG | Telegram dari Odoo + penautan akun | MVP | F6 |
| FR-MON | Next.js Monitor (data Odoo) | MVP | F7 |
| FR-GW | Go gateway: ingest, sensor_db, ringkasan | Tambahan (prioritas tinggi) | F9 |
| FR-GW-ANM | Deteksi anomali + QC otomatis + Telegram | Tambahan (prioritas tinggi) | F9 |
| FR-GW-BOT | Bot long polling `/start` & `/produksi` | Tambahan | F9 |
| FR-SIM | Simulator mesin | Tambahan (wajib jika ada gateway) | F9 |
| FR-MON-LIVE | Streaming live ke Next.js | Tambahan | F10 |
| FR-MQTT | MQTT + Mosquitto | Opsional | setelah F10 |

⚠️ **Keputusan desain sejak awal:** long polling Telegram (`getUpdates`) hanya boleh dijalankan **satu** konsumen per token, dan tidak bisa bersamaan dengan webhook. Di F6 (belum ada Go), penautan `/start <kode>` dijalankan oleh **cron Odoo**. Di F9 tanggung jawab ini **dipindah ke Go** (diatur lewat parameter sistem `mrp_keripik.telegram_poller = odoo|gateway`).

---

## 6. Batasan Community vs Enterprise

> Berdasarkan pengetahuanku tentang Odoo 17–18 dan perkiraan untuk 19. **Verifikasi** di <https://www.odoo.com/page/editions> dan di repo `odoo/odoo` branch `19.0` (kalau module ada di repo itu = Community).

| Fitur | Community? | Di proyek ini |
|---|---|---|
| MRP: MO, BoM, work center, work order, by-product, unbuild | ✅ | Dipakai |
| Shop Floor / tablet view (`mrp_workorder`) | ❌ Enterprise | → **Layar operator OWL** (FR-OPS) |
| Quality (`quality_control`) | ❌ Enterprise | → **QC custom** (FR-QC); cek OCA `quality_control_oca` |
| MPS, PLM | ❌ Enterprise | Di luar cakupan |
| Lot & expiry (`product_expiry`) | ✅ | Dipakai + cron custom |
| Traceability report standar | ✅ (dasar) | Custom: laporan recall QWeb dua arah |
| IoT Box | ❌ Enterprise + hardware | → **Go gateway** |
| Accounting penuh / MO cost analysis | ❌ (Community = Invoicing) | Biaya per batch dihitung sendiri |
| Maintenance | ✅ (cek di 19) | Dipakai |
| Website, eCommerce, Live Chat, chatbot | ✅ (chatbot: cek di 19) | Dipakai |
| Barcode, Studio | ❌ Enterprise | Tidak dipakai; semua via kode |

### Cek OCA sebelum membangun

Buka branch **`19.0`** di repo berikut dan catat module yang sudah dimigrasi:

- **OCA/manufacture**: `quality_control_oca`, `mrp_production_*`
- **OCA/stock-logistics-workflow**, **OCA/product-attribute**: lot & expiry
- **OCA/social**: `mail_gateway_telegram`
- **OCA/rest-framework**: `fastapi`, `base_rest` (alternatif controller manual)

Cara cepat: `https://github.com/OCA/manufacture/tree/19.0` atau <https://odoo-community.org/shop> (filter versi 19).

**Rekomendasi:** walau `quality_control_oca` ada, **bangun QC sendiri yang ramping** karena kamu harus bisa menjelaskan model, logika, dan security-nya. Tulis di README: *"Saya mengevaluasi `quality_control_oca`; memilih custom karena butuh integrasi WO, blokir MO, dan input otomatis dari gateway."*

---

## 7. Kamus Padanan ERPNext ↔ Odoo

| ERPNext / Frappe | Odoo 19 |
|---|---|
| DocType | Model (`models.Model`) + view XML |
| BOM | `mrp.bom` (+ `mrp.bom.line`, `mrp.bom.byproduct`) |
| Routing / Operation | Operasi di BoM (`mrp.routing.workcenter`) |
| Workstation | `mrp.workcenter` |
| **Work Order** | **`mrp.production` (MO)** |
| **Job Card** | **`mrp.workorder` (WO)** |
| Job Card Time Log | `mrp.workcenter.productivity` |
| Quality Inspection (+ Template) | custom `keripik.quality.check` (+ `keripik.quality.point`) |
| Batch | `stock.lot` |
| Stock Entry / Stock Ledger | `stock.picking` / `stock.move` / `stock.move.line` |
| Scheduler (`hooks.py` scheduler_events) | `ir.cron` |
| Email Template / Notification | `mail.template` + `mail.thread` |
| `@frappe.whitelist` | `http.Controller` + `@http.route` |
| Fixtures | data XML/CSV di module |
| Custom Field via Customize Form | `_inherit` + field baru di Python |

> Jebakan paling umum: **Work Order ERPNext = MO Odoo; Job Card ERPNext = Work Order Odoo.**

---

## 8. Tugas Verifikasi (±1 jam)

- [ ] Repo `odoo/odoo` branch `19.0`: ada `addons/maintenance`, `addons/im_livechat`, `addons/product_expiry`, `addons/mrp`?
- [ ] OCA/manufacture branch 19.0: ada `quality_control_oca`?
- [ ] <https://github.com/gofiber/fiber/releases>: versi stabil terbaru v3? (dokumen ini memakai v3; lihat D5)
- [ ] Apakah Odoo 20 sudah rilis? (biasanya diumumkan di Odoo Experience sekitar Sept/Okt). Jika sudah, **tetap pakai 19** untuk proyek ini (OCA & stabilitas), migrasi di F12.
