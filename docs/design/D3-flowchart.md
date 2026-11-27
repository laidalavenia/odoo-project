# D3. Flowchart Proses Bisnis

Lima alur:

- (a) Pesanan → MO → produksi → QC → pengiriman
- (b) Pengadaan bahan baku
- (c) Layar operator
- (d) Data sensor & deteksi anomali
- (e) Notifikasi email & Telegram (termasuk penautan akun)

Kotak berwarna gelap = **logika custom `mrp_keripik` / Go**. Kotak lain = fitur standar Odoo.

---

## (a) Pesanan → MO → Produksi → QC → Pengiriman

Struktur BoM multi-level (detail di F1):

```mermaid
flowchart LR
    TM["Tempe Mentah (dibeli)"] -->|BoM 1: Pengirisan| TI["Tempe Iris"]
    TI -->|BoM 2: Penggorengan + Pembumbuan| KS["Keripik Siap Kemas"]
    TP["Tepung Bumbu, Minyak, Bumbu (dibeli)"] --> KS
    KS -->|BoM 3: Pengemasan| KK["Keripik Tempe 200 g (dijual)"]
    PL["Plastik Kemasan (dibeli)"] --> KK
    KS -.->|by-product| MB["Minyak Bekas, Remahan"]
```

Setiap produk setengah jadi punya route **Manufacture + MTO**, jadi satu SO memicu **rantai 3 MO** (Odoo menautkannya lewat smart button *Child/Source MO*). Alternatifnya: produk setengah jadi diproduksi ke stok dengan reorder rule. Untuk demo, rantai MTO lebih menarik. Di ERPNext ini mirip *"Use Multi-Level BOM"* tanpa sub-assembly WO, tapi Odoo benar-benar membuat MO terpisah per level.

```mermaid
flowchart TD
    A([Pelanggan checkout di website]) --> B[Quotation / SO draft]
    B --> C{Pembayaran online?}
    C -- Ya, provider demo --> D[SO otomatis Confirmed]
    C -- Tidak / transfer manual --> E[Admin Sales cek & klik Confirm] --> D
    D --> M1[/Email: Pesanan dikonfirmasi/]
    D --> F[Stock rule MTO membuat Delivery Order + MO Keripik 200 g]
    F --> G[MO level 1 butuh Keripik Siap Kemas -> MO level 2 -> MO level 3 Tempe Iris]
    G --> H[MO Confirmed: work order dibuat]
    H --> QG[[mrp_keripik: generate QC dari quality point per operasi]]
    QG --> I{Bahan tersedia? Check availability}
    I -- Tidak --> PR[Proses pengadaan, lihat alur b] --> I
    I -- Ya --> J[Operator kerjakan WO, lihat alur c]
    J --> K[[QC diisi manual / otomatis dari gateway]]
    K --> L{Semua QC pass?}
    L -- Ada fail --> L1[[Telegram ke PPIC & QC]] --> L2{Retest lulus?}
    L2 -- Ya --> L
    L2 -- Tidak --> L3[Scrap batch / MO dibatalkan sebagian] --> X([Selesai dengan tindakan korektif])
    L -- Ada todo --> L4[UserError: QC belum diisi] --> K
    L -- Semua pass --> N[PPIC klik Produce All / Mark as Done]
    N --> N1[Stok produk jadi + by-product masuk dengan lot]
    N1 --> N2[[Hitung biaya batch & yield]]
    N2 --> M2[/Email: Produksi selesai/]
    N2 --> T1[[Telegram: MO selesai]]
    N1 --> O[Delivery Order jadi Ready]
    O --> M3[/Email: Barang siap dikirim/]
    O --> P[Admin Gudang validasi pengiriman]
    P --> Q[Admin Sales buat invoice dari SO]
    Q --> Z([Selesai])

    classDef custom fill:#1f3b57,color:#fff,stroke:#0b1c2c
    class QG,K,L1,N2,T1 custom
```

**Catatan keputusan:**
- QC dibuat **saat MO dikonfirmasi** (bukan saat WO mulai), supaya QC berstatus `todo` sudah ada dan aturan "belum diisi" bisa memblokir.
- QC gagal **tidak dihapus**, tapi diselesaikan dengan **retest** (QC baru yang menunjuk QC lama). Jejak audit tetap ada, mirip *amend* di Frappe.

---

## (b) Pengadaan Bahan Baku

