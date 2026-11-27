# D6. Spesifikasi API, Layar Utama, dan Struktur Repo

## 6.1 Konvensi Umum API

| Hal | Odoo REST (`mrp_keripik`) | Go Gateway |
|---|---|---|
| Base URL internal | `http://odoo:8069/api/keripik/v1` | `http://gateway:8080/api/v1` |
| Base URL publik (lewat Nginx) | `https://<domain>/api/keripik/v1` | `https://<domain>/gateway/api/v1` |
| Format | JSON UTF-8 | JSON UTF-8, SSE untuk stream |
| Auth | `Authorization: Bearer <token>` (tabel `keripik.api.token`, punya scope) | Mesin: `X-Machine-Token`; Odoo→Go: `Authorization: Bearer <GATEWAY_INBOUND_TOKEN>` |
| Waktu | ISO-8601 UTC (`2026-10-12T03:15:00Z`) | sama |
| Idempotensi | Header `Idempotency-Key` atau field `external_ref` | `ref` UUID di setiap event |

### Bentuk respons

```json
// success
{ "ok": true, "data": { } }

// error
{ "ok": false, "error": { "code": "QC_NOT_FOUND", "message": "Quality check 42 not found" } }
```

| HTTP | Kapan |
|---|---|
| 200 / 201 | Sukses (201 untuk create) |
| 200 + `"duplicate": true` | Request idempoten diulang (tidak membuat data baru) |
| 400 | JSON tidak valid |
| 401 | Token tidak ada / salah |
| 403 | Token valid tapi scope tidak cukup |
| 404 | Resource tidak ditemukan |
| 409 | Konflik state (misal WO sudah selesai) |
| 422 | Validasi gagal (field wajib, nilai di luar rentang) |
| 500 | Error tak terduga (detail di log, bukan di respons) |

> **Odoo vs Frappe:** di Frappe kamu tinggal `@frappe.whitelist()` lalu panggil `/api/method/...`; auth & JSON diurus framework. Di Odoo, `@http.route(type='http', auth='none', csrf=False)` membuatmu **mengurus sendiri** auth token, parsing JSON, dan bentuk respons. Ini justru yang ingin ditunjukkan di portofolio Backend & Integration.
> ⚠️ Odoo 19: setahuku `type='json'` di-rename menjadi `type='jsonrpc'` dan ada `auth='bearer'`. Kita pakai `type='http'` + `request.make_json_response()` untuk REST murni. **Cek `odoo/http.py` di 19.0.**

---

## 6.2 REST Controller Odoo (`/api/keripik/v1`)

### Scope `gateway` (dipakai Go)

#### `POST /workorders/{wo_id}/sensor-summary`
Menyimpan ringkasan sensor ketika WO selesai, lalu mengisi otomatis QC `oil_temp` / `pack_weight` milik WO itu yang `source_allowed` ∈ {gateway, both} dan masih `todo`.

Request:
```json
{
  "external_ref": "sum-7f3c2a9e",
  "machine_code": "FRY-01",
  "period_start": "2026-10-12T02:00:00Z",
  "period_end": "2026-10-12T03:00:00Z",
  "temperature": { "avg": 172.4, "min": 165.0, "max": 183.1, "samples": 1800, "seconds_over_limit": 24 },
  "weight": null
}
```
Response `201`:
```json
{ "ok": true, "data": { "summary_id": 15, "qc_updated": [{ "id": 88, "result": "pass" }] } }
```
Error: `404 WO_NOT_FOUND`, `409 WO_NOT_DONE` (opsional: diterima juga bila WO sedang `progress`), `422 INVALID_PAYLOAD`. `external_ref` sama dikirim ulang → `200 duplicate`.

