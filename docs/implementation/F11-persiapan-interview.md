# F11. Persiapan Interview

| | |
|---|---|
| **Minggu** | 9 (3–6 Des 2026) |
| **Hasil** | Bisa menjelaskan setiap keputusan teknis tanpa catatan + demo 5 menit terlatih |

---

## 1. Ringkasan Keputusan Teknis (hafalkan "kenapa"-nya)

| # | Keputusan | Kenapa | Alternatif & trade-off |
|---|---|---|---|
| 1 | Data sensor di Go + `sensor_db`, bukan Odoo | Volume/frekuensi tinggi; ORM write mahal; worker Odoo terbatas; DB bisnis tetap kecil; retensi & time-series; isolasi kegagalan | Tabel di Odoo (sederhana tapi tidak skala); TimescaleDB/Influx (lebih tepat untuk skala besar, menambah teknologi) |
| 2 | Odoo menerima ringkasan & event saja | Odoo = system of record bisnis; QC & laporan cukup agregat | Kirim data mentah per menit (kompromi, tetap membebani) |
| 3 | QC dibuat saat MO confirm | Supaya status `todo` bisa memblokir | Dibuat saat WO start (celah: WO dilewati) |
| 4 | Blokir di `button_mark_done` + WO finish | Titik masuk bisnis; validasi sebelum `super()` | Constraint di `write(state)` (state compute, rapuh) |
| 5 | Retest, bukan edit/hapus QC gagal | Jejak audit (pangan) | Edit nilai (hilang jejak) |
| 6 | Yield via SQL view | Agregasi di DB, pivot/graph gratis | Compute Python (lambat untuk banyak MO) |
| 7 | Traceability BFS + visited + depth | Graf lot bisa bercabang; cegah loop | Rekursi tanpa batas |
| 8 | Layar operator: 1 method data + 1 method aksi | Sedikit RPC, validasi server, mudah diuji | Panggil method standar langsung dari JS |
| 9 | Token custom ber-hash + scope | Least privilege, bisa dicabut, tidak membuka ORM penuh | API key + JSON-2/XML-RPC Odoo (lebih luas aksesnya) |
| 10 | Controller tipis, logika di model | Testable tanpa HTTP, reusable | Logika di controller |
| 11 | Outbox (Odoo & Go) + `postcommit` | Tidak ada event hilang; tidak ada efek samping untuk transaksi gagal | Panggil HTTP langsung (rapuh) / message broker (berat) |
| 12 | Idempotensi `external_ref`/`ref` | Retry aman | Tidak ada (duplikat) |
| 13 | Telegram anomali dari Go | Cepat, tidak bergantung Odoo | Lewat Odoo (bisa terlambat saat Odoo sibuk) |
| 14 | Satu poller Telegram (cron → Go) | Batasan Bot API | Webhook (butuh URL publik stabil) |
| 15 | Channel + batch `COPY` + 503 backpressure | Latensi rendah, efisien, beban terkendali | Insert per request (lambat), antrean tak terbatas (OOM) |
| 16 | Detector murni pakai timestamp reading | Deterministik, mudah diuji, tanpa mutex | Pakai `time.Now()` (flaky) |
| 17 | SSE lewat Next.js route | Token tetap di server; same-origin | Browser langsung ke Go (token bocor/CORS) |
| 18 | Nginx path-based satu domain | ngrok gratis 1 domain; tanpa CORS | Subdomain |
| 19 | Fiber | Familiar (Express-like), cepat | `net/http`+chi (idiomatik, kompatibel ekosistem) |
| 20 | Custom QC, bukan `quality_control_oca` | Kebutuhan spesifik + menunjukkan pemahaman | OCA (lebih cepat jika cocok; sudah dievaluasi) |

---

## 2. Bank Pertanyaan & Poin Jawaban

### 2.1 Odoo fungsional
1. **Jelaskan alur MTO di Odoo.** Route + stock rule; procurement dari SO line; MO terhubung ke delivery; multi-level karena komponen juga MTO.
2. **Beda BoM normal, kit, subcontracting?** Normal = membuat MO; kit (phantom) = meledakkan komponen di picking tanpa MO; subcontracting = pihak ketiga (module terpisah).
3. **Bagaimana Odoo menghitung ketersediaan bahan?** Reservasi `stock.move` → `stock.move.line` dari quant sesuai removal strategy (FEFO di sini).
4. **By-product & cost share?** `mrp.bom.byproduct` dengan `cost_share` persen; di sini 0%.
5. **Backorder MO?** Produksi sebagian → MO backorder untuk sisa qty.
6. **Apa yang tidak ada di Community dan bagaimana kamu menggantinya?** Quality, Shop Floor, IoT, Accounting penuh, Studio → custom module, OWL, Go gateway, perhitungan biaya sendiri.

### 2.2 Odoo teknis
1. **`_inherit` vs `_inherits` vs `_name+_inherit`?** Lihat F2 §1.2.
2. **`@api.depends` vs `@api.onchange` vs `@api.constrains`?** Compute (server, dependency tracking) / UI form sebelum simpan / validasi saat simpan.
3. **Kapan `store=True`?** Butuh search/group/sort, atau mahal dihitung; konsekuensinya harus depends yang lengkap.
4. **ACL vs record rule vs field groups vs `sudo()`?** F2 §1.4. Rule global AND, rule grup OR.
5. **Bagaimana mencegah N+1?** Prefetch ORM, `mapped`, `read_group`/`_read_group`, batch create.
6. **Transaksi di Odoo?** Satu request = satu transaksi; exception → rollback; `cr.savepoint()`; `postcommit` untuk efek samping; cron commit per job.
7. **Cara menulis test?** `TransactionCase`, `HttpCase`, `tagged('post_install','-at_install')`, `Form`, tour JS.
8. **QWeb report bagaimana?** `ir.actions.report` + template; `_get_report_values` untuk data custom; wkhtmltopdf.
9. **OWL?** Class component, template QWeb, `useState`, hooks lifecycle, services, registry.
10. **Perubahan versi yang kamu temui (17→19)?** `attrs` → ekspresi, `<tree>` → `<list>`, `numbercall` dihapus, `quantity_done` → `quantity`, `aggregator`, `models.Constraint` (19), produk storable → `is_storable`, URL `/odoo` ⚠️ (konfirmasi dengan pengalaman nyata saat implementasi).

