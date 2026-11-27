# D7. Functional Specification Document (FSD)
## Sistem Produksi Pabrik Keripik — Odoo 19 Community + Machine Data Gateway

| Atribut | Nilai |
|---|---|
| Versi dokumen | 1.0 |
| Tanggal | 5 Oktober 2026 |
| Penulis | Laida L |
| Status | Draft untuk review |

### Riwayat Revisi

| Versi | Tanggal | Perubahan |
|---|---|---|
| 1.0 | 2026-10-05 | Versi awal, mencakup seluruh cakupan |

---

## 1. Pendahuluan

### 1.1 Tujuan Dokumen
Dokumen ini mendefinisikan kebutuhan fungsional dan non-fungsional untuk sistem produksi pabrik keripik tempe skala UMKM. Dokumen ini menjadi acuan implementasi (F1–F10), pengujian (acceptance criteria), dan bahan penjelasan teknis saat interview.

### 1.2 Ruang Lingkup
Termasuk:
- Konfigurasi standar Odoo 19 Community: Sales, Website/eCommerce, Live Chat, Manufacturing, Inventory, Purchase, Maintenance, Invoicing, email keluar.
- Custom module `mrp_keripik`: quality check, laporan yield & susut, traceability lot, layar operator OWL, biaya per batch, scheduled action kedaluwarsa, notifikasi email & Telegram, REST API.
- Go Machine Data Gateway, simulator mesin, dan aplikasi Next.js Monitor Produksi.

Di luar cakupan: lihat §5.3.

### 1.3 Definisi & Singkatan

| Istilah | Arti |
|---|---|
| SO | Sales Order (`sale.order`) |
| PO / RFQ | Purchase Order / Request for Quotation (`purchase.order`) |
| MO | Manufacturing Order (`mrp.production`), setara *Work Order* di ERPNext |
| WO | Work Order (`mrp.workorder`), setara *Job Card* di ERPNext |
| BoM | Bill of Materials (`mrp.bom`) |
| Work center | Stasiun/mesin kerja (`mrp.workcenter`), setara *Workstation* |
| Operasi | Langkah pada BoM (`mrp.routing.workcenter`), setara *Operation* di Routing |
| QC | Quality check (`keripik.quality.check`) |
| Quality point | Titik pemeriksaan (`keripik.quality.point`) |
| MTO | Make-to-Order: produksi dipicu pesanan |
| Lot | Batch bahan/produk (`stock.lot`) |
| Yield | Hasil aktual ÷ hasil rencana × 100% |
| Susut | Selisih konsumsi aktual dan rencana bahan |
| Gateway | Go Machine Data Gateway |
| sensor_db | Database PostgreSQL milik gateway |
| SSE | Server-Sent Events |
| Outbox | Tabel antrean event keluar dengan retry |
| PPIC | Production Planning & Inventory Control |
| chat_id | ID chat Telegram penerima pesan |
| AC | Acceptance criteria |

### 1.4 Referensi
- D1–D6 (dokumen desain di folder ini)
- Dokumentasi Odoo 19: <https://www.odoo.com/documentation/19.0/>
- Source Odoo 19: <https://github.com/odoo/odoo/tree/19.0>
- OCA: <https://github.com/OCA>
- Telegram Bot API: <https://core.telegram.org/bots/api>
- Fiber: <https://docs.gofiber.io>
- Next.js: <https://nextjs.org/docs>

> **Catatan versi:** nama teknis field **standar** Odoo di dokumen ini berdasarkan Odoo 17/18. Field yang ditandai ⚠️ perlu diverifikasi di source 19.0 saat implementasi.

---

## 2. Gambaran Sistem

### 2.1 Konteks Bisnis
Pabrik membeli tempe mentah, tepung bumbu, minyak, bumbu, dan plastik kemasan, lalu memproduksi **Keripik Tempe 200 g** melalui 3 level BoM dan 4 operasi (Pengirisan → Penggorengan → Pembumbuan → Pengemasan). Produksi dipicu pesanan (MTO) dari toko online atau Admin Sales. Kualitas dikendalikan lewat suhu minyak, kadar air, dan berat kemasan. Data mesin dikumpulkan gateway Go.

```mermaid
flowchart LR
    V[Vendor] -->|bahan baku| P[Pabrik Keripik]
    C[Pelanggan] -->|pesanan web / chat| P
    P -->|keripik 200 g| C
    P -->|email| C
    P <-->|Telegram| T[Tim internal & Pemilik]
    M[Mesin penggoreng & timbangan] -->|sensor| G[Gateway Go] --> P
```

### 2.2 Aktor
Lihat D1 §3. Ringkas: Pelanggan, Admin Sales, Admin Gudang, Purchasing, PPIC, Operator, Petugas QC, Teknisi, Pemilik, Go Gateway (sistem), Next.js Monitor (sistem), Odoo Scheduler (sistem), Telegram Bot API, SMTP.

### 2.3 Grup Odoo Custom

| XML ID | Nama | Implied (mewarisi) |
|---|---|---|
| `mrp_keripik.group_keripik_operator` | Keripik / Operator | `mrp.group_mrp_user` |
| `mrp_keripik.group_keripik_quality` | Keripik / Quality User | `base.group_user` |
| `mrp_keripik.group_keripik_manager` | Keripik / Production Manager | operator + quality, `mrp.group_mrp_manager` |
| `mrp_keripik.group_keripik_owner` | Keripik / Owner (read-only) | `base.group_user` |

> ⚠️ Odoo 19 kemungkinan mengubah cara grup dikategorikan (setahuku ada konsep *privilege* baru menggantikan `category_id`). Ikuti pola di `addons/mrp/security/mrp_security.xml` branch 19.0.

### 2.4 Matriks Hak Akses (C = create, R = read, U = update/write, D = delete)

| Model | Operator | Quality | Manager (PPIC) | Owner | Gateway (user teknis) | Monitor (user teknis) | Admin Gudang | Admin Sales |
|---|---|---|---|---|---|---|---|---|
| `keripik.quality.point` | R | R | CRUD | R | R | R | – | – |
| `keripik.quality.check` | RU¹ | CRU | CRUD | R | CRU | R | – | – |
| `keripik.sensor.summary` | R | R | RUD | R | CR | R | – | – |
| `keripik.api.token` | – | – | – (admin saja) | – | – | – | – | – |
| `keripik.gateway.event` | C² | – | R | – | – | – | – | – |
| `keripik.telegram.log` | – | – | R | R | – | – | – | – |
| `keripik.yield.report` | – | R | R | R | – | R | – | – |
| `mrp.production` | RU | R | CRUD | R | R | R | R | R |
| `mrp.workorder` | RU | R | CRUD | R | R | R | – | – |
| `mrp.bom` | R | R | CRUD | R | R | – | R | – |
| `stock.lot` | R | R | CRU | R | – | – | CRU | R |
| `hr.employee` (field Telegram) | RU³ | RU³ | RU | RU³ | U⁴ | – | RU³ | RU³ |

¹ Hanya QC untuk WO di work center yang sedang ia kerjakan (record rule); tidak bisa mengubah QC `source = gateway`.
² Dibuat lewat method server dengan `sudo()`, bukan akses langsung.
³ Hanya record karyawan miliknya sendiri (generate kode, ubah preferensi).
⁴ Lewat endpoint `/telegram/link` yang menjalankan `sudo()` terbatas.

**Record rules:**

| Rule | Model | Grup | Domain |
|---|---|---|---|
| Operator: QC milik WO aktif | `keripik.quality.check` | Operator | `[('workorder_id.workcenter_id', 'in', user.employee_ids.keripik_workcenter_ids.ids)]` (opsional, bila work center ditetapkan per operator) |
| Multi-company | semua model custom | semua | `[('company_id', 'in', company_ids)]` |

---

## 3. Kebutuhan Fungsional

> Format setiap fitur: ID & deskripsi → Aktor & hak akses → Prasyarat & pemicu → Alur utama / alternatif / error → Aturan bisnis → Spesifikasi field → State → Catatan tampilan → Acceptance criteria.

---

### 3(a) Penjualan, eCommerce & Live Chat

#### FR-SAL-01 — Katalog produk di website
Produk **Keripik Tempe 200 g** tampil di toko online dengan harga, foto, deskripsi, dan stok tidak dibatasi (karena MTO).

#### FR-SAL-02 — Checkout pelanggan
Pelanggan dapat menambah ke keranjang, checkout dengan alamat pengiriman, dan memilih metode pembayaran (provider demo/"Wire Transfer").

#### FR-SAL-03 — Konfirmasi SO memicu MO (MTO)
Saat SO dikonfirmasi, sistem membuat delivery order dan MO berantai untuk setiap level BoM.

#### FR-SAL-04 — Live Chat dengan chatbot berbasis skrip
Pengunjung website disapa chatbot yang menjawab FAQ (harga, minimal order, lama produksi, ongkir). Pilihan "Bicara dengan admin" meneruskan ke Admin Sales.

#### FR-SAL-05 — Portal pelanggan
Pelanggan dapat melihat status pesanan dan invoice di portal `/my/orders`.

**Aktor & hak akses:** Pelanggan (public/portal), Admin Sales (Sales User, Live Chat User).

**Prasyarat:** module `website_sale`, `im_livechat`, `website_livechat` (⚠️ cek nama module di 19), `sale_mrp`, `mrp` ter-install; route MTO aktif; produk dipublikasikan.

**Pemicu:** pelanggan membuka website / checkout; Admin Sales klik *Confirm*.

**Alur utama (checkout → MO):**
1. Pelanggan membuka `/shop`, memilih Keripik Tempe 200 g, qty 30.
2. Klik *Add to Cart* → *Checkout* → isi alamat → pilih metode pembayaran → *Pay Now*.
3. Odoo membuat SO (state `sent`/`sale` tergantung provider).
4. Jika belum confirmed: Admin Sales membuka SO → *Confirm*.
5. Sistem membuat delivery order (state `waiting`) dan MO level 1 (Keripik 200 g) → MO level 2 (Keripik Siap Kemas) → MO level 3 (Tempe Iris).
6. Email konfirmasi terkirim ke pelanggan (FR-EML-01).

**Alur alternatif:**
- A1: SO dibuat manual oleh Admin Sales dari backend (pesanan via chat) → langkah 4–6 sama.
- A2: Stok produk jadi tersedia (misal sisa produksi) → MO tetap dibuat karena route MTO (aturan bisnis BR-SAL-01).

**Alur error:**
- E1: Produk tidak punya BoM → Odoo menampilkan exception pada procurement ("No rule/BoM found"). Admin perlu melengkapi BoM, lalu *Retry* dari Inventory → Operations → Replenishment/Exceptions (⚠️ lokasi menu cek di 19).
- E2: Alamat pelanggan tidak lengkap → validasi checkout standar Odoo.

**Aturan bisnis:**
- BR-SAL-01: Produk jadi dan setengah jadi memakai route **Manufacture + Replenish on Order (MTO)**, sehingga setiap SO line memicu MO.
- BR-SAL-02: Minimal order 10 pcs (diinformasikan chatbot; validasi opsional di checkout).
- BR-SAL-03: Live chat aktif jam 08.00–17.00; di luar jam itu chatbot menyarankan meninggalkan pesan.

**Spesifikasi field (konfigurasi standar):**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `product.template.route_ids` | Routes | Many2many | ✔ | – | harus berisi Manufacture & MTO | Produk jadi & setengah jadi |
| `product.template.is_published` | Published | Boolean | – | False | – | True untuk Keripik 200 g |
| `product.template.list_price` | Sales Price | Float | ✔ | – | > 0 | Rp 15.000 |
| `product.template.sale_ok` | Can be Sold | Boolean | – | True | – | False untuk bahan baku |
| `product.template.allow_out_of_stock_order` ⚠️ | Continue selling when out of stock | Boolean | – | – | – | True (MTO) |
| `sale.order.state` | Status | Selection | ✔ | `draft` | – | lihat state |
| `im_livechat.channel.name` | Channel | Char | ✔ | – | – | "Keripik Support" |
| `chatbot.script.title` ⚠️ | Chatbot | Char | ✔ | – | – | "Asisten Keripik" |

**State SO (standar):**
```mermaid
stateDiagram-v2
    [*] --> draft: Quotation
    draft --> sent: Send by email / checkout
    sent --> sale: Confirm / payment
    draft --> sale: Confirm
    sale --> cancel: Cancel
    draft --> cancel
    sent --> cancel
    cancel --> draft: Set to quotation
    sale --> [*]
```
> ⚠️ Status `done` (Locked) di versi lama diganti flag `locked` sejak Odoo 17.

**Catatan tampilan:**
- Halaman produk menampilkan teks "Diproduksi setelah pesanan, estimasi 2 hari kerja".
- SO form: smart button **Manufacturing** (dari `sale_mrp`) menunjukkan MO terkait.
- Widget live chat di pojok kanan bawah website.

**Acceptance criteria:**
- **AC-SAL-01** Given produk Keripik 200 g dipublikasikan, When pelanggan membuka `/shop`, Then produk tampil dengan harga Rp 15.000.
- **AC-SAL-02** Given pelanggan checkout 30 pcs, When pembayaran selesai atau Admin Sales klik Confirm, Then SO berstatus `sale` dan smart button Manufacturing menunjukkan 1 MO Keripik 200 g qty 30.
- **AC-SAL-03** Given MO Keripik 200 g dibuat, When membuka MO tersebut, Then terdapat MO turunan untuk Keripik Siap Kemas dan Tempe Iris dengan qty sesuai BoM.
- **AC-SAL-04** Given pengunjung membuka website jam kerja, When widget live chat dibuka, Then chatbot menyapa dan menampilkan pilihan FAQ; memilih "Bicara dengan admin" membuat sesi yang muncul di Discuss Admin Sales.
- **AC-SAL-05** Given pelanggan login portal, When membuka `/my/orders`, Then pesanan miliknya tampil dan pesanan orang lain tidak.

---

### 3(b) Manufacturing Standar (BoM, Work Center, MO, Work Order, By-product)

#### FR-MRP-01 — BoM multi-level
Tiga BoM: (1) Tempe Iris ← Tempe Mentah; (2) Keripik Siap Kemas ← Tempe Iris, Tepung Bumbu, Minyak, Bumbu; (3) Keripik Tempe 200 g ← Keripik Siap Kemas, Plastik Kemasan.