```mermaid
flowchart TD
    S([Scheduler / Run Scheduler]) --> A{Stok forecast di bawah min reorder rule?}
    A -- Tidak --> Z([Tidak ada aksi])
    A -- Ya --> B[RFQ dibuat ke vendor utama]
    M([MO butuh bahan, stok kurang]) --> B
    B --> C[Purchasing review harga & qty]
    C --> D{Setuju?}
    D -- Tidak --> D1[Cancel RFQ / ganti vendor] --> Z
    D -- Ya --> E[Confirm Order -> PO]
    E --> F[Receipt dibuat otomatis]
    F --> G[Barang datang: Admin Gudang buka receipt]
    G --> H[Isi Lot + Expiration Date per baris]
    H --> I{Lot & tanggal valid?}
    I -- Tidak --> H
    I -- Ya --> J[Validate: stok masuk ke WH/Stock/Bahan Baku]
    J --> K{Qty kurang dari PO?}
    K -- Ya --> K1[Backorder dibuat] --> G
    K -- Tidak --> L[Vendor bill, opsional di Invoicing]
    J --> X[[Cron harian: lot mendekati kedaluwarsa]]
    X --> X1{"Ada lot dengan alert_date sampai hari ini + N?"}
    X1 -- Ya --> X2[[Activity ke Admin Gudang + Telegram PPIC]]
    X1 -- Tidak --> Z2([Selesai])

    classDef custom fill:#1f3b57,color:#fff,stroke:#0b1c2c
    class X,X1,X2 custom
```

Padanan ERPNext: reorder rule ≈ *Item Reorder* + *Material Request* otomatis. Bedanya, Odoo langsung membuat **RFQ**, tanpa dokumen Material Request.

---

## (c) Alur Layar Operator

```mermaid
flowchart TD
    A([Operator login di tablet]) --> B[Menu Layar Operator: client action OWL]
    B --> C[Pilih Work Center]
    C --> D[Daftar WO: Ready / In Progress / Paused]
    D --> E[Tap kartu WO]
    E --> F{Status WO}
    F -- Ready / Paused --> G[Tombol MULAI]
    G --> G1[RPC: start WO -> time log dibuka]
    G1 --> G2[[Odoo kirim event started ke Go: machine_code -> WO]]
    F -- In Progress --> H[Panel kerja]
    G2 --> H
    H --> H1[Ringkasan sensor live: suhu terakhir, rata-rata]
    H --> H2[Form QC untuk operasi ini]
    H --> H3[Input qty hasil & scrap]
    H --> I{Aksi}
    I -- JEDA --> J[Pilih alasan jeda] --> J1[RPC pause WO + event paused] --> D
    I -- SIMPAN QC --> K[[Validasi nilai -> pass/fail]]
    K -- fail --> K1[Banner merah + Telegram] --> H
    K -- pass --> H
    I -- SELESAI --> L{QC wajib operasi ini sudah diisi?}
    L -- Belum --> L1[Pesan: Isi QC terlebih dahulu] --> H
    L -- Sudah --> M{"Qty hasil lebih dari 0?"}
    M -- Tidak --> M1[Pesan: Isi qty hasil] --> H
    M -- Ya --> N[RPC finish WO -> time log ditutup]
    N --> N1[[Event finished ke Go -> Go hitung & kirim ringkasan]]
    N1 --> O{WO terakhir di MO?}
    O -- Ya --> O1[Notifikasi PPIC: MO siap diselesaikan] --> D
    O -- Tidak --> O2[WO berikutnya jadi Ready] --> D

    classDef custom fill:#1f3b57,color:#fff,stroke:#0b1c2c
    class G2,K,N1 custom
```

---

## (d) Data Sensor & Deteksi Anomali

