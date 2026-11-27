# Dokumentasi — Sistem Produksi Pabrik Keripik

Odoo 19 Community (manufacturing) + Go Machine Data Gateway + Next.js Monitor.
Mulai: **5 Oktober 2026** · Target: **±9 minggu** (lihat [D8](design/D8-roadmap.md)).

## Cara membaca
1. Baca **D1–D6** untuk memahami desain (±2 jam).
2. Review **D7 FSD**. Ini kontrak kebutuhan sekaligus acceptance criteria untuk pengujian.
3. Ikuti **D8 Roadmap** dan kerjakan **F1 → F12** berurutan. Setiap fase berisi ID requirement, konsep, langkah, kode, cara uji, error umum, dan pertanyaan interview.
4. **INSTALL.md** merangkum semua perintah instalasi.

> **Konvensi kode:**
> - Boilerplate (Docker, config, kerangka module, UI Next.js, simulator, README) ditulis **lengkap**.
> - Bagian inti (model, logika QC & yield, security, OWL, controller, logika gateway, Telegram) berisi **konsep + kerangka + petunjuk `TODO`** untuk kamu tulis sendiri.
> - Tanda ⚠️ = bergantung versi/kebijakan pihak ketiga. **Verifikasi di dokumentasi resmi.**

## Desain

| # | Dokumen | Isi |
|---|---|---|
| D1 | [Gambaran aplikasi](design/D1-gambaran-aplikasi.md) | Masalah, solusi, aktor & akses, Odoo vs Go, prioritas fitur, Community vs Enterprise, OCA |
| D2 | [Use case](design/D2-use-case.md) | 3 diagram use case + deskripsi |
| D3 | [Flowchart](design/D3-flowchart.md) | Pesanan→MO→QC→kirim, pengadaan, layar operator, sensor & anomali, notifikasi & penautan Telegram |
| D4 | [Data model](design/D4-data-model.md) | Model standar, model custom, skema sensor_db (ER diagram) |
| D5 | [Arsitektur](design/D5-arsitektur.md) | Diagram deployment & komponen Go, versi stack, alasan pilihan |
| D6 | [API, layar, struktur repo](design/D6-api-layar-struktur.md) | REST Odoo, endpoint Go, wireframe layar, struktur folder |
| D7 | [**FSD**](design/D7-FSD.md) | Kebutuhan fungsional (a)–(q), non-fungsional, asumsi, matriks keterlacakan |
| D8 | [Roadmap](design/D8-roadmap.md) | Gantt, rencana mingguan, checklist "selesai kalau…" |

## Implementasi

| # | Fase | Minggu |
|---|---|---|
| F1 | [Setup Odoo + Docker + konfigurasi standar](implementation/F01-setup-odoo-docker.md) | 1 |
| F2 | [Module mrp_keripik: QC & blokir MO](implementation/F02-module-quality-check.md) | 2 |
| F3 | [Laporan yield & traceability](implementation/F03-laporan-yield-traceability.md) | 3 |
| F4 | [Layar operator OWL](implementation/F04-layar-operator-owl.md) | 4 |
| F5 | [REST API, biaya batch, cron, email](implementation/F05-api-biaya-cron-email.md) | 5 |
| F6 | [Telegram dari Odoo](implementation/F06-telegram-odoo.md) | 6 |
| F7 | [Next.js Monitor + deploy v1](implementation/F07-nextjs-monitor-deploy-v1.md) ⭐ | 6–7 |
| F8 | [Dasar Go](implementation/F08-dasar-go.md) | paralel 5–7 |
| F9 | [Machine Data Gateway + simulator](implementation/F09-machine-data-gateway.md) | 8 |
| F10 | [Streaming, deploy lengkap, data demo, README](implementation/F10-streaming-deploy-readme.md) | 9 |
| F11 | [Persiapan interview](implementation/F11-persiapan-interview.md) | 9 |
| F12 | [Migrasi ke Odoo 20](implementation/F12-migrasi-odoo-20.md) | setelah rilis |

## Lainnya
- [INSTALL.md](INSTALL.md): step by step instalasi
- Membaca diagram: semua diagram berformat **Mermaid**. Tampil otomatis di GitHub, VS Code (ekstensi *Markdown Preview Mermaid Support*), Obsidian, atau <https://mermaid.live>.