#### FR-MRP-02 — Work center dengan kapasitas & biaya per jam
WC-IRIS, WC-GORENG-1, WC-GORENG-2, WC-BUMBU, WC-KEMAS.

#### FR-MRP-03 — Operasi per BoM
Pengirisan (BoM 1), Penggorengan & Pembumbuan (BoM 2), Pengemasan (BoM 3), dengan durasi standar.

#### FR-MRP-04 — By-product
Minyak Bekas dan Remahan dihasilkan pada BoM 2 (operasi Penggorengan) dan Remahan pada BoM 3.

#### FR-MRP-05 — Siklus MO & WO
MO dikonfirmasi → WO dibuat per operasi → WO dikerjakan berurutan → MO diselesaikan → stok produk jadi dan by-product bertambah dengan lot.

#### FR-MRP-06 — Scrap
Operator/PPIC dapat mencatat scrap dari MO/WO (`stock.scrap`).

**Aktor:** PPIC (Manufacturing Admin), Operator (Manufacturing User).

**Prasyarat:** Settings Manufacturing: *Work Orders*, *By-Products*, *Lots & Serial Numbers*, *Expiration Dates* aktif. Produk & UoM tersedia.

**Pemicu:** MO dibuat dari MTO atau manual.

**Alur utama:**
1. PPIC membuka MO (state `confirmed`) → *Check availability* → bahan `assigned`.
2. Operator mengerjakan WO sesuai urutan operasi (detail FR-OPS).
3. PPIC mengisi lot produk jadi (`lot_producing_id`) → *Produce All* / *Mark as Done*.
4. Sistem memvalidasi QC (FR-QC-05), mengonsumsi bahan, dan memasukkan produk jadi + by-product.

**Alur alternatif:**
- A1: Produksi parsial → isi `qty_producing` < `product_qty` → Odoo menawarkan **backorder**.
- A2: MO dibuat manual (produksi untuk stok) tanpa SO.

**Alur error:**
- E1: Bahan tidak cukup → status komponen *Not Available*; PPIC dapat tetap memproduksi (konsumsi negatif tidak diizinkan untuk produk lot tertrack, Odoo minta lot).
- E2: Lot produk jadi belum diisi → Odoo meminta lot sebelum done.
- E3: Konsumsi berbeda dari BoM → Odoo menampilkan wizard *consumption warning* (tergantung setting `consumption` di BoM: `flexible`/`warning`/`strict`).

**Aturan bisnis:**
- BR-MRP-01: BoM memakai `consumption = 'warning'` agar selisih konsumsi tercatat (bahan laporan susut) tetapi tidak memblokir.
- BR-MRP-02: Urutan operasi wajib: WO berikutnya `blocked` sampai WO sebelumnya selesai (⚠️ cek mekanisme dependensi operasi di 19: `allow_operation_dependencies` pada BoM).
- BR-MRP-03: Biaya by-product (`cost_share`) 0%, jadi seluruh biaya dibebankan ke produk utama.
- BR-MRP-04: Produk setengah jadi dan produk jadi wajib lot; tanggal kedaluwarsa otomatis dari `expiration_time` produk.

**Spesifikasi field (konfigurasi standar):**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `mrp.bom.product_tmpl_id` | Product | Many2one | ✔ | – | – | |
| `mrp.bom.product_qty` | Quantity | Float | ✔ | 1 | > 0 | BoM1 10 kg, BoM2 6 kg, BoM3 30 Units |
| `mrp.bom.type` | BoM Type | Selection | ✔ | `normal` | – | `normal` (bukan kit) |
| `mrp.bom.consumption` | Flexible Consumption | Selection | ✔ | `warning` | – | |
| `mrp.bom.line.product_qty` | Quantity | Float | ✔ | – | > 0 | |
| `mrp.bom.line.operation_id` | Consumed in Operation | Many2one | – | – | operasi milik BoM yang sama | Penting untuk yield per operasi |
| `mrp.bom.byproduct.product_qty` | Quantity | Float | ✔ | – | > 0 | |
| `mrp.bom.byproduct.operation_id` | Produced in Operation | Many2one | – | – | | |
| `mrp.bom.byproduct.cost_share` | Cost Share (%) | Float | – | 0 | 0–100 | |
| `mrp.workcenter.code` | Code | Char | – | – | unik (konvensi) | WC-GORENG-1 |
| `mrp.workcenter.costs_hour` | Cost per hour | Float | – | 0 | ≥ 0 | Rp |
| `mrp.workcenter.default_capacity` | Capacity | Float | – | 1 | > 0 | |
| `mrp.workcenter.time_efficiency` | Time Efficiency | Float | – | 100 | > 0 | |
| `mrp.routing.workcenter.time_cycle_manual` | Default Duration (min) | Float | – | 60 | ≥ 0 | |
| `mrp.production.lot_producing_id` | Lot/Serial | Many2one | ✔ saat done | – | – | |

**State MO (standar):**
```mermaid
stateDiagram-v2
    [*] --> draft
    draft --> confirmed: Confirm
    confirmed --> progress: WO pertama dimulai / qty_producing diisi
    progress --> to_close: Semua WO selesai
    to_close --> done: Mark as Done (lolos validasi QC)
    progress --> done: Mark as Done (lolos validasi QC)
    confirmed --> cancel
    progress --> cancel
    done --> [*]
```

**State WO (standar):**
```mermaid
stateDiagram-v2
    [*] --> blocked: menunggu WO sebelumnya
    [*] --> ready
    blocked --> ready: WO sebelumnya done
    ready --> progress: Start
    progress --> ready: Pause (time log ditutup)
    progress --> done: Finish
    ready --> cancel
    done --> [*]
```
> ⚠️ Di Odoo 17+ nama state WO bisa berbeda (`pending`/`waiting`/`ready`/`progress`/`done`/`cancel`). Verifikasi.

**Catatan tampilan:** MO form standar + tambahan dari `mrp_keripik` (tab Quality, biaya, smart button QC). List MO menampilkan badge `qc_state`.

**Acceptance criteria:**
- **AC-MRP-01** Given 3 BoM terkonfigurasi, When MO Keripik 200 g qty 30 dikonfirmasi, Then MO menampilkan komponen Keripik Siap Kemas 6 kg dan Plastik 30 Units, serta 1 WO Pengemasan di WC-KEMAS.
- **AC-MRP-02** Given MO Keripik Siap Kemas, When dikonfirmasi, Then terdapat 2 WO (Penggorengan, Pembumbuan) dan Pembumbuan tidak bisa dimulai sebelum Penggorengan selesai.
- **AC-MRP-03** Given MO Keripik Siap Kemas selesai, When melihat stok, Then Minyak Bekas bertambah 1,5 L dan Remahan 0,3 kg dengan biaya 0.
- **AC-MRP-04** Given qty_producing 20 dari 30, When Mark as Done, Then Odoo menawarkan backorder 10.
- **AC-MRP-05** Given operator mencatat scrap 0,2 kg Tempe Iris, When divalidasi, Then stok berpindah ke lokasi Scrap dan tercatat di MO.

---

### 3(c) Inventory & Purchase

#### FR-INV-01 — Struktur lokasi
Satu warehouse `WH` (Pabrik Keripik) dengan sub-lokasi `WH/Stock/Bahan Baku` dan `WH/Stock/Produk Jadi`, plus putaway rule berdasarkan kategori produk.

#### FR-INV-02 — Lot & tanggal kedaluwarsa
Semua bahan baku pangan, produk setengah jadi, dan produk jadi dilacak per lot dengan `expiration_date`, `use_date`, `removal_date`, `alert_date`.

#### FR-INV-03 — FEFO
Removal strategy **FEFO** (First Expired First Out) pada kategori bahan pangan.

#### FR-PUR-01 — Vendor & harga
Setiap bahan baku punya vendor utama (`product.supplierinfo`) dengan harga dan lead time.

#### FR-PUR-02 — Reorder otomatis
Reorder rule min/max per bahan baku; scheduler membuat RFQ saat forecast < min.

#### FR-PUR-03 — Penerimaan barang
Admin Gudang memvalidasi receipt dengan mengisi lot dan tanggal kedaluwarsa per baris.

**Aktor:** Purchasing (Purchase User), Admin Gudang (Inventory User), Odoo Scheduler.

**Prasyarat:** Settings Inventory: *Storage Locations*, *Lots & Serial Numbers*, *Expiration Dates* aktif; module `purchase_stock`, `product_expiry`.

**Pemicu:** forecast stok turun di bawah min; Purchasing membuat RFQ manual; barang tiba.

**Alur utama (reorder → receipt):** lihat D3(b).
1. Scheduler (cron harian/manual *Run Scheduler*) mengevaluasi `stock.warehouse.orderpoint`.
2. RFQ dibuat ke vendor utama dengan qty = max − forecast (dibulatkan ke `qty_multiple`).
3. Purchasing mengecek → *Confirm Order* → PO + receipt `WH/IN/xxxxx`.
4. Barang tiba → Admin Gudang membuka receipt → isi Lot (misal `TM-20261012-A`) dan Expiration Date → *Validate*.
5. Stok masuk ke `WH/Stock/Bahan Baku` (putaway rule).

**Alur alternatif:**
- A1: Barang datang sebagian → *Create Backorder*.
- A2: Barang ditolak (rusak) → *Return* ke vendor.
- A3: PO manual tanpa reorder rule.

**Alur error:**
- E1: Lot kosong untuk produk ber-lot → Odoo menolak validasi ("You need to supply a Lot/Serial number").
- E2: Expiration date di masa lalu → **custom warning** (opsional, BR-INV-03).

**Aturan bisnis:**
- BR-INV-01: Masa simpan default: Tempe Mentah 3 hari, Tempe Iris 1 hari, Keripik Siap Kemas 60 hari, Keripik 200 g 90 hari, Tepung 180 hari, Minyak 365 hari, Bumbu 180 hari.
- BR-INV-02: `alert_time` (hari sebelum expiry untuk alert): Tempe Mentah 1 hari, lainnya 14 hari.
- BR-INV-03: Receipt dengan expiration date < hari ini menampilkan peringatan (tidak memblokir).
- BR-PUR-01: Reorder rule Tempe Mentah: min 20 kg, max 60 kg; Minyak min 20 L, max 60 L; Tepung min 10 kg, max 40 kg; Plastik min 200, max 1000.

**Spesifikasi field (standar):**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `product.template.tracking` | Tracking | Selection | ✔ | `none` | – | `lot` untuk bahan pangan & produk |
| `product.template.is_storable` ⚠️ | Track Inventory | Boolean | – | – | – | Odoo 18+: pengganti `detailed_type='product'` |
| `product.template.use_expiration_date` | Expiration Date | Boolean | – | False | – | True |
| `product.template.expiration_time` | Expiration Time (days) | Integer | – | – | ≥ 0 | BR-INV-01 |
| `product.template.alert_time` | Alert Time (days) | Integer | – | – | ≥ 0 | BR-INV-02 |
| `product.category.removal_strategy_id` | Force Removal Strategy | Many2one | – | – | – | FEFO |
| `stock.lot.expiration_date` | Expiration Date | Datetime | – | otomatis | – | |
| `stock.lot.alert_date` | Alert Date | Datetime | – | otomatis | – | Dipakai cron FR-EXP |
| `stock.warehouse.orderpoint.product_min_qty` | Min | Float | ✔ | 0 | ≥ 0 | |
| `stock.warehouse.orderpoint.product_max_qty` | Max | Float | ✔ | 0 | ≥ min | |
| `stock.warehouse.orderpoint.trigger` | Trigger | Selection | – | `auto` | – | `auto` |
| `product.supplierinfo.partner_id` | Vendor | Many2one | ✔ | – | – | |
| `product.supplierinfo.price` | Price | Float | ✔ | – | ≥ 0 | |
| `product.supplierinfo.delay` | Delivery Lead Time | Integer | – | 1 | ≥ 0 | hari |

**State PO (standar):**
```mermaid
stateDiagram-v2
    [*] --> draft: RFQ
    draft --> sent: Send RFQ
    sent --> purchase: Confirm Order
    draft --> purchase: Confirm Order
    purchase --> cancel
    draft --> cancel
    purchase --> [*]
```

**State Picking (standar):**
```mermaid
stateDiagram-v2
    [*] --> draft
    draft --> waiting: menunggu operasi lain
    draft --> confirmed: menunggu stok
    waiting --> assigned: barang tersedia
    confirmed --> assigned: Check availability
    assigned --> done: Validate
    draft --> cancel
    assigned --> cancel
    done --> [*]
```

**Catatan tampilan:** di receipt, kolom Lot dan Expiration Date tampil di baris *Detailed Operations*. Menu *Inventory → Reporting → Lots/Serial Numbers* difilter "Expiring soon".

**Acceptance criteria:**
- **AC-INV-01** Given Tempe Mentah stok 15 kg dan min 20 kg, When scheduler dijalankan, Then RFQ ke vendor utama dibuat dengan qty 45 kg.
- **AC-INV-02** Given receipt Tempe Mentah 45 kg, When Admin Gudang memvalidasi tanpa lot, Then Odoo menolak dengan pesan lot wajib.
- **AC-INV-03** Given lot diisi dan tanggal diterima 12/10, When divalidasi, Then lot memiliki expiration date 15/10 dan alert date 14/10.
- **AC-INV-04** Given dua lot Tempe Mentah dengan expiry berbeda, When MO memesan bahan, Then lot dengan expiry paling awal dipesan lebih dulu (FEFO).
- **AC-INV-05** Given barang diterima, When melihat lokasi, Then stok berada di `WH/Stock/Bahan Baku`.

---

### 3(d) Maintenance

#### FR-MNT-01 — Equipment mesin
Equipment: Penggoreng 1, Penggoreng 2, Mesin Iris, Mesin Kemas, Timbangan, masing-masing terhubung ke work center.

#### FR-MNT-02 — Maintenance preventif
Penggoreng dijadwalkan preventif tiap 30 hari (ganti minyak total, bersihkan saringan, cek termostat), Timbangan dikalibrasi tiap 90 hari.

#### FR-MNT-03 — Maintenance korektif
PPIC/Operator dapat membuat maintenance request bila mesin rusak; Teknisi mengerjakan dan menutupnya.