#### `POST /quality-checks/auto-fail`
```json
{
  "external_ref": "a1b2c3d4-...",
  "workorder_id": 31,
  "measure_type": "oil_temp",
  "value": 186.2,
  "message": "Oil temperature above 180.0 C for 75 s on FRY-01"
}
```
Logika: cari QC `todo` untuk WO + `measure_type` → isi `fail`; jika tidak ada, buat QC baru (`source = gateway`, `result = fail`). Telegram **tidak** dikirim lagi dari Odoo untuk QC dengan `source = gateway` (Go sudah mengirim), untuk menghindari duplikat.
Response `201`: `{ "ok": true, "data": { "qc_id": 90, "production": "WH/MO/00012" } }`

#### `POST /telegram/link`
```json
{ "code": "K7P-93QX", "chat_id": "123456789", "username": "budi_ppic" }
```
`200`: `{ "ok": true, "data": { "employee_name": "Budi" } }` · `404 CODE_INVALID` · `410 CODE_EXPIRED`

#### `GET /production/today`
Untuk `/produksi` (bot) dan Monitor.
```json
{ "ok": true, "data": {
  "date": "2026-10-12",
  "mo_done": 3, "mo_in_progress": 2, "mo_late": 1,
  "qty_done_units": 90,
  "yield_pct_avg": 96.4,
  "qc_fail_today": 1
}}
```

#### `GET /workcenters`
Daftar work center dengan `machine_code`, untuk sinkronisasi tabel `machines` di Go.

#### `GET /recipients?event=anomaly`
Daftar `chat_id` karyawan terhubung yang harus menerima peringatan (PPIC & QC dengan `notify_qc_fail`). Go meng-cache hasilnya 5 menit, jadi Telegram anomali tetap bisa dikirim walau Odoo sedang down.
```json
{ "ok": true, "data": [ { "employee": "Budi", "chat_id": "123456789" } ] }
```

### Scope `monitor` (dipakai Next.js; juga boleh untuk `gateway`)

| Method & path | Query | Data |
|---|---|---|
| `GET /productions` | `state=progress,confirmed` `limit` | `[{id, name, product, qty_plan, qty_produced, state, date_start, date_deadline, qc_state, workorders:[{name, workcenter, state}]}]` |
| `GET /yield` | `date_from`, `date_to` | `{ by_product:[{product, planned, actual, yield_pct}], by_operation:[{workcenter, planned, actual, variance_pct}] }` |
| `GET /quality-checks` | `result=fail`, `date_from`, `limit` | `[{id, name, production, workorder, point, value, unit, source, checked_at, is_resolved}]` |
| `GET /production/today` | – | lihat di atas |

---

## 6.3 Endpoint Go Gateway (`/api/v1`)

| Method & path | Auth | Fungsi |
|---|---|---|
| `GET /healthz` | – | Liveness: `{"status":"ok"}` |
| `GET /readyz` | – | Cek koneksi `sensor_db` |
| `POST /readings/temperature` | `X-Machine-Token` | Satu atau batch pembacaan suhu |
| `POST /readings/weight` | `X-Machine-Token` | Berat kemasan |
| `GET /ws/ingest` | token di query | (opsional) WebSocket ingest |
| `POST /events/workorder` | Bearer inbound | Event dari Odoo: `started`/`paused`/`finished` |
| `GET /machines` | Bearer monitor | Daftar mesin + nilai terakhir |
| `GET /machines/{code}/temperature` | Bearer monitor | `?from=&to=&bucket=10s` → data historis (downsample) untuk grafik awal |
| `GET /stream` | Bearer monitor | SSE: `event: temperature` / `weight` / `anomaly` |
| `GET /anomalies` | Bearer monitor | `?status=open` |

### `POST /readings/temperature`
```json
{ "machine_code": "FRY-01", "readings": [ { "ts": "2026-10-12T02:00:02Z", "celsius": 171.8 } ] }
```
`202 Accepted` → `{ "accepted": 1 }`. Pakai **202**, bukan 201, karena data masuk antrean (channel) dan ditulis ke DB secara batch.