```mermaid
flowchart TD
    subgraph SIM["Simulator mesin"]
        S1[Penggoreng: suhu tiap 2 detik]
        S2[Timbangan: berat per kemasan]
    end
    S1 --> R[POST /gateway/api/v1/readings]
    S2 --> R
    R --> V{Token mesin valid & payload valid?}
    V -- Tidak --> V1[401 / 422, dicatat di log] --> END1([Ditolak])
    V -- Ya --> W[Tentukan WO aktif dari machine_code: tabel wo_intervals]
    W --> B[Masuk channel buffer -> batch insert ke sensor_db]
    W --> BR[Broadcast ke hub SSE -> Next.js]
    W --> DET{Detektor per mesin}
    DET -->|"Suhu di atas 180 C"| T1{"Sudah lebih dari 60 detik berturut-turut?"}
    T1 -- Belum --> T0[Catat waktu mulai di state detektor]
    T1 -- Ya --> T2{Anomali open untuk WO ini sudah ada?}
    T2 -- Ya --> T3[Update peak value saja, dedup]
    T2 -- Tidak --> A1[Insert anomalies status open]
    DET -->|"Berat di luar 195-210 g"| A1
    DET -- Normal --> T4{Ada anomali open?}
    T4 -- Ya --> T5[Tutup anomali: ended_at]
    T4 -- Tidak --> OK([Lanjut])
    A1 --> P1[Kirim Telegram ke PPIC & QC langsung]
    A1 --> P2[Outbox: POST Odoo quality-checks/auto-fail]
    P2 --> P3{"Odoo merespons 2xx?"}
    P3 -- Ya --> P4[Simpan odoo_qc_id]
    P3 -- Tidak --> P5[Retry exponential backoff] --> P2

    subgraph FIN["Saat WO selesai"]
        F1[Odoo: event finished] --> F2[Go tutup interval, hitung agregat]
        F2 --> F3[Outbox: POST Odoo workorders/id/sensor-summary]
        F3 --> F4[Odoo simpan keripik.sensor.summary + isi QC suhu/berat otomatis]
    end
```

**Kenapa outbox?** Kalau Odoo down saat anomali terjadi, permintaan tidak boleh hilang. Go menulis ke tabel `outbox` di transaksi yang sama dengan `anomalies`, lalu worker mengirimnya dengan retry. Ini pola **transactional outbox**, istilah yang bagus untuk interview Backend & Integration.

---

## (e) Notifikasi Email & Telegram

### e.1 Penautan akun Telegram

```mermaid
sequenceDiagram
    actor K as Karyawan
    participant O as Odoo (hr.employee)
    participant T as Telegram
    participant P as Poller (cron Odoo F6 / Go F9)

    K->>O: Klik "Hubungkan Telegram"
    O->>O: Generate kode unik (TTL 15 menit), simpan
    O-->>K: Tampilkan kode + link t.me/bot?start=KODE
    K->>T: Tekan Start / kirim /start KODE
    loop long polling
        P->>T: getUpdates(offset, timeout)
        T-->>P: update berisi /start KODE + chat.id
    end
    alt F6: poller = cron Odoo
        P->>O: cari karyawan dengan kode valid
    else F9: poller = Go
        P->>O: POST /api/keripik/v1/telegram/link {code, chat_id}
    end
    alt kode valid & belum kedaluwarsa
        O->>O: simpan telegram_chat_id, hapus kode
        P->>T: sendMessage "Halo Budi, akun terhubung"
    else kode tidak valid
        P->>T: sendMessage "Kode tidak valid / kedaluwarsa"
    end
```

### e.2 Pengiriman notifikasi

```mermaid
flowchart TD
    subgraph EV["Event bisnis di Odoo"]
        E1[SO confirmed]
        E2[MO done]
        E3[Delivery ready]
        E4[QC fail]
        E5[MO melewati tanggal deadline / tertunda]
        E6[Lot mendekati kedaluwarsa, cron]
    end
    E1 --> M[mail.template -> antrian mail.mail]
    E2 --> M
    E3 --> M
    M --> MS[Cron Mail Queue -> SMTP / Mailpit] --> C([Pelanggan])

    E2 --> TG
    E4 --> TG
    E5 --> TG
    E6 --> TG
    TG[[Tentukan penerima: karyawan dengan grup & preferensi notifikasi]]
    TG --> TG1{Punya telegram_chat_id?}
    TG1 -- Tidak --> TG2[Lewati + catat di log: belum terhubung]
    TG1 -- Ya --> TG3[Setelah commit: sendMessage]
    TG3 --> TG4{HTTP 200?}
    TG4 -- Ya --> TG5[Log sent]
    TG4 -- 403 user blokir bot --> TG6[Kosongkan chat_id, status putus]
    TG4 -- 429 rate limit --> TG7[Tunggu retry_after lalu ulang]
    TG4 -- Error jaringan --> TG8[Log failed, cron retry]

    classDef custom fill:#1f3b57,color:#fff,stroke:#0b1c2c
    class TG,TG1,TG3 custom
```

**Kenapa "setelah commit"?** Jika Telegram dikirim di tengah transaksi lalu transaksi di-rollback (misal error validasi), pesan sudah terlanjur terkirim padahal datanya tidak tersimpan. Odoo menyediakan `self.env.cr.postcommit.add(fn)`. Padanannya di Frappe: `frappe.enqueue(..., enqueue_after_commit=True)`.