**Aktor:** Teknisi, PPIC (membuat request), Operator (melapor).

**Prasyarat:** module `maintenance` ter-install (`mrp_maintenance` adalah Enterprise, tidak dipakai); tim maintenance "Tim Teknik".

**Pemicu:** jadwal preventif jatuh tempo (cron standar); laporan kerusakan.

**Alur utama:**
1. Odoo membuat request preventif saat jatuh tempo (berdasarkan `period`/recurrence ⚠️ field di 19).
2. Teknisi melihat di kanban → pindah stage *In Progress* → kerjakan → *Repaired*.
3. Durasi tercatat di `duration`.

**Alur alternatif:** A1: request korektif dibuat manual dengan prioritas tinggi.

**Alur error:** E1: request di-*scrap* bila mesin tidak bisa diperbaiki (stage Scrap).

**Aturan bisnis:**
- BR-MNT-01: Saat Penggoreng sedang maintenance (request *In Progress*), PPIC sebaiknya memindahkan WO ke WC-GORENG-2 (alternative workcenter). Pemblokiran otomatis di luar cakupan.
- BR-MNT-02: Semua request terhubung ke equipment dan tim.

**Spesifikasi field (standar):**

| Nama teknis | Label | Tipe | Wajib | Default | Keterangan |
|---|---|---|---|---|---|
| `maintenance.equipment.name` | Equipment Name | Char | ✔ | – | "Penggoreng 1" |
| `maintenance.equipment.maintenance_team_id` | Team | Many2one | – | – | Tim Teknik |
| `maintenance.request.maintenance_type` | Type | Selection | ✔ | `corrective` | `preventive`/`corrective` |
| `maintenance.request.schedule_date` | Scheduled Date | Datetime | – | – | |
| `maintenance.request.stage_id` | Stage | Many2one | ✔ | New | |
| `maintenance.request.duration` | Duration (h) | Float | – | – | |

**State (stage) maintenance request:**
```mermaid
stateDiagram-v2
    [*] --> New
    New --> InProgress: Teknisi mulai
    InProgress --> Repaired: Selesai
    InProgress --> Scrap: Tidak bisa diperbaiki
    Repaired --> [*]
    Scrap --> [*]
```

**Acceptance criteria:**
- **AC-MNT-01** Given Penggoreng 1 dengan preventif tiap 30 hari, When jatuh tempo, Then request preventif baru muncul di kanban Tim Teknik.
- **AC-MNT-02** Given Teknisi memindahkan request ke Repaired, When melihat equipment, Then riwayat maintenance bertambah dan request berikutnya terjadwal.

---

### 3(e) Invoicing

#### FR-INVC-01 — Invoice dari SO
Admin Sales membuat invoice dari SO dengan kebijakan *Delivered quantities*.

#### FR-INVC-02 — Pembayaran manual
Pembayaran dicatat manual (*Register Payment*) di Invoicing Community.

**Aktor:** Admin Sales (Invoicing/Billing ⚠️ nama grup).

**Prasyarat:** module `account` (Invoicing). Chart of account Indonesia (l10n_id) bila tersedia. Pajak PPN opsional.

**Pemicu:** delivery divalidasi.

**Alur utama:** SO → *Create Invoice* → Regular invoice → *Confirm* → *Send & Print* (email dengan PDF) → *Register Payment*.

**Alur alternatif:** A1: Down payment (DP 50%) untuk pesanan besar.

**Alur error:** E1: Invoice dibuat sebelum delivery dengan kebijakan *Delivered* → Odoo: "There is no invoiceable line".

**Aturan bisnis:**
- BR-INVC-01: `invoice_policy = 'delivery'` untuk Keripik 200 g.
- BR-INVC-02: Laporan keuangan penuh (neraca, laba rugi) di luar cakupan karena Accounting penuh adalah Enterprise.

**Spesifikasi field (standar):**

| Nama teknis | Label | Tipe | Default | Keterangan |
|---|---|---|---|---|
| `product.template.invoice_policy` | Invoicing Policy | Selection | `order` | ubah ke `delivery` |
| `account.move.state` | Status | Selection | `draft` | draft/posted/cancel |
| `account.move.payment_state` | Payment Status | Selection | `not_paid` | |

**State invoice:**
```mermaid
stateDiagram-v2
    [*] --> draft
    draft --> posted: Confirm
    posted --> cancel: Cancel (via Reset to draft)
    posted --> draft: Reset to draft
    posted --> [*]
```

**Acceptance criteria:**
- **AC-INVC-01** Given SO 30 pcs sudah dikirim, When Create Invoice, Then invoice draft berisi 30 × Rp 15.000.
- **AC-INVC-02** Given invoice posted, When Register Payment penuh, Then `payment_state` = `paid` (atau `in_payment`).

---

### 3(f) Quality Check (custom `mrp_keripik`)

#### FR-QC-01 — Master quality point
PPIC/QC mendefinisikan titik pemeriksaan per operasi BoM dengan jenis uji (ukur / lulus-gagal), batas min/maks, satuan, wajib/tidak, dan sumber (manual/gateway/keduanya).

#### FR-QC-02 — Pembuatan QC otomatis
Saat MO dikonfirmasi, sistem membuat satu QC `todo` untuk setiap quality point aktif pada operasi BoM MO tersebut, terhubung ke WO terkait.

#### FR-QC-03 — Pengisian QC manual
Petugas QC atau Operator mengisi nilai ukur (atau memilih Lulus/Gagal untuk uji visual). Sistem menentukan hasil otomatis berdasarkan batas.

#### FR-QC-04 — QC otomatis dari gateway
Gateway dapat (a) mengisi QC `todo` dengan ringkasan sensor dan (b) mencatat QC gagal saat anomali (lihat FR-API-02/03).

#### FR-QC-05 — Blokir penyelesaian MO
MO tidak bisa diselesaikan bila ada QC wajib berstatus `todo`, atau QC `fail` yang belum terselesaikan lewat retest.

#### FR-QC-06 — Blokir penyelesaian WO
WO tidak bisa diselesaikan bila QC wajib untuk WO tersebut masih `todo`. QC `fail` tidak memblokir WO (barang sudah diproses), tetapi memblokir MO.

#### FR-QC-07 — Retest
QC `fail` dapat di-retest: sistem membuat QC baru (`retest_of_id` = QC lama). Jika retest `pass`, QC lama ditandai `is_resolved = True`.

#### FR-QC-08 — Notifikasi QC gagal
QC `fail` dengan `source = manual` memicu Telegram ke PPIC & QC (FR-TG-03) dan pesan di chatter MO. QC `source = gateway` tidak memicu Telegram dari Odoo (sudah dikirim Go).

#### FR-QC-09 — Daftar & analisis QC
Menu *Keripik → Quality → Checks* (list, form, pivot, graph) dengan filter Gagal, Hari ini, Belum diisi, Sumber gateway.

**Aktor & hak akses:** Petugas QC (CRU check, R point), Operator (RU check milik WO yang ia kerjakan, tidak bisa mengubah QC gateway), PPIC (CRUD), Gateway (CRU via API), Owner (R).

**Prasyarat:** quality point terdefinisi; MO punya BoM dengan operasi.

**Pemicu:** `mrp.production.action_confirm`; simpan nilai QC; `button_mark_done`; `mrp.workorder` finish; API gateway.

**Alur utama (QC manual):**
1. PPIC mengonfirmasi MO → QC `todo` dibuat (FR-QC-02).
2. Operator memulai WO Penggorengan → panel QC menampilkan "Suhu minyak (160–180 °C)".
3. Operator/QC mengisi `measure_value = 172` → *Simpan*.
4. Sistem menghitung `result = pass`, mengisi `checked_by_id`, `checked_at`.
5. WO diselesaikan → semua WO selesai → PPIC *Mark as Done* → validasi lolos → MO `done`.

**Alur alternatif:**
- A1: Uji `passfail` (visual warna) → user klik **Lulus**/**Gagal**.
- A2: QC dibuat ulang bila MO dikonfirmasi ulang setelah dibatalkan? → Tidak: QC hanya dibuat bila MO belum punya QC untuk point tersebut (idempoten).
- A3: Gateway mengisi QC suhu secara otomatis saat WO selesai (FR-QC-04a).

**Alur error:**
- E1: `measure_value` kosong saat menyimpan hasil `measure` → `ValidationError("Nilai ukur wajib diisi")`.
- E2: Hasil `fail` tanpa catatan → `ValidationError("Catatan wajib diisi untuk hasil gagal")`.
- E3: Mark as Done saat ada QC `todo` → `UserError("MO WH/MO/00012 belum bisa diselesaikan. QC belum diisi: Suhu minyak (Penggorengan), Berat kemasan (Pengemasan).")`.
- E4: Mark as Done saat ada QC `fail` belum resolved → `UserError("... QC gagal: QC/2026/00031 Suhu minyak 186 °C. Lakukan retest atau hubungi Production Manager.")`.
- E5: Operator mengubah QC `source = gateway` → `AccessError`/`UserError("QC otomatis tidak dapat diubah operator")`.
- E6: Quality point `norm_min > norm_max` → `ValidationError`.

**Aturan bisnis:**
- BR-QC-01: `result` untuk `measure` = `pass` jika `norm_min ≤ value ≤ norm_max`, selain itu `fail`.
- BR-QC-02: MO done diblokir bila ∃ QC wajib `todo`, atau ∃ QC `fail` dengan `is_resolved = False`.
- BR-QC-03: Production Manager dapat **membebaskan** (waive) QC gagal dengan alasan tertulis → ditandai `is_resolved = True`, `waived_by_id` terisi (Tambahan, opsional).
- BR-QC-04: QC tidak bisa dihapus bila `result != 'todo'` (jejak audit), kecuali oleh admin.
- BR-QC-05: Satu quality point menghasilkan tepat satu QC "asli" per MO (constraint unik `production_id, point_id, retest_of_id IS NULL` dicek di Python).
- BR-QC-06: `external_ref` unik untuk idempotensi API.

**Quality point default (data):**

| Operasi | Nama | Jenis | Min | Maks | Satuan | Sumber |
|---|---|---|---|---|---|---|
| Pengirisan | Ketebalan irisan | measure | 1,5 | 2,5 | mm | manual |
| Penggorengan | Suhu minyak | measure | 160 | 180 | °C | both |
| Penggorengan | Warna keemasan | passfail | – | – | – | manual |
| Pembumbuan | Kadar air | measure | 0 | 4 | % | manual |
| Pengemasan | Berat kemasan rata-rata | measure | 198 | 205 | g | both |
| Pengemasan | Seal kemasan rapat | passfail | – | – | – | manual |

**Spesifikasi field — `keripik.quality.point`:**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `name` | Nama | Char | ✔ | – | – | translate |
| `sequence` | Urutan | Integer | – | 10 | – | handle widget |
| `operation_id` | Operasi | Many2one `mrp.routing.workcenter` | ✔ | – | – | `ondelete='cascade'` |
| `bom_id` | BoM | Many2one (related, store) | – | – | – | |
| `test_type` | Jenis uji | Selection(`measure`,`passfail`) | ✔ | `measure` | – | |
| `measure_type` | Besaran | Selection(`thickness`,`oil_temp`,`color`,`moisture`,`pack_weight`,`seal`,`other`) | ✔ | `other` | – | dicocokkan gateway |
| `norm_min` | Batas min | Float | ✔ bila measure | 0 | ≤ `norm_max` | |
| `norm_max` | Batas maks | Float | ✔ bila measure | 0 | ≥ `norm_min` | |
| `unit` | Satuan | Char | – | – | – | |
| `mandatory` | Wajib | Boolean | – | True | – | |
| `source_allowed` | Sumber diizinkan | Selection(`manual`,`gateway`,`both`) | ✔ | `manual` | – | |
| `instructions` | Instruksi | Html | – | – | – | |
| `active` | Aktif | Boolean | – | True | – | |
| `company_id` | Perusahaan | Many2one | ✔ | company aktif | – | |

**Spesifikasi field — `keripik.quality.check`:**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `name` | Nomor | Char | ✔ | sequence `QC/%(year)s/00001` | – | readonly, copy=False |
| `point_id` | Titik QC | Many2one | ✔ | – | – | |
| `production_id` | MO | Many2one `mrp.production` | ✔ | – | – | cascade, index |
| `workorder_id` | WO | Many2one `mrp.workorder` | – | – | WO.production_id = production_id | |
| `test_type` | Jenis uji | related | – | – | – | |
| `measure_type` | Besaran | related store | – | – | – | |
| `norm_min`, `norm_max`, `unit` | Batas, satuan | related | – | – | – | |
| `measure_value` | Nilai ukur | Float | ✔ saat hasil measure | – | E1 | |
| `result` | Hasil | Selection(`todo`,`pass`,`fail`) | ✔ | `todo` | – | tracking=True |
| `source` | Sumber | Selection(`manual`,`gateway`) | ✔ | `manual` | – | |
| `checked_by_id` | Diperiksa oleh | Many2one `res.users` | – | – | – | |
| `checked_at` | Waktu periksa | Datetime | – | – | – | |
| `note` | Catatan | Text | ✔ bila fail | – | E2 | |
| `retest_of_id` | Retest dari | Many2one self | – | – | – | |
| `retest_ids` | Retest | One2many | – | – | – | |
| `is_resolved` | Terselesaikan | Boolean compute store | – | False | – | depends `result`, `retest_ids.result` |
| `waived_by_id`, `waive_reason` | Dibebaskan oleh, alasan | Many2one, Text | – | – | hanya Manager | opsional BR-QC-03 |
| `external_ref` | Ref eksternal | Char | – | – | unik | index |
| `company_id` | Perusahaan | related production | – | – | – | |

**Spesifikasi field tambahan — `mrp.production` & `mrp.workorder`:** lihat D4 §4.2 (`qc_check_ids`, `qc_state`, `qc_todo_count`, `qc_fail_count`).

**State QC:**
```mermaid
stateDiagram-v2
    [*] --> todo: MO dikonfirmasi
    todo --> pass: nilai dalam batas / klik Lulus
    todo --> fail: nilai di luar batas / klik Gagal / anomali gateway
    pass --> fail: koreksi nilai oleh QC (tercatat di chatter)
    fail --> pass: koreksi nilai oleh QC
    fail --> resolved: retest pass / waive Manager
    pass --> [*]
    resolved --> [*]
    note right of resolved
        resolved = result fail + is_resolved True
    end note
```

**State `qc_state` MO (compute):**
```mermaid
stateDiagram-v2
    [*] --> none: tidak ada QC
    [*] --> todo: ada QC todo
    todo --> fail: ada QC fail belum resolved
    todo --> pass: semua QC pass/resolved
    fail --> pass: retest pass
    pass --> fail: QC baru gagal
```

**Catatan tampilan:**
- Menu: *Manufacturing → Keripik → Quality Points*, *Quality Checks*.
- MO form: smart button "QC 4/6" (warna merah bila ada gagal), tab **Quality** berisi list QC inline (editable sesuai hak akses).
- MO list: kolom `qc_state` badge (abu = todo, hijau = pass, merah = fail).
- QC form: statusbar `result`, tombol **Lulus**/**Gagal** (untuk passfail), **Retest** (bila fail & belum resolved), **Waive** (Manager).
- Pesan error memakai bahasa Indonesia dan menyebut nomor QC serta operasi.