### 2.3 Integrasi & backend
1. **Desain API-mu?** Versi di path, JSON envelope, kode error, idempotensi, scope token, limit.
2. **Bagaimana menjamin konsistensi Odoo ↔ Go?** Outbox dua arah + idempotensi + retry backoff + dead-letter; konsistensi eventual.
3. **Apa yang terjadi bila Odoo down 1 jam?** Go tetap ingest & deteksi; Telegram tetap terkirim (penerima di-cache); outbox menumpuk dan dikirim saat pulih; ringkasan tetap sampai.
4. **Bila Go down?** Odoo tetap jalan; event WO menumpuk di `keripik.gateway.event`; data sensor selama Go mati hilang kecuali mesin buffer/MQTT persistent → disebut sebagai batasan.
5. **Keamanan?** Hash token, constant-time compare, least privilege, tidak ada secret di repo, DB manager tertutup, proxy_mode hanya di balik proxy.
6. **Kenapa REST custom dan bukan JSON-RPC?** Kontrak sempit vs akses ORM luas; validasi bisnis; independen dari perubahan RPC eksternal Odoo.

### 2.4 Go
Lihat F8 §12 + F9 §8. Tambahan:
1. **Bagaimana kamu menghindari goroutine leak?** Setiap goroutine punya jalur keluar via `ctx.Done()`; WaitGroup di main; test dengan `-race`.
2. **Error wrapping?** `fmt.Errorf("...: %w", err)`, `errors.Is/As`, tipe `APIError` + `IsRetryable`.
3. **Kenapa pgx `CopyFrom`?** Protokol COPY PostgreSQL jauh lebih cepat untuk batch.

### 2.5 Perbandingan ERPNext (nilai tambah kamu)
| Topik | ERPNext | Odoo |
|---|---|---|
| Produksi | Work Order + Job Card | MO + Work Order |
| Quality | Quality Inspection built-in | Enterprise; di sini custom |
| Kustomisasi | DocType JSON, Customize Form, hooks | Model Python + XML view inheritance |
| Permission | Role + User Permission | Group + ACL + record rule |
| Background job | RQ + Redis | ir.cron (tanpa Redis) |
| Realtime | socket.io | bus + websocket (gevent) |
| Laporan | Query/Script report | SQL view model + pivot/graph, QWeb |

Cerita yang bagus: *"Di ERPNext saya terbiasa dengan Job Card dan Quality Inspection. Di Odoo Community, padanannya tidak ada (Quality = Enterprise), jadi saya membangunnya dengan memahami lifecycle MO/WO Odoo dan titik override yang tepat."*

### 2.6 Behavioral (format STAR)
- **Situasi sulit:** contoh "QC dobel saat MO dikonfirmasi ulang/backorder" → bagaimana kamu menemukan (test), memperbaiki (idempoten), mencegah (test regresi).
- **Trade-off:** "Long polling vs webhook untuk Telegram".
- **Belajar cepat:** Go dari nol dalam 3 minggu → latihan bertahap, test table-driven.
- **Jika diberi waktu lagi:** MQTT, metrics Prometheus, bus untuk layar operator, test E2E Playwright.

---

## 3. Skrip Demo 5 Menit

| Menit | Tampilkan | Kalimat kunci |
|---|---|---|
| 0:00 | README diagram | "Odoo untuk fakta bisnis, Go untuk telemetri." |
| 0:30 | Website → pesan 30 bungkus → SO → 3 MO berantai | "Make-to-order multi-level dengan route standar." |
| 1:15 | Layar operator di mode tablet: Mulai WO Penggorengan | "Client action OWL; semua aksi lewat satu method server." |
| 1:45 | Terminal: `simulator -scenario overheat` → Monitor grafik naik → banner anomali → Telegram masuk | "Deteksi di Go: >180 °C selama 60 dtk, dedup, outbox." |
| 2:45 | Odoo: QC gagal otomatis (🤖) → coba Mark as Done → ditolak → Retest → berhasil | "Aturan bisnis di Odoo, bukan di klien." |
| 3:30 | Ringkasan sensor di WO; laporan yield pivot; PDF | "SQL view untuk agregasi." |
| 4:00 | Wizard recall dari lot tempe mentah → pelanggan terdampak | "BFS di relasi consume/produce move line." |
| 4:30 | Matikan Odoo → overheat → hidupkan → QC tetap tercatat | "Transactional outbox, idempoten." |
| 5:00 | Penutup | "Yang ingin saya kembangkan berikutnya: MQTT & WhatsApp." |

Latih 3× dengan timer; siapkan database demo yang bisa di-restore cepat (`make backup`).

---

## 4. Checklist Hari-H
- [ ] Stack demo berjalan, domain ngrok aktif, data demo ada
- [ ] Simulator siap di terminal kedua
- [ ] Telegram di HP terbuka di chat bot
- [ ] Tab: website, backend MO, layar operator (mode device), monitor, Mailpit
- [ ] Backup DB terbaru
- [ ] Repo GitHub publik rapi: README, docs, tidak ada secret, commit history jelas