### `POST /readings/weight`
```json
{ "machine_code": "SCL-01", "readings": [ { "ts": "2026-10-12T04:10:00Z", "grams": 201.3 } ] }
```

### `POST /events/workorder` (Odoo → Go)
```json
{
  "ref": "evt-000123",
  "event": "started",
  "workorder_id": 31,
  "production_id": 12,
  "machine_code": "FRY-01",
  "at": "2026-10-12T02:00:00Z"
}
```
`finished` memicu: tutup interval → agregasi SQL → tulis outbox `sensor_summary` → worker POST ke Odoo.

### SSE `GET /stream?machine=FRY-01`
```
event: temperature
data: {"machine_code":"FRY-01","ts":"2026-10-12T02:00:02Z","celsius":171.8,"workorder_id":31}

event: anomaly
data: {"ref":"a1b2...","kind":"temp_over_limit","machine_code":"FRY-01","peak":186.2}

: keep-alive
```

---

## 6.4 Deskripsi Layar Utama

### A. Layar Operator (OWL client action, tablet 10" landscape)

```
┌────────────────────────────────────────────────────────────────────┐
│ ☰ Layar Operator     Work Center: [ Penggoreng 1 ▼ ]   👤 Sari  ⏻ │
├───────────────┬────────────────────────────────────────────────────┤
│ WO ANTREAN    │  WH/MO/00012 · Penggorengan · Keripik Siap Kemas   │
│ ┌───────────┐ │  Status: ● Sedang berjalan   ⏱ 00:42:10            │
│ │MO/00012 ● │ │  Target: 6,00 kg     Sensor: FRY-01                 │
│ │Penggoreng │ │ ┌──────────────┬──────────────┬──────────────┐     │
│ │6 kg  42m  │ │ │ Suhu kini    │ Rata-rata    │ Maks         │     │
│ └───────────┘ │ │ 172,4 °C     │ 170,9 °C     │ 178,2 °C     │     │
│ ┌───────────┐ │ └──────────────┴──────────────┴──────────────┘     │
│ │MO/00013 ○ │ │  QUALITY CHECK                                      │
│ │Ready      │ │  ☐ Suhu minyak (160–180 °C)  [ 172 ] [Simpan]       │
│ └───────────┘ │  ✔ Visual warna keemasan       LULUS                │
│               │  HASIL                                              │
│               │  Qty hasil [ 5,8 ] kg   Scrap [ 0,2 ] kg [Alasan ▼]│
│               │ ┌────────────┐ ┌────────────┐ ┌────────────────┐   │
│               │ │  ▶ MULAI   │ │  ⏸ JEDA    │ │  ✔ SELESAI     │   │
│               │ └────────────┘ └────────────┘ └────────────────┘   │
└───────────────┴────────────────────────────────────────────────────┘
```

- Tombol minimal **56 px** tinggi, font ≥ 18 px, kontras tinggi; tidak bergantung pada hover.
- Kartu WO diberi warna status: hijau = berjalan, abu = ready, kuning = jeda, merah = ada QC gagal.
- Ringkasan sensor di-*poll* tiap 5 detik dari Odoo (`keripik.sensor.summary` terakhir) atau live dari Go (F10, opsional).
- Pesan error tampil sebagai **notification** OWL (toast) dan banner di panel, bukan dialog modal.
- Mode layar penuh (Fullscreen API) dari tombol ☰.

### B. Form Quality Check (backend Odoo + panel di layar operator)

```
QC/2026/00031                                   [ Lulus ] [ Gagal ] [ Retest ]
────────────────────────────────────────────────────────────────────────────
Titik QC     Suhu minyak penggorengan          Status bar: Todo › Pass/Fail
MO           WH/MO/00012                        Sumber   manual
WO           Penggorengan                       Oleh     Sari · 12/10 09:41
Batas        160,0 – 180,0 °C
Nilai ukur   [ 186,0 ]           → Hasil: GAGAL (di atas batas maks)
Catatan *    [ Minyak terlalu panas, api diperkecil ]
Retest       QC/2026/00032 (Lulus)
────────────────────────────────────────────────────────────────────────────
Chatter: perubahan hasil, pengguna, waktu
```