**Acceptance criteria:**
- **AC-QC-01** Given BoM Keripik Siap Kemas punya 2 quality point Penggorengan dan 1 Pembumbuan, When MO dikonfirmasi, Then 3 QC `todo` terbentuk dan terhubung ke WO masing-masing.
- **AC-QC-02** Given MO yang sama dikonfirmasi ulang/aksi dijalankan dua kali, When QC digenerate lagi, Then tidak ada QC duplikat.
- **AC-QC-03** Given QC suhu (160–180), When nilai 172 disimpan, Then `result = pass` dan `checked_by_id` = user saat ini.
- **AC-QC-04** Given QC suhu, When nilai 186 disimpan tanpa catatan, Then muncul error "Catatan wajib diisi untuk hasil gagal".
- **AC-QC-05** Given ada QC wajib `todo`, When PPIC klik Mark as Done, Then muncul UserError yang menyebut nama QC dan MO tetap tidak `done`.
- **AC-QC-06** Given ada QC `fail` belum resolved, When Mark as Done, Then ditolak; When retest dibuat dan bernilai pass, Then Mark as Done berhasil.
- **AC-QC-07** Given QC wajib WO Pengemasan masih `todo`, When operator klik Selesai di WO itu, Then ditolak dengan pesan "Isi QC terlebih dahulu".
- **AC-QC-08** Given QC manual berubah ke `fail`, When disimpan, Then Telegram terkirim ke PPIC & QC yang terhubung dan chatter MO mencatatnya.
- **AC-QC-09** Given user Operator, When mencoba mengubah QC `source = gateway`, Then ditolak.
- **AC-QC-10** Given QC `pass`, When user non-admin mencoba menghapus, Then ditolak.

---

### 3(g) Laporan Yield & Susut

#### FR-YLD-01 — Analisis konsumsi bahan
Model laporan `keripik.yield.report` (SQL view) menampilkan per baris: MO, tanggal selesai, produk jadi, bahan, operasi/work center, qty rencana, qty aktual, selisih, selisih %.

#### FR-YLD-02 — Yield produk
Per MO: `yield_pct = qty_produced / product_qty × 100` dan *yield bahan utama*: kg produk jadi ÷ kg bahan utama dikonsumsi.

#### FR-YLD-03 — Pivot & graph
View pivot (baris: produk jadi/operasi, kolom: bulan, ukuran: rencana, aktual, selisih) dan graph (bar selisih % per operasi, line yield per minggu).

#### FR-YLD-04 — QWeb report per MO
PDF "Laporan Yield & Susut" dari MO: header MO, tabel bahan (rencana/aktual/selisih), tabel per operasi (durasi rencana vs aktual), by-product, ringkasan yield.

#### FR-YLD-05 — Laporan periode
Wizard pilih periode & produk → PDF rekap yield semua MO.

**Aktor:** PPIC (R semua), QC (R), Owner (R), Monitor (R via API).

**Prasyarat:** MO `done` dengan `stock.move` bahan yang punya `operation_id` (diisi dari `bom_line.operation_id`).

**Pemicu:** buka menu *Keripik → Laporan → Yield & Susut*; klik *Print → Laporan Yield* di MO.

**Alur utama:**
1. PPIC membuka menu Yield & Susut → default pivot, filter "Bulan ini", group by produk jadi.
2. Expand ke operasi → melihat Penggorengan selisih minyak +12%.
3. Klik sel pivot → list detail move → buka MO.
4. Dari MO, *Print → Laporan Yield & Susut* → PDF.

**Alur alternatif:** A1: graph view untuk tren; A2: export ke XLSX (fitur standar list/pivot).

**Alur error:** E1: MO belum `done` → tidak tampil di laporan (filter domain `state = 'done'`). E2: Bahan tanpa `operation_id` → dikelompokkan sebagai "(Tanpa operasi)".

**Aturan bisnis:**
- BR-YLD-01: `qty_planned` = `stock.move.product_uom_qty` saat MO dikonfirmasi, dinormalisasi ke qty yang benar-benar diproduksi: `planned_adj = product_uom_qty × (qty_produced / product_qty)`. Tanpa normalisasi, produksi parsial terlihat seperti "hemat bahan".
- BR-YLD-02: `qty_actual` = jumlah `stock.move.quantity` (⚠️ nama field 19) dengan state `done`.
- BR-YLD-03: `variance = actual − planned_adj`; `variance_pct = variance / planned_adj × 100` (0 bila planned 0).
- BR-YLD-04: Satuan dikonversi ke UoM produk (`product_uom` move → `uom_id` produk).
- BR-YLD-05: Susut > 5% ditandai merah di QWeb report (ambang di `ir.config_parameter` `mrp_keripik.yield_variance_threshold`).

**Spesifikasi field — `keripik.yield.report` (read-only):**

| Nama teknis | Label | Tipe | Keterangan |
|---|---|---|---|
| `id` | ID | Integer | = `stock_move.id` |
| `production_id` | MO | Many2one `mrp.production` | |
| `date` | Tanggal selesai | Datetime | `mrp_production.date_finished` |
| `finished_product_id` | Produk jadi | Many2one `product.product` | |
| `product_id` | Bahan | Many2one `product.product` | |
| `operation_id` | Operasi | Many2one `mrp.routing.workcenter` | |
| `workcenter_id` | Work center | Many2one `mrp.workcenter` | |
| `uom_id` | Satuan | Many2one `uom.uom` | |
| `qty_planned` | Qty rencana | Float | BR-YLD-01, `aggregator='sum'` ⚠️ (dulu `group_operator`) |
| `qty_actual` | Qty aktual | Float | sum |
| `qty_variance` | Selisih | Float | sum |
| `variance_pct` | Selisih % | Float | `aggregator='avg'` |
| `company_id` | Perusahaan | Many2one | |

**Field tambahan `mrp.production`:** `yield_pct` (Float, compute store, depends `qty_produced`, `product_qty`, `state`).

**State:** tidak ada (laporan).

**Catatan tampilan:** pivot default; graph type bar; search view filter: Bulan ini, Minggu ini, Selisih > 5%; group by: Produk jadi, Operasi, Bahan, Bulan. QWeb A4 portrait dengan logo perusahaan (`web.external_layout`).

**Acceptance criteria:**
- **AC-YLD-01** Given MO Keripik Siap Kemas rencana Minyak 3 L dan aktual 3,3 L, When membuka laporan, Then baris Minyak menunjukkan selisih +0,3 L dan +10%.
- **AC-YLD-02** Given MO diproduksi 15 dari 30 (parsial), When laporan dihitung, Then qty rencana bahan dinormalisasi 50%.
- **AC-YLD-03** Given MO belum done, When membuka laporan, Then MO tersebut tidak tampil.
- **AC-YLD-04** Given MO done, When Print → Laporan Yield & Susut, Then PDF berisi tabel bahan, tabel operasi, by-product, dan yield %; selisih > 5% berwarna merah.
- **AC-YLD-05** Given user Owner, When membuka pivot, Then bisa melihat tetapi tidak ada tombol create/edit.

---

### 3(h) Traceability Lot

#### FR-TRC-01 — Telusur ke belakang (backward)
Dari lot produk jadi (misal `KK-20261012-01`), tampilkan pohon semua lot bahan yang dikonsumsi di semua level: Keripik Siap Kemas → Tempe Iris → Tempe Mentah, beserta tepung, minyak, bumbu, plastik, vendor dan PO asal.

#### FR-TRC-02 — Telusur ke depan (forward)
Dari lot bahan baku (misal `TM-20261010-A` dari vendor X), tampilkan semua lot produk setengah jadi dan produk jadi yang memakainya, sampai ke **pelanggan & delivery order**.

#### FR-TRC-03 — Wizard simulasi recall
Wizard: pilih lot + arah → hasil: daftar lot terdampak, qty masih di gudang, qty terkirim, daftar pelanggan (nama, SO, delivery, tanggal, qty).

#### FR-TRC-04 — QWeb report recall
PDF "Laporan Traceability / Recall" dari wizard.

**Aktor:** PPIC, QC (R), Owner (R).

**Prasyarat:** semua produk ber-lot; MO done dengan lot terisi; relasi `stock.move.line.consume_line_ids`/`produce_line_ids` ⚠️ (verifikasi di 19).

**Pemicu:** menu *Keripik → Laporan → Simulasi Recall*; tombol *Traceability* di form lot.

**Alur utama (backward):**
1. QC membuka wizard → pilih lot `KK-20261012-01` → arah *Ke belakang* → *Telusuri*.
2. Sistem mengambil move line produksi yang menghasilkan lot itu → baris bahan yang dikonsumsi (`consume_line_ids`) → untuk setiap lot bahan yang juga hasil produksi, ulangi (rekursif).
3. Lot bahan tanpa produksi asal = bahan baku → cari receipt/PO asal dan vendor.
4. Tampilkan pohon + tombol *Cetak PDF*.

**Alur utama (forward):**
1. Pilih lot bahan baku → arah *Ke depan*.
2. Cari move line konsumsi lot itu → `produce_line_ids` → lot hasil → rekursif sampai produk jadi.
3. Untuk lot produk jadi: cari move line ke lokasi pelanggan (delivery `outgoing` done) → SO & pelanggan.
4. Hitung qty terkirim vs masih di stok (`stock.quant`).

**Alur alternatif:** A1: dari form `stock.lot` klik smart button **Recall** → wizard terisi otomatis.

**Alur error:** E1: lot tanpa pergerakan → "Tidak ada data pergerakan untuk lot ini". E2: siklus data (tidak semestinya) → batas kedalaman 10 level, hentikan dan beri peringatan.

**Aturan bisnis:**
- BR-TRC-01: Penelusuran memakai `sudo()` agar lengkap lintas modul, tetapi akses wizard hanya untuk grup Manager/Quality/Owner.
- BR-TRC-02: Kedalaman maksimum 10; node yang sudah dikunjungi tidak diulang (cegah loop).
- BR-TRC-03: Qty ditampilkan dalam UoM produk.

**Spesifikasi field — wizard `keripik.recall.wizard` (TransientModel):**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `lot_id` | Lot | Many2one `stock.lot` | ✔ | dari context | – | |
| `direction` | Arah | Selection(`backward`,`forward`) | ✔ | `forward` jika lot bahan baku, `backward` jika produk jadi | – | |
| `line_ids` | Hasil | One2many `keripik.recall.wizard.line` | – | – | – | |
| `customer_count` | Jumlah pelanggan | Integer compute | – | – | – | |

`keripik.recall.wizard.line`: `level` (Integer), `lot_id`, `product_id`, `qty`, `uom_id`, `production_id`, `picking_id`, `partner_id`, `date`, `location_id`, `qty_in_stock`.

**State:** tidak ada.

**Catatan tampilan:** wizard dialog dengan list indentasi per `level` (prefix "— ") dan tombol *Telusuri*, *Cetak PDF*, *Tutup*. PDF memuat ringkasan "X lot terdampak, Y pelanggan, Z kg masih di gudang".

**Acceptance criteria:**
- **AC-TRC-01** Given lot `KK-…-01` diproduksi dari lot Keripik Siap Kemas `KS-…-01` yang berasal dari Tempe Iris `TI-…-01` dari Tempe Mentah `TM-…-A`, When telusur ke belakang, Then keempat level tampil beserta vendor Tempe Mentah.
- **AC-TRC-02** Given lot Tempe Mentah `TM-…-A` dipakai 2 MO dan hasilnya dikirim ke 2 pelanggan, When telusur ke depan, Then wizard menampilkan 2 pelanggan dengan SO dan qty masing-masing.
- **AC-TRC-03** Given hasil wizard, When klik Cetak PDF, Then PDF memuat seluruh baris yang sama dengan wizard.
- **AC-TRC-04** Given user Operator, When mencoba membuka menu recall, Then menu tidak tampil.

---

### 3(i) Layar Operator (OWL)

#### FR-OPS-01 — Akses layar
Client action `mrp_keripik.operator_screen` di menu *Manufacturing → Layar Operator*, bisa dibuka layar penuh.

#### FR-OPS-02 — Pilih work center
Dropdown work center (diingat di `localStorage`).

#### FR-OPS-03 — Daftar WO
Kartu WO dengan status `ready`, `progress`, (paused); urut berdasar prioritas & tanggal; refresh tiap 15 detik atau lewat bus notification.

#### FR-OPS-04 — Mulai/Jeda/Selesai
Tombol besar memanggil method server yang sesuai; jeda meminta alasan (loss reason).

#### FR-OPS-05 — Input hasil & scrap
Input `qty_producing`/qty hasil dan scrap (produk, qty, alasan) dari panel.

#### FR-OPS-06 — Form QC
QC milik WO tampil sebagai daftar; isi nilai/Lulus/Gagal langsung dari panel.

#### FR-OPS-07 — Ringkasan sensor
Panel menampilkan suhu terakhir/rata-rata/maks dari `keripik.sensor.summary` atau endpoint gateway (bila aktif).

#### FR-OPS-08 — Ramah tablet
Layout responsif ≥ 768 px, tombol ≥ 56 px, tanpa hover, bisa dipakai dengan sarung tangan (target sentuh besar).

**Aktor:** Operator (grup Operator), PPIC.

**Prasyarat:** user punya grup Operator; work center & WO tersedia; asset bundle `web.assets_backend` memuat komponen.

**Pemicu:** operator membuka menu.

**Alur utama:** lihat D3(c).

**Alur alternatif:**
- A1: Dua operator membuka WO sama → aksi kedua menerima state terbaru dari server (refresh otomatis setelah setiap aksi).
- A2: Koneksi terputus → banner "Offline, aksi tidak tersimpan" dan tombol nonaktif.

**Alur error:**
- E1: Mulai WO yang `blocked` → pesan dari server "WO sebelumnya belum selesai".
- E2: Selesai dengan QC wajib `todo` → pesan FR-QC-06.
- E3: Qty hasil ≤ 0 → validasi di klien + server.
- E4: Error RPC lain → notifikasi merah dengan pesan server.

**Aturan bisnis:**
- BR-OPS-01: Semua aksi lewat **satu method server** `mrp.workorder.keripik_operator_action(action, values)` agar validasi tidak bergantung klien.
- BR-OPS-02: Start/pause/finish juga membuat `keripik.gateway.event` (bila gateway aktif).
- BR-OPS-03: Operator hanya melihat WO di work center yang dipilih; tidak ada akses hapus.

**Spesifikasi field (input layar):**

| Nama | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `workcenter_id` | Work center | select | ✔ | terakhir dipakai | – | localStorage `keripik.operator.wc` |
| `qty_producing` | Qty hasil | number | ✔ saat Selesai | qty rencana WO | > 0, ≤ 2 × rencana | |
| `scrap_qty` | Scrap | number | – | 0 | ≥ 0 | |
| `scrap_reason` | Alasan scrap | select | ✔ bila scrap > 0 | – | – | `stock.scrap.reason.tag` ⚠️ |
| `pause_reason_id` | Alasan jeda | select | ✔ saat Jeda | – | – | `mrp.workcenter.productivity.loss` |
| `qc_value[i]` | Nilai QC | number / tombol | sesuai point | – | – | |

**State tampilan WO di layar:**
```mermaid
stateDiagram-v2
    [*] --> Ready
    Ready --> Running: MULAI
    Running --> Paused: JEDA + alasan
    Paused --> Running: MULAI
    Running --> Done: SELESAI (QC wajib terisi, qty > 0)
    Running --> Running: Simpan QC / scrap
    Done --> [*]
```

**Catatan tampilan:** lihat D6 §6.4A. Warna status kartu: hijau (berjalan), abu (ready), kuning (jeda), merah (ada QC gagal). Toast notification untuk sukses/gagal.

**Acceptance criteria:**
- **AC-OPS-01** Given operator memilih WC-GORENG-1, When layar dimuat, Then hanya WO WC-GORENG-1 berstatus ready/progress yang tampil.
- **AC-OPS-02** Given WO ready, When tap MULAI, Then status menjadi berjalan, timer berjalan, dan time log baru tercatat di backend.
- **AC-OPS-03** Given WO berjalan, When tap JEDA dan memilih alasan, Then time log ditutup dengan loss reason tersebut.
- **AC-OPS-04** Given QC wajib belum diisi, When tap SELESAI, Then muncul pesan "Isi QC terlebih dahulu" dan WO tetap berjalan.
- **AC-OPS-05** Given QC terisi dan qty 5,8, When SELESAI, Then WO done dan kartu hilang dari antrean.
- **AC-OPS-06** Given tablet 1024×768, When layar dibuka, Then semua tombol utama terlihat tanpa scroll horizontal dan tinggi ≥ 56 px.
- **AC-OPS-07** Given operator input scrap 0,2 kg dengan alasan, When disimpan, Then `stock.scrap` tercipta dan terhubung ke MO/WO.

---

### 3(j) Biaya Produksi per Batch

#### FR-CST-01 — Biaya bahan
`cost_material` = Σ (qty aktual move bahan done × harga per unit bahan saat konsumsi).

#### FR-CST-02 — Biaya work center
`cost_workcenter` = Σ per time log (`mrp.workcenter.productivity.duration` menit ÷ 60 × `workcenter.costs_hour`).

#### FR-CST-03 — Biaya total & per unit
`cost_total = cost_material + cost_workcenter`; `cost_per_unit = cost_total / qty_produced` (UoM produk).

#### FR-CST-04 — Tampilan & laporan
Tab **Biaya** di MO, kolom di list, dan bagian ringkasan di QWeb yield report; pivot biaya per produk per bulan.

**Aktor:** PPIC (R), Owner (R), Monitor (R via API).

**Prasyarat:** `standard_price` bahan terisi (dari PO/valuasi); `costs_hour` work center terisi.

**Pemicu:** MO berubah ke `done` (hitung & simpan); tombol *Hitung ulang biaya* (Manager).

**Alur utama:** MO done → compute dipicu (`@api.depends('state', 'move_raw_ids.state', 'workorder_ids.time_ids.duration')`) → nilai tersimpan.

**Alur alternatif:** A1: MO masih berjalan → tampil estimasi (biaya bahan rencana + durasi expected) dengan label "Estimasi".

**Alur error:** E1: `costs_hour = 0` → biaya WC 0 dan peringatan kecil di form "Biaya per jam work center belum diatur".

**Aturan bisnis:**
- BR-CST-01: Harga bahan diambil dari nilai valuasi move bila tersedia (`stock_account`), jika tidak dari `standard_price` produk pada saat MO done (snapshot, disimpan di field stored agar tidak berubah saat harga berubah).
- BR-CST-02: By-product cost share 0% (BR-MRP-03) → seluruh biaya ke produk utama.
- BR-CST-03: Biaya MO turunan (Tempe Iris → Keripik Siap Kemas) **tidak** dijumlahkan dua kali: biaya bahan MO induk sudah memakai harga produk setengah jadi.
- BR-CST-04: Mata uang = mata uang perusahaan (IDR).

**Spesifikasi field — `mrp.production` (tambahan):**

| Nama teknis | Label | Tipe | Wajib | Default | Keterangan |
|---|---|---|---|---|---|
| `currency_id` | Mata uang | Many2one related `company_id.currency_id` | – | – | |
| `cost_material` | Biaya bahan | Monetary compute store | – | 0 | FR-CST-01 |
| `cost_workcenter` | Biaya work center | Monetary compute store | – | 0 | FR-CST-02 |
| `cost_total` | Total biaya | Monetary compute store | – | 0 | |
| `cost_per_unit` | Biaya per unit | Monetary compute store | – | 0 | |
| `cost_is_estimate` | Estimasi | Boolean compute | – | True | False setelah done |
| `duration_actual_hours` | Jam aktual | Float compute | – | – | info |

**State:** mengikuti MO (estimasi → final saat done).

**Catatan tampilan:** tab "Biaya" di MO form: tabel ringkas bahan (qty × harga), work center (jam × tarif), total, per unit; badge "Estimasi" bila belum done.

**Acceptance criteria:**
- **AC-CST-01** Given MO Keripik Siap Kemas mengonsumsi Tempe Iris 10 kg × Rp 14.000, Tepung 2 kg × Rp 12.000, Minyak 3,3 L × Rp 18.000, Bumbu 0,3 kg × Rp 60.000, When done, Then `cost_material` = 140.000 + 24.000 + 59.400 + 18.000 = **Rp 241.400**.
- **AC-CST-02** Given WO Penggorengan 70 menit di WC Rp 40.000/jam dan Pembumbuan 20 menit di WC Rp 20.000/jam, When done, Then `cost_workcenter` = 46.666,67 + 6.666,67 = **Rp 53.333,33**.
- **AC-CST-03** Given kedua nilai di atas dan qty produced 6 kg, Then `cost_total` = Rp 294.733,33 dan `cost_per_unit` ≈ Rp 49.122,22/kg.
- **AC-CST-04** Given harga standar Minyak berubah setelah MO done, When membuka MO lama, Then biaya tidak berubah.

---

### 3(k) Scheduled Action — Bahan Baku Mendekati Kedaluwarsa

#### FR-EXP-01 — Cron harian
`ir.cron` "Keripik: Cek lot mendekati kedaluwarsa" berjalan tiap hari pukul 06.00 (waktu server; atur `nextcall`).

#### FR-EXP-02 — Deteksi
Cari `stock.lot` dengan stok > 0 di lokasi internal dan `alert_date ≤ sekarang` atau `expiration_date ≤ sekarang + N hari` (N = `mrp_keripik.expiry_warning_days`, default 2), yang belum diberi peringatan hari ini.

#### FR-EXP-03 — Tindakan
(a) Buat *activity* "Lot mendekati kedaluwarsa" untuk Admin Gudang (user di parameter) pada lot; (b) kirim satu Telegram ringkasan ke penerima `notify_expiry`; (c) tandai `keripik_expiry_notified_date` di lot.

#### FR-EXP-04 — Lot kedaluwarsa
Lot yang sudah lewat `expiration_date` diberi tag "EXPIRED" di ringkasan (pemindahan ke lokasi karantina manual).

**Aktor:** Odoo Scheduler (sistem), Admin Gudang (menerima activity), PPIC (Telegram).

**Prasyarat:** `product_expiry` aktif; lot memiliki tanggal.

**Pemicu:** cron; tombol *Run Manually* di cron (admin, untuk uji).

**Alur utama:**
1. Cron memanggil `stock.lot._cron_keripik_expiry_alert()`.
2. Query `stock.quant` (lokasi `usage = 'internal'`, `quantity > 0`) → kumpulkan lot dan qty.
3. Filter lot sesuai FR-EXP-02 dan `keripik_expiry_notified_date != today`.
4. Buat activity per lot (`activity_schedule`), susun pesan ringkasan, kirim Telegram (FR-TG).
5. Update `keripik_expiry_notified_date = today`.

**Alur alternatif:** A1: tidak ada lot → cron selesai tanpa pesan (opsional: tidak kirim apa pun).

**Alur error:** E1: Telegram gagal → tercatat di `keripik.telegram.log`, activity tetap dibuat (activity tidak bergantung Telegram). E2: Exception di satu lot → di-log, lanjut lot lain (try/except per lot dengan savepoint).

**Aturan bisnis:**
- BR-EXP-01: Satu lot maksimal satu peringatan per hari.
- BR-EXP-02: Hanya lot dengan stok internal > 0.
- BR-EXP-03: Pesan dikelompokkan per produk, diurutkan tanggal kedaluwarsa terdekat; maksimal 30 baris lalu "+N lot lainnya".

> Catatan: `product_expiry` standar sudah punya cron yang membuat activity berdasarkan `alert_date`. ⚠️ Cek `addons/product_expiry/data/product_expiry_data.xml` di 19. Fitur custom ini **melengkapi** dengan Telegram, ambang N hari, dan ringkasan, bukan menduplikasi. Jelaskan ini saat interview.

**Spesifikasi field:**

| Nama teknis | Label | Tipe | Default | Keterangan |
|---|---|---|---|---|
| `stock.lot.keripik_expiry_notified_date` | Tgl peringatan terakhir | Date | – | baru |
| `ir.config_parameter mrp_keripik.expiry_warning_days` | – | Char (int) | `2` | |
| `ir.config_parameter mrp_keripik.expiry_responsible_login` | – | Char | `gudang` | user penerima activity |
| `ir.cron` `interval_number` / `interval_type` | Interval | Integer/Selection | 1 / `days` | ⚠️ `numbercall` sudah dihapus sejak Odoo 17 |

**State:** tidak ada.

**Catatan tampilan:** activity muncul di lot dan di menu Activity Admin Gudang. Format Telegram:
```
⚠️ Bahan mendekati kedaluwarsa (12/10/2026)
• Tempe Mentah — TM-20261010-A — 12 kg — exp 13/10 (besok)
• Tepung Bumbu — TB-20260415-B — 3 kg — exp 14/10
```

**Acceptance criteria:**
- **AC-EXP-01** Given lot Tempe Mentah stok 12 kg dan expiry besok, When cron dijalankan, Then activity dibuat untuk Admin Gudang dan Telegram ringkasan terkirim ke PPIC yang terhubung.
- **AC-EXP-02** Given cron dijalankan dua kali di hari yang sama, Then lot yang sama tidak diberi peringatan dua kali.
- **AC-EXP-03** Given lot expiry besok tetapi stok 0, When cron dijalankan, Then lot tidak termasuk.
- **AC-EXP-04** Given Telegram API tidak dapat dihubungi, When cron dijalankan, Then activity tetap dibuat dan log mencatat kegagalan.

---

### 3(l) Notifikasi Email Pelanggan

#### FR-EML-01 — Pesanan dikonfirmasi
Saat SO berubah ke `sale`, email "Pesanan Anda dikonfirmasi" dengan ringkasan item, estimasi produksi, dan link portal. Memakai/menyesuaikan template standar `sale.mail_template_sale_confirmation` ⚠️ (cek XML ID di 19).

#### FR-EML-02 — Produksi selesai
Saat MO **level teratas** (yang menghasilkan produk yang dipesan) `done`, email "Pesanan Anda selesai diproduksi" ke pelanggan SO terkait.

#### FR-EML-03 — Barang siap dikirim
Saat delivery order pelanggan menjadi `assigned` (siap), email "Pesanan siap dikirim" dengan nomor delivery & estimasi pengiriman.

#### FR-EML-04 — SMTP
Outgoing mail server ke Mailpit (dev) / SMTP asli (produksi).

**Aktor:** Pelanggan (penerima), sistem.

**Prasyarat:** pelanggan punya email; outgoing mail server aktif; `mail.template` ter-install dari data module.

**Pemicu:** write state SO/MO/picking.

**Alur utama (FR-EML-02):**
1. `mrp.production.button_mark_done` sukses → untuk setiap MO done yang terhubung ke SO (lewat relasi `sale_mrp` ⚠️), panggil `template.send_mail(mo.id)` atau `message_post_with_source`.
2. Email masuk antrean `mail.mail` → cron *Mail: Email Queue Manager* mengirim ke SMTP.

**Alur alternatif:** A1: satu SO menghasilkan beberapa MO (beberapa line) → email dikirim saat **semua** MO level teratas SO done (BR-EML-02).

**Alur error:** E1: pelanggan tanpa email → dilewati, pesan di chatter "Email tidak dikirim: alamat kosong". E2: SMTP gagal → `mail.mail` state `exception`, dapat di-*retry* dari menu Technical → Emails.