- `note` wajib bila hasil `fail` (constraint Python).
- Tombol **Retest** hanya muncul bila `result = fail` dan belum resolved.
- QC dari gateway tampil dengan badge "🤖 Otomatis" dan field nilai readonly untuk operator.

### C. Next.js Monitor Produksi (`/monitor`)

```
┌──────────────────────────────────────────────────────────────────────┐
│ 🍘 Monitor Produksi Keripik          Senin, 12 Okt 2026   ⟳ 30 dtk   │
├──────────┬──────────┬──────────┬──────────┐                          │
│ MO aktif │ Selesai  │ Yield    │ QC gagal │                          │
│    2     │   3      │  96,4 %  │   1 ⚠    │                          │
├──────────┴──────────┴──────────┴──────────┴──────────────────────────┤
│ Suhu Penggoreng (live)              [FRY-01 ▼]       ● terhubung     │
│  190┤                         ▁▂▃▅▇█▇▅    ---- batas 180 °C ----     │
│  170┤▃▄▅▅▆▅▅▆▅▅▆▅▅▅▆▅▅▆▅▅▆▅▅▆                                         │
│  150┼────────────────────────────────────────── waktu (15 menit)     │
├──────────────────────────────────┬───────────────────────────────────┤
│ MO berjalan                      │ QC gagal terbaru                  │
│ MO/00012 Keripik 200g ▓▓▓▓░ 60%  │ 09:41 Suhu 186 °C · MO/00012 🤖  │
│ MO/00013 Tempe Iris   ▓░░░░ 20%  │ 08:10 Berat 191 g · MO/00010     │
└──────────────────────────────────┴───────────────────────────────────┘
```

- Data Odoo diambil **server-side** (Server Component + `revalidate: 30`), jadi token tidak bocor ke browser.
- Grafik live: Client Component `EventSource('/monitor/api/stream?machine=FRY-01')` → route handler Next.js → Go `/stream`. Menampilkan maksimal 450 titik (15 menit × 30/menit) di buffer.
- Status koneksi SSE: ● terhubung / ○ menyambung ulang.
- Login sederhana (password pemilik dari env) dengan cookie httpOnly.

---

## 6.5 Struktur Folder Repo