**Aturan bisnis:**
- BR-EML-01: Email dikirim maksimal sekali per event per dokumen (flag `keripik_mail_done_sent` di SO, `keripik_mail_ready_sent` di picking).
- BR-EML-02: "Produksi selesai" dikirim per SO, bukan per MO.
- BR-EML-03: Bahasa template mengikuti `partner_id.lang` (template translatable).
- BR-EML-04: Hanya MO level teratas (tidak untuk MO Tempe Iris/Keripik Siap Kemas).

**Spesifikasi field:**

| Nama teknis | Label | Tipe | Default | Keterangan |
|---|---|---|---|---|
| `sale.order.keripik_mail_done_sent` | Email produksi selesai terkirim | Boolean | False | baru, copy=False |
| `stock.picking.keripik_mail_ready_sent` | Email siap kirim terkirim | Boolean | False | baru, copy=False |
| `mail.template` `mrp_keripik.mail_template_production_done` | – | record | – | model `sale.order` |
| `mail.template` `mrp_keripik.mail_template_delivery_ready` | – | record | – | model `stock.picking` |

**Catatan tampilan:** template HTML sederhana dengan logo, salam, tabel item, tombol "Lihat pesanan" (link portal `object.get_portal_url()`). Email terekam di chatter dokumen.

**Acceptance criteria:**
- **AC-EML-01** Given SO dikonfirmasi, When membuka Mailpit (`localhost:8025`), Then ada email "Pesanan … dikonfirmasi" ke alamat pelanggan.
- **AC-EML-02** Given 3 MO berantai untuk 1 SO, When MO Tempe Iris dan Keripik Siap Kemas done, Then belum ada email produksi selesai; When MO Keripik 200 g done, Then tepat satu email terkirim.
- **AC-EML-03** Given delivery menjadi Ready, When dicek di Mailpit, Then email "siap dikirim" berisi nomor `WH/OUT/xxxxx`.
- **AC-EML-04** Given picking kembali ke waiting lalu Ready lagi, Then email tidak dikirim dua kali.

---

### 3(m) Notifikasi & Bot Telegram

#### FR-TG-01 — Konfigurasi bot
Parameter sistem: `mrp_keripik.telegram_bot_token`, `mrp_keripik.telegram_bot_username`, `mrp_keripik.telegram_poller` (`odoo`/`gateway`/`off`).

#### FR-TG-02 — Penautan akun karyawan
Di form karyawan (atau profil "My Profile"), tombol **Hubungkan Telegram** menghasilkan kode 8 karakter (`A-Z0-9`, tanpa karakter ambigu), berlaku 15 menit, beserta link `https://t.me/<bot>?start=<kode>`. Saat bot menerima `/start <kode>`, `telegram_chat_id` tersimpan dan kode dihapus.

#### FR-TG-03 — Notifikasi event internal
| Event | Penerima | Isi |
|---|---|---|
| QC gagal (manual) | PPIC (Manager) + QC dengan `notify_qc_fail` | MO, operasi, nilai, batas, pemeriksa |
| MO selesai | PPIC dengan `notify_mo` | MO, produk, qty, yield %, biaya/unit |
| MO tertunda | PPIC dengan `notify_mo` | MO yang `date_deadline` < sekarang dan belum done (cron tiap jam, sekali per MO per hari) |
| Bahan mendekati kedaluwarsa | PPIC + Gudang dengan `notify_expiry` | FR-EXP-03 |

#### FR-TG-04 — Poller Odoo (F6)
Bila `telegram_poller = odoo`: cron tiap 1 menit memanggil `getUpdates` (`timeout=0` atau kecil agar cron tidak menggantung worker), memproses `/start`, menyimpan offset di parameter `mrp_keripik.telegram_offset`.

#### FR-TG-05 — Putus tautan
Tombol **Putuskan Telegram** mengosongkan chat_id. Jika Telegram mengembalikan 403 (bot diblokir), chat_id dikosongkan otomatis.

#### FR-TG-06 — Log
Semua pengiriman tercatat di `keripik.telegram.log`.

**Aktor:** Karyawan internal (menautkan), PPIC/QC/Gudang (menerima), sistem.

**Prasyarat:** bot dibuat lewat @BotFather; token disimpan di parameter; karyawan punya user terkait.

**Pemicu:** event bisnis; `/start` dari Telegram.

**Alur utama (penautan):** lihat D3(e.1).

**Alur utama (kirim):**
1. Event terjadi → method `keripik.telegram.notify(event_type, records)` menentukan penerima (grup + preferensi + punya chat_id).
2. Pesan disusun (HTML parse mode, escape karakter `< > &`).
3. Pengiriman didaftarkan di `cr.postcommit` → setelah commit, HTTP POST `sendMessage` (timeout 5 dtk).
4. Hasil dicatat di log.

**Alur alternatif:** A1: karyawan belum terhubung → dilewati, log `skipped`. A2: poller = gateway → cron Odoo nonaktif; `/start` ditangani Go via API FR-API-04.

**Alur error:**
- E1: Kode salah/kedaluwarsa → bot membalas "Kode tidak valid atau kedaluwarsa. Buat kode baru di Odoo."
- E2: `/start` tanpa kode → bot membalas cara menautkan.
- E3: HTTP 429 → hormati `parameters.retry_after`; tandai failed untuk dicoba ulang oleh cron retry.
- E4: HTTP 403 → kosongkan chat_id, log, chatter karyawan "Telegram terputus".
- E5: Timeout jaringan → log failed; transaksi bisnis **tidak** di-rollback.

**Aturan bisnis:**
- BR-TG-01: Bot tidak bisa mengirim pesan ke user yang belum menekan Start / sudah memblokir bot (batasan Telegram).
- BR-TG-02: Kode sekali pakai, TTL 15 menit, unik di seluruh karyawan.
- BR-TG-03: Satu chat_id hanya untuk satu karyawan (constraint unik).
- BR-TG-04: Pengiriman setelah commit; kegagalan Telegram tidak membatalkan transaksi.
- BR-TG-05: Batas laju: ±1 pesan/detik per chat dan ±30 pesan/detik total (pedoman Telegram, ⚠️ cek FAQ resmi).
- BR-TG-06: Hanya satu poller aktif per token (`getUpdates` tidak bisa paralel dan tidak bisa bersamaan dengan webhook).
- BR-TG-07: Token bot hanya bisa dibaca grup System (`groups='base.group_system'` pada field settings).

**Spesifikasi field — `hr.employee` (tambahan):**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `telegram_chat_id` | Telegram Chat ID | Char | – | – | unik, numerik (boleh negatif untuk grup) | groups HR/Manager |
| `telegram_username` | Username Telegram | Char | – | – | – | info |
| `telegram_link_code` | Kode penautan | Char | – | – | unik | copy=False, groups system/HR |
| `telegram_link_expires_at` | Kode berlaku sampai | Datetime | – | – | – | |
| `telegram_linked_at` | Terhubung sejak | Datetime | – | – | – | |
| `telegram_status` | Status Telegram | Selection compute (`not_linked`,`pending`,`linked`) | – | – | – | |
| `notify_qc_fail` | Notif QC gagal | Boolean | – | True | – | |
| `notify_mo` | Notif MO | Boolean | – | True | – | |
| `notify_expiry` | Notif kedaluwarsa | Boolean | – | False | – | |

**Spesifikasi field — `keripik.telegram.log`:** `employee_id` (M2o), `chat_id` (Char), `event_type` (Selection: `qc_fail`, `mo_done`, `mo_late`, `expiry`, `link`, `other`), `message` (Text), `state` (`sent`/`failed`/`skipped`), `error` (Char), `res_model`, `res_id`.

**State penautan:**
```mermaid
stateDiagram-v2
    [*] --> not_linked
    not_linked --> pending: Generate kode
    pending --> not_linked: Kode kedaluwarsa (15 menit)
    pending --> linked: /start kode valid
    linked --> not_linked: Putuskan / 403 diblokir
    linked --> pending: Generate kode baru (ganti akun)
```

**Catatan tampilan:** tab **Telegram** di form karyawan: status badge, tombol Hubungkan/Putuskan, kode besar + link + instruksi, checkbox preferensi. Format pesan contoh:
```
❌ QC GAGAL — WH/MO/00012
Operasi: Penggorengan (Penggoreng 1)
Suhu minyak: 186,0 °C (batas 160–180)
Diperiksa: Sari, 09:41
```

**Acceptance criteria:**
- **AC-TG-01** Given karyawan klik Hubungkan, When kode dibuat, Then kode 8 karakter dan link t.me tampil, status `pending`.
- **AC-TG-02** Given kode valid, When karyawan mengirim `/start KODE` ke bot dan poller berjalan, Then dalam ≤ 1 menit (poller Odoo) chat_id tersimpan, status `linked`, dan bot membalas sapaan dengan nama karyawan.
- **AC-TG-03** Given kode berumur > 15 menit, When `/start KODE`, Then bot membalas kode kedaluwarsa dan chat_id tidak tersimpan.
- **AC-TG-04** Given PPIC terhubung, When QC manual gagal, Then PPIC menerima pesan dalam ≤ 10 detik setelah simpan.
- **AC-TG-05** Given transaksi QC gagal di-rollback karena error lain, Then tidak ada pesan Telegram terkirim.
- **AC-TG-06** Given karyawan memblokir bot, When notifikasi dikirim, Then chat_id dikosongkan dan log `failed` dengan error 403.
- **AC-TG-07** Given MO melewati deadline, When cron MO tertunda berjalan dua kali dalam sehari, Then hanya satu pesan untuk MO itu.

---

### 3(n) REST API Odoo

#### FR-API-01 — Autentikasi token
Header `Authorization: Bearer <token>`. Token di-hash SHA-256 dan dibandingkan dengan `keripik.api.token.token_hash` (perbandingan constant-time `hmac.compare_digest`). Token menentukan user eksekusi dan scope.

#### FR-API-02 — `POST /api/keripik/v1/workorders/<id>/sensor-summary` (scope gateway)
#### FR-API-03 — `POST /api/keripik/v1/quality-checks/auto-fail` (scope gateway)
#### FR-API-04 — `POST /api/keripik/v1/telegram/link` (scope gateway)
#### FR-API-05 — `GET /api/keripik/v1/production/today` (gateway, monitor)
#### FR-API-06 — `GET /api/keripik/v1/productions` (monitor)
#### FR-API-07 — `GET /api/keripik/v1/yield` (monitor)
#### FR-API-08 — `GET /api/keripik/v1/quality-checks` (monitor)
#### FR-API-09 — `GET /api/keripik/v1/workcenters` (gateway)
#### FR-API-11 — `GET /api/keripik/v1/recipients?event=anomaly` (gateway)
#### FR-API-10 — Event outbound ke gateway
Model `keripik.gateway.event` + cron tiap 1 menit mengirim event `pending` ke `POST {gateway_url}/api/v1/events/workorder`; juga dicoba langsung setelah commit. Retry dengan backoff (1, 2, 4, 8 … maks 60 menit), `failed` setelah 10 percobaan.

Spesifikasi request/response lengkap: **D6 §6.2**.

**Aktor:** Go Gateway, Next.js Monitor (sistem), Admin (mengelola token).

**Prasyarat:** token dibuat di *Settings → Technical → Keripik API Tokens* (token mentah tampil sekali di wizard).

**Pemicu:** HTTP request.

**Alur utama (umum):**
1. Request masuk → decorator `@keripik_api(scope='gateway')`.
2. Ambil header, hash, cari token aktif → tidak ada: 401.
3. Cek scope → tidak cocok: 403.
4. Ganti environment ke user token (`request.update_env(user=token.user_id)`) ⚠️ (API ini ada sejak Odoo 16/17).
5. Parse JSON body → validasi field wajib → 422 bila salah.
6. Jalankan logika model (method di model, bukan di controller) → respons JSON standar.
7. Update `last_used_at` (maks sekali per menit untuk mengurangi write).

**Alur alternatif:** A1: request idempoten diulang (`external_ref` sudah ada) → 200 `duplicate: true` + data yang sama.

**Alur error:** 400/401/403/404/409/422/500 (D6 §6.1). Exception tak terduga → log `_logger.exception`, respons 500 tanpa stack trace.

**Aturan bisnis:**
- BR-API-01: Controller tipis; logika di model (`keripik.quality.check.api_auto_fail(vals)`), sehingga bisa diuji tanpa HTTP.
- BR-API-02: Semua endpoint `csrf=False`, `auth='none'`, `save_session=False` (tidak membuat session cookie) ⚠️.
- BR-API-03: Token mentah tidak pernah disimpan; rotasi = buat token baru, arsipkan yang lama.
- BR-API-04: User teknis `gateway_bot`/`monitor_bot` adalah internal user dengan grup minimal; bukan admin.
- BR-API-05: Respons list dibatasi `limit` maksimum 200.

**Spesifikasi field — `keripik.api.token`:**

| Nama teknis | Label | Tipe | Wajib | Default | Validasi | Keterangan |
|---|---|---|---|---|---|---|
| `name` | Nama | Char | ✔ | – | – | |
| `user_id` | User eksekusi | Many2one `res.users` | ✔ | – | bukan superuser | |
| `scope` | Scope | Selection(`gateway`,`monitor`) | ✔ | – | – | |
| `token_hash` | Hash | Char | ✔ | – | unik | groups system, readonly |
| `token_prefix` | Prefix | Char | – | – | – | tampil di list |
| `last_used_at` | Terakhir dipakai | Datetime | – | – | – | |
| `active` | Aktif | Boolean | – | True | – | |

**Spesifikasi field — `keripik.gateway.event`:** `event_type` (Selection, wajib), `workorder_id` (M2o, wajib), `ref` (Char unik, UUID), `payload` (Text JSON), `state` (`pending`/`sent`/`failed`, default `pending`), `attempts` (Integer, 0), `next_attempt_at` (Datetime), `last_error` (Char).

**State event outbound:**
```mermaid
stateDiagram-v2
    [*] --> pending
    pending --> sent: HTTP 2xx
    pending --> pending: gagal, attempts++, backoff
    pending --> failed: attempts >= 10
    failed --> pending: Retry manual (Manager)
    sent --> [*]
```

**Catatan tampilan:** menu *Settings → Technical → Keripik → API Tokens*, *Gateway Events* (list dengan filter state, tombol Retry). Wizard "Buat token" menampilkan token sekali dengan peringatan "Simpan sekarang, token tidak dapat ditampilkan lagi".

**Acceptance criteria:**
- **AC-API-01** Given tanpa header Authorization, When GET `/productions`, Then 401 `UNAUTHORIZED`.
- **AC-API-02** Given token scope monitor, When POST `/quality-checks/auto-fail`, Then 403 `FORBIDDEN_SCOPE`.
- **AC-API-03** Given token gateway dan WO 31 dengan QC suhu `todo`, When POST auto-fail measure_type oil_temp value 186, Then QC itu menjadi `fail` dengan `source = gateway` dan respons 201 berisi `qc_id`.
- **AC-API-04** Given request auto-fail yang sama (external_ref sama) dikirim dua kali, Then hanya satu QC terpengaruh dan respons kedua `duplicate: true`.
- **AC-API-05** Given body JSON tanpa `workorder_id`, Then 422 dengan pesan field yang hilang.
- **AC-API-06** Given POST sensor-summary untuk WO 31, Then `keripik.sensor.summary` tercipta dan QC suhu `todo` WO 31 terisi nilai `temp_avg` (QC berat memakai `weight_avg`) dan hasilnya dihitung sesuai batas quality point. Lonjakan sesaat ditangani detektor anomali, bukan ringkasan.
- **AC-API-07** Given WO dimulai di layar operator dan gateway mati, When gateway hidup lagi, Then event `started` terkirim oleh cron dalam ≤ 2 menit.
- **AC-API-08** Given token diarsipkan, When dipakai, Then 401.

---

### 3(o) Machine Data Gateway (Go)

#### FR-GW-01 — Ingest suhu & berat via HTTP
`POST /api/v1/readings/temperature` dan `/readings/weight` menerima satu/batch pembacaan, memvalidasi, dan merespons `202` dalam < 50 ms (p95) tanpa menunggu tulis DB.

#### FR-GW-02 — Ingest via WebSocket (opsional)
`GET /api/v1/ws/ingest?token=` untuk koneksi persisten simulator.

#### FR-GW-03 — Penyimpanan sensor_db
Batch writer menulis ke `temperature_readings`/`weight_readings` (flush tiap 500 item atau 1 detik) memakai `pgx.CopyFrom` atau batch insert.

#### FR-GW-04 — Pemetaan WO aktif
Event WO dari Odoo (`started`/`paused`/`finished`) memperbarui `wo_intervals` dan cache memori `machine_code → workorder_id`. Setiap pembacaan diberi `odoo_workorder_id` dari cache.

#### FR-GW-05 — Ringkasan WO selesai
Event `finished` → agregasi SQL (avg/min/max/count suhu, detik di atas batas, total/avg/count berat, jumlah di luar toleransi) → outbox `sensor_summary` → POST ke Odoo (FR-API-02).

#### FR-GW-06 — Deteksi anomali suhu
Per mesin fryer: jika `celsius > temp_limit` (default 180) terus-menerus selama ≥ `temp_limit_seconds` (default 60), buat anomali `temp_over_limit`. Anomali ditutup saat suhu ≤ batas selama 10 detik (hysteresis).

#### FR-GW-07 — Deteksi anomali berat
Per kemasan: `grams < weight_min` atau `> weight_max` (195–210) → anomali `weight_out_of_tolerance`. Untuk menghindari spam: jika ≥ 3 kemasan di luar toleransi dalam 5 menit, satu anomali (bukan per kemasan).

#### FR-GW-08 — Aksi anomali
Dalam satu transaksi: insert `anomalies` + outbox `qc_auto_fail`. Langsung kirim Telegram ke daftar chat_id penerima (dari Odoo `GET /recipients?event=anomaly` yang di-cache 5 menit, atau konfigurasi env sebagai fallback).

#### FR-GW-09 — Outbox worker
Ticker 2 detik mengambil outbox `sent_at IS NULL AND next_attempt_at <= now()` (`FOR UPDATE SKIP LOCKED`), kirim, retry exponential backoff dengan jitter (maks 10 menit antar percobaan).

#### FR-GW-10 — Streaming SSE
`GET /api/v1/stream?machine=` mem-broadcast `temperature`, `weight`, `anomaly` ke klien; keep-alive komentar tiap 15 detik; klien lambat di-drop (channel non-blocking).

#### FR-GW-11 — Data historis
`GET /api/v1/machines/{code}/temperature?from&to&bucket` dengan `date_bin`/`time_bucket` agregasi rata-rata per bucket (maks 1.000 titik).

#### FR-GW-12 — Bot Telegram long polling
Bila `TELEGRAM_POLLER=gateway`: goroutine `getUpdates` (timeout 30 dtk, offset tersimpan di `bot_state`). Perintah:
- `/start <kode>` → `POST /telegram/link` ke Odoo → balas hasil.
- `/produksi` → `GET /production/today` → balas ringkasan (hanya untuk chat_id yang terdaftar sebagai karyawan; Owner/PPIC).
- `/help` → daftar perintah.

#### FR-GW-13 — Health & graceful shutdown
`/healthz`, `/readyz`; saat SIGTERM: berhenti menerima request, flush buffer, tutup koneksi DB dalam ≤ 10 detik.

#### FR-GW-14 — Retensi
Job harian menghapus readings > `RETENTION_DAYS` (default 90).

**Aktor:** Mesin/simulator, Odoo (event), Next.js (stream), Telegram.

**Prasyarat:** `sensor_db` termigrasi; tabel `machines` terisi (seed atau sinkron dari `GET /workcenters`); token di `.env`.

**Pemicu:** HTTP request; event; ticker; update Telegram.

**Alur utama:** lihat D3(d).

**Alur alternatif:**
- A1: Pembacaan datang saat tidak ada WO aktif → disimpan dengan `odoo_workorder_id = NULL` (mesin pemanasan), tidak memicu anomali QC (tapi tetap Telegram bila suhu berbahaya > 200 °C, opsional).
- A2: Event `finished` diterima dua kali → idempoten berdasarkan `ref`.

**Alur error:**
- E1: Token mesin salah → 401.
- E2: Payload tidak valid (suhu di luar −20…400, berat di luar 0…5000, ts > 1 menit di masa depan) → 422.
- E3: Channel buffer penuh (DB lambat) → 503 `BACKPRESSURE` agar simulator mengulang (tidak memblok goroutine handler).
- E4: Odoo down → outbox menumpuk dan terkirim saat Odoo kembali.
- E5: Telegram 429 → tunggu `retry_after`.

**Aturan bisnis:**
- BR-GW-01: Gateway tidak pernah menulis langsung ke DB Odoo; hanya lewat REST.
- BR-GW-02: Satu anomali berkelanjutan = satu QC gagal + satu Telegram (dedup per `machine_code + workorder_id + kind` selama status open).
- BR-GW-03: Semua waktu disimpan UTC (`timestamptz`).
- BR-GW-04: Konfigurasi lewat environment variable (12-factor), tidak ada secret di kode.
- BR-GW-05: Log terstruktur JSON (`log/slog`) dengan field `machine_code`, `workorder_id`, `ref`.

**Spesifikasi field — payload & tabel:** lihat D6 §6.3 dan D4 §4.3. Konfigurasi env:

| Variabel | Wajib | Default | Keterangan |
|---|---|---|---|
| `GATEWAY_ADDR` | – | `:8080` | |
| `SENSOR_DB_URL` | ✔ | – | `postgres://sensor:...@sensor-db:5432/sensor_db` |
| `ODOO_BASE_URL` | ✔ | – | `http://odoo:8069` |
| `ODOO_API_TOKEN` | ✔ | – | token scope gateway |
| `GATEWAY_INBOUND_TOKEN` | ✔ | – | dipakai Odoo → Go |
| `MONITOR_TOKEN` | ✔ | – | dipakai Next.js → Go |
| `TELEGRAM_BOT_TOKEN` | – | – | kosong = Telegram nonaktif |
| `TELEGRAM_POLLER` | – | `off` | `gateway` untuk menjalankan bot |
| `TELEGRAM_ALERT_CHAT_IDS` | – | – | fallback, dipisah koma |
| `INGEST_BUFFER` | – | `10000` | ukuran channel |
| `BATCH_SIZE` / `BATCH_INTERVAL` | – | `500` / `1s` | |
| `RETENTION_DAYS` | – | `90` | |

**State anomali:**
```mermaid
stateDiagram-v2
    [*] --> candidate: suhu > batas (state memori)
    candidate --> [*]: kembali normal sebelum 60 dtk
    candidate --> open: > 60 dtk, insert anomalies + outbox
    open --> open: update peak_value
    open --> closed: normal >= 10 dtk
    closed --> [*]
```

**State outbox:**
```mermaid
stateDiagram-v2
    [*] --> pending
    pending --> sent: 2xx dari Odoo
    pending --> pending: error, attempts++, next_attempt_at = backoff
    pending --> dead: attempts >= 15 (log error, tampil di /anomalies)
    sent --> [*]
```

**Catatan tampilan:** tidak ada UI; log terstruktur; endpoint `GET /api/v1/anomalies` untuk Monitor.

**Acceptance criteria:**
- **AC-GW-01** Given token mesin valid, When POST 1 pembacaan suhu, Then respons 202 dan dalam ≤ 2 detik baris ada di `temperature_readings`.
- **AC-GW-02** Given simulator mengirim 2 mesin × 1 data/2 dtk selama 10 menit, Then 600 baris tersimpan tanpa kehilangan data dan p95 latensi ingest < 50 ms.
- **AC-GW-03** Given WO 31 started di FRY-01, When pembacaan masuk, Then `odoo_workorder_id = 31`.
- **AC-GW-04** Given suhu 185 °C selama 75 detik pada WO 31, Then tepat satu anomali open, satu Telegram, dan satu QC gagal di Odoo.
- **AC-GW-05** Given suhu 185 °C selama 40 detik lalu turun, Then tidak ada anomali.
- **AC-GW-06** Given Odoo dimatikan saat anomali, When Odoo hidup kembali, Then QC gagal tercatat (outbox terkirim) tanpa duplikat.
- **AC-GW-07** Given event finished WO 31, Then dalam ≤ 5 detik `keripik.sensor.summary` WO 31 ada di Odoo dengan avg/min/max sesuai data.
- **AC-GW-08** Given klien SSE terhubung, When pembacaan masuk, Then klien menerima event `temperature` dalam ≤ 1 detik.
- **AC-GW-09** Given karyawan terhubung mengirim `/produksi`, Then bot membalas ringkasan hari ini; given chat tak dikenal, Then bot membalas "Akses ditolak".
- **AC-GW-10** Given container menerima SIGTERM, Then data di buffer ter-flush dan proses keluar dengan kode 0 dalam ≤ 10 detik.

---

### 3(p) Simulator Mesin

#### FR-SIM-01 — Simulasi penggoreng
Untuk setiap mesin fryer: suhu awal 25 °C → pemanasan ke ~170 °C (kurva eksponensial) → stabil 165–178 °C dengan noise ±2 °C; kirim tiap 2 detik.

#### FR-SIM-02 — Simulasi timbangan
Berat kemasan normal mean 201 g, std dev 2 g; satu kemasan tiap 3 detik (bisa diatur).

#### FR-SIM-03 — Skenario
Flag `-scenario`: `normal`, `overheat` (setelah N menit suhu naik ke 185–190 °C selama 90 detik), `spike` (lonjakan 40 detik, tidak boleh memicu anomali), `underweight` (5 kemasan berturut 190 g), `burst` (load test 20 mesin, 1 data/100 ms).

#### FR-SIM-04 — Kecepatan waktu
Flag `-speed` (misal 5×) mempercepat **fase pemanasan dan waktu tunggu sebelum event skenario** untuk demo cepat. Event anomali dan semua timestamp tetap **waktu nyata**, sehingga aturan 60 detik di gateway (dan validasi `ts` tidak lebih dari 1 menit di masa depan) tetap teruji.

#### FR-SIM-05 — Ketahanan
Bila gateway merespons 503/timeout, retry dengan backoff; tampilkan statistik terkirim/gagal tiap 10 detik.

**Aktor:** Developer/penguji.

**Prasyarat:** gateway berjalan; token mesin.

**Pemicu:** `go run ./cmd/simulator -scenario overheat`.

**Alur utama:** parse flag → goroutine per mesin → loop ticker → bangun payload → POST → log statistik.

**Alur error:** E1: gateway tidak terjangkau → retry, tidak crash. E2: flag tidak valid → pesan usage, exit 2.

**Aturan bisnis:** BR-SIM-01: simulator tidak tahu WO; pemetaan WO dilakukan gateway (realistis: mesin tidak mengenal ERP).

**Spesifikasi flag:**

| Flag | Tipe | Default | Keterangan |
|---|---|---|---|
| `-url` | string | `http://localhost:8080` | base URL gateway |
| `-token` | string | env `MACHINE_TOKEN` | |
| `-fryers` | string | `FRY-01,FRY-02` | |
| `-scale` | string | `SCL-01` | kosong = tanpa timbangan |
| `-interval` | duration | `2s` | interval suhu |
| `-pack-interval` | duration | `3s` | |
| `-scenario` | string | `normal` | |
| `-overheat-after` | duration | `2m` | |
| `-speed` | float | `1` | |
| `-duration` | duration | `0` (tanpa batas) | |

**State mesin simulasi:**
```mermaid
stateDiagram-v2
    [*] --> heating
    heating --> stable: suhu >= 165
    stable --> overheat: skenario overheat
    overheat --> stable: setelah 90 dtk
    stable --> [*]: durasi habis / Ctrl+C
```

**Acceptance criteria:**
- **AC-SIM-01** Given skenario normal 5 menit, Then tidak ada anomali di gateway.
- **AC-SIM-02** Given skenario overheat, Then gateway membuat tepat satu anomali suhu.
- **AC-SIM-03** Given skenario spike, Then tidak ada anomali.
- **AC-SIM-04** Given skenario underweight, Then satu anomali berat.
- **AC-SIM-05** Given gateway dimatikan 30 detik lalu hidup, Then simulator melanjutkan tanpa crash.

---

### 3(q) Monitor Produksi (Next.js)