```
odoo-project/                          # repo root (monorepo)
├── README.md                          # README portofolio (F10)
├── docker-compose.yml                 # semua layanan
├── docker-compose.override.yml        # khusus dev (port debug, --dev)
├── .env.example                       # salin ke .env (jangan commit .env)
├── .gitignore
├── Makefile                           # shortcut: make up, make logs, make update-module
├── docs/
│   ├── README.md                      # indeks dokumen
│   ├── INSTALL.md                     # langkah instalasi ringkas
│   ├── design/                        # D1–D8 (termasuk FSD)
│   ├── implementation/                # F1–F12
│   └── screenshots/
├── nginx/
│   └── default.conf
├── odoo/
│   ├── config/odoo.conf
│   └── addons/
│       └── mrp_keripik/
│           ├── __init__.py
│           ├── __manifest__.py
│           ├── controllers/
│           │   ├── __init__.py
│           │   ├── auth.py            # token decorator & json helpers
│           │   └── api.py             # /api/keripik/v1/*
│           ├── data/
│           │   ├── ir_sequence_data.xml
│           │   ├── ir_cron_data.xml
│           │   ├── ir_config_parameter_data.xml
│           │   ├── mail_template_data.xml
│           │   └── keripik_demo.xml   # quality point contoh
│           ├── models/
│           │   ├── __init__.py
│           │   ├── quality_point.py
│           │   ├── quality_check.py
│           │   ├── mrp_production.py
│           │   ├── mrp_workorder.py
│           │   ├── mrp_workcenter.py
│           │   ├── stock_picking.py
│           │   ├── stock_lot.py        # cron expiry
│           │   ├── hr_employee.py
│           │   ├── sensor_summary.py
│           │   ├── api_token.py
│           │   ├── gateway_event.py
│           │   ├── telegram_service.py # AbstractModel keripik.telegram
│           │   ├── telegram_log.py
│           │   └── yield_report.py     # SQL view
│           ├── report/
│           │   ├── __init__.py
│           │   ├── report_actions.xml
│           │   ├── yield_report_templates.xml
│           │   ├── traceability_report.py
│           │   └── traceability_report_templates.xml
│           ├── security/
│           │   ├── mrp_keripik_groups.xml
│           │   ├── mrp_keripik_rules.xml
│           │   └── ir.model.access.csv
│           ├── static/
│           │   ├── description/icon.png
│           │   └── src/operator_screen/
│           │       ├── operator_screen.js
│           │       ├── operator_screen.xml
│           │       ├── operator_screen.scss
│           │       └── components/
│           │           ├── workorder_card.{js,xml}
│           │           ├── workorder_panel.{js,xml}
│           │           ├── quality_check_form.{js,xml}
│           │           └── sensor_summary.{js,xml}
│           ├── tests/
│           │   ├── __init__.py
│           │   ├── common.py
│           │   ├── test_quality_check.py
│           │   ├── test_batch_cost.py
│           │   ├── test_expiry_cron.py
│           │   └── test_api.py
│           ├── views/
│           │   ├── quality_point_views.xml
│           │   ├── quality_check_views.xml
│           │   ├── mrp_production_views.xml
│           │   ├── mrp_workorder_views.xml
│           │   ├── mrp_workcenter_views.xml
│           │   ├── hr_employee_views.xml
│           │   ├── sensor_summary_views.xml
│           │   ├── api_token_views.xml
│           │   ├── yield_report_views.xml
│           │   ├── operator_screen_action.xml
│           │   └── menus.xml
│           └── wizard/
│               ├── __init__.py
│               ├── recall_wizard.py
│               └── recall_wizard_views.xml
├── gateway/                           # Go module
│   ├── go.mod
│   ├── Dockerfile
│   ├── cmd/
│   │   ├── gateway/main.go            # entrypoint service
│   │   └── simulator/main.go          # simulator mesin
│   ├── internal/
│   │   ├── config/config.go
│   │   ├── domain/types.go            # Reading, Anomaly, Event structs
│   │   ├── store/                     # pgx: readings, intervals, anomalies, outbox
│   │   ├── ingest/                    # channel + batch writer
│   │   ├── detector/                  # deteksi anomali (state per mesin)
│   │   ├── stream/                    # SSE hub
│   │   ├── odoo/                      # client REST Odoo
│   │   ├── outbox/                    # worker retry/backoff
│   │   ├── telegram/                  # client + bot poller
│   │   └── httpapi/                   # Fiber routes & handlers & middleware
│   └── migrations/
│       └── 0001_init.sql
└── monitor/                           # Next.js
    ├── package.json
    ├── next.config.ts                 # basePath: "/monitor", output: "standalone"
    ├── Dockerfile
    ├── middleware.ts                  # cek cookie login
    └── src/
        ├── app/
        │   ├── layout.tsx
        │   ├── page.tsx               # dashboard
        │   ├── login/page.tsx
        │   └── api/
        │       ├── login/route.ts
        │       └── stream/route.ts    # proxy SSE ke Go
        ├── components/
        │   ├── KpiCards.tsx
        │   ├── RunningMoTable.tsx
        │   ├── QcFailList.tsx
        │   └── LiveTemperatureChart.tsx
        └── lib/
            ├── odoo.ts                # fetch server-side ke Odoo
            └── types.ts
```