#### FR-MON-01 — Login pemilik
Halaman `/monitor/login` dengan password (env `MONITOR_PASSWORD`, disimpan hash di env/compare constant-time); cookie httpOnly `monitor_session` (JWT/HMAC) berlaku 7 hari.

#### FR-MON-02 — KPI hari ini
Kartu: MO aktif, MO selesai hari ini, yield rata-rata, QC gagal hari ini (dari `GET /production/today`).

#### FR-MON-03 — Tabel MO berjalan
Dari `GET /productions?state=progress,confirmed`: nomor, produk, qty, progres WO, deadline (merah bila lewat), `qc_state`.

#### FR-MON-04 — Daftar QC gagal
Dari `GET /quality-checks?result=fail&date_from=today-7`: waktu, MO, titik QC, nilai, sumber (badge 🤖 gateway).

#### FR-MON-05 — Grafik suhu live
Line chart suhu per mesin (dropdown), garis batas 180 °C, data awal 15 menit dari `/machines/{code}/temperature`, lalu live via SSE. Indikator koneksi.

#### FR-MON-06 — Notifikasi anomali di layar
Event SSE `anomaly` → banner merah "Suhu FRY-01 186 °C" sampai ditutup.

#### FR-MON-07 — Auto refresh
Data Odoo di-revalidate tiap 30 detik (`router.refresh()` di client timer atau `revalidate`).

#### FR-MON-08 — Responsif
Nyaman di ponsel (pemilik memantau dari HP).

**Aktor:** Pemilik.

**Prasyarat:** token monitor di env server Next.js; Go & Odoo terjangkau dari container Next.js.

**Pemicu:** buka `/monitor`.

**Alur utama:** login → middleware cek cookie → Server Component fetch Odoo (paralel `Promise.all`) → render → Client Component membuka `EventSource('/monitor/api/stream?machine=FRY-01')` → route handler memvalidasi cookie dan mem-proxy stream Go dengan header `Authorization` monitor.

**Alur alternatif:** A1: Go mati (F7, sebelum F9) → grafik menampilkan "Gateway belum aktif", bagian lain tetap jalan.

**Alur error:** E1: Odoo error/timeout → kartu menampilkan "Data tidak tersedia" + waktu terakhir sukses (tidak crash seluruh halaman, pakai `error.tsx`/try-catch per bagian). E2: password salah → pesan, delay 1 detik (anti brute force sederhana). E3: SSE terputus → `EventSource` reconnect otomatis, indikator kuning.

**Aturan bisnis:**
- BR-MON-01: Token Odoo/Go tidak pernah dikirim ke browser.
- BR-MON-02: `basePath = '/monitor'`; semua link & fetch internal memperhitungkan basePath.
- BR-MON-03: Buffer grafik maksimal 450 titik per mesin.

**Spesifikasi field (form login & state UI):**

| Nama | Label | Tipe | Wajib | Validasi | Keterangan |
|---|---|---|---|---|---|
| `password` | Password | password | ✔ | min 8 | |
| `machine` | Mesin | select | ✔ | dari `/machines` | default FRY-01 |
| `connection` | Status | enum `connecting`/`open`/`error` | – | – | indikator |

**State koneksi SSE:**
```mermaid
stateDiagram-v2
    [*] --> connecting
    connecting --> open: onopen
    open --> error: onerror
    error --> connecting: auto retry
    open --> [*]: unmount
```

**Catatan tampilan:** lihat D6 §6.4C. Tailwind, dark mode opsional, angka format `id-ID`.

**Acceptance criteria:**
- **AC-MON-01** Given belum login, When membuka `/monitor`, Then diarahkan ke `/monitor/login`.
- **AC-MON-02** Given login benar, Then dashboard menampilkan KPI sesuai `GET /production/today`.
- **AC-MON-03** Given membuka DevTools → Network, Then tidak ada request langsung ke `/api/keripik/v1` dari browser dan tidak ada token di bundle JS.
- **AC-MON-04** Given simulator berjalan, Then grafik bergerak tiap ~2 detik.
- **AC-MON-05** Given skenario overheat, Then banner anomali muncul ≤ 2 detik setelah gateway mendeteksi.
- **AC-MON-06** Given Odoo dimatikan, Then halaman tetap tampil dengan pesan "Data tidak tersedia" pada kartu terkait.
- **AC-MON-07** Given layar 390 px, Then layout menjadi satu kolom tanpa scroll horizontal.

---

## 4. Kebutuhan Non-Fungsional

### 4.1 Keamanan

| ID | Kebutuhan |
|---|---|
| NFR-SEC-01 | Setiap model custom punya baris `ir.model.access.csv`; tidak ada model tanpa ACL (Odoo memberi warning saat install). |
| NFR-SEC-02 | Record rule multi-company untuk model custom. |
| NFR-SEC-03 | Token API di-hash SHA-256, dibandingkan constant-time, punya scope, bisa dicabut. |
| NFR-SEC-04 | Semua secret di `.env` (tidak di-commit), `ir.config_parameter` hanya terbaca grup System. |
| NFR-SEC-05 | `list_db = False`, `admin_passwd` kuat, `dbfilter` di produksi; database manager tidak terekspos via ngrok. |
| NFR-SEC-06 | Hanya Nginx yang mengekspos port; DB tidak punya port publik. |
| NFR-SEC-07 | Controller tidak mengembalikan stack trace; input divalidasi (tipe, rentang). |
| NFR-SEC-08 | `sudo()` hanya di titik yang dijelaskan (traceability, API link) dan selalu setelah validasi hak/token. |
| NFR-SEC-09 | Next.js: cookie httpOnly, `Secure` di HTTPS, `SameSite=Lax`. |
| NFR-SEC-10 | Escape HTML pada pesan Telegram (parse_mode HTML) dan QWeb (`t-esc`/`t-out` otomatis escape). |

### 4.2 Performa & Volume Data

| ID | Kebutuhan |
|---|---|
| NFR-PERF-01 | Volume dasar: 2 fryer × 0,5 Hz × 8 jam = 14.400 baris suhu/hari; ±1.500 berat/hari; ±0,5 juta baris/bulan. |
| NFR-PERF-02 | Gateway sanggup ≥ 200 pembacaan/detik (skenario burst 20 mesin × 10 Hz) dengan p95 ingest < 50 ms di laptop dev. |
| NFR-PERF-03 | Ukuran `sensor_db` ±50–80 MB/bulan (perkiraan ±100–150 byte/baris termasuk index); retensi 90 hari. |
| NFR-PERF-04 | Odoo hanya menerima ±1 ringkasan per WO + anomali (puluhan request/hari dari gateway). |
| NFR-PERF-05 | Layar operator: aksi tombol < 1 detik di jaringan lokal; polling ringan (≤ 1 request/15 detik/tablet). |
| NFR-PERF-06 | Laporan yield memakai SQL view ber-index (`stock_move.raw_material_production_id`, `state`); pivot bulan < 2 detik untuk 1.000 MO. |
| NFR-PERF-07 | Monitor: TTFB < 1,5 detik; SSE < 1 detik latensi end-to-end. |

### 4.3 Kompatibilitas Tablet & Browser

| ID | Kebutuhan |
|---|---|
| NFR-UX-01 | Layar operator diuji di Chrome Android & Safari iPadOS terbaru, resolusi 1024×768 dan 1280×800. |
| NFR-UX-02 | Target sentuh ≥ 48×48 px (tombol utama 56 px), font ≥ 16 px. |
| NFR-UX-03 | Tidak bergantung hover/klik kanan/keyboard shortcut. |
| NFR-UX-04 | Bahasa antarmuka Indonesia (file `i18n/id.po`) dengan string sumber Inggris. |

### 4.4 Batasan Odoo Community

| ID | Batasan | Dampak |
|---|---|---|
| NFR-CE-01 | Tidak ada Quality, Shop Floor, IoT, MPS, PLM, Studio, Barcode, Accounting penuh | Diganti custom module / Go / di luar cakupan |
| NFR-CE-02 | Tidak ada laporan cost analysis MO Enterprise | FR-CST custom |
| NFR-CE-03 | Module OCA untuk 19.0 mungkin belum lengkap | Dievaluasi, tidak dijadikan ketergantungan wajib |
| NFR-CE-04 | Upgrade major (19 → 20) tanpa layanan upgrade Odoo SA | Gunakan OpenUpgrade (OCA) untuk data & migrasi kode manual (F12) |

### 4.5 Batasan Telegram Bot API

| ID | Batasan |
|---|---|
| NFR-TG-01 | Bot tidak bisa memulai chat; user harus menekan Start lebih dulu. |
| NFR-TG-02 | `getUpdates` dan webhook saling eksklusif; satu konsumen `getUpdates` per token. |
| NFR-TG-03 | Update disimpan server Telegram terbatas waktu (setahuku ±24 jam) bila tidak diambil. |
| NFR-TG-04 | Rate limit (pedoman): ±1 pesan/dtk per chat, ±30 pesan/dtk global, ±20 pesan/menit ke grup. |
| NFR-TG-05 | Panjang pesan maks 4.096 karakter; payload deep link `start` maks 64 karakter. |
| NFR-TG-06 | Butuh akses internet keluar ke `api.telegram.org`. |

> ⚠️ Angka di 4.5 dari pengetahuanku; **cek** <https://core.telegram.org/bots/api> dan <https://core.telegram.org/bots/faq>.

### 4.6 Keandalan & Observabilitas

| ID | Kebutuhan |
|---|---|
| NFR-REL-01 | Integrasi antar-sistem memakai outbox + retry; tidak ada event hilang bila salah satu layanan restart. |
| NFR-REL-02 | Semua endpoint idempoten untuk event berulang (`external_ref`/`ref`). |
| NFR-REL-03 | Healthcheck Docker untuk db, sensor-db, odoo, gateway. |
| NFR-REL-04 | Log terstruktur di Go; `_logger` di Odoo dengan konteks ID. |
| NFR-REL-05 | Unit test Odoo (`odoo-bin --test-tags /mrp_keripik`) dan Go (`go test ./...`) lulus sebelum deploy. |

---

## 5. Asumsi, Ketergantungan, dan Di Luar Cakupan

### 5.1 Asumsi
- Satu perusahaan, satu warehouse, mata uang IDR, zona waktu Asia/Jakarta.
- Mesin disimulasikan; tidak ada perangkat fisik.
- Tablet operator terhubung ke jaringan lokal/ngrok dengan login Odoo masing-masing.
- Harga bahan tersedia dari PO (`standard_price`).
- Volume rendah (UMKM): ≤ 20 MO/hari.

### 5.2 Ketergantungan
- Odoo 19 Community image resmi; module standar: `sale_management`, `website_sale`, `im_livechat`, `website_livechat`, `mrp`, `sale_mrp`, `purchase_stock`, `mrp_account` ⚠️, `product_expiry`, `maintenance`, `hr`, `account`.
- Telegram Bot API (internet), ngrok (akun gratis).
- Docker & Docker Compose v2.
- Library Go: Fiber, pgx; Next.js & Recharts.

### 5.3 Di Luar Cakupan (pengembangan berikutnya)
- Notifikasi **WhatsApp** (WhatsApp Cloud API / BSP) untuk pelanggan.
- Integrasi payment gateway Indonesia (Midtrans/Xendit) di eCommerce.
- Integrasi kurir/ongkir (RajaOngkir).
- Accounting penuh (jurnal, laporan keuangan) — butuh Enterprise atau OCA `account_financial_report`.
- MQTT produksi dengan TLS & autentikasi per perangkat (hanya demo opsional).
- Aplikasi mobile native; mode offline layar operator.
- Penjadwalan kapasitas (MPS) & optimasi urutan produksi.
- Multi-company / multi-warehouse.
- Machine learning prediksi kualitas.

---

## 6. Matriks Keterlacakan

| ID requirement | Fase | Cara uji |
|---|---|---|
| FR-SAL-01…05 | F1 | Manual: AC-SAL-01…05 (checklist F1) |
| FR-MRP-01…06 | F1 | Manual: AC-MRP-01…05 |
| FR-INV-01…03, FR-PUR-01…03 | F1 | Manual: AC-INV-01…05 |
| FR-MNT-01…03 | F1 | Manual: AC-MNT-01…02 |
| FR-INVC-01…02 | F1 | Manual: AC-INVC-01…02 |
| FR-EML-04 (SMTP) | F1 | Manual: email uji di Mailpit |
| FR-QC-01…09 | F2 | Unit test `test_quality_check.py` (AC-QC-01…07, 09, 10) + manual AC-QC-08 (setelah F6) |
| FR-YLD-01…05 | F3 | Unit test SQL view + manual PDF (AC-YLD-01…05) |
| FR-TRC-01…04 | F3 | Unit test penelusuran + manual wizard (AC-TRC-01…04) |
| FR-OPS-01…08 | F4 | Manual di tablet/DevTools device mode (AC-OPS-01…07); opsional tour test |
| FR-API-01…09, 11 | F5 | `HttpCase` test `test_api.py` + curl/Bruno (AC-API-01…06, 08) |
| FR-API-10 | F5 (model), F9 (end-to-end) | AC-API-07 |
| FR-CST-01…04 | F5 | Unit test `test_batch_cost.py` (AC-CST-01…04) |
| FR-EXP-01…04 | F5 | Unit test `test_expiry_cron.py` + Run Manually (AC-EXP-01…04) |
| FR-EML-01…03 | F5 | Unit test (`mail.mail` dibuat) + Mailpit (AC-EML-01…04) |
| FR-TG-01…06 | F6 | Unit test dengan mock `requests` + uji bot sungguhan (AC-TG-01…07) |
| FR-MON-01…04, 07, 08 | F7 | Manual + Playwright opsional (AC-MON-01…03, 06, 07) |
| Deploy v1 (Nginx, ngrok) | F7 | Akses publik via domain ngrok |
| FR-GW-01…14 | F9 | `go test ./...` (detector, outbox) + simulator (AC-GW-01…10) |
| FR-SIM-01…05 | F9 | Jalankan skenario (AC-SIM-01…05) |
| FR-MON-05…06 | F10 | Simulator + browser (AC-MON-04…05) |
| NFR-PERF-02 | F10 | Skenario burst + catat p95 dari log/`hey`/`k6` |
| NFR-SEC-* | F2, F5, F7, F10 | Checklist keamanan F10 |
| NFR-CE-04 | F12 | Migrasi & test ulang di Odoo 20 |
